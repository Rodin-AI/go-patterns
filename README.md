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

Give your agent these instructions depending on the task:

### Solving a problem

> You have access to a patterns repo containing proven solutions to recurring Go problems. When I describe a problem:
>
> 1. Identify which pattern files are relevant (read them)
> 2. Check if my problem matches a "When to use" case
> 3. Check if it matches a "When NOT to use" case
> 4. If a pattern fits: suggest the approach, cite the pattern, explain why it applies here
> 5. If no pattern fits: say so, and suggest an approach grounded in the principles you see across the patterns
> 6. If my problem matches a smell: warn me before I make the mistake
>
> Never suggest something that contradicts a documented pattern without explicitly calling out the deviation and justifying it.

### Reviewing code

> You have access to a patterns repo that defines how Go code should be written. For each file in the diff:
>
> 1. Read the relevant pattern files
> 2. Verify the code follows the documented patterns
> 3. If it deviates: flag it with a reference to the specific pattern, section, and why it matters
> 4. If it matches a smell: flag it as a known anti-pattern
> 5. A deviation without justification is a finding
>
> Don't invent rules. Only flag what the patterns document.

### Evaluating a pattern

> Read the pattern file. Compare against how the following projects handle the same problem: [list projects]. Does the pattern hold? Are there cases where it breaks down? Should it be updated, split, or retired? File your findings as an issue.

## Patterns vs Conventions

**Pattern** = prescriptive. "When you face X, do Y." Language-scoped. Follow these.

**Convention** = descriptive. "Project Z does it this way." Context-specific. Study for ideas — applying another project's conventions to yours without understanding their constraints causes harm.

The `sources/` directory is convention material absorbed from thin repos. The `patterns/` directory is what you actually follow.
