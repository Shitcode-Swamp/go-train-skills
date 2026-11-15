---
name: service-layers
description: Backend service architecture - Controller, Usecase, Repository layers plus business entities, what each layer may do, and the package layout. Use whenever planning, writing or reviewing backend service code. Framework details live in go-train-di, go-train-controller and go-train-repository.
---

# Service Layers

```
Controller  ->  Usecase  ->  store.Store[T]  ->  DB
                   |
                   +------>  external API clients
```

Every arrow is an interface. The controller and API-client arrows use interfaces declared in the consuming file (`go-interfaces` skill); the DB arrow uses the framework's `store` interfaces directly. A layer never builds or names the concrete type of another layer. All wiring goes through DI (`go-train-di` skill).

## Controller

Parses user input into the usecase's input types, calls a usecase, and maps errors to status codes. No business logic, no DB access. Details: `go-train-controller`.

## Usecase

Business logic only:

- Works with **business entities**, not HTTP requests or DB rows.
- Gets every dependency injected: `store.Store[T]` for data, interfaces of its own for external API clients, clocks and ID generators. **Never** a direct dependency, even for a "simple" API call.
- Returns its own business-rule errors (`entity.ErrOrderAlreadyPaid`) and passes `store` errors (`store.ErrNotFound`) through unchanged. The controller maps both.
- Never imports `net/http`, gin, controller packages or DB drivers.

## Entities

Structs with **behavior**. An entity validates and changes its own state through methods (`order.Pay()`), so the usecase coordinates and the entity enforces the rules. Entities depend on no other layer.

## Repository

The only code that talks to the DB: injectable per-entity stores built on the framework, plus custom queries. Details: `go-train-repository`.

## Package layout

Follow the existing layout if there is one. Otherwise:

```
internal/
  entity/        business entities and business-rule errors
  usecase/       one file per usecase, each declaring the interfaces it needs
  repository/    injectable stores and custom queries
  controller/    HTTP handlers and route registration
  client/        external API clients
```

## Example: entity and usecase

```go
// internal/entity/order.go
var ErrOrderAlreadyPaid = errors.New("order already paid")

type Order struct {
    ID   string `store:"id,pk"`
    Paid bool   `store:"paid"`
}

func (o *Order) Pay() error {
    if o.Paid {
        return ErrOrderAlreadyPaid
    }
    o.Paid = true
    return nil
}
```

```go
// internal/usecase/pay_order.go
var orderID = filters.Field[entity.Order, string]("id")

type paymentGateway interface {
    Charge(ctx context.Context, orderID string) error
}

type PayOrder struct {
    orders   store.Store[entity.Order]
    payments paymentGateway
}

// @Injection(payOrder, no-getter)
func NewPayOrder(
    orders store.Store[entity.Order] /* @Inject(orderStore) */,
    payments paymentGateway /* @Inject(paymentClient) */,
) *PayOrder {
    return &PayOrder{orders: orders, payments: payments}
}

func (uc *PayOrder) Execute(ctx context.Context, id string) error {
    byID := orderID.Is(filters.Eq(id))
    o, err := uc.orders.One(ctx, byID)
    if err != nil {
        return err
    }
    if err := o.Pay(); err != nil {
        return err
    }
    if err := uc.payments.Charge(ctx, o.ID); err != nil {
        return fmt.Errorf("charge order %s: %w", o.ID, err)
    }
    _, err = uc.orders.Update(ctx, byID, o)
    return err
}
```

## Review checklist

- [ ] No layer constructs or names the concrete type of another layer.
- [ ] Usecases use `store.Store[T]` directly instead of re-declaring its methods.
- [ ] External APIs are injected into usecases, never called directly.
- [ ] Controllers hold no business logic; usecases hold no HTTP or DB-driver code.
- [ ] Rules about one entity's state live on that entity.
