# Go Concurrency Patterns

Patterns extracted from the Go standard library source code.

---

## 1. sync.Mutex — The Basic Lock

### Source: `src/sync/mutex.go:18-34`, `src/sync/mutex.go:42-67`

```go
// src/sync/mutex.go:18-34
// A Mutex is a mutual exclusion lock.
// The zero value for a Mutex is an unlocked mutex.
//
// A Mutex must not be copied after first use.
type Mutex struct {
    _ noCopy
    mu isync.Mutex
}

// src/sync/mutex.go:36-39
type Locker interface {
    Lock()
    Unlock()
}

// src/sync/mutex.go:43-46
func (m *Mutex) Lock() {
    m.mu.Lock()
}
```

### Why

- **Zero value is ready to use** — no constructor needed
- **Must not be copied** — enforced by `noCopy` field (go vet detects copies)
- **Not associated with a goroutine** — one goroutine can Lock, another can Unlock
- **Locker interface** — abstracts over Mutex and RWMutex

### When to Use

**Triggers:**
- Multiple goroutines read AND write the same data structure
- You need to protect a small critical section (a few field accesses)
- A channel-based solution would add complexity without benefit (no coordination needed, just protection)

**Example — before:**
```go
type Stats struct {
    hits   int
    misses int
}

func (s *Stats) RecordHit()  { s.hits++ }   // DATA RACE when called from multiple goroutines
func (s *Stats) RecordMiss() { s.misses++ } // DATA RACE
```

**Example — after:**
```go
type Stats struct {
    mu     sync.Mutex
    hits   int
    misses int
}

func (s *Stats) RecordHit() {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.hits++
}
```


### When NOT to Use

**Don't use this when:**
- Goroutines need to coordinate or communicate (not just protect state) — use channels
- The critical section involves blocking I/O (holding a mutex during network calls starves other goroutines)
- You can restructure to have a single goroutine own the state (no sharing = no lock needed)

**Over-application example:**
```go
// Using a mutex to coordinate work between goroutines
type TaskQueue struct {
    mu    sync.Mutex
    tasks []Task
    ready bool
}

func (q *TaskQueue) WaitForReady() {
    for {
        q.mu.Lock()
        if q.ready {
            q.mu.Unlock()
            return
        }
        q.mu.Unlock()
        time.Sleep(10 * time.Millisecond) // spin-waiting with a lock — terrible
    }
}
```

**Better alternative:**
```go
// Use a channel for coordination/signaling
type TaskQueue struct {
    tasks chan Task
    ready chan struct{}
}

func (q *TaskQueue) WaitForReady() {
    <-q.ready // blocks efficiently, no spinning
}
```

**Why:** Mutexes protect data; channels coordinate goroutines. If you're polling a mutex-protected flag, you've reinvented a bad channel. The Go proverb applies: "Don't communicate by sharing memory; share memory by communicating."


### Idiomatic Usage

```go
var mu sync.Mutex
mu.Lock()
defer mu.Unlock()
// ... critical section ...
```

### Anti-pattern

```go
// DON'T: Copy a mutex
type Config struct {
    mu sync.Mutex
    data map[string]string
}
c2 := *c1  // COPIES the mutex — data race

// DON'T: Forget defer
mu.Lock()
doSomething()  // if this panics, mutex stays locked forever
mu.Unlock()
```

---

## 2. sync.Once — Exactly-Once Initialization

### Source: `src/sync/once.go:12-36`, `src/sync/once.go:56-79`

```go
// src/sync/once.go:12-23
type Once struct {
    _ noCopy
    done atomic.Bool
    m    Mutex
}

// src/sync/once.go:56-63
func (o *Once) Do(f func()) {
    if !o.done.Load() {
        o.doSlow(f)
    }
}

// src/sync/once.go:65-72
func (o *Once) doSlow(f func()) {
    o.m.Lock()
    defer o.m.Unlock()
    if !o.done.Load() {
        defer o.done.Store(true)
        f()
    }
}
```

### Why

The implementation reveals a subtle guarantee: **when Do returns, f has finished**. The naive CAS-only approach (documented in comment at line 56-63) would let the second caller return before f completes. The mutex ensures all callers wait.

The `done` field is first in the struct for hot-path performance on amd64/386 (noted in comment at line 24-27).

### When to Use

**Triggers:**
- You have expensive initialization that should happen exactly once (DB connection, config parse, compiled regex)
- Multiple goroutines may trigger the initialization concurrently
- You're using `var` + `if instance == nil` checks that aren't goroutine-safe

