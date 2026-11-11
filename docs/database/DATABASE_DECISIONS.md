# Your_Cart - Architectural Design Decisions & Trade-Off Analysis

This document records the architectural decisions, trade-offs, and rationale behind the **Your_Cart** PostgreSQL database design.

---

## 1. Decision 1: Shared Monitoring Architecture vs. Per-User Polling

### Context
When 1,000 users add the same popular product (e.g., an iPhone or Nike sneakers on Amazon/Myntra) to their carts, polling the retailer 1,000 times will result in IP bans, severe rate-limiting (HTTP 429), and wasted worker resources.

### Decision
Decouple user cart selections from the actual background polling engine:
- Canonical records for `products` and `product_variants` are stored once.
- `monitoring_jobs` contains at most **one** active record per product/variant (`UNIQUE NULLS NOT DISTINCT (product_id, variant_id)`).
- A single worker process polls the retailer, writes one entry to `price_history` and `stock_history`, and updates `products.current_price`.
- A lightweight notification evaluation query (`idx_cart_items_tracking_eval`) checks all active `cart_items` referencing that product and dispatches notifications only to users whose `target_price` condition is met.

### Trade-offs
- **Pros:** Scalable from 1 to 100,000+ users without increasing requests to retailer endpoints; respects rate limits.
- **Cons:** Requires a fan-out notification dispatcher phase after each monitoring check.

---

## 2. Decision 2: Products Without Options & Default Variant Pattern

### Context
Certain product categories (electronics, books, homeware) have no size, color, or style options, whereas apparel and footwear have intricate variants. The application must treat both symmetrically without creating nullable foreign key ambiguity throughout `cart_items`, `stock_history`, and `monitoring_jobs`.

### Decision
Introduce `is_default BOOLEAN NOT NULL DEFAULT FALSE` in `product_variants`:
- For optionless products, exactly one default variant is generated (`is_default = TRUE`).
- A PostgreSQL partial unique index `idx_single_default_variant_per_product ON product_variants (product_id) WHERE is_default = TRUE` strictly prevents any product from possessing multiple default variants.
- A table check constraint `chk_default_variant_attributes` guarantees that default variants have `size IS NULL` and `color IS NULL`, preventing default variants from ever being confused with dimensional variants.
- For apparel/variant products, standard variants are created with `is_default = FALSE`.
- Cart and monitoring routines reference `product_variants.id` uniformly.

### Trade-offs
- **Pros:** Full domain consistency; all cart items link to a concrete variant ID; no branching logic in worker code.
- **Cons:** One extra row inserted for single-variant products upon initial discovery.

---

## 3. Decision 3: Product Deduplication & Fallback Normalization

### Context
Relying solely on `(retailer_id, external_product_id)` fails when retailers (such as D2C boutiques on Shopify or custom storefronts like Snitch) do not expose stable external product IDs in DOM/APIs. Furthermore, users paste URLs with marketing parameters (`?utm_source=...`, `&ref=...`, affiliate tags, session tokens), leading to duplicate product entries for identical items.

### Decision
Implement a multi-tier deduplication strategy:
1. **Application-Layer URL Normalizer:**
   - Standardize protocol (`https://`) and hostname (`www.example.com` -> `example.com`).
   - Strip tracking/campaign queries (`utm_*`, `ref`, `ref_`, `fbclid`, `gclid`, `tag`, `affid`, session tokens).
   - Retailer-specific canonical path resolution:
     - *Amazon:* `https://amazon.in/dp/{ASIN}` (extracts 10-character ASIN directly into `external_product_id`).
     - *Flipkart:* `https://flipkart.com/p/p/{pid}` (extracts PID into `external_product_id`).
     - *Myntra:* `https://myntra.com/{styleId}` (extracts numeric style ID).
     - *D2C / Generic Retailers (Snitch, H&M, AJIO):* Resolve to normalized path `https://snitch.co.in/products/{product-handle}`.
2. **Database Schema Constraints:**
   - `canonical_url` is stored on `products` and enforced with a unique constraint: `UNIQUE (retailer_id, canonical_url)`.
   - `external_product_id` is nullable, and when present, enforced with a partial unique index: `UNIQUE (retailer_id, external_product_id) WHERE external_product_id IS NOT NULL`.
   - `original_url` is stored for historical debugging and auditing.

