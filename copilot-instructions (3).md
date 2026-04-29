# GitHub Copilot Instructions - Database

## Current State

- We are migrating from MongoDB/AWS DocumentDB to PostgreSQL.
- PostgreSQL schema migrations follow a database-first approach using Atlas.

## Schema Workflow (PostgreSQL)

- Define and migrate schema changes via Atlas using the HCL files in this folder.
- After any HCL change, run the appropriate per-project `db_scaffold.ps1` script (for example, `Source/AntiMoneyLaundering/Aca.AntiMoneyLaundering/db_scaffold.ps1`) to regenerate that project's scaffolded artifacts.

## Indexing Rules

- If a field is used for lookups, it should be indexed.
- For compound lookups, create a compound index (not multiple individual indexes).

