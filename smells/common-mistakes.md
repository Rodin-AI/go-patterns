# Common Mistakes in Go

Code smells that come from writing Go like it's Java, Python, or C++.
These are patterns that compile fine but cause real problems at runtime,
in reviews, or when the codebase grows.

Each entry shows what people do, why it hurts, and what idiomatic Go looks like instead.

---

## 1. Nil Pointer Check After Use

**What they avoid:** Checking whether a pointer is nil before dereferencing it.

**Why it's bad:** The program panics on the dereference line, not at the nil check.
The nil check below the access is dead code — the crash already happened. This
pattern is especially insidious because it *looks* like the author considered nil
safety, and reviewers gloss over it.

```go
// BAD
func process(cfg *Config) {
    name := cfg.Name // panics here if cfg is nil
    if cfg == nil {
        return
    }
    fmt.Println(name)
}
```

```go
// GOOD
func process(cfg *Config) {
    if cfg == nil {
        return
    }
    name := cfg.Name
    fmt.Println(name)
}
```

### When to Apply This Rule

- Any function that receives a pointer parameter
- Methods on types that could be called on a nil receiver
- After type assertions that return a pointer (`val, ok := x.(*Foo)`)

### Exceptions

- Methods with documented nil-receiver behavior (e.g., `(*bytes.Buffer).Len()` returns 0 on nil)
- When the contract explicitly guarantees non-nil (internal unexported helpers where callers are controlled)

---

## 2. Goroutine Leak

**What they avoid:** Ensuring every goroutine has a path to termination.

**Why it's bad:** Leaked goroutines hold memory, file descriptors, and network
connections forever. In long-running services, this is a slow-motion OOM. The
goroutine count climbs monotonically and the only fix is a restart.

```go
// BAD
func fetchAll(urls []string) []string {
    results := make(chan string)
    for _, url := range urls {
        go func(u string) {
            resp, err := http.Get(u)
            if err != nil {
                return // goroutine exits, but nobody reads from channel
            }
            defer resp.Body.Close()
            body, _ := io.ReadAll(resp.Body)
            results <- string(body) // blocks forever if caller stops reading
        }(url)
    }

    var out []string
    // Only read first 3 — remaining goroutines leak
    for i := 0; i < 3; i++ {
        out = append(out, <-results)
    }
    return out
}
```

```go
// GOOD
func fetchAll(ctx context.Context, urls []string) ([]string, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make([]string, len(urls))

    for i, url := range urls {
        i, url := i, url
        g.Go(func() error {
            req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
            if err != nil {
                return err
            }
            resp, err := http.DefaultClient.Do(req)
            if err != nil {
                return err
            }
            defer resp.Body.Close()
            body, err := io.ReadAll(resp.Body)
            if err != nil {
                return err
            }
            results[i] = string(body)
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

### When to Apply This Rule

- Every `go func()` call — ask "what makes this goroutine stop?"
- Any channel send without a guaranteed receiver
- Background workers in servers (they need shutdown signals)

### Exceptions

- Goroutines that are genuinely meant to run for the process lifetime (e.g., a single background GC loop with no resources to release)
- Fire-and-forget logging where the channel is buffered and drained at shutdown

---

## 3. Interface Pollution

**What they avoid:** Keeping interfaces small and focused.

**Why it's bad:** Large interfaces (Java-style `Service` with 10+ methods) are
impossible to mock, impossible to satisfy partially, and couple every consumer
to every method. They defeat Go's implicit interface satisfaction — the primary
mechanism for decoupling.

```go
// BAD — Java-style "service interface"
type UserService interface {
    Create(ctx context.Context, u *User) error
    Update(ctx context.Context, u *User) error
    Delete(ctx context.Context, id string) error
    Get(ctx context.Context, id string) (*User, error)
    List(ctx context.Context, filter Filter) ([]*User, error)
    Search(ctx context.Context, q string) ([]*User, error)
    Activate(ctx context.Context, id string) error
    Deactivate(ctx context.Context, id string) error
    ChangePassword(ctx context.Context, id, pw string) error
    ResetPassword(ctx context.Context, id string) error
    AssignRole(ctx context.Context, id, role string) error
    RemoveRole(ctx context.Context, id, role string) error
}
```

```go
// GOOD — small interfaces defined by consumers
type UserGetter interface {
    Get(ctx context.Context, id string) (*User, error)
}

