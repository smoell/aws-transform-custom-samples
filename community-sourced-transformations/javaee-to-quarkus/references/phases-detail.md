# Migration Phases — Detailed Guide

## Phase 0: Project Analysis (ALWAYS)

**Exit:** Feature flags set, transformation roadmap generated.

**Detection commands:**
```bash
# Critical blockers
grep -rn '@Remote\|@RemoteHome' src/                                              # → HALT
grep -rn 'extends javax.ejb.EntityBean\|implements EntityBean' src/               # → HALT (EJB2_ENTITY_BEAN_FOUND)
grep -rn 'new InitialContext\|context\.lookup' src/main/java/                     # → P0 blocker
grep -rn 'implements Extension\|META-INF/services/javax.enterprise.inject.spi.Extension' src/  # → WARNING (CDI Portable Extension)
# Feature flags
grep -rn '@Stateless\|@Stateful\|@Singleton' src/                                 # → EJB_NEEDED
grep -rn '@Entity\|@PersistenceContext\|javax\.persistence\|jakarta\.persistence' src/main/java/  # → JPA_NEEDED
grep -rn '@NotNull\|@Size\|@Email\|@Positive\|@Min\|@Max\|@Pattern\|@Valid' src/main/java/  # → VALIDATION_NEEDED
grep -rn '@MessageDriven\|javax.jms' src/                                         # → JMS_NEEDED
grep -rn '@RolesAllowed' src/                                                     # → SECURITY_NEEDED
find src/ -name '*.xhtml'   # → JSF_NEEDED     find src/ -name '*.jsp'   # → JSP migration
```

**Complexity flags:** `REMOTE_EJB_DETECTED`/`EJB2_ENTITY_BEAN_FOUND` → BLOCKER (halt); `CDI_PORTABLE_EXTENSION` → WARNING (manual convert to BuildExtension); `EJB_NEEDED`/`JPA_NEEDED` → Phase 2 (JPA also drives JDBC-extension derivation in Phase 1); `VALIDATION_NEEDED` → add `quarkus-hibernate-validator` in Phase 2 (NOT transitive from quarkus-hibernate-orm); `JMS_NEEDED`/`JSF_NEEDED`/`SECURITY_NEEDED` → respective phases; `HAS_JNDI_LOOKUPS` → P0 blocker (replace before proceeding).

## Phase 1: Build System & Namespace (ALWAYS)

**Exit:** `./mvnw clean compile -Dmaven.test.skip=true` passes.

**Multi-module consolidation** (if needed): merge all `src/main/java` → single module, remove `<modules>` from POM.

### Quarkus Foundation — Maven
```xml
<quarkus.platform.version>3.33.2</quarkus.platform.version>
<dependencyManagement><dependencies>
  <dependency>
    <groupId>io.quarkus.platform</groupId><artifactId>quarkus-bom</artifactId>
    <version>${quarkus.platform.version}</version><type>pom</type><scope>import</scope>
  </dependency>
</dependencies></dependencyManagement>
<packaging>jar</packaging> <!-- was: war -->
<plugin><groupId>io.quarkus</groupId><artifactId>quarkus-maven-plugin</artifactId><version>${quarkus.platform.version}</version></plugin>
```
- BOM groupId is `io.quarkus.platform` (NOT `io.quarkus`); the maven-plugin groupId is `io.quarkus` (core artifact — correct).
- **WAR→JAR cleanup**: remove `failOnMissingWebXml`, `version.wildfly`, `version.microprofile` properties and `maven-war-plugin` blocks.
- **Always add in the Phase 1 pom task**: `quarkus-smallrye-health` (provides `/q/health`, self-activating) + the JDBC extension (derived below).

### Quarkus Foundation — Gradle
```groovy
plugins { id 'java'; id 'io.quarkus' version '3.33.2' }
dependencies {
    implementation enforcedPlatform("io.quarkus.platform:quarkus-bom:3.33.2")
    implementation 'io.quarkus:quarkus-arc'
    implementation 'io.quarkus:quarkus-rest'   // further extensions without version (BOM-managed)
}
```
(Kotlin DSL: `id("io.quarkus") version "3.33.2"` + `enforcedPlatform("io.quarkus.platform:quarkus-bom:3.33.2")`.)

