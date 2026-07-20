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

# SKILL: JavaEE to Quarkus

## TD Version & Inputs/Outputs

**Version**: 2.4 (v2.3 auto-evolved + manual condensation) — Targets Quarkus 3.33.x LTS (review for 3.34+) | **Category**: javaee-to-quarkus

| I/O | Files |
|-----|-------|
| **Reads** | `pom.xml`, `persistence.xml`, `jboss-*.xml`, `web.xml`, `faces-config.xml`, Java source files |
| **Produces** | `application.properties`, `Dockerfile.jvm`, `BLOCKERS.md` (if blockers), `MIGRATION-WARNINGS.md` (optional), `SECURITY-NOTES.md` |
| **Modifies** | Source `.java` files, `pom.xml` (Quarkus BOM), `src/main/resources/` |

**Troubleshooting** → `references/troubleshooting-pitfalls.md`. **Examples** → `references/worked-examples-complete.md`.

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 2.4 | 2026-07 | Manual condensation of v2.3 (four large reference files + SKILL.md trimmed ~30%, all patterns/rules preserved); colon-aware grep filters retained. |
| 2.3 | 2026-07 | Fix javax.* grep filter (colon-aware); Phase 1 Javadoc cleanup; rewrite assertion reorder rule (message-first, any arg count); expand strict-stubbing audit; Phase 5 .bak gate; qualify @PostConstruct removal scope; H2 test profile full property set; Gradle companion steps. |
| 2.2–2.0 | 2026-07 | Shell-safety denylist; Gradle wrapper bootstrap; mockito BOM; FQN scan; deprecated hibernate key; Dockerfile layers; Phase 0 exit-table contradictions; JUnit 4→5 rules. |
| 1.0 | 2025-06 | Initial release — 52 apps benchmarked. |

## Prerequisites
JDK 17 or 21 · Maven 3.8.6+ (or Gradle 8.10+) · Docker (for dev services) · Maven Central network access · source compiles cleanly before migration · clean git tree recommended (`git restore .` rollback).

## Shell Safety Constraints

- **NEVER** invoke `curl`/`wget` under ANY argument (even `--version`) — the sandbox denylist fires on the command name and **hard-terminates the pipeline**. To check availability: `which curl` / `command -v curl`.
- **grep** exits 1 on no match — never chain grep scans with `&&`; use `;` or append `|| true`.
- **find** exits 0 even when empty — use `result=$(find ...); [ -n "$result" ] && echo FOUND || echo CLEAN`.

<a id="bak-sweep"></a>**`.bak` sweep** (referenced throughout as *[bak-sweep]*): run after each editor-tool session and as a Phase 5 pre-validation gate:
```bash
find . -not -path './target/*' -not -path './build/*' \( -name '*.bak' -o -name '*.orig' \) -delete
```

## Quick Reference — Dependency Mapping

