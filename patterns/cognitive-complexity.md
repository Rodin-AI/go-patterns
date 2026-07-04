# Cognitive Complexity: Keeping Functions Under the Threshold

Cognitive complexity measures how hard a function is to understand by
penalizing nested control flow, compound conditions, and breaks in
linear reading. SonarQube rule `go:S3776` flags functions exceeding 30.

The `gocognit` linter (available in golangci-lint) enforces this locally
so violations are caught at `make check` time rather than in a daily scan.

---

## How Complexity Accumulates

Each of these adds +1:
- `if`, `else if`, `else`
- `for`, `switch`, `select`
- `goto`, `break` to label, `continue` to label
- Sequences of logical operators in a condition (`&&`, `||`)

**Nesting** adds a further +1 per enclosing level. A simple `if` inside
a `for` costs +2 (1 for the `if`, 1 for nesting inside the `for`).

Compound boolean conditions (`a && b || c`) cost per *change* of operator
(not per operand), so `a && b && c` is +1 but `a && b || c` is +2.

---

## Extraction Strategy

When a function exceeds the threshold, extract along these seams
(in order of preference):

### 1. Input Validation Block

Sequential guard checks at the top of a function are often 5–8 `if`
statements that each cost +1. Extract into `validateXxx(input) error`.

```go
// Before: 5 sequential ifs in the parent function body.
func ReadData(ctx context.Context, db DB, q Query) (Result, error) {
    if q.Name == "" { return Result{}, errInvalid }
    if q.Limit <= 0 { return Result{}, errInvalid }
    // ... 3 more checks ...
    // ... core logic ...
}

// After: validation is one call + one if in the parent.
func ReadData(ctx context.Context, db DB, q Query) (Result, error) {
    if err := validateQuery(q); err != nil {
        return Result{}, err
    }
    // ... core logic ...
}

func validateQuery(q Query) error {
    if q.Name == "" { return newError(CodeInvalid, "read data", errInvalid) }
    if q.Limit <= 0 { return newError(CodeInvalid, "read data", errInvalid) }
    // ...
    return nil
}
```

