---
name: fermyon-provision-and-link-cloud-state
description: Create Fermyon Cloud key-value stores, SQLite databases and application variables, and link or unlink them to applications through resource labels.
api: Fermyon Cloud REST API
base_url: https://cloud.fermyon.com
generated: '2026-09-09'
method: generated
source: openapi/_original/fermyon-openapi.yml + https://developer.fermyon.com/cloud/linking-applications-to-resources-using-labels
operations:
  - 'GET /api/key-value-stores'
  - 'POST /api/key-value-stores/{store}'
  - 'PATCH /api/key-value-stores/{store}/rename'
  - 'POST /api/key-value-stores/{store}/links'
  - 'DELETE /api/key-value-stores/{store}/links'
  - 'POST /api/key-value-pairs'
  - 'GET /api/sql-databases'
  - 'POST /api/sql-databases/create'
  - 'POST /api/sql-databases/execute'
  - 'PATCH /api/sql-databases/{database}/rename'
  - 'POST /api/sql-databases/{database}/links'
  - 'DELETE /api/sql-databases/{database}/links'
  - 'GET /api/variable-pairs'
  - 'POST /api/variable-pairs'
---

# Provision and link Fermyon Cloud state

Stateful resources are **not owned by** an application. A key-value store or SQL database is
its own entity that carries a `links[]` of `ResourceLabel` objects, each naming an app and
the local label that app reads it under. That is why one store can serve several apps — and
why unlinking is not the same as deleting.

## Key-value store

1. `GET /api/key-value-stores` — check whether the store already exists (`KeyValueStoreItem`
   with `name` and `links`).
2. `POST /api/key-value-stores/{store}` — create it.
3. `POST /api/key-value-stores/{store}/links` — attach it to an app under a label.
4. `POST /api/key-value-pairs` with a `CreateKeyValuePairCommand` (`appId`, key, value) to
   seed data.
5. `DELETE /api/key-value-stores/{store}/links` detaches without destroying data.
   `PATCH /api/key-value-stores/{store}/rename` renames in place.

## SQL (SQLite) database

1. `GET /api/sql-databases` — list; `Database` carries `name`, `default` and `links`.
2. `POST /api/sql-databases/create` with a `CreateSqlDatabaseCommand` (`appId`, name).
3. `POST /api/sql-databases/execute` with an `ExecuteSqlStatementCommand` to run DDL/DML.
4. `POST` / `DELETE /api/sql-databases/{database}/links` to attach and detach.

## Application variables

`GET /api/variable-pairs` (`GetVariablesQuery.appId`), `POST /api/variable-pairs` to set,
`DELETE /api/variable-pairs` to remove. Variables are per-app, not per-channel — the
channel-level `environmentVariables` on `ChannelItem` are a separate, read-side view.

## Rules an agent must follow here

- **Unlink, do not delete.** `DELETE /api/key-value-stores/{store}` and
  `DELETE /api/sql-databases` destroy the data. Neither has a restore path or a retention
  window in any Fermyon documentation. Detaching via the `/links` endpoints is the
  reversible action; deletion is not.
- **Deleting the app takes the state with it.** Fermyon documents that deleting an
  application also permanently deletes its default key-value store and all its application
  variables.
- **Quotas are per plan.** 5 key-value stores and 1 GB of KV storage on Starter; 100 stores
  and 2 GB on Growth. KV key size is capped at 255 bytes, value size at 1 MB, and 1,024 keys
  per store. Check `rate-limits/fermyon-rate-limits.yml` before bulk-loading.
- No idempotency key exists on any of these writes. Read back with the matching `GET`
  before retrying a create that timed out.
