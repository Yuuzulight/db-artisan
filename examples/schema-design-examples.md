# Schema Design Examples

Real-world examples of the data-schema-design skill in action.

## Example 1: E-Commerce Platform

### Scenario
Building an e-commerce platform with 500K users, 5M products, 2M orders/month. Need schemas for users, products, orders, and order items.

### Generated Schema (Using Skill)

```sql
-- Database: PostgreSQL
-- Purpose: E-commerce order management
-- Scale: 500K users, 5M products, 2M orders/month (24M orders/year)
-- Retention: Keep indefinitely (compliance), archive after 3 years

CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,
  username VARCHAR(100) NOT NULL UNIQUE,
  email_domain VARCHAR(255) NOT NULL,  -- Denormalized for "all @company.com" queries
  full_name VARCHAR(255) NOT NULL,
  country_code CHAR(2) NOT NULL,
  status VARCHAR(50) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'suspended')),
  is_verified BOOLEAN NOT NULL DEFAULT FALSE,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_email_domain ON users(email_domain);
CREATE INDEX idx_users_status_created ON users(status, created_at DESC);
CREATE INDEX idx_users_country ON users(country_code);

-- ---

CREATE TABLE products (
  id BIGSERIAL PRIMARY KEY,
  external_id VARCHAR(255) NOT NULL UNIQUE,  -- SKU or product code
  name VARCHAR(500) NOT NULL,
  category VARCHAR(100) NOT NULL,
  subcategory VARCHAR(100),
  price DECIMAL(12, 2) NOT NULL CHECK (price > 0),
  cost DECIMAL(12, 2) CHECK (cost > 0),
  stock_quantity INT NOT NULL DEFAULT 0 CHECK (stock_quantity >= 0),
  sku VARCHAR(100) UNIQUE,
  status VARCHAR(50) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'discontinued')),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_products_external_id ON products(external_id);
CREATE INDEX idx_products_category ON products(category, subcategory);
CREATE INDEX idx_products_sku ON products(sku);
CREATE INDEX idx_products_status ON products(status);

-- ---

CREATE TABLE orders (
  id BIGSERIAL PRIMARY KEY,
  order_number VARCHAR(50) NOT NULL UNIQUE,  -- Human-readable order ID
  user_id BIGINT NOT NULL REFERENCES users(id),
  total_amount DECIMAL(12, 2) NOT NULL CHECK (total_amount >= 0),
  tax_amount DECIMAL(12, 2) NOT NULL DEFAULT 0,
  shipping_amount DECIMAL(12, 2) NOT NULL DEFAULT 0,
  discount_amount DECIMAL(12, 2) NOT NULL DEFAULT 0,
  status VARCHAR(50) NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled', 'refunded')),
  payment_status VARCHAR(50) NOT NULL DEFAULT 'pending' CHECK (payment_status IN ('pending', 'paid', 'failed', 'refunded')),
  shipping_country CHAR(2),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id, created_at)  -- Composite for partitioning by date
);

-- Indexes for common queries
CREATE INDEX idx_orders_user_id ON orders(user_id, created_at DESC);
CREATE INDEX idx_orders_status ON orders(status, created_at DESC);
CREATE INDEX idx_orders_payment_status ON orders(payment_status, created_at DESC);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);  -- For time-range queries

-- ---

CREATE TABLE order_items (
  id BIGSERIAL PRIMARY KEY,
  order_id BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id BIGINT NOT NULL REFERENCES products(id),
  quantity INT NOT NULL CHECK (quantity > 0),
  unit_price DECIMAL(12, 2) NOT NULL CHECK (unit_price > 0),
  discount_amount DECIMAL(12, 2) NOT NULL DEFAULT 0,
  line_total DECIMAL(12, 2) NOT NULL CHECK (line_total >= 0),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);
```

### Key Design Decisions

✅ **BIGINT for IDs**: Expecting millions of orders
✅ **order_number UNIQUE**: Customers need human-readable order IDs to look up
✅ **Denormalized shipping_country**: Used for regional analytics; cheaper than joining
✅ **Status enums with CHECK**: Enforce valid states at database layer
✅ **Composite PK (order_id, created_at)**: Enables partitioning by date for fast archiving
✅ **Indexes on status + created_at DESC**: Queries like "show me pending orders from last 7 days"
✅ **Foreign keys with CASCADE**: When order is deleted, clean up items automatically
✅ **Audit columns**: created_at/updated_at for tracking and debugging

