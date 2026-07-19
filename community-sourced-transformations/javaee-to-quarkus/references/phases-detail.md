# Migration Phases - Detailed Guide

## Phase 0: Project Analysis (ALWAYS)

**Exit:** Feature flags set, transformation roadmap generated

**Key Detection Commands:**
```bash
# Critical blockers
grep -rn '@Remote\|@RemoteHome' src/ # → HALT if found
grep -rn 'extends javax.ejb.EntityBean\|implements EntityBean' src/ # → HALT if found (EJB2_ENTITY_BEAN_FOUND)
grep -rn 'new InitialContext\|context\.lookup' src/main/java/ # → P0 blocker

# Feature flags  
grep -rn '@Stateless\|@Stateful\|@Singleton' src/ # → EJB_NEEDED
grep -rn '@Entity\|@PersistenceContext\|javax\.persistence\|jakarta\.persistence' src/main/java/ # → JPA_NEEDED
grep -rn '@MessageDriven\|javax.jms' src/ # → JMS_NEEDED
grep -rn '@RolesAllowed' src/ # → SECURITY_NEEDED
find src/ -name '*.xhtml' # → JSF_NEEDED
find src/ -name '*.jsp' # → JSP migration required
```

Scan for migration complexity flags:
- `REMOTE_EJB_DETECTED` → **BLOCKER** - halt migration
- `EJB2_ENTITY_BEAN_FOUND` → **BLOCKER** - halt migration (EJB 2.x Entity Beans not supported)
- `EJB_NEEDED` → Phase 2 required  
- `JPA_NEEDED` → Phase 2 required (JDBC extension derivation in Phase 1)
- `JMS_NEEDED`, `JSF_NEEDED`, `SECURITY_NEEDED` → respective phases
- `HAS_JNDI_LOOKUPS` → **P0 BLOCKER** - replace before proceeding

## Phase 1: Build System & Namespace (ALWAYS)

**Exit:** `./mvnw clean compile -Dmaven.test.skip=true` passes

### Multi-module Consolidation
**Multi-module consolidation** (if needed): Merge all `src/main/java` → single module, remove `<modules>` from POM

### Quarkus Foundation Setup
**Quarkus foundation:**
```xml
<quarkus.platform.version>3.33.2</quarkus.platform.version>
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-bom</artifactId></dependency>
<packaging>jar</packaging> <!-- was: war -->
<plugin><groupId>io.quarkus</groupId><artifactId>quarkus-maven-plugin</artifactId></plugin>
```

### JDBC Extension Derivation

When `JPA_NEEDED=true`, derive the JDBC extension from persistence.xml:

| persistence.xml Signal | Quarkus Extension |
|---|---|
| `jta-data-source` contains `H2` or driver `org.h2.Driver` | `quarkus-jdbc-h2` |
| `jta-data-source` contains `Postgres` or driver `org.postgresql.Driver` | `quarkus-jdbc-postgresql` |
| `jta-data-source` contains `MySQL` or driver `com.mysql.cj.jdbc.Driver` | `quarkus-jdbc-mysql` |
| `jta-data-source` contains `MSSQL`/`SQLServer` or driver `com.microsoft.sqlserver.jdbc.SQLServerDriver` | `quarkus-jdbc-mssql` |
| `jta-data-source` contains `Oracle` or driver `oracle.jdbc.OracleDriver` | `quarkus-jdbc-oracle` |

Add the `quarkus-jdbc-{kind}` dependency in the same POM task that adds the Quarkus BOM.

### Namespace Migration
**Namespace migration** (javax→jakarta for EE packages only):
```
javax.persistence → jakarta.persistence
javax.inject → jakarta.inject  
javax.enterprise → jakarta.enterprise
javax.ws.rs → jakarta.ws.rs
javax.transaction → jakarta.transaction
```
**DO NOT replace:** javax.crypto, javax.net, javax.sql, javax.xml (Java SE)