| JavaEE/Jakarta | Quarkus Extension | Notes |
|---|---|---|
| EJB | quarkus-arc | @Stateless → @ApplicationScoped |
| Stateful EJB | quarkus-arc | @Stateful → @SessionScoped (see ejb-to-cdi-mapping.md) |
| @Asynchronous EJB | quarkus-arc + smallrye-mutiny | Future<T>→Uni<T>; void→Uni<Void> (ejb-to-cdi Example 4) |
| JAX-RS | quarkus-rest | add **quarkus-rest-jackson** if endpoints return/consume non-String/Response types |
| Servlet | quarkus-rest | @WebServlet → @Path + @GET/POST (health servlets → remove) |
| JPA | quarkus-hibernate-orm | @PersistenceContext → @Inject |
| Bean Validation | quarkus-hibernate-validator | **NOT transitive** — add when @NotNull/@Size/@Email/@Valid present |
| JMS/MDB | quarkus-messaging-amqp (reactive, recommended) or quarkus-artemis-jms (JMS API preserved) | MDB → @Incoming/@Outgoing; @Blocking on blocking consumers/endpoints |
| JSF | quarkus-undertow + myfaces-quarkus | see jsf-migration-patterns.md |
| Batch | quarkus-jberet (limited) | fallback: CDI composition — batch-jberet-fallback.md |
| Security | quarkus-security | @RolesAllowed unchanged |
| JASPIC | quarkus-security | ServerAuthModule → HttpAuthenticationMechanism (full rewrite; security-migration.md) |
| Health | quarkus-smallrye-health | auto /q/health; self-activating |
| OpenTelemetry | quarkus-opentelemetry | quarkus.otel.* |
| REST Client | quarkus-rest-client | MicroProfile REST Client |
| SOAP/JAX-WS | quarkus-cxf | @WebService unchanged |
| @Schedule (EJB Timer) | quarkus-scheduler | @Schedule → @Scheduled; @Singleton → @ApplicationScoped |
| EE Concurrency | quarkus-smallrye-context-propagation | ManagedExecutorService → @Inject ManagedExecutor |
| LRA | quarkus-narayana-lra | @LRA/@Compensate/@Complete unchanged; **ALWAYS add quarkus-rest-jackson + quarkus-rest-client** |
| WebSocket | quarkus-websockets | @ServerEndpoint unchanged |
| Testcontainers | quarkus-test + testcontainers | Arquillian→@QuarkusTest |
| JTA | quarkus-narayana-jta | @Transactional preserved |
| BMT | quarkus-narayana-jta | drop @TransactionManagement(BEAN); do NOT add @Transactional; @Inject UserTransaction works |
| ServiceLoader SPI | quarkus-arc | → @Inject Instance<T>; add @ApplicationScoped, delete META-INF/services |

**Common Patterns:** @Stateless→@ApplicationScoped + class-level @Transactional (ONLY when bean uses EntityManager), @EJB→@Inject · @Stateful→@SessionScoped (Serializable; no @ConversationScoped in ArC) · @Singleton→@ApplicationScoped (@Startup observer for eager init) · @Singleton+@Startup+@PostConstruct → @ApplicationScoped + `void onStart(@Observes StartupEvent ev)` calling `init()`, **@PostConstruct REMOVED (ONLY this pattern)** · @Remove → remove annotation only (NOT @PreDestroy) · @Asynchronous Future<T>→Uni<T> (scan callers; Mutiny is lazy) · @PersistenceContext→@Inject EntityManager · @WebServlet→@Path · Remote EJB → not supported (pattern-remote-ejb-limitation.md).

**REST Path Configuration:** check pom.xml REST extension — `quarkus-rest` → `quarkus.rest.path=/path`; `quarkus-resteasy` → `quarkus.resteasy.path=/path`. **Wrong key silently reverts to `/`.** `@ApplicationPath("/")` or empty → write NO property. Write it in the SAME task as Application-class removal.

## Reference Index

| File | Purpose |
|---|---|
| [application-properties-checklist.md](references/application-properties-checklist.md) | Required config properties by extension |
| [arc-limitations.md](references/arc-limitations.md) | CDI ArC limitations, proxy restrictions |
| [arquillian-to-quarkustest.md](references/arquillian-to-quarkustest.md) | Test framework migration |
| [batch-jberet-fallback.md](references/batch-jberet-fallback.md) | CDI-based batch when JBeret fails |
| [compatibility-matrix.md](references/compatibility-matrix.md) | Platform/extension compatibility |
| [ear-consolidation.md](references/ear-consolidation.md) | Multi-module EAR → single JAR |
| [ejb-to-cdi-mapping.md](references/ejb-to-cdi-mapping.md) | EJB → CDI conversion guide |
| [jms-to-smallrye.md](references/jms-to-smallrye.md) | JMS/MDB → SmallRye messaging |
| [jpa-to-quarkus-persistence.md](references/jpa-to-quarkus-persistence.md) | JPA persistence migration |
| [jsf-migration-patterns.md](references/jsf-migration-patterns.md) | JSF→MyFaces / JSF→Qute |
| [pattern-remote-ejb-limitation.md](references/pattern-remote-ejb-limitation.md) | Remote EJB limitations |
| [phase0-detection-flags.md](references/phase0-detection-flags.md) | Feature detection, conditional logic |
| [phases-detail.md](references/phases-detail.md) | Detailed phase-by-phase guide |
| [quarkus-extension-catalog.md](references/quarkus-extension-catalog.md) | Extension selection, dep mapping |
| [security-migration.md](references/security-migration.md) | Security migration, SECURITY-NOTES.md |
| [troubleshooting-pitfalls.md](references/troubleshooting-pitfalls.md) | Troubleshooting, pitfalls |
| [worked-examples-complete.md](references/worked-examples-complete.md) | Complete before/after examples |