**Example — before:**
```go
var db *sql.DB

func GetDB() *sql.DB {
    if db == nil { // RACE: two goroutines can both see nil
        db, _ = sql.Open("postgres", connStr)
    }
    return db
}
```

**Example — after:**
```go
var (
    db   *sql.DB
    once sync.Once
)

func GetDB() *sql.DB {
    once.Do(func() {
        db, _ = sql.Open("postgres", connStr)
    })
    return db
}
```


### When NOT to Use

**Don't use this when:**
- Initialization can fail and you need to retry — Once runs the func at most once, even on failure
- You need to reset/reinitialize later (Once has no Reset method)
- The initialization is cheap — just do it in `init()` or at declaration time

**Over-application example:**
```go
var (
    conn *grpc.ClientConn
    once sync.Once
)

func GetConn() (*grpc.ClientConn, error) {
    var err error
    once.Do(func() {
        conn, err = grpc.Dial("server:443")
    })
    if err != nil {
        return nil, err // PROBLEM: next call to GetConn returns nil conn, nil err
        // because once.Do won't run again
    }
    return conn, nil
}
```

**Better alternative:**
```go
// Use sync.OnceValues (Go 1.21+) which caches both value and error,
// or handle retry logic explicitly
var getConn = sync.OnceValues(func() (*grpc.ClientConn, error) {
    return grpc.Dial("server:443")
})

// Or for retry scenarios, use a mutex with a nil check
var (
    mu   sync.Mutex
    conn *grpc.ClientConn
)

func GetConn() (*grpc.ClientConn, error) {
    mu.Lock()
    defer mu.Unlock()
    if conn != nil {
        return conn, nil
    }
    var err error
    conn, err = grpc.Dial("server:443") // retries on next call if this fails
    return conn, err
}
```

**Why:** `sync.Once` guarantees exactly-once execution regardless of success or failure. If the initialization can fail transiently (network timeout, service unavailable), Once will permanently cache the failure. Use `sync.OnceValues` for caching results, or a mutex with a nil-check pattern when retry is needed.


### Idiomatic Usage

```go
var (
    instance *DB
    once     sync.Once
)

func GetDB() *DB {
    once.Do(func() {
        instance = connectToDB()
    })
    return instance
}
```

### Anti-pattern

```go
// DON'T: Call Do recursively (deadlocks)
var once sync.Once
once.Do(func() {
    once.Do(func() { /* deadlock */ })
})
```

---

## 3. sync.WaitGroup — Waiting for Goroutine Completion

### Source: `src/sync/waitgroup.go:14-43`, `src/sync/waitgroup.go:236-260`

```go
// src/sync/waitgroup.go:14-43
// Typically, a main goroutine will start tasks by calling WaitGroup.Go
// and then wait for all tasks to complete by calling WaitGroup.Wait:
//
//   var wg sync.WaitGroup
//   wg.Go(task1)
//   wg.Go(task2)
//   wg.Wait()
type WaitGroup struct {
    noCopy noCopy
    state  atomic.Uint64
    sema   uint32
}
```

### Go 1.25+: WaitGroup.Go

```go
// src/sync/waitgroup.go:236-260
func (wg *WaitGroup) Go(f func()) {
    wg.Add(1)
    go func() {
        defer func() {
            if x := recover(); x != nil {
                // Don't call Done — let panic propagate fatally.
                panic(x)
            }
            wg.Done()
        }()
        f()
    }()
}
```

### Why

`WaitGroup.Go` encapsulates the Add/go/Done pattern. Key design: if `f` panics, it re-panics **without** calling Done, preventing the main goroutine from racing to exit before the panic stack trace prints.

### Classic Pattern (pre-Go 1.25)

```go
var wg sync.WaitGroup
for _, item := range items {
    wg.Add(1)
    go func() {
        defer wg.Done()
        process(item)
    }()
}
wg.Wait()
```

### Anti-pattern

```go
// DON'T: Add inside the goroutine (race with Wait)
for _, item := range items {
    go func() {
        wg.Add(1)  // might run after Wait is called!
        defer wg.Done()
        process(item)
    }()
}
wg.Wait()
```

---

## 4. sync.Pool — Object Reuse for GC Pressure

### Source: `src/sync/pool.go:44-63`

