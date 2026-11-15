---
name: go-train-controller
description: Writing the controller layer on go-train-framework's controller contract - the root controller, one package per controller, routes, parsing input for usecases, mapping errors to HTTP status codes, route docs. Use whenever writing, editing or reviewing HTTP handlers.
---

# go-train Controller

A controller is the HTTP edge. It:

- **Parses** path, query and body into the usecase's input types, and validates the format (required fields, types, ranges). Bad input is `400` before the usecase is called.
- **Calls** a usecase through an interface declared in the controller file.
- **Maps** errors to status codes (business errors from domain entities and `store` sentinels such as `store.ErrNotFound`), and results to response DTOs.

No business logic, no repository or DB access.

## Framework contract

From `github.com/Shitcode-Swamp/go-train-framework/src/core/controller/contract`:

- Handlers are `contract.HandlerFunc`: `func(ctx *contract.Context)`, where `contract.Context` is `gin.Context`.
- Register routes on a `contract.RouterGroup` (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `Group`, `Use`).
- Document request shapes on the returned docs: `.RouteDocs().Body(req{})`, `.Query(...)`, `.Path(...)`.
- `controller.Default()` from `.../src/core/controller` creates the server, a `contract.Controller`.

## Root controller

`source/controllers` holds the root controller. It creates the server, gives each controller its route group, and is the only dependency `source/main` gets from `AppContext`. Each new controller is injected into it.

```go
// source/controllers/controllers.go
package controllers

type routes interface {
    Register(r contract.RouterGroup)
}

type RootController struct {
    server contract.Controller
}

// @Injection(rootController)
func NewRootController(
    users routes /* @Inject(usersController) */,
    orders routes /* @Inject(ordersController) */,
) *RootController {
    server := controller.Default()
    users.Register(server.Group("/users"))
    orders.Register(server.Group("/orders"))
    return &RootController{server: server}
}

func (c *RootController) Run(addr string) error {
    return c.server.Run(addr)
}
```

## One package per controller

Each controller gets its own package in `source/controllers/<name>/`, for example `source/controllers/orders` with `OrdersController`.

```go
// source/controllers/orders/orders.go
package orders

type orderPayer interface {
    Pay(ctx context.Context, id string) error
}

type OrdersController struct {
    orders orderPayer
}

// @Injection(ordersController, no-getter)
func NewOrdersController(orders orderPayer /* @Inject(ordersUsecase) */) *OrdersController {
    return &OrdersController{orders: orders}
}

func (c *OrdersController) Register(r contract.RouterGroup) {
    r.POST("/:id/pay", c.pay)
}

func (c *OrdersController) pay(ctx *contract.Context) {
    err := c.orders.Pay(ctx.Request.Context(), ctx.Param("id"))
    ctx.Status(statusFor(err))
}

func statusFor(err error) int {
    switch {
    case err == nil:
        return http.StatusNoContent
    case errors.Is(err, store.ErrNotFound):
        return http.StatusNotFound
    case errors.Is(err, order.ErrAlreadyPaid):
        return http.StatusConflict
    default:
        return http.StatusInternalServerError
    }
}
```

Unknown errors map to `500` and their details never go into the response body.
