# Struct Design Patterns in the Go Standard Library


**Source:** [golang/go](https://github.com/golang/go) at commit [`17bd5ab`](https://github.com/golang/go/tree/17bd5ab8c650155dd2bd09f7005726552639eea0)
## 1. Zero-Value Usability

**Pattern name:** Zero Value Ready

**Source citation:** [net/http/client.go#L31](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/net/http/client.go#L31), [strings/builder.go#L14](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/strings/builder.go#L14)

**What it does:** Structs are designed so their zero value is immediately useful without
explicit initialization. Nil fields fall back to sensible defaults at method call time.

**Why:** Eliminates mandatory constructors, reduces boilerplate, makes the type
self-documenting about its defaults. Users can write `var c http.Client` and start
making requests.

**When to Use**

**Triggers:**
- You're designing a type where the "empty" or "default" state is meaningful and safe
- Users should be able to write `var x MyType` and immediately call methods
- Your struct's nil/zero fields can fall back to sensible defaults at call time

**Example — before:**
```go
type Cache struct {
    store map[string][]byte
    ttl   time.Duration
}

// Panics on zero value — store is nil!
func (c *Cache) Set(k string, v []byte) { c.store[k] = v }
```

**Example — after:**
```go
type Cache struct {
    store map[string][]byte
    ttl   time.Duration // zero means no expiry
}

func (c *Cache) Set(k string, v []byte) {
    if c.store == nil {
        c.store = make(map[string][]byte) // lazy init on first use
    }
    c.store[k] = v
}
```

### When NOT to Use

**Don't use this when:**
- The type has mandatory dependencies that *must* be provided (a DB connection, an io.Reader with no sensible default)
- The zero value would be dangerous rather than merely useless (e.g., a zero-value security config that disables auth)
- Lazy initialization adds per-call overhead on a hot path and the constructor is called once

**Over-application example:**
```go
// Trying to make a DB-backed store zero-value ready
type UserStore struct {
    db *sql.DB // nil means... what? No sensible default exists.
}

func (s *UserStore) Get(id int) (*User, error) {
    if s.db == nil {
        return nil, errors.New("no database configured") // every call pays for nil check
    }
    // ...
}
```

**Better alternative:**
```go
// A constructor makes the requirement explicit
func NewUserStore(db *sql.DB) *UserStore {
    return &UserStore{db: db}
}
```

**Why:** When there's no meaningful default for a dependency, forcing zero-value usability
just moves the error from compile time (missing argument) to runtime (nil check on every call).
Use a constructor instead.

**Anti-pattern:** Requiring a constructor for basic use; panicking on zero-value use;
requiring all fields be set before the type is functional.

**Code examples from source:**

```go
// net/http/client.go:31-35
// A Client is an HTTP client. Its zero value ([DefaultClient]) is a
// usable client that uses [DefaultTransport].
type Client struct {
    Transport RoundTripper  // If nil, DefaultTransport is used.
    // ...
}

// net/http/client.go:109
var DefaultClient = &Client{}
```

```go
// strings/builder.go:14-16
// A Builder is used to efficiently build a string using [Builder.Write] methods.
// It minimizes memory copying. The zero value is ready to use.
// Do not copy a non-zero Builder.
type Builder struct {
    addr *Builder
    buf  []byte
}
```

```go
// bytes/buffer.go:19-20
// A Buffer is a variable-sized buffer of bytes with [Buffer.Read] and [Buffer.Write] methods.
// The zero value for Buffer is an empty buffer ready to use.
type Buffer struct {
    buf      []byte
    off      int
    lastRead readOp
}
```

---

## 2. Unexported Struct with Exported Wrapper

**Pattern name:** Indirection via Unexported Impl

**Source citation:** [os/types.go#L15](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/os/types.go#L15), [os/file_unix.go#L59](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/os/file_unix.go#L59)

**What it does:** The exported type (`File`) embeds a pointer to an unexported type
(`*file`) that holds the real implementation state. Users interact only with the
exported wrapper.

**Why:** Prevents users from directly constructing or copying the implementation struct.
Allows platform-specific implementations behind a uniform exported API. The extra
indirection ensures finalizers close the correct descriptor.

**Anti-pattern:** Exporting all implementation fields; allowing users to construct
the struct via a literal (bypassing invariants); needing platform #ifdefs in the
public API.

**Code example from source:**

```go
// os/types.go:15-20
// File represents an open file descriptor.
//
// The methods of File are safe for concurrent use.
type File struct {
    *file // os specific
}

// os/file_unix.go:59-71
// file is the real representation of *File.
// The extra level of indirection ensures that no clients of os
// can overwrite this data, which could cause the finalizer
// to close the wrong file descriptor.
type file struct {
    pfd         poll.FD
    name        string
    dirinfo     atomic.Pointer[dirInfo]
    nonblock    bool
    stdoutOrErr bool
    appendMode  bool
    inRoot      bool
}
```

---

## 3. Constructor Functions (NewXxx)

**Pattern name:** NewXxx Constructor

**Source citation:** [bufio/scan.go#L89](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/bufio/scan.go#L89), [bufio/bufio.go#L50](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/bufio/bufio.go#L50)

**What it does:** A package-level function `NewXxx(deps) *Xxx` constructs the type
with required dependencies and internal defaults that can't be expressed via zero
value alone.

**Why:** When a type has mandatory dependencies (e.g., an `io.Reader`), a constructor
clearly communicates what's required. The constructor can set internal invariants
(buffer sizes, split functions) that users shouldn't need to know about.

**When to Use**

**Triggers:**
- Your type has mandatory dependencies that can't be expressed as zero values (an `io.Reader`, a DB connection)
- Internal invariants must be set up (buffer allocation, goroutine start)
- The type isn't useful without initialization (unlike `sync.Mutex` or `bytes.Buffer`)

**Example — before:**
```go
type Parser struct {
    lexer    *Lexer
    buf      []Token
    maxDepth int
}

// User must know about all internal state:
p := &Parser{lexer: NewLexer(input), buf: make([]Token, 0, 64), maxDepth: 100}
```

**Example — after:**
```go
func NewParser(input io.Reader) *Parser {
    return &Parser{
        lexer:    NewLexer(input),
        buf:      make([]Token, 0, 64),
        maxDepth: 100,
    }
}

// User writes:
p := NewParser(file)
```

### When NOT to Use

**Don't use this when:**
- The type's zero value is already useful — adding a constructor creates unnecessary ceremony
- Your constructor takes 5+ optional parameters (use a config struct instead)
- The "mandatory dependency" is actually optional and has a sensible default (e.g., a logger defaulting to `slog.Default()`)

**Over-application example:**
```go
// Constructor for something that doesn't need one
func NewCounter() *Counter {
    return &Counter{count: 0} // zero value already does this!
}

// Forces users to write:
c := NewCounter()
```

**Better alternative:**
```go
type Counter struct {
    count int64
}
// Users write: var c Counter — done.
```

**Why:** If the zero value works, a constructor is just noise. It obscures the type's
actual simplicity and makes users wonder what hidden initialization they're missing.

**Anti-pattern:** Forcing users to manually set unexported fields; having a constructor
that takes 10 optional parameters (use config struct instead); requiring New when
zero value would suffice.

**Code examples from source:**

```go
// bufio/scan.go:89-96
func NewScanner(r io.Reader) *Scanner {
    return &Scanner{
        r:            r,
        split:        ScanLines,
        maxTokenSize: MaxScanTokenSize,
    }
}
```

```go
// bufio/bufio.go:50-62
func NewReaderSize(rd io.Reader, size int) *Reader {
    // Is it already a Reader?
    b, ok := rd.(*Reader)
    if ok && len(b.buf) >= size {
        return b
    }
    r := new(Reader)
    r.reset(make([]byte, max(size, minReadBufferSize)), rd)
    return r
}

// NewReader returns a new [Reader] whose buffer has the default size.
func NewReader(rd io.Reader) *Reader {
    return NewReaderSize(rd, defaultBufSize)
}
```

```go
// net/http/request.go:867-869
func NewRequest(method, url string, body io.Reader) (*Request, error) {
    return NewRequestWithContext(context.Background(), method, url, body)
}
```

---

## 4. NewXxx with Size/Options Variant

**Pattern name:** NewXxx / NewXxxSize Pair

**Source citation:** [bufio/bufio.go#L50](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/bufio/bufio.go#L50), 62, 589, 607

**What it does:** Provides two constructors — one with defaults (`NewReader`) and one
with explicit configuration (`NewReaderSize`). The default version calls the
configurable one.

**Why:** Most users want the default; power users need control. Layering avoids a
proliferation of constructor parameters for the common case.

**Anti-pattern:** Having only the complex constructor; making users guess the right
buffer size; inconsistent naming (e.g., `NewReaderWithSize`).

**Code example from source:**

```go
// bufio/bufio.go:589-607
func NewWriterSize(w io.Writer, size int) *Writer {
    // ...
}

func NewWriter(w io.Writer) *Writer {
    return NewWriterSize(w, defaultBufSize)
}
```

---

## 5. Config Struct Pattern

**Pattern name:** Configuration Struct (Exported Fields, Nil-Means-Default)

**Source citation:** [net/http/server.go#L3020](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/net/http/server.go#L3020), [crypto/tls/common.go#L566](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/crypto/tls/common.go#L566)+, [log/slog/handler.go#L135](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/log/slog/handler.go#L135)

**What it does:** A struct with exported, documented fields provides all
configuration knobs. Nil/zero values always mean "use the default".

**Why:** Self-documenting via godoc; no need for a setter method per option; easy to
construct partially; serializable; the zero value works. This is Go's primary
configuration pattern (preferred over functional options in the stdlib).

**When to Use**

**Triggers:**
- Your constructor has 4+ optional parameters that would make a function signature unwieldy
- You want users to see all options in one place with godoc documentation
- Zero/nil values should mean "use the default" — no required fields beyond what the constructor demands

**Example — before:**
```go
// 7 parameters — impossible to remember the order
func NewServer(addr string, handler http.Handler, readTimeout, writeTimeout time.Duration,
    maxConns int, logger *log.Logger, tlsConfig *tls.Config) *Server { ... }
```

**Example — after:**
```go
type ServerConfig struct {
    Addr         string        // ":8080" if empty
    Handler      http.Handler  // http.DefaultServeMux if nil
    ReadTimeout  time.Duration // zero means no timeout
    WriteTimeout time.Duration // zero means no timeout
    MaxConns     int           // 1000 if zero
    Logger       *log.Logger   // log.Default() if nil
    TLSConfig    *tls.Config   // plain HTTP if nil
}

func NewServer(cfg ServerConfig) *Server { ... }
```

### When NOT to Use

**Don't use this when:**
- You only have 1–2 options — just use function parameters
- The options are truly required (not optional) — they belong in the constructor signature
- Your config struct has methods or behavior — it's no longer a plain config, it's becoming a type

**Over-application example:**
```go
// Config struct for two required parameters
type ClientConfig struct {
    BaseURL string // required!
    APIKey  string // required!
}

func NewClient(cfg ClientConfig) (*Client, error) {
    if cfg.BaseURL == "" { return nil, errors.New("base URL required") }
    if cfg.APIKey == "" { return nil, errors.New("API key required") }
    // ...
}
```

**Better alternative:**
```go
// Required params are function args; only truly optional things go in a config
func NewClient(baseURL, apiKey string, opts *ClientOptions) (*Client, error) {
    // baseURL and apiKey are enforced by the signature
    // opts can be nil for defaults
}
```

**Why:** Config structs shine for *optional* configuration. When fields are required,
the compiler can't enforce them — you end up validating at runtime what the type system
could have caught. Keep required parameters as explicit function arguments.

**Anti-pattern:** Undocumented fields; requiring all fields set; using sentinel values
other than zero/nil for defaults; providing setters when direct assignment works.

**Code example from source:**

```go
// net/http/server.go:3020-3075 (abbreviated)
type Server struct {
    Addr string             // ":http" if empty
    Handler Handler         // http.DefaultServeMux if nil
    TLSConfig *tls.Config   // optional
    ReadTimeout time.Duration  // zero means no timeout
    WriteTimeout time.Duration // zero means no timeout
    MaxHeaderBytes int         // DefaultMaxHeaderBytes if zero
    ErrorLog *log.Logger       // log.Default() if nil
    // ...
}
```

```go
// log/slog/handler.go:135-175
type HandlerOptions struct {
    AddSource bool
    Level     Leveler        // LevelInfo if nil
    ReplaceAttr func(groups []string, a Attr) Attr
}

// Usage: If opts is nil, the default options are used.
func NewTextHandler(w io.Writer, opts *HandlerOptions) *TextHandler {
    if opts == nil {
        opts = &HandlerOptions{}
    }
    // ...
}
```

---

## 6. Interface-Based Pluggability

**Pattern name:** Interface Abstraction for Pluggable Implementations

**Source citation:** [crypto/crypto.go#L180](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/crypto/crypto.go#L180), [net/http/transport.go#L66](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/net/http/transport.go#L66)

**What it does:** Core behavior is defined via an interface. The package provides
a default concrete implementation, but any user type satisfying the interface
can be substituted.

**Why:** Decouples high-level logic from low-level implementation. Enables testing
(mock transports), hardware integration (HSM-backed signers), and third-party
extensions without forking the package.

**Anti-pattern:** Concrete-type coupling everywhere; interfaces with too many methods
(hard to implement); accepting an interface but only ever using one implementation.

**Code example from source:**

```go
// crypto/crypto.go:180-200
// Signer is an interface for an opaque private key that can be used for
// signing operations. For example, an RSA key kept in a hardware module.
type Signer interface {
    Public() PublicKey
    Sign(rand io.Reader, digest []byte, opts SignerOpts) (signature []byte, err error)
}
```

```go
// net/http/transport.go (line 66+)
// Transport is an implementation of [RoundTripper] that supports HTTP,
// HTTPS, and HTTP proxies...
// Transports should be reused instead of created as needed.
// Transports are safe for concurrent use by multiple goroutines.

// net/http/client.go:59-60
type Client struct {
    Transport RoundTripper  // If nil, DefaultTransport is used.
    // ...
}
```

---

## 7. Copy Protection via Dynamic Check

**Pattern name:** copyCheck (Runtime Copy Detection)

**Source citation:** [strings/builder.go#L32](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/strings/builder.go#L32)

**What it does:** On first mutation, the Builder records its own address. Subsequent
mutations compare the current receiver address against the recorded one. If they
differ, the struct was copied — it panics.

**Why:** Go has no language-level move semantics. For types where copying after first
use would cause data corruption or unsafe behavior (e.g., sharing an unsafe string
buffer), a runtime check is the pragmatic solution.

**Anti-pattern:** Silently allowing copies that corrupt state; using `sync.Mutex`-style
`noCopy` (vet catches it but it doesn't work for zero vs non-zero discrimination).

**Code example from source:**

```go
// strings/builder.go:32-40
func (b *Builder) copyCheck() {
    if b.addr == nil {
        b.addr = (*Builder)(abi.NoEscape(unsafe.Pointer(b)))
    } else if b.addr != b {
        panic("strings: illegal use of non-zero Builder copied by value")
    }
}
```

---

## 8. DefaultXxx Singleton

**Pattern name:** Package-Level Default Instance

**Source citation:** [net/http/client.go#L109](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/net/http/client.go#L109), [net/http/transport.go#L47](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/net/http/transport.go#L47)

**What it does:** The package provides a pre-configured, ready-to-use instance as
a package-level variable. Package-level convenience functions delegate to it.

**Why:** Makes the simple case trivial (`http.Get(url)`) while allowing custom
instances for advanced use. Users never need to touch the defaults unless they
have specific requirements.

**Anti-pattern:** Forcing construction for basic use; not providing convenience
functions; making the default mutable in ways that affect all users.

**Code example from source:**

```go
// net/http/client.go:108-109
// DefaultClient is the default [Client] and is used by [Get], [Head], and [Post].
var DefaultClient = &Client{}

// net/http/transport.go:47-58
var DefaultTransport RoundTripper = &Transport{
    Proxy:                 ProxyFromEnvironment,
    DialContext:           defaultTransportDialContext(&net.Dialer{
        Timeout:   30 * time.Second,
        KeepAlive: 30 * time.Second,
    }),
    ForceAttemptHTTP2:     true,
    MaxIdleConns:          100,
    IdleConnTimeout:       90 * time.Second,
    TLSHandshakeTimeout:   10 * time.Second,
    ExpectContinueTimeout: 1 * time.Second,
}
```

---

## 9. Functional Configuration via Method Chaining (Scanner Pattern)

**Pattern name:** Post-Construction Configuration via Methods

**Source citation:** [bufio/scan.go#L275](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/bufio/scan.go#L275)

**What it does:** After construction with `NewScanner`, optional configuration is
applied via methods (`Split`, `Buffer`) before the first call to `Scan`.

**Why:** Keeps the constructor minimal (only the required `io.Reader`). Optional
configuration is discoverable via methods. Panics if called after scanning starts
(enforcing a construction → configure → use lifecycle).

**Anti-pattern:** Trying to pass all options into the constructor; allowing
configuration changes mid-use that corrupt state.

**Code example from source:**

```go
// bufio/scan.go:275-293
// Buffer sets the initial buffer to use when scanning
// and the maximum size of buffer that may be allocated during scanning.
// ...
// Buffer panics if it is called after scanning has started.
func (s *Scanner) Buffer(buf []byte, max int) {
    if s.scanCalled {
        panic("Buffer called after Scan")
    }
    s.buf = buf
    s.maxTokenSize = max
}

// Split sets the split function for the [Scanner].
// ...
// Split panics if it is called after scanning has started.
func (s *Scanner) Split(split SplitFunc) {
    if s.scanCalled {
        panic("Split called after Scan")
    }
    s.split = split
}
```

<!-- PATTERN_COMPLETE -->
