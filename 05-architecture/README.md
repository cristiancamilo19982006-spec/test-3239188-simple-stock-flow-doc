# 05 · System Architecture — Simple Stock Flow

## 1. Architectural Style
The system follows a **Hexagonal Architecture (Ports and Adapters)** combined with **Domain-Driven Design (DDD)** principles [data-model §0, §2]:
- **Domain Core:** Encapsulates domain entities, aggregates, and value objects along with pure business invariants, decoupled from infrastructure libraries or relational engines [data-model §0, §2].
- **Application Layer:** Orchestrates use cases through inbound ports and handles side effects via outbound ports [data-model §1, §6.1].
- **Outbound Adapters (Secondary):**
  - **Persistence:** Implemented with Entity Framework Core targeting PostgreSQL 16 (`simple_stock_flow` database, `sales` schema) [data-model preamble, §0, §3.1].
  - **Binary Storage:** External file storage adapter; the domain and relational database only track an opaque string identifier (`image_key`), never direct file streams or physical disk paths [data-model §1, §3, D-08].
  - **Cryptography / Security:** Password hashing port for secure credential creation and verification [data-model §1, §2.5, D-09].

## 2. Aggregate Boundaries and Aggregate Roots
Based on the physical and relational schema consisting of five tables, three transactional aggregates and one static reference entity are identified [data-model §2]:

1. **`Product` Aggregate (Catalog):**
   - **Aggregate Root:** `Product` [data-model §2.2].
   - **Internal Entities:** None [data-model §2].
   - **Value Objects:** `Money` (strictly positive monetary value with symmetric rounding to 2 decimal places `MidpointRounding.AwayFromZero`, strictly single-currency without currency codes) [data-model §1, §2.2, D-05].
   - **Boundaries & Invariants:** Controls stock levels (`stock >= 0`). Concurrency is handled via optimistic concurrency control using PostgreSQL's system column `xmin` as a shadow property [data-model §3, D-04, ADR-002]. Deletion is strictly soft-delete via the shadow property `deleted_at` [data-model §2.2, §3, ADR-003].

2. **`Sale` Aggregate (Sales):**
   - **Aggregate Root:** `Sale` [data-model §2.3].
   - **Internal Entities:** `SaleItem` (pure composition 1:N; lifecycle is strictly bound to `Sale`, reflected by `FK-2` cascade at relational level) [data-model §2.4, §5 FK-2].
   - **Value Objects:** `Quantity` (strictly positive integer) [data-model §1, §2.4].
   - **Boundaries & Invariants:** Immutable once confirmed [data-model §1, §2.3]. Freezes product name, unit price, and category name at transaction time to isolate historical auditability from catalog mutations [data-model §1, §2.4, §3, ADR-004].

3. **`User` Aggregate (Identity & Operations):**
   - **Aggregate Root:** `User` [data-model §2.5].
   - **Value Attributes:** `Role` (restricted to the closed set `'admin'` or `'seller'`) [data-model §1, §2.5].
   - **Boundaries:** Models internal operators. The schema does not include entities for end buyers or external customers [data-model §1, §7].

4. **Reference Entity `Category`:**
   - **Nature:** Static read-only reference entity [data-model §2.1].
   - **Boundaries:** Not an aggregate root; lacks runtime mutation ports; seeded during initial database migration with five fixed UUID identifiers [data-model §2.1, §9.1].

## 3. Ports Derived from the Data Model

### Inbound Ports (Use Cases / Commands & Queries)
- `IProductCatalogUseCase`: Product creation, price updates, stock replenishment/withdrawal, and soft deletion [data-model §2.2, §6.1 Q1-Q3].
- `IRegisterSaleUseCase`: Atomic sale registration and inventory stock decrement [data-model §2.3, §6.1 Q6].
- `ISalesReportQuery`: Aggregated sales reporting by product and frozen category across time ranges [data-model §1, §6.1 Q9, §11.1].
- `IAuthenticationUseCase`: Operator authentication via lowercase normalized username [data-model §2.5, §6.1 Q10].
- *(Assumption)* `IUserProvisioningUseCase`: Seller operator provisioning by authenticated administrators [data-model §9.2, §11 H-3].