type UserWriter interface {
    Create(ctx context.Context, u *User) error
    Update(ctx context.Context, u *User) error
    Delete(ctx context.Context, id string) error
}

// Handlers accept only what they need
func NewProfileHandler(users UserGetter) *ProfileHandler {
    return &ProfileHandler{users: users}
}
```

### When to Apply This Rule

- When defining an interface with more than 3-5 methods
- When test mocks are painful to write
- When multiple consumers only use a subset of methods

### Exceptions

- Standard library compatibility (e.g., implementing `http.Handler`, `io.ReadWriteCloser`)
- Internal package boundaries where the full surface is always consumed together
- Wrapper types that must delegate the entire surface (e.g., middleware around a DB driver)

---

## 4. Stuttering Names

**What they avoid:** Considering how a name reads at the call site with its package qualifier.

**Why it's bad:** Go names are always read with their package prefix: `http.HTTPClient`
becomes "HTTP HTTP Client." This is noise. The stdlib gets this right (`http.Client`,
not `http.HTTPClient`), but newcomers from Java/C# (where package names aren't part
of the identifier at the call site) repeat context constantly.

```go
// BAD
package user

type UserService struct{}      // user.UserService
type UserRepository struct{}   // user.UserRepository
func NewUserService() {}       // user.NewUserService

package http

type HTTPClient struct{}       // http.HTTPClient
type HTTPResponse struct{}     // http.HTTPResponse
```

```go
// GOOD
package user

type Service struct{}          // user.Service
type Repository struct{}       // user.Repository
func NewService() {}           // user.NewService

package http

type Client struct{}           // http.Client
type Response struct{}         // http.Response
```

### When to Apply This Rule

- Exported types, functions, and constants — read them aloud with the package prefix
- If you hear the same word twice, you have stutter

### Exceptions

- When removing the prefix creates genuine ambiguity (`log.Logger` is fine even though `log.Log` might be "simpler")
- Test helper packages where the package name is generic (e.g., `testutil.TestHelper` — but consider renaming the package instead)

---

## 5. init() Abuse

**What they avoid:** Making initialization explicit and testable.

**Why it's bad:** `init()` runs at import time with no arguments and no error
return. You can't test it in isolation, can't skip it, can't control ordering
across packages, and can't handle errors gracefully. Complex init() functions
create invisible dependencies and make binaries fail at startup with no clear
trace.

```go
// BAD
package database

var db *sql.DB

func init() {
    var err error
    db, err = sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err) // crashes at import time
    }
    if err := db.Ping(); err != nil {
        log.Fatal(err) // network call during init
    }
}

func GetDB() *sql.DB { return db }
```

```go
// GOOD
package database

type DB struct {
    conn *sql.DB
}

func Open(dsn string) (*DB, error) {
    conn, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, fmt.Errorf("database open: %w", err)
    }
    if err := conn.Ping(); err != nil {
        conn.Close()
        return nil, fmt.Errorf("database ping: %w", err)
    }
    return &DB{conn: conn}, nil
}
```

### When to Apply This Rule

- init() that does I/O (network, disk, environment reads)
- init() that can fail (anything with an error you're ignoring or fatal-ing on)
- init() with side effects that affect tests

### Exceptions

- Registering drivers/codecs (`sql.Register`, `image.RegisterFormat`) — the standard pattern
- Setting simple package-level defaults that don't involve I/O
- `init()` that registers flags (before `flag.Parse()` runs)

---

## 6. Ignoring Errors

**What they avoid:** Handling (or explicitly documenting) every error return.

**Why it's bad:** `_ = someFunc()` silently swallows failures. The code continues
with invalid state, corrupt data, or half-finished operations. When things break
later, the root cause is invisible — you're debugging symptoms three layers removed
from the actual failure.

```go
// BAD
func saveUser(u *User) {
    data, _ := json.Marshal(u)           // what if Marshal fails?
    _ = os.WriteFile("user.json", data, 0644) // what if disk is full?
    _ = notifyService(u.ID)              // what if service is down?
}
```

```go
// GOOD
func saveUser(u *User) error {
    data, err := json.Marshal(u)
    if err != nil {
        return fmt.Errorf("marshal user %s: %w", u.ID, err)
    }
    if err := os.WriteFile("user.json", data, 0644); err != nil {
        return fmt.Errorf("write user file: %w", err)
    }
    if err := notifyService(u.ID); err != nil {
        // Log but don't fail — notification is best-effort
        log.Printf("warn: notify failed for user %s: %v", u.ID, err)
    }
    return nil
}
```

### When to Apply This Rule

- Every function call that returns an error
- Deferred calls that can fail (`defer f.Close()` — consider checking the error)
- Type assertions without the comma-ok form

### Exceptions

- `fmt.Fprintf` to stdout/stderr in CLI tools (the program is about to exit anyway)
- `(*bytes.Buffer).Write` — documented to never return an error
- Hash `Write` methods (`hash.Hash` — documented to never error)

---

## 7. Returning Concrete Types from Constructors

**What they avoid:** Returning interfaces from constructors to enable decoupling.

**Why it's bad:** Wait — this one is backwards from what Java/C# developers expect.
In Go, **returning concrete types from constructors is correct**. The smell is
the *opposite*: returning an interface from a constructor, which hides the concrete
type unnecessarily, prevents access to type-specific methods, and makes the code
harder to understand.

The Go proverb: "Accept interfaces, return structs."

```go
// BAD — returning interface from constructor (Java factory pattern)
type Storage interface {
    Save(key string, data []byte) error
    Load(key string) ([]byte, error)
}