### Queries This Design Optimizes For

```sql
-- Lookup user by email (indexed)
SELECT * FROM users WHERE email = 'user@example.com';

-- Recent orders for a user (composite index on user_id, created_at DESC)
SELECT * FROM orders WHERE user_id = 123 ORDER BY created_at DESC LIMIT 10;

-- Orders awaiting shipment (index on status + created_at)
SELECT * FROM orders WHERE status = 'confirmed' ORDER BY created_at ASC LIMIT 100;

-- Revenue by country (denormalized country in orders table)
SELECT shipping_country, SUM(total_amount) FROM orders GROUP BY shipping_country;

-- Product availability (join on FK - indexed)
SELECT o.order_number, p.name, oi.quantity FROM orders o
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE o.status = 'pending';
```

---

## Example 2: Analytics Event Stream

### Scenario
Collecting 10M events/day from mobile & web apps. Need to track user behavior, system events, and errors. Heavy append, moderate query.

### Generated Schema

```sql
-- Database: Snowflake (or BigQuery with clustering)
-- Purpose: Event stream for analytics
-- Scale: 10M events/day (365B events/year)
-- Retention: Keep 2 years; archive to cold storage after

CREATE TABLE events (
  event_id VARCHAR(36) NOT NULL,  -- UUID from client
  user_id BIGINT,
  anonymous_id VARCHAR(36),  -- For unlogged users
  session_id VARCHAR(36) NOT NULL,  -- To group events in a session
  event_type VARCHAR(100) NOT NULL,  -- 'page_view', 'click', 'error', etc.
  event_category VARCHAR(100),  -- 'engagement', 'transaction', 'system'
  -- Denormalized event source for quick filtering
  source VARCHAR(50) NOT NULL CHECK (source IN ('web', 'mobile_ios', 'mobile_android', 'backend')),
  source_version VARCHAR(50),
  -- Key attributes
  properties VARIANT,  -- JSON in Snowflake, JSONB in Postgres
  -- Experiment tracking
  experiment_id VARCHAR(100),
  experiment_variant VARCHAR(50),
  -- Geographic + device info (denormalized for common queries)
  country_code CHAR(2),
  device_type VARCHAR(50),  -- 'mobile', 'desktop', 'tablet'
  -- Audit
  created_at TIMESTAMP_NTZ NOT NULL DEFAULT CURRENT_TIMESTAMP(),
  ingested_at TIMESTAMP_NTZ NOT NULL DEFAULT CURRENT_TIMESTAMP()
);

-- Snowflake: Cluster by event_type + user_id for fast filtering
ALTER TABLE events CLUSTER BY (event_type, user_id, created_at);

-- BigQuery equivalent: Partitioning + clustering
-- CREATE TABLE events (
--   PARTITION BY DATE(created_at)
--   CLUSTER BY event_type, user_id
-- )
```

### Key Decisions

✅ **VARIANT/JSONB for properties**: Flexible schema for different event types
✅ **Denormalized source + device_type**: Common filters without JSON parsing
✅ **session_id**: Groups related events for funnel analysis
✅ **experiment tracking**: Built in for A/B test analysis
✅ **Cluster keys**: (event_type, user_id, created_at) for typical queries
✅ **High-volume design**: No FK constraints (too expensive at scale)
✅ **Partition by date**: Archive old data cheaply

### Optimized Queries

