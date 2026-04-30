# Go Configuration Patterns

Patterns for configuring Go types and services, extracted from the
Go standard library source.

**Source:** [golang/go](https://github.com/golang/go) at commit
[`17bd5ab`](https://github.com/golang/go/tree/17bd5ab8c650155dd2bd09f7005726552639eea0)

**Stats:** 33 Config/Options structs, 20 `With*` functions, 14
`Default*` exports, 9 `Register*` functions in public stdlib.

---

## 1. Zero-Value Usable Config Structs

Struct with sensible defaults when all fields are zero. Users only
set what they need to change.

### Source:

[crypto/tls/common.go#L566](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/crypto/tls/common.go#L566)

```go
// src/crypto/tls/common.go:566
type Config struct {
    // Rand provides the source of entropy for nonces and RSA blinding.
    // If Rand is nil, TLS uses the cryptographic random reader in package
    // crypto/rand.
    Rand io.Reader

    // Time returns the current time as the number of seconds since the epoch.
    // If Time is nil, TLS uses time.Now.
    Time func() time.Time

    // Certificates contains one or more certificate chains to present to the
    // other side of the connection.
    Certificates []Certificate
    // ...
}
```

Every field documents its zero-value behavior: "If nil, uses X."
The entire struct can be used as `&tls.Config{}` and it works.

### Why

Users only think about what they're changing. The stdlib handles
defaults internally. This eliminates "forgot to set field X" bugs
and makes constructor boilerplate unnecessary for simple cases.

### When to Use

**Triggers:**
- You're configuring a long-lived object (server, client, handler)
- Most users will only change 1-3 fields
- Each field has an obvious sensible default
- The struct will grow over time (backward compatibility via zero values)

**Example — before:**
```go
// Without zero-value defaults — every user must know about every field
func NewServer(addr string, handler http.Handler, readTimeout time.Duration,
    writeTimeout time.Duration, maxHeaderBytes int, tlsConfig *tls.Config,
    errorLog *log.Logger) *Server {
    return &Server{
        Addr: addr, Handler: handler, ReadTimeout: readTimeout,
        WriteTimeout: writeTimeout, MaxHeaderBytes: maxHeaderBytes,
        TLSConfig: tlsConfig, ErrorLog: errorLog,
    }
}

// Caller must specify everything:
s := NewServer(":8080", mux, 30*time.Second, 30*time.Second, 1<<20, nil, nil)
```

**Example — after:**
```go
// With zero-value usable struct — users set only what they care about
s := &http.Server{
    Addr:    ":8080",
    Handler: mux,
}
// ReadTimeout, WriteTimeout, MaxHeaderBytes all have documented defaults.
// TLSConfig, ErrorLog use stdlib defaults when nil.
```

### When NOT to Use

**Don't use this when:**
- There is no sensible default for a field (e.g., a database
  connection string — there's no "default" database)
- The zero value is dangerous (e.g., `Timeout: 0` meaning "no
  timeout" when you WANT a timeout by default)
- Users MUST make a conscious choice (use a constructor that
  forces the required parameters)

**Over-application example:**
```go
// Bad: zero value is DANGEROUS
type RetryConfig struct {
    MaxRetries int     // zero = no retries? or infinite retries?
    Timeout    time.Duration  // zero = no timeout = hang forever
}

// User creates: &RetryConfig{} — is that safe? Nobody knows.
```

**Better alternative:**
```go
// Required parameters in constructor, optional in struct
func NewRetrier(maxRetries int, timeout time.Duration) *Retrier {
    if maxRetries <= 0 {
        panic("maxRetries must be positive")
    }
    if timeout <= 0 {
        panic("timeout must be positive")
    }
    return &Retrier{maxRetries: maxRetries, timeout: timeout}
}
```

### Anti-pattern

```go
// DON'T: Config struct with fields that mean different things at zero
type Config struct {
    Port int  // 0 = random port? or invalid? or default 8080?
}

// DO: Document and handle zero explicitly
type Config struct {
    // Port specifies the TCP port to listen on.
    // If zero, defaults to 8080.
    Port int
}
```

---

## 2. Options Struct as Function Parameter

Pass an exported struct of optional settings to a constructor or
method.

### Source:

[log/slog/handler.go#L135](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/log/slog/handler.go#L135)

```go
// src/log/slog/handler.go:135
type HandlerOptions struct {
    // AddSource causes the handler to compute the source code position
    // of the log statement and add a SourceKey attribute to the output.
    AddSource bool

    // Level reports the minimum record level that will be logged.
    // The handler discards records with lower levels.
    // If Level is nil, the handler assumes LevelInfo.
    Level Leveler

    // ReplaceAttr is called to rewrite each non-group attribute
    // before it is logged.
    ReplaceAttr func(groups []string, a Attr) Attr
}
```

Usage:
```go
h := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    AddSource: true,
    Level:     slog.LevelDebug,
})
```

### Why

Separates required arguments (the writer) from optional configuration.
The struct can be `nil` (use all defaults) or partially filled.
Adding fields later doesn't break callers.

### When to Use

**Triggers:**
- You have 3+ optional parameters for a constructor
- Parameters are related and configure the same subsystem
- Users will often use defaults for most of them
- You need to be able to add options without breaking callers

**Example — before:**
```go
// Growing parameter list — every new option breaks all callers
func NewHandler(w io.Writer, addSource bool, level Level,
    replaceAttr func([]string, Attr) Attr) *Handler { ... }

// Every caller must pass all args even for defaults:
h := NewHandler(os.Stdout, false, LevelInfo, nil)
```

**Example — after:**
```go
// Options struct — nil means "all defaults"
func NewHandler(w io.Writer, opts *HandlerOptions) *Handler { ... }

// Simple case — no options needed:
h := slog.NewJSONHandler(os.Stdout, nil)

// Custom case — only set what you need:
h := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelDebug,
})
```

### When NOT to Use

**Don't use this when:**
- You have 1-2 optional parameters (just use direct params with
  zero-value semantics)
- The options change per-call, not per-instance (use functional
  options for per-call variation)
- Every user needs different options (nothing is truly "optional")

**Over-application example:**
```go
// Unnecessary: only one option, and it's always set
type ParseOptions struct {
    Format string
}

func Parse(input string, opts *ParseOptions) (*Result, error) {
    format := "json" // default
    if opts != nil {
        format = opts.Format
    }
    // ...
}

// Every caller writes: Parse(input, &ParseOptions{Format: "yaml"})
// This is MORE awkward than: Parse(input, "yaml")
```

**Better alternative:**
```go
// When there's really only one option, make it a parameter:
func Parse(input string, format string) (*Result, error) { ... }
```

---

## 3. Functional Options (With* Pattern)

Functions that return an opaque Options type, composed via variadic
parameters.

### Source:

[encoding/json/jsontext/options.go#L232](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/encoding/json/jsontext/options.go#L232)

```go
// src/encoding/json/jsontext/options.go:232
func WithIndent(indent string) Options {
    // ...
}

// src/encoding/json/jsontext/encode.go:91
func NewEncoder(w io.Writer, opts ...Options) *Encoder {
    e := new(Encoder)
    e.Reset(w, opts...)
    return e
}
```

Usage:
```go
enc := jsontext.NewEncoder(w,
    jsontext.WithIndent("  "),
    jsontext.WithByteLimit(1024*1024),
)
```

### Why

Options can be added over time without breaking callers. Each option
is self-documenting (the function name says what it does). Options
can be pre-composed and reused. The zero-option case reads cleanly:
`NewEncoder(w)`.

### When to Use

**Triggers:**
- The option set will grow over time (new features, new modes)
- Options should be individually composable and reusable
- Per-call configuration (not just per-instance)
- You want option names in the API surface (not struct field names)

**Example — before:**
```go
// Options struct works but gets unwieldy with many fields:
enc := json.NewEncoder(w, &json.EncoderOptions{
    Indent:       "  ",
    ByteLimit:    1024*1024,
    DepthLimit:   100,
    EscapeHTML:   true,
    SortMapKeys:  true,
})
```

**Example — after:**
```go
// Functional options — compose only what you need:
enc := json.NewEncoder(w,
    json.WithIndent("  "),
    json.WithByteLimit(1024*1024),
)

// Pre-compose for reuse:
var prettyJSON = []json.Options{
    json.WithIndent("  "),
    json.WithByteLimit(10*1024*1024),
}
enc := json.NewEncoder(w, prettyJSON...)
```

### When NOT to Use

**Don't use this when:**
- You have <3 options that won't grow (use direct parameters or
  an options struct)
- Options interact with each other in complex ways (a struct makes
  dependencies visible; functional options hide them)
- Users need to inspect/read back the configuration (options are
  typically write-only — you can set them but not query them)

**Over-application example:**
```go
// Overkill for 2 stable options:
func Connect(addr string, opts ...ConnectOption) (*Conn, error)

// Every caller writes:
conn, _ := Connect("localhost:5432",
    WithTimeout(5*time.Second),
    WithTLS(true),
)
// vs simply:
conn, _ := Connect("localhost:5432", 5*time.Second, true)
// or:
conn, _ := Connect("localhost:5432", &ConnectOptions{
    Timeout: 5*time.Second, TLS: true,
})
```

**Better alternative:** Use an options struct when:
- The option set is stable (<5 options)
- Users need to read options back
- Options interact (struct makes co-dependencies visible)

---

## 4. Package-Level Default Instances

A package provides a pre-configured, ready-to-use instance.

### Source:

[net/http/client.go#L109](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/net/http/client.go#L109)

```go
// src/net/http/client.go:109
var DefaultClient = &Client{}

// src/net/http/transport.go:47
var DefaultTransport RoundTripper = &Transport{
    Proxy:                 ProxyFromEnvironment,
    DialContext:           defaultTransportDialContext(&net.Dialer{
        Timeout:   30 * time.Second,
        KeepAlive: 30 * time.Second,
    }),
    ForceAttemptHTTP2:     true,
    MaxIdleConns:          100,
    IdleConnTimeout:       90 * time.Second,
    TLSHandshakeTimeout:  10 * time.Second,
    ExpectContinueTimeout: 1 * time.Second,
}

// src/log/slog/logger.go:55
func Default() *Logger { return defaultLogger.Load() }
```

### Why

Most programs need exactly one instance with default settings. The
package-level default eliminates boilerplate for the common case
while still allowing custom instances for tests or specialized needs.

### When to Use

**Triggers:**
- 90% of users will use the default configuration
- The type is safe for concurrent use
- Creating an instance requires non-trivial setup (transport pools,
  connection config)
- Package-level functions exist that delegate to the default

**Example — before:**
```go
// Without default — every caller must create and configure
client := &http.Client{
    Transport: &http.Transport{
        MaxIdleConns: 100,
        // ... 10 more fields for reasonable defaults
    },
}
resp, err := client.Get(url)
```

**Example — after:**
```go
// With package-level default — simple case is one line
resp, err := http.Get(url) // uses http.DefaultClient internally
```

### When NOT to Use

**Don't use this when:**
- Every user needs different configuration (no meaningful default)
- The instance holds resources that should be explicitly closed
- Global mutable state would cause test interference
- The type is NOT safe for concurrent use

**Over-application example:**
```go
// Bad: default database connection — there IS no universal default
var DefaultDB = MustConnect("postgres://localhost/mydb")
// What database? What credentials? This makes no sense as a default.
```

**Better alternative:**
```go
// Force users to be explicit about connections
db, err := sql.Open("postgres", connString)
```

### Anti-pattern

```go
// DON'T: Mutable default that tests can't isolate
var DefaultLogger = NewLogger(os.Stdout)
// Tests that modify DefaultLogger race with each other

// DO: Immutable default with replacement via function
func Default() *Logger { return defaultLogger.Load() }
func SetDefault(l *Logger) { defaultLogger.Store(l) }
// Atomic replacement — tests can use SetDefault safely
```

---

## 5. Init-Time Registration

Plugins/drivers register themselves in `init()`, looked up at runtime
by name.

### Source:

[database/sql/sql.go#L53](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/database/sql/sql.go#L53)

```go
// src/database/sql/sql.go:53
func Register(name string, driver driver.Driver) {
    driversMu.Lock()
    defer driversMu.Unlock()
    if driver == nil {
        panic("sql: Register driver is nil")
    }
    if _, dup := drivers[name]; dup {
        panic("sql: Register called twice for driver " + name)
    }
    drivers[name] = driver
}
```

Driver packages register in init:
```go
// In the driver package (e.g., github.com/lib/pq):
func init() {
    sql.Register("postgres", &Driver{})
}
```

Users import for side effects:
```go
import (
    "database/sql"
    _ "github.com/lib/pq" // registers "postgres" driver
)

db, err := sql.Open("postgres", connString)
```

### Why

Decouples the framework from implementations. The `database/sql`
package doesn't import any driver — drivers import IT and register.
New drivers can be added without changing the framework.

### When to Use

**Triggers:**
- You're building a framework/registry with pluggable backends
- Implementations are in separate packages (compile-time decoupling)
- Users choose implementations at link time (import selection)
- The set of implementations is open-ended

**Example — before:**
```go
// Hard-coded implementations — every new driver requires editing this
func Open(driverName, dataSourceName string) (*DB, error) {
    switch driverName {
    case "postgres":
        return openPostgres(dataSourceName)
    case "mysql":
        return openMySQL(dataSourceName)
    // Can't add new drivers without modifying this code
    }
}
```

**Example — after:**
```go
// Registration pattern — open to extension, closed to modification
func Open(driverName, dataSourceName string) (*DB, error) {
    driver, ok := drivers[driverName]
    if !ok {
        return nil, fmt.Errorf("sql: unknown driver %q", driverName)
    }
    return driver.Open(dataSourceName)
}
// New drivers register themselves — zero changes to this code.
```

### When NOT to Use

**Don't use this when:**
- You control all implementations (use interfaces directly)
- Registration order matters (init order is non-deterministic across
  packages)
- You need to test without global state pollution
- The "plugin" needs configuration beyond just existing

**Over-application example:**
```go
// Over-engineered for an internal app with 2 known implementations
var handlers = map[string]Handler{}
func Register(name string, h Handler) { handlers[name] = h }

func init() { Register("json", &JSONHandler{}) }
func init() { Register("xml", &XMLHandler{}) }

// These are always the same two. Just use a constructor:
func NewHandler(format string) Handler {
    switch format {
    case "json": return &JSONHandler{}
    case "xml":  return &XMLHandler{}
    }
}
```

### Anti-pattern

```go
// DON'T: Registration without duplicate detection
func Register(name string, d Driver) {
    drivers[name] = d  // silently overwrites — last-import-wins
}

// DO: Panic on duplicate (from database/sql)
if _, dup := drivers[name]; dup {
    panic("sql: Register called twice for driver " + name)
}
```

---

## 6. Context-Carried Configuration (WithValue)

Request-scoped configuration passed through context.Context.

### Source:

[context/context.go#L728](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/context/context.go#L728)

```go
// src/context/context.go:728
func WithValue(parent Context, key, val any) Context {
    if parent == nil {
        panic("cannot create context from nil parent")
    }
    if key == nil {
        panic("nil key")
    }
    if !reflectlite.TypeOf(key).Comparable() {
        panic("key is not comparable")
    }
    return &valueCtx{parent, key, val}
}
```

Usage with unexported key type (the idiom):
```go
// src/net/http/httptrace/trace.go:33
type clientEventContextKey struct{}

func WithClientTrace(ctx context.Context, trace *ClientTrace) context.Context {
    // ...
    return context.WithValue(ctx, clientEventContextKey{}, trace)
}
```

### Why

Passes request-scoped data through call chains without adding
parameters to every function signature. The unexported key type
prevents collision between packages.

### When to Use

**Triggers:**
- Data is request-scoped (trace ID, auth token, deadline)
- Data must cross package boundaries without coupling them
- The data is "ambient" (needed by middleware/infrastructure, not
  business logic)
- Adding a parameter to every function in the chain is impractical

**Example — before:**
```go
// Propagating trace ID through 5 layers of function calls:
func HandleRequest(traceID string, r *Request) {
    result := processOrder(traceID, r.Order)
    notify(traceID, result)
}
func processOrder(traceID string, o Order) Result {
    validated := validate(traceID, o)
    return persist(traceID, validated)
}
// traceID is threaded through EVERY function — pollutes all signatures
```

**Example — after:**
```go
type traceIDKey struct{}

func WithTraceID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, traceIDKey{}, id)
}

func HandleRequest(ctx context.Context, r *Request) {
    result := processOrder(ctx, r.Order)
    notify(ctx, result)
}
// traceID travels invisibly in ctx — only extracted where needed
```

### When NOT to Use

**Don't use this when:**
- The data is required for correctness (make it an explicit param —
  context values are invisible, easy to forget)
- The data is needed in EVERY function (it's not ambient, it's core)
- You're using it to avoid adding a parameter to 2-3 functions
  (that's not enough pain to justify the indirection)
- The data is mutable (context values are immutable by convention)

**Over-application example:**
```go
// Bad: config that EVERY function needs — should be a field
type serverConfig struct{}

func handleRequest(ctx context.Context, r *Request) {
    cfg := ctx.Value(serverConfig{}).(*Config)
    // Every handler digs into context for core config
    // This is just dependency injection with extra steps and no type safety
}
```

**Better alternative:**
```go
// Make it a field on the server/handler:
type Server struct {
    config *Config
}
func (s *Server) handleRequest(r *Request) {
    // s.config is always there, typed, visible
}
```

### Anti-pattern

```go
// DON'T: Exported key type (allows collision)
var TraceKey = "trace-id"  // any package can use this string

// DO: Unexported struct type (package-scoped, collision-proof)
type traceKey struct{}
```

---

## 7. Builder Pattern via Method Chaining

Not a stdlib pattern — but its ABSENCE is instructive.

### Source:

The Go stdlib does NOT use builder patterns. Zero instances of
method chaining for configuration exist in the public API.

### Why

Go prefers struct literals for construction. Builders hide what's
being set, make types non-trivially copyable, and create an
awkward "build phase" vs "use phase" distinction.

### When to Use

**Almost never in Go.** The only legitimate case:
- Building immutable objects where the construction process is
  genuinely complex (>10 steps with conditionals)
- Even then, prefer a config struct + constructor.

### When NOT to Use

**Don't use this when:**
- A struct literal works (always try struct literal first)
- You're porting patterns from Java/C# (they use builders because
  they lack struct literals with named fields)
- You want "fluent" APIs (Go culture values explicit over clever)

**Over-application example:**
```go
// Java-brain in Go:
server := NewServerBuilder().
    WithAddr(":8080").
    WithTimeout(30 * time.Second).
    WithHandler(mux).
    WithTLS(cert, key).
    Build()

// In Go, this is just:
server := &http.Server{
    Addr:         ":8080",
    ReadTimeout:  30 * time.Second,
    Handler:      mux,
    TLSConfig:    tlsConfig,
}
// Clearer, no hidden state, no build/use phase split.
```

---

## 8. Exported Fields with Documented Nil Behavior

Config fields that accept function values, with nil meaning "use
default behavior."

### Source:

[crypto/tls/common.go#L572](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/crypto/tls/common.go#L572)

```go
// src/crypto/tls/common.go:572
// Time returns the current time as the number of seconds since the epoch.
// If Time is nil, TLS uses time.Now.
Time func() time.Time
```

Also: [log/slog/handler.go#L169](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/log/slog/handler.go#L169)
```go
// ReplaceAttr is called to rewrite each non-group attribute before
// it is logged. If ReplaceAttr returns a zero Attr, the attribute
// is discarded.
ReplaceAttr func(groups []string, a Attr) Attr
```

### Why

Allows injection of custom behavior without requiring interfaces.
The nil check is simpler than defining an interface, implementing a
default, and wiring it. Good for hooks where most users want the
default but advanced users need to customize.

### When to Use

**Triggers:**
- Optional behavior customization (not every user needs it)
- The "interface" would have exactly one method
- Default behavior is obvious (time.Now, os.Stderr, etc.)
- Function signature is stable

**Example — before:**
```go
// Interface for one method — ceremony for no gain
type TimeProvider interface {
    Now() time.Time
}
type defaultTimeProvider struct{}
func (d defaultTimeProvider) Now() time.Time { return time.Now() }

type Config struct {
    TimeProvider TimeProvider  // required interface, must set
}
```

**Example — after:**
```go
// Function field — nil means default
type Config struct {
    // Time returns the current time.
    // If nil, time.Now is used.
    Time func() time.Time
}

// Usage in implementation:
func (c *Config) now() time.Time {
    if c.Time != nil {
        return c.Time()
    }
    return time.Now()
}
```

### When NOT to Use

**Don't use this when:**
- The function has side effects that need lifecycle management
  (use an interface with Close)
- Multiple methods are needed together (use an interface)
- The function needs to carry state (use a struct implementing
  an interface)

---

## 9. Immutable-After-Use Convention

Config structs that must not be modified after being passed to a
constructor.

### Source:

[crypto/tls/common.go#L566](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/crypto/tls/common.go#L566)

```go
// A Config structure is used to configure a TLS client or server.
// After one has been passed to a TLS function it must not be modified.
// A Config may be reused; the tls package will also not modify it.
type Config struct { ... }
```

### Why

Avoids defensive copying of large structs. The TLS config has 30+
fields — copying on every handshake would waste memory. Instead,
the contract is: "you give it to us, you stop touching it."

### When to Use

**Triggers:**
- Config struct is large (>5 fields)
- Copying would be expensive (contains slices, maps, or pointers)
- The configured object is long-lived (server, pool, transport)
- Concurrent access to config is possible

**Example — before:**
```go
// Defensive copy — safe but expensive for large configs
func NewServer(cfg Config) *Server {
    cfgCopy := cfg  // copies all fields
    return &Server{config: &cfgCopy}
}
```

**Example — after:**
```go
// Document immutability constraint — no copy needed
// "After one has been passed to NewServer it must not be modified."
func NewServer(cfg *Config) *Server {
    return &Server{config: cfg}  // shared reference, caller must not mutate
}
```

### When NOT to Use

**Don't use this when:**
- The struct is small and cheap to copy (just copy it)
- Users frequently need to create variations (provide a `Clone()`
  method instead)
- The contract is hard to enforce (tests can't catch violations)

---

## 10. Clone for Config Variation

Provide a `Clone()` method when users need to create modified copies
of immutable-after-use configs.

### Source:

[crypto/tls/common.go#L925](https://github.com/golang/go/blob/17bd5ab8c650155dd2bd09f7005726552639eea0/src/crypto/tls/common.go#L925) (tls.Config.Clone)

```go
// Clone returns a shallow clone of c or nil if c is nil. It is safe to clone
// a Config that is being used concurrently by a TLS client or server.
func (c *Config) Clone() *Config {
    if c == nil {
        return nil
    }
    c.mutex.Lock()
    defer c.mutex.Unlock()
    return &Config{
        Rand:                c.Rand,
        Time:                c.Time,
        Certificates:        c.Certificates,
        // ... all fields copied
    }
}
```

### Why

When a config is immutable-after-use but users need variations
(e.g., same TLS config but different ServerName for each host),
Clone gives them a safe way to fork without modifying the original.

### When to Use

**Triggers:**
- You have immutable-after-use config structs
- Users need slight variations of a base config
- The struct contains reference types (slices, maps) that need
  safe copying
- The struct has unexported fields or a mutex

**Example — before:**
```go
// Without Clone — users attempt (broken) manual copy
baseCfg := &tls.Config{MinVersion: tls.VersionTLS12}
// Can't just: hostCfg := *baseCfg (unexported fields, shared slices)
```

**Example — after:**
```go
baseCfg := &tls.Config{MinVersion: tls.VersionTLS12}
hostCfg := baseCfg.Clone()
hostCfg.ServerName = "example.com"
```

### When NOT to Use

**Don't use this when:**
- The struct has no unexported fields and no reference types
  (plain struct copy `*s` works fine)
- Users rarely need variations (one config for the whole app)

---

## Summary: Configuration Decision Tree

```
Is the configuration per-instance or per-call?
├── Per-instance → struct-based patterns (#1, #2, #8, #9, #10)
│   ├── <3 options? → Direct parameters
│   ├── 3-10 options, stable? → Options struct (#2)
│   ├── Long-lived, most fields have defaults? → Zero-value config (#1)
│   ├── Must not mutate after use? → Immutable convention (#9) + Clone (#10)
│   └── Hook/callback injection? → Function fields (#8)
├── Per-call → functional patterns (#3, #6)
│   ├── Options will grow? → Functional options / With* (#3)
│   └── Request-scoped ambient data? → Context values (#6)
└── Framework/plugin boundary? → Registration (#5)
    └── Global default for common case? → Default instance (#4)
```

**Key principle:** Start with the simplest pattern that works. Only
reach for functional options or registration when you have evidence
the option set is growing or implementations are external.

See also:
- [interfaces.md](interfaces.md) — Accept interfaces, return structs
- [api-conventions.md](api-conventions.md) — Backward compatibility

<!-- PATTERN_COMPLETE -->
