# Go Error Handling Patterns

Patterns extracted from the Go standard library source code.

---

## 1. Sentinel Errors

### Source: `src/io/io.go:40-43` (EOF), `src/errors/errors.go:81-83` (ErrUnsupported)

```go
// src/io/io.go:40-43
// EOF is the error returned by Read when no more input is available.
// (Read must return EOF itself, not an error wrapping EOF,
// because callers will test for EOF using ==.)
var EOF = errors.New("EOF")

// src/io/io.go:47-49
var ErrUnexpectedEOF = errors.New("unexpected EOF")
```

```go
// src/errors/errors.go:81-83
var ErrUnsupported = New("unsupported operation")
```

### Why

Sentinel errors are package-level values that represent specific, well-known error conditions. They enable callers to test for specific failures:

```go
if err == io.EOF {
    // end of input — not an error, just done
}
```

**Critical rule from io.EOF's doc comment**: Read must return EOF itself, **not an error wrapping EOF**, because callers test for it with `==`. This is the distinction between sentinel errors (identity-checked) and wrapped errors (tree-checked).

### When to Use

**Triggers:**
- You have a specific, well-known failure condition callers need to check by identity
- Multiple packages compare against the same error value (`io.EOF`, `sql.ErrNoRows`)
- The error represents a **state** ("end of stream", "not found"), not a bug

**Example — before:**
```go
func fetchUser(id int) (*User, error) {
    row := db.QueryRow("SELECT ...")
    var u User
    err := row.Scan(&u.Name)
    if err != nil {
        return nil, fmt.Errorf("user not found") // caller can't distinguish "not found" from "db down"
    }
    return &u, nil
}
```

**Example — after:**
```go
var ErrUserNotFound = errors.New("users: not found")

func fetchUser(id int) (*User, error) {
    row := db.QueryRow("SELECT ...")
    var u User
    err := row.Scan(&u.Name)
    if errors.Is(err, sql.ErrNoRows) {
        return nil, ErrUserNotFound // sentinel: callers can test with errors.Is
    }
    if err != nil {
        return nil, fmt.Errorf("fetchUser: %w", err)
    }
    return &u, nil
}
```

### Anti-pattern

```go
// DON'T: Use string matching
if err.Error() == "EOF" { ... }  // fragile, not guaranteed

// DON'T: Return a new error each time for sentinel conditions
func Read() error {
    return errors.New("EOF")  // every call creates a new value, can't compare with ==
}
```

---

## 2. errors.New — Minimal Error Construction

### Source: `src/errors/errors.go:62-69`

```go
// src/errors/errors.go:62-64
func New(text string) error {
    return &errorString{text}
}

// src/errors/errors.go:66-69
type errorString struct {
    s string
}

func (e *errorString) Error() string {
    return e.s
}
```

### Why

`errors.New` returns a pointer to a private struct. Each call creates a **distinct value** even with identical text — this is intentional for sentinel errors. Two calls to `errors.New("foo")` produce different errors (`!=`).

The `error` interface itself is the smallest possible:
```go
type error interface {
    Error() string
}
```

### Anti-pattern

```go
// DON'T: Export the error type
type MyError string  // callers can create values that accidentally == your sentinels

// DON'T: Use plain strings as errors
func doThing() error {
    return "something failed"  // doesn't implement error interface
}
```

---

## 3. Error Wrapping with fmt.Errorf and %w

### Source: `src/fmt/errors.go:13-23`, `src/fmt/errors.go:70-80`

```go
// src/fmt/errors.go:13-23
// Errorf formats according to a format specifier and returns the string
// as a value that satisfies error.
//
// If the format specifier includes a %w verb with an error operand,
// the returned error will implement an Unwrap method returning the operand.
// If there is more than one %w verb, the returned error will implement an
// Unwrap method returning a []error containing all the %w operands.
func Errorf(format string, a ...any) (err error) { ... }

// src/fmt/errors.go:70-80
type wrapError struct {
    msg string
    err error
}

func (e *wrapError) Error() string {
    return e.msg
}

func (e *wrapError) Unwrap() error {
    return e.err
}
```

### Why

`%w` creates an error chain: the returned error wraps the original. `errors.Is` and `errors.As` walk this chain. Use `%w` when callers should be able to inspect the underlying cause.

```go
// Wraps: callers can detect the original error
return fmt.Errorf("open config: %w", err)

// Does NOT wrap: hides the original error
return fmt.Errorf("open config: %v", err)
```

### When to Use

**Triggers:**
- You're adding context to an error before returning it up the call stack
- The caller's error message would be meaningless without knowing *what* operation failed
- You have a chain of function calls and want a readable error trail: `"open config: read file: permission denied"`

**Example — before:**
```go
func loadConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, err // caller sees "open /etc/app.conf: permission denied" — no context about WHO called ReadFile
    }
    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        return nil, err // caller can't tell if this was a read error or a parse error
    }
    return &cfg, nil
}
```

**Example — after:**
```go
func loadConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("load config: %w", err) // wraps: callers can errors.Is(err, os.ErrNotExist)
    }
    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        return nil, fmt.Errorf("load config: parse %s: %v", path, err) // %v: hides internal JSON error type
    }
    return &cfg, nil
}
```

