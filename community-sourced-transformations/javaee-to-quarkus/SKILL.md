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

## Pre-Migration Blockers

**REMOTE_EJB_DETECTED** — @Remote interfaces detected. Quarkus has no EJB remote invocation support. Document which beans are remote, remove @Remote annotations from interfaces, and plan REST/gRPC replacements before proceeding.

## Constraints and Guardrails

- **API & Functional Parity**: Preserve all public class names, method signatures, REST endpoint paths, HTTP methods, request/response formats, and status codes. Business logic unchanged — only framework/infrastructure code modified.
- **Test Integrity**: Do NOT mark tests @Disabled or delete tests as migration shortcuts. Migrate Arquillian tests to @QuarkusTest equivalents preserving all assertions.
- **ArC CDI Limitations**: See `references/arc-limitations.md` for complete list. Key restrictions: no Portable Extensions, no bean-discovery-mode="all", normal-scoped beans require no-arg constructor.
- **Code Quality**: Constructor injection preferred. Remove unused imports. Follow Quarkus conventions.
- **Incremental Migration**: Verify build passes after each phase before proceeding.

## Worked Examples

See `references/worked-examples-conditional.md` for detailed before/after code examples covering:
- EJB to CDI Bean Migration
- JNDI to Configuration Properties  
- JMS to SmallRye Reactive Messaging
- JSF to Qute Templates
- Arquillian to @QuarkusTest

## Workflow

### Phase 0: Project Analysis (ALWAYS)

**Feature Flag Detection Table:**

| Flag | Triggers | Phase Activated |
|------|----------|-----------------|
| `REMOTE_EJB_DETECTED` | `grep -rn '@Remote\|@RemoteHome\|jboss-ejb-client\|ejb:/' src/ pom.xml | head -1` | **PRE-MIGRATION BLOCKER** — Remote EJB/IIOP is a Non-Goal. Halt with message: "Remote EJB interfaces detected. This is a Non-Goal per steering context — redesign remote calls to REST/gRPC before proceeding." |
| `EJB_NEEDED` | `@Stateless/@Stateful/@Singleton`, `ejb-jar.xml` | Phase 2 (EJB→CDI) |
| `JMS_NEEDED` | `@MessageDriven`, `javax.jms.*` imports | Phase 3 (Messaging) |
| `SECURITY_NEEDED` | `@RolesAllowed`, `<security-constraint>` in web.xml | Phase 3 (Security) |
| `JSF_NEEDED` | `.xhtml` files, `javax.faces.*` imports | Phase 4 (UI) |
| `HAS_JSF_NAMESPACE_TYPO` | `grep -rn "jakarta\.face\." src/ | grep -v "jakarta\.faces\."` | Fix IMMEDIATELY — rename to jakarta.faces.* |
| `HAS_DOTTED_NAMED_BEANS` | `grep -rn '@Named(".*\\..*")' src/main/java/` | Phase 2 (CDI migration) |
| `HAS_JSP` | `find src/ -name '*.jsp' | head -1` returns a result | Phase 4 — JSP files must be migrated to Facelets or removed; Quarkus undertow has no JSP compiler |
| `BATCH_NEEDED` | `find src/ -name '*.xml' -path '*META-INF/batch-jobs*' | head -1 || grep -rn 'jakarta.batch\|javax.batch' src/main/java/ | head -1` | Phase 3 (Batch) |
| `MAIL_NEEDED` | `grep -rn 'jakarta.mail\|javax.mail\|@Resource.*mail\|Session.getInstance' src/main/java/` | Phase 3 (Mail) |
| `SOAP_NEEDED` | `@WebService`, WSDL files | Phase 3 (SOAP) |
| `IS_MULTI_MODULE` | Multiple `src/main/java` trees (EAR/EJB/WAR) | Phase 1 consolidation |

See `references/phase0-detection-flags.md` for complete flag list and detection bash commands.

**Phase 0 Actions:**
- Scan project structure and dependencies
- Set feature flags for conditional pipeline execution  
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

- Replace parent POM or add Quarkus BOM (import scope)
- Change packaging from WAR/EAR to JAR
- Add `quarkus-maven-plugin` with build goal
- Add core extensions based on Phase 0 flags (quarkus-arc always, plus quarkus-rest, hibernate-orm, etc. as needed)

CRITICAL: Always set `quarkus.platform.version=3.33.x` (current LTS). The ATX agent may default to a different Quarkus version if not explicitly specified. The recommended property in pom.xml: `<quarkus.platform.version>3.33.2</quarkus.platform.version>`. For Gradle: `id 'io.quarkus' version '3.33.2'`.

Configuration examples: See `references/quarkus-extension-catalog.md`

**MicroProfile API → Quarkus Extension Mapping:**

If the original project uses `org.eclipse.microprofile:*` APIs, add the corresponding Quarkus extensions:

| MicroProfile API | Quarkus Extension | Notes |
|---|---|---|
| microprofile-config-api | Included in core (quarkus-arc) | @ConfigProperty works natively |
| microprofile-fault-tolerance-api | quarkus-smallrye-fault-tolerance | @Retry, @CircuitBreaker, @Bulkhead, @Timeout, @Fallback |
| microprofile-health-api | quarkus-smallrye-health | @Liveness, @Readiness |
| microprofile-metrics-api | quarkus-micrometer-registry-prometheus | Different API — see Micrometer |
| microprofile-rest-client-api | quarkus-rest-client-reactive | @RegisterRestClient |
| microprofile-jwt-api | quarkus-smallrye-jwt | @Claim, @JsonWebToken |
| microprofile-openapi-api | quarkus-smallrye-openapi | @OpenAPIDefinition, @Operation |
| microprofile-opentracing-api | quarkus-opentelemetry | Different API — OpenTelemetry replaces OpenTracing |

