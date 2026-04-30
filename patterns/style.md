# Code Style Patterns in the Go Standard Library

## 1. Naming Conventions: mixedCaps (No Underscores)

**Pattern name:** mixedCaps / MixedCaps

**Source citation:** All stdlib code (enforced by `gofmt` convention, documented in Effective Go)

**What it does:** All identifiers use mixedCaps (unexported) or MixedCaps (exported).
Underscores are never used in Go names except for test helpers and generated code.

**Why:** Consistent casing makes code scannable. The exported/unexported distinction
is communicated solely through initial capitalization — no separate `public`/`private`
keywords needed.

**Anti-pattern:** `snake_case` names; `ALL_CAPS` for constants; Hungarian notation
(`strName`, `iCount`).

**Code examples from source:**

```go
// net/http/server.go — exported
type Server struct { ... }
func ListenAndServe(addr string, handler Handler) error

// net/http/server.go — unexported
func (s *Server) shuttingDown() bool
const shutdownPollIntervalMax = 500 * time.Millisecond
```

---

## 2. Acronyms Are All-Caps

**Pattern name:** Acronym Capitalization

**Source citation:** `net/http/request.go` line 130 (`URL`), `net/http/server.go` line 3041 (`TLSConfig`), `encoding/json/stream.go` line 280 (`JSON`)

**What it does:** Acronyms and initialisms (URL, HTTP, ID, JSON, XML, HTML, TLS, TCP)
are always fully capitalized when exported, and fully lowercased when unexported.

**Why:** Consistency. `URL` not `Url`, `ID` not `Id`, `HTTP` not `Http`. This
applies even mid-word: `ServeHTTP`, `xmlEncoder`, `htmlEscape`.

**Anti-pattern:** `Url`, `Http`, `Json`, `Id` — mixing cases within an acronym.

**Code examples from source:**

```go
// net/http/request.go:130
URL *url.URL

// net/http/request.go:822
func ParseHTTPVersion(vers string) (major, minor int, ok bool)

// net/http/server.go:3041
TLSConfig *tls.Config

// encoding/json/stream.go:280
var _ Marshaler = (*RawMessage)(nil)
```

---

## 3. File Organization by Responsibility

**Pattern name:** One Concept Per File

**Source citation:** `net/http/` directory structure

**What it does:** Large packages split code into files by topic/type: `client.go`,
`server.go`, `transport.go`, `request.go`, `response.go`, `cookie.go`, `header.go`,
`fs.go`, `doc.go`. Each file is focused.

**Why:** Navigability. When you want to find client logic, you open `client.go`.
Files stay manageable sizes. Related code lives together.

**Anti-pattern:** One giant file with everything; splitting by access level
(`public.go` / `private.go`); splitting by method count rather than concept.

**File layout from `net/http/`:**

```
client.go        — Client type and methods
transport.go     — Transport type (low-level RoundTripper)
server.go        — Server, Handler, ServeMux
request.go       — Request type and parsing
response.go      — Response type and reading
cookie.go        — Cookie parsing and serialization
header.go        — Header type and canonicalization
fs.go            — FileServer, file serving
doc.go           — Package documentation
clone.go         — Clone helpers
method.go        — HTTP method constants
pattern.go       — URL pattern matching (ServeMux routing)
```

---

## 4. Blank Identifier for Interface Compliance

**Pattern name:** `var _ Interface = (*Type)(nil)`

**Source citation:** `io/io.go` line 645, `os/file.go` lines 747–750, `encoding/json/stream.go` lines 280–281

**What it does:** A package-level `var _ InterfaceName = (*ConcreteType)(nil)` declares
that the concrete type must satisfy the interface. The compiler verifies this at
build time.

**Why:** Catches interface drift at compile time without creating an instance. The
blank identifier discards the value — this is purely a static assertion.

**Anti-pattern:** Relying on tests to catch interface conformance; skipping the check
and discovering the mismatch at runtime; using reflection.

**Code examples from source:**

```go
// io/io.go:645
var _ ReaderFrom = discard{}

// os/file.go:747-750
var _ fs.StatFS = dirFS("")
var _ fs.ReadFileFS = dirFS("")
var _ fs.ReadDirFS = dirFS("")
var _ fs.ReadLinkFS = dirFS("")

// encoding/json/stream.go:280-281
var _ Marshaler = (*RawMessage)(nil)
var _ Unmarshaler = (*RawMessage)(nil)

// net/http/server.go:4071
var _ Pusher = (*timeoutWriter)(nil)
```

---

## 5. Named Return Values

**Pattern name:** Named Returns for Documentation (and Defer)

**Source citation:** `io/io.go` lines 87, 100, 314, 387; `os/file.go` lines 140, 175

