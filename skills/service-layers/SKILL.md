---
name: service-layers
description: Backend service architecture - Controller, Usecase, Repository layers plus entities with their DB records, what each layer may do, the source/ and test/ folder layout, entity IDs. Use whenever planning, writing or reviewing backend service code. Framework details live in go-train-di, go-train-controller and go-train-repository.
---

# Service Layers

```
Controller  ->  Usecase  ->  store.Store[XRecord]  ->  DB
                   |
                   +------>  external API clients
```

Every arrow is an interface. The controller and API-client arrows use interfaces declared by the consumer (`go-interfaces` skill); the DB arrow uses the framework's `store` interfaces directly. A layer never builds or names the concrete type of another layer.

The first entity built in a service is the reference implementation: copy its shape for every new entity.

## Folder structure

```
main.go                         builds AppContext, hands it to controller.New, runs the server
go-train.yaml                   DI codegen config
migrations/                     plain SQL, applied by hand (NNNNNN_name.up.sql / .down.sql)
source/
├── app/app.go                  GENERATED DI container: never edit by hand
├── entity/                     ONE package: domain objects + their DB records
│   ├── product.go
│   └── product_record.go
├── repository/                 ONE package: DB connection + one thin store per entity
│   ├── postgres.go
│   └── product_store.go
├── usecases/<entity>/          one package per entity
│   ├── usecase.go              Usecase struct + constructor + shared helpers (filters)
│   └── create.go get.go list.go edit.go delete.go   one action per file
├── controller/
│   ├── controller.go           New(app): builds the router and every feature controller
│   └── <entity>/               one package per entity
│       ├── controller.go       usecase interface, Controller, NewController, Register, shared DTOs
│       ├── errors.go           error sentinels → HTTP status
│       └── create.go get.go list.go edit.go delete.go   one route per file
└── utils/<name>/               value objects and helpers, e.g. utils/money (Price)
test/                           mirrors source/; no _test.go files next to production code
```

- `entity` and `repository` are single packages: entities are referenced everywhere as `entity.Product`, stores as `repository.ProductStore`.
- `usecases/<entity>` and `controller/<entity>` are one package per entity and share the entity's name (`usecases/product`, `controller/product`). The controller imports the usecase package with an alias: `productusecase`.
- No stutter (`go-naming`): `product.Usecase`, `product.Controller`, `repository.ProductStore`, never `product.ProductController`.

## Controller

Parses user input into the usecase's input types, calls a usecase, maps errors to status codes. No business logic, no DB access. Details: `go-train-controller`.

## Usecase

Business logic only:

- One `Usecase` struct per entity in `usecase.go`, one exported method per action, each action in its own file together with its input type.
- Input types are named without stutter: `product.CreateInput`, `product.EditInput`.
- Works with **entities**; uses `store.Store[entity.XRecord]` directly and converts with `.Record()` / `.Product()` (`go-train-repository`).
- Gets every dependency injected: stores for data, interfaces of its own for external API clients, clocks and ID generators. **Never** a direct dependency, even for a "simple" API call.
- Returns entity validation errors and `store` errors (`store.ErrNotFound`) unchanged. The controller maps them.
- Never imports `net/http`, gin, controller packages or DB drivers.

## Entities

Structs with **behavior** that protect their invariants (`go-oop`):

- Domain fields are **unexported** and read through getters (`p.Price()`). Fields hold real value objects (`money.Price`), not primitives.
- Two validating constructors:
  - `NewProduct(...)` for a brand-new entity. It generates the ID.
  - `RestoreProduct(id, ...)` to rebuild one from stored data. `NewProduct` calls it, so validation lives in one place.
- State changes go through methods (`p.Edit(...)`) that validate.
- Entities depend on no other layer.

### DB records

Building the entity from DB data is the **entity's job**, so the storage shape lives next to it in `entity/<name>_record.go`:

- `ProductRecord` has exported, `store`-tagged fields. It is what the store reads and writes.
- `p.Record()` converts the entity to a record.
- `r.Product() (Product, error)` converts back through `RestoreProduct`, returning an error for bad stored data instead of panicking.
- The framework maps exactly one tagged field to one column, so nested structs can't be stored. Split value objects into columns: `price_amount`, `price_currency`.

```go
// source/entity/product.go
type Product struct {
    id    string
    name  string
    price money.Price
}

func NewProduct(name string, price money.Price) (Product, error) {
    return RestoreProduct(uuid.NewString(), name, price)
}

func RestoreProduct(id, name string, price money.Price) (Product, error) {
    if id == "" {
        return Product{}, ErrEmptyProductID
    }
    p := Product{id: id}
    if err := p.Edit(name, price); err != nil {
        return Product{}, err
    }
    return p, nil
}

func (p Product) ID() string          { return p.id }
func (p Product) Price() money.Price  { return p.price }
```

```go
// source/entity/product_record.go
type ProductRecord struct {
    ID            string `store:"id,pk"`
    Name          string `store:"name"`
    PriceAmount   int64  `store:"price_amount"`
    PriceCurrency string `store:"price_currency"`
}

func (p Product) Record() ProductRecord {
    return ProductRecord{ID: p.id, Name: p.name, PriceAmount: p.price.Amount(), PriceCurrency: p.price.Currency()}
}

func (r ProductRecord) Product() (Product, error) {
    price, err := money.NewPrice(r.PriceAmount, r.PriceCurrency)
    if err != nil {
        return Product{}, fmt.Errorf("restore product %s: %w", r.ID, err)
    }
    return RestoreProduct(r.ID, r.Name, price)
}
```

### Value objects

Value objects (`utils/money.Price`) validate in their constructor and implement `MarshalJSON` / `UnmarshalJSON` (with validation), so controllers use them directly in request and response types instead of separate DTOs.

## IDs

- IDs are **random UUID strings generated in the app** by `NewX`, with `uuid.NewString()`. The column is `UUID PRIMARY KEY` with no DB default.
- Reason: the framework's `Insert` always writes the PK column, so app-generated IDs work with it as is, with no custom insert SQL.
- Controllers validate path IDs with `uuid.Parse` and pass on the canonical form (`id.String()`); anything else is `400`.

## Migrations

The framework has no migration tooling. Schema changes are plain SQL files in `migrations/` (`000001_create_products.up.sql` / `.down.sql`), applied by hand.

## Tests

- All tests live under `test/`, mirroring `source/` (`test/usecases/product/`, `test/entity/`), in external `<pkg>_test` packages that use only the exported API. No `_test.go` files next to production code.
- Coverage needs `go test -coverpkg=./source/... ./test/...`, since tests live in other packages.
- Integration tests carry `//go:build integration` and `t.Skip` unless `POSTGRES_TEST_DSN` is set. Run them with `go test -tags integration ./test/...`.

## Review checklist

- [ ] Code sits in the right folder: `entity`, `repository` as single packages; `usecases/<entity>` and `controller/<entity>` per entity; one action or route per file.
- [ ] Entity fields are unexported, hold value objects, and are built through `NewX` / `RestoreX`; DB shape lives in `XRecord` with `Record()` / `X()`.
- [ ] IDs are app-generated UUIDs; path IDs are checked with `uuid.Parse`.
- [ ] No layer constructs or names the concrete type of another layer.
- [ ] Usecases use `store.Store[entity.XRecord]` directly instead of re-declaring its methods.
- [ ] External APIs are injected into usecases, never called directly.
- [ ] Controllers hold no business logic; usecases hold no HTTP or DB-driver code.
- [ ] Tests are under `test/`, in `<pkg>_test` packages.