### When to use %w vs %v

- **%w**: When the wrapped error is part of your API contract. Callers can depend on it.
- **%v**: When you want to include the error text but NOT let callers depend on the underlying type. Use for implementation details.

### Anti-pattern

```go
// DON'T: Lose the original error
return errors.New("failed to open config")  // original error vanished

// DON'T: Wrap errors that aren't part of your contract with %w
return fmt.Errorf("internal: %w", internalErr)  // now callers depend on internalErr's type
```

---

## 4. errors.Is — Checking Error Identity Through Chains

### Source: `src/errors/wrap.go:30-44`

```go
// src/errors/wrap.go:30-44
func Is(err, target error) bool {
    if err == nil || target == nil {
        return err == target
    }
    isComparable := reflectlite.TypeOf(target).Comparable()
    return is(err, target, isComparable)
}

func is(err, target error, targetComparable bool) bool {
    for {
        if targetComparable && err == target {
            return true
        }
        if x, ok := err.(interface{ Is(error) bool }); ok && x.Is(target) {
            return true
        }
        switch x := err.(type) {
        case interface{ Unwrap() error }:
            err = x.Unwrap()
            if err == nil {
                return false
            }
        case interface{ Unwrap() []error }:
            for _, err := range x.Unwrap() {
                if is(err, target, targetComparable) {
                    return true
                }
            }
            return false
        default:
            return false
        }
    }
}
```

### Why

`errors.Is` walks the entire error tree (depth-first). It checks:
1. Direct equality (`err == target`)
2. Custom `Is(error) bool` method on the error
3. Then unwraps and recurses

This means wrapped errors are transparent:
```go
err := fmt.Errorf("config: %w", os.ErrNotExist)
errors.Is(err, os.ErrNotExist)  // true! walks the chain
```

### Anti-pattern

```go
// DON'T: Use == directly on potentially-wrapped errors
if err == os.ErrNotExist { ... }  // fails if err wraps ErrNotExist

// DO: Use errors.Is
if errors.Is(err, os.ErrNotExist) { ... }  // works through wrapping
```

---

## 5. errors.As — Extracting Error Types Through Chains

### Source: `src/errors/wrap.go:96-120`

```go
// src/errors/wrap.go:96-120
func As(err error, target any) bool {
    if err == nil {
        return false
    }
    // ... validation ...
    targetType := typ.Elem()
    return as(err, target, val, targetType)
}
```

The `as` function (line 121+) walks the tree checking `AssignableTo` and custom `As(any) bool` methods.

### Why

Extract specific error types from wrapped chains:

```go
var pathErr *fs.PathError
if errors.As(err, &pathErr) {
    fmt.Println("failed path:", pathErr.Path)
}
```

### Go 1.24+: errors.AsType (generic version)

From `src/errors/errors.go:48-56` doc:
```go
if perr, ok := errors.AsType[*fs.PathError](err); ok {
    fmt.Println(perr.Path)
}
```

### Anti-pattern

```go
// DON'T: Type-assert directly on potentially-wrapped errors
if pathErr, ok := err.(*fs.PathError); ok { ... }  // fails if wrapped

// DO: Use errors.As
var pathErr *fs.PathError
if errors.As(err, &pathErr) { ... }  // works through wrapping
```

---

## 6. errors.Join — Multi-Error Aggregation

### Source: `src/errors/join.go:20-39`

```go
// src/errors/join.go:20-39
func Join(errs ...error) error {
    n := 0
    for _, err := range errs {
        if err != nil {
            n++
        }
    }
    if n == 0 {
        return nil
    }
    e := &joinError{
        errs: make([]error, 0, n),
    }
    for _, err := range errs {
        if err != nil {
            e.errs = append(e.errs, err)
        }
    }
    return e
}
```

The `joinError` type implements `Unwrap() []error`, making both `Is` and `As` traverse correctly.

### Why

For operations that can produce multiple errors (closing multiple resources, validating multiple fields), `Join` collects them into a single error.

```go
var errs []error
errs = append(errs, closeDB())
errs = append(errs, closeCache())
return errors.Join(errs...)  // nil if all nil
```

### When to Use

**Triggers:**
- You're closing/cleaning up multiple resources and each can fail independently
- A validation function checks multiple fields and you want ALL errors, not just the first
- You're running parallel operations and collecting errors from each

**Example — before:**
```go
func cleanup(db *sql.DB, cache *redis.Client, file *os.File) error {
    if err := db.Close(); err != nil {
        return err // stops here — cache and file leak!
    }
    if err := cache.Close(); err != nil {
        return err // file still leaks
    }
    return file.Close()
}
```

**Example — after:**
```go
func cleanup(db *sql.DB, cache *redis.Client, file *os.File) error {
    return errors.Join(
        db.Close(),
        cache.Close(),
        file.Close(),
    ) // nil if all nil; contains all failures otherwise
}
```

### Anti-pattern

```go
// DON'T: Return only the last error
var lastErr error
for _, r := range resources {
    if err := r.Close(); err != nil {
        lastErr = err  // silently loses previous errors
    }
}
return lastErr
```

