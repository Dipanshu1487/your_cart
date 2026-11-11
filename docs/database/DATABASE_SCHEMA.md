# Your_Cart - Production Database Schema Specification

This document outlines the refined relational PostgreSQL database schema for **Your_Cart**, targeted for **PostgreSQL 15+** on Google Cloud SQL, SQLAlchemy 2.0 ORM, and Alembic migrations.

---

## 1. Global Architectural Conventions & PostgreSQL 15+ Compatibility

- **Minimum Supported Version:** **PostgreSQL 15.0+**
  - Utilizes standard `gen_random_uuid()` (built-in core function without requiring `pgcrypto` extension).
  - Utilizes `UNIQUE NULLS NOT DISTINCT` (introduced in PostgreSQL 15, SQL:2023 standard compliant).
  - Utilizes partial unique indexes for single-variant defaults (`WHERE is_default = TRUE`).
- **Primary Keys:** `UUID` generated via `gen_random_uuid()`.
- **Timestamps:** `TIMESTAMPTZ` (Timestamp with Time Zone), defaulting to `(TIMEZONE('utc', NOW()))`.
- **Monetary Values:** `NUMERIC(12, 2)` for prices and threshold values. Never use floating-point types (`REAL`, `FLOAT`, `DOUBLE PRECISION`) to avoid roundoff errors.
- **Identity & Authentication:** Firebase Authentication manages credentials, tokens, and sessions. Passwords are never stored or processed locally.
- **Strings & URLs:** `VARCHAR(n)` for constrained identifiers, codes, and enums; `TEXT` for dynamic user content, titles, descriptions, and long/canonical URLs.
- **Deletion Policies:** Foreign keys specify explicit `ON DELETE` rules (`CASCADE`, `RESTRICT`, or `SET NULL`) to prevent orphaned rows and retain auditability.

---

## 2. Table Definitions & Constraints

### 2.1 `users`
Stores user profile information federated with Firebase Authentication.

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firebase_uid VARCHAR(128) NOT NULL,
    email VARCHAR(255) NOT NULL,
    full_name VARCHAR(150),
    profile_image TEXT,
    role VARCHAR(32) NOT NULL DEFAULT 'USER',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT (TIMEZONE('utc', NOW())),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT (TIMEZONE('utc', NOW())),
    CONSTRAINT uq_users_firebase_uid UNIQUE (firebase_uid)
);

-- Case-insensitive unique email index ensuring normalized lower-case identity
CREATE UNIQUE INDEX idx_users_email_lower ON users (LOWER(email));
```

- **Security & Constraints:**
  - `firebase_uid` NOT NULL and UNIQUE: Enforces 1:1 binding with Firebase Authentication.
  - No password columns exist in the database.
  - `LOWER(email)` functional unique index: Guarantees that `user@example.com` and `USER@example.com` cannot create duplicate accounts.

---

### 2.2 `retailers`
Stores configuration and catalog metadata for supported e-commerce platforms.

```sql
CREATE TABLE retailers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    retailer_name VARCHAR(100) NOT NULL UNIQUE,
    base_url TEXT NOT NULL,
    logo_url TEXT,
    integration_type VARCHAR(50) NOT NULL DEFAULT 'SCRAPER',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT (TIMEZONE('utc', NOW())),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT (TIMEZONE('utc', NOW()))
);
```

- **Constraints:**
  - `retailer_name` UNIQUE (e.g., `'Amazon'`, `'Flipkart'`, `'Myntra'`, `'H&M'`, `'Snitch'`, `'AJIO'`).
  - `integration_type`: `'SCRAPER'`, `'OFFICIAL_API'`, `'AFFILIATE_FEED'`, etc.

---

### 2.3 `products`
Canonical representation of a product listed on a retailer platform. Multiple users save and track this single record.

```sql
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    retailer_id UUID NOT NULL REFERENCES retailers(id) ON DELETE RESTRICT,
    canonical_url TEXT NOT NULL,
    original_url TEXT NOT NULL,
    external_product_id VARCHAR(100),
    product_title TEXT NOT NULL,
    description TEXT,
    image_url TEXT,
    brand VARCHAR(100),
    currency CHAR(3) NOT NULL DEFAULT 'INR',
    current_price NUMERIC(12, 2) NOT NULL CHECK (current_price >= 0),
    availability_status VARCHAR(32) NOT NULL DEFAULT 'UNKNOWN',
    last_checked_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT (TIMEZONE('utc', NOW())),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT (TIMEZONE('utc', NOW()))
);