```go
// src/sync/pool.go:44-63
// Pool's purpose is to cache allocated but unused items for later reuse,
// relieving pressure on the garbage collector. That is, it makes it easy to
// build efficient, thread-safe free lists.
//
// An appropriate use of a Pool is to manage a group of temporary items
// silently shared among and potentially reused by concurrent independent
// clients of a package. Pool provides a way to amortize allocation overhead
// across many clients.
//
// An example of good use of a Pool is in the fmt package, which maintains a
// dynamically-sized store of temporary output buffers.
type Pool struct {
    noCopy noCopy
    local     unsafe.Pointer
    localSize uintptr
    victim     unsafe.Pointer
    victimSize uintptr
    New func() any
}
```

### Why

Pool is **not** a general cache. Items can vanish between GC cycles. Use for reducing allocation pressure on hot paths.

### Idiomatic Usage (from fmt package)

```go
var ppFree = sync.Pool{
    New: func() any { return new(pp) },
}

func newPrinter() *pp {
    p := ppFree.Get().(*pp)
    p.reset()
    return p
}

func (p *pp) free() {
    p.buf = p.buf[:0]  // reset before returning
    ppFree.Put(p)
}
```

### Anti-pattern

```go
// DON'T: Use Pool for connection pooling (items disappear!)
var connPool = sync.Pool{
    New: func() any { return connectToDB() },
}
// Connections may be GC'd — use database/sql's pool instead

// DON'T: Put dirty objects back without resetting
pool.Put(buf)  // still has data from last use
```

---

## 5. Channel as Done Signal (Context Pattern)

### Source: `src/context/context.go:83-100` (Done channel), `src/io/pipe.go:42-45`

```go
// src/context/context.go:83-100
// Done returns a channel that's closed when work done on behalf of this
// context should be canceled.
Done() <-chan struct{}

// src/io/pipe.go:42-45
type pipe struct {
    once sync.Once
    done chan struct{}  // closed on pipe close
}
```

### Why

A `chan struct{}` costs zero bytes per send and closing it broadcasts to all receivers simultaneously. This is the canonical "done" signal in Go.

```go
select {
case <-ctx.Done():
    return ctx.Err()
case result := <-work:
    return result, nil
}
```

### When to Use

**Triggers:**
- You need to broadcast "stop" to multiple goroutines simultaneously
- A goroutine needs to select between work and cancellation
- You're implementing graceful shutdown for a long-running service

**Example — before:**
```go
type Server struct {
    stopped bool // RACE: no synchronization
}

func (s *Server) worker() {
    for {
        if s.stopped { return } // busy-polls, racy
        doWork()
    }
}
```

**Example — after:**
```go
type Server struct {
    done chan struct{}
}

func NewServer() *Server {
    return &Server{done: make(chan struct{})}
}

func (s *Server) worker() {
    for {
        select {
        case <-s.done:
            return
        case work := <-s.workCh:
            process(work)
        }
    }
}

func (s *Server) Stop() { close(s.done) } // broadcasts to ALL workers
```


### When NOT to Use

**Don't use this when:**
- You need to send actual data between goroutines — use a typed channel
- You only have one goroutine to signal — a simple `return` or function call suffices
- You're using it as a poor substitute for `context.Context` (which already provides Done())

**Over-application example:**
```go
// Rolling your own done channel when context already provides this
type Worker struct {
    done   chan struct{}
    ctx    context.Context
    cancel context.CancelFunc
}

func (w *Worker) Start() {
    go func() {
        select {
        case <-w.done:   // redundant — ctx.Done() does the same thing
            return
        case <-w.ctx.Done():
            return
        }
    }()
}
```

**Better alternative:**
```go
// Just use the context — it already IS a done signal
type Worker struct {
    ctx    context.Context
    cancel context.CancelFunc
}

func (w *Worker) Start() {
    go func() {
        select {
        case <-w.ctx.Done():
            return
        case work := <-w.workCh:
            process(work)
        }
    }()
}

func (w *Worker) Stop() { w.cancel() }
```

**Why:** If you already have a `context.Context`, its `Done()` channel is your cancellation signal. Adding a separate `chan struct{}` duplicates functionality and creates two shutdown paths that must be kept in sync. Use raw done channels only when you don't have a context (e.g., standalone libraries that predate context).


### Anti-pattern

```go
// DON'T: Use chan bool for done signals
done := make(chan bool)  // wastes 1 byte, true/false meaningless

// DON'T: Send to done (only unblocks one receiver)
done <- struct{}{}

// DO: Close the channel (broadcasts to all)
close(done)
```

