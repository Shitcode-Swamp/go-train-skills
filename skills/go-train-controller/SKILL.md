---
name: go-train-controller
description: Writing the controller layer on go-train-framework's controller contract - controller.New(app) building the router outside DI, one package per entity with Controller/NewController and a usecase interface, one route per file, parsing input, errors.go mapping errors to HTTP status codes, route docs. Use whenever writing, editing or reviewing HTTP handlers.
---

# go-train Controller

A controller is the HTTP edge. It:

- **Parses** path, query and body into the usecase's input types, and validates the format (required fields, UUIDs, JSON). Bad input is `400` before the usecase is called.
- **Calls** a usecase through an interface declared in the controller package.
- **Maps** errors to status codes and results to response types.

No business logic, no repository or DB access.

## Framework contract

From `github.com/Shitcode-Swamp/go-train-framework/v3/src/core/controller/contract`:

- Handlers are `contract.HandlerFunc`: `func(ctx *contract.Context)`, where `contract.Context` is `gin.Context`.
- Register routes on a `contract.RouterGroup` (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `Group`, `Use`).
- Document request shapes on the returned docs: `.RouteDocs().Body(req{})`, `.Query(...)`, `.Path(...)`.
- `Default()` from `.../v3/src/core/controller` (import it as `gotrain`) creates the server, a `contract.Controller`.

## controller.New: the router is not in DI

Controllers and the router have **no `@Injection`**. `source/controller/controller.go` exposes `New(app)`, which `main.go` calls with the generated `AppContext`:

```go
// source/controller/controller.go
package controller

func New(app *appctx.AppContext) contract.Controller {
    c := gotrain.Default()
    product.NewController(app.Productusecase()).Register(c)
    return c
}
```

Each new entity adds one line here, built from its usecase getter (`go-train-di`).

## One package per entity

`source/controller/<entity>/`, with the same name as the usecase package. Import that usecase package with an alias: `productusecase`.

| File | Contents |
|---|---|
| `controller.go` | the `usecase` interface, `Controller`, `NewController`, `Register`, and types and helpers shared by several routes (`toResponse`, `parseID`) |
| `errors.go` | controller error sentinels, `writeError`, `statusFor` |
| `create.go`, `get.go`, `list.go`, `edit.go`, `delete.go` | **one route per file**, with the request and response types only that route uses |

- Exported names are `Controller` and `NewController` (no stutter: `product.Controller`).
- The usecase dependency is one interface named **`usecase`**, listing the methods the controller calls.

```go
// source/controller/product/controller.go
package product

type usecase interface {
    Create(ctx context.Context, in productusecase.CreateInput) (entity.Product, error)
    Get(ctx context.Context, id string) (entity.Product, error)
    Delete(ctx context.Context, id string) error
}

type Controller struct {
    products usecase
}

func NewController(products usecase) *Controller {
    return &Controller{products: products}
}

func (c *Controller) Register(r contract.RouterGroup) {
    r.POST("/products", c.create).RouteDocs().Body(productRequest{})
    r.GET("/products/:id", c.get)
    r.DELETE("/products/:id", c.delete)
}

func parseID(ctx *contract.Context) (string, error) {
    id, err := uuid.Parse(ctx.Param("id"))
    if err != nil {
        return "", errInvalidID
    }
    return id.String(), nil
}
```

```go
// source/controller/product/get.go
func (c *Controller) get(ctx *contract.Context) {
    id, err := parseID(ctx)
    if err != nil {
        writeError(ctx, err)
        return
    }
    p, err := c.products.Get(ctx.Request.Context(), id)
    if err != nil {
        writeError(ctx, err)
        return
    }
    ctx.JSON(http.StatusOK, toResponse(p))
}
```

## Value objects in requests and responses

Value objects marshal themselves: `money.Price` implements `MarshalJSON` and `UnmarshalJSON` with validation, so request and response types use it directly. No separate price DTOs. When binding fails, unwrap the value object's sentinel with `errors.Is` so the client gets the precise `400` message.

## Errors

`errors.go` maps errors with `errors.Is`:

| Error | Status |
|---|---|
| `store.ErrNotFound` | 404 |
| entity and value-object validation sentinels, controller input errors (`errInvalidID`, `errInvalidBody`) | 400 |
| anything else | 500, with a generic `{"error":"internal error"}` body |

```go
// source/controller/product/errors.go
var (
    errInvalidID      = errors.New("id must be a valid UUID")
    errInvalidBody    = errors.New("request body is not valid JSON")
    errInternalServer = errors.New("internal error")
)

type errorResponse struct {
    Error string `json:"error"`
}

func writeError(ctx *contract.Context, err error) {
    status := statusFor(err)
    if status == http.StatusInternalServerError {
        err = errInternalServer
    }
    ctx.JSON(status, errorResponse{Error: err.Error()})
}

func statusFor(err error) int {
    switch {
    case errors.Is(err, store.ErrNotFound):
        return http.StatusNotFound
    case errors.Is(err, errInvalidID),
        errors.Is(err, errInvalidBody),
        errors.Is(err, entity.ErrEmptyProductName):
        return http.StatusBadRequest
    default:
        return http.StatusInternalServerError
    }
}
```

Unknown errors never leak their details into the response body.

## Review checklist

- [ ] No `@Injection` on controllers; each is built in `controller.New(app)` from a usecase getter.
- [ ] Package per entity with `Controller`, `NewController` and a `usecase` interface; usecase package imported with an alias.
- [ ] One route per file; route-only types sit in that route's file.
- [ ] Path IDs are validated with `uuid.Parse` and passed on in canonical form.
- [ ] Every error goes through `writeError`; 500s use the generic body.
