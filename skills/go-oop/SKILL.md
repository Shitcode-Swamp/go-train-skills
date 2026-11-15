---
name: go-oop
description: How to benefit from OOP in Go - encapsulation, behavior on the owning struct, polymorphism through interfaces, composition. Use whenever designing types or writing, editing or reviewing Go code.
---

# OOP in Go

Go has no classes, but the code should still benefit from OOP:

- **Encapsulation:** structs keep their fields unexported when they have invariants to protect. Create them with a constructor (`NewXxx`) that sets them up in a valid state, and change them through methods.
- **Behavior lives with the data:** if a function mostly reads or changes one struct's fields, make it a method on that struct. Avoid "anemic" structs that only carry data while free functions do all the work on them.
- **Polymorphism through interfaces:** pick behavior by passing a different implementation, not by `switch`ing on a type or a kind field.
- **Composition over inheritance:** reuse behavior by embedding or holding other types. Don't build deep embedding chains to imitate class hierarchies.
- **Single responsibility:** a type has one concern, so it has only one reason to change. If changes coming from different concerns (business rules, storage format, report layout) would all edit the same type, split it so each change touches one type.

```go
// Bad: anemic struct, logic outside
func applyDiscount(o *Order, pct int) { o.Total = o.Total * (100 - pct) / 100 }

// Good: the entity guards its own state
func (o *Order) ApplyDiscount(pct int) error {
    if pct < 0 || pct > 100 {
        return ErrInvalidDiscount
    }
    o.total = o.total * (100 - pct) / 100
    return nil
}
```
