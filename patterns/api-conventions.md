# API Conventions in the Go Standard Library

## 1. The Must Pattern

**Pattern name:** MustXxx (Panic on Error)

**Source citation:** `regexp/regexp.go` lines 310–320, `text/template/helper.go` lines 19–30

**What it does:** A function wraps a fallible constructor and panics if the error
is non-nil. Named `MustXxx` or `Must` (when wrapping a generic `(T, error)` pair).

**Why:** Safe initialization of package-level variables at program startup. Since
`var` initializers can't handle errors, `Must` converts programmer errors (bad
regex literals, bad templates) into immediate panics that surface during init.

**When to Use**

**Triggers:**
- You're initializing a package-level variable with a value that is known at compile time (regex, template, URL)
- Failure means a programmer bug, not a runtime condition (the regex literal is wrong, not user input)
- `var` initialization can't handle the `(T, error)` return

**Example — before:**
```go
var emailRegex *regexp.Regexp

func init() {
    var err error
    emailRegex, err = regexp.Compile(`^[a-z]+@[a-z]+\.[a-z]+$`)
    if err != nil {
        panic(err) // manual panic, verbose
    }
}
```

**Example — after:**
```go
var emailRegex = regexp.MustCompile(`^[a-z]+@[a-z]+\.[a-z]+$`)
// One line. Panics on typo (caught immediately in tests). Clean.
```

**Anti-pattern:** Using Must in runtime code where the input is dynamic/user-provided;
panicking on recoverable errors; naming it something other than Must (e.g., `PanicOnError`).

**Code examples from source:**

```go
// regexp/regexp.go:310-320
// MustCompile is like [Compile] but panics if the expression cannot be parsed.
// It simplifies safe initialization of global variables holding compiled regular
// expressions.
func MustCompile(str string) *Regexp {
    regexp, err := Compile(str)
    if err != nil {
        panic(`regexp: Compile(` + quote(str) + `): ` + err.Error())
    }
    return regexp
}
```

```go
// text/template/helper.go:19-30
// Must is a helper that wraps a call to a function returning ([*Template], error)
// and panics if the error is non-nil. It is intended for use in variable
// initializations such as
//
//     var t = template.Must(template.New("name").Parse("text"))
func Must(t *Template, err error) *Template {
    if err != nil {
        panic(err)
    }
    return t
}
```

---

## 2. Compile / MustCompile Pair

**Pattern name:** Fallible Constructor + Must Wrapper

**Source citation:** `regexp/regexp.go` lines 130–131, 310–320

**What it does:** The real constructor returns `(*T, error)`. A parallel `Must` variant
wraps it for use in global variable initialization.

**Why:** Separates concerns: `Compile` is for runtime use where errors are handled;
`MustCompile` is for compile-time-known values where failure is a programming bug.

**Anti-pattern:** Only providing the Must variant (no way to handle errors gracefully);
only providing the error variant (verbose for package-level vars).

**Code example from source:**

```go
// regexp/regexp.go:130-131
func Compile(expr string) (*Regexp, error) {
    return compile(expr, syntax.Perl, false)
}

// regexp/regexp.go:310-315
func MustCompile(str string) *Regexp {
    regexp, err := Compile(str)
    if err != nil {
        panic(`regexp: Compile(` + quote(str) + `): ` + err.Error())
    }
    return regexp
}
```

---

## 3. XxxWithContext Variant

**Pattern name:** WithContext Function Overload

**Source citation:** `net/http/request.go` lines 867–869, 894–930

**What it does:** Provides two function variants: `NewRequest` (uses `context.Background()`)
and `NewRequestWithContext` (accepts an explicit context). The simple version delegates
to the context-aware one.

**Why:** Context was added after the original API was established. The `WithContext`
variant enables cancellation and deadlines; the plain variant preserves backward
compatibility and ergonomics for the common case.

**Anti-pattern:** Breaking the existing API signature; always requiring context even
for fire-and-forget uses; naming it `NewRequestCtx`.

**Code example from source:**

```go
// net/http/request.go:867-869
func NewRequest(method, url string, body io.Reader) (*Request, error) {
    return NewRequestWithContext(context.Background(), method, url, body)
}

// net/http/request.go:894+
func NewRequestWithContext(ctx context.Context, method, url string, body io.Reader) (*Request, error) {
    // full implementation...
}
```

---

## 4. Nil-Opts Convention (Optional Config Pointer)

**Pattern name:** `*Options` Parameter — Nil Means Defaults

**Source citation:** `log/slog/text_handler.go` lines 28–42, `log/slog/handler.go` lines 135–175