-- Deduplication Constraint 1: Canonical URL uniqueness per retailer
CREATE UNIQUE INDEX idx_products_retailer_canonical_url 
ON products (retailer_id, canonical_url);

-- Deduplication Constraint 2: External product ID uniqueness per retailer (when present)
CREATE UNIQUE INDEX idx_products_retailer_external_id 
ON products (retailer_id, external_product_id) 
WHERE external_product_id IS NOT NULL;
```

- **Deduplication Strategy:**
  - Primary deduplication is enforced by `(retailer_id, canonical_url)`.
  - When retailers supply an unambiguous external ID (e.g. Amazon ASIN, Flipkart PID, Myntra Style ID), `idx_products_retailer_external_id` provides secondary deduplication.
  - If a retailer does not provide external IDs (e.g. niche D2C fashion boutiques), canonical URL normalization acts as the deterministic identifier.
  - `original_url`: Stores the exact URL pasted by the user for debugging URL normalizer changes.

---

### 2.4 `product_variants`
Maintains SKU-, size-, and color-specific items belonging to a product.

```sql
CREATE TABLE product_variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    external_variant_id VARCHAR(100),
    size VARCHAR(50),
    color VARCHAR(50),
    sku VARCHAR(100),
    is_default BOOLEAN NOT NULL DEFAULT FALSE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT (TIMEZONE('utc', NOW())),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT (TIMEZONE('utc', NOW())),
    CONSTRAINT uq_product_variant_attributes UNIQUE NULLS NOT DISTINCT (product_id, size, color, sku),
    CONSTRAINT chk_default_variant_attributes CHECK (
        (is_default = FALSE) OR 
        (is_default = TRUE AND size IS NULL AND color IS NULL)
    )
);

