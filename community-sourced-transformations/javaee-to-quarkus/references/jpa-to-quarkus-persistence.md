# JPA → Quarkus Persistence Reference

> Reference for Phase 1 Step 4 (config migration) and Phase 2 Step 7 (JPA patterns).
> See also: https://quarkus.io/guides/hibernate-orm

## persistence.xml → application.properties Mapping

**BEFORE** — `META-INF/persistence.xml` with `<persistence-unit name="primary" transaction-type="JTA">`, a `<jta-data-source>`, and `<properties>` for JDBC URL/user/password/driver + `hibernate.dialect`, `hbm2ddl.auto`, `show_sql`, `format_sql`, `default_schema`, `jdbc.batch_size`, `order_inserts`, `order_updates`, second-level cache.

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
quarkus.hibernate-orm.schema-management.strategy=validate
quarkus.hibernate-orm.log.sql=true
quarkus.hibernate-orm.log.format-sql=true
quarkus.hibernate-orm.database.default-schema=app
quarkus.hibernate-orm.jdbc.statement-batch-size=25
# Dialect auto-detected from db-kind — omit (see Dialect Rules)
quarkus.hibernate-orm.cache."com.example.model.Product".expiration.max-idle=3600
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
| `hibernate.dialect` | `quarkus.hibernate-orm.dialect` | **Omit** — ORM 6 auto-detects from `db-kind`. See Dialect Rules. |
| `hibernate.hbm2ddl.auto` | `quarkus.hibernate-orm.schema-management.strategy` | `none`/`create`/`drop-and-create`/`update`/`validate`. Renamed from `database.generation` in 3.23 (old key warns). |
| `javax.persistence.schema-generation.database.action` | `quarkus.hibernate-orm.schema-management.strategy` | JPA standard — see Property Conflict Resolution |
| `hibernate.show_sql` | `quarkus.hibernate-orm.log.sql` | |
| `hibernate.format_sql` | `quarkus.hibernate-orm.log.format-sql` | |
| `hibernate.default_schema` | `quarkus.hibernate-orm.database.default-schema` | |
| `hibernate.default_catalog` | `quarkus.hibernate-orm.database.default-catalog` | |
| `hibernate.jdbc.batch_size` | `quarkus.hibernate-orm.jdbc.statement-batch-size` | |
| `hibernate.order_inserts` | *(drop — no equivalent)* | ORM 6 orders automatically when batch-size set |
| `hibernate.order_updates` | *(drop — no equivalent)* | Same |
| `hibernate.generate_statistics` | `quarkus.hibernate-orm.statistics` | |
| `hibernate.physical_naming_strategy` | `quarkus.hibernate-orm.physical-naming-strategy` | FQ class name |
| `hibernate.implicit_naming_strategy` | `quarkus.hibernate-orm.implicit-naming-strategy` | FQ class name |
| `hibernate.cache.use_second_level_cache` | (per-entity) | See caching |
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

## Property Conflict Resolution

**Rule 1 — JPA standard wins over Hibernate vendor property.** If both `javax.persistence.schema-generation.database.action` and `hibernate.hbm2ddl.auto` are present, map the JPA standard value to `schema-management.strategy`; discard the Hibernate one. (`none`→`none`, `create`→`create`, `drop-and-create`→`drop-and-create`, `drop`→`drop`.)

**Rule 2 — Do NOT set dialect when db-kind is configured.** ORM 6 auto-detects; setting `quarkus.hibernate-orm.dialect` alongside `db-kind` causes version-mismatch errors. Discard `hibernate.dialect`.

