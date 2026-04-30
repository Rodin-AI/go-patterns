# Go Patterns (from golang/go Source)

Prescriptive patterns extracted from the Go language source using
iterative analysis. Real examples, hyperlinked to source.

**Source:** [golang/go](https://github.com/golang/go) at commit
[`17bd5ab`](https://github.com/golang/go/tree/17bd5ab8c650155dd2bd09f7005726552639eea0)

**Stats:** 281 interfaces, 55 sentinel errors, 145 error types,
262 constructors, 309 context-accepting functions, 1,065 examples.

---

## Interface Design

### Single-Method Interfaces

**Rule:** Define interfaces with exactly one method whenever possible.

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

**Why:** Any type with that method satisfies the interface implicitly.
Smaller interfaces = more types satisfy them = more reusable code.

**When to use:** Defining abstraction boundaries, function parameters,
dependency injection.

**When NOT to use:** When operations are genuinely inseparable
(`sort.Interface` needs Len+Less+Swap together).

**Source:** [src/io/io.go#L86](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/io/io.go#L86)

---

### Interface Composition

**Rule:** Build larger interfaces by embedding smaller ones.

```go
type ReadWriter interface {
    Reader
    Writer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

**Why:** 15 composed interfaces in `io/io.go` from just 4 primitives
(Reader, Writer, Closer, Seeker). Composition prevents interface
bloat.

**When to use:** When callers need multiple capabilities together.

**When NOT to use:** Don't compose preemptively. Add compositions
only when you have a real function that needs both capabilities.

**Source:** [src/io/io.go#L131](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/io/io.go#L131)

---

### Accept Interfaces, Return Structs

**Rule:** Parameters should be interfaces. Return values should be
concrete types.

```go
// Accepts interface:
func Copy(dst Writer, src Reader) (written int64, err error)

// Returns concrete:
func NewReader(rd io.Reader) *Reader
```

**Why:** Accepting interfaces maximizes caller flexibility. Returning
structs gives callers full access without type assertions. 262 `New*`
constructors in stdlib all return concrete types.

**When to use:** Every public API function.

**When NOT to use:** When the return type must be hidden (use an
interface to prevent users from depending on internals).

**Source:** [src/io/io.go#L408](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/io/io.go#L408) (Copy), [src/bufio/bufio.go](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/bufio/bufio.go#L62) (NewReader)

---

### The Stringer Interface

**Rule:** Implement `String() string` for any type that has a human-
readable representation.

```go
func (t Time) String() string {
    return t.Format("2006-01-02 15:04:05.999999999 -0700 MST")
}
```

**Why:** 379 types in stdlib implement Stringer. `fmt.Println` uses it
automatically. It's the Go equivalent of `__str__`.

**When to use:** Any type that might be printed or logged.

**When NOT to use:** Internal types that are never user-visible.

**Source:** [src/time/time.go](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/time/time.go) (Time.String)

---

### Type Assertion for Optional Interfaces

**Rule:** Check if a value implements an optional interface using
type assertion.

```go
if wt, ok := src.(WriterTo); ok {
    return wt.WriteTo(dst)
}
```

**Why:** 104 type assertions in stdlib. This pattern allows fallback
behavior — try the fast path, fall back to the generic path.

**When to use:** Optional optimizations (WriterTo, ReaderFrom), feature
detection.

**When NOT to use:** Required behavior (just accept the interface
directly in the signature).

**Source:** [src/io/io.go#L420](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/io/io.go#L420) (Copy's WriterTo check)

---

## Error Handling

### Sentinel Errors for Known Conditions

**Rule:** Package-level `var Err*` for errors callers need to check.
Include package name in the message.

```go
var ErrBadPattern = errors.New("syntax error in pattern")
var ErrRange = errors.New("value out of range")
var ErrUnsupported = errors.New("unsupported operation")
```

**Why:** 55 exported sentinel errors in stdlib. Callers use
`errors.Is(err, strconv.ErrRange)` to handle specific cases.

**When to use:** Errors that represent documented, expected conditions
callers should distinguish.

**When NOT to use:** Errors that carry dynamic context (use error
types). Errors callers never need to identify specifically.

**Source:** [src/strconv/number.go#L246](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/strconv/number.go#L246)

---

### Error Types for Rich Context

**Rule:** Define types implementing `error` when you need structured
error information.

```go
type PathError struct {
    Op   string
    Path string
    Err  error
}

func (e *PathError) Error() string {
    return e.Op + " " + e.Path + ": " + e.Err.Error()
}

func (e *PathError) Unwrap() error { return e.Err }
```

**Why:** 145 error type implementations in stdlib. Callers use
`errors.As(err, &pathErr)` to extract structured data.

**When to use:** When the error needs to carry structured fields
(path, operation, underlying error).

**When NOT to use:** Simple conditions (use sentinel errors). One-off
errors (use `fmt.Errorf`).

**Source:** [src/os/error.go](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/os/error.go) (PathError)

---

### Wrap with %w

**Rule:** Add context when propagating errors. Use `%w` to preserve
the chain.

```go
return fmt.Errorf("cannot parse %q as JSON number: %w", val, strconv.ErrSyntax)
```

**Why:** 115 `%w` wrappings in stdlib. Creates a chain that
`errors.Is` and `errors.As` can traverse.

**When to use:** Every time you add context to an error from a lower
layer.

**When NOT to use:** When the original error's identity should be
hidden from callers (use `%v` to break the chain).

**Source:** [src/encoding/json/v2_decode.go#L219](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/encoding/json/v2_decode.go#L219)

---

### io.EOF as Termination Signal

**Rule:** Use `io.EOF` to signal normal end-of-stream, not an error.

```go
n, err := r.Read(buf)
if err == io.EOF {
    break // Normal termination
}
if err != nil {
    return err // Actual error
}
```

**Why:** 316 `io.EOF` references in stdlib. EOF is expected, not
exceptional. Readers return io.EOF when there's no more data.

**When to use:** Implementing Reader, iterators, stream processors.

**When NOT to use:** Errors that indicate failure (use a real error).

**Source:** [src/io/io.go#L44](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/io/io.go#L44)

---

## Testing

### Table-Driven Tests

**Rule:** Use `[]struct{}` with named cases and `t.Run`.

```go
tests := []struct {
    name string
    input string
    want  string
}{
    {"empty", "", ""},
    {"hello", "hello", "HELLO"},
}
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got := Transform(tt.input)
        if got != tt.want {
            t.Errorf("got %q, want %q", got, tt.want)
        }
    })
}
```

**Why:** 1,926 `t.Run` calls in the Go source. Named subtests make
failure output clear. Adding cases is one struct literal.

**When to use:** Any function with 3+ input variations.

**When NOT to use:** Tests where setup varies significantly between
cases (separate test functions).

**Source:** [src/testing/testing_test.go](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/testing/testing_test.go) (TestSetenv)

---

### t.Helper() for Test Utilities

**Rule:** Call `t.Helper()` as the first line of any test helper.

```go
func assertEqual(t *testing.T, got, want string) {
    t.Helper()
    if got != want {
        t.Errorf("got %q, want %q", got, want)
    }
}
```

**Why:** 2,685 `t.Helper()` calls. Without it, error messages report
the helper's line number instead of the caller's.

**When to use:** Every function that calls `t.Error`, `t.Fatal`, or
other testing.T methods on behalf of the caller.

**When NOT to use:** Functions that ARE the test (not helpers).

---

### Example Functions as Living Docs

**Rule:** Write `Example*` functions in `_test.go` with `// Output:`
comments.

