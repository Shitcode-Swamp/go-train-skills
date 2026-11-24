---
name: go-interfaces
description: Depend on interfaces instead of concrete types, inject them through constructors, and declare each interface in the consumer's package next to the struct that holds it, named by what it is (duplication across packages is intentional). Use whenever a Go type needs a dependency, or when writing, editing or reviewing Go code.
---

# Interfaces

## Interfaces over direct dependencies

When a type needs something that does I/O, has side effects, or belongs to another layer or package, depend on an **interface** and receive the implementation through the constructor. Don't build the dependency inside the type, and don't take a concrete type from another package as a field or parameter.

- **Accept interfaces, return structs:** constructors take interfaces and return concrete types.
- **Keep interfaces small:** list only the methods the consumer actually calls.
- Plain values with no behavior to swap (config, entities, DTOs) don't need interfaces.
- Go interfaces are implicit (duck typing), so the implementation never imports or names the interface.

## Declare interfaces where they are used

An interface is a contract owned by the code that **uses** it, so declare it in the consumer's package, **next to the struct that holds the dependency** (for a controller, in `controller.go` beside `Controller`).

- Never import an interface from the package that implements it, or from a shared `interfaces`/`contracts` package.
- One interface per dependency, listing every method the consumer calls. Don't split it into per-method interfaces across the files that call them.
- If another package needs a similar interface, it declares its own. **This duplication is intentional:** changing what one package needs touches only that package. Reviewers must not flag it as duplicated code.
- Name it by what it is (`usecase`, `productRepository`, `paymentGateway`), not by a single-action role (`productCreator`, `paymentCharger`). See `go-naming`.
- Keep the interface unexported unless code outside the package needs to name it.
- **Exception: go-train-framework's own interfaces** (`store.Store[T]`, `store.AggregateStore[T, U]`, `store.Transactor`) are used directly. They are a stable framework contract, not something another layer of this project owns.

```go
// source/controller/product/controller.go
type usecase interface {
    Create(ctx context.Context, in productusecase.CreateInput) (entity.Product, error)
    Get(ctx context.Context, id string) (entity.Product, error)
}

type Controller struct {
    products usecase
}
```

```go
// source/usecases/order/usecase.go
type paymentGateway interface {
    Charge(ctx context.Context, orderID string) error
    Refund(ctx context.Context, orderID string) error
}
```

The usecase and the payments client implement these without knowing they exist.
