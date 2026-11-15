---
name: go-interfaces
description: Depend on interfaces instead of concrete types, inject them through constructors, and declare each interface in the file that uses it (duplication across files is intentional). Use whenever a Go type needs a dependency, or when writing, editing or reviewing Go code.
---

# Interfaces

## Interfaces over direct dependencies

When a type needs something that does I/O, has side effects, or belongs to another layer or package, depend on an **interface** and receive the implementation through the constructor. Don't build the dependency inside the type, and don't take a concrete type from another package as a field or parameter.

- **Accept interfaces, return structs:** constructors take interfaces and return concrete types.
- **Keep interfaces small:** list only the methods the consumer actually calls.
- Plain values with no behavior to swap (config, entities, DTOs) don't need interfaces.
- Go interfaces are implicit (duck typing), so the implementation never imports or names the interface.

## Declare interfaces where they are used

An interface is a contract owned by the code that **uses** it, so declare it in the **same file** as that code: if `a.go` depends on it, `a.go` declares it.

- Never import an interface from the package that implements it, or from a shared `interfaces`/`contracts` package.
- If another file or package needs a similar interface, it declares its own. **This duplication is intentional:** changing what `a.go` needs then touches only `a.go`. Reviewers must not flag it as duplicated code.
- Two files in one package can't share a name, so name each interface after what that consumer needs (`orderSaver` in one file, `orderFinder` in another). Each one lists only the methods its file calls.
- Keep the interface unexported unless code outside the package needs to name it.
- **Exception: go-train-framework's own interfaces** (`store.Store[T]`, `store.AggregateStore[T, U]`, `store.Transactor`) are used directly. They are a stable framework contract, not something another layer of this project owns.

```go
// usecase/create_order.go
type orderSaver interface {
    Save(ctx context.Context, o *entity.Order) error
}

type CreateOrder struct {
    orders orderSaver
}

func NewCreateOrder(orders orderSaver) *CreateOrder {
    return &CreateOrder{orders: orders}
}
```

```go
// usecase/get_order.go
type orderFinder interface {
    ByID(ctx context.Context, id string) (*entity.Order, error)
}
```

The repository implements both without knowing either exists.
