# Advanced Go Testing Patterns

Patterns extracted from the Go standard library (`src/net/http/`, `src/encoding/json/`, `src/testing/`) and Kubernetes source code.

---

## 1. Table-Driven Tests

The canonical Go test style. Every Go stdlib test file uses this pattern.

### Pattern Name: Anonymous Struct Test Table

**Source:** `/tmp/go-src/src/net/http/header_test.go` lines 17-108

**What they do:** Define test cases as a slice of anonymous structs, iterate with a range loop.

**Why:** Eliminates repetition, makes adding cases trivial, keeps the assertion logic in one place. Every test case gets the same verification path — no "special" cases hidden in different code paths.

**Anti-pattern:** Writing individual assertions for each case, or copy-pasting test functions that differ by one input.

**Code example (stdlib):**
```go
var headerWriteTests = []struct {
	h        Header
	exclude  map[string]bool
	expected string
}{
	{Header{}, nil, ""},
	{
		Header{
			"Content-Type":   {"text/html; charset=UTF-8"},
			"Content-Length": {"0"},
		},
		nil,
		"Content-Length: 0\r\nContent-Type: text/html; charset=UTF-8\r\n",
	},
	// ... more cases
}

func TestHeaderWrite(t *testing.T) {
	var buf strings.Builder
	for i, test := range headerWriteTests {
		test.h.WriteSubset(&buf, test.exclude)
		if buf.String() != test.expected {
			t.Errorf("#%d:\n got: %q\nwant: %q", i, buf.String(), test.expected)
		}
		buf.Reset()
	}
}
```

---

### Pattern Name: Named Table Tests with t.Run (Subtests)

**Source:** `/tmp/go-src/src/encoding/json/encode_test.go` lines 285-320, `/tmp/go-src/src/encoding/json/scanner_test.go` lines 30-50

**What they do:** Combine table-driven tests with `t.Run` for named subtests. Use a `CaseName` struct that captures file/line for error reporting.

**Why:** Each case gets its own subtest name — visible in `go test -v`, filterable with `-run`, and individually re-runnable. The `CaseName`/`Where` pattern provides precise file:line for failures even in large test tables.

**Anti-pattern:** Using index-only identification (hard to find which case failed), or creating separate `TestFoo_Case1`, `TestFoo_Case2` functions.

