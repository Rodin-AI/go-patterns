# Go Interface Design Patterns

Patterns extracted from the Go standard library source code.

---

## 1. Small Interfaces (1-2 Methods)

### Source: `src/io/io.go:80-92` (Reader), `93-103` (Writer), `105-109` (Closer)

Go's most powerful interfaces have exactly **one method**:

```go
// src/io/io.go:80-92
type Reader interface {
    Read(p []byte) (n int, err error)
}

// src/io/io.go:93-103
type Writer interface {
    Write(p []byte) (n int, err error)
}

// src/io/io.go:105-109
type Closer interface {
    Close() error
}
```

### Why

Small interfaces are easy to implement and easy to compose. Any type can satisfy `io.Reader` by implementing a single method. This maximizes the number of types that can participate in the ecosystem — files, network connections, buffers, compressors, encryptors all satisfy `Reader`.

### When to Use

**Triggers:**
- You're defining a function that only needs ONE capability from its argument (reading, writing, closing)
- You want maximum reusability — many different types should be able to satisfy your requirement
- You're tempted to create a big interface but realize most consumers only use 1-2 methods

**Example — before:**
```go
// Accepts only *os.File — can't use with buffers, HTTP bodies, test mocks
func countLines(f *os.File) (int, error) {
    scanner := bufio.NewScanner(f)
    count := 0
    for scanner.Scan() { count++ }
    return count, scanner.Err()
}
```

**Example — after:**
```go
// Accepts io.Reader — works with files, HTTP bodies, strings.NewReader, gzip.Reader, etc.
func countLines(r io.Reader) (int, error) {
    scanner := bufio.NewScanner(r)
    count := 0
    for scanner.Scan() { count++ }
    return count, scanner.Err()
}
```

### Anti-pattern

```go
// DON'T: Giant interface that tries to cover everything
type FileSystem interface {
    Open(name string) (File, error)
    Create(name string) (File, error)
    Remove(name string) error
    Stat(name string) (FileInfo, error)
    ReadDir(name string) ([]DirEntry, error)
    MkdirAll(path string, perm FileMode) error
    // ... 10 more methods
}
```

Large interfaces are hard to implement, hard to mock, and couple consumers to capabilities they don't need.

---

## 2. Interface Composition

### Source: `src/io/io.go:131-155`

Compose small interfaces into larger ones only when needed:

```go
// src/io/io.go:131-134
type ReadWriter interface {
    Reader
    Writer
}

// src/io/io.go:136-139
type ReadCloser interface {
    Reader
    Closer
}

// src/io/io.go:141-144
type WriteCloser interface {
    Writer
    Closer
}

// src/io/io.go:146-150
type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

### Why

Composition lets you express precisely what you need. A function that only reads should accept `Reader`, not `ReadWriteCloser`. Callers provide the minimum; composers build up.

### Anti-pattern

```go
// DON'T: Define a big interface, then use only part of it
func processData(rw ReadWriteCloser) {
    // only calls Read... why require Write and Close?
}
```

---

## 3. Accept Interfaces, Return Structs

### Source: `src/io/io.go:461` (LimitReader), `src/io/io.go:618` (TeeReader)

```go
// src/io/io.go:461
func LimitReader(r Reader, n int64) Reader { return &LimitedReader{r, n} }

// src/io/io.go:467-471
type LimitedReader struct {
    R Reader // underlying reader
    N int64  // max bytes remaining
}
```

```go
// src/io/io.go:618-620
func TeeReader(r Reader, w Writer) Reader {
    return &teeReader{r, w}
}
```

### Why

- **Accept interfaces**: Maximizes what callers can pass in — any `Reader` works.
- **Return structs** (or concrete types): Gives callers access to the full type, including fields and methods not in any interface. `LimitedReader` exposes `R` and `N` publicly so callers can inspect remaining bytes.

The return type of `LimitReader` is `Reader` (interface), but the underlying value is `*LimitedReader` (struct). Functions like `io.Copy` can type-assert to `*LimitedReader` to optimize buffer sizes (line 425).

### When to Use

**Triggers:**
- You're writing a function/constructor that operates on a capability (reading, hashing, connecting)
- Your return type has useful fields or methods beyond the interface contract
- You want callers to pass anything that satisfies the contract, but return something concrete they can inspect

**Example — before:**
```go
// Too restrictive input, too vague output
func NewLogger(f *os.File) io.Writer {
    return &logger{out: f, level: "info"} // hides SetLevel, Flush methods
}
```

**Example — after:**
```go
type Logger struct {
    out   io.Writer
    level string
}

func (l *Logger) SetLevel(lvl string) { l.level = lvl }
func (l *Logger) Flush() error { /* ... */ }

