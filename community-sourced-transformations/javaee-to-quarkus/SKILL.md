---
name: javaee-to-quarkus
description: >-
  Migrates Java EE/Jakarta EE enterprise applications from JBoss EAP/WildFly,
  Payara, GlassFish, or WebLogic to Quarkus 3.x (LTS 3.33) standalone JARs. Covers
  EJB→CDI (ArC), JPA→Hibernate ORM/Panache, JMS→SmallRye Reactive Messaging,
  JSF→Qute, security, namespace migration, containerization. Uses conditional
  pipeline: Phase 0 detects features, subsequent phases run only when needed.
  Trigger: JavaEE, JakartaEE, Jakarta EE, Quarkus, EJB, CDI, JPA, JAX-RS,
  application server, WildFly, JBoss, Payara, GlassFish, WebLogic migration.
---

# JavaEE/JakartaEE to Quarkus Transformation

## Objective

Migrate JavaEE/JakartaEE applications to Quarkus 3.x, eliminating application server dependencies and modernizing to cloud-native containerized deployment. Transform enterprise components to leverage build-time CDI (ArC), convert to uber-jar/native packaging, and adopt Quarkus-optimized patterns while maintaining functional parity.

## Scope

This transformation covers:
- **EJB Migration**: @Stateless/@Singleton/@Stateful → CDI beans with @ApplicationScoped/@Singleton + @Transactional
- **JPA→Hibernate ORM/Panache**: Entity manager patterns, repository abstraction
- **JMS→SmallRye Reactive Messaging**: Message-driven beans to reactive channels
- **JSF→Qute**: View layer migration with template engine
- **Build System**: Maven/Gradle → Quarkus BOM, WAR→JAR packaging
- **Namespace Migration**: javax→jakarta when applicable
- **Configuration**: JNDI→Config properties, connection pools
- **Conditional Pipeline**: Phase 0 scanner determines migration complexity and feature flags

## Non-Goals

Items requiring separate handling:
- **EJB Remote/IIOP**: Distributed communication patterns
- **JCA Resource Adapters**: Custom connector implementations  
- **EJB 2.x Entity Beans**: Legacy persistence models
- **Proprietary App Server Clustering**: Vendor-specific features
- **CDI Portable Extensions**: Must convert to Quarkus BuildExtension
- **XA Distributed Transactions**: Complex transaction coordination
- **Vaadin UI**: Rich web framework migration

## Constraints and Guardrails

### API & Functional Parity
- Preserve all public class names, method signatures, REST endpoint paths, HTTP methods, request/response formats, and status codes.
- Business logic must remain unchanged — only framework/infrastructure code is modified.
- Transaction boundaries and isolation levels must be preserved.

### Test Integrity
- Do NOT mark tests `@Disabled` or delete tests as a migration shortcut.
- Migrate Arquillian tests to `@QuarkusTest` equivalents preserving all assertions.
- Test count after migration must match original.

### ArC CDI Limitations
- Quarkus ArC is a build-time CDI implementation with restrictions vs full CDI. See `references/arc-limitations.md` for the complete list.
- No Portable Extensions (use Quarkus `@BuildStep`), no `bean-discovery-mode="all"` (annotate all beans explicitly), no `@Specializes`.
- Normal-scoped beans require a no-arg constructor for proxy generation.
- `@Transactional` only applies to calls through the proxy (self-invocation bypasses it).

### Code Quality
- Constructor injection preferred throughout. Exceptions: `@Inject EntityManager` (Quarkus idiom), `@ConfigProperty` on scalar config fields.
- Remove unused imports and dead code.
- Follow Quarkus conventions (application.properties, JAX-RS resources as CDI beans).

### Incremental Migration
- Verify build passes after each phase before proceeding. Commit after each successful phase.

## Worked Examples

### Example 1: EJB to CDI Bean Migration

**Before (JavaEE):**
```java
@Stateless
public class OrderService {
    @PersistenceContext
    private EntityManager em;
    
    public Order createOrder(OrderRequest request) {
        Order order = new Order(request);
        em.persist(order);
        return order;
    }
}
```

**After (Quarkus):**
```java
@ApplicationScoped
public class OrderService {
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        Order order = new Order(request);
        order.persist();
        return order;
    }
}
```

### Example 2: JNDI to Configuration Properties

**Before (JavaEE):**
```java
@Stateless
public class DatabaseService {
    private DataSource dataSource;
    
    @PostConstruct
    public void init() {
        try {
            Context ctx = new InitialContext();
            dataSource = (DataSource) ctx.lookup("java:jboss/datasources/MyDS");
        } catch (NamingException e) {
            throw new RuntimeException(e);
        }
    }
}
```

**After (Quarkus):**
```java
@ApplicationScoped
public class DatabaseService {
    
    @Inject
    AgroalDataSource dataSource;
    
    @ConfigProperty(name = "quarkus.datasource.jdbc.url")
    String jdbcUrl;
}
```

## Workflow

### Phase 0: Project Analysis (ALWAYS)
- Scan project structure and dependencies
- Detect JavaEE/JakartaEE specifications in use
- Set feature flags for conditional pipeline execution
- Determine migration complexity score
- Generate transformation roadmap

### Phase 1: Build System & Namespace (ALWAYS)

Exit: `./mvnw clean compile -Dmaven.test.skip=true` passes.

**Step 1** (if IS_MULTI_MODULE): Consolidate multi-module (EJB+WAR+EAR) into single Quarkus module. Merge all `src/main/java` trees into one module. Move resources from EJB/WAR submodules into the consolidated `src/main/resources`. After consolidation: (a) verify only ONE `src/main/java` tree exists — dual source trees cause edits to the wrong tree to have zero effect; (b) remove stale legacy module directories (ear/, ejb/, war/ submodules); (c) collapse the parent POM `<modules>` section — the final POM must NOT declare submodules; (d) delete any `build-helper-maven-plugin` executions that referenced old module paths.

- **Old module directories remain on disk** after consolidation (e.g., daytrader-ee7-ejb/, daytrader-ee7-web/, daytrader-ee7/) — they are NOT deleted and serve as reference material. Ensure `settings.gradle` (Gradle) or root `pom.xml` does NOT include them as submodules, so they are excluded from the build.
- **For Gradle projects**: if no gradlew wrapper exists, download `gradle-wrapper.jar` from CDN and create a minimal `gradlew` shell script. Set the distribution URL in `gradle/wrapper/gradle-wrapper.properties` to Gradle 8.x.

Example — before consolidation:
```
my-app/
├── my-app-ear/pom.xml
├── my-app-ejb/src/main/java/...
├── my-app-war/src/main/java/...
└── pom.xml (parent with <modules>)
```

After consolidation:
```
my-app/
├── src/main/java/...          ← merged from ejb + war
├── src/main/resources/...     ← merged from ejb + war
└── pom.xml                    ← single module, no <modules> section
```

**Step 2**: Create Quarkus project foundation.

- Replace parent POM or add Quarkus BOM via `<dependencyManagement>`:
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.quarkus.platform</groupId>
            <artifactId>quarkus-bom</artifactId>
            <version>3.33.x</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

- Change packaging from WAR/EAR to JAR:
```xml
<packaging>jar</packaging>
```

- Add `quarkus-maven-plugin` with build goal:
```xml
<build>
    <plugins>
        <plugin>
            <groupId>io.quarkus.platform</groupId>
            <artifactId>quarkus-maven-plugin</artifactId>
            <version>3.33.x</version>
            <extensions>true</extensions>
            <executions>
                <execution>
                    <goals>
                        <goal>build</goal>
                        <goal>generate-code</goal>
                        <goal>generate-code-tests</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

- Add core extensions based on Phase 0 flags:
```xml
<!-- ALWAYS required — Quarkus CDI (ArC) -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-arc</artifactId>
</dependency>

<!-- if JAXRS_PRESENT -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-rest</artifactId>
</dependency>

<!-- if JPA_NEEDED -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-hibernate-orm</artifactId>
</dependency>
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-jdbc-postgresql</artifactId> <!-- or quarkus-jdbc-mysql, quarkus-jdbc-h2 -->
</dependency>

<!-- if HAS_BEAN_VALIDATION -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-hibernate-validator</artifactId>
</dependency>

<!-- if JSF_NEEDED (MyFaces support) -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-undertow</artifactId>
</dependency>
```

- Remove application server dependencies:
```xml
<!-- DELETE all of these -->
<dependency><groupId>javax</groupId><artifactId>javaee-api</artifactId></dependency>
<dependency><groupId>javax</groupId><artifactId>javaee-web-api</artifactId></dependency>
<dependency><groupId>org.jboss.spec</groupId><artifactId>jboss-javaee-8.0</artifactId></dependency>
<dependency><groupId>org.jboss.spec.javax.ws.rs</groupId><artifactId>*</artifactId></dependency>
<dependency><groupId>org.jboss.spec.javax.ejb</groupId><artifactId>*</artifactId></dependency>
<!-- Also remove any provided-scope JavaEE umbrella deps -->
```

- Remove build plugins that are no longer applicable:
```xml
<!-- DELETE all of these -->
<plugin><artifactId>maven-war-plugin</artifactId></plugin>
<plugin><artifactId>maven-ear-plugin</artifactId></plugin>
<plugin><groupId>org.jboss.as.plugins</groupId><artifactId>jboss-as-maven-plugin</artifactId></plugin>
<plugin><groupId>org.wildfly.plugins</groupId><artifactId>wildfly-maven-plugin</artifactId></plugin>
```

- Add Maven wrapper if not present: `mvn wrapper:wrapper -Dmaven=3.9.6`

**Step 3** (if NAMESPACE_STATUS=JAVAX_ONLY): Migrate javax→jakarta namespace.

Perform project-wide find-and-replace on all `.java` files in `src/`:

| Old Namespace | New Namespace |
|---|---|
| `javax.persistence` | `jakarta.persistence` |
| `javax.inject` | `jakarta.inject` |
| `javax.enterprise` | `jakarta.enterprise` |
| `javax.ws.rs` | `jakarta.ws.rs` |
| `javax.transaction` | `jakarta.transaction` |
| `javax.validation` | `jakarta.validation` |
| `javax.annotation` | `jakarta.annotation` |
| `javax.json` | `jakarta.json` |

**DO NOT replace** these Java SE packages — they are NOT part of the Jakarta migration:
- `javax.crypto` — Java Cryptography Extension
- `javax.net` — Java Networking (SSL sockets)
- `javax.sql` — JDBC extensions (DataSource)
- `javax.security.cert` — Legacy certificate API
- `javax.swing` — Desktop GUI
- `javax.xml.parsers` — DOM/SAX parsing
- `javax.xml.transform` — XSLT

Example transformation:
```java
// BEFORE
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.inject.Inject;
import javax.enterprise.context.ApplicationScoped;
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.crypto.Cipher;  // DO NOT CHANGE

// AFTER
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.inject.Inject;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import javax.crypto.Cipher;  // unchanged — Java SE package
```

**Inline FQCN sweep**: after import migration, run `grep -rn "javax\." src/ | grep -v "^.*import "` to catch fully-qualified class references in method bodies, catch blocks, instanceof checks, and annotations. Replace matching EE namespaces only.

**Step 4**: Migrate configuration.

- Create `src/main/resources/application.properties` (Quarkus default config file):
```properties
# Datasource configuration (from persistence.xml)
quarkus.datasource.db-kind=postgresql
quarkus.datasource.username=${DB_USER:admin}
quarkus.datasource.password=${DB_PASSWORD:admin}
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/mydb
quarkus.datasource.jdbc.max-size=20