---

## 6. Context Propagation Rules

### Source: `src/context/context.go:37-48`

```go
// src/context/context.go:37-48
// Do not store Contexts inside a struct type; instead, pass a Context
// explicitly to each function that needs it. The Context should be the first
// parameter, typically named ctx:
//
//   func DoSomething(ctx context.Context, arg Arg) error {
//       // ... use ctx ...
//   }
//
// Do not pass a nil Context, even if a function permits it. Pass context.TODO
// if you are unsure about which Context to use.
```

### Why

Context flows **down** the call chain, never stored in structs. It carries deadlines and cancellation signals for the current request, not persistent state.

### Anti-pattern

```go
// DON'T: Store context in a struct
type Server struct {
    ctx context.Context  // stale context persists beyond request
}

// DON'T: Pass nil
doWork(nil, data)  // use context.TODO() if unsure

// DON'T: Put context anywhere other than first parameter
func doWork(data Data, ctx context.Context)  // wrong position
```

---

## 7. Context Cancellation with Timeout

### Source: `src/net/http/server.go:4007-4050` (TimeoutHandler)

```go
// src/net/http/server.go:4011-4050
func (h *timeoutHandler) ServeHTTP(w ResponseWriter, r *Request) {
    ctx, cancelCtx := context.WithTimeout(r.Context(), h.dt)
    defer cancelCtx()
    r = r.WithContext(ctx)
    done := make(chan struct{})
    panicChan := make(chan any, 1)
    go func() {
        defer func() {
            if p := recover(); p != nil {
                panicChan <- p
            }
        }()
        h.handler.ServeHTTP(tw, r)
        close(done)
    }()
    select {
    case p := <-panicChan:
        panic(p)
    case <-done:
        // handler completed — copy response
    case <-ctx.Done():
        // timeout — write 503
    }
}
```

### Why

This is the full pattern: context with timeout + goroutine + select on done/timeout/panic. Key details:
1. `defer cancelCtx()` — always release resources
2. Panic propagation via dedicated channel
3. Select on three outcomes: success, timeout, panic

### Anti-pattern

```go
// DON'T: Forget to call cancel (leaks timer goroutines)
ctx, _ := context.WithTimeout(parent, 5*time.Second)

// DON'T: Ignore context in long operations
func longWork(ctx context.Context) {
    time.Sleep(10 * time.Minute)  // ignores cancellation
}
```

---

## 8. Select with Non-Blocking Check

### Source: `src/io/pipe.go:51-60`

```go
// src/io/pipe.go:51-60
func (p *pipe) read(b []byte) (n int, err error) {
    select {
    case <-p.done:
        return 0, p.readCloseError()
    default:
    }

    select {
    case bw := <-p.wrCh:
        nr := copy(b, bw)
        p.rdCh <- nr
        return nr, nil
    case <-p.done:
        return 0, p.readCloseError()
    }
}
```

### Why

The double-select pattern: first a non-blocking check (with `default`), then a blocking wait. The non-blocking check prevents a race where `done` was closed between the last operation and entering the blocking select.

### Anti-pattern

```go
// DON'T: Check ctx.Done() in a busy loop
for {
    select {
    case <-ctx.Done():
        return
    default:
    }
    // busy-spins CPU at 100%!
}
```

---

## 9. Channel Pipeline (io.Pipe)

### Source: `src/io/pipe.go:38-45`, `src/io/pipe.go:195-205`

```go
// src/io/pipe.go:38-45
type pipe struct {
    wrCh chan []byte   // writer sends data slices
    rdCh chan int      // reader returns bytes consumed
    done chan struct{}
}

// src/io/pipe.go:195-205
func Pipe() (*PipeReader, *PipeWriter) {
    pw := &PipeWriter{r: PipeReader{pipe: pipe{
        wrCh: make(chan []byte),   // unbuffered
        rdCh: make(chan int),      // unbuffered
        done: make(chan struct{}),
    }}}
    return &pw.r, pw
}
```

### Why

`io.Pipe` uses **unbuffered channels** — each Write blocks until Read consumes. Backpressure is automatic. The `done` channel signals shutdown.

### Pipeline Pattern Template

