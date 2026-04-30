# Go Patterns

Idiomatic Go patterns extracted from the [Go standard library](https://github.com/golang/go) and [Kubernetes](https://github.com/kubernetes/kubernetes) source code with verified file:line citations.

## Structure

- `patterns/` — Go stdlib patterns (interfaces, errors, concurrency, structs, testing, docs, style, API conventions, packages)
- `kubernetes/` — Production-scale patterns from Kubernetes (controllers, informers, workqueues)
- `comparison/` — stdlib vs Kubernetes patterns
- `smells/` — Anti-patterns and common Go mistakes
- `changelog/` — Daily digest of merged PRs

## Philosophy

These rules are derived from what the Go source code actually does, not opinions or blog posts. Every pattern cites specific files and line numbers.

When unsure how to do something in Go, look at how the standard library does it.
