# Data Artisan

AI-powered skills for building production-grade data pipelines, schemas, and systems—with Claude Code, Cursor, or any agent that supports Agent Skills.

[![CI](https://github.com/Yuuzulight/db-artisan/actions/workflows/ci.yml/badge.svg)](https://github.com/Yuuzulight/db-artisan/actions/workflows/ci.yml)
[![Agent Skills compatible](https://img.shields.io/badge/Agent%20Skills-compatible-success)](https://github.com/vercel-labs/agent-skills)
[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![GitHub Stars](https://img.shields.io/github/stars/Yuuzulight/db-artisan?style=social)](https://github.com/Yuuzulight/db-artisan)

## The Problem

When you ask Claude Code to generate a database schema or data pipeline, it creates **working code**, but often misses the patterns that make systems production-ready:

- Missing indexes on high-cardinality columns
- No partition strategies for scale
- Incomplete data quality checks
- No lineage or observability patterns
- Designs that work at 1GB but break at 1TB

**Data Artisan** teaches AI agents how to think like data engineers: designing for scale, reliability, and operations from day one.

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

One skill so far. The rest are on the [roadmap](#roadmap).

## Quick Start

### Install with CLI

```bash
npx skills add https://github.com/Yuuzulight/db-artisan --skill "data-schema-design"
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
db-artisan/
├── README.md                           # This file
├── LICENSE                             # MIT
├── CHANGELOG.md                        # Version history
├── CONTRIBUTING.md                     # How to contribute
├── .github/
│   └── workflows/
│       └── ci.yml                      # Markdown lint on push and PR
├── skills/
│   └── data-schema-design/
│       └── SKILL.md                    # The skill file
└── examples/
    └── schema-design-examples.md       # Real-world schema examples
```

## Why This Skill

Most AI-generated schemas are under-indexed. They compile, they pass a smoke test, and then they buckle at scale—no partition strategy, no constraints catching bad data on the way in, and indexes that don't match the queries anyone actually runs.

Schema Design Fundamentals front-loads the decisions a data engineer would make anyway, so the first draft lands closer to something you'd be willing to ship.

## Who Should Use This

- **Data Engineers** who want Claude Code to generate better database designs
- **ML Engineers** who need production data pipelines but aren't data specialists
- **Startups** building data infrastructure quickly without hiring a full data team
- **Data Teams** who want to standardize how AI generates data systems

## Getting Help

- **Have a question?** Open a [GitHub Discussion](https://github.com/Yuuzulight/db-artisan/discussions)
- **Found a bug or gap?** [Open an Issue](https://github.com/Yuuzulight/db-artisan/issues)
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

- [ ] **ETL Pattern Library** (`data-etl-patterns`) – SCD Type 2, slowly changing facts, incremental loads, idempotent full refreshes
- [ ] **Data Quality Framework** (`data-quality-checks`) – dbt tests, Great Expectations, SQL validation for completeness, uniqueness, referential integrity
- [ ] **DuckDB & Analytics** (`duckdb-analytics`) – columnar design, query optimization, analytical SQL best practices
- [ ] **Local AI Data Stack** (`local-ai-data`) – context windows, fallback strategies, and model selection for local LLMs
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
