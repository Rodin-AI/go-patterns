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

1. **Before writing code:** check if a relevant pattern exists
2. **During review:** verify code follows documented patterns
3. **If code deviates:** either fix it or document why the deviation is justified

## Patterns vs Conventions

**Pattern** = prescriptive. "When you face X, do Y." Language-scoped. Follow these.

**Convention** = descriptive. "Project Z does it this way." Context-specific. Study for ideas — applying another project's conventions to yours without understanding their constraints causes harm.

The `sources/` directory is convention material absorbed from thin repos. The `patterns/` directory is what you actually follow.
