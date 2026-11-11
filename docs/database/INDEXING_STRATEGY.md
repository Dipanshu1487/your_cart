# Your_Cart - Production Indexing Strategy & Performance Optimization

This document defines the indexing strategy across all database entities for **Your_Cart** on **PostgreSQL 15+**, addressing high-volume scraping intervals, concurrent user cart queries, and analytical price lookups.

---

## 1. Indexing Principles & PostgreSQL 15+ Enhancements

1. **Foreign Key Coverage:** PostgreSQL does not automatically create indexes on foreign key columns. All foreign key references are indexed to prevent sequential scans during `JOIN` queries and `CASCADE` deletions.
2. **Composite Indexes for Filtering & Sorting:** Indexes are structured with equality filters first, followed by range/timestamp ordering to allow single-index scan execution.
3. **Partial Indexes for Active Workloads:** Hot working sets (such as active carts or pending monitoring jobs) utilize partial indexes (`WHERE ...`) to save RAM and maintain compact B-Trees.
4. **PostgreSQL 15 `UNIQUE NULLS NOT DISTINCT`:** Ensures deduplication for composite keys containing nullable columns (such as `product_variants(product_id, size, color, sku)` and `monitoring_jobs(product_id, variant_id)`).

---

## 2. Table-by-Table Index Definitions

### 2.1 `users`
```sql
-- Fast login lookups and profile resolution
CREATE INDEX idx_users_firebase_uid ON users (firebase_uid);

-- Case-insensitive normalized email lookup
CREATE UNIQUE INDEX idx_users_email_lower ON users (LOWER(email));

-- Active user filter
CREATE INDEX idx_users_active ON users (is_active) WHERE is_active = TRUE;
```

---

### 2.2 `retailers`
```sql
-- Retailer discovery and active adapter checks
CREATE INDEX idx_retailers_name ON retailers (retailer_name);
CREATE INDEX idx_retailers_active ON retailers (is_active) WHERE is_active = TRUE;
```

---

### 2.3 `products`
```sql
-- Foreign key index
CREATE INDEX idx_products_retailer_id ON products (retailer_id);

-- Primary deduplication: Canonical URL uniqueness per retailer
CREATE UNIQUE INDEX idx_products_retailer_canonical_url 
ON products (retailer_id, canonical_url);

-- Secondary deduplication: External product ID uniqueness per retailer (when present)
CREATE UNIQUE INDEX idx_products_retailer_external_id 
ON products (retailer_id, external_product_id) 
WHERE external_product_id IS NOT NULL;

-- Filtering products needing price/availability checks
CREATE INDEX idx_products_last_checked ON products (last_checked_at NULLS FIRST);

-- Search and status filters
CREATE INDEX idx_products_brand ON products (brand) WHERE brand IS NOT NULL;
CREATE INDEX idx_products_status ON products (availability_status);
```

---

### 2.4 `product_variants`
```sql
-- Foreign key index
CREATE INDEX idx_variants_product_id ON product_variants (product_id);

-- Enforces exactly one default variant per product
CREATE UNIQUE INDEX idx_single_default_variant_per_product 
ON product_variants (product_id) 
WHERE is_default = TRUE;

-- External variant / SKU lookups
CREATE INDEX idx_variants_external_id ON product_variants (external_variant_id) WHERE external_variant_id IS NOT NULL;
CREATE INDEX idx_variants_sku ON product_variants (sku) WHERE sku IS NOT NULL;

-- Variant resolution lookup
CREATE INDEX idx_variants_lookup ON product_variants (product_id, size, color);
```

---

### 2.5 `carts`
```sql
-- Direct user cart resolution (enforced unique)
CREATE UNIQUE INDEX idx_carts_user_id ON carts (user_id);
```

---

### 2.6 `cart_items`
```sql
-- Foreign keys
CREATE INDEX idx_cart_items_cart_id ON cart_items (cart_id);
CREATE INDEX idx_cart_items_product_id ON cart_items (product_id);
CREATE INDEX idx_cart_items_variant_id ON cart_items (variant_id);

-- Unique index preventing duplicate variant additions per cart
CREATE UNIQUE INDEX idx_cart_items_unique_variant ON cart_items (cart_id, variant_id);

-- Notification dispatcher lookup: quickly finds users tracking a product where target price is met
CREATE INDEX idx_cart_items_tracking_eval 
ON cart_items (product_id, variant_id, tracking_enabled, target_price) 
WHERE tracking_enabled = TRUE;
```

---

### 2.7 `stock_history`
```sql
-- Foreign key & historical timeline query
CREATE INDEX idx_stock_history_variant_checked 
ON stock_history (variant_id, checked_at DESC);

-- Status filter for restock event detection
CREATE INDEX idx_stock_history_status 
ON stock_history (variant_id, availability_status, checked_at DESC);
```

---

### 2.8 `price_history`
```sql
-- Analytical lookup for 7-day, 30-day, and 1-year product price charts
CREATE INDEX idx_price_history_product_recorded 
ON price_history (product_id, recorded_at DESC);

-- Variant-specific price trends
CREATE INDEX idx_price_history_variant_recorded 
ON price_history (variant_id, recorded_at DESC) 
WHERE variant_id IS NOT NULL;
```

---

### 2.9 `monitoring_jobs`
```sql
-- Foreign keys
CREATE INDEX idx_monitoring_jobs_product_id ON monitoring_jobs (product_id);
CREATE INDEX idx_monitoring_jobs_variant_id ON monitoring_jobs (variant_id) WHERE variant_id IS NOT NULL;

-- High-frequency scheduler polling index: Worker queries jobs scheduled for execution
CREATE INDEX idx_monitoring_jobs_scheduler 
ON monitoring_jobs (job_status, next_check_at ASC) 
WHERE job_status IN ('SCHEDULED', 'FAILED');
```

---

### 2.10 `notifications`
```sql
-- Foreign key
CREATE INDEX idx_notifications_cart_item_id ON notifications (cart_item_id) WHERE cart_item_id IS NOT NULL;

-- User inbox query: fetch unread notifications first, ordered newest first
CREATE INDEX idx_notifications_user_inbox 
ON notifications (user_id, is_read, created_at DESC);

-- Unread count badge query
CREATE INDEX idx_notifications_user_unread 
ON notifications (user_id) 
WHERE is_read = FALSE;
```

---

### 2.11 `restock_predictions`
```sql
-- Foreign key and latest prediction lookup
CREATE INDEX idx_predictions_variant_generated 
ON restock_predictions (variant_id, generated_at DESC);

-- Filtering high-confidence upcoming restocks
CREATE INDEX idx_predictions_upcoming 
ON restock_predictions (estimated_restock_from ASC, confidence_score DESC);
```
