# Data Artisan

AI-powered skills for building production-grade data pipelines, schemas, and systems—with Claude Code, Cursor, or any agent that supports Agent Skills.

[![Agent Skills compatible](https://img.shields.io/badge/Agent%20Skills-compatible-success)](https://github.com/vercel-labs/agent-skills)
[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![GitHub Stars](https://img.shields.io/github/stars/Yuuzulight/data-engineering-skills?style=social)](https://github.com/Yuuzulight/data-engineering-skills)

## The Problem

When you ask Claude Code to generate a database schema or data pipeline, it creates **working code**, but often misses the patterns that make systems production-ready:

- Missing indexes on high-cardinality columns
- No partition strategies for scale
- Incomplete data quality checks
- No lineage or observability patterns
- Designs that work at 1GB but break at 1TB

**Data Engineering Skills** teaches AI agents how to think like data engineers: designing for scale, reliability, and operations from day one.

## What You Get

Each skill is a portable SKILL.md file that works with:
- **Claude Code** in Claude.ai
- **Cursor** (Claude Mode)
- **Codex** and other Agent Skills–compatible tools
- **Direct paste** into any coding agent

### Available Skills

| Skill | Install Name | What It Does |
|-------|--------------|--------------|
| **Schema Design Fundamentals** | `data-schema-design` | Generate production-grade SQL schemas with indexing, partitioning, and data type strategies for Postgres/Snowflake/BigQuery |
| **ETL Pattern Library** | `data-etl-patterns` | Common ETL patterns: SCD Type 2, slowly changing facts, incremental loads, full refreshes with idempotency |
| **Data Quality Framework** | `data-quality-checks` | Generate dbt tests, Great Expectations, or SQL validation rules for completeness, uniqueness, referential integrity |
| **Local AI Data Stack** | `local-ai-data` | Optimize Claude Skills for local LLMs with proper context windows, fallback strategies, and model selection |
| **DuckDB & Analytics** | `duckdb-analytics` | Modern analytics patterns: columnar design, query optimization, analytical SQL best practices |

## Quick Start

### Install with CLI

```bash
npx skills add https://github.com/Yuuzulight/data-engineering-skills --skill "data-schema-design"
```

### Or Copy-Paste

1. Open [skills/data-schema-design/SKILL.md](skills/data-schema-design/SKILL.md)
2. Copy the full file
3. Paste into Claude Code / Cursor / your agent of choice

## Examples

### Before: Naive Schema
```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255),
  email VARCHAR(255),
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);
```

### After: Production Schema (with skill)
```sql
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,
  name VARCHAR(255) NOT NULL,
  -- Denormalized domain for efficient filtering
  email_domain VARCHAR(255) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- High-cardinality search
CREATE INDEX idx_users_email ON users(email);
-- Domain-based queries (e.g., "all @company.com users")
CREATE INDEX idx_users_email_domain ON users(email_domain);
-- Time-range queries
CREATE INDEX idx_users_created_at ON users(created_at DESC);

-- Constraints for data quality
ALTER TABLE users ADD CONSTRAINT chk_email_format 
  CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$');
```

## Repository Structure

```
data-engineering-skills/
├── README.md                           # This file
├── LICENSE                             # MIT
├── CHANGELOG.md                        # Version history
├── skills/
│   ├── data-schema-design/
│   │   └── SKILL.md                    # Main skill file
│   ├── data-etl-patterns/
│   │   └── SKILL.md
│   ├── data-quality-checks/
│   │   └── SKILL.md
│   ├── local-ai-data/
│   │   └── SKILL.md
│   └── duckdb-analytics/
│       └── SKILL.md
├── examples/
│   ├── schema-design-examples.md       # Real-world schema examples
│   ├── etl-patterns-examples.md        # ETL pattern implementations
│   └── data-quality-examples.md        # Quality check examples
├── research/
│   └── data-engineering-principles.md  # Background reading
└── scripts/
    └── skill.sh                        # CLI helper script
```

## Why These Skills

### Real Problems, Real Solutions

- **Schema Design**: Most AI-generated schemas are under-indexed. We fix that.
- **ETL Patterns**: Incremental loads and slowly-changing dimensions are complex. We teach the patterns.
- **Data Quality**: 80% of data issues come from quality problems upstream. We make them systemic.
- **Local AI Data**: As AI moves local, data stacks need to optimize for resource constraints. We show how.
- **Analytics**: Modern analytics (DuckDB, Trino, etc.) require different thinking than OLTP. We bridge the gap.

## Who Should Use This

- **Data Engineers** who want Claude Code to generate better database designs
- **ML Engineers** who need production data pipelines but aren't data specialists
- **Startups** building data infrastructure quickly without hiring a full data team
- **Data Teams** who want to standardize how AI generates data systems

## Getting Help

- **Have a question?** Open a [GitHub Discussion](https://github.com/Yuuzulight/data-engineering-skills/discussions)
- **Found a bug or gap?** [Open an Issue](https://github.com/Yuuzulight/data-engineering-skills/issues)
- **Want to contribute?** See [CONTRIBUTING.md](CONTRIBUTING.md)

## Support Development

Building and maintaining these skills takes time. If they're saving you hours on database design, ETL pipelines, or data quality, consider sponsoring development.

**[Support Data Artisan on GitHub Sponsors](https://github.com/sponsors/Yuuzulight)**

Sponsors get:
- Early access to new skills
- Priority for feature requests
- Recognition in README
- Direct feedback channel

## Sponsors

<!-- sponsors --><!-- sponsors -->

## Roadmap

- [ ] **Kafka & Streaming Skills** – streaming data patterns, exactly-once semantics, backpressure handling
- [ ] **dbt Advanced Patterns** – macro patterns, custom tests, performance optimization
- [ ] **Data Governance** – PII masking, lineage tracking, access control patterns
- [ ] **Cloud Cost Optimization** – Snowflake cost patterns, BigQuery slot management, data lifecycle policies
- [ ] **Real-time Analytics** – event streaming, dimensional modeling for real-time

## License

MIT License © 2026. See [LICENSE](LICENSE) for details.

## Disclaimer

These skills are designed to enhance AI-generated code, not replace human data engineers. Always review generated code, test thoroughly, and follow your organization's data governance policies.

---

**Questions?** Open a discussion or tweet [@Yuuzulight](https://twitter.com/Yuuzulight)