-- Partial Unique Index: Exactly one default variant allowed per product
CREATE UNIQUE INDEX idx_single_default_variant_per_product 
ON product_variants (product_id) 
WHERE is_default = TRUE;
```

- **Default Variant Integrity:**
  - `idx_single_default_variant_per_product`: Ensures a product **can never have more than one default variant**.
  - `chk_default_variant_attributes`: Enforces that default variants cannot specify size or color attributes, eliminating any confusion with real dimensional variants.
  - Non-default variants can have standard sizes/colors (`is_default = FALSE`).
  - Cart and monitoring routines reference `product_variants.id` uniformly regardless of whether `is_default` is `TRUE` or `FALSE`.

---

### 2.5 `carts`
Represents a user's active universal shopping cart.

```sql
CREATE TABLE carts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT (TIMEZONE('utc', NOW())),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT (TIMEZONE('utc', NOW()))
);
```

- **Constraints:**
  - `user_id UNIQUE`: Strictly enforces exactly one active cart per user.
  - `ON DELETE CASCADE`: Removing a user deletes their cart and associated cart items.

---

### 2.6 `cart_items`
Links products/variants to a user's cart along with user-specific alert criteria (e.g., target price threshold).

```sql
CREATE TABLE cart_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cart_id UUID NOT NULL REFERENCES carts(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    variant_id UUID NOT NULL REFERENCES product_variants(id) ON DELETE CASCADE,
    target_price NUMERIC(12, 2) CHECK (target_price IS NULL OR target_price > 0),
    tracking_enabled BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT (TIMEZONE('utc', NOW())),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT (TIMEZONE('utc', NOW())),
    CONSTRAINT uq_cart_variant UNIQUE (cart_id, variant_id)
);
```

- **Constraints:**
  - `CONSTRAINT uq_cart_variant UNIQUE (cart_id, variant_id)`: A user cannot add the exact same variant multiple times into their cart; instead, they update their target price.
  - `target_price`: When `current_price <= target_price`, a `PRICE_DROP_ALERT` is triggered.
  - `tracking_enabled`: Allows users to pause background tracking for specific items without removing them from their cart.

---

### 2.7 `stock_history`
Append-only log of inventory availability checks over time for a given variant.

```sql
CREATE TABLE stock_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    variant_id UUID NOT NULL REFERENCES product_variants(id) ON DELETE CASCADE,
    availability_status VARCHAR(32) NOT NULL,
    available_quantity INTEGER CHECK (available_quantity IS NULL OR available_quantity >= 0),
    checked_at TIMESTAMPTZ NOT NULL DEFAULT (TIMEZONE('utc', NOW())),
    source VARCHAR(50) NOT NULL DEFAULT 'BACKGROUND_WORKER'
);
```

- **Design:** Immutable append-only record. Records are never updated in-place. Standard table (partitioning deferred to future migration).
- **Allowed Statuses:** `'IN_STOCK'`, `'OUT_OF_STOCK'`, `'UNKNOWN'`, `'UNAVAILABLE'`.

---

### 2.8 `price_history`
Append-only record of recorded prices for products and variants.

```sql
CREATE TABLE price_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES product_variants(id) ON DELETE CASCADE,
    price NUMERIC(12, 2) NOT NULL CHECK (price >= 0),
    currency CHAR(3) NOT NULL DEFAULT 'INR',
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT (TIMEZONE('utc', NOW())),
    source VARCHAR(50) NOT NULL DEFAULT 'BACKGROUND_WORKER'
);
```

- **Design:** Immutable audit ledger. If an e-commerce platform has variant-level pricing (e.g., shoe sizes having different prices), `variant_id` is populated; otherwise, it records the parent product base price. Standard table (partitioning deferred to future migration).

---

### 2.9 `monitoring_jobs`
Shared job queue and scheduler state table for tracking products and variants.

```sql
CREATE TABLE monitoring_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES product_variants(id) ON DELETE CASCADE,
    job_status VARCHAR(32) NOT NULL DEFAULT 'SCHEDULED',
    next_check_at TIMESTAMPTZ NOT NULL DEFAULT (TIMEZONE('utc', NOW())),
    last_checked_at TIMESTAMPTZ,
    retry_count INTEGER NOT NULL DEFAULT 0 CHECK (retry_count >= 0),
    created_at TIMESTAMPTZ NOT NULL DEFAULT (TIMEZONE('utc', NOW())),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT (TIMEZONE('utc', NOW())),
    CONSTRAINT uq_product_variant_job UNIQUE NULLS NOT DISTINCT (product_id, variant_id)
);
```

- **Constraints:**
  - `job_status`: `'SCHEDULED'`, `'PROCESSING'`, `'COMPLETED'`, `'FAILED'`, `'RATE_LIMITED'`.
  - `UNIQUE NULLS NOT DISTINCT (product_id, variant_id)`: PostgreSQL 15 feature guaranteeing at most **one** monitoring job entry per product/variant pair in the entire system, regardless of how many users track it.

---

### 2.10 `notifications`
User alert inbox for price changes, restocks, and system communications.

```sql
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    cart_item_id UUID REFERENCES cart_items(id) ON DELETE SET NULL,
    notification_type VARCHAR(50) NOT NULL,
    title VARCHAR(200) NOT NULL,
    message TEXT NOT NULL,
    is_read BOOLEAN NOT NULL DEFAULT FALSE,
    sent_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT (TIMEZONE('utc', NOW()))
);
```

- **Constraints:**
  - `notification_type`: `'RESTOCK_ALERT'`, `'PRICE_DROP_ALERT'`, `'VARIANT_AVAILABLE'`, `'SYSTEM_NOTIFICATION'`.
  - `ON DELETE SET NULL` on `cart_item_id`: If a user deletes an item from their cart, historical notifications received for that item are preserved in the user inbox.

---

### 2.11 `restock_predictions`
Predictive machine learning/heuristics restock estimation records.

```sql
CREATE TABLE restock_predictions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    variant_id UUID NOT NULL REFERENCES product_variants(id) ON DELETE CASCADE,
    estimated_restock_from TIMESTAMPTZ NOT NULL,
    estimated_restock_to TIMESTAMPTZ NOT NULL,
    confidence_score NUMERIC(5, 4) NOT NULL CHECK (confidence_score >= 0.0000 AND confidence_score <= 1.0000),
    prediction_method VARCHAR(50) NOT NULL,
    generated_at TIMESTAMPTZ NOT NULL DEFAULT (TIMEZONE('utc', NOW())),
    model_version VARCHAR(50) NOT NULL,
    CONSTRAINT chk_restock_window CHECK (estimated_restock_to >= estimated_restock_from)
);
```

- **Constraints:**
  - `confidence_score`: Bounded between `0.0000` (0%) and `1.0000` (100%).
  - `chk_restock_window`: Verifies window consistency (`estimated_restock_to >= estimated_restock_from`).