### Outbound Ports (Infrastructure Adapters)
- `IProductRepository`: Single lookups, paginated catalog searches (Q1), and batch lookups for stock locking (Q3) [data-model §6.1].
- `ISaleRepository`: Atomic persistence of the `Sale` aggregate root and its `SaleItem` collection [data-model §2.3, §6.1 Q6-Q7].
- `IReadOnlyCategoryRepository`: Read access to seeded categories (Q4, Q5) [data-model §2.1, §6.1].
- `IUserRepository`: Lookup by exact normalized username for credential validation (Q10) [data-model §2.5, §6.1].
- `ISalesReportPort`: Read-optimized database query port calculating aggregations without hydrating domain entities [data-model §1, §6.1 Q9].
- `IImageStoragePort`: Storage and physical removal of external binary images referenced by `image_key` [data-model §1, §7.1, D-08].
- `IPasswordHasher`: One-way hashing and credential verification [data-model §1, §2.5, D-09].

## 4. Invariant Allocation: Domain vs Database Engine

| Invariant / Rule | Domain Layer | Database Engine | Data Model Citation |
|---|---|---|---|
| Non-negative stock | `Product.Withdraw` / `Restock` checks availability | `ck_product_stock_non_negative` check constraint | data-model §2.2, §4 |
| Withdrawal exceeds stock | `Product.Withdraw` throws domain exception | Cannot be expressed in static DDL | data-model §2.2 |
| Strictly positive price (`price > 0`) | `Money` VO / `Product.ChangePrice` | Pending engine `CHECK` constraint (T-20) | data-model §2.2, §4 |
| Sale immutability | `Sale` root exposes no mutation methods | Absence of `UPDATE`/`DELETE` API endpoints | data-model §1, §2.3, §7.1 |
| Sale-Item composition lifecycle | `Sale.AddItem` manages collection | `FK_sale_item_sale_sale_id ON DELETE CASCADE` | data-model §2.4, §5 FK-2 |
| Prevent deleting products with sales | Domain enforces soft deletion (`deleted_at`) | `FK_sale_item_product_product_id ON DELETE RESTRICT` | data-model §4, §5 FK-3 |
| Single item occurrence per sale | `Sale.AddItem` rejects duplicate product ID | Unique index `(sale_id, product_id)` on `sale_item` | data-model §2.3, §4 |
| Unique username | `User.NormalizeUsername` forces lowercase | Unique index `IX_user_username` | data-model §2.5, §4 |
| Allowed roles (`admin`, `seller`) | Domain validation via `Roles.IsValid` | Pending engine `CHECK` constraint (T-20) | data-model §2.5, §4 |
| Frozen historical transaction data | `Sale.AddItem` snapshots live attributes | `product_name` and `category_name` columns on `sale_item` | data-model §1, §2.4, §3 |
---

## 5. Reverse SDD Traceability & Cross-Validation Matrix

This closing matrix validates the architectural decisions against the artifacts derived throughout the reverse engineering workflow:

| Documentation Layer | Derived Artifact | Architectural Alignment & Enforcement | Data Model Reference |
|---|---|---|---|
| **01 · Context** | System Scope & Actors | Boundaries isolate core application logic from external blob stores (`image_key`) and restrict usage to internal operators (`admin`, `seller`). | data-model §1, §2.5, §7.1 |
| **02 · Domain** | Invariants & Aggregates | Transactional boundaries defined around `Product`, `Sale`, and `User`. Soft-delete (`deleted_at`) and historical snapshot integrity enforced. | data-model §2, §3, §4 |
| **03 · Product** | Problem Statement & Vision | Back-office counter sale focus eliminates shopping cart complexity and payment gateway overhead. Concurrency protected via `xmin`. | data-model §1, §3, §9.1 |
| **04 · Requirements** | Functional Stories & NFRs | Outbound query ports map directly to index-optimized reporting patterns (Q9) and optimistic concurrency safeguards (NFR-01). | data-model §6.1, §6.2 |