**What it does:** Return values are given names when the names add documentary value
(clarifying which int is what) or when `defer` needs to modify the return value.

**Why:** `(n int, err error)` is immediately understandable — `n` is the byte count.
Named returns also enable `defer func() { err = wrap(err) }()` patterns.

**Anti-pattern:** Naming returns for trivial functions where the types are
self-explanatory; using named returns as implicit variables throughout the function
body (confusing naked returns); always using naked `return` statements.

**Code examples from source:**

```go
// io/io.go:87 — Interface documentation
type Reader interface {
    Read(p []byte) (n int, err error)
}

// io/io.go:100
type Writer interface {
    Write(p []byte) (n int, err error)
}

// io/io.go:387 — Named return used with defer-style logic
func Copy(dst Writer, src Reader) (written int64, err error) {
    return copyBuffer(dst, src, nil)
}

// os/file.go:140 — Named return for readability
func (f *File) Read(b []byte) (n int, err error) {
    if err := f.checkValid("read"); err != nil {
        return 0, err
    }
    n, e := f.read(b)
    return n, f.wrapErr("read", e)
}
```

---

## 6. Defer for Resource Cleanup

**Pattern name:** `defer mu.Unlock()` / `defer f.Close()`

**Source citation:** `net/http/server.go` lines 3173–3174, `net/http/example_handle_test.go` lines 21–22

**What it does:** Resources acquired at the top of a scope are immediately deferred
for cleanup. Mutexes are locked then immediately `defer Unlock()`'d.

**Why:** Guarantees cleanup regardless of return path (early returns, panics). Keeps
the acquire/release pair visually adjacent. Reduces bugs from forgotten unlocks.

**Anti-pattern:** Manual unlock at each return point; deferring in a loop (deferred
calls accumulate until function exit); deferring expensive operations that should
run earlier.

**Code examples from source:**

```go
// net/http/server.go:3173-3174
func (s *Server) Close() error {
    s.inShutdown.Store(true)
    s.mu.Lock()
    defer s.mu.Unlock()
    // ...
}

// net/http/example_handle_test.go:21-22
func (h *countHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    h.mu.Lock()
    defer h.mu.Unlock()
    h.n++
    fmt.Fprintf(w, "count is %d\n", h.n)
}
```

---

## 7. Error Wrapping and Sentinel Errors

**Pattern name:** Sentinel Errors + Structured Error Types

**Source citation:** `os/error.go` lines 14–27, `os/error.go` lines 46–67

**What it does:** Package-level sentinel errors (`ErrNotExist`, `ErrPermission`) are
declared as `var` for use with `errors.Is()`. Structured error types (`*PathError`,
`*SyscallError`) carry context and implement `Unwrap()` for the errors chain.

**Why:** Enables programmatic error handling without string matching. `errors.Is(err, os.ErrNotExist)` works regardless of wrapping depth. Structured types let callers
extract the operation, path, or underlying syscall error.

**Anti-pattern:** Comparing error strings; creating unique error types for every
possible failure; not implementing `Unwrap`; sentinel errors as `const` (breaks
`errors.Is` for wrapped errors — use `var`).

**Code examples from source:**

```go
// os/error.go:14-27
var (
    ErrInvalid    = fs.ErrInvalid    // "invalid argument"
    ErrPermission = fs.ErrPermission // "permission denied"
    ErrExist      = fs.ErrExist      // "file already exists"
    ErrNotExist   = fs.ErrNotExist   // "file does not exist"
    ErrClosed     = fs.ErrClosed     // "file already closed"
)

// os/error.go:46
type PathError = fs.PathError

// os/error.go:49-57
type SyscallError struct {
    Syscall string
    Err     error
}

func (e *SyscallError) Error() string { return e.Syscall + ": " + e.Err.Error() }
func (e *SyscallError) Unwrap() error { return e.Err }
```

---

## 8. Receiver Naming: Short, Consistent, Never `this`/`self`

**Pattern name:** Single-Letter or Short Receiver Names

**Source citation:** All stdlib code; `net/http/server.go` uses `s` for Server, `bufio/scan.go` uses `s` for Scanner

**What it does:** Method receivers use 1–2 letter abbreviations of the type name,
consistent across all methods of that type: `s` for `*Server`, `b` for `*Builder`,
`f` for `*File`, `t` for `*Timer`.

**Why:** Receivers appear on every method. Short names reduce visual noise. Consistency
within a type avoids confusion. `this`/`self` are alien to Go's conventions.

**Anti-pattern:** `this`, `self`, `me`; long receiver names like `server`, `scanner`;
inconsistent receivers across methods of the same type.

**Code examples from source:**

