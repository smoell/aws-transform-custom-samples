# JPA → Quarkus Persistence Reference

> Reference for Phase 1 Step 4 (config migration) and Phase 2 Step 7 (JPA patterns).
> See also: https://quarkus.io/guides/hibernate-orm

## persistence.xml → application.properties Mapping

### Before/After Example

**BEFORE** — `META-INF/persistence.xml` with a `<persistence-unit name="primary" transaction-type="JTA">`, a `<jta-data-source>`, and `<properties>` for JDBC URL/user/password/driver plus `hibernate.dialect`, `hibernate.hbm2ddl.auto`, `show_sql`, `format_sql`, `default_schema`, `jdbc.batch_size`, `order_inserts`, `order_updates`, second-level cache.

**AFTER** — `src/main/resources/application.properties`:

```properties
# Datasource
quarkus.datasource.db-kind=postgresql
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/myapp
quarkus.datasource.username=dbuser
quarkus.datasource.password=[REDACTED_PASSWORD]
quarkus.datasource.jdbc.max-size=20
quarkus.datasource.jdbc.min-size=5

# Hibernate ORM
quarkus.hibernate-orm.database.generation=validate
quarkus.hibernate-orm.log.sql=true
quarkus.hibernate-orm.log.format-sql=true
quarkus.hibernate-orm.database.default-schema=app
quarkus.hibernate-orm.jdbc.statement-batch-size=25
# Dialect auto-detected from db-kind — omit unless overriding
# Second-level cache (per entity)
quarkus.hibernate-orm.cache."com.example.model.Product".expiration.max-idle=3600
# Statistics (dev/test only)
%dev.quarkus.hibernate-orm.statistics=true
%test.quarkus.hibernate-orm.statistics=true
```

### Property Mapping Table

| persistence.xml / Hibernate Property | Quarkus Property | Notes |
|---|---|---|
| `javax.persistence.jdbc.url` | `quarkus.datasource.jdbc.url` | Direct |
| `javax.persistence.jdbc.user` | `quarkus.datasource.username` | Direct |
| `javax.persistence.jdbc.password` | `quarkus.datasource.password` | Direct |
| `javax.persistence.jdbc.driver` | (auto-detected) | Inferred from `db-kind` |
| `<jta-data-source>java:jboss/...` | `quarkus.datasource.*` | No JNDI — configure directly |
| `hibernate.dialect` | `quarkus.hibernate-orm.dialect` | Usually auto-detected; omit unless custom |
| `hibernate.hbm2ddl.auto` | `quarkus.hibernate-orm.database.generation` | `none`/`create`/`drop-and-create`/`update`/`validate` |
| `hibernate.show_sql` | `quarkus.hibernate-orm.log.sql` | |
| `hibernate.format_sql` | `quarkus.hibernate-orm.log.format-sql` | |
| `hibernate.default_schema` | `quarkus.hibernate-orm.database.default-schema` | |
| `hibernate.default_catalog` | `quarkus.hibernate-orm.database.default-catalog` | |
| `hibernate.jdbc.batch_size` | `quarkus.hibernate-orm.jdbc.statement-batch-size` | |
| `hibernate.order_inserts` | `quarkus.hibernate-orm.order-inserts` | |
| `hibernate.order_updates` | `quarkus.hibernate-orm.order-updates` | |
| `hibernate.generate_statistics` | `quarkus.hibernate-orm.statistics` | |
| `hibernate.physical_naming_strategy` | `quarkus.hibernate-orm.physical-naming-strategy` | FQ class name |
| `hibernate.implicit_naming_strategy` | `quarkus.hibernate-orm.implicit-naming-strategy` | FQ class name |
| `hibernate.cache.use_second_level_cache` | (configure per-entity) | See caching below |
| Connection pool (C3P0/HikariCP) | `quarkus.datasource.jdbc.*` | Quarkus uses Agroal |

### Database Kind Values

| Database | `db-kind` | JDBC Extension |
|---|---|---|
| PostgreSQL | `postgresql` | `quarkus-jdbc-postgresql` |
| MySQL / MariaDB | `mysql` / `mariadb` | `quarkus-jdbc-mysql` / `-mariadb` |
| H2 | `h2` | `quarkus-jdbc-h2` |
| Oracle | `oracle` | `quarkus-jdbc-oracle` |
| SQL Server | `mssql` | `quarkus-jdbc-mssql` |
| DB2 | `db2` | `quarkus-jdbc-db2` |
| Derby | `derby` | `quarkus-jdbc-derby` |

### Connection Pool (Agroal)

Quarkus uses Agroal (replaces C3P0/HikariCP/app-server pools):

