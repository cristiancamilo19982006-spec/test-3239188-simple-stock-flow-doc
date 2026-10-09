# 01 · System Context — Simple Stock Flow

## 1. System Overview
*Simple Stock Flow* is an internal transactional back-office management system designed for physical counter sales and immediate inventory synchronization [data-model §1, §6.1]. The architecture enforces absolute transactional consistency, optimistic concurrency control, and accounting immutability without external customer-facing overhead [data-model §1, §2.4, §3, D-04, ADR-002, ADR-004].

## 2. System Scope

### In-Scope (Implemented Core Capabilities)
- Product catalog management with fixed monetary precision (`numeric(18,2)`), inventory stock tracking, seeded category assignment, and optional external binary image keys [data-model §1, §2.2, §3].
- Soft-delete semantics (`deleted_at`) preserving referential integrity against completed historical sales (`FK-3`) [data-model §2.2, §3, §5, ADR-003].
- Immediate, atomic counter sales recording with concurrent stock decrements and overselling prevention via database check constraints (`ck_product_stock_non_negative`) [data-model §2.2, §2.3, §4, D-04].
- Snapshot freezing of product name, unit price, and category name directly on the sale item line at transaction time [data-model §1, §2.4, §3, ADR-004].
- Database-level sales reporting aggregated by product and frozen categories over custom date ranges (Query Pattern Q9) [data-model §1, §6.1, §11.1].
- Internal role-based authentication and authorization restricted strictly to `'admin'` and `'seller'` [data-model §1, §2.5].

### Out-of-Scope (Explicitly Excluded by Design)
- Shopping carts, draft orders, or delayed checkout sessions *(assumption)*.
- Customer management, buyer accounts, loyalty programs, or CRM integrations [data-model §1, §7].
- External payment gateway integrations, credit processing, or transaction fees [data-model §1, §3].
- Multi-currency transactions and exchange rate conversions (strictly single-currency by design D-05) [data-model §1, §3, D-05].
- Generalized audit tracking columns (`created_at`, `updated_at`) across generic entities [data-model §1, §3].
- Runtime category CRUD operations (categories remain a static reference dataset seeded during migration) [data-model §2.1, §9.1].

## 3. External Actors and Boundary Interfaces
- **Administrator:** Manages product lifecycle, adjusts stock, and generates aggregate financial reports [data-model §1, §2.2, §6.1].
- **Seller:** Queries available inventory and records immediate atomic sales at the point of sale [data-model §1, §2.5, §6.1].
- **Relational Storage (PostgreSQL 16):** Hosts the isolated `sales` schema, enforcing constraints and optimistic concurrency via `xmin` [data-model preamble, §3, ADR-002].
- **External Image Storage:** External object store referenced only via an opaque identifier (`image_key`), decoupling the database from binary blobs [data-model §1, §3, D-08].