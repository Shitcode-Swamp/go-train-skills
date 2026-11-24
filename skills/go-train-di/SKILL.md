---
name: go-train-di
description: Wiring dependencies with the go-train-framework DI codegen - @Injection and @Inject annotations, what belongs in AppContext (repositories and usecases only), getters, constructor limits, regenerating source/app, and how main.go hands AppContext to controller.New. Use whenever adding or changing a constructor in a backend service, or reviewing how dependencies are wired.
---

# go-train DI

Framework: `github.com/Shitcode-Swamp/go-train-framework/v3`. Use the `/v3` module path in every import and in the `tool` directive. Never hand-write the wiring of repositories and usecases; the codegen builds `AppContext`.

## What goes into AppContext

- `AppContext` holds **only repositories and usecases** (plus what they need, such as the DB connection).
- **Usecases get a getter**: `@Injection(productUsecase)`. Everything else is `no-getter`.
- **Controllers and the router are not in DI.** `main.go` creates the `AppContext` and passes it to `controller.New(app)`, which builds the feature controllers from the usecase getters (see `go-train-controller`).

## Annotations

- `// @Injection(token)` in a constructor's doc comment registers it. Tokens are unique `camelCase` names: `postgres`, `productStore`, `productUsecase`.
- `/* @Inject(token) */` inline after a constructor parameter asks for the dependency registered under `token`.
- `@Injection(token, no-getter)` skips the `AppContext` getter.
- Parameters are the consumer's own interfaces or framework interfaces like `store.Store[T]`.
- Declaration order doesn't matter; the generator resolves it.

```go
// source/usecases/product/usecase.go
// @Injection(productUsecase)
func NewUsecase(records store.Store[entity.ProductRecord] /* @Inject(productStore) */) *Usecase {
    return &Usecase{records: records}
}
```

These annotations are required by the codegen and are allowed by `minimal-comments`.

## Constructor limits

The generator types each `AppContext` field as `*<constructor's package>.<returned type>` and calls the constructor as `x := pkg.NewX(...)`. So an `@Injection` constructor must:

- **Return exactly one value, no `error`.** A constructor that can't work without valid config (missing or bad `POSTGRES_DSN`) **panics** with a clear message; the app can't start anyway.
- **Return a pointer to a non-generic struct declared in its own package.** Returning an interface, a generic type like `store.Store[T]`, or another package's type (`*pgxpool.Pool`) generates broken code or makes the generator panic.

To inject such a value, wrap it in a local struct that embeds it:

```go
// source/repository/postgres.go
type Postgres struct {
    *pgxpool.Pool
}

// @Injection(postgres, no-getter)
func NewPostgres() *Postgres {
    dsn := os.Getenv("POSTGRES_DSN")
    if dsn == "" {
        panic("POSTGRES_DSN is not set")
    }
    pool, err := pgxpool.New(context.Background(), dsn)
    if err != nil {
        panic(fmt.Errorf("connect to postgres: %w", err))
    }
    return &Postgres{Pool: pool}
}
```

## Configuration and regenerating

Config lives in `go-train.yaml` at the repo root:

```yaml
root: ./source
output: ./source/app
```

Run after adding or changing any annotated constructor:

```sh
go tool go-train -config=./go-train.yaml
```

- Install if needed: `go get -tool github.com/Shitcode-Swamp/go-train-framework/v3/cmd/go-train@latest`.
- It writes a single `app.go` in package `app_context` to `source/app`. Keep nothing else in that folder. Import it with an alias: `appctx "<module>/source/app"`. The package name comes from the framework, an accepted exception to `go-naming`.
- The generated file is output only. Never edit it by hand; change the annotations and regenerate.
- Getter names come from `cases.Title(token)`: only the first letter is upper-cased and the rest lower-cased, so `productUsecase` gives `Productusecase()`.

## Execution point

`main.go` at the repo root builds the context and hands it to the controller layer:

```go
func main() {
    app := appctx.NewAppContext()
    c := controller.New(app)
    if err := c.Run(":8080"); err != nil {
        log.Fatal(err)
    }
}
```

## Review checklist

- [ ] Every repository and usecase constructor has `@Injection`; every injected parameter has `@Inject`.
- [ ] Controllers and the router have no `@Injection`; they are built in `controller.New(app)`.
- [ ] Usecases have getters; everything else is `no-getter`.
- [ ] Every `@Injection` constructor returns one pointer to a local, non-generic struct, and panics instead of returning an error.
- [ ] Imports use the `/v3` module path.
- [ ] The generated `source/app/app.go` is up to date and not hand-edited.
