# Go Patterns (from Source)

Prescriptive patterns extracted from golang/go source.
"If writing new Go, follow these rules."

---

## Interface Design

### Single-Method Interfaces

**Rule:** Define interfaces with one method. Compose for larger
contracts.

```go
// Source: src/io/io.go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Composition:
type ReadWriter interface {
    Reader
    Writer
}
```

**Why:** Single-method interfaces maximize implementability. Any type
with a `Read` method satisfies `Reader` — no explicit declaration
needed. The Go stdlib has 15 compositions of just 4 primitives in
`io/io.go`.

**When NOT to use:** When the interface genuinely requires multiple
methods that always go together (e.g., `http.Handler` is one method
but `sort.Interface` is three because Len/Less/Swap are inseparable).

### Accept Interfaces, Return Structs

**Rule:** Function parameters should be interfaces. Return values
should be concrete types.

```go
// Source: src/io/io.go — Copy accepts interfaces
func Copy(dst Writer, src Reader) (written int64, err error)

// Source: src/bufio/bufio.go — NewReader returns concrete
func NewReader(rd io.Reader) *Reader
```

**Why:** Accepting interfaces means any implementation works. Returning
structs means callers get full access to methods and fields without
type assertions.

**When NOT to use:** When you need to return different types based on
input (return an interface). When the concrete type is unexported
(return an interface to hide it).

---

## Error Handling

### Sentinel Errors for Known Conditions

**Rule:** Define package-level `var Err*` for conditions callers need
to check.

```go
// Source: src/path/filepath/match.go
var ErrBadPattern = errors.New("syntax error in pattern")

// Source: src/encoding/hex/hex.go
var ErrLength = errors.New("encoding/hex: odd length hex string")
```

**Why:** Callers use `errors.Is(err, filepath.ErrBadPattern)` to handle
specific cases. The error message includes the package name for
context.

**When NOT to use:** For errors that callers never need to distinguish.
For errors that carry dynamic context (use error types instead).

### Wrap with %w for Context

**Rule:** Add context when propagating errors. Use `%w` to preserve
the original.

```go
// Source: src/encoding/json/v2_decode.go
return fmt.Errorf("cannot parse %q as JSON number: %w", val, strconv.ErrSyntax)
```

**Why:** Wrapping creates a chain. Callers can `errors.Is()` to find
the root cause while seeing the full context path.

**When NOT to use:** When the original error's identity should be
hidden (use `%v` instead of `%w` to break the chain intentionally).

---

## Testing

### Table-Driven Tests

**Rule:** Use `[]struct{}` slices for test cases. Each case has a name.

```go
// Source: src/testing/testing_test.go
func TestSetenv(t *testing.T) {
    tests := []struct {
        name               string
        key                string
        initialValueExists bool
        initialValue       string
        newValue           string
    }{
        {
            name:               "initial value exists",
            key:                "GO_TEST_KEY_1",
            initialValueExists: true,
            initialValue:       "111",
            newValue:           "222",
        },
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // ...
        })
    }
}
```

**Why:** Each case is named, making failure output clear. Adding cases
is one struct literal. The pattern is universal in Go's own tests
(1,811 test files use it).

**When NOT to use:** Single-case tests. Tests where setup varies
significantly between cases (use separate test functions).

### testdata/ for Fixtures

**Rule:** Put test fixtures in a `testdata/` directory. The Go tool
ignores it.

**Why:** Convention enforced by the toolchain — `testdata/` is never
compiled. Golden files, sample inputs, and expected outputs live here.

**When NOT to use:** Generated test data (create it in TestMain or
setup).

---

## Package Organization

### Flat Packages

**Rule:** No `pkg/` wrapper. Top-level directories ARE the packages.

```
src/
├── fmt/
├── io/
├── net/
│   └── http/
├── encoding/
│   └── json/
└── internal/
```

**Why:** The import path IS the directory path. `import "fmt"` loads
`src/fmt/`. No indirection, no wrapper directories.

**When NOT to use:** Never. There is no legitimate reason for a `pkg/`
directory in Go. (The `pkg/` convention from early community projects
was a mistake that the Go team never endorsed.)

### internal/ for Shared Private Code

**Rule:** Put shared utilities that shouldn't be public API in
`internal/`.

```go
// Only packages within the same tree can import this:
import "internal/singleflight"
```

**Why:** The compiler enforces the boundary. External packages get a
build error if they try to import your internals. This is stronger than
unexported identifiers (which still allow same-package access).