# Hibernate ORM (from persistence.xml properties)
quarkus.hibernate-orm.schema-management.strategy=none
quarkus.hibernate-orm.log.sql=false

# HTTP server (from web.xml / jboss-web.xml)
quarkus.http.port=8080
quarkus.http.root-path=/api

# JMS connection config (custom properties for Artemis)
artemis.url=tcp://localhost:61616
artemis.username=admin
artemis.password=admin
```

- **Remove deprecated config keys**: `quarkus.hibernate-orm.database.generation` may appear alongside the new key in migration outputs — remove it entirely, keep only `quarkus.hibernate-orm.schema-management.strategy`.

- **MyFaces configuration**: `org.apache.myfaces.PROJECT_STAGE` is NOT set via `quarkus.myfaces.*` application.properties keys. Set it as a `<context-param>` in `web.xml` (e.g., `jakarta.faces.PROJECT_STAGE=Production`) or leave as default. Remove any `quarkus.myfaces.*` keys from application.properties — they will generate "Unrecognized configuration key" warnings.

- **JMS connection config**: Use custom property names like `artemis.url`, `artemis.username`, `artemis.password` (not `quarkus.artemis.*` which is reserved for the quarkiverse Artemis extension). Reference these in JmsProducer beans via `@ConfigProperty`.

- **Before deleting persistence.xml**: extract all configuration values:
  - Datasource JNDI names → `quarkus.datasource.*` properties
  - Hibernate properties (dialect, ddl-auto, show_sql) → `quarkus.hibernate-orm.*`
  - Multiple persistence units → named datasources (`quarkus.datasource."named".*)
  - If datasource is JNDI (`<jta-data-source>java:jboss/…</jta-data-source>`), locate `*-ds.xml` or `standalone.xml` for actual JDBC URL/driver/credentials

- **Move server config** from deployment descriptors:
  - `<context-root>` in jboss-web.xml → `quarkus.http.root-path`
  - Port configuration → `quarkus.http.port`
  - Session timeout → `quarkus.http.session.max-age`

- **Remove all application server descriptors**:
  - `jboss-web.xml`
  - `jboss-ejb3.xml`
  - `jboss-deployment-structure.xml`
  - `glassfish-web.xml`
  - `sun-web.xml`
  - `ibm-web-ext.xml`
  - `META-INF/persistence.xml` (after extracting config)
  - `META-INF/beans.xml` (Quarkus does not require beans.xml — ArC discovers beans at build time)

- **Remove web.xml** unless HAS_SERVLET_FILTERS=true. If servlet filters exist, migrate filter definitions to Quarkus `@ServerFilter` or JAX-RS `ContainerRequestFilter` before removing web.xml.

**Step 5**: Create application entry point.

- **No main class needed** — Quarkus generates the entry point at build time. Unlike Spring Boot, there is no `public static void main(String[] args)` required.
- Remove any existing `main()` method that bootstraps an embedded/app server (e.g., `EJBContainer.createEJBContainer()`, `Undertow.builder()`).
- Remove all application server runtime config references:
  - `standalone.xml`
  - `domain.xml`
  - `standalone-full.xml`
  - `host.xml`
- Remove `src/main/webapp/WEB-INF/` directory entirely (Quarkus does not use webapp structure).
- If the project has a JAX-RS `Application` subclass with `@ApplicationPath`:
  - **Record the `@ApplicationPath` value** — this becomes `quarkus.rest.path` in application.properties
  - The class itself can remain (Quarkus supports it) or be deleted if configuration is moved:
```properties
# Replaces @ApplicationPath("/api")
quarkus.rest.path=/api
```

**Exit gate**: Run `./mvnw clean compile -Dmaven.test.skip=true` — must succeed with zero errors. Common failures at this stage:
- Missing dependencies → check BOM import is correct
- Package not found → verify namespace migration completed (Step 3)
- Plugin errors → confirm quarkus-maven-plugin version matches BOM version

### Phase 2: Core Migration (CONDITIONAL - EJB detected)

Exit: `./mvnw clean compile` passes.

> ⚠ **MIGRATION COMMENT RULE** — Scope: applies to ALL comment text (Javadoc, inline, block) in MIGRATION-labeled lines. Regular Javadoc describing annotations is NOT subject to this rule — only MIGRATION-labeled comments.
>
> **Rule**: MIGRATION comments must NOT contain `@AnnotationName` tokens. Write plain English instead.
>
> | ❌ Wrong | ✅ Correct |
> |---|---|
> | `// MIGRATION: Removed @Stateless` | `// MIGRATION: Converted stateless session bean to CDI application-scoped bean` |
> | `// MIGRATION: @EJB replaced` | `// MIGRATION: Replaced EJB injection with CDI inject` |
> | `// MIGRATION: @TransactionAttribute removed` | `// MIGRATION: Replaced EJB transaction attribute with Quarkus transactional annotation` |
> | `// MIGRATION: @Schedule converted` | `// MIGRATION: Converted EJB timer to Quarkus scheduled task` |

**Step 6**: Migrate EJBs to CDI beans. See `references/ejb-to-cdi-mapping.md` for complete mapping reference and `references/arc-limitations.md` for ArC build-time CDI restrictions.

- `@Stateless` → `@ApplicationScoped` + `@Transactional` **ONLY when the bean performs persistence operations** (EntityManager calls, repository calls). For pure-computation services with no persistence, omit `@Transactional`. If a class has both `@Stateless` and `@Path`, produce a single `@ApplicationScoped @Path` resource (not separate service + resource) — JAX-RS resources are CDI beans in Quarkus.
- `@Singleton` (javax.ejb/jakarta.ejb) → `@ApplicationScoped`. If `@Startup` is present, add `void onStart(@Observes StartupEvent ev)` method from `io.quarkus.runtime.StartupEvent`. Do NOT use `jakarta.inject.Singleton` — it is not a normal scope and cannot be intercepted by `@Transactional`.
- `@Stateful` → `@SessionScoped`. **WARNING**: evaluate if stateful pattern is still needed — Quarkus is designed for stateless cloud-native patterns. Consider converting to `@ApplicationScoped` with externalized state (Redis, database). If keeping `@SessionScoped`: the bean MUST be `Serializable`, and ArC requires a no-arg constructor (unlike full CDI).
- Remove `@LocalBean`, `@Local`, `@Remote` — no equivalent in CDI. Remote calls require redesign to REST/gRPC.
- `@EJB` → `@Inject`. All injection via `@Inject` — constructor injection preferred throughout.
- **Constructor injection**: Quarkus ArC supports constructor injection with a single constructor (no `@Inject` annotation needed on the constructor). Exceptions to constructor injection rule:
  1. `@PersistenceContext EntityManager` — Quarkus Hibernate ORM injects via `@Inject EntityManager` (field injection acceptable for EntityManager).
  2. `@ConfigProperty` on scalar fields in config holder beans.
- **CDI no-arg constructor requirement**: ArC requires a no-arg constructor (can be package-private) for normal-scoped beans (`@ApplicationScoped`, `@RequestScoped`, `@SessionScoped`). This is a build-time proxy requirement. Add a package-private no-arg constructor when using constructor injection:

```java
// BEFORE (JavaEE EJB)
import javax.ejb.Stateless;
import javax.ejb.EJB;
import javax.ejb.TransactionAttribute;
import javax.ejb.TransactionAttributeType;

@Stateless
public class OrderService {
    @EJB
    private InventoryService inventoryService;

    @EJB
    private PaymentService paymentService;

    @TransactionAttribute(TransactionAttributeType.REQUIRES_NEW)
    public Order placeOrder(OrderRequest request) {
        inventoryService.reserve(request.getItemId(), request.getQuantity());
        paymentService.charge(request.getPaymentInfo());
        return createOrder(request);
    }
}

// AFTER (Quarkus)
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.transaction.Transactional.TxType;

@ApplicationScoped
@Transactional
public class OrderService {
    private final InventoryService inventoryService;
    private final PaymentService paymentService;

    OrderService() {} // package-private no-arg constructor for ArC proxy

    @Inject
    public OrderService(InventoryService inventoryService, PaymentService paymentService) {
        this.inventoryService = inventoryService;
        this.paymentService = paymentService;
    }

    @Transactional(TxType.REQUIRES_NEW)
    public Order placeOrder(OrderRequest request) {
        inventoryService.reserve(request.getItemId(), request.getQuantity());
        paymentService.charge(request.getPaymentInfo());
        return createOrder(request);
    }
}
```

- **@TransactionAttribute mappings**:

| EJB TransactionAttributeType | Quarkus @Transactional TxType |
|---|---|
| `REQUIRED` (default) | `@Transactional` (TxType.REQUIRED is default) |
| `REQUIRES_NEW` | `@Transactional(TxType.REQUIRES_NEW)` |
| `NOT_SUPPORTED` | `@Transactional(TxType.NOT_SUPPORTED)` |
| `SUPPORTS` | `@Transactional(TxType.SUPPORTS)` |
| `MANDATORY` | `@Transactional(TxType.MANDATORY)` |
| `NEVER` | `@Transactional(TxType.NEVER)` |

- Apply `@Transactional` at **method level**, not class level. Remove any class-level `@Transactional` from the migrated service bean. Add it explicitly only to write methods (buy, sell, completeOrder, login, logout, register, createQuote, updateQuotePriceVolume, getClosedOrders). Use `@Transactional(TxType.SUPPORTS)` or no annotation for read-only methods (getQuote, getAllQuotes, getHoldings, getAccountData, getAccountProfileData, getMarketSummary). Keep special overrides: `@Transactional(TxType.NOT_SUPPORTED)` for resetTrade, `@Transactional(TxType.REQUIRES_NEW)` for publishQuotePriceChange.
- Convert non-CDI service façade classes to `@ApplicationScoped` CDI beans. Any class that was a client-side EJB façade instantiated via `new` must itself become CDI-managed. Add `@ApplicationScoped` and inject dependencies via `@Inject`. Using `Arc.container().instance(...).get()` as a JNDI replacement is an anti-pattern that bypasses CDI lifecycle. Detection: `grep -rn "Arc\.container\(\)" src/main/java/`
- Remove static resource caches from migrated CDI beans. Static `DataSource`, `ConnectionFactory`, `Queue`, `Topic` fields initialized in `static init()` methods must be replaced with `@Inject` fields. Static state breaks Quarkus dev mode hot reload. Detection: `grep -rn "private static.*DataSource\|private static.*initialized\b" src/main/java/`
- For async work in CDI beans, replace `Executors.newCachedThreadPool()` with `@Inject org.eclipse.microprofile.context.ManagedExecutor`. Unmanaged thread pools have no graceful shutdown integration with Quarkus.

- **@Asynchronous EJB methods**: Quarkus does not have a direct `@Asynchronous` equivalent. Options:
  - Return `Uni<T>` (Mutiny) or `CompletionStage<T>` — Quarkus reactive model handles non-blocking execution natively.
  - For fire-and-forget: use `@ActivateRequestContext` + `vertx.executeBlocking()` or SmallRye `@Blocking` on a reactive messaging channel.
  - **CRITICAL**: if the async method accesses `@RequestScoped` beans, annotate with `@ActivateRequestContext` — ArC does not propagate request context to async threads automatically.

