# JPA → Quarkus Persistence Reference

> Reference for Phase 1 Step 4 (config migration) and Phase 2 Step 7 (JPA patterns).
> See also: https://quarkus.io/guides/hibernate-orm

## persistence.xml → application.properties Mapping

### Complete Before/After Example

**BEFORE — `src/main/resources/META-INF/persistence.xml`:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence version="2.2"
             xmlns="http://xmlns.jcp.org/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence
             http://xmlns.jcp.org/xml/ns/persistence/persistence_2_2.xsd">

    <persistence-unit name="primary" transaction-type="JTA">
        <jta-data-source>java:jboss/datasources/MyAppDS</jta-data-source>

        <properties>
            <property name="javax.persistence.jdbc.url"
                      value="jdbc:postgresql://localhost:5432/myapp"/>
            <property name="javax.persistence.jdbc.user" value="dbuser"/>
            <property name="javax.persistence.jdbc.password" value="dbpass"/>
            <property name="javax.persistence.jdbc.driver"
                      value="org.postgresql.Driver"/>

            <property name="hibernate.dialect"
                      value="org.hibernate.dialect.PostgreSQLDialect"/>
            <property name="hibernate.hbm2ddl.auto" value="validate"/>
            <property name="hibernate.show_sql" value="true"/>
            <property name="hibernate.format_sql" value="true"/>
            <property name="hibernate.default_schema" value="app"/>
            <property name="hibernate.jdbc.batch_size" value="25"/>
            <property name="hibernate.order_inserts" value="true"/>
            <property name="hibernate.order_updates" value="true"/>
            <property name="hibernate.generate_statistics" value="false"/>
            <property name="hibernate.cache.use_second_level_cache" value="true"/>
            <property name="hibernate.cache.region.factory_class"
                      value="org.hibernate.cache.jcache.JCacheRegionFactory"/>
        </properties>
    </persistence-unit>
</persistence>
```

**AFTER — `src/main/resources/application.properties`:**

```properties
# Datasource configuration
quarkus.datasource.db-kind=postgresql
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/myapp
quarkus.datasource.username=dbuser
quarkus.datasource.password=dbpass
quarkus.datasource.jdbc.max-size=20
quarkus.datasource.jdbc.min-size=5

# Hibernate ORM configuration
quarkus.hibernate-orm.database.generation=validate
quarkus.hibernate-orm.log.sql=true
quarkus.hibernate-orm.log.format-sql=true
quarkus.hibernate-orm.database.default-schema=app
quarkus.hibernate-orm.jdbc.statement-batch-size=25
quarkus.hibernate-orm.fetch.batch-size=16

# Dialect — usually auto-detected from db-kind (omit unless override needed)
# quarkus.hibernate-orm.dialect=org.hibernate.dialect.PostgreSQLDialect

# Second-level cache
quarkus.hibernate-orm.cache."com.example.model.Product".expiration.max-idle=3600