**Why it works:** Each `if` in the parent was +1 (no nesting penalty
since they're sequential). Moving N checks out saves exactly N points.

### 2. Post-Loop Finalization

A common pattern: a `for rows.Next()` loop accumulates state, then a
second loop finalizes (promotes buckets to points, sorts, serializes).
The finalization loop often has its own nested `if` and sub-loops.

```go
// Before: finalization adds ~6–8 complexity at the end of a long function.
func processRows(...) {
    for rows.Next() { /* scan + classify */ }
    for _, item := range items {
        for _, bucket := range item.buckets { /* promote */ }
        sort.SliceStable(item.points, func(i, j int) bool { /* compare */ })
    }
}

// After: finalization is one call.
func processRows(...) {
    for rows.Next() { /* scan + classify */ }
    finalizeItems(items, query)
}

func finalizeItems(items map[string]*itemState, query Query) {
    for _, item := range items { /* promote + sort */ }
}
```

### 3. Loop Body Classification

When a loop body has multiple mutually exclusive branches that each
do non-trivial work (map initialization, struct construction), extract
the classification into a helper that returns a result type or mutates
accumulators passed by pointer/reference.

```go
// Before: deeply nested ifs inside a for loop.
for i := range resources {
    mapping := resources[i].Mapping
    if mapping == nil { continue }
    if mapping.Status == Matched && mapping.ID != nil {
        // 10 lines of map init + struct construction
    }
    if mapping.Status != Matched && mapping.Code != nil {
        // 5 lines of counting
    }
}

// After: one call per iteration.
for i := range resources {
    classifyResourceMapping(resources[i], accumulators)
}
```

### 4. Nested Distribution/Attribution Logic

Triple-nested loops (for config → for accumulator → for reason) that
distribute counts or propagate state. These are self-contained: they
read from collected maps and mutate accumulators.

```go
// Extract the inner body, keep the outer iteration visible in the parent.
for configKey, byReason := range unresolvedByConfig {
    distributeToAccumulators(configKey, byReason, registrations, accumulators)
}
```

---

## Rules of Thumb

| Guideline | Rationale |
|-----------|-----------|
| Keep the parent function's shape readable | The caller should still show the high-level flow: validate → query → accumulate → finalize |
| Name extracted functions by *what* they do, not *where* they came from | `validateQuery` not `topOfReadData` |
| Prefer returning `error` over multiple returns | One error check in the parent costs +1; that's cheaper than 5 sequential ifs |
| Don't extract single-`if` blocks just to game the metric | Extraction should improve readability, not just satisfy a counter |
| Extracted functions should be testable in isolation | If the extraction requires passing 8 arguments, reconsider the seam |

---

## Linter Configuration

Enable `gocognit` in `.golangci.yml`:

```yaml
linters:
  enable:
    - gocognit

settings:
  gocognit:
    min-complexity: 30
```

This matches SonarQube's default threshold and catches violations at
local lint time rather than in a delayed daily scan.

---

## When NOT to Extract

Not every function over the threshold should be split. Extracting can
hurt readability when the function is a single cohesive algorithm whose
steps share tightly-coupled local state. The question is: "Would a
reader need to bounce between multiple functions to understand what this
code does?"

### Example: `(*pp).doPrintf` in the Go standard library (score: 77)

The core of `fmt.Printf` is a single format-string parser that scans
character-by-character, maintaining a cursor `i` and accumulated state
(flags, width, precision, argument index). It scores 77 because it has
nested switches, conditional flag parsing, and multiple labeled loops:

```go
// src/fmt/print.go — simplified skeleton
func (p *pp) doPrintf(format string, a []any) {
    end := len(format)
    argNum := 0
    afterIndex := false
formatLoop:
    for i := 0; i < end; {
        // skip literal text up to '%'
        // parse flags: '#', '0', '+', '-', ' '
        // fast-path: simple lowercase verb without width/precision
        // parse explicit argument index [N]
        // parse width ('*' or digits)
        // parse precision ('.' then '*' or digits)
        // decode verb rune and dispatch
        switch {
        case verb == '%': p.buf.writeByte('%')
        case !p.goodArgNum: p.badArgNum(verb)
        case argNum >= len(a): p.missingArg(verb)
        case verb == 'v': /* handle Go-syntax, fallthrough */
        default: p.printArg(a[argNum], verb); argNum++
        }
    }
    // report extra arguments
}
```

**Why it should NOT be split:**

| Tempting extraction | Why it fails |
|---|---|
| `parseFlags(format, i) → i` | Shares `simpleFormat` label + fast-path `continue formatLoop`; extracting requires returning multiple values AND a "should continue outer loop" signal |
| `parseWidth(format, i, a, argNum) → (i, argNum)` | Mutates `p.fmt.wid`, `p.fmt.widPresent`, `p.fmt.minus`, `p.fmt.zero` — passing all as params/returns creates a worse signature than the inline code |
| `parsePrecision(...)` | Same problem — tightly coupled to width/afterIndex state above it |
| `dispatchVerb(...)` | References `argNum`, `p.wrappedErrs`, flag state set earlier in the same iteration — extraction gains nothing because you pass everything through |

The cursor position `i` threads linearly through every stage. A reader
can follow it top-to-bottom in one function. Splitting would force them
to mentally stitch together 4–5 helpers that each advance `i` and mutate
shared state, making the code *harder* to understand despite a lower score.

### Indicators that a high-complexity function should stay intact

- **Single linear cursor or state variable** threads through all branches
  (parser, scanner, protocol state machine).
- **Tightly-coupled mutation** — extracting would require passing 5+
  parameters or returning multiple values including "flow control"
  signals (continue/break labels).
- **No natural seam** — the function doesn't have a clear "before/after"
  boundary; every step depends on state from the previous step.
- **Well-known algorithm** — readers expect to find the logic in one
  place (sort partitions, format parsing, accept loops).

In these cases, suppress with a comment:

```go
//nolint:gocognit // single-pass format parser; splitting scatters tightly-coupled cursor state
func (p *pp) doPrintf(format string, a []any) {
```

### Contrast: when extraction IS correct despite seeming "cohesive"

The functions fixed in #397–#399 *looked* like single algorithms but
had clear seams:

- **Validation guards** at the top — sequential, no shared state with
  the body below.
- **Post-loop finalization** — reads accumulated state but doesn't feed
  back into the loop.
- **Distribution/attribution** logic — triple-nested but self-contained.

The test: can you draw a horizontal line in the function where everything
above doesn't depend on anything below (or vice versa)? If yes, extract.
If no (as with `doPrintf`), leave it alone.