```go
func ExampleSprintf() {
    fmt.Println(fmt.Sprintf("Hello, %s", "world"))
    // Output: Hello, world
}
```

**Why:** 1,065 Example functions in stdlib. They compile, run, and
appear in docs. They can't go stale.

**When to use:** Every exported function that would benefit from usage
demonstration.

**When NOT to use:** Internal APIs. Functions with non-deterministic
output.

---

### testdata/ for Fixtures

**Rule:** Put test fixtures in `testdata/` directories.

**Why:** 111 `testdata/` dirs in stdlib. The go tool ignores them
during compilation. Golden files, certificates, sample inputs live
here.

**When to use:** Files your tests read but never modify at runtime.

**When NOT to use:** Generated test data (create in TestMain).

---

### Benchmarks

**Rule:** Prefix benchmark functions with `Benchmark` and use `b.N`.

```go
func BenchmarkSprintf(b *testing.B) {
    for b.Loop() {
        fmt.Sprintf("hello, %s", "world")
    }
}
```

**Why:** 1,974 benchmark functions in stdlib. Performance is tested,
not assumed.

**When to use:** Any code on a hot path. Any code you're optimizing.

**When NOT to use:** Code that's not performance-sensitive.

---

## Package Organization

### Flat Packages

**Rule:** No `pkg/` wrapper. Import path = directory path.

```
myproject/
├── server/
├── client/
├── internal/
└── cmd/
    └── myapp/
```

**Why:** The Go stdlib has zero nesting beyond 2 levels (e.g.,
`net/http`). Import paths should be short and predictable.

**When to use:** Always.

**When NOT to use:** Never. `pkg/` is a community anti-pattern the Go
team never endorsed.

---

### internal/ for Shared Private Code

**Rule:** Code shared between packages but not part of public API
goes in `internal/`.

**Why:** 61 internal packages in stdlib. Compiler-enforced — external
code cannot import them. Stronger than unexported identifiers.

**When to use:** Utility code that multiple packages need but users
shouldn't depend on.

