# ADR-002: PostgreSQL-only — the stack is omakase

> **Status:** Accepted — 2026-06-10
> **Context:** While planning the `chasqui new` generator (Sprint 6), the
> question came up: should the stack support MariaDB/MySQL/SQLite as
> alternative databases, the way Rails does?

## Decision

**PostgreSQL + pgvector is the identity of the stack, not a default.**
Chasqui does not and will not abstract the database engine.

## Rationale

1. **pgvector is the heart, not a detail.** Long-term memory and FAQ-RAG are
   built on it (`Vector` columns, cosine distance, HNSW/halfvec indexes).
   MariaDB 11.7's native VECTOR, MySQL 9's type, and sqlite-vec exist, but
   each has different SQL, different index semantics, and a different
   LangChain maturity level — abstraction would cost a permanent 3–4×
   migration/test matrix for a small team.
2. **Postgres-specific features are used deliberately:** `JSONB`
   (contact metadata, conversation_state, agent_config), `asyncpg`, the
   savepoint-based test isolation.
3. **The Rails lesson is the opposite one** — Rails calls its curated stack
   *omakase*. ActiveRecord abstracts engines for 20-year-old historical
   reasons; any modern Rails app with vector search pins Postgres anyway.
4. **Postgres is the least restrictive choice in practice:** free locally
   (Postgres.app, brew), Docker (`pgvector/pgvector`), and on every managed
   tier (Neon, Supabase, RDS, Cloud SQL).

## What the generator asks instead

`chasqui new` never asks *which engine* — it asks **where your Postgres is**
(local / docker-compose / managed URL). That covers the real use case behind
the question without mortgaging the project.

## Revisit trigger

If a *dev-toy* mode (zero-install prototyping) shows real demand, a
SQLite/sqlite-vec profile could be evaluated as an isolated experiment —
never as a supported production path.