## Known Limitations & Blockers

- **REMOTE_EJB_DETECTED** — @Remote not supported → pattern-remote-ejb-limitation.md
- **EJB2_ENTITY_BEAN_FOUND** — EJB 2.x Entity Beans (`extends EntityBean`) not supported → HALT
- **ELYTRON_SECURITY_DOMAIN** — EJB security context propagation via jboss-ejb3.xml `<security-domain>` or `SessionContext.runAs()`/`EJBContext.runAs()`/`SecurityContext.runAs()` not portable → security-migration.md. **Detect**: `find . \( -name 'jboss-ejb3.xml' -o -name 'jboss-app.xml' \) | xargs grep -l '<security-domain>' 2>/dev/null` AND `grep -rn 'EJBContext.*runAs\|SessionContext.*runAs\|SecurityContext.*runAs' src/`. **`<security-domain>` in jboss-web.xml is NOT a blocker** — maps to `quarkus.http.auth.basic=true`.
- **JNDI_LOOKUPS** — InitialContext.lookup() fails at runtime. **Resolvable blocker**, priority: (1) `@ConfigProperty` if static config; (2) `@Inject` CDI bean if managed resource (DataSource, ConnectionFactory); (3) `throw new UnsupportedOperationException(...)` ONLY for dead code / truly unresolvable dynamic lookups. See troubleshooting-pitfalls.md.
- **JAVAX_PINNED_BINARY_DEPENDENCY** — ONLY when BOTH: (1) genuine JavaEE app (has EE APIs), AND (2) a **binary** dep pins `javax.*` with no `jakarta.*` release. Do NOT halt until: (a) find jakarta replacement/newer version; (b) **MANDATORY** apply Eclipse Transformer (`org.eclipse.transformer`) to rewrite `javax.*`→`jakarta.*`; (c) only if both fail → HALT + BLOCKERS.md. **An unfamiliar dependency alone is NOT a blocker.**

## Rollback & Abort
On hard blocker (REMOTE_EJB_DETECTED, EJB2_ENTITY_BEAN_FOUND, ELYTRON_SECURITY_DOMAIN): emit `BLOCKERS.md` with detection evidence, do NOT modify source, exit status 1. Mid-migration failure: `git restore .`.

## Exit Status
**0** Success | **1** Blocker (BLOCKERS.md emitted) | **NO_OP** Already Quarkus | **WRONG_FRAMEWORK** Spring Boot

**Phase 0 Exit Behavior:**

| Exit Type | Required Outputs | Reference Files | Validation |
|---|---|---|---|
| **WRONG_FRAMEWORK** | none (no modification); `spring-boot-starter-parent` in `<parent>` alone suffices | SKILL.md only | build check only |
| **NO_OP** | `Dockerfile.jvm` (`mkdir -p src/main/docker/` first), `SECURITY-NOTES.md` | SKILL.md + phase0-detection-flags.md + security-migration.md | build check only |
| **BLOCKER (HALT)** | `BLOCKERS.md` | SKILL.md + phase0-detection-flags.md | legacy-content checks NON-APPLICABLE |

**NO_OP SECURITY-NOTES.md inline template** (when no vulnerabilities — avoids loading security-migration.md; scans MUST still run first):
```markdown
# Security Notes
No pre-existing security vulnerabilities detected during migration analysis.
## Scan Summary
- Path traversal patterns: CLEAN
- Hardcoded credentials: CLEAN
- Deprecated cryptography: CLEAN
- Insecure deserialization: CLEAN
```

