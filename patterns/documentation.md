# Documentation Patterns in the Go Standard Library


**Source:** [golang/go](https://github.com/golang/go) at commit [`17bd5ab`](https://github.com/golang/go/tree/17bd5ab8c650155dd2bd09f7005726552639eea0)
## 1. Package Documentation (doc.go or Package Comment)

**Pattern name:** Package Doc Comment

**Source citation:** [net/http/doc.go#L6](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/net/http/doc.go#L6), [os/file.go#L5](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/os/file.go#L5), [log/slog/doc.go#L6](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/log/slog/doc.go#L6)

**What it does:** The first file in a package (by convention `doc.go`, or the main
source file) starts with a `// Package xxx ...` comment that explains the package's
purpose, key types, and typical usage patterns.

**Why:** This is the first thing users see in `go doc <pkg>` and on pkg.go.dev. It
sets context, teaches the mental model, and provides copy-paste examples.

**Anti-pattern:** No package comment; package comment that just restates the package
name ("Package http provides http"); putting documentation in README instead of code.

**Code examples from source:**

```go
// net/http/doc.go:6-12
/*
Package http provides HTTP client and server implementations.

[Get], [Head], [Post], and [PostForm] make HTTP (or HTTPS) requests:

    resp, err := http.Get("http://example.com/")
    ...
*/
```

```go
// os/file.go:5-43
// Package os provides a platform-independent interface to operating system
// functionality. The design is Unix-like, although the error handling is
// Go-like; failing calls return values of type error rather than error numbers.
// Often, more information is available within the error. For example,
// if a call that takes a file name fails, such as [Open] or [Stat], the error
// will include the failing file name when printed and will be of type
// [*PathError], which may be unpacked for more information.
```

```go
// log/slog/doc.go:6-10
/*
Package slog provides structured logging,
in which log records include a message,
a severity level, and various other attributes
expressed as key-value pairs.
*/
```

---

## 2. Section Headers in Package Docs

**Pattern name:** `# Heading` in Doc Comments

**Source citation:** [os/file.go#L37](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/os/file.go#L37), `net/http/doc.go` (multiple sections)

**What it does:** Uses `# Section Name` within the package doc comment to organize
long documentation into navigable sections.

**Why:** Large packages need structure. Section headers render as links in pkg.go.dev
and provide a scannable table of contents.

**Anti-pattern:** Wall-of-text package docs; using `===` or `---` (not recognized);
too many sections (fragmenting simple docs).

**Code example from source:**

```go
// os/file.go:37
// # Concurrency
//
// The methods of [File] correspond to file system operations. All are
// safe for concurrent use.
```

---

## 3. Type/Function Comment Convention

**Pattern name:** `// TypeName verb...` or `// FuncName verb...`

**Source citation:** [net/http/server.go#L65](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/net/http/server.go#L65) (Handler), [bufio/scan.go#L14](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/bufio/scan.go#L14) (Scanner)

**What it does:** Every exported identifier's doc comment starts with the identifier
name, followed by a verb phrase describing what it does or represents.

**Why:** `go doc` extracts the first sentence as a summary. Starting with the name
ensures it reads correctly in both isolation (summary lists) and full context.
This is enforced by convention and checked by linters.

**Anti-pattern:** Starting with "This function..." or "The Foo type..."; starting
with articles ("A Handler is...") for functions (acceptable for types); omitting
the comment entirely.

**Code examples from source:**

```go
// net/http/server.go:65
// A Handler responds to an HTTP request.

// bufio/scan.go:14-17
// Scanner provides a convenient interface for reading data such as
// a file of newline-delimited lines of text.

// net/http/request.go:867
// NewRequest wraps NewRequestWithContext using context.Background.

// os/file.go:389-390
// Open opens the named file for reading.

// regexp/regexp.go:310-312
// MustCompile is like [Compile] but panics if the expression cannot be parsed.
// It simplifies safe initialization of global variables holding compiled regular
// expressions.
```

---

## 4. Doc Links (Square Bracket References)

**Pattern name:** `[TypeName]`, `[Package.Symbol]`, `[Method]` Links

**Source citation:** [net/http/server.go#L65](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/net/http/server.go#L65), [os/file.go#L9](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/os/file.go#L9)

**What it does:** Doc comments use `[SymbolName]` to create hyperlinks to other
identifiers. These render as clickable links on pkg.go.dev.

**Why:** Cross-references help users navigate the API. Links are concise and
don't clutter the plain-text rendering.

**Anti-pattern:** Using full URLs to godoc pages; not linking related types;
over-linking (every mention of every type).

**Code examples from source:**

```go
// net/http/server.go:65-70
// [Handler.ServeHTTP] should write reply headers and data to the [ResponseWriter]
// and then return. Returning signals that the request is finished; it
// is not valid to use the [ResponseWriter] or read from the
// [Request.Body] after or concurrently with the completion of the
// ServeHTTP call.

// os/file.go:9-11
// if a call that takes a file name fails, such as [Open] or [Stat], the error
// will include the failing file name when printed and will be of type
// [*PathError], which may be unpacked for more information.
```

---

## 5. Example Test Functions

**Pattern name:** `func ExampleXxx()` / `func ExampleType_Method()`

**Source citation:** [regexp/example_test.go#L13](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/regexp/example_test.go#L13), [net/http/example_handle_test.go#L16](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/net/http/example_handle_test.go#L16)

**What it does:** Functions named `Example`, `ExampleXxx`, or `ExampleType_Method`
in `_test.go` files serve as both executable tests and documentation. They include
an `// Output:` comment that `go test` verifies.

**Why:** Examples that compile, run, and are verified can never go stale. They appear
in `go doc` and pkg.go.dev alongside the relevant symbol. They teach by showing
real, working code.

**When to Use**

**Triggers:**
- You have a public function/type whose usage isn't obvious from the signature alone
- Your README examples have drifted from the actual API (broken examples in docs)
- You want examples that appear on pkg.go.dev AND are verified by `go test`

**Example — before:**
```go
// README.md (may be stale):
// ```go
// result := mylib.Process("input")
// fmt.Println(result.Data)
// ```
// ← compiles? who knows. API changed last month.
```

**Example — after:**
```go
// example_test.go
func ExampleProcess() {
    result := mylib.Process("input")
    fmt.Println(result.Data)
    // Output:
    // processed: input
}
// ← go test verifies this compiles and produces the expected output
```

**When NOT to Use**

**Don't write Example tests when:**
- The function signature is self-explanatory (`func Max(a, b int) int` doesn't need an example)
- The example would just restate the doc comment with no additional insight
- You're testing internal/unexported functions (examples must use the public API)
- The output is non-deterministic (timestamps, random values, goroutine ordering)

**Over-application example:**
```go
// Pointless — the signature tells you everything
func ExampleAbs() {
    fmt.Println(math.Abs(-5))
    // Output:
    // 5
}
```

**Better alternative:**
```go
// Skip the example for trivial functions. Write examples for non-obvious behavior:
func ExampleAbs_nan() {
    fmt.Println(math.Abs(math.NaN()))
    // Output:
    // NaN
}
// ← This teaches something surprising that the signature doesn't convey
```

**Why:** Examples exist to teach usage patterns that aren't obvious from the type signature and doc comment. Trivial examples add maintenance burden without teaching anything.

**Anti-pattern:** Examples that don't compile; examples without Output comments
(not verified); examples in README that drift from reality.

**Code examples from source:**

```go
// regexp/example_test.go:13-28
func Example() {
    // Compile the expression once, usually at init time.
    // Use raw strings to avoid having to quote the backslashes.
    var validID = regexp.MustCompile(`^[a-z]+\[[0-9]+\]$`)

    fmt.Println(validID.MatchString("adam[23]"))
    fmt.Println(validID.MatchString("eve[7]"))
    fmt.Println(validID.MatchString("Job[48]"))
    fmt.Println(validID.MatchString("snakey"))
    // Output:
    // true
    // true
    // false
    // false
}
```

```go
// net/http/example_handle_test.go:16-31
type countHandler struct {
    mu sync.Mutex // guards n
    n  int
}

func (h *countHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    h.mu.Lock()
    defer h.mu.Unlock()
    h.n++
    fmt.Fprintf(w, "count is %d\n", h.n)
}

func ExampleHandle() {
    http.Handle("/count", new(countHandler))
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## 6. Inline Code Examples in Doc Comments

**Pattern name:** Indented Code Blocks in Comments

**Source citation:** [os/file.go#L16](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/os/file.go#L16), [time/time.go#L928](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/time/time.go#L928)

**What it does:** Doc comments include indented code snippets (4 spaces) that render
as preformatted code blocks in godoc.

**Why:** Shows typical usage patterns directly in the doc comment without requiring
a separate Example test function. Good for short, illustrative snippets.

**Anti-pattern:** Non-indented code that doesn't render as code; examples too long
for inline (use Example functions instead); examples that reference unexported symbols.

**Code examples from source:**

```go
// os/file.go:16-21
// Here is a simple example, opening a file and reading some of it.
//
//     file, err := os.Open("file.go") // For read access.
//     if err != nil {
//         log.Fatal(err)
//     }

// time/time.go:928-933
// To count the number of units in a [Duration], divide:
//
//     second := time.Second
//     fmt.Print(int64(second/time.Millisecond)) // prints 1000
//
// To convert an integer number of units to a Duration, multiply:
//
//     seconds := 10
//     fmt.Print(time.Duration(seconds)*time.Second) // prints 10s
```

---

## 7. Deprecated Annotations

**Pattern name:** `// Deprecated: ...` in Doc Comments

**Source citation:** [net/http/server.go#L57](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/net/http/server.go#L57) (ErrWriteAfterFlush), [os/file.go#L93](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/os/file.go#L93)

**What it does:** A paragraph starting with `Deprecated:` marks an identifier as
deprecated and explains what to use instead.

**Why:** Recognized by tooling (go vet, staticcheck, IDEs). Provides a migration
path without breaking backward compatibility.

**When to Use**

**Triggers:**
- You have a better replacement for an existing function but can't remove the old one (semver)
- Users are still calling a function that has known issues or a superior alternative
- You want IDEs to show a strikethrough and linters to warn on usage

**Example — before:**
```go
// Just delete it? Breaks everyone's code.
// Leave it silently? Users never learn about the better way.
```

**Example — after:**
```go
// ParseDuration parses a duration string.
//
// Deprecated: Use [time.ParseDuration] instead, which handles
// all standard duration formats.
func ParseDuration(s string) (time.Duration, error) {
    return time.ParseDuration(s) // delegate to the replacement
}
```

**When NOT to Use**

**Don't deprecate when:**
- The function still works fine and there's no better replacement yet
- You're deprecating to "clean up" the API without a migration path (users will just ignore it)
- The replacement has a different behavior or isn't a drop-in substitute (document the difference instead)
- You're on v0.x and can just remove it (pre-1.0 allows breaking changes)

**Over-application example:**
```go
// Deprecated: Use NewClientV2 instead.
func NewClient(addr string) *Client { ... }

// But NewClientV2 has a completely different API:
func NewClientV2(cfg Config) (*ClientV2, error) { ... }
// Users can't just find-and-replace — this isn't a deprecation, it's a migration
```

**Better alternative:**
```go
// NewClient creates a client with default configuration.
// For advanced configuration (TLS, timeouts, connection pooling),
// use [NewClientWithConfig] instead.
func NewClient(addr string) *Client {
    return NewClientWithConfig(Config{Addr: addr})
}
```

**Why:** Deprecation means "there's a better way to do the same thing." If the replacement requires a fundamentally different approach, provide a migration guide — don't just slap `Deprecated:` on it and leave users stranded.

**Anti-pattern:** Removing deprecated APIs (breaks semver); deprecating without
suggesting an alternative; using non-standard deprecation markers.

**Code example from source:**

```go
// net/http/server.go:55-57
// Deprecated: ErrWriteAfterFlush is no longer returned by
// anything in the net/http package. Callers should not
// compare errors against this variable.
ErrWriteAfterFlush = errors.New("unused")
```

---

## 8. Error Documentation Convention

**Pattern name:** "If there is an error, it will be of type [*XxxError]"

**Source citation:** [os/file.go#L388](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/os/file.go#L388), 406

**What it does:** Functions document the concrete error type they return, enabling
callers to type-assert for additional context.

**Why:** Go's error handling relies on type assertions and `errors.Is/As`. Knowing
the concrete type lets callers extract structured information (path, operation,
underlying cause).

**Anti-pattern:** Returning opaque errors with no documented structure; returning
different error types from the same function without documenting which.

**Code example from source:**

```go
// os/file.go:388-390
// Open opens the named file for reading. If successful, methods on
// the returned file can be used for reading; the associated file
// descriptor has mode [O_RDONLY].
// If there is an error, it will be of type [*PathError].
func Open(name string) (*File, error) {
```

---

## 9. Concurrency Documentation

**Pattern name:** "Safe for concurrent use" / Concurrency Guarantees

**Source citation:** [net/http/transport.go#L79](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/net/http/transport.go#L79), [os/types.go#L17](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/os/types.go#L17), [regexp/regexp.go#L77](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/regexp/regexp.go#L77)

**What it does:** Doc comments explicitly state the concurrency safety of a type
or note exceptions where concurrent use is not safe.

**Why:** Go programs are inherently concurrent. Without explicit documentation,
users must guess whether a type needs external synchronization.

**Anti-pattern:** Leaving concurrency safety undocumented; documenting it
inconsistently across methods; saying "thread-safe" (Java-ism, use "safe for
concurrent use by multiple goroutines").

**Code examples from source:**

```go
// net/http/transport.go:79-80
// Transports should be reused instead of created as needed.
// Transports are safe for concurrent use by multiple goroutines.

// os/types.go:17
// The methods of File are safe for concurrent use.

// regexp/regexp.go:77-79
// A Regexp is safe for concurrent use by multiple goroutines,
// except for configuration methods, such as [Regexp.Longest].
```

<!-- PATTERN_COMPLETE -->