# Statistics (dev/test only)
%dev.quarkus.hibernate-orm.statistics=true
%test.quarkus.hibernate-orm.statistics=true
```

### Property Mapping Table

| persistence.xml / Hibernate Property | Quarkus Property | Notes |
|---|---|---|
| `javax.persistence.jdbc.url` | `quarkus.datasource.jdbc.url` | Direct mapping |
| `javax.persistence.jdbc.user` | `quarkus.datasource.username` | Direct mapping |
| `javax.persistence.jdbc.password` | `quarkus.datasource.password` | Direct mapping |
| `javax.persistence.jdbc.driver` | (auto-detected) | Quarkus infers from `db-kind` |
| `<jta-data-source>java:jboss/...</jta-data-source>` | `quarkus.datasource.*` (direct config) | No JNDI — configure directly |
| `hibernate.dialect` | `quarkus.hibernate-orm.dialect` | Usually auto-detected from `db-kind`; omit unless custom dialect needed |
| `hibernate.hbm2ddl.auto` | `quarkus.hibernate-orm.database.generation` | Values: `none`, `create`, `drop-and-create`, `update`, `validate` |
| `hibernate.show_sql` | `quarkus.hibernate-orm.log.sql` | `true`/`false` |
| `hibernate.format_sql` | `quarkus.hibernate-orm.log.format-sql` | `true`/`false` |
| `hibernate.default_schema` | `quarkus.hibernate-orm.database.default-schema` | Direct mapping |
| `hibernate.default_catalog` | `quarkus.hibernate-orm.database.default-catalog` | Direct mapping |
| `hibernate.jdbc.batch_size` | `quarkus.hibernate-orm.jdbc.statement-batch-size` | Direct mapping |
| `hibernate.order_inserts` | `quarkus.hibernate-orm.order-inserts` | `true`/`false` |
| `hibernate.order_updates` | `quarkus.hibernate-orm.order-updates` | `true`/`false` |
| `hibernate.generate_statistics` | `quarkus.hibernate-orm.statistics` | `true`/`false` |
| `hibernate.physical_naming_strategy` | `quarkus.hibernate-orm.physical-naming-strategy` | Fully-qualified class name |
| `hibernate.implicit_naming_strategy` | `quarkus.hibernate-orm.implicit-naming-strategy` | Fully-qualified class name |
| `hibernate.cache.use_second_level_cache` | (configure per-entity) | See caching section below |
| Connection pool (C3P0/HikariCP) | `quarkus.datasource.jdbc.*` | Quarkus uses Agroal — see pool config below |

### Database Kind Values

| Database | `quarkus.datasource.db-kind` | JDBC Extension |
|---|---|---|
| PostgreSQL | `postgresql` | `quarkus-jdbc-postgresql` |
| MySQL/MariaDB | `mysql` / `mariadb` | `quarkus-jdbc-mysql` / `quarkus-jdbc-mariadb` |
| H2 | `h2` | `quarkus-jdbc-h2` |
| Oracle | `oracle` | `quarkus-jdbc-oracle` |
| SQL Server | `mssql` | `quarkus-jdbc-mssql` |
| DB2 | `db2` | `quarkus-jdbc-db2` |
| Derby | `derby` | `quarkus-jdbc-derby` |

### Connection Pool Configuration (Agroal)

Quarkus uses Agroal as the connection pool (replaces C3P0, HikariCP, or app-server managed pools):

```properties
quarkus.datasource.jdbc.min-size=5
quarkus.datasource.jdbc.max-size=20
quarkus.datasource.jdbc.initial-size=5
quarkus.datasource.jdbc.acquisition-timeout=30S
quarkus.datasource.jdbc.idle-removal-interval=5M
quarkus.datasource.jdbc.max-lifetime=30M
quarkus.datasource.jdbc.leak-detection-interval=1M
```

### JTA DataSource JNDI Resolution

When persistence.xml references a JNDI datasource (`<jta-data-source>java:jboss/datasources/MyDS</jta-data-source>`), locate the actual JDBC connection details from:

1. `*-ds.xml` in the project (e.g., `my-app-ds.xml`)
2. `standalone.xml` datasource subsystem
3. Application server admin console config

Extract the JDBC URL, driver, username, password and configure directly in `application.properties`. There is no JNDI in Quarkus.

## Multi-Datasource Patterns

### application.properties — Two Datasources

```properties
# Default datasource (unnamed — injected with plain @Inject EntityManager)
quarkus.datasource.db-kind=postgresql
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/orders
quarkus.datasource.username=order_user
quarkus.datasource.password=order_pass
quarkus.datasource.jdbc.max-size=20

# Named datasource: "inventory"
quarkus.datasource."inventory".db-kind=postgresql
quarkus.datasource."inventory".jdbc.url=jdbc:postgresql://localhost:5432/inventory
quarkus.datasource."inventory".username=inv_user
quarkus.datasource."inventory".password=inv_pass
quarkus.datasource."inventory".jdbc.max-size=10

# Hibernate ORM — default persistence unit
quarkus.hibernate-orm.database.generation=validate
quarkus.hibernate-orm.packages=com.example.orders.model