**Fast-path**: WRONG_FRAMEWORK/NO_OP detection runs FIRST (pom.xml/build.gradle only — do NOT load reference files beyond SKILL.md + phase0-detection-flags.md for the decision). **Guard**: if `src/main/java/` absent, set all flags false and skip annotation scans.

> **WARNING**: Pre-existing pom.xml descriptions, comments, and Javadoc may carry stale migration labels. SKILL.md and reference docs are the ONLY authoritative sources.

**Non-Goals:** EJB Remote/IIOP, JCA Resource Adapters, EJB 2.x Entity Beans, CDI Portable Extensions (→ BuildExtension), XA Distributed Transactions, JSP compilation, HA Singleton, EJB Security Context Propagation across Remote calls, automatic BMT→CMT conversion.

**ArC CDI Restrictions:** normal-scoped beans need no-arg constructor; @ApplicationScoped must be thread-safe. See arc-limitations.md.

## Exit Criteria Checklist

**MANDATORY — all must pass:**
- [ ] Build: Maven → `./mvnw clean verify`; Gradle-only → `./gradlew build`
- [ ] javax.* check (comment-filtered, colon-aware):
  ```bash
  grep -rn --include='*.java' '\bjavax\.' src/ | grep -v 'javax\.sql\|javax\.crypto\|javax\.security\.auth\|javax\.net\|javax\.naming\|javax\.xml\|javax\.annotation\|javax\.management' | grep -v '^[^:]*:[0-9]*:\s*//' | grep -v '^[^:]*:[0-9]*:\s*\*'
  ```
- [ ] No EJB annotations: `grep -rn '@Stateless\|@EJB\|@TransactionAttribute\|@MessageDriven' src/main/java/ | grep -v '^[^:]*:[0-9]*:\s*//' | grep -v '^[^:]*:[0-9]*:\s*\*'` empty
  > Migration comments must never quote literal `@EJB`/`@Stateless`/`@TransactionAttribute`/`@MessageDriven` or `javax.jms`/`jakarta.jms` — use prose (grep matches inside comments; the colon-aware filter only removes `//`/`*` lines).
- [ ] No JNDI lookups: `grep -rn 'InitialContext\|context\.lookup' src/main/java/ | grep -v '^[^:]*:[0-9]*:\s*//' | grep -v '^[^:]*:[0-9]*:\s*\*'` empty
- [ ] No app-server descriptors (persistence.xml, jboss-web.xml); no redundant web.xml/beans.xml
- [ ] Dockerfile present: `test -f src/main/docker/Dockerfile.jvm` (Quarkus base image + `quarkus-app/` layers)
- [ ] `application.properties` present with health/HTTP config
- [ ] Clean build artifacts: *[bak-sweep]* print variant returns empty
- [ ] SECURITY-NOTES.md present

**Phase 0 HALT scoring:** correct BLOCKERS.md emission without source modification = Exit Status 1 (correct, NOT failure).

**ENVIRONMENT-DEPENDENT (non-blocking — NON-APPLICABLE if tooling absent):** Docker build (`docker build -f src/main/docker/Dockerfile.jvm`) · Health check (`curl localhost:8080/q/health`, localhost only) · JSF namespace (`grep -rn "jakarta\.face\." src/ | grep -v "jakarta\.faces\."` empty) · No WildFly/JBoss refs in deploy manifests.

**OVERALL STATUS:** `COMPLETE` = every applicable criterion passes. Env-dependent checks that can't execute = NON-APPLICABLE, not PARTIAL/FAIL.

## Migration Phases — Checklist

Full detail → [phases-detail.md](references/phases-detail.md). This is the operational checklist.