### Configuration Migration
**Configuration migration:**
- persistence.xml → quarkus.datasource.*, quarkus.hibernate-orm.* in application.properties
- web.xml → quarkus.http.* properties
- Remove: all *-ds.xml, jboss-web.xml, beans.xml, app server descriptors

### Descriptor Removal Warning
**IMPORTANT:** When removing app server descriptors, preserve the following files:
- `src/main/resources/import.sql` — Quarkus-native data initialization file loaded by Hibernate ORM at startup
- `src/main/resources/data.sql` — same role as import.sql
- Any `.sql` files referenced in tests (`src/test/resources/`)

Only remove: `persistence.xml`, `web.xml`, `jboss-web.xml`, `jboss-ejb3.xml`, `jboss-app.xml`, `*-ds.xml`, `beans.xml`

## Phase 2: Core Migration (CONDITIONAL - if EJB detected)

**Exit:** `./mvnw clean compile` passes.

**Migration Comment Rule**: MIGRATION comments must use plain English, not annotation names (e.g., "Converted stateless session bean" not "Removed @Stateless").

### Step: Rename Dotted @Named Bean Names
**Step: Rename dotted @Named bean names BEFORE any other CDI migration.** EL resolvers interpret dots as property accessors — `#{public.track}` is parsed as bean `public` → property `track`, not as a single bean named `public.track`. Detection: `grep -rn '@Named(".*\\..*")' src/main/java/`. Fix: rename to camelCase and update ALL EL references.

### Step 6: Migrate EJBs to CDI Beans

**CRITICAL THREAD-SAFETY FIX**: Beans with mutable instance state MUST be `@RequestScoped`, NOT `@ApplicationScoped`. Detection: `grep -rn '@ApplicationScoped' src/main/java/ -A5 | grep -i 'inSession\|inGlobal\|mutable\|boolean.*=.*false'`.

- @Stateless → @ApplicationScoped + @Transactional (class-level)
- @Singleton → @ApplicationScoped (add StartupEvent observer if @Startup present)
- @Stateful → @SessionScoped (evaluate if stateful pattern still needed)
- @EJB → @Inject (constructor injection preferred)
- @Remove → remove annotation only; method becomes regular business method. Do NOT convert to @PreDestroy (which fires on bean destruction, not after a business call). Bean lifecycle in Quarkus is managed by @SessionScoped/HTTP session expiry.
- Remove @LocalBean, @Local, @Remote annotations
- Add no-arg constructor for normal-scoped beans (ArC proxy requirement)

**@Transactional mapping:** Apply class-level `@Transactional` when the original `@Stateless` bean uses EntityManager in multiple methods (read AND write). This preserves EJB default REQUIRED semantics where ALL methods participate in a transaction. Use method-level `@Transactional` only when a specific method needs different propagation (REQUIRES_NEW, NOT_SUPPORTED, etc.).

**Rationale:** Quarkus Hibernate ORM requires an active transaction for `EntityManager.find()`, `TypedQuery.getResultList()`, and other read operations — not just writes. Applying `@Transactional` only to write methods leaves read methods broken at runtime with `TransactionRequiredException`.

### Step 7: Migrate JPA Configuration

- Remove persistence.xml (config moved to application.properties in Phase 1)
- Keep entity annotations as-is — jakarta.persistence.* works unchanged  
- @PersistenceContext → @Inject EntityManager
- Multi-datasource: Use named datasources + @PersistenceUnit("name") qualifier

### Step 8: Adjust JAX-RS Resources

- `@Path`, `@GET`, `@POST`, etc. stay unchanged
- Replace JAX-RS `ClientBuilder` with `@RegisterRestClient` interface + `@RestClient` injection
- Remove the JAX-RS `Application` subclass — if it has a custom `@ApplicationPath("/api")`, add the base path to application.properties:
  - For `quarkus-rest` (RESTEasy Reactive): `quarkus.rest.path=/api`
  - For `quarkus-resteasy` (Classic): `quarkus.resteasy.path=/api`
  - **WARNING**: Using the wrong property key silently has no effect — the base path reverts to `/` without any error.