```go
func generate(ctx context.Context) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for i := 0; ; i++ {
            select {
            case out <- i:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

### When to Use

**Triggers:**
- You have a producer-consumer flow where the consumer's speed should limit the producer (backpressure)
- Data flows through multiple transformation stages
- You want to decouple stages that can run concurrently

**Example — before:**
```go
func processAll(items []string) []Result {
    var results []Result
    for _, item := range items {
        fetched := fetch(item)      // sequential: fetch then transform
        results = append(results, transform(fetched))
    }
    return results
}
```

**Example — after:**
```go
func processAll(ctx context.Context, items []string) []Result {
    fetched := make(chan Fetched)
    go func() {
        defer close(fetched)
        for _, item := range items {
            select {
            case fetched <- fetch(item): // backpressure: blocks if transform is slow
            case <-ctx.Done():
                return
            }
        }
    }()

    var results []Result
    for f := range fetched {
        results = append(results, transform(f))
    }
    return results
}
```


### When NOT to Use

**Don't use this when:**
- The "pipeline" has only one stage — you're just adding goroutine overhead to sequential code
- All items are already in memory and processing is CPU-bound with no I/O — `for` loop is simpler and faster
- You don't actually need backpressure (data fits in memory, producer is finite)

**Over-application example:**
```go
// Channel pipeline for a simple in-memory transformation
func doubleAll(nums []int) []int {
    ch := make(chan int)
    go func() {
        defer close(ch)
        for _, n := range nums {
            ch <- n // channel overhead for no benefit
        }
    }()

    var result []int
    for n := range ch {
        result = append(result, n*2)
    }
    return result
}
```

**Better alternative:**
```go
// Just use a loop — no concurrency needed
func doubleAll(nums []int) []int {
    result := make([]int, len(nums))
    for i, n := range nums {
        result[i] = n * 2
    }
    return result
}
```

**Why:** Channel pipelines shine when stages involve I/O (network, disk) and can overlap waiting. For pure computation on in-memory data, the channel send/receive overhead (~50-100ns per item) adds up and the goroutine scheduling has no useful work to overlap with. A plain loop is faster, simpler, and easier to debug.


### Anti-pattern

```go
// DON'T: Forget to close channels (receivers block forever)
func produce() <-chan int {
    ch := make(chan int)
    go func() {
        for i := 0; i < 10; i++ {
            ch <- i
        }
        // forgot close(ch) — range receivers hang
    }()
    return ch
}
```

---

## 10. Background Worker with Context Shutdown

### Source: `src/database/sql/sql.go:836-843`

```go
// src/database/sql/sql.go:836-843
func OpenDB(c driver.Connector) *DB {
    ctx, cancel := context.WithCancel(context.Background())
    db := &DB{
        connector: c,
        openerCh:  make(chan struct{}, connectionRequestQueueSize),
        stop:      cancel,
    }
    go db.connectionOpener(ctx)
    return db
}
```

### Why

A dedicated background goroutine processes work from a buffered channel. It's controlled by a context — calling `cancel()` (stored as `db.stop`) shuts it down. This is the standard "long-lived worker goroutine with graceful shutdown" pattern.

### Anti-pattern

```go
// DON'T: Start goroutines without shutdown mechanism
go func() {
    for {
        processWork()  // runs forever, no way to stop
    }
}()
```

---

## 11. noCopy — Preventing Value Copies

### Source: `src/sync/cond.go:120-126`

```go
// src/sync/cond.go:120-126
type noCopy struct{}

// Lock is a no-op used by -copylocks checker from `go vet`.
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
```

### Why

Embedding `noCopy` makes `go vet` report errors when the struct is copied by value. All sync primitives use this because copying a locked mutex or active WaitGroup is always a bug.

### Anti-pattern

```go
// DON'T: Pass sync types by value
func doWork(wg sync.WaitGroup) {  // copies!
    defer wg.Done()  // operates on copy, not original
}

// DO: Pass by pointer
func doWork(wg *sync.WaitGroup) {
    defer wg.Done()
}
```

---

## Summary: Concurrency Decision Guide

| Need | Use |
|------|-----|
| Protect shared state | `sync.Mutex` + `defer Unlock()` |
| One-time initialization | `sync.Once` |
| Wait for N goroutines | `sync.WaitGroup` (prefer `.Go()` in 1.25+) |
| Reduce allocation pressure | `sync.Pool` (not for connections!) |
| Signal completion/cancellation | `chan struct{}` + `close()` |
| Deadline/timeout propagation | `context.WithTimeout` / `context.WithCancel` |
| Backpressure between producer/consumer | Unbuffered channels |
| Long-lived background worker | Goroutine + context cancellation |
| Prevent struct copying | Embed `noCopy` field |
