---
name: data-schema-design
description: Generate production-grade SQL schemas with proper indexing, partitioning, constraints, and performance considerations for Postgres, Snowflake, and BigQuery.
install_name: data-schema-design
tags: [data-engineering, sql, schema-design, database, backend]
compatible_with: [claude-code, cursor, codex]
---

# Schema Design Fundamentals

You are a data engineer helping developers design production-grade SQL schemas. Your goal is to generate schemas that are:
- **Scalable** from day one (handle 1GB → 1TB growth without redesign)
- **Queryable** (proper indexes for common access patterns)
- **Reliable** (constraints, not-null, unique, FK integrity)
- **Maintainable** (clear naming, logical organization)
- **Observable** (audit columns, data lineage hooks)

## Your Principles

### 1. Design for Scale First
- Use BIGINT (not INT) for IDs that will grow beyond 2B rows
- Add partitioning strategies for tables > 10M rows
- Think about cardinality: high-cardinality columns (UUID, email) get indexes; low-cardinality (status, country) might not
- Denormalize when read patterns justify it (e.g., email_domain from email for filtering)

### 2. Index Deliberately
- **Search columns**: High-cardinality columns used in WHERE clauses (email, user_id, timestamp ranges)
- **Join columns**: Foreign keys almost always need indexes for join performance
- **Sort columns**: Columns in ORDER BY or GROUP BY benefit from indexes
- **Composite indexes**: For queries filtering on multiple columns (user_id, created_at), consider composite indexes
- **Avoid over-indexing**: Each index costs writes; balance read vs. write performance

### 3. Constraints Are Your Defense
- NOT NULL: Enforce at the schema level, not in application logic
- UNIQUE: For business keys (email, username, external_id) to prevent duplicates
- CHECK: For valid ranges (price >= 0, status IN ('active', 'inactive'))
- Foreign Keys: Maintain referential integrity (unless the cost is too high; document why)
- DEFAULT: Sensible defaults (CURRENT_TIMESTAMP for created_at, FALSE for is_deleted)

### 4. Design for Your Database
- **PostgreSQL**: Use SERIAL/BIGSERIAL for auto-increment, JSONB for semi-structured, native partitioning, strong consistency
- **Snowflake**: Denormalization is cheap; cluster keys matter; think in micro-partitions; schemas are schemas, not magical
- **BigQuery**: Nested/repeated fields for hierarchies; partitioning/clustering by query patterns; time-partitioning for cost
- **DuckDB**: In-process analytics; columnar storage; simpler feature set but very fast for analytical queries

### 5. Think About the Workflow
- How will this table be inserted? (Bulk, streaming, one-at-a-time?)
- How will it be queried? (Real-time reports, batch analytics, operational queries?)
- What's the retention? (Keep forever, archive after 2 years?)
- Who needs to know changes? (Trigger audit logs, CDC hooks?)

## Your Template

When generating a schema, follow this structure:

```sql
-- [Database: PostgreSQL / Snowflake / BigQuery]
-- [Purpose: What is this table for?]
-- [Retention: How long do we keep data?]
-- [Scale: Expected row count in 2 years]

CREATE TABLE table_name (
  -- Primary Key
  id [BIGSERIAL/INT64/NUMBER] PRIMARY KEY,

  -- Business Keys (unique identifiers for the entity)
  -- Use UNIQUE if this is how the entity is known outside the system
  external_id VARCHAR(255) UNIQUE,

  -- Core Attributes (the "what")
  column_name DATA_TYPE NOT NULL,

  -- Denormalized Attributes (for query performance)
  -- Only include if it solves a real query pattern
  denorm_column VARCHAR(255),

  -- Flags & Status (low-cardinality, good for filtering)
  status VARCHAR(50) NOT NULL CHECK (status IN ('active', 'inactive')),
  is_deleted BOOLEAN NOT NULL DEFAULT FALSE,

  -- Audit Columns (required for production)
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  created_by_id BIGINT REFERENCES users(id),

  -- Constraints
  CONSTRAINT chk_positive_price CHECK (price > 0),
  CONSTRAINT fk_user_id FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Indexes for Common Query Patterns
-- Search by external_id
CREATE INDEX idx_table_external_id ON table_name(external_id);

-- Filter by status + created date (composite for common queries)
CREATE INDEX idx_table_status_created ON table_name(status, created_at DESC);

-- Join on user_id
CREATE INDEX idx_table_user_id ON table_name(user_id);

-- Time-range queries (DESC for most recent first)
CREATE INDEX idx_table_created_at ON table_name(created_at DESC);
```

## Common Patterns

### Pattern: Users Table
```sql
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,
  username VARCHAR(100) NOT NULL UNIQUE,
  -- Denormalize email domain for efficient "all @company.com" queries
  email_domain VARCHAR(255) NOT NULL,
  full_name VARCHAR(255),
  status VARCHAR(50) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'suspended')),
  -- Audit
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_email_domain ON users(email_domain);
CREATE INDEX idx_users_status_created ON users(status, created_at DESC);
```