- JAX-RS `@Provider`-annotated classes work unchanged
- Health-check servlets → remove entirely (quarkus-smallrye-health provides `/q/health` automatically via the `HealthCheck` interface and its `call()` method)

### Cross-Task Dependency: @Asynchronous Future→Uni

When converting `@Asynchronous` EJB methods from `Future<T>` to `Uni<T>`:
1. Scan all callers for `Future<T>` assignments (variable declarations, return types)
2. Schedule caller task AFTER callee task, or update callers in the same task
3. Flag `Future<T>` usages referencing the converted method as a cross-task dependency at planning time

Failing to update callers causes compile errors that surface only after both tasks complete.

### Additional Migration Steps

**Step 9** (if SCHEDULER_NEEDED): Migrate EJB Timers to Quarkus Scheduler
**Step 10** (if WEBSOCKET_NEEDED): Migrate WebSockets  
**Step 12** (if HAS_JNDI): Replace JNDI lookups
**Step 14** (if JMS_NEEDED): Migrate messaging

**Phase 3 messaging validation:** After all MDB→@Incoming migrations complete:
```bash
# Must return empty — all JMS/MDB annotations replaced
grep -rn '@MessageDriven\|javax\.jms\|jakarta\.jms' src/main/java/
```
**Step 15** (if SOAP_NEEDED): Migrate SOAP web services

## Phase 4: Testing & UI (ALWAYS)

**Exit:** `./mvnw clean test` passes

### Step 16: Migrate Test Framework
- Remove Arquillian dependencies, add Quarkus test dependencies
- @RunWith(Arquillian.class) → @QuarkusTest
- Delete @Deployment ShrinkWrap archive
- @ArquillianResource URL → @TestHTTPResource
- Convert to JUnit 5
- JUnit 4→5 assertion argument order: all 3-argument assertions move the message String from FIRST to LAST position (see arquillian-to-quarkustest.md)
- Prerequisite: `mkdir -p src/main/resources/META-INF/resources/` (for JSF resource migration)

### Step 17 (if JSF_NEEDED): Migrate UI

**Option A** — Convert JSF to Qute templates
**Option B** — MyFaces extension (preserves JSF with minimal changes)

### Step 18 (if BATCH_NEEDED): Migrate batch processing
### Additional conditional steps based on detected features

## Phase 5: Deployment & Verification (ALWAYS)

**Exit:** `./mvnw clean verify` passes AND Docker image builds AND `/q/health` returns UP

### Step 19: Generate Containerization Artifacts
- Prerequisite: `mkdir -p src/main/docker/`
- Remove old app server Dockerfiles
- Generate Quarkus Dockerfiles (JVM, native if compatible)
- Update docker-compose.yml with Quarkus configuration

### Step 20: Configure Health and Observability
- Add `quarkus-smallrye-health` extension
- Custom health checks implement `org.eclipse.microprofile.health.HealthCheck` interface → implement `public HealthCheckResponse call()` method
- Configure production profile
- Set up health endpoints

### Step 21: Final Verification Scan
- Build verification
- Legacy import/annotation/descriptor scans
- Docker image build verification
- Container smoke test

## Phase 5: Container Image Replacement Table
| Old image pattern | Quarkus replacement |
|-------------------|---------------------|
| quay.io/wildfly/wildfly:* | registry.access.redhat.com/ubi8/openjdk-17:latest |
| jboss/wildfly | quay.io/quarkus/ubi-quarkus-native-image:* |
| /eap[0-9]* | registry.access.redhat.com/jboss-eap-7/eap74-openjdk11-openshift-rhel8 → migrate to UBI |

Scan command: `grep -rEn 'wildfly|jboss|eap[0-9]' charts/ k8s/ deploy/ helm/ README.md **/*.yaml`
