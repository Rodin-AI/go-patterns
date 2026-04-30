# Go Package Design Patterns

Patterns extracted from the Go standard library source code.

---

## 1. Package-Level Documentation

### Source: `src/io/io.go:5-13`, `src/sync/mutex.go:5-11`, `src/context/context.go:5-57`

```go
// src/io/io.go:5-13
// Package io provides basic interfaces to I/O primitives.
// Its primary job is to wrap existing implementations of such primitives,
// such as those in package os, into shared public interfaces that
// abstract the functionality, plus some other related primitives.
//
// Because these interfaces and primitives wrap lower-level operations with
// various implementations, unless otherwise informed clients should not
// assume they are safe for parallel execution.
package io
```

```go
// src/sync/mutex.go:5-11
// Package sync provides basic synchronization primitives such as mutual
// exclusion locks. Other than the Once and WaitGroup types, most are intended
// for use by low-level library routines. Higher-level synchronization is
// better done via channels and communication.
//
// Values containing the types defined in this package should not be copied.
package sync
```

### Why

The package comment:
1. **States the purpose** in one sentence
2. **Establishes contracts** (not safe for parallel execution, values must not be copied)
3. **Guides users** toward correct usage (prefer channels over mutexes)
4. **Appears before `package` keyword** — becomes `go doc` output

### Convention

- First sentence: `"Package X does Y."` or `"Package X provides Y."`
- For multi-file packages, put the package comment in `doc.go` or the primary file

### Anti-pattern

```go
// DON'T: No package comment
package myutil

// DON'T: Restate the obvious
// Package http provides HTTP stuff.
package http
```

---

## 2. Package Naming

### Source: All stdlib packages follow these conventions

**Stdlib examples:**
- `io` — not `ioutil`, not `ioutils`
- `fmt` — not `format`, not `formatting`
- `sync` — not `synchronization`
- `net/http` — not `net/httpserver`
- `encoding/json` — not `encoding/jsonparser`
- `context` — not `ctx` or `contexts`

### Why

Go package names are **short, lowercase, no underscores or mixedCaps**. The package name is part of every qualified identifier:

```go
// Good: package name provides context
http.Server     // not http.HTTPServer
json.Encoder    // not json.JSONEncoder
context.Context // the type IS the context
```

### Anti-pattern

```go
// DON'T: Stutter
package http
type HTTPServer struct{}  // http.HTTPServer — redundant

// DON'T: Utility package names
package utils    // what does it DO?
package helpers  // grab bag, no cohesion
package common   // everything ends up here
```

---

## 3. internal/ Packages — Restricting Visibility

### Source: `src/net/http/internal/`, `src/encoding/json/internal.go`

```
src/net/http/internal/
├── ascii/
├── chunked.go
├── http2/
├── httpcommon/
├── sniff.go
└── testcert/
```

### Why

Packages under `internal/` can only be imported by code rooted at the parent of `internal`. This lets you share code between sub-packages without making it public API.

- `net/http/internal/ascii` → importable by `net/http` and children
- NOT importable by `net/url` or any other package

### When to Use

**Triggers:**
- You have helper code shared between sub-packages but NOT part of your public API
- You're tempted to export a function "just for testing" — put it in `internal/` instead
- Your package has grown and you want to split it without committing to new public APIs

**Example — before:**
```go
// pkg/mylib/helpers.go — exported just so pkg/mylib/sub can use it
package mylib

func ParseInternalFormat(s string) Thing { ... } // now anyone can depend on this!
```

**Example — after:**
```go
// pkg/mylib/internal/parse/parse.go
package parse

func InternalFormat(s string) Thing { ... } // only importable by pkg/mylib and children

// pkg/mylib/sub/handler.go
import "pkg/mylib/internal/parse" // ✓ allowed
```

### Anti-pattern

```go
// DON'T: Export implementation details
package mylib
func HelperThatOnlyIUse() {}  // pollutes API surface

// DO: Move to internal/
```

---

## 4. Export Rules — The Capital Letter Boundary

### Source: `src/io/io.go` — exported vs unexported

```go
// src/io/io.go
var EOF = errors.New("EOF")              // exported: uppercase
var errInvalidWrite = errors.New(...)    // unexported: lowercase

type teeReader struct {  // unexported type
    r Reader
    w Writer
}

func TeeReader(r Reader, w Writer) Reader {  // exported constructor
    return &teeReader{r, w}
}
```