**What it does:** A constructor accepts a pointer to an options struct. If the pointer
is nil, all defaults apply. The constructor internally substitutes a zero-value struct.

**Why:** Keeps the simple case clean (`NewTextHandler(os.Stderr, nil)`) while allowing
full customization. The pointer type signals "this entire argument is optional."

**Anti-pattern:** Requiring a non-nil options struct even with zero customization;
using variadic functional options when a simple struct suffices.

**Code example from source:**

```go
// log/slog/text_handler.go:28-42
// NewTextHandler creates a [TextHandler] that writes to w,
// using the given options.
// If opts is nil, the default options are used.
func NewTextHandler(w io.Writer, opts *HandlerOptions) *TextHandler {
    if opts == nil {
        opts = &HandlerOptions{}
    }
    return &TextHandler{
        &commonHandler{
            json: false,
            w:    w,
            opts: *opts,
            mu:   &sync.Mutex{},
        },
    }
}
```

---

## 5. Builder Pattern (Accumulate + Finalize)

**Pattern name:** Builder (Write Methods + String/Bytes Finalizer)

**Source citation:** `strings/builder.go` lines 14–113

**What it does:** A zero-value struct accumulates data via Write/WriteByte/WriteString
methods, then produces a final result via String(). The builder is not reusable after
a copyCheck-protected modification.

**Why:** Avoids repeated string concatenation (O(n²) allocations). The zero value is
ready to use. Implements `io.Writer` so it integrates with `fmt.Fprintf`, etc.

**Anti-pattern:** Allocating on every append; requiring explicit initialization;
not implementing standard interfaces (`io.Writer`).

**Code example from source:**

```go
// strings/builder.go:14-16
// A Builder is used to efficiently build a string using [Builder.Write] methods.
// It minimizes memory copying. The zero value is ready to use.
// Do not copy a non-zero Builder.
type Builder struct {
    addr *Builder
    buf  []byte
}

// strings/builder.go:92-96
func (b *Builder) WriteString(s string) (int, error) {
    b.copyCheck()
    b.buf = append(b.buf, s...)
    return len(s), nil
}

// strings/builder.go:48-50
func (b *Builder) String() string {
    return unsafe.String(unsafe.SliceData(b.buf), len(b.buf))
}
```

---

## 6. Layered API (Convenience → Full Control)

**Pattern name:** Convenience Wrappers over Configurable Core

**Source citation:** `os/file.go` lines 385–415

**What it does:** Simple functions (`Open`, `Create`) delegate to the fully configurable
`OpenFile` with pre-set flags. Users choose their level of control.

**Why:** 90% of file opens are reads or creates. Layered APIs serve the common case
without hiding power. The naming makes intent clear.

**When to Use**

**Triggers:**
- 90% of callers need the simple case (open for read, create and truncate)
- You have a powerful function with many flags/options but most combinations are rare
- You find yourself writing the same flag combination repeatedly in calling code

**Example — before:**
```go
// User must know about flags for every file open
f, err := os.OpenFile("data.json", os.O_RDONLY, 0)
// Every. Single. Time.
```

**Example — after:**
```go
// Simple case:
f, err := os.Open("data.json") // just reads — no flags to remember

// Power case (when you actually need it):
f, err := os.OpenFile("data.json", os.O_RDWR|os.O_CREATE|os.O_APPEND, 0644)
```

**Anti-pattern:** Only exposing the full-power version; making users learn flag
constants for simple reads; duplicating implementation across convenience functions.

**Code example from source:**

```go
// os/file.go:389-393
// Open opens the named file for reading.
func Open(name string) (*File, error) {
    return OpenFile(name, O_RDONLY, 0)
}

// os/file.go:399-403
// Create creates or truncates the named file.
func Create(name string) (*File, error) {
    return OpenFile(name, O_RDWR|O_CREATE|O_TRUNC, 0666)
}

// os/file.go:410+ (the general form)
func OpenFile(name string, flag int, perm FileMode) (*File, error) {
    // ...
}
```

---

## 7. Package-Level Functions Delegating to DefaultXxx

**Pattern name:** Convenience Package Functions

**Source citation:** `net/http/client.go` line 109, implied by `http.Get`, `http.Post`

**What it does:** Top-level functions like `http.Get(url)` call methods on the
`DefaultClient`. Users can bypass by creating their own `Client`.

**Why:** Makes the simple case trivial (one-liner HTTP requests). No import of
constructors or setup needed. The package "just works" for basic usage.

**Anti-pattern:** Not providing convenience functions (forcing explicit construction
even for prototyping); making the default's behavior non-obvious.

**Code example from source:**

