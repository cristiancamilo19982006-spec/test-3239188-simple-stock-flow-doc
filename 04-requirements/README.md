# 04 · System Requirements — Simple Stock Flow

## 1. Functional User Stories

### US-01: Product Catalog Management and Browsing
- **As:** System Administrator / Internal Operator.
- **I want to:** Create, browse, and update catalog products with name, price, stock, category, and optional image.
- **So that:** The store maintains an up-to-date commercial offering ready for direct sales.
- **Acceptance Criteria:**
  - Product requires a trimmed, non-empty name, price strictly greater than zero, and a valid foreign key to an existing category (`FK-1`) [data-model §2.1, §2.2, §5].
  - Image is optional; if omitted it must be stored as SQL `NULL` (`image_key IS NULL`), never an empty string [data-model §1, §2.2, §3].
  - Deleting a product must not execute a physical delete; it must set the soft-delete timestamp (`deleted_at`) to preserve relational integrity with past sales (`FK-3`) [data-model §2.2, §3, §5, ADR-003].
  - Catalog browsing supports paginated filtering by category, search text, and active state (Query Pattern Q1) [data-model §6.1].

### US-02: Immutable Sale Registration
- **As:** Authenticated Seller (`seller`).
- **I want to:** Record an immediate counter sale transaction specifying product IDs and quantities.
- **So that:** The sale is finalized and stock is deducted atomically.
- **Acceptance Criteria:**
  - The sale records the exact UTC timestamp (`sold_at`) and the authenticated operator (`sold_by`) [data-model §3, §8].
  - The sale requires at least one item line to be confirmed [data-model §2.3].
  - Duplicate product entries within the same sale transaction are strictly rejected (`IX_sale_item_sale_id_product_id`) [data-model §2.3, §4].
  - Line registration and stock withdrawal execute together; if requested quantity exceeds available stock or leaves negative balance, the operation fails (`ck_product_stock_non_negative`) [data-model §2.2, §2.3, §4].
  - Each item line snapshots and freezes the product name, unit price, and category name at sale time [data-model §1, §2.4, §3].
  - A registered sale is strictly immutable: no updates or deletions are permitted [data-model §1, §2.3, §7.1].

### US-03: Consolidated Sales Report Generation
- **As:** Business Administrator.
- **I want to:** Generate an aggregated sales report across a specified date range.
- **So that:** I can evaluate quantities sold and total revenue grouped by product and category.
- **Acceptance Criteria:**
  - Date range requires end date greater than or equal to start date [data-model §1].
  - Aggregation is executed directly by the database engine grouping by frozen `product_id`, `product_name`, and `category_name` (Query Pattern Q9) [data-model §1, §6.1, §11.1].
  - Historical category updates result in distinct reporting rows reflecting frozen labels without mutating closed accounting periods [data-model §11.1, ADR-004].
  - Reports do not break down by seller or end-user customer details (the system tracks no buyer entities) [data-model §1, §7, DP-02].

### US-04: Internal Operator Authentication
- **As:** Internal Operator (`admin` or `seller`).
- **I want to:** Sign in using my username and password.
- **So that:** I gain authorized access to features corresponding to my role.
- **Acceptance Criteria:**
  - Username is normalized to lowercase and trimmed before querying (`IX_user_username`) [data-model §2.5, §4].
  - Passwords are never stored or verified in plain text; they are verified against `password_hash` via a dedicated cryptographic port [data-model §1, §2.5, §3, D-09].
  - Assigned role belongs strictly to `'admin'` or `'seller'` [data-model §1, §2.5].

---

## 2. Non-Functional Requirements (NFR)

- **NFR-01 (Transactional Integrity & Optimistic Concurrency):** Prevent overselling and negative stock under concurrent sales using PostgreSQL's system column `xmin` and the engine-level check constraint `ck_product_stock_non_negative` [data-model §2.2, §3, D-04, ADR-002].
- **NFR-02 (Auditability & Immutability):** All confirmed sales and items are strictly append-only and immutable; no update or physical deletion ports exist (`FK-2` cascade exists purely for compositional definition) [data-model §1, §2.3, §7.1].
- **NFR-03 (Data Privacy & Secret Handling):** `password_hash` must never appear in API responses, logs, or search indices. Operator usernames are treated as personal data with restricted access [data-model §7].
- **NFR-04 (Reporting Query Performance):** The composite index `IX_sale_item_sale_id_product_id` with `INCLUDE (quantity, unit_price)` must allow resolving sales reports (Q9) via index-only scans without scanning base tables [data-model §6.2, T-13].
- **NFR-05 (Monetary Precision):** Exact decimal arithmetic (`numeric(18,2)`) with `MidpointRounding.AwayFromZero` strategy in the `Money` value object matching database definitions [data-model §2.2, §3].
- **NFR-06 (Single-Currency Constraint):** *(Assumption based on D-05)* The platform operates under a single implicit currency; no currency codes or exchange rate tables exist [data-model §1, §3, D-05].