```properties
quarkus.datasource.jdbc.min-size=5
quarkus.datasource.jdbc.max-size=20
quarkus.datasource.jdbc.acquisition-timeout=30S
quarkus.datasource.jdbc.idle-removal-interval=5M
quarkus.datasource.jdbc.max-lifetime=30M
quarkus.datasource.jdbc.leak-detection-interval=1M
```

### JTA DataSource JNDI Resolution

When persistence.xml references a JNDI datasource, locate the actual JDBC details from: (1) `*-ds.xml` in the project, (2) `standalone.xml` datasource subsystem, or (3) app-server admin console. Extract URL/driver/username/password and configure directly — there is no JNDI in Quarkus.

## WildFly ExampleDS Default → H2 In-Memory

When persistence.xml references `java:jboss/datasources/ExampleDS` (WildFly default) with no explicit JDBC URL, this implies H2 in-memory:

```properties
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:appname;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
```

**Do NOT set `quarkus.hibernate-orm.dialect`** — Hibernate ORM 6.x auto-detects from `db-kind`; an explicit dialect can cause version-mismatch errors.

## Persistence Unit Name Resolution

`@PersistenceContext(unitName = "primary")` referencing the single PU named "primary" (common WildFly convention) maps to a plain `@Inject EntityManager` — no qualifier needed:

```java
// BEFORE                                  // AFTER
@PersistenceContext(unitName = "primary")  @Inject
private EntityManager em;                  EntityManager em;
```

Only use `@PersistenceUnit("name")` when the app has MULTIPLE persistence units to distinguish.

## Multi-Datasource Patterns

```properties
# Default datasource (unnamed — plain @Inject EntityManager)
quarkus.datasource.db-kind=postgresql
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/orders
quarkus.datasource.username=order_user
quarkus.datasource.password=[REDACTED_PASSWORD]

# Named datasource "inventory"
quarkus.datasource."inventory".db-kind=postgresql
quarkus.datasource."inventory".jdbc.url=jdbc:postgresql://localhost:5432/inventory
quarkus.datasource."inventory".username=inv_user
quarkus.datasource."inventory".password=[REDACTED_PASSWORD]

# Hibernate ORM — default + named persistence units
quarkus.hibernate-orm.database.generation=validate
quarkus.hibernate-orm.packages=com.example.orders.model
quarkus.hibernate-orm."inventory".datasource=inventory
quarkus.hibernate-orm."inventory".database.generation=validate
quarkus.hibernate-orm."inventory".packages=com.example.inventory.model
```

```java
@ApplicationScoped
public class MultiDatasourceService {
    @Inject EntityManager defaultEm;                          // default PU
    @Inject @PersistenceUnit("inventory") EntityManager inventoryEm;  // named PU

    @Transactional
    public void createOrder(Order order) { defaultEm.persist(order); }

    @Transactional
    @PersistenceUnit("inventory")  // qualifies the transaction manager too
    public void updateInventory(InventoryItem item) { inventoryEm.merge(item); }
}
```

**Notes**: Each named PU needs its own `packages` config; an entity cannot belong to multiple PUs. `@Transactional` uses the default transaction manager — qualify with `@PersistenceUnit("name")` on the method for named datasources. Each `persistence.xml` `<persistence-unit>` becomes a named datasource + named Hibernate ORM block.

## Optional: Panache Migration

> **IMPORTANT**: Panache adoption is OPTIONAL. Do NOT force conversion on existing codebases. Adopt only when: starting new entities/repositories from scratch, the team explicitly requests it, or the app has simple CRUD patterns that benefit from boilerplate reduction.

### Repository Pattern — PanacheRepository

```xml
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-hibernate-orm-panache</artifactId></dependency>
```

```java
// A manual EntityManager repository becomes:
@ApplicationScoped
public class OrderRepository implements PanacheRepository<Order> {
    // findById(), persist(), delete(), count(), listAll() — all inherited
    public List<Order> findByStatus(String status) { return list("status", status); }
}
```

### Active Record Pattern — PanacheEntity

```java
@Entity
public class Order extends PanacheEntity {   // id field inherited (auto-generated Long)
    public String status;                    // public fields — no getters/setters needed
    public BigDecimal total;
    public static List<Order> findByStatus(String status) { return list("status", status); }
}
// Usage: order.persist();  Order.findById(1L);  Order.deleteById(1L);
```

- `PanacheEntity` — provides auto-generated `Long id`. Use for simple auto-increment IDs.
- `PanacheEntityBase` — no ID field. Use for custom ID type or composite key (declare your own `@Id`).

### When to Recommend Panache vs Plain JPA

