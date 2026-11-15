---
name: go-train-di
description: Wiring dependencies with the go-train-framework DI codegen - @Injection and @Inject annotations, the no-getter flag, and regenerating AppContext. Use whenever adding or changing a constructor in a backend service, or reviewing how dependencies are wired.
---

# go-train DI

Framework: https://github.com/Shitcode-Swamp/go-train-framework. Never hand-write the wiring or construct a dependency inside another layer; the codegen builds `AppContext`.

## Annotations

- `// @Injection(token)` in a constructor's doc comment registers it. Tokens are unique `camelCase` names.
- `/* @Inject(token) */` inline after a constructor parameter asks for the dependency registered under `token`.
- `@Injection(token, no-getter)` skips the `AppContext` getter. Use it for everything nothing outside `AppContext` needs to reach, which is usually everything except controllers.
- Parameters are the consumer's own interfaces (or framework interfaces like `store.Store[T]`).
- Declaration order doesn't matter; the generator resolves it.

## Constructor limits

The generator types each `AppContext` field as `*<constructor's package>.<returned type>` and calls the constructor as `x := pkg.NewX(...)`. So an `@Injection` constructor must:

- **Return exactly one value**, no `error`.
- **Return a pointer to a non-generic struct declared in its own package.** Returning an interface, a generic type like `store.Store[T]`, or another package's type (`*pgxpool.Pool`) generates broken code or makes the generator panic.

To inject such a value, wrap it in a local struct that embeds it, and register that struct's constructor:

```go
type OrderStore struct {
    store.Store[entity.Order]
}

type Postgres struct {
    *pgxpool.Pool
}
```

```go
// @Injection(orderController)
func NewOrderController(payOrder orderPayer /* @Inject(payOrder) */) *OrderController {
    return &OrderController{payOrder: payOrder}
}
```

These annotations are required by the codegen and are allowed by `minimal-comments`.

## Regenerating

Run after adding or changing any annotated constructor:

```sh
go tool go-train -root=./internal -ctxPath=./internal/app
```

- Install if needed: `go get -tool github.com/Shitcode-Swamp/go-train-framework/cmd/go-train@latest`.
- Match `-root` and `-ctxPath` to the project, or pass a `go-train.yaml` with `-config`.
- The generated file is output only. Never edit it by hand; change the annotations and regenerate.

## Review checklist

- [ ] Every constructor has `@Injection`, every injected parameter has `@Inject`.
- [ ] Every `@Injection` constructor returns one pointer to a local, non-generic struct.
- [ ] The generated `AppContext` is up to date and not hand-edited.