Detection: `grep -rn "microprofile" pom.xml build.gradle`

- Remove application server dependencies:
    <!-- xml — see references/ for details -->

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
<!-- Detailed java example: see references/worked-examples-conditional.md -->

**Inline FQCN sweep**: after import migration, scan for fully-qualified class references in method bodies, catch blocks, instanceof checks, and annotations (see `references/phase0-detection-flags.md` for detection commands). Replace matching EE namespaces only.

**Step 4**: Migrate configuration.

Extract configuration from persistence.xml, web.xml, and app server descriptors to `src/main/resources/application.properties`:
- Datasource → quarkus.datasource.* properties
- Hibernate → quarkus.hibernate-orm.* properties  
- HTTP server → quarkus.http.* properties

Configuration examples: See `references/jpa-to-quarkus-persistence.md`

Remove all app server descriptors after extraction (jboss-web.xml, persistence.xml, beans.xml, etc.)

- **Remove deprecated config keys**: `quarkus.hibernate-orm.database.generation` may appear alongside the new key in migration outputs — remove it entirely, keep only `quarkus.hibernate-orm.schema-management.strategy`.

- **MyFaces configuration**: `org.apache.myfaces.PROJECT_STAGE` is NOT set via `quarkus.myfaces.*` application.properties keys. Set it as a `<context-param>` in `web.xml` (e.g., `jakarta.faces.PROJECT_STAGE=Production`) or leave as default. Remove any `quarkus.myfaces.*` keys from application.properties — they will generate "Unrecognized configuration key" warnings.

- When migrating `import.sql` or `src/test/resources/import.sql`: verify table names match actual JPA entity table names (default is the simple class name). `INSERT INTO User` will fail if the entity class is `Person`. Detection: `grep -i "INSERT INTO" src/test/resources/import.sql` — cross-check each table name against `@Table(name=...)` annotations or simple class names.

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

**Step 5**: Clean up application structure.

- No main class needed (Quarkus generates entry point at build time)
- Remove app server config files (standalone.xml, domain.xml, etc.)  
- Remove src/main/webapp/WEB-INF/ directory
- Convert @ApplicationPath to quarkus.rest.path in application.properties

**Exit gate**: `./mvnw clean compile -Dmaven.test.skip=true` succeeds

### Phase 2: Core Migration (CONDITIONAL - EJB detected)

Exit: `./mvnw clean compile` passes.

**Migration Comment Rule**: MIGRATION comments must use plain English, not annotation names (e.g., "Converted stateless session bean" not "Removed @Stateless").

**Step: Rename dotted @Named bean names BEFORE any other CDI migration.** EL resolvers interpret dots as property accessors — `#{public.track}` is parsed as bean `public` → property `track`, not as a single bean named `public.track`. This bug exists in the ORIGINAL SOURCE and must be fixed first. Detection: `grep -rn '@Named(".*\\..*")' src/main/java/`. For each match: rename to camelCase (`@Named("foo.bar")` → `@Named("fooBar")`), then update ALL EL references: `grep -rn 'foo\.bar' src/main/resources/`.

**Step 6**: Migrate EJBs to CDI beans.

- @Stateless → @ApplicationScoped + @Transactional (only for persistence operations)
- @Singleton → @ApplicationScoped (add StartupEvent observer if @Startup present)
- @Stateful → @SessionScoped (evaluate if stateful pattern still needed)
- @EJB → @Inject (constructor injection preferred)
- Remove @LocalBean, @Local, @Remote annotations
- Add no-arg constructor for normal-scoped beans (ArC proxy requirement)
- *(see `references/ejb-to-cdi-mapping.md` for complete mapping table and examples)*

**@TransactionAttribute mapping**: See reference table in `references/ejb-to-cdi-mapping.md`
- Apply @Transactional at method level, not class level
- Use appropriate TxType (REQUIRED, REQUIRES_NEW, etc.)
- Remove class-level `@Transactional` from service beans. Add `@Transactional` only to methods that write to the database. Methods that call external REST services (like `requestPossibleRoutesForCargo()`) must NOT have `@Transactional` — holding a DB connection during network I/O wastes pool resources. Detection: `grep -rn '@Transactional' src/main/java/ | grep -v '//'` — for each class-level annotation, split to method-level.
- *(see references/ for remaining transaction patterns)*

**Step 7**: Migrate JPA configuration.