```java
// BEFORE (JavaEE @Asynchronous)
@Stateless
public class NotificationService {
    @Asynchronous
    public Future<Void> sendEmail(String to, String body) {
        // long-running email send
        return new AsyncResult<>(null);
    }
}

// AFTER (Quarkus — Mutiny reactive)
@ApplicationScoped
public class NotificationService {
    public Uni<Void> sendEmail(String to, String body) {
        return Uni.createFrom().item(() -> {
            // long-running email send
            return null;
        }).runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }
}
```

- **@Singleton with @Startup** → `@ApplicationScoped` + startup observer:

```java
// BEFORE (JavaEE)
@Singleton
@Startup
public class CacheWarmer {
    @PostConstruct
    public void init() {
        warmCaches();
    }
}

// AFTER (Quarkus)
@ApplicationScoped
public class CacheWarmer {
    void onStart(@Observes StartupEvent ev) {
        warmCaches();
    }
}
```

- **EJB lifecycle annotations**: `@PostActivate` / `@PrePassivate` — DELETE the annotation AND the method body (no Quarkus equivalent for stateful passivation). `@PostConstruct` / `@PreDestroy` (JSR-250) — RETAIN as-is (Quarkus CDI supports them natively).
- **EJB exception handling**: `@ApplicationException(rollback=true)` → catch the exception in a `@Transactional` method and call `TransactionManager.setRollbackOnly()`, or throw a `RuntimeException` (triggers automatic rollback). For non-rollback app exceptions, simply remove the annotation — Quarkus `@Transactional` only rolls back on `RuntimeException` by default.
- CDI `@Produces` Logger → `private static final Logger logger = Logger.getLogger(ClassName.class)` (JBoss Logging) or `LoggerFactory.getLogger()` (SLF4J with `quarkus-logging-slf4j` extension). Delete LoggerProducer classes.
- CDI scopes: `@RequestScoped` stays as-is. `@ApplicationScoped` stays. `@Model` → `@ApplicationScoped` + appropriate JAX-RS annotations.
- CDI events: `Event<T>.fire()` and `Event<T>.fireAsync()` work unchanged in Quarkus ArC. `@Observes` / `@ObservesAsync` work as-is. **ArC restriction**: `@ObservesAsync` observers run on the event loop — do NOT perform blocking I/O without `@Blocking`.
- CDI interceptors: `@Interceptor` + `@InterceptorBinding` work in Quarkus ArC. Add the interceptor binding annotation to the interceptor class (ArC does not read beans.xml `<interceptors>` — use `@Priority` for ordering).
- CDI `@Alternative` → `@Alternative` + `@Priority` (ArC does not read beans.xml `<alternatives>` section). Or use build-time profile-based selection: `@IfBuildProfile("dev")` / `@UnlessBuildProfile("prod")`.
- Scan all `package-info.java` for `@Vetoed` — retain it (ArC supports `@Vetoed`).
- **Remove `Serializable` from CDI beans**: `Serializable` was required for EJB passivation but is unnecessary in Quarkus — ArC proxies do not require it. Scan: `grep -rn 'implements Serializable' src/main/java/` and remove from CDI beans.
- **Remove subclasses of `jakarta.ws.rs.core.Application`** (the `@ApplicationPath` class): In Quarkus, set the REST base path via `quarkus.rest.path=/path` in application.properties. The Application subclass is a Jakarta EE pattern that is unnecessary in Quarkus.
- **javax→jakarta completeness**: when migrating a file, ALL `javax.*` EE namespaces in that file must be updated in the same pass. Mixed-namespace files cause compilation failures.
- Remove all `javax.ejb.*` / `jakarta.ejb.*` imports after migration — no EJB APIs should remain.

**Step 7**: Migrate JPA configuration.

- **persistence.xml removal**: configuration was already moved to `application.properties` in Phase 1 Step 4. Verify the file is deleted. If it still exists, extract any remaining properties and delete it.
- Keep entity annotations as-is — `jakarta.persistence.*` annotations (after namespace migration in Step 3) work unchanged with `quarkus-hibernate-orm`.
- **EntityManager injection pattern change**:

```java
// BEFORE (JavaEE)
@PersistenceContext
private EntityManager em;

// AFTER (Quarkus) — @Inject replaces @PersistenceContext
@Inject
EntityManager em;
```

- If HAS_HIBERNATE_SPECIFIC: Hibernate-specific annotations (`@Type`, `@GenericGenerator`, `@Filter`, `@SQLDelete`, `@Where`, `@Formula`) work unchanged with `quarkus-hibernate-orm` (it bundles Hibernate ORM 6.x). **Exception**: `@Type(type="org.hibernate.type.TextType")` → remove (Hibernate 6 uses `@JdbcTypeCode(Types.LONGVARCHAR)` or `@Column(columnDefinition="TEXT")`).
- **Panache opt-in** (OPTIONAL — do NOT force conversion): if the project wants to adopt Panache active-record or repository patterns, add `quarkus-hibernate-orm-panache`:
```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-hibernate-orm-panache</artifactId>
</dependency>
```
  - Active record: Entity `extends PanacheEntity` (uses auto-generated Long id) or `extends PanacheEntityBase` (custom id).
  - Repository: Create `@ApplicationScoped class OrderRepository implements PanacheRepository<Order>`.
  - **Do NOT force Panache** on existing codebases — it changes the programming model significantly. Only adopt if explicitly requested or for greenfield entities.

- **Multi-datasource** (if HAS_MULTI_DATASOURCE): use named datasources in `application.properties` and `@PersistenceUnit("name")` qualifier:
```properties
# Default datasource
quarkus.datasource.db-kind=postgresql
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/maindb

# Named datasource "inventory"
quarkus.datasource."inventory".db-kind=postgresql
quarkus.datasource."inventory".jdbc.url=jdbc:postgresql://localhost:5432/inventorydb

# Hibernate ORM for named persistence unit
quarkus.hibernate-orm."inventory".datasource=inventory
quarkus.hibernate-orm."inventory".packages=com.example.inventory.model
```

```java
// Inject named EntityManager
@Inject
@PersistenceUnit("inventory")
EntityManager inventoryEm;
```

- **After migration, re-enable ArC unused bean removal**: ensure `quarkus.arc.remove-unused-beans=false` is NOT present in application.properties (or remove it). Add `@io.quarkus.arc.Unremovable` to beans that are used only via CDI event observers or programmatic lookup (e.g., LoggerProducer, CDI event handler beans). Disabling bean removal is a migration crutch that hurts startup performance.

- **Validation annotation mapping** (same rules as namespace migration but verify completeness):
  - `@NotEmpty` → `@NotBlank` **for String/CharSequence fields ONLY**; for Collection/array/Map fields use `jakarta.validation.constraints.@NotEmpty`.
  - `@Length` (Hibernate Validator) → `@Size` (jakarta.validation).
  - `@URL` (Hibernate Validator) → `@Pattern(regexp="^(https?|ftp)://.*")`.
  - `@SafeHtml` → remove (no equivalent; add custom ConstraintValidator if needed).

**Step 8**: Adjust JAX-RS resources (minimal changes — Quarkus is JAX-RS native via RESTEasy Reactive).

- `@Path`, `@GET`, `@POST`, `@PUT`, `@DELETE`, `@PathParam`, `@QueryParam`, `@HeaderParam` — **stay unchanged**. Quarkus RESTEasy Reactive is fully JAX-RS compatible.
- **Replace JAX-RS `ClientBuilder` with MicroProfile type-safe REST client**: Any class using `ClientBuilder.newClient()` should be migrated to `@RegisterRestClient` interface + `@RestClient` injection. This provides connection pooling, timeout configuration, and fault tolerance integration. Update application.properties with `quarkus.rest-client.<name>.url=...`
- **Remove the JAX-RS `Application` subclass** (`extends Application` or `extends javax.ws.rs.core.Application`). Quarkus auto-discovers all `@Path`-annotated classes without an Application class.
  - If `@ApplicationPath("/api")` exists → set `quarkus.rest.path=/api` in `application.properties` (already done in Phase 1 Step 5). Delete the class.
  - If `@ApplicationPath("")` or `@ApplicationPath("/")` → no `quarkus.rest.path` needed. Delete the class.
- `Response` / `Response.ok()` / `Response.status()` patterns remain valid — no changes needed.
- JAX-RS `@Provider`-annotated classes (`ExceptionMapper`, `MessageBodyReader/Writer`, `ContainerRequestFilter`, `ContainerResponseFilter`) — **work unchanged** in Quarkus. ArC discovers `@Provider` beans automatically.
- **Quarkus-specific enhancements** (OPTIONAL — do NOT force): if migrating to RESTEasy Reactive idioms:
  - Return types can be `Uni<T>` for non-blocking (requires `quarkus-rest` extension, already added in Step 2).
  - `@Blocking` annotation on resource methods that perform blocking I/O (JDBC, file I/O) when using the reactive REST extension.
  - `@ServerExceptionMapper` can replace `@Provider ExceptionMapper<T>` classes (simpler syntax).

```java
// JAX-RS resource — minimal changes needed for Quarkus
@Path("/orders")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class OrderResource {

    @Inject
    OrderService orderService;  // was @EJB, now @Inject

    @GET
    @Path("/{id}")
    public Response getOrder(@PathParam("id") Long id) {
        Order order = orderService.findById(id);
        if (order == null) {
            return Response.status(Response.Status.NOT_FOUND).build();
        }
        return Response.ok(order).build();
    }
}
```

- **@Consumes/@Produces**: remain unchanged. Quarkus defaults to JSON serialization via `quarkus-rest-jackson` (included transitively by `quarkus-rest`). If the project uses JSON-B instead: add `quarkus-rest-jsonb` and remove `quarkus-rest-jackson`.
- **Multipart handling**: if using `@Consumes(MediaType.MULTIPART_FORM_DATA)` with `org.jboss.resteasy.plugins.providers.multipart.*` → migrate to `@RestForm` / `@MultipartForm` from `org.jboss.resteasy.reactive`.

**Step 9** (if SCHEDULER_NEEDED): Migrate EJB Timers to Quarkus Scheduler.

- Add the scheduler extension (if not already present):
```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-scheduler</artifactId>
</dependency>
```

- **@Schedule (EJB)** → `@Scheduled` (Quarkus):

```java
// BEFORE (JavaEE EJB Timer)
import javax.ejb.Schedule;
import javax.ejb.Singleton;

@Singleton
public class ReportGenerator {
    @Schedule(hour = "*", minute = "*/5", persistent = false)
    public void generateReport() {
        // runs every 5 minutes
    }
}

// AFTER (Quarkus)
import io.quarkus.scheduler.Scheduled;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class ReportGenerator {
    @Scheduled(every = "5m")
    public void generateReport() {
        // runs every 5 minutes
    }
}
```

- **Cron expression mapping**: Quarkus uses Quartz-style cron (`second minute hour dayOfMonth month dayOfWeek`) or duration expressions (`every = "2h"`, `every = "30s"`).

| EJB @Schedule | Quarkus @Scheduled |
|---|---|
| `@Schedule(hour="*", minute="*/5")` | `@Scheduled(every="5m")` |
| `@Schedule(hour="2", minute="0")` | `@Scheduled(cron="0 0 2 * * ?")` |
| `@Schedule(dayOfWeek="Mon", hour="8")` | `@Scheduled(cron="0 0 8 ? * MON")` |