**Gradle wrapper bootstrap**: if `gradlew` is absent — `gradle wrapper --gradle-version=8.10 --no-daemon`, then verify `gradlew` + `gradle/wrapper/*`. Do NOT curl/wget the wrapper JAR — the system `gradle` generates it locally. **If BOTH `pom.xml` and `build.gradle` exist, `build.gradle` is the source of truth.**

### JDBC Extension Derivation (when JPA_NEEDED)

| persistence.xml signal (jta-data-source / driver) | Extension |
|---|---|
| H2 / `org.h2.Driver` | `quarkus-jdbc-h2` |
| Postgres / `org.postgresql.Driver` | `quarkus-jdbc-postgresql` |
| MySQL / `com.mysql.cj.jdbc.Driver` | `quarkus-jdbc-mysql` |
| MSSQL·SQLServer / `com.microsoft.sqlserver.jdbc.SQLServerDriver` | `quarkus-jdbc-mssql` |
| Oracle / `oracle.jdbc.OracleDriver` | `quarkus-jdbc-oracle` |

Add `quarkus-jdbc-{kind}` in the same POM task as the BOM.

### Namespace Migration (javax→jakarta, EE packages only)
`javax.persistence`→`jakarta.persistence`, `javax.inject`→`jakarta.inject`, `javax.enterprise`→`jakarta.enterprise`, `javax.ws.rs`→`jakarta.ws.rs`, `javax.transaction`→`jakarta.transaction`. **DO NOT replace** `javax.crypto`, `javax.net`, `javax.sql`, `javax.xml` (Java SE).
```bash
# Find ALL affected files (don't rely on explicit lists)
grep -rl 'import javax\.' src/ | grep -v 'javax\.sql\|javax\.crypto\|javax\.security\.auth\|javax\.net\|javax\.naming'
```

### Context-Root Extraction
From any app-server descriptor (`jboss-web.xml`, `glassfish-web.xml`, `payara-resources.xml`, `weblogic.xml`):
```properties
quarkus.http.root-path=/myapp
quarkus.http.non-application-root-path=/q   # ALWAYS pair — keeps health/metrics off {root-path}/q
```

### Configuration Migration
- persistence.xml → `quarkus.datasource.*` / `quarkus.hibernate-orm.*`; web.xml → `quarkus.http.*`.
- **Atomic-ownership rule**: create `application.properties` in ONE task covering ALL Phase 1–3 properties (HTTP root-path, REST path, datasource, messaging channels both directions, `quarkus.arc.remove-unused-beans=false`). Partial files cause `@QuarkusTest` startup failures.

### Descriptor Removal — Complete List
**Delete without replacement** (no Quarkus equivalent): `jboss-web.xml`, `jboss-ejb3.xml`, `jboss-app.xml`, `weblogic-ejb-jar.xml`, `weblogic.xml`, `glassfish-ejb-jar.xml`, `payara-resources.xml`, `sun-web.xml`, `sun-ejb-jar.xml`, `*-ds.xml`, `web.xml`, `persistence.xml` (config → application.properties), and `beans.xml` (UNLESS it has non-empty `<interceptors>`/`<alternatives>`/`<decorators>` — migrate those to `@Interceptor`+`@Priority` first; see ejb-to-cdi-mapping.md).

**PRESERVE**: `src/main/resources/import.sql`, `data.sql`, and any `.sql` referenced by tests (`src/test/resources/`) — Quarkus-native data init, NOT app-server descriptors.

## Phase 2: Core Migration (CONDITIONAL — if EJB_NEEDED or JPA_NEEDED)

**Exit:** `./mvnw clean compile` passes.

**Migration Comment Rule**: MIGRATION comments must use plain English, not annotation names ("Converted stateless session bean", not "Removed @Stateless") — literal annotation names in comments trigger false-positive Exit-Criteria grep failures.

**Rename dotted @Named bean names FIRST.** EL resolvers read dots as property accessors (`#{public.track}` → bean `public`.property `track`). Detect: `grep -rn '@Named(".*\..*")' src/main/java/`; rename to camelCase and update ALL EL references.

