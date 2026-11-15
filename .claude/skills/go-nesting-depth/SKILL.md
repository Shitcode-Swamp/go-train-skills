---
name: go-nesting-depth
description: Keep Go control-flow nesting (if/else, loops, switch/case) at depth 3 or less, and how to flatten deeper code. Strong recommendation, not a strict rule. Use whenever writing, editing or reviewing Go functions.
---

# Nesting Depth

Keep control-flow depth at **3 or less**. Count each nested `if`/`else`, `for`, `switch`/`select` and `case` body, and each closure that contains control flow.

Depth above 3 is a signal that something isn't right. Fix it with:

- **Guard clauses and early returns** instead of `if ... else` pyramids.
- **`continue`** in loops instead of wrapping the loop body in an `if`.
- **Extracting a function or method** with a name that says what the inner block does.
- **Moving behavior to the type that owns the data.**

This is a strong recommendation, not a hard rule. Deeper code is acceptable when splitting it would make it harder to read. Reviewers report it as minor, not blocking.

```go
// Bad: depth 4
for _, o := range orders {
    if o.Paid {
        for _, item := range o.Items {
            if item.Stock < item.Qty {
                return ErrOutOfStock
            }
        }
    }
}

// Good: guard clause + behavior on the entity
for _, o := range orders {
    if !o.Paid {
        continue
    }
    if err := o.CheckStock(); err != nil {
        return err
    }
}
```
