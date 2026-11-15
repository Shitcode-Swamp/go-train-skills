---
name: service-layers
description: Backend service architecture - Controller, Usecase, Repository layers plus domain entities, what each layer may do, and the source/ folder structure with one package per entity, usecase and repository. Use whenever planning, writing or reviewing backend service code. Framework details live in go-train-di, go-train-controller and go-train-repository.
---

# Service Layers

```
Controller  ->  Usecase  ->  store.Store[T]  ->  DB
                   |
                   +------>  external API clients
```

Every arrow is an interface. The controller and API-client arrows use interfaces declared in the consuming file (`go-interfaces` skill); the DB arrow uses the framework's `store` interfaces directly. A layer never builds or names the concrete type of another layer. All wiring goes through DI (`go-train-di` skill).

## Folder structure

```
source/
  app/                    generated DI root (AppContext), never edited by hand
  main/                   execution point: builds AppContext and runs the root controller
  domain/
    order/                one package per domain entity
  usecases/
    orders/               one package per usecase
  repositories/
    orders/               one package per repository
  controllers/            root controller: creates the server, registers every controller
    users/                one package per controller (UsersController)
    orders/
  utils/                  utility functions and constants
```

- **One package per item:** every domain entity, usecase, repository and controller gets its own package inside its layer folder. This keeps names and package-level variables from mixing.
- **Layer structs** are named after the item and the layer: `UsersController`, `OrdersUsecase`, `OrdersRepository`. This is a project convention and takes precedence over the no-stutter rule in `go-naming`.
- Packages from different layers often share a name (`usecases/orders`, `repositories/orders`). Alias imports when one file needs both.

## Controller

Parses user input into the usecase's input types, calls a usecase, and maps errors to status codes. No business logic, no DB access. Details: `go-train-controller`.

## Usecase

Business logic only:

- Works with **domain entities**, not HTTP requests or DB rows.
- Gets every dependency injected: `store.Store[T]` for data, interfaces of its own for external API clients, clocks and ID generators. **Never** a direct dependency, even for a "simple" API call.
- Returns its own business-rule errors (`order.ErrAlreadyPaid`) and passes `store` errors (`store.ErrNotFound`) through unchanged. The controller maps both.
- Never imports `net/http`, gin, controller packages or DB drivers.

## Domain entities

Structs with **behavior**. An entity validates and changes its own state through methods (`o.Pay()`), so the usecase coordinates and the entity enforces the rules. Entities depend on no other layer.

## Repository

The only code that talks to the DB: injectable per-entity stores built on the framework, plus custom queries. Details: `go-train-repository`.

## Example: entity and usecase

```go
// source/domain/order/order.go
package order

var ErrAlreadyPaid = errors.New("order already paid")

type Order struct {
    ID   string `store:"id,pk"`
    Paid bool   `store:"paid"`
}

func (o *Order) Pay() error {
    if o.Paid {
        return ErrAlreadyPaid
    }
    o.Paid = true
    return nil
}
```

```go
// source/usecases/orders/orders.go
package orders

var orderID = filters.Field[order.Order, string]("id")

type paymentGateway interface {
    Charge(ctx context.Context, orderID string) error
}

type OrdersUsecase struct {
    orders   store.Store[order.Order]
    payments paymentGateway
}

// @Injection(ordersUsecase, no-getter)
func NewOrdersUsecase(
    orders store.Store[order.Order] /* @Inject(ordersRepository) */,
    payments paymentGateway /* @Inject(paymentsClient) */,
) *OrdersUsecase {
    return &OrdersUsecase{orders: orders, payments: payments}
}

func (uc *OrdersUsecase) Pay(ctx context.Context, id string) error {
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

- [ ] Code sits in the right `source/` folder, one package per entity, usecase, repository and controller.
- [ ] No layer constructs or names the concrete type of another layer.
- [ ] Usecases use `store.Store[T]` directly instead of re-declaring its methods.
- [ ] External APIs are injected into usecases, never called directly.
- [ ] Controllers hold no business logic; usecases hold no HTTP or DB-driver code.
- [ ] Rules about one entity's state live on that entity.