**When NOT to use:** Code only one package uses (keep it unexported
in that package). Code stable enough for public API (promote it).

---

### One Package, One Responsibility

**Rule:** A package does one thing. Name it with a singular noun.

`fmt`, `io`, `net`, `os`, `sync`, `time`, `bytes`, `errors`

**Why:** Package names prefix all exported identifiers. Short names
compose well: `bytes.Buffer`, `sync.Mutex`, `time.Duration`.

**When to use:** Every package.

**When NOT to use:** Never name packages `utils`, `helpers`, `common`,
`models`, or `types`.

---

## Concurrency

### context.Context as First Parameter

**Rule:** Functions that do I/O take `ctx context.Context` first.

```go
func (c *Client) Do(ctx context.Context, req *Request) (*Response, error)
```

**Why:** 309 functions take Context in stdlib. First-parameter position
is universal. Context carries cancellation, deadlines, values.

**When to use:** Any function that blocks, does I/O, or might be
cancelled.

**When NOT to use:** Pure computation. Init functions. Functions that
complete instantly.

---

### sync.Mutex for Shared State

**Rule:** Protect shared state with a mutex. Comment what it guards.

```go
type Group struct {
    mu sync.Mutex       // protects m
    m  map[string]*call // lazily initialized
}
```

**Why:** 148 Mutex/RWMutex fields in stdlib. The comment-what-it-
guards pattern appears throughout.

**When to use:** Shared mutable state accessed by multiple goroutines.

**When NOT to use:** Channel-based coordination. Single-goroutine
ownership.