# Hibernate ORM — named persistence unit "inventory"
quarkus.hibernate-orm."inventory".datasource=inventory
quarkus.hibernate-orm."inventory".database.generation=validate
quarkus.hibernate-orm."inventory".packages=com.example.inventory.model
```

### Injection Pattern

```java
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import io.quarkus.hibernate.orm.PersistenceUnit;

@ApplicationScoped
public class MultiDatasourceService {

    // Default persistence unit (no qualifier needed)
    @Inject
    EntityManager defaultEm;

    // Named persistence unit
    @Inject
    @PersistenceUnit("inventory")
    EntityManager inventoryEm;

    @Transactional
    public void createOrder(Order order) {
        defaultEm.persist(order);
    }

    @Transactional
    @PersistenceUnit("inventory")  // qualifies the transaction manager too
    public void updateInventory(InventoryItem item) {
        inventoryEm.merge(item);
    }
}
```

### Multi-Datasource Notes

- Each named persistence unit requires its own `packages` configuration pointing to the entity package.
- Entities MUST be in separate packages per persistence unit — an entity class cannot belong to multiple persistence units.
- `@Transactional` uses the default transaction manager. For named datasources, qualify with `@PersistenceUnit("name")` on the method to route to the correct transaction manager.
- Migration from `persistence.xml` with multiple `<persistence-unit>` elements: each unit becomes a named datasource + named Hibernate ORM configuration block.

## Optional: Panache Migration

> **IMPORTANT**: Panache adoption is OPTIONAL. Do NOT force conversion on existing codebases. Only adopt if:
> - Starting new entities/repositories from scratch
> - The team explicitly requests the Panache programming model
> - The application has simple CRUD patterns that benefit from boilerplate reduction

### Repository Pattern — PanacheRepository

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-hibernate-orm-panache</artifactId>
</dependency>
```

```java
// BEFORE (JavaEE — manual repository with EntityManager)
@ApplicationScoped
public class OrderRepository {
    @Inject
    EntityManager em;

    public Order findById(Long id) {
        return em.find(Order.class, id);
    }

    public List<Order> findByStatus(String status) {
        return em.createQuery("SELECT o FROM Order o WHERE o.status = :status", Order.class)
            .setParameter("status", status)
            .getResultList();
    }

    public void save(Order order) {
        em.persist(order);
    }

    public void delete(Order order) {
        em.remove(em.merge(order));
    }

    public long count() {
        return em.createQuery("SELECT COUNT(o) FROM Order o", Long.class)
            .getSingleResult();
    }
}

// AFTER (Quarkus — PanacheRepository)
@ApplicationScoped
public class OrderRepository implements PanacheRepository<Order> {
    // findById(), persist(), delete(), count(), listAll() — all inherited

    public List<Order> findByStatus(String status) {
        return list("status", status);  // Panache simplified query
    }
}
```

### Active Record Pattern — PanacheEntity

```java
// BEFORE (JavaEE — standard JPA entity)
@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String status;
    private BigDecimal total;

    // getters/setters...
}

// AFTER (Quarkus — PanacheEntity active record)
@Entity
public class Order extends PanacheEntity {
    // id field is inherited (auto-generated Long)
    public String status;    // public fields — no getters/setters needed
    public BigDecimal total;

    // Static finder methods (active record pattern)
    public static List<Order> findByStatus(String status) {
        return list("status", status);
    }

    public static long countActive() {
        return count("status", "ACTIVE");
    }
}

// Usage:
Order order = new Order();
order.status = "NEW";
order.total = BigDecimal.valueOf(99.99);
order.persist();  // save

Order found = Order.findById(1L);  // find
Order.deleteById(1L);              // delete
List<Order> active = Order.findByStatus("ACTIVE");  // custom query
```

### When to Recommend Panache vs Plain JPA

