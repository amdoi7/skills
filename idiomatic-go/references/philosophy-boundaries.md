# Philosophy: Boundaries, Parsing & Trusted Data

> "Parse, don't validate."
>
> "Make illegal states unrepresentable."

Use this reference when the design problem starts at the boundary: raw input, DTO churn, repeated validation, weak types, or framework objects leaking inward.

## Core Idea

Validate once at the boundary, convert into trusted types, and stop re-checking the same facts everywhere else.

```text
raw input
    │
    ▼
parse + validate at boundary
    │
    ▼
trusted domain types
    │
    ▼
internal code assumes invariants already hold
```

## Rules

- Parse raw input into strong types as early as possible
- Keep DTOs, ORM rows, and transport payloads at the edge
- Let internal code consume trusted types, not raw strings and maps
- Keep invariants inside constructors, parsers, or value objects
- Fail fast when boundary contracts are broken

## Good Shape

```go
type OrderID string

type Money struct {
    cents int64
}

func ParseOrderID(s string) (OrderID, error) {
    if s == "" {
        return "", errors.New("empty order id")
    }
    return OrderID(s), nil
}

func ParseMoney(cents int64) (Money, error) {
    if cents <= 0 {
        return Money{}, errors.New("amount must be positive")
    }
    return Money{cents: cents}, nil
}
```

```go
func CreateOrderHandler(w http.ResponseWriter, r *http.Request) {
    var req createOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid json", http.StatusBadRequest)
        return
    }

    id, err := ParseOrderID(req.OrderID)
    if err != nil {
        http.Error(w, "invalid order id", http.StatusBadRequest)
        return
    }

    amount, err := ParseMoney(req.AmountCents)
    if err != nil {
        http.Error(w, "invalid amount", http.StatusBadRequest)
        return
    }

    if err := orderApp.Create(r.Context(), id, amount); err != nil {
        http.Error(w, "create order failed", http.StatusInternalServerError)
        return
    }
}
```

## Bad Shape

```go
func CreateOrder(ctx context.Context, orderID string, amount float64) error {
    if orderID == "" {
        return errors.New("empty order id")
    }
    if amount <= 0 {
        return errors.New("invalid amount")
    }
    return saveOrder(ctx, orderID, amount)
}

func saveOrder(ctx context.Context, orderID string, amount float64) error {
    if orderID == "" {
        return errors.New("empty order id")
    }
    if amount <= 0 {
        return errors.New("invalid amount")
    }
    return nil
}
```

The problem is not only duplication. The program never decides where truth becomes trustworthy.

## Common Boundary Smells

| Smell | Why it hurts | Better |
| --- | --- | --- |
| repeated `if x == ""` checks | truth is not established once | parse once, then trust |
| DTOs mirrored into domain with no semantic change | extra churn with no new abstraction | keep DTO at boundary |
| framework types passed deep into app code | boundary concerns leak inward | translate at edge |
| IDs as plain `string` everywhere | semantics disappear | use strong types |
| `float64` money | precision and invariant drift | use domain type |

## Review Questions

- Where does raw input become trusted
- Which package owns parsing and validation
- Are internal APIs still asking callers to prove facts already known
- Are transport or storage details leaking into the domain model
- Can a stronger type remove a whole class of checks
