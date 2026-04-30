# Patterns Extracted from prometheus/prometheus

## Pattern: Atomic File Operations with Suffix Convention

**Source:** `tsdb/db.go`
**Category:** storage

**What:** Use directory suffixes (`.tmp-for-creation`,
`.tmp-for-deletion`) to make multi-step file operations
crash-safe. On startup, clean up any dirs with these
suffixes (they represent incomplete operations).

**Why:** Database storage needs atomicity. If the process
crashes between creating a block and finalizing it, you
need to know the block is incomplete. The suffix convention
makes incomplete state visible at the filesystem level
without requiring a separate journal.

**Example:**

```go
const (
    tmpForDeletionBlockDirSuffix = ".tmp-for-deletion"
    tmpForCreationBlockDirSuffix = ".tmp-for-creation"
)

// On startup: remove any .tmp-* dirs (incomplete ops)
// On create: write to dir.tmp-for-creation, then rename
// On delete: rename to dir.tmp-for-deletion, then remove
```

**When to use:** Any system that manages files/directories
and needs crash consistency without a full WAL. Simpler
than a write-ahead log for coarse-grained operations.

**When NOT to use:** When you already have a WAL or
transaction log. Or for fine-grained operations where
rename semantics are insufficient.

---

## Pattern: DefaultOptions() Function

**Source:** `tsdb/db.go`
**Category:** configuration

**What:** Provide a `DefaultOptions()` function returning a
fully-populated config struct. Users copy and override only
what they need. No nil-means-default ambiguity.

**Why:** Large config structs (20+ fields) are unwieldy.
By providing sane defaults as a function (not a
package-level var), you avoid mutation bugs and make it
clear what "normal" looks like. Users only specify
deviations.

**Example:**

```go
func DefaultOptions() *Options {
    return &Options{
        WALSegmentSize:    wlog.DefaultSegmentSize,
        RetentionDuration: int64(15*24*time.Hour / ...),
        MinBlockDuration:  DefaultBlockDuration,
        MaxBlockDuration:  DefaultBlockDuration,
        SamplesPerChunk:   DefaultSamplesPerChunk,
        // ... 20 more fields with sane defaults
    }
}

// Usage:
opts := tsdb.DefaultOptions()
opts.RetentionDuration = 30 * 24 * time.Hour
db, err := tsdb.Open(dir, nil, nil, opts, nil)
```

**When to use:** Config structs with many fields where most
users want defaults. Especially when zero-value semantics
would be confusing (e.g., 0 retention = infinite? or off?).

**When NOT to use:** Small configs (3-4 fields) where
struct literal with zero-means-default is clear enough.

---

## Pattern: Scrape Loop with Aligned Timestamps

**Source:** `scrape/scrape.go`
**Category:** concurrency

**What:** Periodic scrape loops that align timestamps to
intervals with a small tolerance, enabling better storage
compression downstream.

**Why:** Time-series databases compress better when
timestamps are regular. A 2ms tolerance on alignment
means scraped data aligns to the expected grid while
accommodating real-world jitter.

**Example:**

```go
var ScrapeTimestampTolerance = 2 * time.Millisecond
var AlignScrapeTimestamps = true

// In scrape loop: if scrape finishes within tolerance
// of expected timestamp, snap to the grid
```

**When to use:** Any periodic data collection where
downstream storage benefits from timestamp regularity.
Metrics, heartbeats, polling loops.

**When NOT to use:** Event-driven data where timestamps
must reflect actual occurrence time. Audit logs, user
actions, financial transactions.

---

## Pattern: Sentinel Errors with Interface Check

**Source:** `tsdb/db.go`
**Category:** error-handling

**What:** Define package-level sentinel errors with
`errors.New()` and use compile-time interface assertions
to verify implementations satisfy storage interfaces.

**Why:** `ErrNotReady` as a sentinel lets callers use
`errors.Is` for retry logic. The pattern ensures error
identity is stable across versions (not string-matched).

**Example:**

```go
var ErrNotReady = errors.New("TSDB not ready")

// Callers can reliably detect this:
if errors.Is(err, tsdb.ErrNotReady) {
    // Retry later — DB is still initializing
}
```

**When to use:** Any error that callers need to handle
programmatically (retry, fallback, special UI). Make it a
named sentinel, not a string comparison.

**When NOT to use:** Errors that are always terminal or
always logged-and-discarded. Not every error needs a name.

---

## Pattern: Compile-Time Interface Satisfaction

**Source:** `scrape/scrape.go`
**Category:** organization

**What:** Use `var _ Interface = (*Type)(nil)` to verify at
compile time that a type satisfies an interface, even if
the type is only used dynamically.

**Why:** Without this, you discover missing methods only
when the type is actually used — which might be in a
rarely-exercised code path or only in production. The
compile-time check catches it immediately.

**Example:**

```go
var _ FailureLogger = (*logging.JSONFileLogger)(nil)
// Fails at compile time if JSONFileLogger doesn't
// implement FailureLogger
```

**When to use:** Any type that implements an interface
consumed dynamically (registered in a map, stored as
interface value, passed to framework code).

**When NOT to use:** Types whose interface satisfaction is
already enforced by direct usage in the same package.

<!-- PATTERN_COMPLETE -->
