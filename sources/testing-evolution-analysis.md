# Testing Philosophy & API Evolution

How codebases prove correctness and manage change over
time reveals their deepest architectural commitments.

---

## Testing Philosophy: Four Models of Proof

### CockroachDB: Defense in Depth

**Levels of proof:**
1. **Unit tests** — co-located in same package
2. **Echotest/golden files** — snapshot expected output (209
   testdata directories, auto-rewrite with -rewrite flag)
3. **Data-driven tests** — declarative test specs in txt files
4. **KVNemesis** — chaos/fuzzing that generates random KV
   operations and checks linearizability
5. **Leak detection** — goroutines, stoppers tracked globally

**The echotest pattern:**
```go
echotest.Require(t, output, filepath.Join("testdata", name+".txt"))
```

Golden file says:
```
echo
----
result is ambiguous: boom with a secret
result is ambiguous: boom with a ‹secret›
```

The test produces output, compares against the golden file.
Run with `-rewrite` to update. This means:
- Tests are **self-documenting** (the golden file IS the spec)
- Regressions are **visible in diffs** (the golden file changes)
- No manual expected-value maintenance

**KVNemesis (chaos testing at ecosystem level):**
Generates random sequences of KV operations (puts, gets,
splits, merges, transfers) against a real cluster, then
validates that results satisfy serializable isolation.

This isn't unit testing. This is proving the *system* is
correct, not individual functions.

**Resource leak detection as CI gate:**
```go
// Every test file
defer leaktest.AfterTest(t)()

// Every TestMain
func init() {
    leaktest.PrintLeakedStoppers = PrintLeakedStoppers
}
```

If a test leaks a goroutine or Stopper, it **fails**. Not
a warning. A failure. This means resource correctness is
as enforceable as logic correctness.

### Prometheus: Golden Files + Goroutine Verification

**Testing DSL for PromQL:**
```
load 5m
  http_requests{job="api-server"} 0+10x10

eval instant at 50m SUM BY (group) (http_requests)
  {group="canary"} 700
  {group="production"} 300
```

This is a custom test language. Load data, evaluate
expressions, assert results. **205 test config files**
in `config/testdata/` alone.

**Force:** PromQL is complex enough that example-based
testing would be insufficient. The DSL lets you write
hundreds of test cases concisely, covering edge cases
that would require dozens of Go test functions.

**Goroutine leak detection:**
```go
func TolerantVerifyLeak(m *testing.M) {
    goleak.VerifyTestMain(m,
        goleak.IgnoreTopFunction("go.opencensus.io/..."),
        goleak.IgnoreTopFunction("k8s.io/klog/..."),
    )
}
```

Explicit allowlist for known third-party leaks. Everything
else is a test failure. Zero-tolerance with escape hatches
for unfixable external dependencies.

### Ecto: Fake Adapter + Process Mailbox Assertions

```elixir
defmodule Ecto.TestAdapter do
  @behaviour Ecto.Adapter
  @behaviour Ecto.Adapter.Queryable
  @behaviour Ecto.Adapter.Schema
  @behaviour Ecto.Adapter.Transaction

  def execute(_, _, {:nocache, {:all, query}}, _, _) do
    send(self(), {:all, query})
    Process.get(:test_repo_all_results) || results_for_all_query(query)
  end
end
```

**Ecto tests the entire query pipeline without a database.**
The fake adapter:
- Sends messages to `self()` on every operation
- Tests assert on `receive {:insert, meta}` etc.
- No network, no state, pure message-passing verification

**48 test files, 43 with `async: true`.** The test suite
runs in parallel because there's no shared state — every
test talks to its own process mailbox.

**Force:** Ecto is a *library*, not a service. It can't
require Postgres in CI for every contributor. The fake
adapter makes the entire query compilation + planning
pipeline testable without external dependencies.

### Oban: Testing Modes as First-Class Feature

```elixir
# In test config
config :my_app, Oban, testing: :inline

# In test
use Oban.Testing, repo: MyApp.Repo

test "job was enqueued" do
  assert_enqueued worker: MyWorker, args: %{id: 1}
end

test "job executes correctly" do
  assert :ok = perform_job(MyWorker, %{id: 1})
end
```

Three modes:
- **`:inline`** — jobs execute synchronously in the test
  process. No GenServers, no queues, no async.
- **`:manual`** — jobs are enqueued but not executed.
  Use `assert_enqueued` to verify they were created.
