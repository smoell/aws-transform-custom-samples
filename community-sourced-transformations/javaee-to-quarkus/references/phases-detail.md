# Migration Phases - Detailed Guide

## Phase 0: Project Analysis (ALWAYS)

**Exit:** Feature flags set, transformation roadmap generated

**Key Detection Commands:**
```bash
# Critical blockers
grep -rn '@Remote\|@RemoteHome' src/ # → HALT if found
grep -rn 'new InitialContext\|context\.lookup' src/main/java/ # → P0 blocker

# Feature flags  
grep -rn '@Stateless\|@Stateful\|@Singleton' src/ # → EJB_NEEDED
grep -rn '@MessageDriven\|javax.jms' src/ # → JMS_NEEDED
grep -rn '@RolesAllowed' src/ # → SECURITY_NEEDED
find src/ -name '*.xhtml' # → JSF_NEEDED
find src/ -name '*.jsp' # → JSP migration required
```

Scan for migration complexity flags:
- `REMOTE_EJB_DETECTED` → **BLOCKER** - halt migration
- `EJB_NEEDED` → Phase 2 required  
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

## Phase 2: Core Migration (CONDITIONAL - if EJB detected)

**Exit:** `./mvnw clean compile` passes.

**Migration Comment Rule**: MIGRATION comments must use plain English, not annotation names (e.g., "Converted stateless session bean" not "Removed @Stateless").

### Step: Rename Dotted @Named Bean Names
**Step: Rename dotted @Named bean names BEFORE any other CDI migration.** EL resolvers interpret dots as property accessors — `#{public.track}` is parsed as bean `public` → property `track`, not as a single bean named `public.track`. Detection: `grep -rn '@Named(".*\\..*")' src/main/java/`. Fix: rename to camelCase and update ALL EL references.

### Step 6: Migrate EJBs to CDI Beans

**CRITICAL THREAD-SAFETY FIX**: Beans with mutable instance state MUST be `@RequestScoped`, NOT `@ApplicationScoped`. Detection: `grep -rn '@ApplicationScoped' src/main/java/ -A5 | grep -i 'inSession\|inGlobal\|mutable\|boolean.*=.*false'`.

- @Stateless → @ApplicationScoped + @Transactional (only for persistence operations)
- @Singleton → @ApplicationScoped (add StartupEvent observer if @Startup present)
- @Stateful → @SessionScoped (evaluate if stateful pattern still needed)
- @EJB → @Inject (constructor injection preferred)
- Remove @LocalBean, @Local, @Remote annotations
- Add no-arg constructor for normal-scoped beans (ArC proxy requirement)

**@TransactionAttribute mapping**: Apply @Transactional at method level, not class level. Remove class-level `@Transactional` from service beans. Add `@Transactional` only to methods that write to the database.

### Step 7: Migrate JPA Configuration

- Remove persistence.xml (config moved to application.properties in Phase 1)
- Keep entity annotations as-is — jakarta.persistence.* works unchanged  
- @PersistenceContext → @Inject EntityManager
- Multi-datasource: Use named datasources + @PersistenceUnit("name") qualifier

### Step 8: Adjust JAX-RS Resources

- `@Path`, `@GET`, `@POST`, etc. stay unchanged
- Replace JAX-RS `ClientBuilder` with `@RegisterRestClient` interface + `@RestClient` injection
- Remove the JAX-RS `Application` subclass
- JAX-RS `@Provider`-annotated classes work unchanged

### Additional Migration Steps

**Step 9** (if SCHEDULER_NEEDED): Migrate EJB Timers to Quarkus Scheduler
**Step 10** (if WEBSOCKET_NEEDED): Migrate WebSockets  
**Step 12** (if HAS_JNDI): Replace JNDI lookups
**Step 14** (if JMS_NEEDED): Migrate messaging
**Step 15** (if SOAP_NEEDED): Migrate SOAP web services

## Phase 4: Testing & UI (ALWAYS)

**Exit:** `./mvnw clean test` passes

### Step 16: Migrate Test Framework
- Remove Arquillian dependencies, add Quarkus test dependencies
- @RunWith(Arquillian.class) → @QuarkusTest
- Delete @Deployment ShrinkWrap archive
- @ArquillianResource URL → @TestHTTPResource
- Convert to JUnit 5

### Step 17 (if JSF_NEEDED): Migrate UI

**Option A** — Convert JSF to Qute templates
**Option B** — MyFaces extension (preserves JSF with minimal changes)

### Step 18 (if BATCH_NEEDED): Migrate batch processing
### Additional conditional steps based on detected features

## Phase 5: Deployment & Verification (ALWAYS)

**Exit:** `./mvnw clean verify` passes AND Docker image builds AND `/q/health` returns UP

### Step 19: Generate Containerization Artifacts
- Remove old app server Dockerfiles
- Generate Quarkus Dockerfiles (JVM, native if compatible)
- Update docker-compose.yml with Quarkus configuration

### Step 20: Configure Health and Observability
- Add health and metrics extensions
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