**Code example (stdlib):**
```go
func TestValid(t *testing.T) {
	tests := []struct {
		CaseName
		data string
		ok   bool
	}{
		{Name(""), `foo`, false},
		{Name(""), `}{`, false},
		{Name(""), `{}`, true},
		{Name("StringDoubleEscapes"), `{"foo":"bar"}`, true},
	}
	for _, tt := range tests {
		t.Run(tt.Name, func(t *testing.T) {
			if ok := Valid([]byte(tt.data)); ok != tt.ok {
				t.Errorf("%s: Valid(`%s`) = %v, want %v", tt.Where, tt.data, ok, tt.ok)
			}
		})
	}
}
```

---

### Pattern Name: CaseName with Caller Position Tracking

**Source:** `/tmp/go-src/src/encoding/json/internal/jsontest/testcase.go` lines 18-37

**What they do:** Create a helper type that captures the caller's file:line at the point of test case declaration, so error messages point back to the exact test case definition.

**Why:** In a 1000-entry test table, `t.Errorf` points to the assertion line (same for all cases). CaseName makes failures point to the case definition.

**Code example (stdlib):**
```go
type CaseName struct {
	Name  string
	Where CasePos
}

func Name(s string) (c CaseName) {
	c.Name = s
	runtime.Callers(2, c.Where.pc[:])
	return c
}

type CasePos struct{ pc [1]uintptr }

func (pos CasePos) String() string {
	frames := runtime.CallersFrames(pos.pc[:])
	frame, _ := frames.Next()
	return fmt.Sprintf("%s:%d", path.Base(frame.File), frame.Line)
}
```

---

## 2. Test Helper Patterns

### Pattern Name: t.Helper() for Clean Stack Traces

**Source:** `/tmp/go-src/src/testing/testing.go` lines 1415-1435

**What they do:** Call `t.Helper()` as the first line in any test utility function. This marks the function as a helper, so test failure messages report the caller's line instead of the helper's line.

**Why:** Without `t.Helper()`, every failure in a helper function points to the helper itself, not the test case that triggered the failure. Makes debugging test failures require reading the full stack.

**Anti-pattern:** Writing test utilities that call `t.Fatal`/`t.Error` without marking themselves as helpers.

**Code example (stdlib):**
```go
// From net/http/clientserver_test.go lines 100-131
func run[T TBRun[T]](t T, f func(t T, mode testMode), opts ...any) {
	t.Helper()
	modes := []testMode{http1Mode, http2Mode, http3Mode}
	parallel := true
	for _, opt := range opts {
		switch opt := opt.(type) {
		case []testMode:
			modes = opt
		case testNotParallelOpt:
			parallel = false
		default:
			t.Fatalf("unknown option type %T", opt)
		}
	}
	// ...
	for _, mode := range modes {
		t.Run(string(mode), func(t T) {
			t.Helper()
			// ...
			f(t, mode)
		})
	}
}
```

---

### Pattern Name: *testing.T as First Argument to Helpers

**Source:** `/tmp/go-src/src/net/http/serve_test.go` lines 4555-4580

**What they do:** Pass `*testing.T` (or `testing.TB`) as the first argument to test helper functions, making the dependency on the test context explicit.

**Why:** The test object provides `Fatal`, `Error`, `Log`, `Helper`, `Cleanup` — everything a helper needs for reporting. Accepting it as a parameter (rather than capturing it in a closure) makes helpers reusable across tests.

**Code example (stdlib):**
```go
mustGet := func(url string, headers ...string) {
	t.Helper()
	req, err := NewRequest("GET", url, nil)
	if err != nil {
		t.Fatal(err)
	}
	for len(headers) > 0 {
		req.Header.Add(headers[0], headers[1])
		headers = headers[2:]
	}
	res, err := c.Do(req)
	if err != nil {
		t.Errorf("Error fetching %s: %v", url, err)
		return
	}
	_, err = io.ReadAll(res.Body)
	defer res.Body.Close()
}
```

---

## 3. t.Cleanup vs defer

### Pattern Name: t.Cleanup for Test-Scoped Resources

**Source:** `/tmp/go-src/src/testing/testing.go` lines 1439-1468, `/tmp/go-src/src/net/http/clientserver_test.go` lines 120-127

**What they do:** Use `t.Cleanup(fn)` instead of `defer` for resource cleanup in tests.

**Why:**
1. `defer` runs at the end of the *function*, not the *test*. In subtests launched with `t.Run`, a `defer` in a helper function runs when the helper returns — not when the subtest completes.
2. `t.Cleanup` runs after the test AND all its subtests finish — guaranteeing resources are available for the full test lifetime.
3. `t.Cleanup` is called in reverse order (LIFO), matching `defer` semantics but scoped to the test.

**Anti-pattern:** Using `defer` for cleanup in test setup functions that return before the test finishes, or in subtests where timing matters.

**Code example (stdlib):**
```go
// From net/http/clientserver_test.go
func run[T TBRun[T]](t T, f func(t T, mode testMode), opts ...any) {
	// ...
	for _, mode := range modes {
		t.Run(string(mode), func(t T) {
			t.Cleanup(func() {
				afterTest(t)  // Goroutine leak detection — runs AFTER subtest body completes
			})
			f(t, mode)
		})
	}
}
```

---

## 4. testdata/ Directory Pattern

### Pattern Name: testdata/ for Test Fixtures

**Source:** `/tmp/go-src/src/net/http/testdata/` (contains `file`, `index.html`, `style.css`), `/tmp/go-src/src/net/http/fs_test.go` line 38

**What they do:** Store test fixtures in a `testdata/` directory adjacent to the test files. Reference them with relative paths like `"testdata/file"`.

**Why:**
1. `go build` ignores `testdata/` directories — they never end up in production binaries.
2. `go test` runs with the package directory as CWD — relative paths to `testdata/` work reliably.
3. Fixtures are version-controlled alongside the code they test.
4. Separates test data from test logic.

**Anti-pattern:** Embedding large test fixtures as string literals in test files, or referencing absolute paths.

**Code example (stdlib):**
```go
// From net/http/fs_test.go line 38
const testFile = "testdata/file"

// Usage in test:
ServeFile(w, r, "testdata/file")
```

---

## 5. Golden File Testing

### Pattern Name: Golden Files with -update Flag

**Source:** `/tmp/go-src/src/cmd/gofmt/gofmt_test.go` lines 18, 113-138

**What they do:** Compare test output against `.golden` files. Provide a `-update` flag that regenerates golden files from current output when behavior intentionally changes.

**Why:**
1. Tests complex output (formatted code, generated HTML, serialized data) without embedding it in test code.
2. The `-update` flag makes intentional changes easy: run `go test -update`, review the diff, commit.
3. Golden files serve as documentation of expected behavior.
4. Reviewers can see exactly what output changed in diffs.

**Anti-pattern:** Comparing against inline expected strings that span 50+ lines, or manually constructing expected output.

**Code example (stdlib):**
```go
var update = flag.Bool("update", false, "update .golden files")

func runTest(t *testing.T, in, out string) {
	// ... produce actual output ...

	expected, err := os.ReadFile(out)
	if err != nil {
		t.Error(err)
		return
	}

	if got := buf.Bytes(); !bytes.Equal(got, expected) {
		if *update {
			if in != out {
				if err := os.WriteFile(out, got, 0666); err != nil {
					t.Error(err)
				}
				return
			}
		}
		t.Errorf("(gofmt %s) != %s\n%s", in, out,
			diff.Diff("expected", expected, "got", got))
	}
}

func TestRewrite(t *testing.T) {
	match, _ := filepath.Glob("testdata/*.input")
	for _, in := range match {
		name := filepath.Base(in)
		t.Run(name, func(t *testing.T) {
			out := in[:len(in)-len(".input")] + ".golden"
			runTest(t, in, out)
		})
	}
}
```

---

## 6. httptest Patterns

### Pattern Name: httptest.NewRecorder for Unit-Testing Handlers

**Source:** `/tmp/go-src/src/net/http/serve_test.go` lines 387-393

**What they do:** Use `httptest.NewRecorder()` to test HTTP handlers without starting a server. Captures status code, headers, and body.

**Why:** Fast, no network, no port allocation, no goroutines. Perfect for unit testing individual handlers in isolation.

**Anti-pattern:** Spinning up a full server to test handler logic that doesn't need networking.

**Code example (stdlib):**
```go
func TestServeMuxHandler(t *testing.T) {
	mux := NewServeMux()
	for _, e := range serveMuxRegister {
		mux.Handle(e.pattern, e.h)
	}
	for _, tt := range serveMuxTests {
		r := &Request{Method: tt.method, Host: tt.host, URL: &url.URL{Path: tt.path}}
		h, pattern := mux.Handler(r)
		rr := httptest.NewRecorder()
		h.ServeHTTP(rr, r)
		if pattern != tt.pattern || rr.Code != tt.code {
			t.Errorf("%s %s %s = %d, %q, want %d, %q",
				tt.method, tt.host, tt.path, rr.Code, pattern, tt.code, tt.pattern)
		}
	}
}
```

---

### Pattern Name: httptest.NewServer for Integration-Style Tests

**Source:** `/tmp/go-src/src/net/http/clientserver_test.go` lines 203-280

**What they do:** Use `httptest.NewServer` / `httptest.NewUnstartedServer` for end-to-end HTTP testing with a real TCP listener on localhost.

**Why:** Tests the full HTTP stack including transport, TLS, connection pooling, timeouts. The `clientServerTest` helper in the stdlib runs each test across HTTP/1.1, HTTP/2, and HTTP/3 modes.

**Code example (stdlib):**
```go
func newClientServerTest(t testing.TB, mode testMode, h Handler, opts ...any) *clientServerTest {
	cst := &clientServerTest{t: t, h2: mode == http2Mode, h: h}
	cst.ts = httptest.NewUnstartedServer(h)
	// ... configure based on mode ...
	switch mode {
	case http1Mode:
		cst.ts.Start()
	case http2Mode:
		cst.ts.EnableHTTP2 = true
		cst.ts.StartTLS()
	}
	cst.c = cst.ts.Client()
	t.Cleanup(cst.close)
	return cst
}
```

---

## 7. Benchmark Patterns

### Pattern Name: b.ReportAllocs + b.RunParallel + b.SetBytes

**Source:** `/tmp/go-src/src/encoding/json/bench_test.go` lines 85-101

**What they do:** Combine `b.ReportAllocs()` for allocation reporting, `b.RunParallel` for concurrent benchmarks, and `b.SetBytes` for throughput metrics.

**Why:**
- `b.ReportAllocs()` shows allocations/op — critical for hot paths.
- `b.RunParallel` measures performance under contention (real-world server behavior).
- `b.SetBytes` converts to MB/s throughput — meaningful for serialization benchmarks.

**Anti-pattern:** Benchmarks that only measure wall time without allocation tracking, or sequential benchmarks for concurrent code.

**Code example (stdlib):**
```go
func BenchmarkCodeEncoder(b *testing.B) {
	b.ReportAllocs()
	if codeJSON == nil {
		b.StopTimer()
		codeInit()
		b.StartTimer()
	}
	b.RunParallel(func(pb *testing.PB) {
		enc := NewEncoder(io.Discard)
		for pb.Next() {
			if err := enc.Encode(&codeStruct); err != nil {
				b.Fatalf("Encode error: %v", err)
			}
		}
	})
	b.SetBytes(int64(len(codeJSON)))
}
```

---

## 8. Integration Test Separation

### Pattern Name: testing.Short() for Expensive Tests

**Source:** `/tmp/go-src/src/net/http/serve_test.go` lines 800, 1000, 2212, 2581

**What they do:** Skip slow/flaky/network-dependent tests with `testing.Short()`. The Go CI runs with `-short` in fast mode, full tests in thorough mode.

**Why:** Fast feedback loop for development (`go test -short`), full validation in CI. No custom build tags needed.

**Anti-pattern:** Separate `_integration_test.go` files with build tags (Go stdlib doesn't do this), or always-slow tests that can't be skipped.

**Code example (stdlib):**
```go
func TestServerTimeouts(t *testing.T) {
	if testing.Short() {
		t.Skip("skipping in short mode")
	}
	// ... expensive test with real timeouts ...
}
```

---

## 9. No Assertion Libraries in Stdlib

### Pattern Name: Plain if/t.Errorf Over Assertion Frameworks

**Source:** Every test file in `/tmp/go-src/src/` (zero imports of `testify`, `gomega`, or any assertion library)

**What they do:** Use plain Go: `if got != want { t.Errorf(...) }`. Never import assertion libraries.

**Why:**
1. No implicit control flow — `t.Errorf` continues execution, so you see ALL failures at once.
2. No magic — the test reads like regular Go code.
3. Error messages are custom-crafted for each assertion, providing context that generic `assert.Equal` cannot.
4. One less dependency.

**Anti-pattern (Kubernetes uses this, stdlib does NOT):**
```go
// Kubernetes style (not stdlib):
assert.Equal(t, expected, actual)
require.NoError(t, err)
```

**Stdlib style:**
```go
if got := v.Elem().Interface(); !reflect.DeepEqual(got, tt.out) {
	t.Fatalf("%s: Decode:\n\tgot:  %#v\n\twant: %#v", tt.Where, got, tt.out)
}
```

---

## 10. Goroutine Leak Detection

### Pattern Name: TestMain + afterTest Goroutine Checking

**Source:** `/tmp/go-src/src/net/http/main_test.go` (entire file)

**What they do:** `TestMain` runs the test suite and checks for leaked goroutines after all tests complete. `afterTest` checks for goroutine leaks after each individual test.

**Why:** HTTP code spawns goroutines for connections, background reads, etc. Leaked goroutines indicate resource leaks (connections not closed, servers not shut down). Catching them prevents production OOMs.

**Code example (stdlib):**
```go
func TestMain(m *testing.M) {
	v := m.Run()
	if v == 0 && goroutineLeaked() {
		os.Exit(1)
	}
	os.Exit(v)
}

func goroutineLeaked() bool {
	for i := 0; i < 5; i++ {
		gs := interestingGoroutines()
		if len(gs) == 0 {
			return false
		}
		time.Sleep(100 * time.Millisecond)
	}
	// Report leaked goroutines
	return true
}

func afterTest(t testing.TB) {
	http.DefaultTransport.(*http.Transport).CloseIdleConnections()
	// Check for leaked goroutines from this specific test...
}
```

---

## 11. export_test.go Pattern

### Pattern Name: Bridge File for Internal Testing

**Source:** `/tmp/go-src/src/net/http/export_test.go` lines 1-50

**What they do:** Create an `export_test.go` file in the package itself (package `http`, not `http_test`) that exports internal symbols to external test packages. Only compiled during testing.

**Why:** Allows `http_test` (external test package) to access internals needed for white-box testing without polluting the public API. The `_test.go` suffix means it's never included in production builds.

**Code example (stdlib):**
```go
// export_test.go — package http (not http_test!)
package http

var (
	DefaultUserAgent             = defaultUserAgent
	ExportRefererForURL          = refererForURL
	ExportServerNewConn          = (*Server).newConn
	ExportErrRequestCanceled     = errRequestCanceled
)
```

---

## 12. Multi-Mode Test Runner

### Pattern Name: Generic Test Runner Across Protocol Modes

**Source:** `/tmp/go-src/src/net/http/clientserver_test.go` lines 100-134

**What they do:** A generic `run[T]` function that executes every client/server test in HTTP/1.1, HTTP/2, and HTTP/3 modes automatically. Tests opt into specific modes via options.

**Why:** Ensures behavioral consistency across protocol versions. A single test function covers all modes — no duplication. Bugs in one protocol version are caught immediately.

**Code example (stdlib):**
```go
// Test declaration (one line runs across 3 protocols):
func TestServerTimeouts(t *testing.T) { run(t, testServerTimeouts, []testMode{http1Mode}) }

// The runner:
func run[T TBRun[T]](t T, f func(t T, mode testMode), opts ...any) {
	t.Helper()
	modes := []testMode{http1Mode, http2Mode, http3Mode}
	for _, mode := range modes {
		t.Run(string(mode), func(t T) {
			t.Helper()
			t.Cleanup(func() { afterTest(t) })
			f(t, mode)
		})
	}
}
```

---

## 13. testLogWriter — Routing Server Logs to Test Output

### Pattern Name: io.Writer Adapter for *testing.T

**Source:** `/tmp/go-src/src/net/http/clientserver_test.go` lines 337-345

**What they do:** Implement `io.Writer` backed by `t.Logf`, so server error logs appear in test output (visible with `-v`, suppressed otherwise).

**Why:** Server logs are crucial for debugging test failures but shouldn't clutter passing output. `t.Log` gives you both: silent on pass, verbose on fail.

**Code example (stdlib):**
```go
type testLogWriter struct {
	t testing.TB
}

func (w testLogWriter) Write(b []byte) (int, error) {
	w.t.Logf("server log: %v", strings.TrimSpace(string(b)))
	return len(b), nil
}

// Usage:
cst.ts.Config.ErrorLog = log.New(testLogWriter{t}, "", 0)
```