// Caller can't access DiskStorage-specific methods (like Sync)
func NewStorage(path string) Storage {
    return &DiskStorage{path: path}
}
```

```go
// GOOD — return concrete, accept interface where needed
type DiskStorage struct {
    path string
}

func NewDiskStorage(path string) *DiskStorage {
    return &DiskStorage{path: path}
}

func (d *DiskStorage) Save(key string, data []byte) error { /* ... */ }
func (d *DiskStorage) Load(key string) ([]byte, error)   { /* ... */ }
func (d *DiskStorage) Sync() error                        { /* ... */ }

// Consumers declare what they need
type Saver interface {
    Save(key string, data []byte) error
}

func NewUploader(s Saver) *Uploader {
    return &Uploader{store: s}
}
```

### When to Apply This Rule

- Constructor functions (`New...`) — return the concrete pointer type
- Interfaces should be defined by consumers, not producers
- When you catch yourself writing a factory that returns an interface

### Exceptions

- When the constructor genuinely chooses between multiple implementations based on config (factory pattern with a legitimate reason)
- Standard library patterns like `errors.New` returning `error` (the interface IS the contract)

---

## 8. sync.Mutex Value Copying

**What they avoid:** Ensuring mutexes are never copied.

**Why it's bad:** A `sync.Mutex` contains internal state. Copying a locked mutex
creates a second mutex that is *also* locked — but independently. The copy and
original now protect nothing. This leads to data races that are invisible to the
race detector in some cases, because both copies appear to lock/unlock correctly
in isolation.

```go
// BAD — mutex embedded in struct passed by value
type Counter struct {
    mu    sync.Mutex
    count int
}

func (c Counter) Value() int { // VALUE RECEIVER — copies the mutex!
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}

func printCounter(c Counter) { // passed by value — copies the mutex!
    fmt.Println(c.Value())
}
```

```go
// GOOD
type Counter struct {
    mu    sync.Mutex
    count int
}

func (c *Counter) Value() int { // pointer receiver
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}

func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func printCounter(c *Counter) { // passed by pointer
    fmt.Println(c.Value())
}
```

### When to Apply This Rule

- Any struct containing `sync.Mutex`, `sync.RWMutex`, `sync.WaitGroup`, or `sync.Cond`
- All methods on such structs must use pointer receivers
- Never pass such structs by value to functions
- Run `go vet` — it catches some (but not all) mutex copy violations

### Exceptions

- None. There is no valid reason to copy a mutex. If `go vet` flags it, fix it.

---

## 9. Channel Misuse for Simple Synchronization

**What they avoid:** Using the simplest synchronization primitive for the job.

**Why it's bad:** Channels are for communication. When you just need "wait for N
goroutines" or "protect a shared variable," channels add unnecessary complexity:
extra goroutines to drain them, easy-to-miss deadlocks, and cognitive overhead
for readers. Using a channel where a `sync.WaitGroup` or `sync.Mutex` would suffice
is like using a message queue to increment a counter.

```go
// BAD — channel as a WaitGroup
func processAll(items []Item) {
    done := make(chan struct{})
    for _, item := range items {
        go func(it Item) {
            process(it)
            done <- struct{}{}
        }(item)
    }
    // Wait for all
    for range items {
        <-done
    }
}

