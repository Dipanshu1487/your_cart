# Your_Cart - Database Relationships & Referential Integrity

This document details all relational mappings, cardinalities, constraints, and cascading rules across the 11 entities in the **Your_Cart** platform on **PostgreSQL 15+**.

---

## 1. Relational Summary Matrix

| Parent Entity | Child Entity | Cardinality | Foreign Key Column | Constraint Name | ON DELETE Action | Description |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`users`** | `carts` | 1:1 | `carts.user_id` | `fk_carts_user_id` | **CASCADE** | Every user has one master cart. Removing user removes cart. |
| **`users`** | `notifications` | 1:N | `notifications.user_id` | `fk_notifications_user_id` | **CASCADE** | User receives notifications. Deleting user deletes inbox. |
| **`retailers`** | `products` | 1:N | `products.retailer_id` | `fk_products_retailer_id` | **RESTRICT** | Retailers supply products. Cannot delete active retailer with products. |
| **`products`** | `product_variants` | 1:N | `product_variants.product_id` | `fk_variants_product_id` | **CASCADE** | A product contains multiple SKU variants or exactly one default variant. |
| **`products`** | `cart_items` | 1:N | `cart_items.product_id` | `fk_cart_items_product_id` | **CASCADE** | Users reference canonical products in their cart. |
| **`products`** | `price_history` | 1:N | `price_history.product_id` | `fk_price_history_product_id` | **CASCADE** | Tracks price adjustments over time for the base product. |
| **`products`** | `monitoring_jobs` | 1:N | `monitoring_jobs.product_id` | `fk_monitoring_product_id` | **CASCADE** | Shared background scheduler tracks product state. |
| **`carts`** | `cart_items` | 1:N | `cart_items.cart_id` | `fk_cart_items_cart_id` | **CASCADE** | Cart aggregates multiple selected items. |
| **`product_variants`** | `cart_items` | 1:N | `cart_items.variant_id` | `fk_cart_items_variant_id` | **CASCADE** | A user tracks a specific variant (or the default variant). |
| **`product_variants`** | `stock_history` | 1:N | `stock_history.variant_id` | `fk_stock_history_variant_id`| **CASCADE** | Stock levels are strictly recorded per variant. |
| **`product_variants`** | `price_history` | 0..1:N | `price_history.variant_id` | `fk_price_history_variant_id`| **CASCADE** | Optional link if retailer assigns prices at variant level. |
| **`product_variants`** | `monitoring_jobs` | 0..1:N | `monitoring_jobs.variant_id` | `fk_monitoring_variant_id` | **CASCADE** | Optional link if monitoring checks a specific variant SKU. |
| **`product_variants`** | `restock_predictions` | 1:N | `restock_predictions.variant_id`| `fk_predictions_variant_id` | **CASCADE** | Machine learning predictions apply to exact variants. |
| **`cart_items`** | `notifications` | 0..1:N | `notifications.cart_item_id` | `fk_notifications_cart_item_id` | **SET NULL** | Retains notification log even if user removes item from cart. |

---

## 2. In-Depth Relationship Analysis & Refinements

### 2.1 User to Cart (1:1)
- **Design:** `carts.user_id` has a `UNIQUE` constraint in addition to the foreign key constraint.
- **Workflow:** When a user registers (or upon first cart action), a single cart is generated.
- **Cascade Behavior:** `ON DELETE CASCADE`. If a user deletes their account, their cart and all dependent cart items are cleanly purged.

### 2.2 Cart to Cart Items (1:N) and Product Variant to Cart Items (1:N)
- **Design:** `cart_items` links `carts`, `products`, and `product_variants`.
- **Constraint:** A composite unique constraint `UNIQUE (cart_id, variant_id)` prevents accidental duplication of the same variant in the same user's cart. If the user attempts to track an already tracked item, the application updates their `target_price` rather than creating a duplicate row.
- **Default Variant Support:** When an item has no size/color options, it binds to the product's default variant (`product_variants.is_default = TRUE`). Cart operations treat default and regular variants identically without branching foreign keys.

### 2.3 Retailer to Product (1:N)
- **Design:** `products.retailer_id` references `retailers.id`.
- **Constraint:** `ON DELETE RESTRICT`. Ensures platform data integrity; an administrator cannot inadvertently delete a retailer while there are canonical products actively tracked for that merchant.

### 2.4 Canonical Product to Product Variants (1:N)
- **Design:** Every product has one or more variants (e.g. Size M, Size L, Color Black).
- **Default Variant Rule:** If a product does not have options (e.g., an electronics item or book), a single default variant is created:
  - `is_default = TRUE`
  - `size IS NULL` and `color IS NULL` (enforced via `CHECK ((is_default = FALSE) OR (is_default = TRUE AND size IS NULL AND color IS NULL))`)
  - Exactly one default variant per product is enforced via partial unique index `idx_single_default_variant_per_product`.

### 2.5 Product/Variant to Stock History (1:N) & Price History (1:N)
- **Design:** These tables represent append-only time-series data. Standard unpartitioned tables for launch.
- **Integrity:** `price_history` maintains both `product_id` and an optional `variant_id`. Some platforms price sizes/colors identically under one product URL, while others price distinct sizes differently. Supporting nullable `variant_id` with non-null `product_id` accommodates both structures.

### 2.6 Product to Monitoring Jobs (Shared Architecture 1:N)
- **Design:** The monitoring job table is **independent of user cart items**.
- **Constraint:** Composite uniqueness `UNIQUE NULLS NOT DISTINCT (product_id, variant_id)`.
- **Shared Job Concept:** When 500 users add the same Amazon product to their carts, only **one** monitoring job entry exists. The worker fetches this single job, executes the transaction-safe price update, and an alert evaluator dispatches notifications for all matching `cart_items`.

### 2.7 Cart Items to Notifications (0..1:N)
- **Design:** `notifications.cart_item_id` uses `ON DELETE SET NULL`.
- **Auditability:** If a user deletes an item from their cart after receiving an alert, the alert record inside `notifications` remains visible in the user's inbox with `cart_item_id = NULL`, preserving historical audit trails.