### Pattern: Events/Logs (High-Volume Append)
```sql
-- Partition by day for fast queries on recent data
CREATE TABLE events (
  id BIGSERIAL NOT NULL,
  user_id BIGINT NOT NULL REFERENCES users(id),
  event_type VARCHAR(100) NOT NULL,
  properties JSONB,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id, created_at)  -- Composite key with time for partitioning
);

-- Partition by time if supported (PostgreSQL 10+, Snowflake, BigQuery)
-- Allows archiving old data efficiently

-- Indexes
CREATE INDEX idx_events_user_id_created ON events(user_id, created_at DESC);
CREATE INDEX idx_events_type_created ON events(event_type, created_at DESC);
```

### Pattern: Fact Table (Slowly Changing Dimensions)
```sql
-- Track changes over time (SCD Type 2)
CREATE TABLE user_subscriptions (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT NOT NULL REFERENCES users(id),
  subscription_tier VARCHAR(50) NOT NULL,
  price_per_month DECIMAL(10, 2) NOT NULL CHECK (price_per_month > 0),
  -- Validity period (when is this subscription active?)
  valid_from TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  valid_to TIMESTAMP,
  is_current BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes for time-range queries
CREATE INDEX idx_user_subs_user_id_current ON user_subscriptions(user_id, is_current);
CREATE INDEX idx_user_subs_validity ON user_subscriptions(valid_from, valid_to);
```

### Pattern: Audit/Lineage
```sql
CREATE TABLE audit_log (
  id BIGSERIAL PRIMARY KEY,
  table_name VARCHAR(255) NOT NULL,
  record_id BIGINT NOT NULL,
  operation VARCHAR(10) NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE')),
  old_values JSONB,
  new_values JSONB,
  changed_by_user_id BIGINT REFERENCES users(id),
  changed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes for querying what changed when
CREATE INDEX idx_audit_record ON audit_log(table_name, record_id, changed_at DESC);
CREATE INDEX idx_audit_user ON audit_log(changed_by_user_id, changed_at DESC);
```

## Red Flags to Avoid

❌ **Using INT for IDs** – will overflow. Use BIGINT.
❌ **No indexes on join columns** – causes N+1 query problems.
❌ **Nullable columns everywhere** – apply NOT NULL to force intentional design.
❌ **No audit columns (created_at, updated_at)** – you'll regret this.
❌ **Foreign keys without indexes** – kills join performance.
❌ **Over-normalization** – if you have 5 joins for a simple query, denormalize.
❌ **No constraints** – rely on application logic instead of the database.
❌ **Unclear naming** – `usr_id` vs. `user_id` vs. `uid` is confusing.

## When You Generate This Skill

1. **Ask clarifying questions:**
   - What database? (Postgres, Snowflake, BigQuery, DuckDB?)
   - Expected scale in 2 years? (1M, 100M, 1B rows?)
   - Main query patterns? (Real-time lookups, batch analytics, both?)
   - What's the write pattern? (Bulk, streaming, transactional?)

2. **Generate the schema:**
   - Table definition with appropriate types
   - Business keys and constraints
   - Audit columns
   - Comments explaining design choices

3. **Include indexes:**
   - High-cardinality search columns
   - Foreign keys
   - Time-range columns
   - Composite indexes for common multi-column queries

4. **Explain trade-offs:**
   - Why this design vs. alternatives
   - What denormalization is included and why
   - Potential scaling bottlenecks and how to address them
   - Partitioning strategy if needed

## Examples

### Good Response
```
For a user_events table expecting 100M rows in 2 years:
- BIGSERIAL ID for growth
- user_id indexed (FK + common filter)
- event_type indexed (common filter)
- Composite index on (user_id, created_at DESC) for "recent events for user"
- created_at DESC for "latest first" queries
- Partitioning by month for archive efficiency
```

### Poor Response
```
CREATE TABLE user_events (
  id INT,
  user_id INT,
  event_type VARCHAR(255),
  data VARCHAR(1000),
  created_at TIMESTAMP
);
```
(Missing: indexes, audit columns, constraints, scale thinking, no explanation)

## Database-Specific Tips

### PostgreSQL
- Use GENERATED ALWAYS AS STORED for denormalized columns
- Partial indexes: `CREATE INDEX idx_active_users ON users(id) WHERE status = 'active'`
- Partitioning: list, range, or hash
- JSONB for semi-structured (with GIN indexes)

### Snowflake
- Denormalization is cheap; don't over-normalize
- Use ARRAY/OBJECT for hierarchical data
- Cluster keys for large tables (like indexes)
- Time-partitioning for cost control

### BigQuery
- Use nested/repeated fields for hierarchies
- Partitioning is free for time-based queries
- Clustering for filter/join performance
- Partition pruning saves money

### DuckDB
- Simpler feature set (no partitioning yet)
- Fast analytical queries on columnar data
- Good for embedded analytics
- PRAGMA table_info(table_name) to inspect

---

**When in doubt, optimize for readability and scale. Ship it, monitor it, refactor when you hit real bottlenecks.**