```sql
-- Sessions with specific event flow (cluster helps with session_id + event_type)
SELECT session_id, COUNT(*) as event_count
FROM events
WHERE created_at >= CURRENT_DATE - 7
  AND event_type IN ('page_view', 'click')
  AND country_code = 'US'
GROUP BY session_id;

-- Funnel: page_view → add_to_cart → checkout (user_id cluster + event_type)
SELECT 
  COUNT(DISTINCT CASE WHEN event_type = 'page_view' THEN user_id END) as viewers,
  COUNT(DISTINCT CASE WHEN event_type = 'add_to_cart' THEN user_id END) as cart_adders,
  COUNT(DISTINCT CASE WHEN event_type = 'checkout' THEN user_id END) as buyers
FROM events
WHERE created_at >= CURRENT_DATE - 1;

-- Experiment analysis (experiment_variant cluster helps)
SELECT experiment_variant, COUNT(*) as events, COUNT(DISTINCT user_id) as users
FROM events
WHERE experiment_id = 'exp_123'
  AND created_at >= CURRENT_DATE - 30
GROUP BY experiment_variant;
```

---

## Example 3: User Subscription with Slowly Changing Dimensions

### Scenario
Tracking user subscriptions where plans, pricing, and features change over time. Need to analyze "what plan was X on Y date?" and "when did they upgrade/downgrade?"

### Generated Schema

```sql
-- SCD Type 2: Track full history of subscription changes
CREATE TABLE user_subscriptions (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT NOT NULL REFERENCES users(id),
  plan_id BIGINT NOT NULL REFERENCES plans(id),
  plan_name VARCHAR(100) NOT NULL,  -- Denormalized for reporting
  price_per_month DECIMAL(10, 2) NOT NULL CHECK (price_per_month > 0),
  billing_cycle_days INT NOT NULL CHECK (billing_cycle_days > 0),
  features JSONB,  -- Denormalized list of features at time of subscription
  -- Validity tracking
  valid_from TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  valid_to TIMESTAMP,  -- NULL = currently active
  is_current BOOLEAN NOT NULL DEFAULT TRUE,  -- Quick filter for "active" subscriptions
  -- Status tracking
  status VARCHAR(50) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'paused', 'cancelled')),
  cancellation_reason VARCHAR(500),
  -- Audit
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  changed_by_user_id BIGINT REFERENCES users(id)
);

-- Indexes for common queries
CREATE INDEX idx_sub_user_id_current ON user_subscriptions(user_id, is_current);
CREATE INDEX idx_sub_user_id_valid ON user_subscriptions(user_id, valid_from, valid_to);
CREATE INDEX idx_sub_plan_id ON user_subscriptions(plan_id, valid_from);
CREATE INDEX idx_sub_status ON user_subscriptions(status, valid_to);
```

### Usage Pattern

```sql
-- What plan is user 123 currently on?
SELECT * FROM user_subscriptions
WHERE user_id = 123 AND is_current = TRUE;

-- What was user 456 on 2024-06-15?
SELECT * FROM user_subscriptions
WHERE user_id = 456
  AND valid_from <= '2024-06-15'::date
  AND (valid_to IS NULL OR valid_to > '2024-06-15'::date);

-- Upgrade/downgrade analysis (from X plan to Y plan)
SELECT 
  s1.plan_name as from_plan,
  s2.plan_name as to_plan,
  COUNT(*) as changes
FROM user_subscriptions s1
JOIN user_subscriptions s2 ON s1.user_id = s2.user_id
WHERE s1.valid_to = s2.valid_from  -- Back-to-back valid periods = upgrade/downgrade
GROUP BY s1.plan_name, s2.plan_name;

-- Churn analysis: when did users cancel?
SELECT cancellation_reason, COUNT(*) as count
FROM user_subscriptions
WHERE status = 'cancelled'
  AND valid_to >= CURRENT_DATE - INTERVAL '90 days'
GROUP BY cancellation_reason;
```

---

## Takeaways

✅ **Design depends on access patterns** – Indexes, denormalization, and schema shape depend on "how will this be queried?"
✅ **Think in scale** – What's acceptable for 1M rows breaks at 1B rows
✅ **Constraints are your friend** – NOT NULL, CHECK, UNIQUE, and FK constraints prevent bad data
✅ **Audit columns are mandatory** – Always include created_at/updated_at and know who changed what
✅ **Database choice matters** – PostgreSQL, Snowflake, BigQuery, and DuckDB each have different trade-offs
✅ **Document your assumptions** – Comment the schema with retention, scale, and query patterns

---

**Questions?** Use these examples in Claude Code with the `data-schema-design` skill installed.
