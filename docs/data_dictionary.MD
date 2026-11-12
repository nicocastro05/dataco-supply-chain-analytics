# DataCo Supply Chain Dataset — Data Dictionary

**Source:** `DataCoSupplyChainDataset.csv` (~180k rows, ~96 MB)
**Grain:** One row per **order line item** — uniquely identified by `Order Item Id`.
**Companion files:** `DescriptionDataCoSupplyChain.csv` (official field list, used to build this dictionary), `Tokenized_Access_Logs.csv` (clickstream, see separate section).

## Key conventions used throughout this project

- **"Late" is defined as:** `Days for shipping (real) > Days for shipment (scheduled)`. We prefer this over `Late_delivery_risk` because it is derived directly from the two day-count fields and is auditable. `Late_delivery_risk` will be used as a cross-check.
- **Cancelled / fraud / on-hold orders** are **excluded** from revenue, profit, and delivery-rate analysis. Only `Order Status = 'COMPLETE'` is used for outcome analysis. Rationale: an order that never shipped cannot be late, and cancelled orders would distort profit aggregates.
- **Two "sales"-like columns exist**: `Sales` (pre-discount line-item value) and `Order Item Total` (post-discount, actual revenue). We use **`Order Item Total`** for revenue. `Sales` is retained only for discount computation.
- **Order-level repeated fields** (`Benefit per order`, `Sales per customer`) must never be summed at the line-item grain — they repeat per line item and would double-count. Use `Order Profit Per Order` for line-item profit.

## Fields

### Transaction & timing
| Column | Type | Definition | Analysis Notes |
|---|---|---|---|
| `Type` | String | Payment method (DEBIT, TRANSFER, CASH, PAYMENT) | Categorical; payment-method analysis |
| `Days for shipping (real)` | Int | Actual days to deliver | Core metric |
| `Days for shipment (scheduled)` | Int | Promised days to deliver | Core metric |
| `order date (DateOrders)` | Timestamp | When the order was placed | Format `M/D/YYYY H:MM` |
| `shipping date (DateOrders)` | Timestamp | When the order shipped | Used for actual delay = `shipping date − order date` vs. scheduled days |

### Order & delivery outcome
| Column | Type | Definition | Analysis Notes |
|---|---|---|---|
| `Delivery Status` | String | **Advance shipping / Late delivery / Shipping canceled / Shipping on time** | Descriptive of outcome; **not** the same as `Late_delivery_risk` |
| `Late_delivery_risk` | Int (0/1) | Pre-computed risk flag: 1 = late, 0 = not late | Cross-check against our own derived late definition |
| `Order Status` | String | One of: COMPLETE, PENDING, CLOSED, PENDING_PAYMENT, CANCELED, PROCESSING, SUSPECTED_FRAUD, ON_HOLD, PAYMENT_REVIEW | Filter to `COMPLETE` for outcome analysis |
| `Shipping Mode` | String | Standard Class, First Class, Second Class, Same Day | Primary shipping-lane dimension |

### Geography
| Column | Type | Definition | Analysis Notes |
|---|---|---|---|
| `Market` | String | Africa, Europe, LATAM, Pacific Asia, USCA | Top-level region |
| `Order Region` | String | Finer-grained region (e.g., Southeast Asia, Western Europe) | Second-level region |
| `Order Country` | String | Destination country | |
| `Order State` | String | Destination state/province | |
| `Order City` | String | Destination city | |
| `Order Zipcode` | String | Destination zip (often blank) | Sparse — handle as nullable |
| `Latitude` / `Longitude` | Float | **Store location**, not destination | Do not use as destination coordinates — use only for store maps if at all |
| `Customer City / State / Country / Street / Zipcode` | String | Customer's registered location (store belongs to) | Note: differs from destination — customer may ship elsewhere |