```go
// net/http/server.go
func (s *Server) ListenAndServe() error { ... }
func (s *Server) Serve(l net.Listener) error { ... }
func (s *Server) Shutdown(ctx context.Context) error { ... }

// strings/builder.go
func (b *Builder) WriteString(s string) (int, error) { ... }
func (b *Builder) String() string { ... }
func (b *Builder) Grow(n int) { ... }

// os/file.go
func (f *File) Read(b []byte) (n int, err error) { ... }
func (f *File) Name() string { ... }
```

---

## 9. Constants: Typed, Grouped, with iota

**Pattern name:** Typed Constants with iota

**Source citation:** `crypto/crypto.go` lines 70–85, `time/time.go` lines 936–943

**What it does:** Related constants are grouped in a `const ( ... )` block using
a named type and `iota` for sequential values. Constants of the same type
are exhaustively listed together.

**Why:** Type safety (can't accidentally pass an `os.Flag` where a `crypto.Hash` is
expected). `iota` eliminates magic numbers. Grouping makes the full set visible.

**Anti-pattern:** Untyped numeric constants; separate `const` declarations for related
values; using raw integers in function signatures.

**Code examples from source:**

```go
// crypto/crypto.go:70-85
const (
    MD4         Hash = 1 + iota
    MD5
    SHA1
    SHA224
    SHA256
    // ...
)

// time/time.go:936-943
const (
    Nanosecond  Duration = 1
    Microsecond          = 1000 * Nanosecond
    Millisecond          = 1000 * Microsecond
    Second               = 1000 * Millisecond
    Minute               = 60 * Second
    Hour                 = 60 * Minute
)
```

---

## 10. Comments: Guard Clauses Over Conditions

**Pattern name:** `// guards x` Field Comments

**Source citation:** `net/http/example_handle_test.go` line 16

**What it does:** When a sync primitive (mutex) protects specific fields, a brief
comment documents what it guards: `mu sync.Mutex // guards n`.

**Why:** Concurrency bugs come from unclear ownership. A one-line comment makes the
lock's scope obvious to every reader.

**Anti-pattern:** No documentation of what a lock protects; locks that protect
"everything" (unclear scope); comments that restate the type.

**Code example from source:**

```go
// net/http/example_handle_test.go:16-17
type countHandler struct {
    mu sync.Mutex // guards n
    n  int
}
```

---

## 11. Duration Type Pattern

**Pattern name:** Named Type for Semantic Units

**Source citation:** `time/time.go` lines 915–943

**What it does:** `Duration` is `type Duration int64` — a named type over a primitive.
This gives it its own method set (`String()`, `Hours()`, `Truncate()`) and prevents
accidental mixing with raw int64 values.

**Why:** Semantic meaning through the type system. You can't accidentally pass
nanoseconds where seconds are expected. Methods provide conversion and formatting.
Constants like `time.Second` make intent clear.

**Anti-pattern:** Using raw `int64` for durations; accepting `int` parameters for
time intervals; mixing units (milliseconds in one place, seconds in another).

**Code example from source:**

```go
// time/time.go:915
type Duration int64

// time/time.go:947-949
func (d Duration) String() string {
    var arr [32]byte
    n := d.format(&arr)
    return string(arr[n:])
}
```

---

## 12. gofmt: Non-Negotiable Formatting

**Pattern name:** Canonical Formatting via gofmt

**Source citation:** Every file in the Go standard library

**What it does:** All Go code is formatted with `gofmt`. Tabs for indentation, spaces
for alignment. No style debates — the tool decides.

**Why:** Eliminates formatting bikesheds. All Go code looks the same regardless of
author. Diffs show only semantic changes, never style changes. Tooling can parse
and emit canonical code.

**Anti-pattern:** Manual formatting; spaces for indentation; custom alignment rules;
checking in code that `gofmt` would modify.

**Key rules enforced by gofmt:**

- Tabs for indentation
- Opening brace on the same line (`if x {`)
- No optional parentheses (`if x`, not `if (x)`)
- Aligned struct field tags
- One blank line between top-level declarations
- No trailing whitespace

---

## 13. Import Organization

**Pattern name:** Grouped Imports (stdlib / external / internal)

**Source citation:** `net/http/server.go` lines 8–36

**What it does:** Imports are organized in groups separated by blank lines:
1. Standard library
2. External packages (golang.org/x, third-party)
3. Internal packages

The `goimports` tool enforces this automatically.

**Why:** Scannable at a glance. Makes dependency provenance clear (stdlib vs.
external). Reduces merge conflicts.

**Code example from source:**

```go
// net/http/server.go:8-36
import (
    "bufio"
    "bytes"
    "context"
    "crypto/tls"
    "errors"
    "fmt"
    // ... more stdlib ...
    "time"
    _ "unsafe" // for linkname

    "golang.org/x/net/http/httpguts"
)
```
