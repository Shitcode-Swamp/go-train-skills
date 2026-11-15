---
name: go-train-repository
description: Data access on go-train-framework's store package - usecases depend on store.Store[T] directly, a per-entity repository struct embedding it makes it injectable, aggregations, filters DSL, transactions, store errors passed through as-is. Use whenever writing, editing or reviewing code that talks to the database.
---

# go-train Repository

Package: `github.com/Shitcode-Swamp/go-train-framework/src/core/persistance/store` (package `store`).

`store.Store[T]`, `store.AggregateStore[T, U]` and `store.Transactor` are already interfaces, so **usecases depend on them directly**. Don't wrap them in a hand-written repository interface or re-declare their methods.

```go
type OrdersUsecase struct {
    orders store.Store[order.Order]
}
```

## Making a store injectable

The DI codegen can't register a constructor that returns `store.Store[T]` (see `go-train-di`). For each entity, create a repository package under `source/repositories/` with a struct that **embeds** the store, and register its constructor. The embedding makes `*OrdersRepository` satisfy `store.Store[order.Order]`.

```go
// source/repositories/orders/orders.go
package orders

type OrdersRepository struct {
    store.Store[order.Order]
}

// @Injection(ordersRepository, no-getter)
func NewOrdersRepository(db *postgres.DB /* @Inject(postgresDB) */) *OrdersRepository {
    return &OrdersRepository{Store: postgre.New[order.Order](db.Pool, "orders")}
}
```

```go
// source/repositories/postgres/postgres.go
package postgres

type DB struct {
    *pgxpool.Pool
}
```

`postgres.DB` embeds `*pgxpool.Pool` for the same codegen reason; how it connects is up to the project. For Mongo, build the store with `mongo.New[T](collection)` from `.../persistance/mongo` (alias it, for example `mongodb`, since the driver's package is also called `mongo`).

## Aggregations

Aggregations go through `store.AggregateStore[T, U]`, a pipeline of `store.Stage` values (`Match`, `Group`, `Sort`, `Limit`, `Skip`, `Join`, `Project`, `Convert`). Only Postgres implements it, via `.../persistance/postgre/aggregate` (package `postgre_aggregate`, so alias it, for example `pgaggregate`). Usecases depend on `store.AggregateStore[T, U]` directly, made injectable with an embedding struct in its own repository package:

```go
// source/repositories/orderstats/orderstats.go
package orderstats

type OrderStatsRepository struct {
    store.AggregateStore[order.Order, order.Summary]
}

// @Injection(orderStatsRepository, no-getter)
func NewOrderStatsRepository(db *postgres.DB /* @Inject(postgresDB) */) *OrderStatsRepository {
    return &OrderStatsRepository{AggregateStore: pgaggregate.New[order.Order, order.Summary](db.Pool, "orders")}
}
```

## Custom queries

Write a repository method by hand only for queries neither `Store[T]` nor `AggregateStore[T, U]` can express. A usecase that needs it declares its own interface in its file, embedding `store.Store[T]` if it also needs CRUD:

```go
type orderLocker interface {
    store.Store[order.Order]
    LockForUpdate(ctx context.Context, id string) (order.Order, error)
}
```

## Filters

`filters.Field[T, V]("column").Is(filters.Eq(v))`, combined with `.And`/`.Or`. Operators: `Eq`, `Ne`, `Gt`, `Gte`, `Lt`, `Lte`, `In`, `Not`. Column names come from `store:"column[,pk]"` tags. Declare reused fields once as package-level variables.

## Transactions

Inject a `store.Transactor` (`postgre.NewDatabase(pool)`, made injectable the same way) and call `WithTransaction(ctx, func(txCtx context.Context) error {...})`, using the same stores with `txCtx`. For read-modify-write under concurrency use `store.IsolationSerializable` and retry on `store.ErrConflict`.

## Errors

Return store errors as-is. The `store` sentinels (`store.ErrNotFound`, `store.ErrDuplicateKey`, `store.ErrConflict`, ...) are already backend-neutral, so don't translate or wrap them. Upper layers check them with `errors.Is`.