### Trade-offs
- **Pros:** Resilient against arbitrary query parameters; accommodates retailers without external IDs; prevents duplicate catalog growth.
- **Cons:** Retailer URL changes require parser maintenance.

---

## 4. Decision 4: Price History Consistency & Transaction-Safe Pipeline

### Context
A race condition or system crash during monitoring must never leave `products.current_price` out of sync with `price_history`. Furthermore, redundant polling checks where the price has not changed should not generate endless duplicate historical rows.

### Decision
Define an atomic, transaction-safe execution pipeline for price updates:

```text
Worker receives observed (product_id, variant_id, new_price, currency)
  │
  ├── 1. Validate: Verify new_price >= 0 and currency matches products.currency
  │
  ├── 2. Begin Transaction (ISOLATION LEVEL READ COMMITTED)
  │       │
  │       ├── SELECT current_price, currency FROM products WHERE id = :product_id FOR UPDATE;
  │       │
  │       ├── Check Price Drift:
  │       │    IF current_price != new_price:
  │       │        INSERT INTO price_history (product_id, variant_id, price, currency, recorded_at, source)
  │       │        VALUES (:product_id, :variant_id, :new_price, :currency, NOW(), 'WORKER');
  │       │        UPDATE products 
  │       │        SET current_price = :new_price, last_checked_at = NOW(), updated_at = NOW()
  │       │        WHERE id = :product_id;
  │       │    ELSE:
  │       │        UPDATE products SET last_checked_at = NOW(), updated_at = NOW() WHERE id = :product_id;
  │       │        (Skip price_history insertion to prevent duplicate noise)
  │       │
  │       └── COMMIT TRANSACTION
  │
  └── 3. Fan-out Notification Dispatch (Triggered only if price changed downwards)
```

- **Row Locking (`FOR UPDATE`):** Eliminates write skew from overlapping workers.
- **Deduplication:** Repeated observations with identical prices update `last_checked_at` but omit redundant `price_history` inserts, preserving storage without losing check freshness.

---

## 5. Decision 5: Firebase Authentication & Identity Policy

### Context
Your_Cart uses Firebase Authentication for account security (OAuth, email/password, social logins, password resets).

### Decision
- `firebase_uid` is `VARCHAR(128) NOT NULL UNIQUE`.
- Zero password hashes or credential secrets are stored locally.
- `LOWER(email)` is indexed via a functional unique index: `CREATE UNIQUE INDEX idx_users_email_lower ON users (LOWER(email))`.
- Application login flows always sanitize and lowercase user emails before lookup/creation, preventing duplicate user accounts caused by case variations.

---

## 6. Decision 6: Partitioning Postponement & Future Migration Plan

### Context
While `price_history` and `stock_history` grow rapidly, partitioning prematurely introduces complexity into Alembic migrations, foreign key constraints, and local development setups.

### Decision
Keep `price_history` and `stock_history` as standard PostgreSQL tables for initial development and launch. Plan a phased migration to declarative table partitioning when records exceed 10 million.

### Future Partitioning Migration Strategy & Implications:
1. **Partition Key Requirement:** In PostgreSQL declarative partitioning, any `PRIMARY KEY` or `UNIQUE` constraint on a partitioned table must include the partition key:
   - Current PK: `id UUID PRIMARY KEY`.
   - Partitioned PK: `PRIMARY KEY (id, recorded_at)` (for `price_history`) and `PRIMARY KEY (id, checked_at)` (for `stock_history`).
2. **Foreign Key Inbound Restrictions:**
   - In PostgreSQL, another table cannot declare a foreign key referencing a partitioned table unless the referenced key includes the partition key. Because `price_history` and `stock_history` are leaf audit tables and have **no inbound foreign keys from other tables**, partitioning them later will not break application relational integrity.
3. **Partition Scheme:** Monthly range partitions:
   ```sql
   CREATE TABLE price_history_y2026m11 PARTITION OF price_history
   FOR VALUES FROM ('2026-11-01 00:00:00+00') TO ('2026-12-01 00:00:00+00');
   ```
4. **Archival & Pruning:** Partitions older than 180 days can be detached (`ALTER TABLE ... DETACH PARTITION`) and exported to Google Cloud Storage (BigQuery / Parquet) with zero table locks on active partitions.