- **TimerService programmatic timers** → Quarkus `Scheduler` API:

```java
// BEFORE (JavaEE)
@Resource
TimerService timerService;

public void startTimer(long interval) {
    timerService.createIntervalTimer(0, interval, new TimerConfig());
}

// AFTER (Quarkus)
@Inject
Scheduler scheduler;

public void startTimer(long interval) {
    scheduler.newJob("dynamic-job")
        .setInterval(Duration.ofMillis(interval).toString())
        .setTask(ctx -> executeTask())
        .schedule();
}
```

- `@Timeout` methods → `@Scheduled` with the appropriate cron/interval expression. Drop `persistent=false` (Quarkus scheduled tasks are non-persistent by default).
- **@Schedules (plural)**: split into separate `@Scheduled` methods, one per schedule expression, each delegating to a shared private method.

**Step 10** (if WEBSOCKET_NEEDED): Migrate WebSockets.

- Add the WebSocket extension:
```xml
<!-- Option A: Classic JSR 356 compatibility -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-websockets</artifactId>
</dependency>

<!-- Option B: Quarkus-native WebSocket API (recommended for new code) -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-websockets-next</artifactId>
</dependency>
```

- **If using `quarkus-websockets`** (JSR 356 compatibility): `@ServerEndpoint`, `@OnOpen`, `@OnMessage`, `@OnClose`, `@OnError` annotations work unchanged. Minimal migration needed.
- **If migrating to `quarkus-websockets-next`** (recommended for new development):
  - `@ServerEndpoint("/chat")` → `@WebSocket(path = "/chat")`
  - `@OnMessage` → `@OnTextMessage` / `@OnBinaryMessage`
  - Session management via `WebSocketConnection` injection instead of `javax.websocket.Session`

**Step 11** (if HAS_JNDI): Replace JNDI lookups.

- `InitialContext.lookup("java:comp/env/...")` → `@Inject` + `@ConfigProperty`:

```java
// BEFORE (JavaEE JNDI)
@Stateless
public class ConfigService {
    @Resource(lookup = "java:comp/env/app/maxRetries")
    private int maxRetries;

    @Resource(lookup = "java:jboss/datasources/ReportDS")
    private DataSource reportDs;
}

// AFTER (Quarkus)
@ApplicationScoped
public class ConfigService {
    @ConfigProperty(name = "app.max-retries", defaultValue = "3")
    int maxRetries;

    @Inject
    @io.quarkus.agroal.DataSource("report")
    AgroalDataSource reportDs;
}
```

```properties
# application.properties
app.max-retries=3

# Named datasource for "report"
quarkus.datasource."report".db-kind=postgresql
quarkus.datasource."report".jdbc.url=jdbc:postgresql://localhost:5432/reportdb
```

- `@Resource DataSource` → `@Inject AgroalDataSource` (default datasource) or `@Inject @DataSource("name") AgroalDataSource` (named datasource).
- `@Resource(lookup="java:comp/DefaultJMSConnectionFactory")` → configure in `application.properties` and inject via SmallRye Reactive Messaging (Phase 3).
- **Programmatic JNDI**: replace all `new InitialContext()` / `context.lookup()` calls with `@Inject` + `@ConfigProperty`. If lookup keys are dynamic, use `ConfigProvider.getConfig().getValue(key, String.class)` (MicroProfile Config programmatic API).
- Environment entries from `web.xml` `<env-entry>`:
```xml
<!-- BEFORE in web.xml -->
<env-entry>
    <env-entry-name>app/apiUrl</env-entry-name>
    <env-entry-type>java.lang.String</env-entry-type>
    <env-entry-value>https://api.example.com</env-entry-value>
</env-entry>
```
```properties
# AFTER in application.properties
app.api-url=https://api.example.com
```
```java
@ConfigProperty(name = "app.api-url")
String apiUrl;
```

**Exit gate**: Run `./mvnw clean compile` — must succeed with zero errors. Common failures at this stage:
- `UnsatisfiedResolutionException` → missing `@ApplicationScoped` on a bean, or ArC cannot find a producer
- `AmbiguousResolutionException` → multiple beans of same type without qualifiers; add `@Default` or named qualifiers
- `DeploymentException: no-arg constructor` → normal-scoped beans need a no-arg constructor for ArC proxy generation
- EJB imports remaining → verify all `javax.ejb.*` / `jakarta.ejb.*` are removed

### Phase 3: Conditional Services (CONDITIONAL — SECURITY_NEEDED || JMS_NEEDED || MDB_NEEDED || SOAP_NEEDED)

SKIP if SECURITY_NEEDED=false AND JMS_NEEDED=false AND MDB_NEEDED=false AND SOAP_NEEDED=false. Exit: `./mvnw clean compile` passes.

**Step 12** (if SECURITY_NEEDED): Migrate security. See `references/security-to-quarkus-security.md` for detailed mapping.

- Add the Quarkus security extension + appropriate identity provider:
```xml
<!-- Core security (always needed) -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-security</artifactId>
</dependency>

<!-- Choose ONE identity provider based on source app's auth mechanism: -->
<!-- JDBC/Database auth -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-elytron-security-jdbc</artifactId>
</dependency>

<!-- OIDC/OAuth2 (Keycloak, Auth0, etc.) -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-oidc</artifactId>
</dependency>

<!-- LDAP -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-elytron-security-ldap</artifactId>
</dependency>

<!-- Properties file (dev/test only) -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-elytron-security-properties-file</artifactId>
</dependency>
```

- **`@RolesAllowed`** → stays unchanged. Quarkus supports `jakarta.annotation.security.@RolesAllowed` natively on JAX-RS resources and CDI beans. No migration needed.
- **`@DeclareRoles`** → DELETE. Quarkus does not require role pre-declaration — roles are discovered from the identity provider at runtime.
- **`@PermitAll` / `@DenyAll`** → stay unchanged (jakarta.annotation.security).
- **`@RunAs`** → replace with `SecurityIdentity` augmentation. Implement a `SecurityIdentityAugmentor` to transform roles/principals:

```java
// BEFORE (JavaEE)
@Stateless
@RunAs("SYSTEM")
public class BatchProcessor {
    // executes with SYSTEM role
}

// AFTER (Quarkus) — programmatic identity augmentation
@ApplicationScoped
public class BatchProcessor {
    @Inject
    SecurityIdentity identity;

    // For elevated operations, use QuarkusSecurity.user() in tests
    // or implement SecurityIdentityAugmentor for runtime augmentation
}
```

- **web.xml `<security-constraint>`** → `quarkus.http.auth.policy.*` in application.properties:

```xml
<!-- BEFORE in web.xml -->
<security-constraint>
    <web-resource-collection>
        <url-pattern>/admin/*</url-pattern>
    </web-resource-collection>
    <auth-constraint>
        <role-name>ADMIN</role-name>
    </auth-constraint>
</security-constraint>
<security-constraint>
    <web-resource-collection>
        <url-pattern>/api/*</url-pattern>
    </web-resource-collection>
    <auth-constraint>
        <role-name>USER</role-name>
        <role-name>ADMIN</role-name>
    </auth-constraint>
</security-constraint>
```

```properties
# AFTER in application.properties
quarkus.http.auth.policy.admin-policy.roles-allowed=ADMIN
quarkus.http.auth.permission.admin-perm.paths=/admin/*
quarkus.http.auth.permission.admin-perm.policy=admin-policy

quarkus.http.auth.policy.user-policy.roles-allowed=USER,ADMIN
quarkus.http.auth.permission.api-perm.paths=/api/*
quarkus.http.auth.permission.api-perm.policy=user-policy
```

- **JAAS `LoginModule`** → implement `io.quarkus.security.identity.IdentityProvider<T>`. The `authenticate()` method replaces `login()/commit()`:

```java
// AFTER (Quarkus custom identity provider)
@ApplicationScoped
public class CustomIdentityProvider implements IdentityProvider<UsernamePasswordAuthenticationRequest> {

    @Override
    public Class<UsernamePasswordAuthenticationRequest> getRequestType() {
        return UsernamePasswordAuthenticationRequest.class;
    }

    @Override
    public Uni<SecurityIdentity> authenticate(
            UsernamePasswordAuthenticationRequest request,
            AuthenticationRequestContext context) {
        // Replace JAAS LoginModule logic here
        String username = request.getUsername();
        String password = new String(request.getPassword().getPassword());
        // validate and build identity...
        return Uni.createFrom().item(
            QuarkusSecurityIdentity.builder()
                .setPrincipal(new QuarkusPrincipal(username))
                .addRole("USER")
                .build()
        );
    }
}
```

- **`EJBContext.isCallerInRole()` / `SessionContext.getCallerPrincipal()`** → inject `SecurityContext`:

```java
// BEFORE (JavaEE)
@Resource
SessionContext ctx;

public boolean isAdmin() {
    return ctx.isCallerInRole("ADMIN");
}

public String getCurrentUser() {
    return ctx.getCallerPrincipal().getName();
}

// AFTER (Quarkus)
@Inject
SecurityContext securityContext;

public boolean isAdmin() {
    return securityContext.isUserInRole("ADMIN");
}

public String getCurrentUser() {
    return securityContext.getUserPrincipal().getName();
}
```

- **Form-based auth** (if `<login-config><auth-method>FORM</auth-method>` in web.xml):
```properties
quarkus.http.auth.form.enabled=true
quarkus.http.auth.form.login-page=/login.html
quarkus.http.auth.form.error-page=/error.html
quarkus.http.auth.form.landing-page=/index.html
```

- **Basic auth**:
```properties
quarkus.http.auth.basic=true
```

**Step 13** (if JMS_NEEDED or MDB_NEEDED): Migrate messaging. See `references/jms-to-smallrye.md` for detailed patterns.

**Option A — Keep JMS API** (minimal change, same programming model):

- Add `quarkus-artemis-jms` extension:
```xml
<dependency>
    <groupId>io.quarkiverse.artemis</groupId>
    <artifactId>quarkus-artemis-jms</artifactId>
</dependency>
```

- Configure broker connection in `application.properties`:
```properties
quarkus.artemis.url=tcp://localhost:61616
quarkus.artemis.username=admin
quarkus.artemis.password=admin
```

- **`@MessageDriven` + `MessageListener.onMessage()`** → `@ApplicationScoped` bean with JMS consumer:

```java
// BEFORE (JavaEE MDB)
@MessageDriven(activationConfig = {
    @ActivationConfigProperty(propertyName = "destinationType", propertyValue = "javax.jms.Queue"),
    @ActivationConfigProperty(propertyName = "destination", propertyValue = "java:/jms/queue/OrderQueue")
})
public class OrderMessageHandler implements MessageListener {
    @EJB
    private OrderService orderService;

    @Override
    public void onMessage(Message message) {
        TextMessage textMsg = (TextMessage) message;
        orderService.processOrder(textMsg.getText());
    }
}

// AFTER (Quarkus — JMS API preserved)
@ApplicationScoped
public class OrderMessageHandler {
    @Inject
    OrderService orderService;

    @Inject
    ConnectionFactory connectionFactory;

    // Use Quarkus scheduler or startup event to poll, OR use SmallRye option below
    void onStart(@Observes StartupEvent ev) {
        // Start JMS consumer — consider Option B for cleaner reactive approach
    }
}
```