**Rule 3 — Drop properties with no Quarkus equivalent**: `hibernate.order_inserts`, `hibernate.order_updates` (ORM 6 orders automatically with JDBC batching), `hibernate.batch_versioned_data` (internal). Source: [Quarkus #19129](https://github.com/quarkusio/quarkus/issues/19129).

## Dialect Rules (Hibernate ORM 6.x)

1. **NEVER set dialect when db-kind is configured** — auto-detected; explicit dialect → version-mismatch errors.
2. **Version-specific dialects removed** — `Oracle12cDialect`, `PostgreSQL10Dialect`, `MySQL8Dialect` etc. gone in ORM 6. Use base: `OracleDialect`, `PostgreSQLDialect`, `MySQLDialect`.
3. **Omit properties equal to Quarkus defaults** (e.g. `log.sql=false`).
4. **sql-load-script conditional** — `sql-load-script=import.sql` only needed when strategy is `create`/`drop-and-create`.

### Connection Pool (Agroal)

```properties
quarkus.datasource.jdbc.min-size=5
quarkus.datasource.jdbc.max-size=20
quarkus.datasource.jdbc.acquisition-timeout=30S
quarkus.datasource.jdbc.idle-removal-interval=5M
quarkus.datasource.jdbc.max-lifetime=30M
quarkus.datasource.jdbc.leak-detection-interval=1M
```

### JTA DataSource JNDI Resolution

When persistence.xml references a JNDI datasource, locate actual JDBC details from: (1) `*-ds.xml`, (2) `standalone.xml` datasource subsystem, (3) app-server admin console. Configure directly — no JNDI in Quarkus.

## WildFly ExampleDS Default → H2 In-Memory

`java:jboss/datasources/ExampleDS` (WildFly default, no explicit URL) implies H2 in-memory:
```properties
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:appname;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
```
Do NOT set an explicit dialect (auto-detected from `db-kind`).

## Persistence Unit Name Resolution

`@PersistenceContext(unitName = "primary")` (or any name) referencing the SINGLE persistence unit maps to plain `@Inject EntityManager` — no qualifier. `@PersistenceUnit("name")` is ONLY for apps with MULTIPLE units.
```java
// BEFORE: @PersistenceContext(unitName = "primary") private EntityManager em;
// AFTER:  @Inject EntityManager em;
```

## Multi-Datasource Patterns

```properties
# Default (unnamed → plain @Inject EntityManager)
quarkus.datasource.db-kind=postgresql
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/orders
quarkus.datasource.username=order_user
quarkus.datasource.password=[REDACTED_PASSWORD]

# Named "inventory"
quarkus.datasource."inventory".db-kind=postgresql
quarkus.datasource."inventory".jdbc.url=jdbc:postgresql://localhost:5432/inventory
quarkus.datasource."inventory".username=inv_user
quarkus.datasource."inventory".password=[REDACTED_PASSWORD]

quarkus.hibernate-orm.schema-management.strategy=validate
quarkus.hibernate-orm.packages=com.example.orders.model
quarkus.hibernate-orm."inventory".datasource=inventory
quarkus.hibernate-orm."inventory".schema-management.strategy=validate
quarkus.hibernate-orm."inventory".packages=com.example.inventory.model
```

```java
@ApplicationScoped
public class MultiDatasourceService {
    @Inject EntityManager defaultEm;                                 // default PU
    @Inject @PersistenceUnit("inventory") EntityManager inventoryEm; // named PU

    @Transactional public void createOrder(Order o) { defaultEm.persist(o); }

    @Transactional @PersistenceUnit("inventory")  // qualifies TX manager too
    public void updateInventory(InventoryItem i) { inventoryEm.merge(i); }
}
```

**Notes**: each named PU needs its own `packages`; an entity cannot belong to multiple PUs; `@Transactional` uses the default TX manager unless qualified with `@PersistenceUnit("name")`; each `persistence.xml` `<persistence-unit>` becomes a named datasource + named ORM block.

## Optional: Panache Migration

> **OPTIONAL** — do NOT force on existing codebases. Adopt only for: new entities from scratch, explicit team request, or simple CRUD that benefits from less boilerplate.

**Repository pattern:**
```java
@ApplicationScoped
public class OrderRepository implements PanacheRepository<Order> {
    // findById/persist/delete/count/listAll inherited
    public List<Order> findByStatus(String status) { return list("status", status); }
}
```

**Active record:**
```java
@Entity
public class Order extends PanacheEntity {   // auto Long id
    public String status; public BigDecimal total;   // public fields, no getters/setters
    public static List<Order> findByStatus(String s) { return list("status", s); }
}
// order.persist();  Order.findById(1L);  Order.deleteById(1L);
```

- `PanacheEntity` — auto `Long id`. `PanacheEntityBase` — no id field (custom/composite key: declare your own `@Id`).

| Use Panache When | Keep Plain JPA When |
|---|---|
| Simple CRUD-heavy app | Complex Criteria API queries |
| New entities in migration | Large existing entity model |
| Team knows active record | Team prefers explicit EntityManager |
| Rapid prototyping | Multi-datasource w/ complex TX routing |

## Hibernate-Specific Features

Fully supported in ORM 6.x: `@NaturalId`, `@Formula`, `@Filter`/`@FilterDef`, `@SQLDelete`/`@Where`, `@DynamicInsert`/`@DynamicUpdate`, `@BatchSize`, `@Fetch`, `@GenericGenerator`, second-level cache (`@jakarta.persistence.Cacheable`). `@TypeDef`/`@Type` changed — see below.

**Envers (audit):**
```xml
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-hibernate-envers</artifactId></dependency>
```
```properties
quarkus.hibernate-envers.store-data-at-delete=true
quarkus.hibernate-envers.audit-table-suffix=_AUD
```
`@Audited` works unchanged after namespace migration.

**Hibernate Search:**
```xml
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-hibernate-search-orm-elasticsearch</artifactId></dependency>
```
```properties
quarkus.hibernate-search-orm.elasticsearch.version=8
quarkus.hibernate-search-orm.elasticsearch.hosts=localhost:9200
```
`@Indexed`, `@FullTextField`, `@KeywordField` work as-is.

**@Type migration (Hibernate 5 → 6):**
```java
// BEFORE @Type(type="org.hibernate.type.TextType")  →  AFTER @JdbcTypeCode(Types.LONGVARCHAR)
// BEFORE @Type(type="json")                          →  AFTER @Type(JsonType.class) + @Column(columnDefinition="jsonb")
```

**Second-level cache:**
```properties
quarkus.hibernate-orm.cache."com.example.model.Product".expiration.max-idle=3600
```
```java
@Entity @Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product { ... }
```

## Key Differences from App-Server JPA

**No JNDI datasource lookup** — configure in `application.properties`; programmatic access via `@Inject AgroalDataSource` (default) or `@Inject @io.quarkus.agroal.DataSource("inventory") AgroalDataSource`.

**persistence.xml not required (but supported)** — all config → `application.properties`. If present, Quarkus reads it for compatibility (useful during incremental migration). Recommendation: remove it; mixed config causes confusion.

**Build-time entity scanning** — ArC discovers entities at build time. App-source packages: automatic. External JARs: list in `quarkus.hibernate-orm.packages` or provide a Jandex index. Missing-scan symptom: `Unknown entity: com.example.X` at runtime.

**Lazy loading in native mode** — build-time bytecode enhancement makes `@ManyToOne`/`@OneToOne`/`@OneToMany`/`@ManyToMany(fetch=LAZY)` work. Accessing a lazy field outside a session still throws `LazyInitializationException` (harder to debug in native). Best practice: `@Fetch(FetchMode.SUBSELECT)`/`@BatchSize` for collections, `JOIN FETCH` in queries.

**EntityManager injection & transactions** — `@PersistenceContext` still works but `@Inject` is idiomatic. App servers manage TX implicitly; Quarkus needs explicit `@Transactional` (no implicit CMT). A transaction is required even for reads — explicit `@Transactional` on read methods is safest.
```java
@Inject EntityManager em;
@Inject @PersistenceUnit("inventory") EntityManager inventoryEm;
```

**import.sql** — supported in `src/main/resources/` (runs when strategy is `create`/`drop-and-create`):
```properties
quarkus.hibernate-orm.schema-management.strategy=drop-and-create
quarkus.hibernate-orm.sql-load-script=import.sql   # disable: no-file
```
Table names in `import.sql` must match `@Table(name=...)` or Hibernate fails at startup. No `data.sql` convention (unlike Spring Boot) — use `import.sql`.