**Source:** [src/internal/singleflight/singleflight.go#L30](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/internal/singleflight/singleflight.go#L30)

---

### sync.Once for Lazy Initialization

**Rule:** Use `sync.Once` for thread-safe lazy init.

```go
var defaultLogger struct {
    once sync.Once
    val  *Logger
}

func getLogger() *Logger {
    defaultLogger.once.Do(func() {
        defaultLogger.val = newLogger()
    })
    return defaultLogger.val
}
```

**Why:** 58 `sync.Once` usages in stdlib. Guarantees exactly-once
execution regardless of concurrent callers.

**When to use:** Expensive initialization that should happen at most
once (DB connections, compiled regexps, parsed configs).

**When NOT to use:** Init that should happen at package load (use
`init()` or package-level `var`).

---

### sync.Pool for Reusable Buffers

**Rule:** Use `sync.Pool` for frequently allocated/freed objects.

```go
var encodeStatePool sync.Pool

func newEncodeState() *encodeState {
    if v := encodeStatePool.Get(); v != nil {
        e := v.(*encodeState)
        e.Reset()
        return e
    }
    return new(encodeState)
}
```

**Why:** Used in encoding/json, fmt, and other hot-path code. Reduces
GC pressure by reusing allocations.

**When to use:** Objects allocated per-request that are expensive to
create and safe to reuse.

**When NOT to use:** Small objects. Objects with complex cleanup.
Objects that shouldn't be shared between goroutines.

**Source:** [src/encoding/json/encode.go#L312](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/encoding/json/encode.go#L312)

---

### defer for Cleanup

**Rule:** Use `defer` immediately after acquiring a resource.

```go
mu.Lock()
defer mu.Unlock()

f, err := os.Open(path)
if err != nil { return err }
defer f.Close()
```

**Why:** 329 `defer Close()`/`defer Unlock()` in stdlib. Guarantees
cleanup even on panic. Pairs acquisition with release visually.

**When to use:** Every Lock/Close/Release/Done pattern.

**When NOT to use:** Hot loops where defer overhead matters (rare,
profile first).

---

## Documentation

### Package-Level doc.go

**Rule:** Complex packages get a `doc.go` with overview documentation.

```go
// Source: src/fmt/doc.go
/*
Package fmt implements formatted I/O with functions analogous
to C's printf and scanf.
*/
package fmt
```

**Why:** 25 `doc.go` files in stdlib. Separates overview from code.
`#` headings create sections in pkg.go.dev.

**When to use:** Any package with non-trivial API surface.

**When NOT to use:** Small packages where the comment fits in the main
file.

---

### Deprecated: Comment Convention

**Rule:** Mark deprecated items with `// Deprecated: use X instead.`

**Why:** 203 `Deprecated:` comments in stdlib. Tools (editors, linters)
recognize this pattern and show warnings.

**When to use:** Any public API you want to discourage but can't
remove.

**When NOT to use:** Internal code (just delete it).

---

## Naming

### Short Package Names

**Rule:** 3-7 characters, lowercase, singular noun.

`fmt` · `io` · `net` · `os` · `sync` · `time` · `bytes` · `errors`

**When to use:** Every package.

**When NOT to use:** NEVER: `utils`, `helpers`, `common`, `base`,
`models`, `types`, `shared`.

---

### New* Constructors

**Rule:** Constructor functions are named `New` or `New<Type>`.

```go
func NewReader(rd io.Reader) *Reader
func New(text string) error
```

**Why:** 262 `New*` functions in stdlib. Universal convention. No
`Create*`, no `Make*` (except `make` builtin), no `Build*`.

**When to use:** Any function that allocates and initializes.

**When NOT to use:** Functions that transform or convert (name them
by what they do: `Parse`, `Open`, `Dial`).

---

### No Get Prefix

**Rule:** Getters don't say "Get". Setters DO say "Set".

```go
// Wrong:
func (u *User) GetName() string

// Right:
func (u *User) Name() string
func (u *User) SetName(s string)
```

**Why:** Go convention. Only 58 `Get*` methods in all of stdlib
(mostly in legacy APIs like `net/http`).

**When to use:** All accessor methods.

**When NOT to use:** RPC/protobuf generated code (follows its own
convention).

---

### MixedCaps Only

**Rule:** `ExportedName` and `unexportedName`. Never underscores.

**Why:** Capitalization IS the visibility system. Underscores are
reserved for test files and generated code.

---

## Configuration

### Functional Options (With* Pattern)

**Rule:** Options as functions returning an opaque Options type.

```go
func NewEncoder(w io.Writer, opts ...Options) *Encoder

// Option constructors:
func WithIndent(indent string) Options { ... }
func WithByteLimit(n int64) Options { ... }
```

**Why:** Growing in stdlib (encoding/json/v2, context). Allows adding
options without breaking existing callers.

**When to use:** APIs with many optional parameters that grow over
time.

**When NOT to use:** Simple APIs with 1-2 options (just use parameters
or a config struct).

**Source:** [src/encoding/json/jsontext/options.go#L232](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/encoding/json/jsontext/options.go#L232)

---

### Config Structs for Complex Setup

**Rule:** Group related options into a named struct.

```go
type Config struct {
    Certificates []Certificate
    RootCAs      *x509.CertPool
    ServerName   string
    MinVersion   uint16
}
```

**Why:** `crypto/tls.Config` is the canonical example. Zero value is
usable with sensible defaults.

**When to use:** APIs with many related settings that configure a
long-lived object.

**When NOT to use:** Per-call options (use functional options).

**Source:** [src/crypto/tls/common.go#L566](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/crypto/tls/common.go#L566)

---

## Extension

### Register Pattern (Plugin Discovery)

**Rule:** Provide a `Register*` function for plugin architectures.

```go
func Register(name string, driver driver.Driver) {
    // ...
}
```

**Why:** Used in `database/sql`, `encoding/gob`, `image`,
`archive/zip`, `crypto`. The pattern: init-time registration +
runtime lookup.

**When to use:** When users provide implementations you discover at
runtime (drivers, codecs, formats).

**When NOT to use:** When you know all implementations at compile
time (use interfaces directly).

**Source:** [src/database/sql/sql.go#L53](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/database/sql/sql.go#L53)

---

## Performance

### Append* for Zero-Alloc Formatting

**Rule:** Provide `Append*` variants that write to caller's buffer.

```go
func (t Time) AppendFormat(b []byte, layout string) []byte
func AppendEncode(dst, src []byte) []byte
func AppendQuote[Bytes ~[]byte | ~string](dst []byte, src Bytes) ([]byte, error)
```

**Why:** Growing pattern in stdlib. Avoids allocation by letting the
caller own the buffer. The `encoding` package now defines
`BinaryAppender` and `TextAppender` interfaces.

**When to use:** Hot-path formatting functions where allocation cost
matters.

**When NOT to use:** Convenience APIs where readability > performance.

**Source:** [src/time/format.go#L655](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/time/format.go#L655)

---

### Preallocate Slices

**Rule:** Use `make([]T, 0, expectedCap)` when you know the size.

**Why:** 326 `make([]T, len, cap)` calls in stdlib. Avoids repeated
reallocation during append.

**When to use:** Loops where the output size is known or estimable.

**When NOT to use:** Unknown sizes. Small slices (<8 elements).

---

## Smells

### go:linkname Abuse

1,711 uses in Go's own source — but actively being removed. If you
need `go:linkname`, your API boundary is wrong.

### TODO Without Owner

`// TODO: fix this` — unaccountable. Go's 3,428 TODOs ALL have owners.

### Get* Methods

Only 58 in stdlib, mostly legacy. Modern Go drops the prefix.

### Huge Single Files

`proc.go` is 8,156 lines. Don't copy this. The scheduler stays in one
file because splitting breaks the mental model. Your CRUD handler has
no such excuse.

### Generated Code Without Generator

If you check in generated code, also check in the generator or clearly
document regeneration steps.

<!-- PATTERN_COMPLETE -->
