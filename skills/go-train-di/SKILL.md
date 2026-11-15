---
name: go-train-di
description: Wiring dependencies with the go-train-framework DI codegen - @Injection and @Inject annotations, the no-getter flag, constructor limits, configuring and regenerating AppContext in source/app. Use whenever adding or changing a constructor in a backend service, or reviewing how dependencies are wired.
---

# go-train DI

Framework: https://github.com/Shitcode-Swamp/go-train-framework. Never hand-write the wiring or construct a dependency inside another layer; the codegen builds `AppContext`.

## Annotations

- `// @Injection(token)` in a constructor's doc comment registers it. Tokens are unique `camelCase` names.
- `/* @Inject(token) */` inline after a constructor parameter asks for the dependency registered under `token`.
- `@Injection(token, no-getter)` skips the `AppContext` getter. Use it for everything `source/main` doesn't need, which is everything except the root controller.
- Parameters are the consumer's own interfaces (or framework interfaces like `store.Store[T]`).
- Declaration order doesn't matter; the generator resolves it.

```go
// @Injection(ordersController, no-getter)
func NewOrdersController(orders orderPayer /* @Inject(ordersUsecase) */) *OrdersController {
    return &OrdersController{orders: orders}
}
```

These annotations are required by the codegen and are allowed by `minimal-comments`.

## Constructor limits

The generator types each `AppContext` field as `*<constructor's package>.<returned type>` and calls the constructor as `x := pkg.NewX(...)`. So an `@Injection` constructor must:

- **Return exactly one value**, no `error`.
- **Return a pointer to a non-generic struct declared in its own package.** Returning an interface, a generic type like `store.Store[T]`, or another package's type (`*pgxpool.Pool`) generates broken code or makes the generator panic.

To inject such a value, wrap it in a local struct that embeds it, and register that struct's constructor:

```go
type OrdersRepository struct {
    store.Store[order.Order]
}

type DB struct {
    *pgxpool.Pool
}
```

## Configuration and regenerating

Run after adding or changing any annotated constructor:

```sh
go tool go-train -root=./source -ctxPath=./source/app
```

- Install if needed: `go get -tool github.com/Shitcode-Swamp/go-train-framework/cmd/go-train@latest`.
- Instead of flags, a config file passed with `-config=./go-train.yaml` can set the paths. `root` is the folder to scan, `output` the folder to write to. Flags override the file.

  ```yaml
  root: ./source
  output: ./source/app
  ```

- It writes a single `app.go` in package `app_context` to `source/app`. Keep nothing else in that folder. The package name comes from the framework, so it is an accepted exception to `go-naming`.
- The generated file is output only. Never edit it by hand; change the annotations and regenerate.
- Getters capitalise only the first letter of the token: `rootController` becomes `Rootcontroller()`.

## Execution point

`source/main` builds the context and runs the root controller:

```go
// source/main/main.go
package main

func main() {
    ctx := app_context.NewAppContext()
    if err := ctx.Rootcontroller().Run(":8080"); err != nil {
        log.Fatal(err)
    }
}
```

## Review checklist

- [ ] Every constructor has `@Injection`, every injected parameter has `@Inject`.
- [ ] Every `@Injection` constructor returns one pointer to a local, non-generic struct.
- [ ] Only the root controller has a getter.
- [ ] The generated `source/app/app.go` is up to date and not hand-edited.