// Accept interface (any io.Writer), return struct (full access)
func NewLogger(w io.Writer) *Logger {
    return &Logger{out: w, level: "info"}
}
```

### Anti-pattern

```go
// DON'T: Accept concrete types
func LimitReader(r *os.File, n int64) Reader  // too restrictive

// DON'T: Return interfaces when you have useful struct fields
func NewServer() ServerInterface  // hides useful config fields
```

---

## 4. Interface Satisfaction as a Compile-Time Check

### Source: `src/io/io.go:645`, `src/net/http/server.go:4071`

```go
// src/io/io.go:645
var _ ReaderFrom = discard{}

// src/net/http/server.go:4071
var _ Pusher = (*timeoutWriter)(nil)
```

### Why

These blank-identifier declarations cause a compile error if the type stops implementing the interface. They're documentation and safety net combined — no runtime cost.

### Anti-pattern

```go
// DON'T: Discover interface failures at runtime
func doSomething(w ResponseWriter) {
    pusher := w.(Pusher) // panics if w doesn't implement Pusher
}
```

---

## 5. Interface-Based Polymorphism (sort.Interface)

### Source: `src/sort/sort.go:16-41`

```go
// src/sort/sort.go:16-41
type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}
```

### Why

The sort package defines *behavior* it needs, not data it operates on. Any collection — slices, linked lists, database result sets — can be sorted by implementing three methods. The algorithm is decoupled from the data structure.

### Anti-pattern

```go
// DON'T: Hardcode to a specific data type
func Sort(data []int)          // only works for []int
func Sort(data []interface{})  // loses type safety
```

Note: Since Go 1.21, `slices.SortFunc` is preferred for slices (generic + faster), but `sort.Interface` remains the pattern for non-slice collections.

---

## 6. The Adapter Pattern (HandlerFunc)

### Source: `src/net/http/server.go:2334-2342`

```go
// src/net/http/server.go:2334-2338
// The HandlerFunc type is an adapter to allow the use of
// ordinary functions as HTTP handlers. If f is a function
// with the appropriate signature, HandlerFunc(f) is a
// Handler that calls f.
type HandlerFunc func(ResponseWriter, *Request)

// src/net/http/server.go:2341-2342
// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```

### Why

This bridges functions and interfaces. Any function with the right signature becomes a `Handler` via `HandlerFunc(myFunc)`. You get the simplicity of functions with the composability of interfaces.

### When to Use

**Triggers:**
- You have an interface with a single method and users frequently implement it with a bare function
- You want to accept both struct-based and function-based implementations of the same behavior
- Requiring a struct definition for simple cases feels like boilerplate

**Example — before:**
```go
type Processor interface {
    Process(data []byte) error
}

// User must create a whole struct just to use a function
type upperProcessor struct{}
func (u upperProcessor) Process(data []byte) error {
    fmt.Println(strings.ToUpper(string(data)))
    return nil
}
```

**Example — after:**
```go
type Processor interface {
    Process(data []byte) error
}

// Adapter: any function with the right signature becomes a Processor
type ProcessorFunc func([]byte) error

func (f ProcessorFunc) Process(data []byte) error { return f(data) }

// Now users can write:
pipeline.Use(ProcessorFunc(func(data []byte) error {
    fmt.Println(strings.ToUpper(string(data)))
    return nil
}))
```

### Anti-pattern

```go
// DON'T: Force users to create a struct just to implement an interface
type myHandler struct{}
func (h myHandler) ServeHTTP(w ResponseWriter, r *Request) {
    // just calls a function anyway
    handleHome(w, r)
}
```

---

## 7. Optional Interfaces (Runtime Feature Detection)

### Source: `src/net/http/server.go:165-175` (Flusher), `src/net/http/server.go:183-206` (Hijacker)

```go
// src/net/http/server.go:165-170
type Flusher interface {
    Flush()
}

// src/net/http/server.go:183-206
type Hijacker interface {
    Hijack() (net.Conn, *bufio.ReadWriter, error)
}
```

Usage pattern (from doc comments):
```go
// Handlers should always test for this ability at runtime.
if flusher, ok := w.(Flusher); ok {
    flusher.Flush()
}
```

### Why

