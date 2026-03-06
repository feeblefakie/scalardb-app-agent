# ScalarDB Application Development Agent

This project provides a Claude Code development agent for building applications with [ScalarDB](https://scalardb.scalar-labs.com/), a distributed transaction database library that provides ACID transactions across diverse databases.

## What This Agent Provides

- **Skills** (interactive slash commands): `/scalardb-model`, `/scalardb-config`, `/scalardb-scaffold`, `/scalardb-error-handler`, `/scalardb-local-env`, `/scalardb-crud-ops`, `/scalardb-jdbc-ops`, `/scalardb-docs`
- **Rules** (always-on passive guidance): Exception handling, CRUD patterns, JDBC patterns, 2PC I/F patterns, config validation, schema design, Java best practices
- **Sub-agents** (specialized autonomous workers): Code reviewer, app builder, migration advisor

## Reference Documentation

Detailed reference documentation lives in `.claude/docs/`:
- `api-reference.md` — Condensed API reference
- `exception-hierarchy.md` — Exception handling reference with decision flowchart
- `configuration-reference.md` — Configuration properties for all storage types
- `schema-format.md` — Schema JSON and SQL DDL format
- `interface-matrix.md` — Guide to the 6 interface combinations
- `sql-reference.md` — Supported SQL grammar for JDBC/SQL interface
- `code-patterns/` — Complete working examples for each interface combination

## ScalarDB Overview

ScalarDB provides ACID transactions across diverse databases. Key concepts:

- **Interface styles**: CRUD API (Java native) or JDBC/SQL (standard SQL)
- **Deployment modes**: Core (direct DB connection) or Cluster (via ScalarDB Cluster)
- **Transaction interfaces**: Standard interface (single `commit()`) or Two-phase commit interface (`prepare()` + `commit()` for microservice coordination). Both work with multiple databases. ScalarDB internally uses two-phase commit regardless; the distinction is only about the interface exposed to the application.
- **6 interface combinations**: Core+CRUD+Standard, Core+CRUD+2PC-I/F, Cluster+CRUD+Standard, Cluster+CRUD+2PC-I/F, Cluster+JDBC+Standard, Cluster+JDBC+2PC-I/F

## Key Source Code Locations

When the user is working in a ScalarDB project, these are the main packages:
- Core API: `com.scalar.db.api` — Transaction managers, CRUD operations, Result, Key
- Exceptions: `com.scalar.db.exception.transaction` — All transaction exceptions
- IO types: `com.scalar.db.io` — Key, Column, DataType
- JDBC driver: `com.scalar.db.sql.jdbc` — JDBC connection and driver

## Critical Rules

### CRUD API
1. **Exception handling order**: Always catch `*ConflictException` BEFORE parent `*Exception` classes
2. **Transaction lifecycle**: Always `commit()` even for read-only transactions; always `rollback()`/`abort()` in catch blocks
3. **Deprecated API**: `Put` is deprecated since 3.13.0 — use `Insert`, `Upsert`, `Update` instead
4. **Builder pattern**: Always use `Get.newBuilder()`, `Scan.newBuilder()`, etc.
5. **Namespace and table**: Always specify `.namespace()` and `.table()` explicitly on operations

### JDBC/SQL
6. **Always `setAutoCommit(false)`** on every JDBC Connection
7. **Error code 301**: `UnknownTransactionStatusException` — do NOT rollback, verify if committed
8. **Always commit** even for read-only JDBC transactions
9. **Use PreparedStatement** with parameter binding — never string concatenation
10. **Quote reserved words** in SQL with double quotes: `"timestamp"`, `"order"`, `"key"`