### Customer
| Column | Type | Definition | Notes |
|---|---|---|---|
| `Customer Id` / `Order Customer Id` | Int | Customer identifier | |
| `Customer Fname / Lname` | String | Name (PII) | Never aggregate on these |
| `Customer Email / Password` | String | Masked/redacted | Ignore |
| `Customer Segment` | String | Consumer, Corporate, Home Office | Useful segmentation dimension |

### Product
| Column | Type | Definition | Notes |
|---|---|---|---|
| `Product Card Id` / `Order Item Cardprod Id` | Int | Product code (RFID-generated) | Product key |
| `Product Name` | String | Product name | **Join key to clickstream** |
| `Product Price` | Float | List price | |
| `Product Status` | Int (0/1) | **1 = NOT available, 0 = available** | Counter-intuitive — read carefully |
| `Product Description` | String | Often blank | |
| `Product Image` | String | URL | |
| `Category Id` / `Category Name` | Int / String | Product category (e.g., Sporting Goods, Cleats) | |
| `Department Id` / `Department Name` | Int / String | Department (e.g., Fitness, Apparel, Golf) | |

### Financials (line-item grain)
| Column | Type | Definition | Notes |
|---|---|---|---|
| `Order Item Id` | Int | **Primary key — unique per line item** | Use this for item counts |
| `Order Item Quantity` | Int | Units in the line | |
| `Order Item Product Price` | Float | Price per unit | |
| `Sales` | Float | `Order Item Product Price × Order Item Quantity` (pre-discount) | For discount computation only |
| `Order Item Discount` | Float | Absolute discount | |
| `Order Item Discount Rate` | Float | Discount % | |
| `Order Item Total` | Float | **Post-discount revenue — use this for revenue** | |
| `Order Profit Per Order` | Float | Profit at line-item grain — safe to sum | |
| `Order Item Profit Ratio` | Float | Profit margin per line item | |

### Order-level repeated fields (DO NOT SUM)
| Column | Type | Why not to sum |
|---|---|---|
| `Benefit per order` | Float | Repeats per line item — summing double-counts. Use `Order Profit Per Order` instead |
| `Sales per customer` | Float | Repeats per line item — summing double-counts |
| `Order Id` | Int | Repeats per line item — use `COUNT(DISTINCT Order Id)` for order counts |

---

## Clickstream file — `Tokenized_Access_Logs.csv`

**Grain:** One row per **page view** (single URL hit). ~95 MB.
**Relationship to orders:** Not a direct foreign key. Join via product name (see below). Represents **browsing interest**, not purchase.

| Column | Type | Definition | Notes |
|---|---|---|---|
| `Product` | String | Product name (URL-decoded) | **Join key to orders' `Product Name`** — needs normalization: lowercase, strip, URL-decode |
| `Category` | String | Category name (matches Category Name in orders) | |
| `Department` | String | Department name (matches Department Name in orders) | Note: has trailing spaces in some rows ("fitness ") |
| `Date` | Timestamp | Page-view timestamp (`M/D/YY H:MM`) | **Two-digit year** — different format from orders |
| `Month` | String | Month abbreviation (Sep, …) | Redundant with `Date` — useful for quick grouping |
| `Hour` | Int | Hour of day (0–23) | |
| `ip` | String | Visitor IP (unhashed) | **Pseudonymous session id** — repeated IP = same visitor |
| `url` | String | Full path; ends in `/add_to_cart` when the user added to cart | Derives `is_add_to_cart` flag |

**What clickstream enables (feeds Q8):**
- **Funnel proxy per product**: page views vs. add-to-cart views → conversion proxy
- **Engagement depth per visitor** (via repeated IPs)
- **Demand side vs. outcome side**: compare browsed products to actually-late products → is lateness driven by *demand mismatch*, not logistics?

**What it cannot do:**
- Link a specific click to a specific order (no shared id; IP ≠ Customer Id). Any claim of "this browser bought that order" would be false. We use it as **product-level aggregate**, not order-level.