### Step 6: EJB → CDI
**Thread-safety**: beans with mutable instance state MUST be `@RequestScoped`, NOT `@ApplicationScoped`. Detect: `grep -rn '@ApplicationScoped' src/main/java/ -A5 | grep -i 'inSession\|inGlobal\|mutable\|boolean.*=.*false'`.
- `@Stateless` → `@ApplicationScoped` + class-level `@Transactional`
- `@Singleton` → `@ApplicationScoped` (+ StartupEvent observer if `@Startup`)
- `@Stateful` → `@SessionScoped` (evaluate if still needed)
- `@EJB` → `@Inject` (constructor injection preferred); remove `@LocalBean`/`@Local`/`@Remote`
- `@Remove` → remove annotation only; becomes regular business method (do NOT convert to `@PreDestroy`)
- Add no-arg constructor for normal-scoped beans (ArC proxy)
- **`@Singleton`+`@Startup`+`@PostConstruct`**: `@PostConstruct` is REMOVED (NOT migrated to `jakarta.annotation.PostConstruct` — supersedes the generic namespace rule); add `void onStart(@Observes StartupEvent ev) { init(); }`.
- **BMT**: drop `@TransactionManagement(BEAN)`; keep `@Inject UserTransaction` (works via quarkus-narayana-jta).

**@Transactional mapping**: class-level when a `@Stateless` bean uses EntityManager in multiple methods (read AND write) — preserves EJB REQUIRED semantics. Method-level only for different propagation. Rationale: Quarkus Hibernate ORM needs an active transaction for `find()`/`getResultList()` too, not just writes — write-only `@Transactional` breaks reads at runtime with `TransactionRequiredException`.

**CONVERSATION_SCOPED_FALLBACK** (if set in Phase 0): replace `@ConversationScoped` with `@SessionScoped`, add `implements Serializable` + `serialVersionUID`, emit MIGRATION-WARNINGS.md note about state-leak across browser tabs.

### Step 7: JPA
Remove persistence.xml (→ Phase 1 properties); keep entity annotations (jakarta.persistence.* unchanged); `@PersistenceContext` → `@Inject EntityManager`; multi-datasource → named datasources + `@PersistenceUnit("name")`.

### Step 8: JAX-RS
`@Path`/`@GET`/`@POST` unchanged; `ClientBuilder` → `@RegisterRestClient` + `@RestClient`; remove `Application` subclass — if it had `@ApplicationPath("/api")`, set `quarkus.rest.path=/api` (RESTEasy Reactive) or `quarkus.resteasy.path=/api` (Classic) — **wrong key silently reverts to `/`**. `@Provider` classes unchanged. Health-check servlets → remove (quarkus-smallrye-health provides `/q/health`).

**@Asynchronous Future→Uni cross-task dependency**: scan callers for `Future<T>`, update them in the same task or schedule after the callee — else late compile errors.

**EE Concurrency ContextService**: remove `createContextualProxy()` calls — `ManagedExecutor` (quarkus-smallrye-context-propagation) propagates context automatically.

**@SessionScoped validation**: `grep -rn '@SessionScoped' src/main/java/ | xargs -r grep -L 'implements Serializable'` must be empty.

**quarkus-hibernate-validator** (if VALIDATION_NEEDED): add it (BOM-managed, NOT transitive from hibernate-orm) — else `jakarta.validation.constraints.*` fails to compile.

**import.sql compat**: verify SQL table names match `@Table(name=...)`; if mismatch set `quarkus.hibernate-orm.physical-naming-strategy=...PhysicalNamingStrategyStandardImpl`.

### Additional steps
Step 9 (SCHEDULER_NEEDED): EJB Timers → Quarkus Scheduler. Step 10 (WEBSOCKET_NEEDED). Step 12 (HAS_JNDI): replace JNDI lookups. Step 14 (JMS_NEEDED): messaging — validate with `grep -rn '@MessageDriven\|javax\.jms\|jakarta\.jms' src/main/java/` (must be empty). Step 15 (SOAP_NEEDED).

