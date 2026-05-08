# Go Patterns

**Prescriptive.** Follow these when writing Go code.

A pattern is a reusable solution to a recurring problem. Each one has:
- **When to use** — the problem it solves
- **When NOT to use** — where it causes harm
- **Why** — the reasoning, not just the rule
- **Source citations** — verified file:line from real codebases

These are derived from what mature Go codebases *actually do*, not opinions or blog posts.

## Structure

- `patterns/` — what to do (interfaces, errors, concurrency, testing, packages, etc.)
- `smells/` — what NOT to do (anti-patterns, common mistakes)
- `sources/` — reference material from specific projects (golang/go, Prometheus). Study for ideas, don't copy blindly.

## How to use

### Writing code

> Read the relevant pattern files from this repo before writing code. Your implementation must follow the documented patterns — if you deviate, justify why in a comment. Check `smells/` to make sure you're not introducing a known anti-pattern.

### Reviewing code

> Read the relevant pattern files from this repo. For each function or module in the diff, verify it follows the documented pattern. If the code deviates, flag it with a reference to the specific pattern and section. A deviation without justification is a finding.

### Evaluating a pattern

> Read the pattern file. Compare against how the following projects handle the same problem: [list projects]. Does the pattern hold? Are there cases where it breaks down? Update the pattern with what you find.

## Patterns vs Conventions

**Pattern** = prescriptive. "When you face X, do Y." Language-scoped. Follow these.

**Convention** = descriptive. "Project Z does it this way." Context-specific. Study for ideas — applying another project's conventions to yours without understanding their constraints causes harm.

The `sources/` directory is convention material absorbed from thin repos. The `patterns/` directory is what you actually follow.
