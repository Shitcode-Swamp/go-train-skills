---
name: minimal-comments
description: Code styling rule for comments. Use whenever writing or editing code (including tests) to decide whether a comment belongs. Comments are only allowed to explain tricky or counter-intuitive code; everything else must be expressed through clear names and structure.
---

# Minimal Comments

Do not write comments by default. Code should explain itself through names, small functions, and clear structure.

## When a comment IS allowed

Write a comment only when a reader would likely misread the code or "fix" it into a bug without it:

- Counter-intuitive behavior: code that looks wrong but is correct.
- Non-obvious reasons: a workaround for a library bug, a protocol quirk, a performance trick.
- Hidden constraints: ordering that matters, invariants the compiler can't enforce, concurrency assumptions.
- Surprising test setup: a test case whose purpose isn't clear from its name and inputs.

The comment explains **why**, never **what**.

## When a comment is NOT allowed

- Restating what the code does (`// increment counter`, `// create user`).
- Section dividers or labels (`// Arrange`, `// Act`, `// Assert`, `// --- helpers ---`).
- Describing function parameters or return values that the signature already shows.
- Commented-out code.
- TODOs without a concrete reason a reader needs right now.
- Explaining a test case whose `name` field or test function name already says it.

## Before writing a comment

1. Can a better variable, function, or test name remove the need? Rename instead.
2. Can extracting a small helper with a descriptive name remove the need? Extract instead.
3. Only if the code is still surprising after that, write a short comment explaining why.

## Examples

Bad:

```go
// loop over users
for _, u := range users {
    // check if user is active
    if u.Active {
        count++ // increment count
    }
}
```

Good:

```go
for _, u := range users {
    if u.Active {
        activeCount++
    }
}
```

Good (counter-intuitive, so the comment earns its place):

```go
// Check the expiry before the signature: expired tokens must return 401, not 403.
if token.Expired(now) {
    return ErrTokenExpired
}
return token.Verify(key)
```

```go
tests := []struct {
    name  string
    input int
    want  int
}{
    {name: "zero returns zero", input: 0, want: 0},
    // MaxInt32 overflows in the legacy encoder, so the result wraps to negative by design.
    {name: "max int32 wraps", input: math.MaxInt32, want: -1},
}
```

## Exceptions

- **Tool annotations** that change how code is built are not comments for readers and are always allowed: go-train DI tags (`// @Injection(token)`, `/* @Inject(token) */`), `//go:generate`, `//go:build`, `//go:embed`, `//nolint:` with a reason.
- If the project's linter requires doc comments on exported identifiers, keep them short and meaningful rather than skipping them.