- **ConnectionFactory** → inject directly via `@Inject ConnectionFactory` (Quarkus Artemis extension provides it).
- **Queue/Topic JNDI lookup** → use `@ConfigProperty` for destination names:
```properties
app.jms.order-queue=OrderQueue
app.jms.notification-topic=NotificationTopic
```

**Option B — Modernize to SmallRye Reactive Messaging** (recommended for greenfield/cloud-native):

- Add reactive messaging extension (choose broker):
```xml
<!-- For AMQP/Artemis -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-messaging-amqp</artifactId>
</dependency>

<!-- OR for Kafka -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-messaging-kafka</artifactId>
</dependency>
```

- **`@MessageDriven`** → `@ApplicationScoped` + `@Incoming("channel-name")`:

```java
// BEFORE (JavaEE MDB)
@MessageDriven(activationConfig = {
    @ActivationConfigProperty(propertyName = "destinationType", propertyValue = "javax.jms.Queue"),
    @ActivationConfigProperty(propertyName = "destination", propertyValue = "java:/jms/queue/OrderQueue")
})
public class OrderMessageHandler implements MessageListener {
    @EJB
    private OrderService orderService;

    @Override
    public void onMessage(Message message) {
        TextMessage textMsg = (TextMessage) message;
        orderService.processOrder(textMsg.getText());
    }
}

// AFTER (Quarkus — SmallRye Reactive Messaging)
@ApplicationScoped
public class OrderMessageHandler {
    @Inject
    OrderService orderService;

    @Incoming("orders-in")
    public void onMessage(String orderPayload) {
        orderService.processOrder(orderPayload);
    }
}
```

- **JMS producer** → `@Outgoing("channel-name")` or inject `Emitter<T>`:

```java
// BEFORE (JavaEE JMS Producer)
@Stateless
public class OrderNotifier {
    @Resource(lookup = "java:/jms/queue/NotificationQueue")
    private Queue queue;

    @Inject
    private JMSContext jmsContext;

    public void notifyOrderCreated(String orderId) {
        jmsContext.createProducer().send(queue, orderId);
    }
}

// AFTER (Quarkus — Emitter)
@ApplicationScoped
public class OrderNotifier {
    @Inject
    @Channel("notifications-out")
    Emitter<String> notificationEmitter;

    public void notifyOrderCreated(String orderId) {
        notificationEmitter.send(orderId);
    }
}
```

- Configure channels in `application.properties`:
```properties
# Incoming channel (consumer)
mp.messaging.incoming.orders-in.connector=smallrye-amqp
mp.messaging.incoming.orders-in.address=OrderQueue
mp.messaging.incoming.orders-in.durable=true

# Outgoing channel (producer)
mp.messaging.outgoing.notifications-out.connector=smallrye-amqp
mp.messaging.outgoing.notifications-out.address=NotificationQueue
```

- **Transacted message consumption**: SmallRye Reactive Messaging handles acknowledgment via `@Acknowledgment(Strategy.POST_PROCESSING)` (default). For manual ack: accept `Message<T>` parameter and call `message.ack()`.
- **Message selectors**: not directly supported in SmallRye — implement filtering logic in the consumer method or use separate channels per message type.
- **Dead-letter queue**: configure via broker-specific properties (`dead-letter-queue`, `failure-strategy=dead-letter-queue`).

**Step 14** (if SOAP_NEEDED): Migrate SOAP web services.

**Option A — Keep SOAP** (minimal change with Apache CXF):

- Add `quarkus-cxf` extension:
```xml
<dependency>
    <groupId>io.quarkiverse.cxf</groupId>
    <artifactId>quarkus-cxf</artifactId>
</dependency>
```

- **`@WebService` annotations stay unchanged** — `@WebService`, `@WebMethod`, `@WebParam`, `@WebResult` work as-is with Quarkus CXF.
- Configure the endpoint path in `application.properties`:
```properties
# Endpoint configuration
quarkus.cxf.endpoint."/OrderService".implementor=com.example.service.OrderWebService
quarkus.cxf.endpoint."/OrderService".features=org.apache.cxf.ext.logging.LoggingFeature

# WSDL exposure
quarkus.cxf.endpoint."/OrderService".published-endpoint-url=https://api.example.com/ws/OrderService
```

- **SOAP client** (`@WebServiceRef`): replace with CXF client injection:
```java
// BEFORE (JavaEE)
@WebServiceRef(wsdlLocation = "http://partner.example.com/service?wsdl")
private PartnerService partnerService;

// AFTER (Quarkus CXF)
@Inject
@CXFClient("partner")
PartnerService partnerService;
```
```properties
quarkus.cxf.client."partner".wsdl=http://partner.example.com/service?wsdl
quarkus.cxf.client."partner".client-endpoint-url=http://partner.example.com/service
```

- **Handler chains** (`@HandlerChain`): configure via `quarkus.cxf.endpoint."/path".handlers` property or retain `@HandlerChain` annotation (CXF supports it).

**Option B — Convert SOAP to REST** (larger effort — evaluate ROI):

- **Threshold**: if the `@WebService` has >5 operations or complex XSD types with deep inheritance, flag for human review rather than automated conversion.
- For simple services: map WSDL operations → JAX-RS `@Path` endpoints, convert complex types to JSON POJOs, replace `SOAPFault` with JAX-RS `ExceptionMapper`.
- **WARNING**: changing from SOAP to REST breaks existing clients. Only proceed if all consumers can be updated, or expose both temporarily during migration.

```java
// BEFORE (SOAP)
@WebService
public class OrderWebService {
    @WebMethod
    public OrderResponse createOrder(@WebParam(name="request") OrderRequest request) {
        // business logic
    }
}

// AFTER (REST — only if clients can migrate)
@Path("/orders")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class OrderResource {
    @POST
    public Response createOrder(OrderRequest request) {
        // same business logic
    }
}
```

**Exit gate**: Run `./mvnw clean compile` — must succeed with zero errors. Additional verification:
- If JMS/MDB: verify message consumer/producer beans are CDI-discovered — check Quarkus build output for `SmallRye Reactive Messaging` or `Artemis JMS` initialization logs at startup
- If SECURITY: verify `@RolesAllowed`-annotated methods compile without additional imports
- If SOAP: verify CXF endpoint is generated — check for `quarkus-cxf` build step in compile output

### Phase 4: Testing & UI (ALWAYS — scope varies by flags)

Exit: `./mvnw clean test` passes. All migrated tests execute and pass.

**Step 15**: Migrate test framework. See `references/arquillian-to-quarkustest.md` for complete mapping.

- **Remove Arquillian dependencies** from pom.xml:
```xml
<!-- DELETE all of these -->
<dependency><groupId>org.jboss.arquillian</groupId><artifactId>arquillian-bom</artifactId></dependency>
<dependency><groupId>org.jboss.arquillian.junit</groupId><artifactId>arquillian-junit-container</artifactId></dependency>
<dependency><groupId>org.jboss.arquillian.container</groupId><artifactId>*</artifactId></dependency>
<dependency><groupId>org.jboss.shrinkwrap</groupId><artifactId>*</artifactId></dependency>
<dependency><groupId>org.jboss.shrinkwrap.resolver</groupId><artifactId>*</artifactId></dependency>
<!-- Also remove arquillian-related managed dependencies from <dependencyManagement> -->
```

- **Add Quarkus test dependencies**:
```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-junit5</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <scope>test</scope>
</dependency>
<!-- If mocking CDI beans -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-junit5-mockito</artifactId>
    <scope>test</scope>
</dependency>
<!-- If testing with real database -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-test-h2</artifactId>
    <scope>test</scope>
</dependency>
```

- **`@RunWith(Arquillian.class)`** → `@QuarkusTest`:

```java
// BEFORE (Arquillian)
import org.jboss.arquillian.junit.Arquillian;
import org.jboss.arquillian.container.test.api.Deployment;
import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.WebArchive;
import org.junit.runner.RunWith;

@RunWith(Arquillian.class)
public class OrderServiceTest {

    @Deployment
    public static WebArchive createDeployment() {
        return ShrinkWrap.create(WebArchive.class)
            .addClasses(OrderService.class, Order.class)
            .addAsResource("META-INF/persistence.xml");
    }

    @Inject
    private OrderService orderService;

    @Test
    public void testCreateOrder() {
        Order order = orderService.createOrder(new OrderRequest("item-1", 2));
        assertNotNull(order);
        assertNotNull(order.getId());
    }
}

// AFTER (Quarkus)
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
public class OrderServiceTest {

    // No @Deployment needed — Quarkus manages the full app lifecycle

    @Inject
    OrderService orderService;

    @Test
    public void testCreateOrder() {
        Order order = orderService.createOrder(new OrderRequest("item-1", 2));
        assertNotNull(order);
        assertNotNull(order.getId());
    }
}
```

- **`@Deployment` ShrinkWrap archive** → DELETE entirely. Quarkus boots the full application for `@QuarkusTest` — no deployment descriptor needed. The test classpath IS the deployment.
- **`@Inject` in tests** → works natively with `@QuarkusTest`. No special injection setup needed.
- **`@ArquillianResource URL`** → `@TestHTTPResource`:

```java
// BEFORE
@ArquillianResource
private URL deploymentUrl;

// AFTER
@TestHTTPResource
URL deploymentUrl;

// Or use REST Assured (preferred — no URL management):
@TestHTTPEndpoint(OrderResource.class)
@TestHTTPResource
URL orderEndpoint;
```

- **Arquillian REST client tests** → REST Assured:

```java
// BEFORE (Arquillian + JAX-RS client)
Client client = ClientBuilder.newClient();
Response response = client.target(deploymentUrl.toString())
    .path("/api/orders/1")
    .request(MediaType.APPLICATION_JSON)
    .get();
assertEquals(200, response.getStatus());
Order order = response.readEntity(Order.class);

// AFTER (REST Assured)
import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

@QuarkusTest
public class OrderResourceTest {

    @Test
    public void testGetOrder() {
        given()
            .when().get("/api/orders/1")
            .then()
            .statusCode(200)
            .body("id", equalTo(1))
            .body("status", equalTo("ACTIVE"));
    }
}
```

- **Test data setup**: `@Deployment` data init → `@TestTransaction` or `@QuarkusTestResource`:

```java
// Per-test transactional data (rolled back after each test)
@QuarkusTest
@TestTransaction
public class OrderServiceTest {
    @Inject
    EntityManager em;

    @BeforeEach
    public void setup() {
        em.persist(new Order("test-order"));
    }

    @Test
    public void testFindOrder() { /* ... */ }
}
```

- **TestContainers integration** (for real database testing):
```java
@QuarkusTest
@QuarkusTestResource(PostgresTestResource.class)
public class OrderRepositoryIT {
    // Uses real PostgreSQL via TestContainers
}
```

- **Mocking CDI beans** — use `@InjectMock` (Quarkus Mockito extension):

```java
@QuarkusTest
public class OrderResourceTest {
    @InjectMock
    OrderService orderService;  // CDI bean replaced with mock

    @Test
    public void testGetOrder() {
        when(orderService.findById(1L)).thenReturn(new Order(1L, "test"));

        given()
            .when().get("/api/orders/1")
            .then()
            .statusCode(200);
    }
}
```

