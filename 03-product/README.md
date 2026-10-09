# 03 · Product — Simple Stock Flow

## 1. Problem Statement
The system addresses inventory management and immediate sales processing for retail counter environments:
- **Overselling Prevention:** Guarantees no product is sold beyond physical stock using database guards (`ck_product_stock_non_negative`) and optimistic concurrency (`xmin`) [data-model §2.2, §3, D-04, ADR-002].
- **Historical Integrity:** Prevents past sales reports from being altered when product names or prices change in the catalog by freezing those values at sale time (`sale_item`) [data-model §1, §2.4, §3, ADR-004].
- **Zero Overhead:** Eliminates unnecessary complexities such as shopping carts, guest checkout sessions, payment gateway integrations, customer profiles, and multi-currency operations, adhering strictly to a single-currency model (D-05) [data-model §1, §3, §7].

## 2. Product Vision
To provide a lightweight, transactional, zero-friction back-office application for direct counter sales *(business assumption supported by seeded categories: General, Tools, Electrical, Plumbing, Paints)* [data-model §9.1]. The core value proposition is deterministic stock consistency and reliable sales auditability [data-model §2.3, §7.1, §8].

## 3. Product Users
The system is exclusively accessed by authenticated internal operators; **it does not support customer or buyer accounts** [data-model §1, §7]:
- **Seller (`seller`):** Browses product catalog and live stock (Q1) and executes atomic sales transactions (Q6) [data-model §1, §2.5, §6.1].
- **Administrator (`admin`):** Manages the product catalog (creation, price updates, external image associations via `image_key`, soft deletes) and accesses consolidated sales reports over date ranges (Q9) [data-model §1, §2.2, §6.1, §11 H-3].
- *(Assumption)* Initial administrators are provisioned via environment variables during deployment; admin role creation is not exposed as a self-service endpoint [data-model §9.2, §11 H-3].