### Why

`teeReader` is unexported because:
1. Users don't need to know its implementation
2. The return type is `Reader` (interface) — maximum flexibility
3. The struct's fields can change without breaking anyone

### Anti-pattern

```go
// DON'T: Export everything "just in case"
type Parser struct {
    Input string   // should this be settable?
    buffer []byte  // internal state
    pos    int
}
```

---

## 5. init() Functions — Use Sparingly

### Source: `src/net/http/http2.go:37`

```go
// src/net/http/http2.go:37
func init() {
    // register HTTP/2 protocol implementation
}
```

### Why

The stdlib uses `init()` for:
- **Driver registration** (database drivers register via init)
- **Protocol negotiation** (HTTP/2 registers its handler)

### Rules

1. Should have no side effects beyond registration
2. No errors possible (can't return error from init)
3. Keep them short
4. Prefer explicit initialization in `main()` when possible

### When to Use

**Triggers:**
- You're writing a driver or plugin that needs to register itself with a central registry on import
- The registration is side-effect-only (no return value, can't fail)
- You want `import _ "mydb/driver"` to make the driver available without explicit setup

**Example — before:**
```go
// main.go — user must manually register every driver
func main() {
    postgres.Register()   // easy to forget
    mysql.Register()      // order matters?
    sqlite.Register()
}
```

**Example — after:**
```go
// postgres/driver.go
func init() {
    sql.Register("postgres", &Driver{}) // auto-registers on import
}

// main.go — import for side-effect
import _ "github.com/lib/pq" // driver registers itself
```

### Anti-pattern

```go
// DON'T: Do heavy work in init
func init() {
    db = connectToDatabase()     // fails silently, crashes later
    cache = loadGigabyteFile()   // blocks startup
}

// DO: Prefer explicit setup in main()
func main() {
    db, err := connectToDatabase()
    if err != nil {
        log.Fatal(err)
    }
}
```

---

## 6. Functional Options Pattern

The stdlib uses struct-based configuration (`http.Server`, `tls.Config`). The functional options pattern emerged from the community for APIs with many optional parameters:

```go
// The pattern (idiom from Rob Pike/Dave Cheney):
type Option func(*Server)

func WithTimeout(d time.Duration) Option {
    return func(s *Server) {
        s.timeout = d
    }
}

func NewServer(addr string, opts ...Option) *Server {
    s := &Server{addr: addr, timeout: 30 * time.Second}
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```

### What stdlib uses: Config structs

```go
// net/http — struct literal configuration
srv := &http.Server{
    Addr:         ":8080",
    ReadTimeout:  5 * time.Second,
    WriteTimeout: 10 * time.Second,
    Handler:      mux,
}
```

### When to use which

| Approach | When |
|----------|------|
| Config struct | Few options, all data (stdlib preference) |
| Functional options | Many options, some involve behavior, public API stability |

---

## 7. Constructor Pattern — NewX Functions

### Source: `src/net/http/server.go:2639`, `src/database/sql/sql.go:836`

```go
// src/net/http/server.go:2639
func NewServeMux() *ServeMux {
    return new(ServeMux)
}

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

- `NewX()` when construction is trivial
- `OpenX()` when construction involves resources or can fail
- Return `*T` (concrete), not an interface
- Zero value should be usable where possible (`sync.Mutex`, `bytes.Buffer`)

### Anti-pattern

```go
// DON'T: Constructor that returns interface
func NewWriter() io.Writer { return &myWriter{} }  // hides methods

// DON'T: Require constructor when zero value works
// var b bytes.Buffer  ← just works
```

---

## 8. Package Organization — One Concern Per Package

### Source: Standard library structure

```
src/
├── io/          # I/O interfaces + helpers
├── os/          # OS operations
├── net/         # network primitives
│   ├── http/    # HTTP protocol
│   └── url/     # URL parsing
├── encoding/
│   ├── json/    # JSON codec
│   └── xml/     # XML codec
├── database/
│   └── sql/     # SQL abstraction
│       └── driver/  # SPI for drivers
└── context/     # cancellation propagation
```

### Why

Each package has a single, clear responsibility. Packages communicate through interfaces, not shared state.

### Anti-pattern

```go
// DON'T: Package per type (50 packages with 1 file each)
package user
package order
package payment

// DON'T: Circular dependencies
package a imports package b
package b imports package a  // compile error
```

---

## 9. API Layering — User vs Implementor (database/sql)

### Source: `src/database/sql/sql.go` vs `src/database/sql/driver/driver.go`

**User-facing (database/sql):**
```go
db, _ := sql.Open("postgres", connStr)
rows, _ := db.QueryContext(ctx, "SELECT ...")
```

**Driver-facing (database/sql/driver):**
```go
type Driver interface {
    Open(name string) (Conn, error)
}
type Conn interface {
    Prepare(query string) (Stmt, error)
    Close() error
    Begin() (Tx, error)
}
```

### Why

The user never sees `driver.Conn`. The driver never sees `sql.DB`'s pool logic. Clean separation: users get high-level safe API; drivers implement minimal interface.

---

## 10. Context Key Pattern — Type-Safe Context Values

### Source: `src/context/context.go:132-164`, `src/net/http/server.go:244-252`

```go
// src/context/context.go:132-164 (from doc)
// package user
//
// type key int
// var userKey key
//
// func NewContext(ctx context.Context, u *User) context.Context {
//     return context.WithValue(ctx, userKey, u)
// }
//
// func FromContext(ctx context.Context) (*User, bool) {
//     u, ok := ctx.Value(userKey).(*User)
//     return u, ok
// }
```

```go
// src/net/http/server.go:244-252
var (
    ServerContextKey = &contextKey{"http-server"}
    LocalAddrContextKey = &contextKey{"local-addr"}
)

type contextKey struct {
    name string
}
```

### Why

- **Unexported key type** prevents other packages from accessing your values
- **Type-safe accessors** avoid repeated type assertions
- **Pointer-based keys** guarantee uniqueness

### When to Use

**Triggers:**
- You need to pass request-scoped metadata through a call chain (user ID, trace ID, auth token)
- The data crosses package boundaries and isn't appropriate as a function parameter
- You want type safety — only your package should read/write its context values

**Example — before:**
```go
// String keys — any package can collide or access your values
ctx = context.WithValue(ctx, "userID", 42)
uid := ctx.Value("userID").(int) // panics if wrong type or missing
```

**Example — after:**
```go
type ctxKey struct{}

func WithUserID(ctx context.Context, id int) context.Context {
    return context.WithValue(ctx, ctxKey{}, id)
}

func UserID(ctx context.Context) (int, bool) {
    id, ok := ctx.Value(ctxKey{}).(int)
    return id, ok
}
```

### Anti-pattern

```go
// DON'T: Use string keys (collision risk)
ctx = context.WithValue(ctx, "user", user)

// DON'T: Store optional parameters in context
ctx = context.WithValue(ctx, "timeout", 5*time.Second)  // use function params!
```

---

## 11. Struct Tags for Codec Configuration

### Source: `src/encoding/json/tags.go:17-21`, `src/encoding/json/encode.go:101-181`

```go
// src/encoding/json/tags.go:17-21
func parseTag(tag string) (string, tagOptions) {
    tag, opt, _ := strings.Cut(tag, ",")
    return tag, tagOptions(opt)
}
```

Usage in struct definitions:
```go
type Person struct {
    Name    string `json:"name"`
    Age     int    `json:"age,omitempty"`
    Secret  string `json:"-"`                // always omitted
    Address string `json:"addr,omitempty"`
}
```

### Why

Struct tags are metadata for codecs. The `json` package reads `json:"..."` tags via reflection to control field names and behavior. The format is `key:"value"` with comma-separated options.

### Convention (from encode.go docs, line 101-181)

- `json:"fieldname"` — override JSON key name
- `json:",omitempty"` — omit if zero value
- `json:"-"` — never include
- `json:"-,"` — use literal `-` as name

---

## Summary: Package Design Principles

| Principle | Rule |
|-----------|------|
| Package comment | `"Package X does Y."` before `package` keyword |
| Naming | Short, lowercase, no stutter |
| Encapsulation | `internal/` for private shared code |
| Exports | Minimum surface; unexported by default |
| init() | Only for registration; prefer explicit setup |
| Constructors | `NewX()` → `*T`; prefer usable zero values |
| Organization | One concern per package |
| API layers | Separate user from implementor (SPI) |
| Context values | Unexported key type + typed accessors |
| Configuration | Struct literals or functional options |