- **JUnit 4 → JUnit 5**: `@RunWith` → `@ExtendWith` (but not needed with `@QuarkusTest`). `Assert.assertEquals` → `Assertions.assertEquals`. `@Test(expected=...)` → `assertThrows(...)`. `@Before`/`@After` → `@BeforeEach`/`@AfterEach`. **Message reordering**: 3-arg `assertEquals("msg", expected, actual)` → `assertEquals(expected, actual, "msg")`.
- If SUREFIRE_DISABLED: re-enable surefire plugin — remove `<skip>true</skip>` or set `<skipTests>false</skipTests>`. Quarkus requires maven-surefire-plugin 3.x:
```xml
<plugin>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <systemPropertyVariables>
            <java.util.logging.manager>org.jboss.logmanager.LogManager</java.util.logging.manager>
        </systemPropertyVariables>
    </configuration>
</plugin>
```

- **Integration tests** (`*IT.java`): use `@QuarkusIntegrationTest` (runs against the built artifact — uber-jar or native):
```java
@QuarkusIntegrationTest
public class OrderResourceIT {
    // REST Assured tests run against the packaged application
    @Test
    public void testHealthEndpoint() {
        given()
            .when().get("/q/health")
            .then()
            .statusCode(200);
    }
}
```

- **Security in tests**: use `@TestSecurity` to inject roles without real auth:
```java
@QuarkusTest
public class AdminResourceTest {
    @Test
    @TestSecurity(user = "admin", roles = "ADMIN")
    public void testAdminEndpoint() {
        given()
            .when().get("/admin/dashboard")
            .then()
            .statusCode(200);
    }
}
```

- Delete `arquillian.xml`, `test-ds.xml`, and any Arquillian container adapter configs.
- Create `src/test/resources/application.properties` for test-specific overrides:
```properties
# Test datasource (H2 in-memory)
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:testdb
quarkus.hibernate-orm.schema-management.strategy=drop-and-create
quarkus.hibernate-orm.log.sql=true
```

**Step 16** (if JSF_NEEDED): Migrate UI. See `references/jsf-to-qute.md` for detailed template conversion patterns.

**Option A — Qute templates** (recommended for cloud-native):

- Remove JSF dependencies:
```xml
<!-- DELETE all of these -->
<dependency><groupId>javax.faces</groupId><artifactId>javax.faces-api</artifactId></dependency>
<dependency><groupId>jakarta.faces</groupId><artifactId>jakarta.faces-api</artifactId></dependency>
<dependency><groupId>org.glassfish</groupId><artifactId>javax.faces</artifactId></dependency>
<dependency><groupId>org.primefaces</groupId><artifactId>primefaces</artifactId></dependency>
<dependency><groupId>org.apache.myfaces.core</groupId><artifactId>*</artifactId></dependency>
```

- Add Qute extension:
```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-rest-qute</artifactId>
</dependency>
```

- **Fix hardcoded context paths in JSF/Facelets templates**: WAR apps have context paths like `/app-name` — since Quarkus serves at root `/`, all hardcoded context references must be removed. Scan: `grep -rn "request.contextPath\|/old-context-path" src/main/resources/` and replace with root paths or `#{request.contextPath}` (which resolves to empty in Quarkus).

- **Verify JSF namespace URIs**: Common corruption during transformation: `jakarta.face.html` (missing 's'). Must be `jakarta.faces.html`, `jakarta.faces.core`, `jakarta.faces.facelets`. Detection: `grep -rn "jakarta\.face\." src/ | grep -v "jakarta\.faces\."` — must return empty.

- **Convert `.xhtml` → `.html` Qute templates** in `src/main/resources/templates/`:

```html
<!-- BEFORE: src/main/webapp/orders.xhtml (JSF) -->
<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:h="http://xmlns.jcp.org/jsf/html"
      xmlns:ui="http://xmlns.jcp.org/jsf/facelets"
      xmlns:f="http://xmlns.jcp.org/jsf/core">
<h:body>
    <h:form>
        <h:dataTable value="#{orderBean.orders}" var="order">
            <h:column>#{order.id}</h:column>
            <h:column>#{order.status}</h:column>
        </h:dataTable>
        <h:commandButton value="Refresh" action="#{orderBean.refresh}" />
    </h:form>
</h:body>
</html>

<!-- AFTER: src/main/resources/templates/orders.html (Qute) -->
<!DOCTYPE html>
<html>
<body>
    <table>
        {#for order in orders}
        <tr>
            <td>{order.id}</td>
            <td>{order.status}</td>
        </tr>
        {/for}
    </table>
    <form action="/orders/refresh" method="POST">
        <button type="submit">Refresh</button>
    </form>
</body>
</html>
```

- **Rename dotted @Named beans** (check BOTH public-facing AND admin backing beans): Scan for `@Named` annotations containing dots: `grep -rn '@Named(".*\\..*")' src/main/java/`. For each match, rename to camelCase (e.g., `@Named("public.track")` → `@Named("publicTrack")`), then find and replace all EL references in `.xhtml` templates (`#{old.name.property}` → `#{newName.property}`). This is required because Quarkus/MyFaces uses the Expressly EL implementation which strictly interprets dots as property accessors.

- **`@ManagedBean`/`@Named` backing bean** → `@ApplicationScoped` CDI bean + `Template` injection:

```java
// BEFORE (JSF)
@Named
@ViewScoped
public class OrderBean implements Serializable {
    @Inject
    private OrderService orderService;
    private List<Order> orders;

    @PostConstruct
    public void init() {
        orders = orderService.findAll();
    }

    public String refresh() {
        orders = orderService.findAll();
        return null; // stay on same page
    }

    public List<Order> getOrders() { return orders; }
}

// AFTER (Quarkus — JAX-RS resource + Qute template)
@Path("/orders")
@ApplicationScoped
public class OrderResource {
    @Inject
    OrderService orderService;

    @Inject
    Template orders;  // matches templates/orders.html

    @GET
    @Produces(MediaType.TEXT_HTML)
    public TemplateInstance listOrders() {
        return orders.data("orders", orderService.findAll());
    }

    @POST
    @Path("/refresh")
    @Produces(MediaType.TEXT_HTML)
    public TemplateInstance refresh() {
        return orders.data("orders", orderService.findAll());
    }
}
```

- **JSF navigation rules** (`faces-config.xml`) → direct JAX-RS endpoint paths with redirects (`Response.seeOther()`).
- **`@ViewScoped`** → `@RequestScoped` or stateless endpoint (Qute is stateless by design — no server-side view state).
- **`@FlowScoped`** → multi-step form patterns with hidden fields or URL path segments.
- **PrimeFaces/RichFaces components** → standard HTML + HTMX or JavaScript library. This is significant effort — flag for human review if >10 PrimeFaces-specific components exist.
- Delete `faces-config.xml`, `src/main/webapp/WEB-INF/web.xml` (if only JSF config remained), and all `.xhtml` files after conversion.

**Option B — MyFaces extension** (preserves JSF — faster migration, less cloud-native):

- Add Apache MyFaces Quarkus extension (community):
```xml
<dependency>
    <groupId>org.apache.myfaces.core.extensions.quarkus</groupId>
    <artifactId>myfaces-quarkus</artifactId>
    <version>4.0.2</version>
</dependency>
```

- **MyFaces init parameters**: NOT configurable via `quarkus.myfaces.*` in `application.properties`. Set JSF context-params in minimal `src/main/resources/META-INF/web.xml` or rely on MyFaces defaults. Common parameters: `jakarta.faces.PROJECT_STAGE`, `jakarta.faces.FACELETS_SKIP_COMMENTS`.

- Minimal code changes — `faces-config.xml` and `.xhtml` templates remain. `@Named`/`@ViewScoped` beans continue working.
- **Limitations**: no native compilation support, larger memory footprint, slower startup. Acceptable for migration step but not final state for cloud-native deployment.
- Move `.xhtml` files from `src/main/webapp/` → `src/main/resources/META-INF/resources/` (Quarkus serves static/template content from resources).

**Step 17** (if BATCH_NEEDED): Migrate batch processing.

- **Simple scheduled jobs** (EJB `@Schedule`): already handled in Phase 2 Step 9 via `@Scheduled`. Verify migration is complete.
- **JSR 352 Batch** (complex multi-step jobs with `ItemReader`/`ItemProcessor`/`ItemWriter`):

**Option A — JBeret extension** (preserves JSR 352 API):
```xml
<dependency>
    <groupId>io.quarkiverse.jberet</groupId>
    <artifactId>quarkus-jberet</artifactId>
</dependency>
```
  - Batch job XML files (`META-INF/batch-jobs/*.xml`) work unchanged.
  - `@BatchProperty` → stays as-is.
  - `JobOperator` → inject via `@Inject JobOperator`.
  - **Note**: JBeret extension may have limited native compilation support — verify if native build is required.

**Option B — Redesign to Quarkus-native** (for simpler jobs):
  - Convert `ItemReader`/`ItemProcessor`/`ItemWriter` → `@ApplicationScoped` bean with `@Scheduled` and chunked processing logic:

```java
// BEFORE (JSR 352 batch job XML + classes)
// batch-jobs/importOrders.xml defines reader/processor/writer chain

// AFTER (Quarkus-native chunked processing)
@ApplicationScoped
public class OrderImportJob {
    @Inject
    OrderService orderService;

    @Inject
    EntityManager em;

    @Scheduled(cron = "0 0 2 * * ?")  // daily at 2am
    @Transactional
    public void importOrders() {
        List<RawOrder> batch;
        int offset = 0;
        do {
            batch = readBatch(offset, 100);
            batch.stream()
                .map(this::processOrder)
                .forEach(em::persist);
            em.flush();
            em.clear();  // prevent OOM on large batches
            offset += batch.size();
        } while (!batch.isEmpty());
    }
}
```

- **Checkpoint/restart semantics**: if the original batch job relies on JSR 352 checkpointing, preserve with JBeret (Option A). Pure Quarkus-native redesign loses automatic checkpoint/restart — implement manually if needed.

**Exit gate**: Run `./mvnw clean test` — all tests must pass. Additional verification:
- No `@RunWith(Arquillian.class)` remaining: `grep -rn "Arquillian" src/test/` — must return empty
- No ShrinkWrap imports remaining: `grep -rn "shrinkwrap\|ShrinkWrap" src/test/` — must return empty
- No JUnit 4 `@Test` (org.junit.Test) remaining: `grep -rn "import org.junit.Test" src/test/` — must return empty (should be `org.junit.jupiter.api.Test`)
- Test count: verify migrated test count matches original (no tests dropped)

### Phase 5: Deployment & Verification (ALWAYS)

Exit: `./mvnw clean verify` passes AND Docker image builds AND `/q/health` returns UP.

**Step 18**: Generate containerization artifacts.

- **Remove old app server Dockerfiles**: Delete root `Dockerfile` (Payara/WildFly/Liberty), `post-boot-commands.asadmin`, `server.xml`, `glassfish-resources.xml`, and other app-server-specific deployment artifacts. Verify: `find . -maxdepth 1 -name "Dockerfile" -o -name "*.asadmin" -o -name "server.xml"` — must return empty.