Not every `ResponseWriter` supports flushing or hijacking (HTTP/2 doesn't support Hijacker). Instead of bloating the main interface, optional capabilities are separate interfaces checked at runtime. This keeps the core interface small while allowing progressive enhancement.

### When to Use

**Triggers:**
- Some implementations support a capability but others don't (flushing, hijacking, seeking)
- You want to keep the core interface small but allow optimizations when available
- You're writing middleware that should enhance behavior when possible, not require it

**Example — before:**
```go
// Forces ALL stores to implement caching, even simple ones
type Store interface {
    Get(key string) ([]byte, error)
    Set(key string, val []byte) error
    InvalidateCache() error  // not all stores have a cache!
}
```

**Example — after:**
```go
type Store interface {
    Get(key string) ([]byte, error)
    Set(key string, val []byte) error
}

// Optional capability — check at runtime
type Cacheable interface {
    InvalidateCache() error
}

func refreshAll(s Store) {
    if c, ok := s.(Cacheable); ok {
        c.InvalidateCache() // only if supported
    }
}
```

### Anti-pattern

```go
// DON'T: Put optional capabilities in the main interface
type ResponseWriter interface {
    Write([]byte) (int, error)
    WriteHeader(int)
    Header() Header
    Flush()                                    // not always supported!
    Hijack() (net.Conn, *bufio.ReadWriter, error) // not always supported!
}
```

---

## 8. The Stringer Interface (Convention-Based Behavior)

### Source: `src/fmt/print.go:63-66`

```go
// src/fmt/print.go:63-66
type Stringer interface {
    String() string
}
```

### Why

`fmt` checks if a value implements `Stringer` and calls `String()` for its text representation. This is a *convention*: any type in any package can participate without importing `fmt`. The interface acts as a protocol.

Similarly, `GoStringer` (line 71) controls `%#v` output, and `Formatter` (line 58-61) gives total control over formatting.

### Anti-pattern

```go
// DON'T: Use type switches over known types
func printThing(v any) string {
    switch v := v.(type) {
    case MyType: return v.name
    case OtherType: return v.label
    // ... must know about every type
    }
}
```

---

## 9. Interface Upgrade Pattern (WriterTo/ReaderFrom in io.Copy)

### Source: `src/io/io.go:410-417`

```go
// src/io/io.go:410-417
func copyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
    // If the reader has a WriteTo method, use it to do the copy.
    // Avoids an allocation and a copy.
    if wt, ok := src.(WriterTo); ok {
        return wt.WriteTo(dst)
    }
    // Similarly, if the writer has a ReadFrom method, use it to do the copy.
    if rf, ok := dst.(ReaderFrom); ok {
        return rf.ReadFrom(src)
    }
    // ... fallback to buffer-based copy
}
```

### Why

The core function works with the minimum interface (`Reader`/`Writer`) but *upgrades* behavior when richer interfaces are available. `*os.File` implements `ReadFrom` using `sendfile(2)` — zero-copy kernel transfer. The generic path works, but specialized implementations optimize transparently.

### Anti-pattern

```go
// DON'T: Always use the slow generic path
func Copy(dst Writer, src Reader) {
    buf := make([]byte, 32*1024)
    // always allocates, never leverages kernel optimizations
}
```

---

## 10. The driver.Driver Pattern (Plugin Interfaces)

### Source: `src/database/sql/driver/driver.go:85-97`, `104-112`

```go
// src/database/sql/driver/driver.go:85-97
type Driver interface {
    Open(name string) (Conn, error)
}

// src/database/sql/driver/driver.go:104-112
type DriverContext interface {
    OpenConnector(name string) (Connector, error)
}
```

### Why

The `database/sql` package defines interfaces that driver authors implement. The base interface (`Driver`) is minimal (one method). Extended capabilities (`DriverContext`, `Pinger`, `Execer`, `Queryer`) are opt-in — the `sql` package checks for them at runtime. This allows the ecosystem to grow without breaking existing drivers.

### Anti-pattern

```go
// DON'T: One massive interface all drivers must implement
type Driver interface {
    Open(name string) (Conn, error)
    OpenConnector(name string) (Connector, error)
    Ping(ctx context.Context) error
    // ... every driver must implement everything, even if not supported
}
```

---

## Summary: Go Interface Design Principles

| Principle | Standard Library Example |
|-----------|------------------------|
| Keep interfaces small (1-2 methods) | `io.Reader`, `io.Writer`, `fmt.Stringer` |
| Compose small interfaces | `io.ReadWriteCloser` |
| Accept interfaces, return structs | `io.LimitReader`, `io.TeeReader` |
| Use adapter types for functions | `http.HandlerFunc` |
| Optional capabilities via separate interfaces | `http.Flusher`, `http.Hijacker` |
| Compile-time interface checks | `var _ Interface = (*Type)(nil)` |
| Runtime interface upgrade for optimization | `io.Copy` → `WriterTo`/`ReaderFrom` |
| Plugin/driver interfaces start minimal | `database/sql/driver.Driver` |