// BAD — channel as a Mutex
func safeIncrement(counter *int, ch chan struct{}) {
    ch <- struct{}{} // "lock"
    *counter++
    <-ch // "unlock"
}
```

```go
// GOOD — WaitGroup for fan-out-wait
func processAll(items []Item) {
    var wg sync.WaitGroup
    for _, item := range items {
        wg.Add(1)
        go func(it Item) {
            defer wg.Done()
            process(it)
        }(item)
    }
    wg.Wait()
}

// GOOD — Mutex for shared state
type SafeCounter struct {
    mu sync.Mutex
    n  int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.n++
}
```

### When to Apply This Rule

- "Wait for N things to finish" → `sync.WaitGroup`
- "Protect shared state" → `sync.Mutex` / `sync.RWMutex`
- "Limit concurrency" → semaphore (`chan struct{}` with buffer) or `golang.org/x/sync/semaphore`

### Exceptions

- When you genuinely need to stream results back (producer/consumer) — channels are correct
- Select with cancellation — channels compose with `context.Done()`
- Pipeline patterns where data flows between stages

---

## 10. Premature Abstraction

**What they avoid:** Waiting until a pattern emerges before creating an abstraction.

**Why it's bad:** Defining an interface before you have two implementations means
you're guessing at the contract. The interface will be wrong — too broad, too narrow,
or shaped around implementation details of the single concrete type. Now you're
locked into a bad abstraction that's harder to change than no abstraction at all.

The Go proverb: "A little copying is better than a little dependency."

```go
// BAD — interface before the need exists
type Cache interface {
    Get(key string) ([]byte, error)
    Set(key string, val []byte, ttl time.Duration) error
    Delete(key string) error
    Keys(pattern string) ([]string, error)
    Flush() error
}

// Only implementation
type RedisCache struct{ client *redis.Client }

// The interface was designed around Redis's capabilities.
// When you add Memcached, half the methods don't map cleanly.
```

```go
// GOOD — start concrete, extract when needed
type RedisCache struct {
    client *redis.Client
}

func (r *RedisCache) Get(key string) ([]byte, error)                      { /* ... */ }
func (r *RedisCache) Set(key string, val []byte, ttl time.Duration) error { /* ... */ }
func (r *RedisCache) Delete(key string) error                             { /* ... */ }
func (r *RedisCache) Keys(pattern string) ([]string, error)               { /* ... */ }
func (r *RedisCache) Flush() error                                        { /* ... */ }

// Later, when you ACTUALLY add a second implementation:
// Extract only the interface that consumers need (see #3)
type Getter interface {
    Get(key string) ([]byte, error)
}

// The handler only needs Get — discovered through real usage
func NewHandler(cache Getter) *Handler {
    return &Handler{cache: cache}
}
```

### When to Apply This Rule

- You're about to write an interface and there's only one implementation
- You're adding an interface "for testing" — consider whether a thin concrete wrapper is simpler
- You're designing a "framework" inside an application

### Exceptions

- Interfaces required by external contracts (stdlib signatures, plugin systems)
- When the interface genuinely represents a boundary you know will have multiple implementations (e.g., `io.Reader` — the abstraction predates and outlives any single implementation)
- Test doubles for external services (database, HTTP APIs) where you control the consumer interface

---

## Summary

| # | Smell | Root Cause |
|---|-------|-----------|
| 1 | Nil check after use | Mechanical nil checks without thinking about order |
| 2 | Goroutine leak | "Fire and forget" mentality from async/await languages |
| 3 | Interface pollution | Java's "program to an interface" taken literally |
| 4 | Stuttering names | Ignoring that package name is part of the identifier |
| 5 | init() abuse | Treating init() like a class constructor |
| 6 | Ignoring errors | Python's "easier to ask forgiveness" without the try/except |
| 7 | Interface-returning constructors | Java/C# factory pattern cargo-culting |
| 8 | Mutex value copying | Not understanding Go's value semantics |
| 9 | Channel misuse | "Channels are Go's thing so I should use them everywhere" |
| 10 | Premature abstraction | OOP instinct to abstract before understanding |

---

*These patterns are drawn from real code review feedback, the Go stdlib design,
and common issues flagged by `go vet`, `staticcheck`, and `golangci-lint`.*

<!-- PATTERN_COMPLETE -->