---

## 7. Custom Is() Method — Equivalence Classes

### Source: `src/errors/wrap.go:42-44` (doc comment), `src/context/context.go:177-179`

From the `errors.Is` doc:
```go
// An error type might provide an Is method so it can be treated as
// equivalent to an existing error. For example, if MyError defines
//
//   func (m MyError) Is(target error) bool { return target == fs.ErrExist }
//
// then Is(MyError{}, fs.ErrExist) returns true.
```

Real example from context:
```go
// src/context/context.go:177-179
type deadlineExceededError struct{}

func (deadlineExceededError) Error() string   { return "context deadline exceeded" }
func (deadlineExceededError) Timeout() bool   { return true }
func (deadlineExceededError) Temporary() bool { return true }
```

### Why

Custom `Is` methods let you define error equivalence beyond pointer identity. A `syscall.Errno` can match `fs.ErrExist` through its `Is` method, bridging OS-specific error codes to portable sentinel errors.

### Anti-pattern

```go
// DON'T: Make Is() too broad
func (e MyError) Is(target error) bool {
    return true  // matches everything — defeats the purpose
}
```

---

## 8. Error Wrapping in Custom Types (Unwrap pattern)

### Source: `src/encoding/json/encode.go:276-293`

```go
// src/encoding/json/encode.go:276-282
type MarshalerError struct {
    Type       reflect.Type
    Err        error
    sourceFunc string
}

// src/encoding/json/encode.go:293
func (e *MarshalerError) Unwrap() error { return e.Err }
```

### Why

Custom error types carry structured data (which type failed, which function) while still participating in the error chain via `Unwrap()`. Callers can use `errors.As` to extract the `MarshalerError` AND use `errors.Is` to check the underlying cause.

### Pattern Template

```go
type OpError struct {
    Op   string
    Path string
    Err  error
}

func (e *OpError) Error() string {
    return e.Op + " " + e.Path + ": " + e.Err.Error()
}

func (e *OpError) Unwrap() error {
    return e.Err
}
```

### Anti-pattern

```go
// DON'T: Store error as string
type MyError struct {
    Message string  // lost the original error!
}

// DON'T: Forget to implement Unwrap
type MyError struct {
    Err error  // has the error but errors.Is can't traverse it
}
func (e *MyError) Error() string { return e.Err.Error() }
// Missing: func (e *MyError) Unwrap() error { return e.Err }
```

---

## 9. ErrUnsupported — Feature Detection via Errors

### Source: `src/errors/errors.go:76-83`

```go
// src/errors/errors.go:76-83
// ErrUnsupported indicates that a requested operation cannot be performed,
// because it is unsupported.
//
// Functions and methods should not return this error but should instead
// return an error including appropriate context that satisfies
//
//   errors.Is(err, errors.ErrUnsupported)
//
// either by directly wrapping ErrUnsupported or by implementing an Is method.
var ErrUnsupported = New("unsupported operation")
```

### Why

This pattern separates "what happened" (detailed context) from "what kind of failure" (sentinel identity). Return a rich error that *wraps* or *matches* the sentinel:

```go
return fmt.Errorf("chmod %s: %w", path, errors.ErrUnsupported)
```

### Anti-pattern

```go
// DON'T: Return the sentinel directly without context
return errors.ErrUnsupported  // no info about what operation or why
```

---

## 10. Error String Conventions

### Source: `src/net/http/server.go:39-56`

```go
// src/net/http/server.go:39-56
var (
    ErrHijacked       = errors.New("http: connection has been hijacked")
    ErrContentLength  = errors.New("http: wrote more than the declared Content-Length")
)
```

### Convention: Error String Format

```
package: description
```

- Lowercase (no capital first letter)
- No trailing punctuation
- Package prefix for disambiguation

### Anti-pattern

```go
// DON'T: Capitalize error strings
errors.New("Connection has been hijacked")

// DON'T: End with punctuation
errors.New("connection failed.")

// DON'T: Include redundant "error" word
errors.New("http error: connection failed")  // it's already an error
```

---

## Summary: Error Handling Decision Tree

```
Is this a specific, well-known condition?
├── YES → Sentinel error (package-level var)
│         └── Should callers detect it? → errors.Is
└── NO → Is there structured info to convey?
         ├── YES → Custom error type with Unwrap()
         │         └── Should callers extract it? → errors.As
         └── NO → fmt.Errorf with %w (wraps) or %v (doesn't wrap)
```

| When to... | Use |
|---|---|
| Create a well-known error condition | `var ErrFoo = errors.New("pkg: foo")` |
| Add context while preserving cause | `fmt.Errorf("doing X: %w", err)` |
| Add context, hide internal cause | `fmt.Errorf("doing X: %v", err)` |
| Check for a specific condition | `errors.Is(err, ErrFoo)` |
| Extract structured error data | `errors.As(err, &target)` |
| Aggregate multiple errors | `errors.Join(err1, err2)` |
| Make custom types traversable | Implement `Unwrap() error` |
| Define error equivalence | Implement `Is(error) bool` |