| Use Panache When | Keep Plain JPA When |
|---|---|
| Simple CRUD-heavy application | Complex query logic with Criteria API |
| New entities created during migration | Large existing entity model (avoid rewrite) |
| Team familiar with active record | Team prefers explicit EntityManager control |
| Rapid prototyping / small services | Multi-datasource with complex TX routing |

## Hibernate-Specific Features

Fully supported in Hibernate 6.x (bundled with quarkus-hibernate-orm): `@NaturalId`, `@Formula`, `@Filter`/`@FilterDef`, `@SQLDelete`/`@Where`, `@DynamicInsert`/`@DynamicUpdate`, `@BatchSize`, `@Fetch`, `@GenericGenerator`, second-level cache (`@jakarta.persistence.Cacheable`). `@TypeDef`/`@Type` changed in Hibernate 6 — see below.

### Hibernate Envers (Audit)

```xml
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-hibernate-envers</artifactId></dependency>
```
```properties
quarkus.hibernate-envers.store-data-at-delete=true
quarkus.hibernate-envers.audit-table-suffix=_AUD
```
`@Audited` works unchanged after namespace migration.

### Hibernate Search

```xml
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-hibernate-search-orm-elasticsearch</artifactId></dependency>
```
```properties
quarkus.hibernate-search-orm.elasticsearch.version=8
quarkus.hibernate-search-orm.elasticsearch.hosts=localhost:9200
```
`@Indexed`, `@FullTextField`, `@KeywordField` work as-is.

### @Type Migration (Hibernate 5 → 6)

```java
// BEFORE (Hibernate 5)                      // AFTER (Hibernate 6)
@Type(type = "org.hibernate.type.TextType")  @JdbcTypeCode(Types.LONGVARCHAR)
private String description;                   private String description;

@Type(type = "json")                          @Type(JsonType.class)  // class ref, not string
private Map<String,Object> metadata;          @Column(columnDefinition = "jsonb")
                                              private Map<String,Object> metadata;
```

### Second-Level Cache

```properties
quarkus.hibernate-orm.cache."com.example.model.Product".expiration.max-idle=3600
quarkus.hibernate-orm.cache."com.example.model.Category".memory.object-count=1000
```
```java
@Entity @Cacheable  // jakarta.persistence.Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product { ... }
```

## Key Differences from App-Server JPA

### No JNDI DataSource Lookup

Datasources are configured directly in `application.properties` — no JNDI context. For programmatic access:
```java
@Inject AgroalDataSource dataSource;                                   // default
@Inject @io.quarkus.agroal.DataSource("inventory") AgroalDataSource inventoryDs;  // named
```

### persistence.xml Not Required (But Supported)

Quarkus does NOT require `persistence.xml` — all config goes to `application.properties`. If present, Quarkus reads it for compatibility (useful during incremental migration). **Recommendation**: remove `persistence.xml` and use `application.properties` exclusively; mixed config causes confusion.

### Build-Time Entity Scanning

ArC discovers entities at build time. Entities in application source packages are found automatically; entities in external JARs must be listed in `quarkus.hibernate-orm.packages` or the JAR must contain a Jandex index. Missing scanning symptom: `Unknown entity: com.example.SomeEntity` at runtime.

### Lazy Loading in Native Mode

Quarkus enables build-time bytecode enhancement, so `@ManyToOne`/`@OneToOne`/`@OneToMany`/`@ManyToMany(fetch = LAZY)` work in native mode. Accessing a lazy field outside a transaction/session still throws `LazyInitializationException` (harder to debug in native). **Best practice**: use `@Fetch(FetchMode.SUBSELECT)` or `@BatchSize` for collections and `JOIN FETCH` in queries for known access patterns.

### EntityManager Injection & Transactions

```java
// @PersistenceContext still works, but @Inject is idiomatic:
@Inject EntityManager em;
@Inject @PersistenceUnit("inventory") EntityManager inventoryEm;
```

App servers manage transactions implicitly; Quarkus needs explicit `@Transactional` (no implicit CMT like `@Stateless`). A transaction is required even for reads for the EntityManager to work — explicit `@Transactional` on read methods is safest.

### import.sql

Quarkus supports `import.sql` in `src/main/resources/` for initial data loading (runs when `database.generation` is `create` or `drop-and-create`):

```properties
quarkus.hibernate-orm.database.generation=drop-and-create
quarkus.hibernate-orm.sql-load-script=import.sql
# Disable loading: quarkus.hibernate-orm.sql-load-script=no-file
```

**import.sql table names must match `@Table(name=...)`** or Hibernate fails at startup/insert with "table not found". Unlike Spring Boot, there is no `data.sql` convention — use `import.sql` (Hibernate-native).