**Native image compatibility check (if JSF_NEEDED=true):**
- DO NOT generate Dockerfile.native or Dockerfile.native-micro when JSF_NEEDED=true
- JSF/MyFaces + PrimeFaces are NOT compatible with GraalVM native compilation due to:
  - Extensive runtime reflection in the JSF lifecycle
  - Dynamic proxy generation for managed beans and view scoping
  - Runtime classpath scanning for tag libraries
  - Serialization of server-side view state
- Only generate Dockerfile.jvm for JSF applications
- Add a comment in the generated Dockerfile.jvm: "# Native compilation not supported for JSF/MyFaces applications"
- If JSF_NEEDED=false, generate all three Dockerfiles (jvm, native, native-micro)

- **Generate Quarkus Dockerfiles** (conditional on JSF_NEEDED):
  - `src/main/docker/Dockerfile.jvm` — Standard JVM mode (required for JSF apps)
  - `src/main/docker/Dockerfile.native` — Multi-stage native build (only if JSF_NEEDED=false)  
  - `src/main/docker/Dockerfile.native-micro` — Pre-built native binary (only if JSF_NEEDED=false)

- Create `src/main/docker/Dockerfile.jvm` (JVM mode — recommended default):

```dockerfile
# -- JVM Mode Dockerfile --
FROM registry.access.redhat.com/ubi8/openjdk-17:1.18

ENV LANGUAGE='en_US:en'

# Quarkus fast-jar layout
COPY --chown=185 target/quarkus-app/lib/ /deployments/lib/
COPY --chown=185 target/quarkus-app/*.jar /deployments/
COPY --chown=185 target/quarkus-app/app/ /deployments/app/
COPY --chown=185 target/quarkus-app/quarkus/ /deployments/quarkus/

EXPOSE 8080
USER 185
ENV JAVA_OPTS_APPEND="-Dquarkus.http.host=0.0.0.0 -Djava.util.logging.manager=org.jboss.logmanager.LogManager"
ENV JAVA_APP_JAR="/deployments/quarkus-run.jar"

ENTRYPOINT [ "/opt/jboss/container/java/run/run-java.sh" ]
```

- **Alternative** — simplified JVM Dockerfile (non-Red Hat base):
```dockerfile
FROM amazoncorretto:17-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY target/quarkus-app/ /app/
USER appuser
EXPOSE 8080
ENTRYPOINT ["java", \
  "-XX:MaxRAMPercentage=75.0", \
  "-Dquarkus.http.host=0.0.0.0", \
  "-jar", "/app/quarkus-run.jar"]
```

- Create `src/main/docker/Dockerfile.native` (multi-stage native build — no local GraalVM needed):

```dockerfile
FROM quay.io/quarkus/ubi-quarkus-mandrel-builder-image:jdk-21 AS build
WORKDIR /build
COPY --chown=quarkus:quarkus . /build
RUN ./mvnw package -Dnative -DskipTests

FROM quay.io/quarkus/quarkus-micro-image:2.0
WORKDIR /work/
COPY --from=build /build/target/*-runner /work/application
RUN chmod 775 /work/application
EXPOSE 8080
CMD ["./application", "-Dquarkus.http.host=0.0.0.0"]
```

- Create `src/main/docker/Dockerfile.native-micro` (pre-built native binary — smallest image):

```dockerfile
FROM quay.io/quarkus/quarkus-micro-image:2.0
WORKDIR /work/
RUN chown 1001 /work \
    && chmod "g+rwX" /work \
    && chown 1001:root /work
COPY --chown=1001:root target/*-runner /work/application
EXPOSE 8080
USER 1001
ENTRYPOINT ["./application", "-Dquarkus.http.host=0.0.0.0"]
```

- **Update docker-compose.yml**: Replace app-server-specific environment variables with Quarkus naming convention. Use `QUARKUS_DATASOURCE_DB_KIND`, `QUARKUS_DATASOURCE_JDBC_URL`, `QUARKUS_DATASOURCE_USERNAME`, `QUARKUS_HIBERNATE_ORM_SCHEMA_MANAGEMENT_STRATEGY`, etc. Update `dockerfile:` path to `src/main/docker/Dockerfile.jvm`. Add a health check to the service:

```yaml
services:
  app:
    build:
      context: .
      dockerfile: src/main/docker/Dockerfile.jvm
    ports:
      - "8080:8080"
    environment:
      - QUARKUS_DATASOURCE_JDBC_URL=jdbc:postgresql://db:5432/mydb
      - QUARKUS_DATASOURCE_USERNAME=admin
      - QUARKUS_DATASOURCE_PASSWORD=admin
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/q/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

**Docker container database configuration:**
- The default Quarkus profile in containers is `prod` — ensure `%prod.quarkus.hibernate-orm.schema-management.strategy` is set appropriately
- For dev/demo containers with embedded H2, pass environment variables:
  - QUARKUS_DATASOURCE_JDBC_URL=jdbc:h2:mem:appname;DB_CLOSE_DELAY=-1
  - QUARKUS_HIBERNATE_ORM_SCHEMA_MANAGEMENT_STRATEGY=drop-and-create
  - QUARKUS_HIBERNATE_ORM_SQL_LOAD_SCRIPT=<load-script-filename>
- Or define a `%container` profile in `application.properties` with in-memory H2 configuration

- If REFLECTION_HEAVY (detected by: heavy use of Jackson polymorphism, JAXB, `Class.forName()`, or frameworks that rely on runtime reflection): add `@RegisterForReflection` to affected classes:

```java
// Classes that need reflection at runtime (native mode)
@RegisterForReflection
public class OrderDTO {
    // Jackson serialization target, etc.
}
```

  Or generate `src/main/resources/META-INF/native-image/reflect-config.json` for third-party classes that cannot be annotated.

- Create `.dockerignore`:
```
*
!target/quarkus-app
!target/*-runner
!target/*-runner.jar
```

- Configure Quarkus packaging in `application.properties`:
```properties
# Build uber-jar (alternative to fast-jar for simpler Docker COPY)
#quarkus.package.jar.type=uber-jar

# Default: fast-jar (recommended — smaller layers, faster startup)
quarkus.package.jar.type=fast-jar
```

**Step 19**: Configure health and observability.

- Add health and metrics extensions:
```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-health</artifactId>
</dependency>
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-micrometer-registry-prometheus</artifactId>
</dependency>
```

- Health endpoints are automatically available:
  - `/q/health` — combined health check (liveness + readiness)
  - `/q/health/live` — liveness probe
  - `/q/health/ready` — readiness probe
  - `/q/metrics` — Prometheus metrics

- Add custom health check if datasource or external service validation is needed:
```java
@Liveness
@ApplicationScoped
public class AppLivenessCheck implements HealthCheck {
    @Override
    public HealthCheckResponse call() {
        return HealthCheckResponse.up("app-alive");
    }
}

@Readiness
@ApplicationScoped
public class DatabaseReadinessCheck implements HealthCheck {
    @Inject
    AgroalDataSource dataSource;

    @Override
    public HealthCheckResponse call() {
        try (var conn = dataSource.getConnection()) {
            return HealthCheckResponse.up("database");
        } catch (Exception e) {
            return HealthCheckResponse.down("database");
        }
    }
}
```

- Configure production profile in `application.properties`:
```properties
# Production profile configuration
%prod.quarkus.http.port=8080
%prod.quarkus.datasource.jdbc.url=${DB_URL}
%prod.quarkus.datasource.username=${DB_USER}
%prod.quarkus.datasource.password=${DB_PASSWORD}
%prod.quarkus.datasource.jdbc.max-size=20
%prod.quarkus.hibernate-orm.schema-management.strategy=none
%prod.quarkus.hibernate-orm.log.sql=false

# Logging
%prod.quarkus.log.level=INFO
%prod.quarkus.log.category."org.hibernate".level=WARN
%prod.quarkus.log.console.format=%d{yyyy-MM-dd HH:mm:ss} %-5p [%c{2.}] (%t) %s%e%n

# Health and metrics
quarkus.smallrye-health.ui.always-include=true
quarkus.micrometer.export.prometheus.enabled=true
```

- **Kubernetes/OpenShift deployment** (optional): generate manifests:
```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-kubernetes</artifactId>
</dependency>
```
```properties
# Kubernetes manifest generation
quarkus.kubernetes.deployment-target=kubernetes
quarkus.kubernetes.liveness-probe.http-action-path=/q/health/live
quarkus.kubernetes.readiness-probe.http-action-path=/q/health/ready
quarkus.kubernetes.resources.requests.cpu=250m
quarkus.kubernetes.resources.requests.memory=256Mi
quarkus.kubernetes.resources.limits.cpu=1000m
quarkus.kubernetes.resources.limits.memory=512Mi
```

  Running `./mvnw package` generates `target/kubernetes/kubernetes.yml` with Deployment, Service, and probe configuration.

**Step 20**: Final verification scan.

- **Build verification**: Run `./mvnw clean verify` — must pass with zero errors and all tests green.

- **Legacy import scan** (if NAMESPACE_STATUS was JAVAX_ONLY):
```bash
grep -rn "import javax\." src/main/java/ | grep -v "javax\.crypto\|javax\.net\|javax\.sql\|javax\.security\.cert\|javax\.swing\|javax\.xml\.parsers\|javax\.xml\.transform"
```
  Expected: ZERO results. Any remaining `javax.*` EE imports indicate incomplete namespace migration.

- **Legacy annotation scan**:
```bash
grep -rn "@Stateless\|@Stateful\|@MessageDriven\|@EJB\|@TransactionAttribute" src/main/java/
```
  Expected: ZERO results.

- **Application server descriptor scan**:
```bash
find src/ -name "jboss-*.xml" -o -name "glassfish-*.xml" -o -name "sun-*.xml" -o -name "ibm-*.xml" -o -name "persistence.xml" -o -name "beans.xml" -o -name "web.xml" -o -name "arquillian.xml"
```
  Expected: ZERO results (all descriptors should have been removed in Phase 1).

- **Dependency tree scan** — verify no app server dependencies leaked through:
```bash
./mvnw dependency:tree | grep -i "jboss\|wildfly\|glassfish\|javax.ejb\|javax.faces"
```
  Expected: ZERO results (jboss-logging transitive from Hibernate is acceptable).

- **Docker image build verification**:
```bash
./mvnw package -DskipTests
docker build -f src/main/docker/Dockerfile.jvm -t app-quarkus-test .
```
  Must complete without errors.

- **Container smoke test**:
```bash
docker run --rm -d -p 8080:8080 --name quarkus-verify app-quarkus-test
# Wait for startup (Quarkus typically starts in <2s)
sleep 5
curl -s http://localhost:8080/q/health | grep -q '"status": "UP"'
docker stop quarkus-verify
```
  Health endpoint must return `"status": "UP"`.

- **Dev mode smoke test** (manual verification):
```bash
./mvnw quarkus:dev
# Verify hot-reload works — edit a source file and confirm changes apply without restart
# Verify dev UI accessible at http://localhost:8080/q/dev-ui
```

- **Performance baseline** (informational — not a pass/fail gate):
  - Startup time: check `Quarkus started in X.XXXs` log line (JVM mode typically <3s, native <0.1s)
  - Memory: `docker stats quarkus-verify` — JVM mode typically <256MB idle, native <64MB idle

- **Cleanup**: update `README.md` with new build/run instructions. Update `.gitignore`:
```
# Quarkus
target/
.quarkus/
```
  Delete remaining legacy directories: `src/main/webapp/`, any `*-ds.xml`, any remaining app server config.

**Exit gate**: ALL of the following must be true:
1. `./mvnw clean verify` passes (compile + all tests)
2. Docker image builds successfully
3. Container starts and `/q/health` returns `"status": "UP"`
4. Zero legacy `javax.*` EE imports in production code
5. Zero application server descriptors remaining
6. Zero EJB annotations remaining in source


## Reference Dispatch

Load reference files on demand when the worker encounters these signals:

| Signal in Source Code | Reference File |
|---|---|
| Phase 0 flag detection, pom.xml dep scanning | `references/phase0-detection-flags.md` |
| `@Stateless`, `@Stateful`, `@Singleton`, `@EJB`, `@TransactionAttribute` | `references/ejb-to-cdi-mapping.md` |
| ArC proxy errors, `@Vetoed`, CDI extensions, bean removal | `references/arc-limitations.md` |
| `persistence.xml`, `@PersistenceContext`, `EntityManager`, Panache | `references/jpa-to-quarkus-persistence.md` |
| `@MessageDriven`, `JMSContext`, `ConnectionFactory`, `@Incoming` | `references/jms-to-smallrye.md` |
| `@RolesAllowed`, `@DeclareRoles`, `SessionContext`, `LoginModule` | `references/security-to-quarkus-security.md` |
| `.xhtml`, `@Named`, `@ViewScoped`, JSF backing beans, Qute templates | `references/jsf-to-qute.md` |
| `@RunWith(Arquillian`, `@Deployment`, `ShrinkWrap`, test migration | `references/arquillian-to-quarkustest.md` |
| Extension selection, dependency mapping, Maven GAV lookup | `references/quarkus-extension-catalog.md` |
| MDB→@Incoming, Security, SOAP, JSF, Arquillian worked examples | `references/worked-examples-conditional.md` |

## Validation / Exit Criteria

All 20 must be met:

1. `./mvnw clean verify` passes with zero test failures and zero compilation errors.
2. No `javax.*` EE imports remain in `src/main/java/` (excluding Java SE packages: `javax.crypto`, `javax.net`, `javax.sql`, `javax.security.cert`, `javax.swing`, `javax.xml.parsers`, `javax.xml.transform`). Verify: `grep -rn "import javax\." src/main/java/ | grep -v "javax\.crypto\|javax\.net\|javax\.sql\|javax\.security\.cert\|javax\.swing\|javax\.xml\.parsers\|javax\.xml\.transform"` — must return empty.
3. No application server descriptors remain: `jboss-web.xml`, `jboss-ejb3.xml`, `jboss-deployment-structure.xml`, `glassfish-web.xml`, `sun-web.xml`, `weblogic.xml`, `ibm-web-ext.xml`, `standalone.xml`, `domain.xml`. Verify: `find src/ -name "jboss-*.xml" -o -name "glassfish-*.xml" -o -name "sun-*.xml" -o -name "weblogic*.xml" -o -name "ibm-*.xml"` — must return empty.
4. No EJB annotations remain (`@Stateless`, `@Stateful`, `@Singleton` from javax.ejb/jakarta.ejb, `@EJB`, `@MessageDriven`, `@TransactionAttribute`). Verify: `grep -rn "@Stateless\|@Stateful\|@MessageDriven\|@EJB\|@TransactionAttribute" src/main/java/` — must return empty.
5. No `persistence.xml` required — all datasource and Hibernate ORM configuration lives in `application.properties`. If `persistence.xml` is retained for compatibility, it must be stripped to `<persistence-unit>` declaration only (no properties).
6. All REST endpoints respond correctly — same paths, HTTP methods, request/response formats, and status codes as the original application. Functional parity verified by tests.
7. All `@Inject` points resolve at build time — no `UnsatisfiedResolutionException` or `AmbiguousResolutionException` at Quarkus startup. Application boots cleanly.
8. Docker image builds successfully from generated `Dockerfile.jvm`: `docker build -f src/main/docker/Dockerfile.jvm -t app-test .` completes without errors.
9. Application starts in <10 seconds in JVM mode. Verify from Quarkus startup log: `Quarkus started in X.XXXs`.
10. `/q/health` endpoint returns `"status": "UP"` with both liveness and readiness checks passing.
11. `/q/metrics` endpoint active and returning Prometheus-format metrics (if `quarkus-micrometer-registry-prometheus` is included).
12. No JNDI lookups remain — no `InitialContext`, no `context.lookup()`, no `@Resource` for environment entries. Verify: `grep -rn "InitialContext\|context\.lookup\|@Resource" src/main/java/ | grep -v "//.*InitialContext"` — must return empty (comments excluded).
13. All original test assertions pass — no `@Disabled` tests added as a migration shortcut. Test count matches original. Verify: `grep -rn "@Disabled" src/test/` — must return empty.
14. No CDI ambiguous dependency errors at build time — Quarkus build output contains no `AmbiguousResolutionException` warnings.
15. If native target is required: `./mvnw package -Dnative -DskipTests` builds without reflection errors or missing class registrations. Add `@RegisterForReflection` where needed.
16. Business logic classes unchanged — only framework/infrastructure code (annotations, injection patterns, configuration) is modified. Public method signatures, algorithmic logic, and data transformations remain identical.
17. Transaction semantics preserved — same transaction boundaries and propagation behavior. `@Transactional` with appropriate `TxType` replaces all EJB `@TransactionAttribute` mappings. No implicit transaction boundary changes.
18. Security constraints preserved — same endpoints protected with same role requirements. `@RolesAllowed` annotations match original, HTTP policy config in `application.properties` mirrors web.xml `<security-constraint>` rules.
19. No unused/dead dependencies in `pom.xml` — removed app server BOMs, removed individual libs now provided by Quarkus BOM, no duplicate dependencies. Verify: `./mvnw dependency:analyze` reports no unused declared dependencies.
20. `application.properties` organized by concern with clear sections: datasource (`quarkus.datasource.*`), HTTP (`quarkus.http.*`), Hibernate ORM (`quarkus.hibernate-orm.*`), security (`quarkus.http.auth.*`), messaging (`mp.messaging.*`), and profile-specific overrides (`%dev.*`, `%test.*`, `%prod.*`).

## Common Pitfalls

See `references/arc-limitations.md` for ArC-specific issues. Additional common migration failures:

- **Bean removed by unused-bean optimization**: set `quarkus.arc.remove-unused-beans=false` during migration, then re-enable after stabilization.
- **Final class cannot be proxied**: add `quarkus.arc.transform-unproxyable-classes=true` or remove `final` keyword.
- **Missing no-arg constructor**: normal-scoped beans (`@ApplicationScoped`, `@RequestScoped`) need a no-arg constructor for ArC proxy generation.
- **Namespace migration incomplete**: mixed `javax.*` and `jakarta.*` in same file causes compilation errors. Always migrate ALL EE namespaces in one pass per file.
- **Self-invocation bypasses interceptors**: `@Transactional` on a method called internally (`this.method()`) is not intercepted. Extract to a separate bean or use `Arc.container().instance()`.

## Tips

- [2026-03] Quarkus 3.33 is the current LTS (Long Term Support) version. LTS releases are supported for 12 months. Use `quarkus update` CLI command to update existing Quarkus apps between versions.
- [2026-06] **Dots in @Named bean names break EL resolution on Quarkus/MyFaces.** CDI beans with `@Named("foo.bar")` work on Payara/GlassFish/WildFly but fail on Quarkus because the Expressly EL resolver treats the dot as a property accessor. `#{public.track.trackingId}` is parsed as "bean `public` → property `track` → property `trackingId`" instead of "bean `public.track` → property `trackingId`". Fix: rename to camelCase (`@Named("fooBar")`) and update all EL references in `.xhtml` templates.
- [2026-06] `quarkus.hibernate-orm.database.generation` is deprecated since Quarkus 3.31. Use `quarkus.hibernate-orm.schema-management.strategy` instead. Values: `drop-and-create`, `update`, `none`, `validate`.
- [2026-06] **WAR→Quarkus context path removal**: WAR apps deployed to `/app-name` context paths must have all hardcoded context references updated. Quarkus serves at root `/` by default. Replace `/old-context/path` → `/path` in templates and JavaScript. `#{request.contextPath}` resolves to empty string in Quarkus.
- [2026-06] `-XX:+UseContainerSupport` has been enabled by default since Java 10 (JDK-8146115). Do NOT include it in Dockerfiles for Java 17+ — it's a no-op and adds clutter.
- [2026-06] **JSF/MyFaces apps cannot use GraalVM native compilation.** JSF relies on runtime reflection, dynamic proxies, and classpath scanning that are fundamentally incompatible with GraalVM's closed-world assumption. Only generate Dockerfile.jvm for apps with JSF_NEEDED=true. Non-JSF apps (pure JAX-RS + CDI) work well in native mode with sub-second startup.
- [2026-06] Quarkus Dev UI at `/q/dev-ui` provides build-time bean inspection, config editor, and extension management — invaluable during migration debugging.
- [2026-06] `quarkus.log.category."io.quarkus.arc".level=DEBUG` logs all bean discovery decisions — helps diagnose missing or removed beans.
- [2026-06] Use `./mvnw quarkus:dev` continuous testing mode for rapid feedback during migration — tests re-run automatically on source changes.
- [2026-06] **Class-level `@Transactional` applies to ALL methods** including read-only getters — unnecessary overhead. Always use method-level `@Transactional`. Use `@Transactional(TxType.SUPPORTS)` or no annotation for read-only methods.
- [2026-06] **JSF `@Named` beans without an explicit CDI scope default to `@Dependent`** (new instance per injection point). This breaks JSF views. Always add `@RequestScoped` (or `@ViewScoped`/`@SessionScoped`) to all JSF backing beans. Detection: `grep -rn "@Named" src/main/java/ | grep -v "//"`
- [2026-06] **`faces-config.xml` must use Jakarta EE 4.0 namespace for Quarkus 3.x / MyFaces 4.x.** Change `xmlns="http://xmlns.jcp.org/xml/ns/javaee"` → `xmlns="https://jakarta.ee/xml/ns/jakartaee"`, version `2.2` → `4.0`.
- [2026-06] **Non-CDI service classes instantiated via `new` that delegate to CDI beans must become `@ApplicationScoped`.** Using `Arc.container().instance(...).get()` as a JNDI replacement is an anti-pattern — bypasses CDI lifecycle and interceptors. Detection: `grep -rn "Arc\.container\(\)\|new.*Action\(\)" src/main/java/`
- [2026-06] **Remove static resource caches from migrated CDI beans.** Static `DataSource`/`ConnectionFactory` fields initialized in `static init()` via `Arc.container()` break Quarkus dev mode. Replace with `@Inject` fields. Detection: `grep -rn "private static.*DataSource\|private static.*initialized\b" src/main/java/`
- [2026-06] **Prefer `@Inject ManagedExecutor` over `Executors.newCachedThreadPool()` in CDI beans.** Unmanaged pools have no graceful shutdown integration. Use `org.eclipse.microprofile.context.ManagedExecutor` from `quarkus-smallrye-context-propagation`.
- [2026-06] **Use UBI9 as the Docker base image.** Replace outdated base images with `registry.access.redhat.com/ubi9/openjdk-17` for security updates and Red Hat support. UBI (Universal Base Image) is optimized for containers and includes security scanning.
