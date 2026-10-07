# 09 — Quote Generation Flow

Diagram: [diagrams/quote-flow.md](diagrams/quote-flow.md)

## What actually exists

Each product is **a single form** (`POST /api/quotes/<product>`). There are no separate "customer details", "coverage" or multi-step wizards:

- the **customer** is the signed-in user; the API creates a `customer.customers` row for that user on the first quote if none exists (staff users have none by default);
- **coverage** exists only for Motor (`coverageCode`);
- the quote is **saved immediately** with status `quoted` and a 30-day expiry (`expires_at`).

Multi-step flows and customer selection by staff are planned ([17](17_Future_Enhancements.md)).

## Pipeline (all products)

1. Policy check for the product permission (`401` / `403`).
2. Read `base_rate` from `product.products` (`404` if the product is inactive).
3. Product-specific lookup: coverage loading (Motor) or country risk factor (Travel).
4. `PremiumCalculator.<Product>(...)` — pure C#, throws `ArgumentException` → `400 { error }`.
5. One transaction: ensure customer → insert `quote.quotes` (number from `quote.fn_next_quote_number()`) → insert the detail row → commit.
6. Return `201` + summary read from `quote.v_quote_summary`.

Tax is 10 % (`PremiumCalculator.TaxRate`); amounts are rounded to 2 decimals, midpoint away from zero; `total = base + tax`.

## Fields and rules (from `ILAPI.Core/Models.cs` and `PremiumCalculator.cs`)

### Motor — Flutter `/motor`

| Field | Type | Validation |
|---|---|---|
| `make`, `model` | text | required in UI |
| `year` | int | UI default = current year − 3 |
| `vehicleValue` | number | > 0 |
| `driverAge` | int | 17–100 |
| `claimsLast5Years` | int | ≥ 0 |
| `coverageCode` | `COMPREHENSIVE` \| `THIRD_PARTY` | must exist in `product.coverages` |

`base = vehicleValue × rate × ageFactor × (1 + 0.15 × claims) × (1 + loading%/100)`; `ageFactor` = 1.6 (< 25), 1.3 (≥ 70), else 1.0; minimum 150.

### Home — Flutter `/home`

| Field | Validation |
|---|---|
| `propertyType` (House, Flat, Bungalow), `postcode` | text |
| `yearBuilt` | 1800 … current year |
| `rebuildValue` | > 0 |
| `contentsValue` | ≥ 0 |

`base = (rebuildValue × rate + contentsValue × rate × 1.5) × ageFactor`, `ageFactor` 1.2 when the building is older than 50 years; minimum 25.

### Travel — Angular `/travel`

| Field | Validation |
|---|---|
| `destinationCode` | must exist in `common.countries` |
| `startDate`, `endDate` | `endDate ≥ startDate` |
| `travellers` | > 0 |
| `tripCost` | ≥ 0 |

`base = (tripCost × rate + days × travellers × 1.5) × countryRiskFactor`, `days` counts both end dates; minimum 25.

### Liability — Angular `/liability`

| Field | Validation |
|---|---|
| `businessType` | text |
| `annualTurnover` | > 0 |
| `employees` | ≥ 0 |
| `limitAmount` | > 0 |

`base = annualTurnover × rate × limitFactor + employees × 12`; `limitFactor = 1 + (limit/1,000,000 − 1) × 0.25`, at least 0.5; minimum 100.

> These formulas, the 10 % tax and the minimums are **development placeholders** written for this project, not approved insurance pricing.

## Where the UI lives

| Product | Form | Result display |
|---|---|---|
| Motor, Home | `ILWEBFlutter/lib/shared/quote_form_view.dart` (field definitions per product in the same file) | card under the form |
| Travel, Liability | `ILWEBAngular/src/app/shared/quote-form/` driven by `travel/models/travel.definition.ts` / `liability/models/liability.definition.ts` | `shared/quote-result/` at `/<module>/result` |

## Quote status and expiry

New quotes are `quoted`. Statuses `draft`, `accepted`, `expired`, `cancelled` exist in the schema (`CHECK` constraint) but no API changes a status yet; `quote.fn_expire_quotes()` is not scheduled.

## Tests covering this

`ILAPI.Tests/PremiumCalculatorTests.cs` (10 tests: rates, factors, minimum premium, validation errors) and the browser end-to-end script, which creates a Travel and a Liability quote through Angular for each staff role ([15](15_Testing_Strategy.md)).