**@Blocking**: add `@io.smallrye.common.annotation.Blocking` to any `@Path` endpoint doing `Emitter.send()`/`Uni.await()`/`@Transactional`/JDBC/EntityManager work, and any `@Incoming` doing blocking I/O or `@Transactional`.

## Phase 3 note
Messaging + security run here (conditional). See jms-to-smallrye.md and security-migration.md. Messaging validation grep as above.

## Phase 4: Testing & UI (ALWAYS)

**Exit:** `./mvnw clean test` passes.

### Step 16: Test Framework
Remove Arquillian, add Quarkus test deps; `@RunWith(Arquillian.class)` → `@QuarkusTest`; delete `@Deployment` ShrinkWrap; `@ArquillianResource URL` → `@TestHTTPResource`; convert to JUnit 5. Key gotchas (full detail → arquillian-to-quarkustest.md):
- **Assertion arg order**: 3-arg assertions move message String FIRST→LAST (compiler-silent when args share type) — Section 10.
- **`@Test(expected=X)` → `assertThrows`**: lambda captures effectively final — Section 11.
- **Strict stubbing**: after MockitoJUnitRunner→MockitoExtension, remove stubs unreachable after a guard throws — Section 9.
- **Mandatory rename `*IT.java` → `*Test.java`**: Surefire excludes `*IT.java` (silently never run); keep `*IT` only for `@QuarkusIntegrationTest`.
- Prerequisite `mkdir -p src/main/resources/META-INF/resources/` for JSF resources.

### Step 17 (JSF_NEEDED)
Option A: JSF → Qute. Option B: MyFaces extension (minimal change). Resources: `mkdir -p src/main/resources/META-INF/resources/`; move `src/main/webapp/*` there; move `faces-config.xml` → `META-INF/`; verify `find src/main/webapp -name '*.xhtml'` empty; `rm -rf src/main/webapp/` if empty. `./mvnw test` must report non-zero test count.

### Step 18 (BATCH_NEEDED)
See batch-jberet-fallback.md.

## Phase 5: Deployment & Verification (ALWAYS)

**Exit:** `./mvnw clean verify` passes. Docker build AND `/q/health` are ENVIRONMENT-DEPENDENT (non-blocking if no Docker CLI / runtime).

### Step 19: Containerization
- `mkdir -p src/main/docker/`; remove old app-server Dockerfiles.
- Dockerfile.jvm base: `registry.access.redhat.com/ubi8/openjdk-17-runtime` (JDK 17) or `ubi9/openjdk-21-runtime` (JDK 21).
- Four-layer COPY from `target/quarkus-app/` (Maven) or `build/quarkus-app/` (Gradle): `lib/`, `*.jar`, `app/`, `quarkus/` (all `--chown=185 … /deployments/…`).
- Use `ENV JAVA_OPTS_APPEND` (NEVER `ENV JAVA_OPTS` — disables UBI ergonomic heap sizing). `EXPOSE 8080`. `ENTRYPOINT ["java","-jar","/deployments/quarkus-run.jar"]`.

### Step 20: Health & Observability
Add `quarkus-smallrye-health`; custom checks implement `org.eclipse.microprofile.health.HealthCheck` → `public HealthCheckResponse call()`.

### Step 21: Final Verification
Build verification; legacy import/annotation/descriptor scans; Docker build + container smoke test (both ENVIRONMENT-DEPENDENT → NON-APPLICABLE if no CLI/runtime).

### Container Image Replacement
| Old pattern | Quarkus replacement |
|---|---|
| `quay.io/wildfly/wildfly:*` | `registry.access.redhat.com/ubi8/openjdk-17:latest` |
| `jboss/wildfly` | `quay.io/quarkus/ubi-quarkus-native-image:*` |
| `/eap[0-9]*` | `jboss-eap-7/eap74-openjdk11-openshift-rhel8` → migrate to UBI |

Scan: `grep -rEn 'wildfly|jboss|eap[0-9]' charts/ k8s/ deploy/ helm/ README.md **/*.yaml`

### Proactive .bak cleanup (all phases)
After each editor-tool session in Phases 2–5, sweep the whole tree (not just current-task files):
```bash
find . -not -path './target/*' -not -path './build/*' \( -name '*.bak' -o -name '*.orig' \) -delete
```
