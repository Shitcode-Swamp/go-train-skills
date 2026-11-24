---
name: go-train-repository
description: Data access on go-train-framework's store package - usecases use store.Store[entity.XRecord] directly, the repository package holds the DB connection and one method-less store struct per entity, Update/DeleteOne not-found semantics, aggregations, filters DSL, transactions, store errors passed through as-is. Use whenever writing, editing or reviewing code that talks to the database.
---

# go-train Repository

Package: `github.com/Shitcode-Swamp/go-train-framework/v3/src/core/persistance/store` (package `store`). The Postgres adapter is `.../v3/src/core/persistance/postgre`. There is no MySQL adapter and no migration tooling.

`store.Store[T]`, `store.AggregateStore[T, U]` and `store.Transactor` are already interfaces, so **usecases depend on them directly**. `T` is the entity's **record** type (`entity.ProductRecord`, see `service-layers`), never the domain entity.

## The repository package

One package, `source/repository/`, holding:

- the DB connection (`postgres.go`, a `Postgres` struct embedding `*pgxpool.Pool`, see `go-train-di`);
- one store per entity (`product_store.go`).

A store is a struct that embeds `store.Store[entity.XRecord]`, plus a DI constructor. That is all: **no methods and no logic.** The embedding makes `*ProductStore` satisfy `store.Store[entity.ProductRecord]`, and the DI codegen can't register a constructor that returns the interface itself.

```go
// source/repository/product_store.go
type ProductStore struct {
    store.Store[entity.ProductRecord]
}

// @Injection(productStore, no-getter)
func NewProductStore(db *Postgres /* @Inject(postgres) */) *ProductStore {
    return &ProductStore{Store: postgre.New[entity.ProductRecord](db.Pool, "products")}
}
```

Don't wrap framework methods in per-entity `Get`, `List`, `FindByID` and similar methods. That only duplicates the store.

## Using the store in a usecase

The usecase calls `One`, `Many`, `Insert`, `Update`, `DeleteOne` with filters, and converts records with `.Record()` and `.Product()`:

```go
// source/usecases/product/usecase.go
var productID = filters.Field[entity.ProductRecord, string]("id")

type Usecase struct {
    records store.Store[entity.ProductRecord]
}

func byProductID(id string) filters.Filtered[entity.ProductRecord] {
    return productID.Is(filters.Eq(id))
}
```

```go
// source/usecases/product/get.go
func (uc *Usecase) Get(ctx context.Context, id string) (entity.Product, error) {
    record, err := uc.records.One(ctx, byProductID(id))
    if err != nil {
        return entity.Product{}, err
    }
    return record.Product()
}
```

### Not-found semantics you must handle

- `One` returns `store.ErrNotFound` when nothing matches.
- **`Update` returns a nil id** (and no error) when no row matches. Turn that into `store.ErrNotFound`.
- **`DeleteOne` returns nil** when no row matches. Call `One` first so not-found is detected.

```go
id, err := uc.records.Update(ctx, byID, p.Record())
if err != nil {
    return entity.Product{}, err
}
if id == nil {
    return entity.Product{}, store.ErrNotFound
}
```

```go
// Delete checks existence first because DeleteOne reports no error and no row count when nothing matches.
func (uc *Usecase) Delete(ctx context.Context, id string) error {
    byID := byProductID(id)
    if _, err := uc.records.One(ctx, byID); err != nil {
        return err
    }
    return uc.records.DeleteOne(ctx, byID)
}
```

`Insert` always writes the PK column, which is why IDs are generated in the app (`service-layers`).

## Aggregations

Aggregations go through `store.AggregateStore[T, U]`, a pipeline of `store.Stage` values (`Match`, `Group`, `Sort`, `Limit`, `Skip`, `Join`, `Project`, `Convert`). Only Postgres implements it, via `.../v3/src/core/persistance/postgre/aggregate` (package `postgre_aggregate`, so alias it, for example `pgaggregate`). Make it injectable the same way, a method-less struct in `repository`:

```go
// source/repository/order_stats_store.go
type OrderStatsStore struct {
    store.AggregateStore[entity.OrderRecord, entity.OrderSummaryRecord]
}
```

## Custom queries

Only for a query neither `Store[T]` nor `AggregateStore[T, U]` can express, and never as a wrapper around a framework method. Raise it with the user first: it is the one case where a store gets a method. The usecase that needs it declares its own interface, embedding `store.Store[T]` if it also needs CRUD.

## Filters

`filters.Field[T, V]("column").Is(filters.Eq(v))`, combined with `.And`/`.Or`. Operators: `Eq`, `Ne`, `Gt`, `Gte`, `Lt`, `Lte`, `In`, `Not`. Column names come from the record's `store:"column[,pk]"` tags. Declare reused fields and filter helpers once in the usecase package's `usecase.go`. An empty `filters.Filtered[T]{}` matches everything.

## Transactions

Inject a `store.Transactor` (`postgre.NewDatabase(pool)`, made injectable the same way) and call `WithTransaction(ctx, func(txCtx context.Context) error {...})`, using the same stores with `txCtx`. For read-modify-write under concurrency use `store.IsolationSerializable` and retry on `store.ErrConflict`.

## Errors

Return store errors as-is. The `store` sentinels (`store.ErrNotFound`, `store.ErrDuplicateKey`, `store.ErrConflict`, ...) are already backend-neutral, so don't translate or wrap them. Upper layers check them with `errors.Is`.

## Review checklist

- [ ] Stores embed `store.Store[entity.XRecord]` and have no methods.
- [ ] Usecases call the store directly and convert with `.Record()` / `.X()`.
- [ ] `Update` nil id becomes `store.ErrNotFound`; `DeleteOne` is preceded by `One`.
- [ ] Store errors are passed through unwrapped.