**Phase 0: Analysis (ALWAYS)**
- [ ] **WRONG_FRAMEWORK/NO_OP FIRST** (before feature scans): `spring-boot-starter-*` → WRONG_FRAMEWORK; quarkus-bom present AND no javax.* → NO_OP
- [ ] **Complete ALL blocker scans before halting** (one-pass BLOCKERS.md)
- [ ] REMOTE_EJB_DETECTED → HALT
- [ ] EJB2_ENTITY_BEAN_FOUND: `grep -rn 'extends javax.ejb.EntityBean\|implements EntityBean' src/` → HALT
- [ ] CDI Portable Extensions: `grep -rn 'implements Extension\|META-INF/services/javax.enterprise.inject.spi.Extension' src/` → WARNING
- [ ] CONVERSATION_SCOPED: `grep -rn '@ConversationScoped' src/main/java/` → set CONVERSATION_SCOPED_FALLBACK=true
- [ ] ELYTRON_SECURITY_DOMAIN (see Blockers section for detection) → HALT
- [ ] Feature flags: EJB_NEEDED, JPA_NEEDED, JMS_NEEDED, JSF_NEEDED, SECURITY_NEEDED
- [ ] JPA_NEEDED: `grep -rn '@Entity\|@PersistenceContext\|javax\.persistence\|jakarta\.persistence' src/main/java/` OR `find src/ -name 'persistence.xml'`
- [ ] VALIDATION_NEEDED: `grep -rn '@NotNull\|@Size\|@Email\|@Positive\|@Min\|@Max\|@Pattern\|@Valid' src/main/java/`
- [ ] BATCH_NEEDED: `grep -rn 'jakarta.batch.api\|javax.batch.api\|batch.xml\|@Named.*Item\(Reader\|Processor\|Writer\)\|implements Item\(Reader\|Processor\|Writer\)\|@BatchProperty\|JobOperator' src/`
- [ ] JNDI_LOOKUPS: `grep -rn 'new InitialContext\|context\.lookup' src/main/java/` → resolve before proceeding
- [ ] **MIGRATABILITY GUARD**: any Jakarta/JavaEE API → it IS a migration target. NOT_APPLICABLE only for ZERO EE APIs or Spring Boot (WRONG_FRAMEWORK).
- [ ] **Build tool**: if BOTH pom.xml and build.gradle exist, Gradle wins (build.gradle = source of truth)