| Use Panache When | Keep Plain JPA When |
|---|---|
| Simple CRUD-heavy application | Complex query logic with Criteria API |
| New entities being created during migration | Large existing entity model (avoid rewrite) |
| Team familiar with active record patterns | Team prefers explicit EntityManager control |
| Rapid prototyping / small services | Multi-datasource with complex transaction routing |
| Entity model is straightforward | Entities use inheritance strategies heavily |

### PanacheEntity vs PanacheEntityBase

- `PanacheEntity` — provides auto-generated `Long id` field. Use when entities use simple auto-increment IDs.
- `PanacheEntityBase` — no ID field provided. Use when entity has a custom ID type or composite key:

```java
@Entity
public class CountryCode extends PanacheEntityBase {
    @Id
    public String code;  // custom String ID (e.g., "US", "GB")
    public String name;
}
```

## Hibernate-Specific Features

### Fully Supported (quarkus-hibernate-orm includes Hibernate 6.x)

| Feature | Status | Notes |
|---|---|---|
| `@NaturalId` | ✅ Supported | Works as-is with `session.byNaturalId()` |
| `@Formula` | ✅ Supported | Computed column mapping |
| `@Filter` / `@FilterDef` | ✅ Supported | Enable via `session.enableFilter("name")` |
| `@SQLDelete` / `@Where` | ✅ Supported | Soft-delete patterns |
| `@DynamicInsert` / `@DynamicUpdate` | ✅ Supported | Omit nulls from INSERT/UPDATE |
| `@BatchSize` | ✅ Supported | Batch loading for collections |
| `@Fetch(FetchMode.SUBSELECT)` | ✅ Supported | Collection fetch strategies |
| `@GenericGenerator` | ✅ Supported | Custom ID generation |
| `@TypeDef` / `@Type` | ⚠ Changed in Hibernate 6 | See migration note below |
| Second-level cache | ✅ Supported | Use `@jakarta.persistence.Cacheable` |

### Hibernate Envers (Audit)

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-hibernate-envers</artifactId>
</dependency>
```

```properties
# Envers configuration
quarkus.hibernate-envers.store-data-at-delete=true
quarkus.hibernate-envers.audit-table-suffix=_AUD
quarkus.hibernate-envers.revision-field-name=REV
quarkus.hibernate-envers.revision-type-field-name=REVTYPE
```

`@Audited` annotations work unchanged after namespace migration.

### Hibernate Search

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-hibernate-search-orm-elasticsearch</artifactId>
</dependency>
```

```properties
quarkus.hibernate-search-orm.elasticsearch.version=8
quarkus.hibernate-search-orm.elasticsearch.hosts=localhost:9200
quarkus.hibernate-search-orm.schema-management.strategy=create-or-validate
```

`@Indexed`, `@FullTextField`, `@KeywordField` annotations work as-is.

### @Type Migration (Hibernate 5 → 6)

Hibernate 6 (bundled with Quarkus 3.x) changes the custom type system:

```java
// BEFORE (Hibernate 5 — @Type with string name)
@Type(type = "org.hibernate.type.TextType")
private String description;

@Type(type = "json")  // custom UserType
private Map<String, Object> metadata;

// AFTER (Hibernate 6 — @JdbcTypeCode or @Type with class ref)
@JdbcTypeCode(Types.LONGVARCHAR)
private String description;

@Type(JsonType.class)  // class reference, not string
@Column(columnDefinition = "jsonb")
private Map<String, Object> metadata;
```

### Second-Level Cache Configuration

```properties
# Enable caching for specific entities
quarkus.hibernate-orm.cache."com.example.model.Product".expiration.max-idle=3600
quarkus.hibernate-orm.cache."com.example.model.Category".memory.object-count=1000

# Or annotate entities
```

```java
@Entity
@Cacheable  // jakarta.persistence.Cacheable — enables L2 cache for this entity
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product {
    // ...
}
```

## Key Differences from App-Server JPA

### No JNDI DataSource Lookup

In application servers, datasources are configured externally and accessed via JNDI:
```xml
<!-- App server configured datasource, accessed via JNDI -->
<jta-data-source>java:jboss/datasources/MyDS</jta-data-source>
```

