# Quote Generation Flow

All four products use the same API pattern. Premium rules are computed in C# (`PremiumCalculator`), rates and loadings are read from PostgreSQL, and the quote is saved in one transaction.

```mermaid
sequenceDiagram
    actor U as User
    participant UI as Flutter page (Motor, Home) or Angular page (Travel, Liability)
    participant A as ILAPI QuotesController
    participant P as PremiumCalculator (ILAPI.Core)
    participant D as PostgreSQL

    U->>UI: fill the form, Get quote
    UI->>A: POST /api/quotes/{motor|home|travel|liability} + JWT
    A->>A: policy check (permission claim, e.g. travel.quote), else 401 / 403
    A->>D: read product base_rate (+ coverage loading or country risk factor)
    A->>P: calculate(request, rate, factor)
    P-->>A: base premium, tax, total (or ArgumentException -> 400)
    A->>D: BEGIN
    A->>D: ensure customer row for the user
    A->>D: INSERT quote.quotes (number from quote.fn_next_quote_number())
    A->>D: INSERT detail row (motor.vehicle_quotes / home.property_quotes / travel.trip_quotes / liability.liability_quotes)
    A->>D: COMMIT
    A-->>UI: 201 Created, quote summary (from quote.v_quote_summary)
    UI-->>U: show quote number, premium, tax, total
```

```mermaid
erDiagram
    CUSTOMERS ||--o{ QUOTES : has
    PRODUCTS ||--o{ QUOTES : prices
    QUOTES ||--o| VEHICLE_QUOTES : "motor detail"
    QUOTES ||--o| PROPERTY_QUOTES : "home detail"
    QUOTES ||--o| TRIP_QUOTES : "travel detail"
    QUOTES ||--o| LIABILITY_QUOTES : "liability detail"
    COUNTRIES ||--o{ TRIP_QUOTES : destination
```
