# 02 · Domain — Simple Stock Flow

## 1. Domain Entities and Aggregates

### `Product` Aggregate (Catalog)
- **Aggregate Root:** `Product`.
- **Identity:** UUID primary key `id`.
- **Attributes:**
  - `name`: Mandatory non-empty string.
  - `price`: `Money` value object with fixed 2-decimal precision (`numeric(18,2)`), non-negative, symmetric rounding [data-model §1, §2.2, §3].
  - `stock`: Non-negative integer representing current on-hand units [data-model §2.2, §3].
  - `category_id`: Mandatory reference to a valid category (`FK-1`) [data-model §2.1, §2.2, §5].
  - `image_key`: Optional opaque string identifier pointing to external binary storage; nullable [data-model §1, §2.2, §3, D-08].
- **Lifecycle:** Soft-deleted via `deleted_at` timestamp; physical deletion is prohibited to protect historical relational integrity (`FK-3`) [data-model §2.2, §3, §5, ADR-003].

### `Sale` Aggregate (Sales)
- **Aggregate Root:** `Sale`.
- **Identity:** UUID primary key `id`.
- **Attributes:**
  - `sold_at`: Mandatory UTC timestamp marking transaction completion [data-model §3, §8].
  - `sold_by`: Operator identifier of the authenticated user who recorded the transaction [data-model §3, §7].
- **Composition:**
  - `SaleItem`: Subordinate entity (1:N composition). Has no independent lifecycle outside `Sale` (`FK-2` cascade at relational level) [data-model §2.3, §2.4, §5].
  - Contains frozen snapshot attributes: `product_id`, `product_name`, `quantity`, `unit_price`, and `category_name` [data-model §1, §2.4, §3, ADR-004].

### `User` Aggregate (Identity & Operations)
- **Aggregate Root:** `User`.
- **Identity:** UUID primary key `id`.
- **Attributes:**
  - `username`: Normalized lowercase login handle (`IX_user_username`) [data-model §2.5, §4].
  - `password_hash`: Cryptographically hashed secret [data-model §1, §2.5, §3, D-09].
  - `role`: Functional access role restricted to `'admin'` or `'seller'` [data-model §1, §2.5].

### Reference Entity `Category`
- **Nature:** Read-only reference entity pre-seeded with five fixed rows [data-model §2.1, §9.1]. Operates without runtime mutation in the domain.

---

## 2. Domain Rules and Invariants

- **R-01 (Non-Negative Stock):** Inventory can never drop below zero (`stock >= 0`), enforced in persistence via `ck_product_stock_non_negative` check constraint [data-model §2.2, §4].
- **R-02 (Strictly Positive Price):** Catalog product price must be strictly greater than zero (`price > 0`) [data-model §2.2, §4].
- **R-03 (Unique Item per Sale):** A product ID cannot appear in more than one line within the same sale transaction (`IX_sale_item_sale_id_product_id`) [data-model §2.3, §4].
- **R-04 (Mandatory Sale Items):** A sale transaction cannot be confirmed empty; it must contain at least one `SaleItem` [data-model §2.3].
- **R-05 (Historical Snapshot Freezing):** When a product is added to a sale, current `name`, `price`, and `category_name` are frozen into the `SaleItem`. Future catalog modifications never affect past transactions [data-model §1, §2.4, §3, ADR-004].
- **R-06 (Sale Immutability):** Confirmed sales are permanent immutable records; no editing or deletion operations exist [data-model §1, §2.3, §7.1].
- **R-07 (Catalog Soft Deletion):** Products are never deleted physically; deactivation is represented exclusively by setting `deleted_at` [data-model §2.2, §3, §5, ADR-003].

---

## 3. Domain Events

- **`SaleRegistered`:** Emitted when a sale is finalized, triggering atomic inventory decrement across affected products [data-model §2.3, §6.1 Q6].
- **`ProductStockAdjusted`:** Emitted when stock is restocked or adjusted [data-model §2.2].
- **`ProductDeactivated`:** Emitted when a product is marked as soft-deleted via `deleted_at` [data-model §2.2, ADR-003].

---

## 4. Ubiquitous Language Glossary

- **Product:** Salable entity containing name, price, stock, category reference, and optional external image key [data-model §2.2].
- **Sale:** Finalized, immutable commercial transaction recorded by an operator in UTC [data-model §2.3, §8].
- **Sale Item:** Line-item component within a sale capturing quantity, unit price, and frozen product/category descriptors [data-model §2.4].
- **Operator:** Authenticated internal user (`admin` or `seller`); the system tracks no external customer profiles [data-model §1, §7].