In Quarkus, datasources are configured directly in `application.properties`. There is no JNDI context. If you need programmatic `DataSource` access:

```java
@Inject
AgroalDataSource dataSource;  // default datasource

@Inject
@io.quarkus.agroal.DataSource("inventory")
AgroalDataSource inventoryDs;  // named datasource
```

### No persistence.xml Required (But Supported)

Quarkus does NOT require `persistence.xml`. All configuration goes to `application.properties`. However, if a `persistence.xml` exists, Quarkus will read it for compatibility — useful during incremental migration.

**Recommendation**: Remove `persistence.xml` and use `application.properties` exclusively. Mixed config (some properties in persistence.xml, some in application.properties) can cause confusion.

### Build-Time Entity Scanning

ArC discovers entities at build time. All entities must be in packages visible to the build:

- Entities in the application's source packages → discovered automatically.
- Entities in external JARs → must be listed in `quarkus.hibernate-orm.packages` or the JAR must contain a Jandex index.

```properties
# If entities are in external JARs or non-standard packages:
quarkus.hibernate-orm.packages=com.example.model,com.thirdparty.entities
```

**Symptom of missing entity scanning**: `Unknown entity: com.example.SomeEntity` at runtime despite the class being on the classpath.

### Lazy Loading in Native Mode

In JVM mode, Hibernate uses runtime bytecode enhancement (proxy generation) for lazy loading. In **native mode**, runtime proxies cannot be generated.

**What still works in native mode**:
- `@ManyToOne(fetch = LAZY)` / `@OneToOne(fetch = LAZY)` — Quarkus enables build-time enhancement
- `@OneToMany(fetch = LAZY)` / `@ManyToMany(fetch = LAZY)` — collection proxies work

**What may require attention**:
- Accessing a lazy-loaded field outside a transaction/session → `LazyInitializationException` (same as JVM mode, but harder to debug in native)
- Custom bytecode-enhanced features relying on runtime instrumentation

**Best practice**: Use `@Fetch(FetchMode.SUBSELECT)` or `@BatchSize` for collections, and `JOIN FETCH` in queries for known access patterns. Avoid relying on implicit lazy loading outside transactions.

### EntityManager Injection Change

```java
// BEFORE (JavaEE — @PersistenceContext)
@PersistenceContext
private EntityManager em;

@PersistenceContext(unitName = "inventory")
private EntityManager inventoryEm;

// AFTER (Quarkus — @Inject)
@Inject
EntityManager em;

@Inject
@PersistenceUnit("inventory")
EntityManager inventoryEm;
```

**Note**: `@PersistenceContext` still works in Quarkus for compatibility, but `@Inject` is the idiomatic pattern. Both produce the same runtime behavior.

### Transaction Management

In app servers, EJB container manages transactions implicitly. In Quarkus, use `@Transactional` (CDI interceptor):

```java
@ApplicationScoped
public class OrderService {
    @Inject
    EntityManager em;

    @Transactional  // required — no implicit CMT like in @Stateless EJBs
    public void createOrder(Order order) {
        em.persist(order);
    }

    // Read-only operations don't strictly need @Transactional,
    // but a transaction is still needed for EntityManager to work.
    // Quarkus auto-creates a transaction for @GET JAX-RS endpoints
    // accessing Hibernate — but explicit @Transactional is safer.
    @Transactional
    public Order findById(Long id) {
        return em.find(Order.class, id);
    }
}
```

### import.sql / data.sql

Quarkus supports `import.sql` in `src/main/resources/` for initial data loading (runs when `database.generation` is `create` or `drop-and-create`):

```properties
quarkus.hibernate-orm.database.generation=drop-and-create
quarkus.hibernate-orm.sql-load-script=import.sql
# For no loading: quarkus.hibernate-orm.sql-load-script=no-file
```

**Note**: Unlike Spring Boot, there is no `data.sql` convention. Use `import.sql` (Hibernate-native feature).
