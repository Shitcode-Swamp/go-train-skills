---
name: go-train-controller
description: Writing the controller layer on go-train-framework's controller contract - routes, parsing input for usecases, mapping errors to HTTP status codes, route docs. Use whenever writing, editing or reviewing HTTP handlers.
---

# go-train Controller

The controller is the HTTP edge. It:

- **Parses** path, query and body into the usecase's input types, and validates the format (required fields, types, ranges). Bad input is `400` before the usecase is called.
- **Calls** a usecase through an interface declared in the controller file.
- **Maps** errors to status codes (business errors from entities and `store` sentinels such as `store.ErrNotFound`), and results to response DTOs.

No business logic, no repository or DB access.

## Framework contract

From `github.com/Shitcode-Swamp/go-train-framework/src/core/controller/contract`:

- Handlers are `contract.HandlerFunc`: `func(ctx *contract.Context)`, where `contract.Context` is `gin.Context`.
- Register routes on a `contract.RouterGroup` (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `Group`, `Use`).
- Document request shapes on the returned docs: `.RouteDocs().Body(req{})`, `.Query(...)`, `.Path(...)`.

```go
type orderPayer interface {
    Execute(ctx context.Context, orderID string) error
}

type OrderController struct {
    payOrder orderPayer
}

// @Injection(orderController)
func NewOrderController(payOrder orderPayer /* @Inject(payOrder) */) *OrderController {
    return &OrderController{payOrder: payOrder}
}

func (c *OrderController) Register(r contract.RouterGroup) {
    r.POST("/orders/:id/pay", c.pay)
}

func (c *OrderController) pay(ctx *contract.Context) {
    err := c.payOrder.Execute(ctx.Request.Context(), ctx.Param("id"))
    ctx.Status(statusFor(err))
}

func statusFor(err error) int {
    switch {
    case err == nil:
        return http.StatusNoContent
    case errors.Is(err, store.ErrNotFound):
        return http.StatusNotFound
    case errors.Is(err, entity.ErrOrderAlreadyPaid):
        return http.StatusConflict
    default:
        return http.StatusInternalServerError
    }
}
```

Unknown errors map to `500` and their details never go into the response body.
