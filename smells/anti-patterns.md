# Anti-Patterns: What Go's Stdlib Avoids (and Why)

Patterns the Go standard library team actively avoids, extracted
from studying what they DON'T do in their source code.

**Source:** [golang/go](https://github.com/golang/go) at commit
[`17bd5ab`](https://github.com/golang/go/tree/17bd5ab8c650155dd2bd09f7005726552639eea0)

---

## 1. Returning Errors and Values Simultaneously

**What they avoid:** Functions that return a valid value alongside
a non-nil error.

**Source evidence:** The entire stdlib follows: if error is non-nil,
other return values are undefined/zero. `io.Reader.Read` is the ONE
exception (documented as "n > 0 AND err possible").

**Why it's bad:** Callers check the error OR use the value — not both.
If you return data with an error, callers might ignore the error
because they got "valid" data.

```go
// BAD — ambiguous: is result valid when err != nil?
func fetch(url string) ([]byte, error) {
    resp, err := http.Get(url)
    if err != nil {
        return partialData, err  // caller might use partialData without checking err
    }
    return io.ReadAll(resp.Body)
}

// GOOD — nil data when error is non-nil
func fetch(url string) ([]byte, error) {
    resp, err := http.Get(url)
    if err != nil {
        return nil, err
    }
    return io.ReadAll(resp.Body)
}
```

### When to Apply This Rule

**Triggers:**
- Returning non-zero values alongside non-nil errors
- Callers that use the value without checking the error

### Exceptions

- `io.Reader.Read()` — explicitly documented as "n > 0 AND err == io.EOF" case
- Functions that return "best effort" partial results (document this clearly)

---

## 2. Large Interfaces (Java-Style)

**What they avoid:** Interfaces with more than 1-3 methods.

**Source evidence:** `io.Reader` (1 method), `io.Writer` (1 method),
`fmt.Stringer` (1 method), `sort.Interface` (3 methods). The largest
stdlib interface is `net.Conn` at 8 methods — and it's widely
considered too large.

**Why it's bad:** Large interfaces are hard to implement (fewer types
satisfy the contract). Small interfaces compose: `io.ReadWriter` is
just `Reader + Writer`. The bigger the interface, the tighter the
coupling.

```go
// BAD — Java-style "service" interface
type UserService interface {
    Create(ctx context.Context, u *User) error
    Get(ctx context.Context, id string) (*User, error)
    Update(ctx context.Context, u *User) error
    Delete(ctx context.Context, id string) error
    List(ctx context.Context, filter Filter) ([]*User, error)
    Count(ctx context.Context) (int, error)
    Search(ctx context.Context, q string) ([]*User, error)
}
// Only ONE implementation will ever satisfy this. It's a class in disguise.

// GOOD — small, composable interfaces
type UserReader interface {
    GetUser(ctx context.Context, id string) (*User, error)
}
type UserWriter interface {
    SaveUser(ctx context.Context, u *User) error
}
// Functions accept the smallest interface they need
func sendWelcome(ctx context.Context, r UserReader, id string) error { ... }
```

### When to Apply This Rule

**Triggers:**
- Interface with 4+ methods
- Only one implementation exists
- Interface defined in the same package as the implementation

### Exceptions

- Interfaces matching an external protocol (net.Conn — mirrors TCP API)
- Well-known stdlib patterns (sort.Interface — exactly 3 methods)

---

## 3. Package-Level init() for Complex Logic

**What they avoid:** Using `init()` for anything beyond simple
registration or flag setup.

**Source evidence:** Stdlib init() functions are exclusively used for:
registering encodings, initializing hash algorithms, setting up flags.
Never for: opening files, making network calls, or complex computation.

**Why it's bad:** init() runs at import time — you can't control when.
Can't pass parameters. Can't return errors. Can't skip in tests.
Hard to debug (runs before main). Creates import side effects.

```go
// BAD — complex logic in init
func init() {
    db, err := sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)  // crashes the program at import time
    }
    globalDB = db
}

// GOOD — explicit initialization
func NewApp(dsn string) (*App, error) {
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, fmt.Errorf("opening database: %w", err)
    }
    return &App{db: db}, nil
}
```

### When to Apply This Rule

**Triggers:**
- init() that opens files, network connections, or databases
- init() with error handling (log.Fatal/panic)
- init() that takes >5 lines
- init() that depends on environment variables

### Exceptions

- Registering encodings: `encoding/gob`, `image/png` init registration
- Seeding randomness or initializing lookup tables
- Flag registration (`flag.String(...)` in init)

---

## 4. Stuttering Names

**What they avoid:** Package-qualified names that repeat the package
name: `http.HTTPServer`, `user.UserService`, `json.JSONEncoder`.

**Source evidence:** The stdlib uses: `http.Server` (not HTTPServer),
`json.Encoder` (not JSONEncoder), `bytes.Buffer` (not BytesBuffer).
The package name provides context.

**Why it's bad:** When you import and use `http.HTTPServer`, you say
"HTTP" twice. Go's package system already provides namespace context.
Stuttering is visual noise that adds no information.

```go
// BAD — stuttering
package user

type UserService struct { ... }     // user.UserService
type UserRepository interface { ... } // user.UserRepository
func NewUserService() *UserService { ... }

// GOOD — the package name provides context
package user

type Service struct { ... }        // user.Service
type Repository interface { ... }  // user.Repository
func NewService() *Service { ... }
```

### When to Apply This Rule

**Triggers:**
- Type name starts with the package name
- `New<PackageName><Type>` constructor pattern
- Reading the qualified name aloud sounds redundant

### Exceptions

- When omitting the prefix would be ambiguous (`error.Error` — Go uses this)
- Single-package binaries where qualification doesn't apply

---

## 5. Naked Returns in Long Functions

**What they avoid:** Using named return values with bare `return`
in functions longer than a few lines.

**Source evidence:** The stdlib uses naked returns only in very short
functions (1-5 lines). Longer functions always return explicit values.
The `go vet` tool warns about this.

**Why it's bad:** In a 50-line function with naked return, you have
to trace every assignment to the named returns to understand what's
being returned. It's like having invisible variables.

```go
// BAD — naked return in long function
func process(data []byte) (result int, err error) {
    // ... 40 lines of code ...
    if something {
        result = compute()
        return  // what is err here? was it set earlier? who knows
    }
    // ... more code ...
    return  // mystery values
}

// GOOD — explicit returns
func process(data []byte) (int, error) {
    // ... 40 lines of code ...
    if something {
        return compute(), nil  // clear what's being returned
    }
    return 0, fmt.Errorf("processing: %w", err)
}
```

### When to Apply This Rule

**Triggers:**
- Function body > 10 lines with naked returns
- Multiple return points with naked returns
- Named returns used ONLY for naked return (not for documentation)

### Exceptions

- Short functions (3-5 lines) where named returns add clarity
- Defer-based error wrapping: `defer func() { err = wrap(err) }()`
  (this is the primary legitimate use of named returns)

---

## 6. Error String Formatting (Capitalization/Punctuation)

**What they avoid:** Error messages that start with capitals or end
with punctuation.

**Source evidence:** Every error string in the stdlib is lowercase,
no trailing period. `fmt.Errorf("open %s: %w", path, err)` — never
`"Failed to open file."`.

**Why it's bad:** Errors compose. `fmt.Errorf("connect: %w", err)`
produces `connect: open /etc/hosts: permission denied`. If inner
errors are capitalized or punctuated, the chain looks broken:
`connect: Failed to open file.`

```go
// BAD — capitalized, punctuated
return fmt.Errorf("Failed to open configuration file: %s.", path)

// GOOD — lowercase, no punctuation, wraps cleanly
return fmt.Errorf("open config %s: %w", path, err)
```

### When to Apply This Rule

**Triggers:**
- Error strings starting with uppercase
- Error strings ending with `.` or `!`
- Error messages that don't compose well with wrapping

### Exceptions

- Proper nouns: `"Google API returned 503"`
- Acronyms at the start: `"DNS lookup failed"` (some teams accept this)

---

## 7. Overusing Channels (When Mutex Suffices)

**What they avoid:** Using channels for simple mutual exclusion
where `sync.Mutex` would be clearer.

**Source evidence:** The stdlib uses Mutex 540 times vs channels
for synchronization ~50 times. Channels are for communication
between goroutines. Mutex is for protecting shared state.

**Why it's bad:** A channel of capacity 1 as a mutex is clever
but obscure. It adds cognitive load, makes locking/unlocking
asymmetric, and doesn't have `defer mu.Unlock()` ergonomics.

```go
// BAD — channel as mutex (clever but obscure)
sem := make(chan struct{}, 1)
sem <- struct{}{}  // "lock"
// ... critical section ...
<-sem  // "unlock"

// GOOD — mutex is clear and idiomatic
var mu sync.Mutex
mu.Lock()
defer mu.Unlock()
// ... critical section ...
```

### When to Apply This Rule

**Triggers:**
- `make(chan struct{}, 1)` used as a lock
- Channel send/receive pairs that don't actually communicate data
- Complex select statements where a simple lock would work

### Exceptions

- Rate limiting (semaphore with capacity > 1)
- Signaling between goroutines (cancellation, done channels)
- Fan-out/fan-in patterns (actual communication)

---

## 8. Accepting Interfaces, Returning Interfaces

**What they avoid:** Returning interfaces from functions (except
well-known ones like `error`).

**Source evidence:** The Go proverb: "Accept interfaces, return structs."
The stdlib returns concrete types: `http.NewServeMux()` returns
`*ServeMux`, not `http.Handler`. This allows callers to access
all methods.

**Why it's bad:** Returning an interface hides the concrete type.
Callers can't access methods that aren't in the interface. Makes
testing harder (can't construct the concrete type). Prevents
future method additions without breaking the interface.

```go
// BAD — returns interface (hides concrete type)
func NewCache() Cacher {
    return &redisCache{...}
}
// Callers can't access redis-specific methods or fields

// GOOD — returns concrete type
func NewCache() *RedisCache {
    return &RedisCache{...}
}
// Callers get the full type. They can still use it as Cacher where needed.
```

### When to Apply This Rule

**Triggers:**
- Factory functions returning interfaces
- Single-implementation interfaces returned from constructors
- Interface return types that prevent callers from accessing useful methods

### Exceptions

- Standard error returns (`error` interface)
- When there genuinely are multiple implementations chosen at runtime
- Plugin systems where the concrete type must be hidden

---

## 9. Panic in Library Code

**What they avoid:** Using `panic()` for conditions callers could
handle.

**Source evidence:** stdlib panic usage is exclusively for: (1) index
out of bounds, (2) nil pointer on documented non-nil parameter,
(3) impossible states (unreachable code). Never for "file not found"
or "invalid input" style errors.

**Why it's bad:** panic kills the goroutine (and the program unless
recovered). Library code shouldn't decide that an error is fatal —
that's the caller's decision.

```go
// BAD — panicking on recoverable error
func MustParse(s string) Config {
    cfg, err := Parse(s)
    if err != nil {
        panic(err)  // caller has no way to handle bad input
    }
    return cfg
}

// GOOD — return error, let caller decide
func Parse(s string) (Config, error) {
    // ... parse logic ...
    if invalid {
        return Config{}, fmt.Errorf("parse config: %w", err)
    }
    return cfg, nil
}
```

### When to Apply This Rule

**Triggers:**
- `panic()` in exported functions on user input
- `Must*` functions without a non-panicking alternative
- `log.Fatal` in library code (same as panic — kills the process)

### Exceptions

- `Must*` pattern for compile-time constants: `regexp.MustCompile("^\\d+$")`
- Truly impossible states after exhaustive validation
- Test helpers that panic on setup failure

---

## 10. Unexported Interface Satisfaction

**What they avoid:** Defining interfaces in the implementor's package
rather than the consumer's package.

**Source evidence:** The Go wiki explicitly says: "Define interfaces
in the package that USES them, not the package that implements them."
The stdlib follows this — `io.Reader` is in `io` (consumer), not
in `os` or `net` (implementors).

**Why it's bad:** Pre-declaring interfaces in the implementor package
couples consumer to producer. It's Java-style "implements Readable"
thinking. In Go, satisfaction is implicit — the consumer defines
what it needs.

```go
// BAD — interface in the implementation package
package storage

type Store interface {  // defined where it's implemented
    Get(key string) ([]byte, error)
    Set(key string, val []byte) error
}

type RedisStore struct { ... }
// Now everyone who imports storage gets this interface forced on them

// GOOD — interface in the consumer package
package handler

type Getter interface {  // defined where it's used
    Get(key string) ([]byte, error)
}

func NewHandler(store Getter) *Handler { ... }
// handler only depends on what it actually needs
```

### When to Apply This Rule

**Triggers:**
- Interface defined next to its only implementation
- Interface in the same package as the struct that satisfies it
- Interface with methods matching exactly one struct

### Exceptions

- Well-known "standard" interfaces (io.Reader, sort.Interface)
- Interfaces that define a protocol many packages implement
- Plugin/driver interfaces where the contract IS the package's purpose

<!-- PATTERN_COMPLETE -->