- Remove persistence.xml (config moved to application.properties in Phase 1)
- Keep entity annotations as-is — jakarta.persistence.* works unchanged  
- @PersistenceContext → @Inject EntityManager
- Hibernate-specific annotations work unchanged (except @Type which is deprecated)
- Optional: Add quarkus-hibernate-orm-panache for active record pattern (don't force conversion)
- Multi-datasource: Use named datasources + @PersistenceUnit("name") qualifier

Configuration examples: See `references/jpa-to-quarkus-persistence.md`
<!-- Detailed properties example: see references/worked-examples-conditional.md -->

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
- **For SSE services (@ServerSentEvents / SseEventSink)**: Do NOT inject `@Context Sse sse` as a field in `@ApplicationScoped` beans — it is null at @PostConstruct. Instead: (a) inject `Sse sse` as a method parameter in the @GET endpoint, (b) initialize `SseBroadcaster` lazily on first use: `if (broadcaster == null) { synchronized(this) { if (broadcaster == null) broadcaster = sse.newBroadcaster(); } }`. Or migrate to Quarkus reactive `@RestStreamElementType + Multi<T>` pattern.
- **Quarkus-specific enhancements** (OPTIONAL — do NOT force): if migrating to RESTEasy Reactive idioms:
  - Return types can be `Uni<T>` for non-blocking (requires `quarkus-rest` extension, already added in Step 2).
  - `@Blocking` annotation on resource methods that perform blocking I/O (JDBC, file I/O) when using the reactive REST extension.
  - `@ServerExceptionMapper` can replace `@Provider ExceptionMapper<T>` classes (simpler syntax).

JAX-RS resources need minimal changes — just replace @EJB with @Inject. Examples: See `references/worked-examples-conditional.md`

**Step 9** (if SCHEDULER_NEEDED): Migrate EJB Timers to Quarkus Scheduler.

- Add scheduler extension: `quarkus-scheduler`
- @Schedule (EJB) → @Scheduled (Quarkus) with simplified cron/interval syntax

Examples: See `references/ejb-to-cdi-mapping.md`

| EJB @Schedule | Quarkus @Scheduled |
|---|---|
| `@Schedule(hour="*", minute="*/5")` | `@Scheduled(every="5m")` |
| `@Schedule(hour="2", minute="0")` | `@Scheduled(cron="0 0 2 * * ?")` |
| `@Schedule(dayOfWeek="Mon", hour="8")` | `@Scheduled(cron="0 0 8 ? * MON")` |

- **TimerService programmatic timers** → Quarkus `Scheduler` API:

<!-- Detailed java example: see references/worked-examples-conditional.md -->

- `@Timeout` methods → `@Scheduled` with the appropriate cron/interval expression. Drop `persistent=false` (Quarkus scheduled tasks are non-persistent by default).
- **@Schedules (plural)**: split into separate `@Scheduled` methods, one per schedule expression, each delegating to a shared private method.

**Step 10** (if WEBSOCKET_NEEDED): Migrate WebSockets.

- Add the WebSocket extension:
<!-- Detailed xml example: see references/worked-examples-conditional.md -->

- **If using `quarkus-websockets`** (JSR 356 compatibility): `@ServerEndpoint`, `@OnOpen`, `@OnMessage`, `@OnClose`, `@OnError` annotations work unchanged. Minimal migration needed.
- **If migrating to `quarkus-websockets-next`** (recommended for new development):
  - `@ServerEndpoint("/chat")` → `@WebSocket(path = "/chat")`
  - `@OnMessage` → `@OnTextMessage` / `@OnBinaryMessage`
  - Session management via `WebSocketConnection` injection instead of `javax.websocket.Session`

**Step: Migrate @WebFilter and @WebListener — conditional on HAS_SERVLET_FILTERS or HAS_SERVLET_LISTENERS**
Detection: `grep -rn '@WebFilter\|@WebListener\|implements Filter\|implements ServletContextListener' src/main/java/`

@WebFilter migration (choose one):
- Option A (keep as-is with quarkus-undertow): quarkus-undertow already added — @WebFilter works unchanged
- Option B (JAX-RS migration): `@Provider @PreMatching public class X implements ContainerRequestFilter`

@WebListener → CDI events:
- `contextInitialized()` → `void onStart(@Observes StartupEvent ev) {...}`
- `contextDestroyed()` → `void onStop(@Observes ShutdownEvent ev) {...}`

**Step 12** (if HAS_JNDI): Replace JNDI lookups.

- `InitialContext.lookup("java:comp/env/...")` → `@Inject` + `@ConfigProperty`:

<!-- Detailed java example: see references/worked-examples-conditional.md -->

- **`@RolesAllowed`** → stays unchanged. Quarkus supports `jakarta.annotation.security.@RolesAllowed` natively on JAX-RS resources and CDI beans. No migration needed.

When adding `@RolesAllowed` to servlets or REST endpoints, a Quarkus security extension MUST be present in pom.xml — otherwise annotations are silently ignored and endpoints remain unprotected. Add: `quarkus-security` + one identity source extension (e.g., `quarkus-elytron-security-properties-file` for basic auth, `quarkus-oidc` for OIDC). Detection: `grep -n '@RolesAllowed\|@PermitAll\|@DenyAll' src/main/java/` — if any found, verify a security extension is in pom.xml.
- **`@DeclareRoles`** → DELETE. Quarkus does not require role pre-declaration — roles are discovered from the identity provider at runtime.
- **`@PermitAll` / `@DenyAll`** → stay unchanged (jakarta.annotation.security).
- **`@RunAs`** → replace with `SecurityIdentity` augmentation. Implement a `SecurityIdentityAugmentor` to transform roles/principals:

<!-- Detailed java example: see references/worked-examples-conditional.md -->

- **Security constraints**: web.xml `<security-constraint>` → `quarkus.http.auth.policy.*` in application.properties
- **JAAS LoginModule** → implement `io.quarkus.security.identity.IdentityProvider<T>`

Security examples: See `references/security-to-quarkus-security.md`

<!-- Detailed java example: see references/worked-examples-conditional.md -->

- Replace `System.getProperty("jboss.node.name")` and similar JBoss/WildFly system properties with `@ConfigProperty`. These are runtime-only properties in WildFly that have no equivalent in Quarkus. Map to Quarkus-native keys: `quarkus.application.name` for node naming, or custom `@ConfigProperty(name = "clusterbench.node-name", defaultValue = "quarkus-node")`.

- **`EJBContext.isCallerInRole()` / `SessionContext.getCallerPrincipal()`** → inject `SecurityContext`:

<!-- Detailed java example: see references/worked-examples-conditional.md -->

- Review all servlets for dangerous operations (system restarts, JVM halt, process execution). Any servlet calling `Runtime.getRuntime().halt()`, `System.exit()`, or similar must have `@RolesAllowed("admin")` or be removed. Without security, these endpoints are accessible to anyone in Quarkus (no server-level admin security).

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

**Step 14** (if JMS_NEEDED or MDB_NEEDED): Migrate messaging. See `references/jms-to-smallrye.md` for detailed patterns.

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

- **JMS Migration Options**:
  1. **Option A**: Preserve JMS API with `@ApplicationScoped` bean + injected ConnectionFactory
  2. **Option B**: Convert to SmallRye Reactive Messaging with `@Incoming/@Outgoing` (recommended)

See `references/jms-to-smallrye.md` for complete JMS migration patterns including:
- @MessageDriven → @Incoming/@Outgoing conversions
- JMS ConnectionFactory injection  
- Queue/Topic configuration in application.properties
- Message producer/consumer patterns
- Error handling and transaction management

**Option B — Modernize to SmallRye Reactive Messaging** (recommended):

- Add reactive messaging extension (quarkus-messaging-amqp or quarkus-messaging-kafka)
- @MessageDriven → @ApplicationScoped + @Incoming("channel-name")
- Configure message channels in application.properties

Examples: See `references/jms-to-smallrye.md`

<!-- Detailed java example: see references/worked-examples-conditional.md -->

- **JMS producer** → `@Outgoing("channel-name")` or inject `Emitter<T>`:

<!-- Detailed java example: see references/worked-examples-conditional.md -->

- Configure channels in `application.properties`:
    <!-- properties — see references/ for details -->

- **Transacted message consumption**: SmallRye Reactive Messaging handles acknowledgment via `@Acknowledgment(Strategy.POST_PROCESSING)` (default). For manual ack: accept `Message<T>` parameter and call `message.ack()`.
- **Message selectors**: not directly supported in SmallRye — implement filtering logic in the consumer method or use separate channels per message type.
- **Dead-letter queue**: configure via broker-specific properties (`dead-letter-queue`, `failure-strategy=dead-letter-queue`).

**Step 15** (if SOAP_NEEDED): Migrate SOAP web services.

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
    <!-- properties — see references/ for details -->

- **SOAP client** (`@WebServiceRef`): replace with CXF client injection:
    <!-- java — see references/ for details -->
```properties
quarkus.cxf.client."partner".wsdl=http://partner.example.com/service?wsdl
quarkus.cxf.client."partner".client-endpoint-url=http://partner.example.com/service
```

- **Handler chains** (`@HandlerChain`): configure via `quarkus.cxf.endpoint."/path".handlers` property or retain `@HandlerChain` annotation (CXF supports it).

**Option B — Convert SOAP to REST** (larger effort — evaluate ROI):

- **Threshold**: if the `@WebService` has >5 operations or complex XSD types with deep inheritance, flag for human review rather than automated conversion.
- For simple services: map WSDL operations → JAX-RS `@Path` endpoints, convert complex types to JSON POJOs, replace `SOAPFault` with JAX-RS `ExceptionMapper`.
- **WARNING**: changing from SOAP to REST breaks existing clients. Only proceed if all consumers can be updated, or expose both temporarily during migration.

<!-- Detailed java example: see references/worked-examples-conditional.md -->

**Exit gate**: Run `./mvnw clean compile` — must succeed with zero errors. Additional verification:
- If JMS/MDB: verify message consumer/producer beans are CDI-discovered — check Quarkus build output for `SmallRye Reactive Messaging` or `Artemis JMS` initialization logs at startup
- If SECURITY: verify `@RolesAllowed`-annotated methods compile without additional imports
- If SOAP: verify CXF endpoint is generated — check for `quarkus-cxf` build step in compile output

### Phase 4: Testing & UI (ALWAYS — scope varies by flags)

Exit: `./mvnw clean test` passes. All migrated tests execute and pass.

**MANDATORY FIRST STEP in Phase 4 — JSF Namespace Verification:**
Run this command and fix ALL results before proceeding with any other Phase 4 work:
`grep -rn "jakarta\.face\." src/ | grep -v "jakarta\.faces\."`

For every match: change `jakarta.face.html` → `jakarta.faces.html`, `jakarta.face.core` → `jakarta.faces.core`, etc.
This error causes h: tags to fail silently — pages render but no JSF components work.
This check MUST pass (return zero results) before the phase exit gate.

**Step 16**: Migrate test framework. See `references/arquillian-to-quarkustest.md` for complete mapping.

- **Delete all commented-out test classes with no active @Test methods.** Detection: `for f in $(find src/test -name '*Test*.java'); do if ! grep -q '@Test' $f; then echo $f; fi; done`. These produce false coverage confidence.

- **Remove Arquillian dependencies** from pom.xml:
    <!-- xml — see references/ for details -->

- **Add Quarkus test dependencies**:
<!-- Detailed xml example: see references/worked-examples-conditional.md -->

- **`@RunWith(Arquillian.class)`** → `@QuarkusTest`:

<!-- Detailed java example: see references/worked-examples-conditional.md -->

- **`@Deployment` ShrinkWrap archive** → DELETE entirely. Quarkus boots the full application for `@QuarkusTest` — no deployment descriptor needed. The test classpath IS the deployment.
- **`@Inject` in tests** → works natively with `@QuarkusTest`. No special injection setup needed.
- **`@ArquillianResource URL`** → `@TestHTTPResource`:

<!-- Detailed java example: see references/worked-examples-conditional.md -->

- **Arquillian REST client tests** → REST Assured:

<!-- Detailed java example: see references/worked-examples-conditional.md -->

- **Test data setup**: `@Deployment` data init → `@TestTransaction` or `@QuarkusTestResource`:

<!-- Detailed java example: see references/worked-examples-conditional.md -->

- **TestContainers integration** (for real database testing):
```java
@QuarkusTest
@QuarkusTestResource(PostgresTestResource.class)
public class OrderRepositoryIT {
    // Uses real PostgreSQL via TestContainers
}
```

- **Mocking CDI beans** — use `@InjectMock` (Quarkus Mockito extension):

<!-- Detailed java example: see references/worked-examples-conditional.md -->

- **JUnit 4 → JUnit 5**: `@RunWith` → `@ExtendWith` (but not needed with `@QuarkusTest`). `Assert.assertEquals` → `Assertions.assertEquals`. `@Test(expected=...)` → `assertThrows(...)`. `@Before`/`@After` → `@BeforeEach`/`@AfterEach`. **Message reordering**: 3-arg `assertEquals("msg", expected, actual)` → `assertEquals(expected, actual, "msg")`.
- If SUREFIRE_DISABLED: re-enable surefire plugin — remove `<skip>true</skip>` or set `<skipTests>false</skipTests>`. Quarkus requires maven-surefire-plugin 3.x:
    <!-- xml — see references/ for details -->

- **Integration tests** (`*IT.java`): use `@QuarkusIntegrationTest` (runs against the built artifact — uber-jar or native):
<!-- Detailed java example: see references/worked-examples-conditional.md -->

- **Security in tests**: use `@TestSecurity` to inject roles without real auth:
<!-- Detailed java example: see references/worked-examples-conditional.md -->

- Delete `arquillian.xml`, `test-ds.xml`, and any Arquillian container adapter configs.
- Create `src/test/resources/application.properties` for test-specific overrides:
```properties
# Test datasource (H2 in-memory)
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:testdb
quarkus.hibernate-orm.schema-management.strategy=drop-and-create
quarkus.hibernate-orm.log.sql=true
```
- Add profile-based test configuration. Rather than a separate `src/test/resources/application.properties`, use `%test.` prefixed properties in the main `application.properties` for better visibility: `%test.quarkus.datasource.db-kind=h2`, `%test.quarkus.hibernate-orm.schema-management.strategy=drop-and-create`.
- Add a global `ExceptionMapper` for `ConstraintViolationException`. Instead of inline error handling in REST controllers, add a class implementing `ExceptionMapper<ConstraintViolationException>` annotated with `@Provider` to return consistent validation error responses:
<!-- Detailed java example: see references/worked-examples-conditional.md -->

**Step 17** (if JSF_NEEDED): Migrate UI.

Check for `.jsp` files in `src/main/webapp/` or `src/main/resources/META-INF/resources/`. Quarkus undertow does NOT support JSP compilation (no Jasper) — `.jsp` files will be served as raw source text. Detection: `find src/ -name '*.jsp'`. Options:
1. **Migrate to Facelets** (preferred): Convert `.jsp` to `.xhtml` Facelets templates. Replace JSP scriptlets (`<% %>`, `<%= %>`) with JSF EL expressions.
2. **Add Jasper** (quick fix): Add `implementation 'io.quarkus:quarkus-undertow-websockets'` — check Quarkus extension catalog for JSP support.
3. **Remove JSP pages**: If they are admin/utility pages not needed at runtime (e.g., config pages, error pages), replace with a JAX-RS endpoint or remove entirely.

See `references/jsf-to-qute.md` for complete JSF→Qute migration patterns including:
- .xhtml → .html Qute template conversion
- JSF backing beans → JAX-RS resources with Qute
- JSF component mappings (h:dataTable → Qute loops)  
- Form validation with Bean Validation
- Context path fixes for Quarkus root serving

Key actions:
- Remove JSF dependencies, add quarkus-rest-qute
- Convert .xhtml templates to .html in src/main/resources/templates/
- Replace JSF backing beans with JAX-RS resources
- Fix hardcoded context paths (WAR → root serving)
- Verify dotted @Named beans were fixed in Phase 2

<!-- Detailed html example: see references/worked-examples-conditional.md -->

- **Verify dotted @Named beans were fixed** (VERIFICATION ONLY — Phase 2 should have already fixed this): Run `grep -rn '@Named(".*\\..*")' src/main/java/` — this MUST return zero results. If any matches found, Phase 2 Step 7a was skipped or incomplete. EL resolvers in Quarkus/MyFaces interpret dots as property accessors, causing `#{public.track}` to be parsed as bean `public` → property `track` instead of bean `public.track`.

- **`@ManagedBean`/`@Named` backing bean** → `@ApplicationScoped` CDI bean + `Template` injection:

<!-- Detailed java example: see references/worked-examples-conditional.md -->

- **JSF navigation rules** (`faces-config.xml`) → direct JAX-RS endpoint paths with redirects (`Response.seeOther()`).
- **`@ViewScoped`** → `@RequestScoped` or stateless endpoint (Qute is stateless by design — no server-side view state).
- **`@FlowScoped`** → multi-step form patterns with hidden fields or URL path segments.
- **`f:setPropertyActionListener` targeting an implicit EL variable (`#{taskForDeletion}`) does NOT work with ArC CDI.** In original JSF managed beans, implicit EL maps allowed setting transient variables. In Quarkus/ArC, every EL-accessible object must be a CDI bean with explicit scope. Fix: Replace `f:setPropertyActionListener target="#{implicitVar}"` with direct method parameter passing: `action="#{controller.method(item)}"` using JSF EL method expressions.
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

**Step 18** (if BATCH_NEEDED): Migrate batch processing.

**Step: Migrate Jakarta Batch (JSR 352) — conditional on `BATCH_NEEDED`.**
Detect: `grep -rn 'jakarta.batch\|javax.batch\|@BatchProperty\|JobOperator' src/main/java/` or `find src/ -name '*.xml' -path '*META-INF/batch-jobs*'`.

Note on extension compatibility: `quarkus-jberet:2.10.0` (latest, Red Hat/IBM, stable) requires Quarkus 3.36+. For Quarkus 3.33 LTS: no compatible version available — replace batch job classes with CDI-based alternatives (see CDI event processing or quarkus-scheduler). Check https://quarkus.io/extensions/io.quarkiverse.jberet/quarkus-jberet/ for the latest compatible version.
Changes required:
- `META-INF/batch-jobs/*.xml` stays in place (no changes needed)
- `@BatchProperty`, `ItemReader`, `ItemProcessor`, `ItemWriter` annotations stay unchanged
- Replace `BatchRuntime.getJobOperator()` JNDI lookup with `@Inject JobOperator jobOperator`
- `JobOperator.start(jobName, props)` API remains the same
Exit gate: `./mvnw clean compile` passes, `BatchRuntime` import replaced.

- **Simple scheduled jobs** (EJB `@Schedule`): already handled in Phase 2 Step 9 via `@Scheduled`. Verify migration is complete.
- **JSR 352 Batch** (complex multi-step jobs with `ItemReader`/`ItemProcessor`/`ItemWriter`):

**Option A — JBeret extension** (preserves JSR 352 API - requires Quarkus 3.36+):
```xml
<dependency>
    <groupId>io.quarkiverse.jberet</groupId>
    <artifactId>quarkus-jberet</artifactId>
    <version>2.10.0</version>
</dependency>
```
  - Batch job XML files (`META-INF/batch-jobs/*.xml`) work unchanged.
  - `@BatchProperty` → stays as-is.
  - `JobOperator` → inject via `@Inject JobOperator`.
  - **Note**: JBeret extension may have limited native compilation support — verify if native build is required.

**Option B — Redesign to Quarkus-native** (for simpler jobs):
  - Convert `ItemReader`/`ItemProcessor`/`ItemWriter` → `@ApplicationScoped` bean with `@Scheduled` and chunked processing logic:

<!-- Detailed java example: see references/worked-examples-conditional.md -->

- **Checkpoint/restart semantics**: if the original batch job relies on JSR 352 checkpointing, preserve with JBeret (Option A). Pure Quarkus-native redesign loses automatic checkpoint/restart — implement manually if needed.

**Step: Migrate JavaMail — conditional on `MAIL_NEEDED`**
Detection: `grep -rn 'jakarta.mail\|javax.mail\|@Resource.*mail\|Session.getInstance' src/main/java/`
Add extension: `io.quarkus:quarkus-mailer`
Migration:
- Remove `@Resource(lookup="java:jboss/mail/Default") Session session` → `@Inject Mailer mailer`
- Replace `new MimeMessage(session)` / `Transport.send(msg)` with `mailer.send(Mail.withText(to, subject, body))`
- Map JNDI mail properties → application.properties: `quarkus.mailer.host`, `quarkus.mailer.port`, `quarkus.mailer.from`
- For HTML email: `Mail.withHtml(to, subject, htmlBody)`

**Exit gate**: Run `./mvnw clean test` — all tests must pass. Additional verification:
- No `@RunWith(Arquillian.class)` remaining: `grep -rn "Arquillian" src/test/` — must return empty
- No ShrinkWrap imports remaining: `grep -rn "shrinkwrap\|ShrinkWrap" src/test/` — must return empty
- No JUnit 4 `@Test` (org.junit.Test) remaining: `grep -rn "import org.junit.Test" src/test/` — must return empty (should be `org.junit.jupiter.api.Test`)
- Test count: verify migrated test count matches original (no tests dropped)

### Phase 5: Deployment & Verification (ALWAYS)

Exit: `./mvnw clean verify` passes AND Docker image builds AND `/q/health` returns UP.

**Step 19**: Generate containerization artifacts.

- **Remove old app server Dockerfiles**: Delete root `Dockerfile` (Payara/WildFly/Liberty), `post-boot-commands.asadmin`, `server.xml`, `glassfish-resources.xml`, and other app-server-specific deployment artifacts. Verify: `find . -maxdepth 1 -name "Dockerfile" -o -name "*.asadmin" -o -name "server.xml"` — must return empty.
- When the repo has a legacy `Containerfile` (or `Dockerfile`) targeting the old app server (WildFly, JBoss), rename it to `Containerfile.legacy` or remove it. Leaving the old Containerfile creates confusion — CI/CD pipelines may pick it up instead of the new `src/main/docker/Dockerfile.jvm`.
- Integration test modules targeting the old app server (e.g., `integration-tests/wildfly/`, `integration-tests/tomcat/`) should be removed or moved to a `legacy/` directory.

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

<!-- Detailed dockerfile example: see references/worked-examples-conditional.md -->

- **Alternative** — simplified JVM Dockerfile (non-Red Hat base):
    <!-- dockerfile — see references/ for details -->

- Create `src/main/docker/Dockerfile.native` (multi-stage native build — no local GraalVM needed):

<!-- Detailed dockerfile example: see references/worked-examples-conditional.md -->

- Create `src/main/docker/Dockerfile.native-micro` (pre-built native binary — smallest image):

    <!-- dockerfile — see references/ for details -->

- **Update docker-compose.yml**: Replace app-server-specific environment variables with Quarkus naming convention. Use `QUARKUS_DATASOURCE_DB_KIND`, `QUARKUS_DATASOURCE_JDBC_URL`, `QUARKUS_DATASOURCE_USERNAME`, `QUARKUS_HIBERNATE_ORM_SCHEMA_MANAGEMENT_STRATEGY`, etc. Update `dockerfile:` path to `src/main/docker/Dockerfile.jvm`. Add a health check to the service:

<!-- Detailed yaml example: see references/worked-examples-conditional.md -->

**Docker container database configuration:**
- The default Quarkus profile in containers is `prod` — ensure `%prod.quarkus.hibernate-orm.schema-management.strategy` is set appropriately
- For dev/demo containers with embedded H2, pass environment variables:
  - QUARKUS_DATASOURCE_JDBC_URL=jdbc:h2:mem:appname;DB_CLOSE_DELAY=-1
  - QUARKUS_HIBERNATE_ORM_SCHEMA_MANAGEMENT_STRATEGY=drop-and-create
  - QUARKUS_HIBERNATE_ORM_SQL_LOAD_SCRIPT=<load-script-filename>
- Or define a `%container` profile in `application.properties` with in-memory H2 configuration

- If REFLECTION_HEAVY (detected by: heavy use of Jackson polymorphism, JAXB, `Class.forName()`, or frameworks that rely on runtime reflection): add `@RegisterForReflection` to affected classes:

**Native image compatibility**: Add @RegisterForReflection for classes needing reflection (JSON serialization targets, etc.)

- **Verify Dockerfile.jvm matches the configured `quarkus.package.jar.type`.** If `uber-jar`: COPY target/app-runner.jar, set ENTRYPOINT to run the JAR. If `fast-jar` (default): use the quarkus-app/ directory layout. If `legacy-jar`: COPY target/app.jar. Mismatch causes 'file not found' at container startup.

**Step 20**: Configure health and observability.

Add health and metrics extensions: `quarkus-smallrye-health`, `quarkus-micrometer-registry-prometheus`

Configuration examples: See `references/quarkus-extension-catalog.md`

- Health endpoints are automatically available:
  - `/q/health` — combined health check (liveness + readiness)
  - `/q/health/live` — liveness probe
  - `/q/health/ready` — readiness probe
  - `/q/metrics` — Prometheus metrics

- Add custom health check if datasource or external service validation is needed:
<!-- Detailed java example: see references/worked-examples-conditional.md -->

- Configure production profile in `application.properties`:
<!-- Detailed properties example: see references/worked-examples-conditional.md -->

- **Kubernetes/OpenShift deployment** (optional): generate manifests:
```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-kubernetes</artifactId>
</dependency>
```
Kubernetes manifest generation example: See `references/quarkus-extension-catalog.md`

**Step 21**: Final verification scan.

See `references/phase0-detection-flags.md` for complete validation commands including:
- Build verification (`./mvnw clean verify` passes)
- Legacy import/annotation/descriptor scans (must return ZERO results)
- Docker image build verification
- Container smoke test

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

**Essential validations** (must all pass):

1. `./mvnw clean verify` passes with all tests green
2. No javax.* EE imports remain in src/main/java/ (excluding Java SE packages)  
3. No EJB annotations remain (@Stateless, @MessageDriven, @EJB, @TransactionAttribute)
4. No app server descriptors exist (jboss-web.xml, persistence.xml, etc.)
5. All REST endpoints respond with same paths/methods/responses as original
6. All @Inject points resolve at build time — no unsatisfied dependency errors
7. Docker image builds: `docker build -f src/main/docker/Dockerfile.jvm -t test .` succeeds
8. Application starts in <10 seconds: `Quarkus started in X.XXXs` log line  
9. Health check returns UP: `curl http://localhost:8080/q/health`
10. All original test assertions pass — no @Disabled tests as migration shortcuts
11. No servlet with dangerous operations (`Runtime.halt`, `System.exit`, `exec()`) without `@RolesAllowed` protection
12. No old app-server Containerfile/Dockerfile at project root — only `src/main/docker/Dockerfile.jvm` should exist
13. No old integration test modules targeting WildFly/TomEE/Liberty in the active build
14. JSF namespace typo check passes: `grep -rn "jakarta\.face\." src/ | grep -v "jakarta\.faces\."` returns zero results

Additional validations: See `references/phase0-detection-flags.md`

## Common Pitfalls

See `references/arc-limitations.md` for ArC-specific issues. Additional common migration failures:

- **Bean removed by unused-bean optimization**: set `quarkus.arc.remove-unused-beans=false` during migration, then re-enable after stabilization.
- **Final class cannot be proxied**: add `quarkus.arc.transform-unproxyable-classes=true` or remove `final` keyword.
- **Missing no-arg constructor**: normal-scoped beans (`@ApplicationScoped`, `@RequestScoped`) need a no-arg constructor for ArC proxy generation.
- **Namespace migration incomplete**: mixed `javax.*` and `jakarta.*` in same file causes compilation errors. Always migrate ALL EE namespaces in one pass per file.
- **Self-invocation bypasses interceptors**: `@Transactional` on a method called internally (`this.method()`) is not intercepted. Extract to a separate bean or use `Arc.container().instance()`.

## Tips

- [2026-06] **Latest Cargo Tracker v5 review findings integrated.** Key fixes: SSE lazy initialization pattern for @ApplicationScoped beans, method-level @Transactional rules, dead test cleanup detection, and Dockerfile JAR type verification.
- [2026-06] **Always explicitly set the Quarkus version to the LTS.** Without explicit `quarkus.platform.version=3.33.2`, the ATX agent may use an older release. Confirm in pom.xml: `<quarkus.platform.version>3.33.2</quarkus.platform.version>` or Gradle: `id 'io.quarkus' version '3.33.2'`.
- [2026-06] **`@RolesAllowed` without a security extension is silently ignored.** Adding `@RolesAllowed` to a method or class does nothing unless a Quarkus security extension (`quarkus-security` + an identity provider) is also in the dependencies. Always verify: `grep -rn 'quarkus-security\|quarkus-oidc\|quarkus-elytron' pom.xml build.gradle`.
- [2026-03] Quarkus 3.33 is the current LTS (Long Term Support) version. LTS releases are supported for 12 months. Use `quarkus update` CLI command to update existing Quarkus apps between versions.
- [2026-06] **WildFly/JBoss system properties** (`jboss.node.name`, `jboss.server.config.dir`, etc.) do not exist in Quarkus. Replace with `@ConfigProperty` using Quarkus-native keys or custom properties.
- [2026-06] **Dots in @Named bean names break EL resolution on Quarkus/MyFaces.** CDI beans with `@Named("foo.bar")` work on Payara/GlassFish/WildFly but fail on Quarkus because the Expressly EL resolver treats the dot as a property accessor. `#{public.track.trackingId}` is parsed as "bean `public` → property `track` → property `trackingId`" instead of "bean `public.track` → property `trackingId`". Fix: rename to camelCase (`@Named("fooBar")`) and update all EL references in `.xhtml` templates.
- [2026-06] `quarkus.hibernate-orm.database.generation` is deprecated since Quarkus 3.31. Use `quarkus.hibernate-orm.schema-management.strategy` instead. Values: `drop-and-create`, `update`, `none`, `validate`.
- [2026-06] **WAR→Quarkus context path removal**: WAR apps deployed to `/app-name` context paths must have all hardcoded context references updated. Quarkus serves at root `/` by default. Replace `/old-context/path` → `/path` in templates and JavaScript. `#{request.contextPath}` resolves to empty string in Quarkus.
- [2026-06] `-XX:+UseContainerSupport` has been enabled by default since Java 10 (JDK-8146115). Do NOT include it in Dockerfiles for Java 17+ — it's a no-op and adds clutter.
- [2026-06] **Quarkus undertow does not support JSP compilation.** `.jsp` files are served as raw text (Jasper is not included). Any JSP pages must be migrated to Facelets (`.xhtml`) or converted to JAX-RS endpoints. Detection: `find src/ -name '*.jsp'`
- [2026-06] **JSF/MyFaces apps cannot use GraalVM native compilation.** JSF relies on runtime reflection, dynamic proxies, and classpath scanning that are fundamentally incompatible with GraalVM's closed-world assumption. Only generate Dockerfile.jvm for apps with JSF_NEEDED=true. Non-JSF apps (pure JAX-RS + CDI) work well in native mode with sub-second startup.
- [2026-06] **`import.sql` table names must match JPA entity names.** By default Hibernate maps entity `Person` to table `PERSON` (H2 is case-insensitive, but `INSERT INTO User` fails if the entity class is `Person`). Always verify: `SELECT table_name FROM information_schema.tables` or check `@Table(name=...)` annotations.
- [2026-06] **`f:setPropertyActionListener` with implicit EL variables fails in Quarkus/ArC.** The pattern `<f:setPropertyActionListener target="#{someVar}" value="#{item}" />` relies on JSF managed beans' implicit EL scope behavior. In Quarkus with ArC, use direct method parameter expressions instead: `action="#{controller.delete(item)}"`.
- [2026-06] Quarkus Dev UI at `/q/dev-ui` provides build-time bean inspection, config editor, and extension management — invaluable during migration debugging.
- [2026-06] `quarkus.log.category."io.quarkus.arc".level=DEBUG` logs all bean discovery decisions — helps diagnose missing or removed beans.