**When NOT to use:** Code that's stable enough for public API (promote
it). Code only used by one package (keep it unexported within that
package).

---

## Concurrency

### context.Context as First Parameter

**Rule:** Functions that do I/O or long-running work take `ctx
context.Context` as the first parameter.

```go
// Source: src/net/http (throughout)
func (c *Client) Do(req *Request) (*Response, error)
// becomes:
func (c *Client) do(ctx context.Context, req *Request) (*Response, error)
```

**Why:** Context carries cancellation, deadlines, and request-scoped
values. First-parameter position is a universal convention — every Go
developer knows to look for it there.

**When NOT to use:** Pure computation (no I/O, no blocking). Package-
level init functions. Short-lived operations that can't be cancelled.

### Don't Start Goroutines in Libraries

**Rule:** Let the caller control concurrency. Return results; don't
spawn goroutines.

**Why:** The Go stdlib almost never starts goroutines in library code
(exception: `net/http` server). Libraries that spawn goroutines create
lifecycle management problems — who stops them? who waits for them?

**When NOT to use:** Servers (http.Server manages connections).
Background work that the caller explicitly requested (provide a
`Start`/`Stop` interface).

---

## Documentation

### Package Comment in doc.go

**Rule:** Put the package overview in a `doc.go` file with a `/*...*/`
comment.

```go
// Source: src/fmt/doc.go
/*
Package fmt implements formatted I/O with functions analogous
to C's printf and scanf.

# Printing

There are four families of printing functions...
*/
package fmt
```

**Why:** Separates documentation from implementation. The `#` headings
create sections in pkg.go.dev. `doc.go` is conventional — reviewers
know where to find the overview.

**When NOT to use:** Small packages where the package comment fits
naturally in the main file.

### Example Functions

**Rule:** Demonstrate usage with `Example*` functions in `*_test.go`.

```go
func ExampleSprintf() {
    fmt.Println(fmt.Sprintf("Hello, %s", "world"))
    // Output: Hello, world
}
```

**Why:** Examples are tests (they run and verify output). They appear
in docs. They can't go stale because the build fails if they break.

---

## Naming

### Short Package Names

**Rule:** Package names are short, lowercase, singular nouns.

`fmt`, `io`, `net`, `os`, `sync`, `time`, `bytes`, `errors`

**Why:** Package names prefix every exported identifier. `bytes.Buffer`
reads well. `utilities.Buffer` doesn't.

**When NOT to use:** Never use plurals (`utils`, `helpers`, `models`).
Never use generic names that could apply to anything.

### MixedCaps, No Underscores

**Rule:** Exported = `UpperCamelCase`. Unexported = `lowerCamelCase`.
Never underscores.

```go
// Exported:
type ReadWriter interface {}
func NewReader(rd io.Reader) *Reader

// Unexported:
type call struct {}
func (g *Group) doCall(c *call, key string) {}
```

**Why:** Capitalization IS the visibility mechanism. Underscores are
reserved for test files (`_test.go`) and generated code.

### Getters Don't Say "Get"

**Rule:** A getter for field `owner` is `Owner()`, not `GetOwner()`.

**Why:** Go convention since the language spec. Setters DO say "Set":
`SetOwner()`.

---

## Smells

### go:linkname (Escape Hatch Abuse)

```go
//go:linkname localFunction remote/package.Function
```

1,711 uses in Go's own source — but the team is actively removing them.
Third-party packages using go:linkname to access stdlib internals are
explicitly unsupported and will break on upgrade.

**Lesson:** If you need go:linkname, your API boundary is wrong.
Redesign the interface instead.

### TODO Without Owner

```go
// TODO: fix this later       ← smell
// TODO(gri): fix this later  ← correct
```

The Go team's 3,428 TODOs all have owners. An unowned TODO is
unaccountable — no one will fix it because no one owns it.

### Huge Files (proc.go = 8,156 lines)

Even Go's own scheduler is a single 8K-line file. This isn't a pattern
to copy — it's a known limitation. The Go team keeps it together
because splitting would break the scheduler's mental model, not because
one file is ideal.

**Lesson:** If your file is >1000 lines, question whether it's a real
unit or just accumulated code. The scheduler has an excuse. Your CRUD
handler doesn't.

### Generated Code Without Generator

If you check in generated code, also check in the generator (or
clearly document how to regenerate). Go's SSA rewrite rules have
generators alongside them. Generated code without visible generators
becomes unmaintainable magic.

<!-- PATTERN_COMPLETE -->
