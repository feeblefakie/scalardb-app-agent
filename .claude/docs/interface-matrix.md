# ScalarDB Interface Combinations Matrix

ScalarDB supports 6 interface combinations along three axes:

- **Deployment**: Core (direct DB connection) or Cluster (via ScalarDB Cluster)
- **API**: CRUD (Java native) or SQL/JDBC (standard SQL)
- **Transaction interface**: Standard interface (single `commit()`) or Two-phase commit interface (`prepare()` + `commit()` for microservices). Both work with multiple databases. ScalarDB internally uses two-phase commit regardless; the distinction is only about the interface exposed to the application.

## The 6 Combinations

| # | Name | Deployment | API | Transaction Interface | Use Case |
|---|------|-----------|-----|----------------------|----------|
| 1 | Core+CRUD+Standard | Core | CRUD | Standard | Standard transactions, development/testing |
| 2 | Core+CRUD+2PC-I/F | Core | CRUD | 2PC I/F | Microservice coordination from single app |
| 3 | Cluster+CRUD+Standard | Cluster | CRUD | Standard | Production standard transactions with ScalarDB Cluster |
| 4 | Cluster+CRUD+2PC-I/F | Cluster | CRUD | 2PC I/F | Production microservice coordination with CRUD API |
| 5 | Cluster+JDBC+Standard | Cluster | JDBC/SQL | Standard | Production with standard SQL interface |
| 6 | Cluster+JDBC+2PC-I/F | Cluster | JDBC/SQL | 2PC I/F | Production microservice coordination with SQL interface |

## Decision Guide

```
Do you need ScalarDB Cluster (production, auth, encryption)?
├── No → Core mode
│   ├── Single transaction manager instance? → Core+CRUD+Standard (#1) ← Most common for development
│   └── Need microservice coordination (multiple tx manager instances)? → Core+CRUD+2PC-I/F (#2)
│
└── Yes → Cluster mode
    ├── Prefer Java native API?
    │   ├── Single transaction manager instance? → Cluster+CRUD+Standard (#3)
    │   └── Need microservice coordination? → Cluster+CRUD+2PC-I/F (#4)
    │
    └── Prefer standard SQL?
        ├── Single transaction manager instance? → Cluster+JDBC+Standard (#5)
        └── Need microservice coordination? → Cluster+JDBC+2PC-I/F (#6)
```

## Dependency Matrix

| Combination | Maven Artifact | Artifact ID |
|-------------|---------------|-------------|
| Core+CRUD (Standard/2PC I/F) | `com.scalar-labs:scalardb` | `scalardb` |
| Cluster+CRUD (Standard/2PC I/F) | `com.scalar-labs:scalardb-cluster-java-client-sdk` | `scalardb-cluster-java-client-sdk` |
| Cluster+JDBC (Standard/2PC I/F) | `com.scalar-labs:scalardb-sql-jdbc` + `com.scalar-labs:scalardb-cluster-java-client-sdk` | `scalardb-sql-jdbc` |

## Configuration Matrix

### Core+CRUD (Standard Interface)

```properties
scalar.db.storage=jdbc
scalar.db.contact_points=jdbc:mysql://localhost:3306/
scalar.db.username=root
scalar.db.password=mysql
```

### Core+CRUD (2PC I/F)

Same as the standard interface, but use `TwoPhaseCommitTransactionManager` instead of `DistributedTransactionManager`.

### Cluster+CRUD (Standard/2PC I/F)

```properties
scalar.db.transaction_manager=cluster
scalar.db.contact_points=indirect:<CLUSTER_HOST>
```

### Cluster+JDBC (Standard/2PC I/F)

```properties
scalar.db.sql.connection_mode=cluster
scalar.db.sql.cluster_mode.contact_points=indirect:<CLUSTER_HOST>
```

JDBC URL: `jdbc:scalardb:<properties-file-path>`

## Code Pattern Matrix

### Core+CRUD — Transaction Initialization

```java
TransactionFactory factory = TransactionFactory.create("database.properties");
DistributedTransactionManager manager = factory.getTransactionManager();
DistributedTransaction tx = manager.begin();
```

### Core+CRUD+2PC-I/F — Transaction Initialization

```java
TransactionFactory factory = TransactionFactory.create("database.properties");
TwoPhaseCommitTransactionManager manager = factory.getTwoPhaseCommitTransactionManager();
TwoPhaseCommitTransaction tx = manager.begin();   // coordinator
TwoPhaseCommitTransaction tx = manager.join(txId); // participant
```

### Cluster+CRUD — Transaction Initialization

Same code as Core+CRUD, but config uses `scalar.db.transaction_manager=cluster`.

### Cluster+JDBC — Connection

```java
Connection conn = DriverManager.getConnection("jdbc:scalardb:scalardb-sql.properties");
conn.setAutoCommit(false);
// Standard JDBC operations
conn.commit();
```

## Schema Loading

| Combination | Schema Format | Loader Tool |
|-------------|--------------|-------------|
| Core+CRUD | JSON (`schema.json`) | `scalardb-schema-loader` |
| Cluster+CRUD | JSON (`schema.json`) | `scalardb-cluster-schema-loader` |
| Cluster+JDBC | SQL (`schema.sql`) or JSON | `scalardb-cluster-sql-cli` or `scalardb-cluster-schema-loader` |

## CRUD API vs JDBC/SQL Comparison

| Feature | CRUD API | JDBC/SQL |
|---------|---------|---------|
| JOIN support | No (application-level only) | Yes (INNER, LEFT, RIGHT JOIN) |
| Aggregate functions | No | Yes (COUNT, SUM, AVG, MIN, MAX) |
| GROUP BY / HAVING | No | Yes |
| Standard SQL syntax | No | Yes |
| Builder pattern | Yes | N/A (SQL strings) |
| Type-safe operations | Yes | No (string-based SQL) |
| Direct column access | `result.getInt("col")` | `resultSet.getInt("col")` |
| Performance | Better (no SQL parsing) | Slightly lower (SQL parsing) |
| Learning curve | ScalarDB-specific | Standard JDBC knowledge |