```go
// net/http/client.go:109
var DefaultClient = &Client{}

// net/http/client.go (implied pattern):
// func Get(url string) (resp *Response, err error) {
//     return DefaultClient.Get(url)
// }
```

---

## 8. Register Pattern (Pluggable Algorithms)

**Pattern name:** RegisterXxx for Side-Effect Imports

**Source citation:** `crypto/crypto.go` lines 145–150

**What it does:** A `RegisterHash(h Hash, f func() hash.Hash)` function allows
algorithm implementations in sub-packages to register themselves via `init()`.
The main package dispatches based on the registered factories.

**Why:** Decouples the algorithm registry from specific implementations. Users import
only the algorithms they need (e.g., `_ "crypto/sha256"`). Reduces binary size and
avoids circular dependencies.

**Anti-pattern:** Hard-coding all implementations; requiring explicit constructor calls
for each algorithm; using global mutable state without clear ownership.

**Code example from source:**

```go
// crypto/crypto.go:145-150
func RegisterHash(h Hash, f func() hash.Hash) {
    if h == 0 || h >= maxHash {
        panic("crypto: RegisterHash of unknown hash function")
    }
    hashes[h] = f
}
```

---

## 9. Graceful Shutdown Pattern

**Pattern name:** Close vs Shutdown (Immediate vs Graceful)

**Source citation:** `net/http/server.go` lines 3171–3220 (Close), 3221+ (Shutdown)

**What it does:** Provides both `Close()` (immediate, forceful) and `Shutdown(ctx)`
(graceful, waits for in-flight requests). The context on Shutdown provides a
timeout mechanism.

**Why:** Different operational scenarios need different termination semantics.
Graceful shutdown is critical for production services; immediate close is needed for
tests and emergency stops.

**When to Use**

**Triggers:**
- Your type manages long-lived connections or in-flight requests
- You need both "stop now" (tests, emergencies) and "drain gracefully" (deploys, SIGTERM)
- A `Close()` that waits forever would make tests hang

**Example — before:**
```go
type Server struct { listener net.Listener }

func (s *Server) Stop() {
    s.listener.Close() // all in-flight requests get connection reset — data loss
}
```

**Example — after:**
```go
type Server struct {
    listener net.Listener
    active   sync.WaitGroup
}

// Immediate: drop everything
func (s *Server) Close() error {
    return s.listener.Close()
}

// Graceful: stop accepting, wait for in-flight with timeout
func (s *Server) Shutdown(ctx context.Context) error {
    s.listener.Close()     // stop accepting new connections
    done := make(chan struct{})
    go func() { s.active.Wait(); close(done) }()
    select {
    case <-done:
        return nil          // all requests finished
    case <-ctx.Done():
        return ctx.Err()   // timed out — caller decides what to do
    }
}
```

**Anti-pattern:** Only providing one shutdown mode; not accepting a context for
timeout control; leaking goroutines on shutdown.

**Code example from source:**

```go
// net/http/server.go:3171-3175
func (s *Server) Close() error {
    s.inShutdown.Store(true)
    s.mu.Lock()
    defer s.mu.Unlock()
    err := s.closeListenersLocked()
    // ... forcefully closes all active connections
}

// net/http/server.go:3221+
// Shutdown gracefully shuts down the server without interrupting any
// active connections.
func (s *Server) Shutdown(ctx context.Context) error {
    s.inShutdown.Store(true)
    // ... closes listeners, waits for idle, respects ctx deadline
}
```

---

## 10. Channel-Based Timer/Ticker API

**Pattern name:** NewXxx Returning Channel-Bearing Struct

**Source citation:** `time/tick.go` lines 16–45, `time/sleep.go` lines 89–155

**What it does:** `NewTicker(d)` and `NewTimer(d)` return structs with a `C <-chan Time`
field. Consumers select on the channel to receive time events.

**Why:** Integrates time-based events with Go's concurrency primitives (select).
The channel-based API composes naturally with other goroutine patterns.

**Anti-pattern:** Callback-based timer APIs that don't compose with select; exposing
the send side of the channel; not documenting goroutine safety.

**Code example from source:**

```go
// time/tick.go:16-18
type Ticker struct {
    C          <-chan Time // The channel on which the ticks are delivered.
    initTicker bool
}

// time/tick.go:36-45
func NewTicker(d Duration) *Ticker {
    if d <= 0 {
        panic("non-positive interval for NewTicker")
    }
    c := make(chan Time, 1)
    t := (*Ticker)(unsafe.Pointer(newTimer(when(d), int64(d), sendTime, c, syncTimer(c))))
    t.C = c
    return t
}
```