**Phase 1: Build System & Namespace (ALWAYS)**
- [ ] `mkdir -p src/main/resources` (may be absent in webapp-only layouts)
- [ ] Add Quarkus BOM + plugin — **BOM: `io.quarkus.platform:quarkus-bom:3.33.2`**; **plugin: `io.quarkus:quarkus-maven-plugin`** (groupIds differ intentionally — swapping = hard build failure)
- [ ] Add `io.quarkus:quarkus-smallrye-health` (mandatory here, ensures /q/health for Phase 5)
- [ ] WAR→JAR packaging + remove `failOnMissingWebXml`, `version.wildfly`, `version.microprofile`, `maven-war-plugin`
- [ ] Gradle wrapper bootstrap if `gradlew` absent: `gradle wrapper --gradle-version=8.10 --no-daemon` (NEVER curl/wget); Gradle-only → do NOT create pom.xml
- [ ] Multi-module EAR/WAR consolidation → single Quarkus module
- [ ] **Namespace javax.*→jakarta.*** (EE only; NOT crypto/net/sql/xml). Discovery: `grep -rl 'import javax\.' src/ | grep -v 'javax\.sql\|javax\.crypto\|javax\.security\.auth\|javax\.net\|javax\.naming'`. **Also replace literal javax.* strings in Javadoc/comments with prose** (exit-criteria grep can't reliably exclude Javadoc).
- [ ] Extract persistence.xml/web.xml → application.properties (atomic: ALL Phase 1–3 properties in ONE task, incl. `quarkus.arc.remove-unused-beans=false`)
- [ ] Context-root: set `quarkus.http.root-path` ONLY if non-root; ALWAYS pair `quarkus.http.non-application-root-path=/q`
- [ ] Delete app-server descriptors (jboss-*, weblogic-*, glassfish-*, payara-*, sun-*, *-ds.xml, web.xml); **beans.xml** only if no non-empty `<interceptors>`/`<alternatives>`/`<decorators>` (migrate those to `@Interceptor`+`@Priority` first)
- [ ] **PRESERVE** import.sql, data.sql
- [ ] Derive JDBC extension from persistence.xml (same task as BOM)
- [ ] *[bak-sweep]*
- [ ] Build: `./mvnw clean compile -Dmaven.test.skip=true`

**Phase 2: Core Migration (CONDITIONAL — if EJB_NEEDED or JPA_NEEDED)**
- [ ] EJB→CDI: @Stateless→@ApplicationScoped + class-level @Transactional (ONLY when bean uses EntityManager), @EJB→@Inject
- [ ] @Stateful → @SessionScoped + `implements Serializable`
- [ ] @Remove → remove annotation only (NOT @PreDestroy)
- [ ] @Singleton+@Startup+@PostConstruct → @ApplicationScoped + `void onStart(@Observes StartupEvent ev)` calling `init()`. **@PostConstruct removal ONLY for this pattern** — for all other bean types RETAIN @PostConstruct and migrate to `jakarta.annotation.PostConstruct`.
- [ ] CONVERSATION_SCOPED_FALLBACK=true → @SessionScoped + MIGRATION-WARNINGS.md
- [ ] VALIDATION_NEEDED → add `quarkus-hibernate-validator` (NOT transitive)
- [ ] Multi-persistence-unit → MIGRATION-WARNINGS.md + jpa-to-quarkus-persistence.md
- [ ] JPA: @PersistenceContext→@Inject EntityManager
- [ ] JAX-RS: remove Application class AND write rest path property in SAME task (see REST Path Configuration)
- [ ] Validate @SessionScoped: `grep -rn '@SessionScoped' src/main/java/ | xargs -r grep -L 'implements Serializable'` empty
- [ ] Build: `./mvnw clean compile` · *[bak-sweep]*
- Detailed → [ejb-to-cdi-mapping.md](references/ejb-to-cdi-mapping.md)

**Phase 3: Messaging & Security (CONDITIONAL — if JMS_NEEDED or SECURITY_NEEDED)**
- [ ] JMS: @Resource ConnectionFactory/Queue → quarkus-messaging-amqp or -kafka; MDB → @Incoming/@Outgoing
- [ ] **@Blocking** on any @Path/@Incoming doing Emitter.send()/Uni.await()/@Transactional/JDBC/EntityManager
- [ ] **Bidirectional channels**: every channel in @Incoming AND @Outgoing/@Channel/Emitter needs BOTH `mp.messaging.incoming.<n>.*` and `outgoing.<n>.*` (except intra-app in-memory)
- [ ] Test-scope `io.smallrye.reactive:smallrye-reactive-messaging-in-memory` if test profile uses in-memory connector (NOT transitive)
- [ ] Messaging validation: `grep -rn '@MessageDriven\|javax\.jms\|jakarta\.jms' src/main/java/ | grep -v '^[^:]*:[0-9]*:\s*//' | grep -v '^[^:]*:[0-9]*:\s*\*'` empty
- [ ] Security: web.xml security-constraint → @RolesAllowed + quarkus-elytron-security-properties-file · *[bak-sweep]*

**Phase 4: Testing & UI (ALWAYS)**
- [ ] @RunWith(Arquillian)→@QuarkusTest; use `io.quarkus:quarkus-junit` (NOT quarkus-junit5, deprecated since 3.31)
- [ ] Remove @Deployment/ShrinkWrap; integration tests → testcontainers via @QuarkusTestResource
- [ ] **JUnit 4→5 assertion reorder**: message String moves FIRST→LAST whenever first arg is a String message, regardless of arg count. **Exemption**: no-message calls (`assertEquals(exp, act)`, `assertTrue(cond)`). Affected: `assertEquals/assertNotEquals/assertNotNull/assertNull/assertTrue/assertFalse/assertSame/assertNotSame(msg, …)`. Detect: `grep -rn 'assertNull("\|assertEquals("\|assertNotNull("' src/test/` — Section 10
- [ ] **@Test(expected=X) → assertThrows**: lambda = only throwing call, captures effectively final. Retain try/catch/fail when catch asserts on multiple exception fields. Remove unreachable pre-throw stubs + dead post-throw code. Capture for property asserts. — Sections 11/18
- [ ] **Strict-stubbing audit** (MockitoJUnitRunner→MockitoExtension): "used-by-all" ONLY if EVERY test reaches the stub, else `lenient().when()`. void→delete `doNothing()` ONLY if never called; `Uni<Void>`→`lenient()...voidItem()`. Retry/loop-body stubs ARE reachable. Audit both @Mock fields AND inline `mock()`. — Section 9
- [ ] **All-@Inject-field audit**: each production @Inject field needs a test mock/stub/real injection (else NPE)
- [ ] **@BeforeEach config audit**: ensure setup doesn't invalidate negative-path assertions
- [ ] **Rename *IT.java → *Test.java** (Surefire excludes *IT); if *Test.java exists → *IntegrationTest.java; keep *IT ONLY for @QuarkusIntegrationTest
- [ ] **Build file companion**: add `quarkus-jdbc-h2` (test) + `mockito-junit-jupiter` (BOM-managed, no version). H2 test profile: `%test.quarkus.datasource.db-kind=h2` + `.jdbc.url=jdbc:h2:mem:test;DB_CLOSE_DELAY=-1` + `.username=sa` + `.password=` + `%test.quarkus.hibernate-orm.schema-management.strategy=drop-and-create`. Full url+username+password needed when main config has non-H2 jdbc.url. Gradle: `testImplementation(...)` with versions from `enforcedPlatform(quarkusBom)`.
- [ ] `@TestSecurity` tests → add `io.quarkus:quarkus-test-security`
- [ ] JSF: MyFaces or Qute; `mkdir -p src/main/resources/META-INF/resources/` then move webapp/*
- [ ] Tests pass: `./mvnw clean test` (or `./gradlew test`); non-zero test count; migrated tests in surefire output · *[bak-sweep]*
- Test migration → [arquillian-to-quarkustest.md](references/arquillian-to-quarkustest.md) | JSF → [jsf-migration-patterns.md](references/jsf-migration-patterns.md)

**Phase 5: Deployment & Verification (ALWAYS)**
- [ ] **MANDATORY pre-validation**: *[bak-sweep]* (catches .bak missed in per-phase sweeps)
- [ ] `mkdir -p src/main/docker/` then Dockerfile.jvm. Base: `ubi8/openjdk-17-runtime` (JDK 17) or `ubi9/openjdk-21-runtime` (JDK 21). Four-layer COPY from `target/quarkus-app/` (Maven) or `build/quarkus-app/` (Gradle): `lib/`→`*.jar`→`app/`→`quarkus/`. Use `ENV JAVA_OPTS_APPEND` (NEVER `ENV JAVA_OPTS`).
- [ ] Health endpoints via quarkus-smallrye-health (added Phase 1)
- [ ] Docker build (ENV-DEPENDENT); Helm/K8s scan: image refs, ports, probes (/health → /q/health/live)
- [ ] SECURITY-NOTES.md; `/q/health` returns UP (ENV-DEPENDENT)
- Container setup → [phases-detail.md](references/phases-detail.md#phase-5-deployment--verification-always)

## Migration Patterns — Summary

| Pattern | Complexity | Outcome | Details |
|---|---|---|---|
| EJB→CDI | Medium | @Stateless→@ApplicationScoped | ejb-to-cdi-mapping.md |
| JPA Persistence | Low | @PersistenceContext→@Inject | jpa-to-quarkus-persistence.md |
| JMS→SmallRye | Medium | @MessageDriven→@Incoming | jms-to-smallrye.md |
| Security | Low | @RolesAllowed unchanged | security-migration.md |
| JSF→MyFaces | Low | Minimal changes | jsf-migration-patterns.md |
| JSF→Qute | High | Template rewrite | jsf-migration-patterns.md |
| Remote EJB | **BLOCKER** | Replace with REST/gRPC | pattern-remote-ejb-limitation.md |
| Arquillian Tests | Medium | @RunWith→@QuarkusTest | arquillian-to-quarkustest.md |

**Compatibility:** Quarkus 3.33.x LTS · JDK 17/21 · Maven 3.8+ · Hibernate ORM 6.x · Jakarta EE 10. Full → compatibility-matrix.md.

## Reference Dispatch

| Signal in Source | Reference |
|---|---|
| Phase 0 flags, pom.xml dep scan | phase0-detection-flags.md |
| @Stateless, @EJB, @TransactionAttribute | ejb-to-cdi-mapping.md |
| ArC proxy errors, CDI extensions | arc-limitations.md |
| persistence.xml, @PersistenceContext | jpa-to-quarkus-persistence.md |
| @MessageDriven, JMSContext, @Incoming | jms-to-smallrye.md |
| @RolesAllowed, SessionContext, SECURITY-NOTES.md | security-migration.md |
| .xhtml, @Named, @ViewScoped | jsf-migration-patterns.md |
| @Remote EJB interfaces | pattern-remote-ejb-limitation.md |
| @RunWith(Arquillian), @Deployment | arquillian-to-quarkustest.md |
| Extension selection | quarkus-extension-catalog.md |
| Complete worked examples | worked-examples-complete.md |

## Tips & Best Practices

- [2025-05] Always set `<quarkus.platform.version>3.33.2</quarkus.platform.version>` (LTS).
- [2025-05] @ApplicationScoped with mutable state → @RequestScoped. Use `quarkus.arc.remove-unused-beans=false` during migration. Prefer constructor injection (testability, avoids ArC proxy issues).
- [2025-06] NEVER delete a test whose target class still exists — rewrite mocks against the CDI type. Compare `grep -rc '@Test' src/test` before/after; drops must map to removed classes.
- [2026-07] Quarkus 3.33.2+: `mockito-core` and `mockito-junit-jupiter` BOM-managed — omit `<version>`. Verify: `mvn dependency:resolve | grep mockito`.
- [2026-07] MIGRATION comments must NOT quote literal annotation names — use prose. Applies to pre-existing Javadoc too (replace literal `javax.*` with prose).
- [2026-07] web.xml `<env-entry>` JNDI path ≠ `@ConfigProperty` key — grep `@ConfigProperty(name=...)` for actual keys.
- [2026-07] Mutiny Uni is lazy — a Uni without `.subscribe()`/`.await()` is a silent no-op. Verify caller subscription after @Asynchronous→Uni.

## Validation Commands

```bash
# Build
./mvnw clean verify     # Maven          ./gradlew build   # Gradle-only

# Legacy scans (must return ZERO) — comment-filtered, colon-aware:
grep -rn --include='*.java' '\bjavax\.' src/ | grep -v 'javax\.sql\|javax\.crypto\|javax\.security\.auth\|javax\.net\|javax\.naming\|javax\.xml\|javax\.annotation\|javax\.management' | grep -v '^[^:]*:[0-9]*:\s*//' | grep -v '^[^:]*:[0-9]*:\s*\*'
grep -rn '@Stateless\|@EJB\|@TransactionAttribute\|@MessageDriven' src/main/java/ | grep -v '^[^:]*:[0-9]*:\s*//' | grep -v '^[^:]*:[0-9]*:\s*\*'
grep -rn 'InitialContext\|context\.lookup' src/main/java/ | grep -v '^[^:]*:[0-9]*:\s*//' | grep -v '^[^:]*:[0-9]*:\s*\*'
find . -name 'persistence.xml' -o -name 'jboss-web.xml' -o -name '*-ds.xml'

# Container (localhost only — env-dependent):
docker build -f src/main/docker/Dockerfile.jvm -t test . && docker run -d -p 8080:8080 test && curl http://localhost:8080/q/health
```
Complete validation checklist → [phase0-detection-flags.md](references/phase0-detection-flags.md)
