# Go Language Source: Architectural Conventions

How does the Go team build Go itself? What does the language source
reveal about conventions, governance, and infrastructure decisions?

**Repo:** [golang/go](https://github.com/golang/go)

---

## 1. Repo Shape

| Metric | Value |
|--------|-------|
| Size | 632M |
| Go files | 11,245 |
| Assembly files (runtime) | 200 |
| Commits | 66,142 |
| Contributors | 2,842 |
| Test files | 1,811 |
| Non-test files | 6,065 |
| Test ratio | 1:3.3 |
| TODOs (non-test) | 3,428 (all owner-attributed) |

### Organizational Philosophy

```
src/
├── cmd/           # The toolchain (compile, link, go, gofmt, vet)
│   └── compile/
│       └── internal/ssa/  # 417,686 lines of SSA compiler
├── internal/      # 61 hidden packages (API firewall)
├── runtime/       # 129,141 lines (scheduler, GC, memory)
├── encoding/      # Serialization (json, xml, gob)
├── net/           # Networking
├── io/            # Stream interfaces
└── testing/       # Test framework
```

The Go source has three layers:
1. **Flat stdlib** — user-visible packages (`fmt`, `io`, `net`)
2. **`internal/`** — shared infrastructure hidden from users (61 packages)
3. **`cmd/`** — the toolchain itself (compiler, linker, build tool)

---

## 2. What the Codebase Values

### By import frequency

| Package | Imports | Role |
|---------|---------|------|
| `fmt` | 2,031 | Formatting (the universal tool) |
| `testing` | 1,658 | Tests are first-class citizens |
| `strings` | 1,454 | String manipulation |
| `os` | 1,306 | System interaction |
| `unsafe` | 1,304 | Low-level memory access |
| `runtime` | 970 | Runtime introspection |
| `io` | 924 | Stream abstraction |

**The surprise:** `unsafe` (1,304 imports) ranks 5th — nearly tied with
`os`. The language that preaches memory safety uses `unsafe` extensively
in its own implementation. This is the "know the rules so you know
where they don't apply" principle.

### By size (what consumes the most lines)

| Component | Lines | Nature |
|-----------|-------|--------|
| SSA compiler | 417,686 | Mostly generated rewrite rules |
| Runtime | 129,141 | Scheduler + GC + memory |
| `testing/` | 4,770 | Testing framework |
| `internal/poll` | 5,087 | OS I/O multiplexing |
| `internal/fuzz` | 4,220 | Fuzzing infrastructure |
| `encoding/json/v2` | 6,387 | The JSON rewrite (experiment) |

---

## 3. The Bootstrap Problem

**How does Go compile itself?**

Go has been self-hosting since May 2015 (Russ Cox, commit `0f4132c907`:
"all: build and use go tool compile, go tool link"). The bootstrap
chain:

1. A pre-built Go 1.4 binary compiles the bootstrap tools
2. Those tools compile the current Go toolchain
3. The new toolchain recompiles itself for correctness

**The `cmd/dist` tool** orchestrates this multi-stage build. Unlike
Elixir (which keeps 33 Erlang files permanently), Go eliminated its
non-Go dependencies entirely. The only external requirement is a
previous Go binary.

**Convention:** Self-hosting means the compiler, linker, and runtime
are all written in Go. The runtime includes 200 assembly files for
architecture-specific operations (scheduler context switches, atomic
operations, system calls).

---

## 4. TODO Culture: Owned Accountability

```go
// TODO(gri)       — 320 occurrences (Robert Griesemer)
// TODO(mdempsky)  — 198 occurrences (Matthew Dempsky)
// TODO(adonovan)  — 170 occurrences (Alan Donovan)
// TODO(mknyszek)  — 98 occurrences (Michael Knyszek)
// TODO(rsc)       — 96 occurrences (Russ Cox)
```

**3,428 TODOs** in non-test code. Every TODO has an owner. The top 5
TODO authors are all core team members (compiler, runtime, and tools
leads).

**Convention:** `// TODO(username): description`

**Growth pattern:** linkname directives — 43 touches in 2019, 48 in
2022, 92 in 2024, 72 in 2025. The codebase is actively evolving its
relationship with internal/external boundaries.

**Contrast with Elixir:** Go TODOs are permanent documentation of known
limitations. Elixir TODOs are time-bombs with version deadlines. Go
accepts technical debt as a layer; Elixir refuses to let it accumulate.

---

## 5. Unique Patterns

### 5.1 `internal/` as API Firewall (61 packages)

Go's most distinctive structural pattern. The `internal/` directory is
enforced by the compiler — code outside the tree cannot import these
packages.

**Key internal packages:**
- `internal/godebug` — runtime feature flags (79 settings)
- `internal/goexperiment` — compile-time experiment guards
- `internal/singleflight` — dedup concurrent function calls
- `internal/bisect` — binary search for debugging (Russ Cox, 2023)
- `internal/poll` — OS I/O polling (5,087 lines)
- `internal/fuzz` — fuzzing coordinator (4,220 lines)
- `internal/coverage` — code coverage (3,821 lines)

**Convention:** Code shared between stdlib packages but not suitable for
public API goes in `internal/`. This is Go's answer to "how do you
share utilities without committing to backward compatibility."

### 5.2 GODEBUG: Runtime Feature Flags

Introduced 2021 (Brad Fitzpatrick), formalized as a compatibility
mechanism by Russ Cox in 2022 (proposal #56986: "extended backwards
compatibility for Go", 70 comments).

```go
var http2server = godebug.New("http2server")

func ServeConn(c net.Conn) {
    if http2server.Value() == "0" {
        // disable HTTP/2
    }
}
```

**79 godebug settings** across the codebase. Each time a non-default
setting causes a behavior change, code calls `IncNonDefault()` to
increment a counter readable via `runtime/metrics`.

**Convention:** When a behavior change might break existing programs,
add a GODEBUG setting. The old behavior remains accessible via
`GODEBUG=setting=old_value`. New Go versions automatically use the
old behavior when building code that declared an older `go` directive
in `go.mod`.

**This is the Go compatibility promise made machine-enforceable.**

### 5.3 GOEXPERIMENT: Compile-Time Feature Gates

```go
// In internal/goexperiment/flags.go:
// GOEXPERIMENT=jsonv2 enables the new JSON API
```

**Convention:** Major API additions ship behind experiment flags.
`encoding/json/v2` (6,387 lines, Apr 2025) exists in the tree but is
only visible when `GOEXPERIMENT=jsonv2` is set. This allows the code
to be developed in-tree, tested by adventurous users, and refined
before the compatibility promise applies.

### 5.4 Compiler Directives as Hidden Language

```go
//go:linkname localFunction remote/package.Function
//go:nosplit
//go:nowritebarrier
//go:noescape
//go:systemstack
```

**1,711 `go:linkname` directives** and **2,428 runtime compiler
directives** in non-test code. These are effectively a hidden language
within Go — they bypass the type system, calling conventions, and
garbage collector safety for performance-critical paths.

**Convention:** Directives are ONLY acceptable in the runtime and
compiler. The Go team is actively trying to reduce `go:linkname` usage
(92 touches in 2024 — many are removals). Third-party packages that use
`go:linkname` to access internals are explicitly unsupported.

### 5.5 Generated Code: The SSA Compiler

The SSA (Static Single Assignment) compiler backend is 417,686 lines,
but the largest files are generated:

```
opGen.go           — 97,135 lines (generated)
rewriteAMD64.go    — 79,703 lines (generated)
rewritegeneric.go  — 38,337 lines (generated)
rewriteARM64.go    — 26,203 lines (generated)
```

**Convention:** Generated files are checked into the repo (not
generated at build time). They contain a `// Code generated` header.
The generators live alongside their output. This means `git blame`
works on generated code — you can trace when a rewrite rule was added.

### 5.6 The Runtime: 129K Lines of Go + Assembly

```
proc.go   — 8,156 lines (THE goroutine scheduler)
malloc.go — 2,501 lines (memory allocator)
mgc.go    — 2,315 lines (garbage collector)
mheap.go  — 3,030 lines (heap management)
panic.go  — 1,788 lines (panic/recover machinery)
```

The scheduler (`proc.go`) documents its own design at the top:
> "The main concepts are: G - goroutine. M - worker thread, or machine.
> P - processor, a resource that is required to execute Go code."

**Convention:** The runtime combines Go and assembly (200 `.s` files).
Architecture-specific operations (context switches, atomic ops, system
calls) are in assembly. Everything else is in Go, using compiler
directives to bypass safety checks where needed.

---

## 6. PR Discussion Patterns

Go uses GitHub issues for proposals, not PRs for discussion. The key
governance mechanism is the **proposal process** with designated
reviewers.

### json/v2 (Issue #71497, 201 comments, Jan 2025)

"The largest major revision of a standard Go package to date."

**Key design decision:** Split into `encoding/json/v2` (semantic) and
`encoding/jsontext` (syntactic). The syntactic layer has no reflection
dependency — it's a pure JSON tokenizer.

**Ship strategy:** Land behind `GOEXPERIMENT=jsonv2`, iterate with
community feedback, then graduate. The code lives in-tree but doesn't
count as a compatibility commitment until the experiment flag is
removed.

**External validation:** Built as `github.com/go-json-experiment/json`
first, iterated for years, then proposed for stdlib. The implementation
preceded the proposal.

### GODEBUG (Issue #56986, 70 comments, Nov 2022)

Russ Cox's proposal for machine-enforced backward compatibility.

**The problem:** Sort algorithm changes, bug fixes, and behavior
improvements can break programs that depend on old behavior.

**The solution:** `go.mod`'s `go` directive becomes a compatibility
declaration. Programs built with Go 1.22 but declaring `go 1.20`
automatically get Go 1.20 behavior for any setting that changed between
1.20 and 1.22.

**Lesson:** The Go team solved "how do you improve a language without
breaking users" with a general mechanism rather than case-by-case
migration. Each new GODEBUG setting is a structured backward
compatibility opt-out.

### Generics (Issue #15292, 874 comments, 2016-2021)

5 years of discussion. The most debated language change in Go's history.

**Lesson:** Committee-driven projects can take years on foundational
decisions because consensus requires addressing every edge case. Compare
to Elixir's formatter (1 hour, zero comments) — the BDFL model moves
faster but accepts more single-point-of-failure risk.

### slog (Issue #56345, 841 comments, 2022-2023)

Structured logging took 10 months and 841 comments to land.

**Lesson:** Even when the need is clear and the solution is well-known,
Go's process requires exhaustive discussion. The result is usually
better (slog is well-designed), but the cost is measured in months.

---

## 7. Cross-Ecosystem Comparisons

| Aspect | Go | Elixir |
|--------|-----|--------|
| TODOs | 3,428, owner-attributed, permanent | 127, version-gated, deadlines |
| Self-hosting | Complete since 2015 | 33 Erlang files permanently |
| Feature gates | GOEXPERIMENT (compile-time) | None (ship or don't) |
| Compat mechanism | GODEBUG (79 settings) | Deprecation → removal on version |
| Governance | Committee (proposals, 874-comment threads) | BDFL (José, 1-hour merges) |
| Internal boundary | `internal/` (compiler-enforced, 61 packages) | OTP applications (convention-enforced) |
| Generated code | Checked in (97K-line files) | Compile-time (no artifacts) |
| Assembly | 200 .s files in runtime | None (delegates to Erlang/BEAM) |
| Biggest file | 97,135 lines (generated) | 7,102 lines (Kernel) |

---

## 8. What This Teaches

1. **`internal/` solves "shared but not public" at the language level.**
   61 packages that other ecosystems have to solve with conventions,
   Go solves with compiler enforcement. This is why Go projects
   rarely have "utils" packages that leak abstraction — the pattern
   exists in the language itself.

2. **GODEBUG is the most sophisticated backward compatibility mechanism
   in any language runtime.** It makes the compatibility promise
   *machine-verifiable* rather than *socially-enforced*. Programs don't
   just get old behavior by default — they get it because their `go.mod`
   declares what era they belong to.

3. **GOEXPERIMENT enables fearless iteration in the stdlib.** json/v2
   can exist in-tree, be tested, be refined — all without triggering
   the compatibility promise. This is "feature flags for a language."

4. **3,428 TODOs is honest, not sloppy.** Each one has an owner. They
   document known limitations rather than hiding them. Go prefers
   "explicitly imperfect" over "implicitly broken."

5. **Compiler directives are a hidden language.** 4,139 directives
   (linkname + runtime pragmas) bypass Go's safety model. The Go team
   accepts that the runtime needs a different language than users —
   but actively restricts this power from escaping to third-party code.

6. **Generated code checked in > generated at build time** for
   archaeology. `git blame` on `rewriteAMD64.go` tells you when a
   codegen rule was added and why. Build-time generation loses this
   history.

7. **Committee governance produces better designs but 10x slower.** Go's
   slog (841 comments, 10 months) vs Elixir's formatter (0 comments,
   1 hour). The designs are comparable in quality — the process cost
   is where they differ.

8. **`unsafe` in Go's own source (1,304 imports) proves that safety
   rules are for users, not for runtime implementors.** The people who
   wrote the safety rules know exactly where they don't apply.

<!-- PATTERN_COMPLETE -->