- **`:disabled`** — production behavior in tests.

**Force:** Background jobs are the #1 source of test
flakiness. Oban eliminates it by making the execution
model configurable. Tests never poll, never sleep, never
race.

---

## API Evolution: Three Strategies

### CockroachDB: Version Gates (Distributed Migration)

```go
const (
    V26_2_AddStatementStatisticsComputedColumns Key = iota
    V26_2_ChangefeedsStopReadingSpanLevelCheckpoints
    V26_2_ChangefeedsStopWritingSpanLevelCheckpoints
)

// In code:
if settings.Version.IsActive(ctx, clusterversion.V26_2) {
    // use new behavior
}
```

**The pattern:** Every change to observable behavior gets
a version constant. The feature is only enabled when ALL
nodes in the cluster have been upgraded past that version.

**Two-phase deprecation for distributed changes:**
```
V26_2_ChangefeedsStopReadingSpanLevelCheckpoints
V26_2_ChangefeedsStopWritingSpanLevelCheckpoints
V26_2_ChangefeedsNoLongerHaveSpanLevelCheckpoints
```

Three versions for one removal:
1. Stop reading (new code doesn't depend on old format)
2. Stop writing (old format no longer produced)
3. Clean up (safe to remove the old code)

**Force:** In a distributed database, you can't change
behavior atomically. Some nodes will be old, some new.
The version gate ensures new behavior only activates
when it's safe — when all nodes understand it.

**Pruning:** Once MinSupported advances past a version
constant, it's deleted. The code path is always active
so the `IsActive` check becomes dead code. Regular
pruning keeps the codebase from accumulating gates.

### Oban: Numbered Migrations (Schema Evolution)

```elixir
lib/oban/migrations/postgres/
├── v01.ex  # Initial schema (job table, state enum)
├── v02.ex  # Add columns
├── v03.ex  # Index optimization
...
├── v14.ex  # Latest
```

Each migration is:
- **Idempotent** (safe to run twice)
- **Prefix-aware** (multi-tenant schemas)
- **Bidirectional** (up + down)
- **Database-specific** (postgres/, sqlite/, myxql/)

**Consumer usage:**
```elixir
defmodule MyApp.Repo.Migrations.AddOban do
  use Ecto.Migration
  def up, do: Oban.Migrations.up(version: 14)
  def down, do: Oban.Migrations.down(version: 14)
end
```

**Force:** Oban owns a database table but lives inside
the consumer's migration system. Numbered versions let
consumers upgrade incrementally without knowing Oban
internals.

### Ecto: Compile-Time Deprecation + Semver

```elixir
# In changeset.ex
IO.warn(
  "passing a list of binaries to cast/3 is deprecated..."
)
```

Ecto deprecates at **compile time**. When you compile
code that uses a deprecated API, you get a warning.
At runtime, everything still works.

**CHANGELOG as contract:**
```
## v3.14.0-dev
### Enhancements
### Bug fixes

## v3.13.5 (2025-11-09)
### Enhancements
```

The changelog is the API evolution document. Breaking
changes require a major version bump (hasn't happened
in years because the adapter pattern provides
extensibility without breakage).

---

## What This Teaches for Code Review

### Testing Questions:
1. Is this testable **without standing up the system**?
   (Ecto's fake adapter, Oban's inline engine)
2. Are resources **tracked and leak-detected**?
   (CockroachDB's stopper/goroutine tracking)
3. Are test assertions **deterministic**? No sleep, no
   poll, no "eventually consistent" in unit tests.
4. Could this be a **golden file test**? If the output
   is deterministic, snapshot it. Regression = visible diff.
5. Is there **chaos/property testing** for invariants?
   (KVNemesis for linearizability)

### Evolution Questions:
1. Can this change be deployed **gradually**? Or does it
   require all consumers to upgrade atomically?
2. Is there a **two-phase** path? (Stop reading → stop
   writing → remove)
3. Is the deprecation **visible at compile time**? Or
   will consumers only discover it at runtime?
4. Is the migration **idempotent**? Can it be run twice
   safely?

### Red Flags:
- Tests that require a running database for unit-level logic
- No resource leak detection in concurrent code
- `time.Sleep` / `Process.sleep` in tests instead of
  deterministic signals
- Breaking changes without version gates or migration path
- Deprecation that only appears in docs, not in tooling

<!-- PATTERN_COMPLETE -->
