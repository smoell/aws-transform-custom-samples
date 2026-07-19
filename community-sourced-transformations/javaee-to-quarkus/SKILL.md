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

**Version**: 1.6.1 (manual condensation) — Targets Quarkus 3.33.x LTS (review for 3.34+) | **Category**: javaee-to-quarkus

| I/O | Files |
|-----|-------|
| **Reads** | `pom.xml`, `persistence.xml`, `jboss-*.xml`, `web.xml`, `faces-config.xml`, Java source files |
| **Produces** | `application.properties`, `Dockerfile.jvm`, `BLOCKERS.md` (if blockers found), `MIGRATION-WARNINGS.md` (optional) |
| **Modifies** | Source `.java` files, `pom.xml` (Quarkus BOM), `src/main/resources/` |

## Troubleshooting
See `references/troubleshooting-pitfalls.md` for common issues, error resolutions, and migration pitfalls.

## Examples
See `references/worked-examples-complete.md` for complete worked migration examples including conditional patterns.

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.6 | 2026-07 | Add Shell Safety Constraints (curl/wget prohibition); add Gradle-only project BUILD COMMAND handling; add @PostConstruct→StartupEvent test-compatibility pattern; add Phase 0 complete-all-scans rule; fix Application class removal conditional (NON-ROOT only); add SmallRye in-memory test connector; add WildFly ExampleDS→H2 mapping; add @Schedules plural pattern. |
| 1.5 | 2026-07 | Fix HealthCheck method (call() not check()), fix JASPIC row, fix @Transactional scope (class-level for EJB), add EJB2_ENTITY_BEAN_FOUND blocker, add @Remove pattern, narrow javax.* grep, add JUnit 5 assertion order, add quarkus-junit5→quarkus-junit rename, add Uni<T> mock pitfall, add JDBC extension derivation, add import.sql preservation, harmonize messaging artifact names. |
| 1.4 | 2026-07 | Fix JNDI_LOOKUPS inconsistency (resolvable blocker, not hard HALT); remove javax.naming from exit-criteria exclusion; add JPA_NEEDED flag; fix BMT property name; add MockitoExtension strict stubbing guidance; add env-entry key resolution and root-path health impact. |
| 1.3 | 2026-07-16 | Fix over-caution regression from v1.1/1.2 (10 migratable apps wrongly halted as NOT_APPLICABLE): narrowed JAVAX_PINNED_BINARY_DEPENDENCY to genuine-JavaEE + binary-dep only, made Eclipse Transformer MANDATORY before halting, added Phase-0 MIGRATABILITY GUARD (presence of EE APIs → migrate; unfamiliar dep is not a blocker). |
| 1.2 | 2026-07-16 | Exit criteria: Docker build + health check reclassified as ENVIRONMENT-DEPENDENT (non-blocking); added OVERALL STATUS rule (unexecutable checks = NON-APPLICABLE, not PARTIAL). Prevents false PARTIAL in the no-Docker benchmark scorer. |
| 1.1 | 2026-07-16 | Added blocker JAVAX_PINNED_BINARY_DEPENDENCY (rapla) and test-preservation rule (Cloud-Connectors), from CEAT 32-app benchmark (beta): 100% build on 17 real migrations, 15 correct Phase-0 NO-OPs. See BENCHMARKS.md. |
| 1.0 | 2025-06-01 | Initial release — 52 apps benchmarked (WildFly quickstarts, JavaEE 7 samples, WASdev, ScarfBench DayTrader, IBM CustomerOrderServices). Hard%: 50%. |
| 0.9 | 2025-05-15 | Added patterns: OpenTelemetry, MicroProfile REST Client, SOAP/JAX-WS (CXF), LRA/Narayana, Micrometer, JTA/@Transactional, @Stateful EJB, JASPIC Security, Testcontainers, JBeret Batch |
| 0.8 | 2025-05-01 | Initial SKILL.md optimization from 1,263 → 196 lines; extracted detail to references/ |

## Prerequisites
- **JDK**: 17 or 21 (Quarkus 3.33.x LTS requires JDK 17+)
- **Maven**: 3.8.6+ (or Gradle 8.x — see Phase 1 for Gradle notes)
- **Docker**: Required for dev services (PostgreSQL, Artemis, Kafka)
- **Maven Central**: Network access required for dependency resolution
- **Source state**: Application must compile cleanly before migration begins
- **Git**: Clean working tree recommended (allows rollback via `git restore .`)

## Shell Safety Constraints

**NEVER** use `curl`, `wget`, or any HTTP fetch tool to query external resources (Maven Central, version registries, API endpoints). The sandbox environment enforces a shell-safety denylist that **hard-terminates the entire pipeline** on violation — no validation report is produced. To determine the target Quarkus version, use the version documented in this spec (3.33.x LTS). The `curl` command in Container Verification (localhost health check) is annotated as ENVIRONMENT-DEPENDENT and local-only.

## Quick Reference

**Dependency Mapping:**
| JavaEE/Jakarta | Quarkus Extension | Notes |
|---|---|---|
| EJB | quarkus-arc | @Stateless → @ApplicationScoped |
| Stateful EJB | quarkus-arc | @Stateful → @SessionScoped — **session-scoped fallback; semantics differ from EJB @Stateful** (HTTP session lifecycle, shared across browser tabs; @ConversationScoped not supported in ArC; see ejb-to-cdi-mapping.md for state-leak warning) |
| JAX-RS | quarkus-rest | Minimal changes |
| Servlet | quarkus-rest | @WebServlet → @Path + @GET/POST (health-check servlets → remove entirely; quarkus-smallrye-health provides /q/health) |
| JPA | quarkus-hibernate-orm | @PersistenceContext → @Inject |
| JMS/MDB | quarkus-messaging-amqp (or quarkus-messaging-kafka) | MDB → @Incoming/@Outgoing channels; requires @Blocking on REST endpoints calling messaging |
| JSF | quarkus-undertow + myfaces-quarkus | See [JSF→MyFaces pattern](references/jsf-migration-patterns.md) |
| Batch | quarkus-jberet (limited compat) — fallback: CDI-based @ApplicationScoped beans with manual ItemReader/Processor/Writer composition | See references/batch-jberet-fallback.md |
| Security | quarkus-security | @RolesAllowed unchanged |
| JASPIC | quarkus-security | ServerAuthModule → HttpAuthenticationMechanism (full class rewrite: authenticate()/getChallenge()/getCredentialTypes(); see security-migration.md). Transitive dep of quarkus-rest — no extra pom.xml entry needed |
| HttpAuthenticationMechanism | quarkus-security | Custom mechanism via HttpAuthenticationMechanism interface (same SPI, works natively) |
| Health | quarkus-smallrye-health | Auto /q/health endpoints; implement `HealthCheck` interface → `public HealthCheckResponse call()` method |
| OpenTelemetry | quarkus-opentelemetry | MicroProfile compatible, quarkus.otel.* config |
| REST Client | quarkus-rest-client | MicroProfile REST Client, quarkus.rest-client.* config |
| SOAP/JAX-WS | quarkus-cxf | @WebService unchanged, quarkus.cxf.endpoint.* config |
| @Schedule (EJB Timer) | quarkus-scheduler | @Schedule → @Scheduled; @Singleton timer bean → @ApplicationScoped |
| EE Concurrency (ManagedExecutorService) | quarkus-smallrye-context-propagation | @Resource ManagedExecutorService → @Inject ManagedExecutor |
| LRA | quarkus-narayana-lra | @LRA, @Compensate, @Complete unchanged |
| WebSocket | quarkus-websockets | @ServerEndpoint unchanged |
| Testcontainers | quarkus-test + testcontainers | Arquillian→@QuarkusTest, embedded containers for integration tests |
| JTA | quarkus-narayana-jta | @Transactional preserved, Narayana JTA, ACID properties maintained |
| BMT | quarkus-narayana-jta | @Inject UserTransaction works in Quarkus via quarkus-narayana-jta; prefer @Transactional unless explicit boundary control needed; configure quarkus.transaction-manager.object-store.directory for crash recovery |

**Common Patterns:**
- EJB→CDI: @Stateless → @ApplicationScoped + @Transactional (class-level), @EJB → @Inject
- @Stateful EJB → @SessionScoped (requires Serializable; no @ConversationScoped in ArC — use @SessionScoped as fallback)
- @Singleton EJB → @ApplicationScoped (use @Startup observer for eager init; @ConcurrencyManagement not portable)
- @Remove (on @Stateful) → remove annotation only; method becomes regular business method. Do NOT convert to @PreDestroy.
- JPA: @PersistenceContext → @Inject EntityManager  
- JAX-RS: @Path, @GET, @POST stay unchanged
- Servlet→JAX-RS: @WebServlet → @Path, doGet()/doPost() → @GET/@POST methods
- JSF: Use MyFaces extension → [details](references/jsf-migration-patterns.md)
- Remote EJB: Not supported → [migration strategies](references/pattern-remote-ejb-limitation.md)
- Application class removal: When removing @ApplicationPath class with a **NON-ROOT** custom path (anything other than "/"), set `quarkus.rest.path=/your-path` (RESTEasy Reactive) or `quarkus.resteasy.path=/your-path` (Classic). For root path "/", delete the class with no property addition.

## Reference Index

**All reference files ship with this transformation definition:**

| File | Purpose |
|---|---|
| [application-properties-checklist.md](references/application-properties-checklist.md) | Required configuration properties by extension |
| [arc-limitations.md](references/arc-limitations.md) | CDI ArC limitations and proxy restrictions |
| [arquillian-to-quarkustest.md](references/arquillian-to-quarkustest.md) | Test framework migration patterns |
| [batch-jberet-fallback.md](references/batch-jberet-fallback.md) | CDI-based batch processing when JBeret fails |
| [compatibility-matrix.md](references/compatibility-matrix.md) | Platform and extension compatibility matrix |
| [ear-consolidation.md](references/ear-consolidation.md) | Multi-module EAR to single JAR consolidation |
| [ejb-to-cdi-mapping.md](references/ejb-to-cdi-mapping.md) | Complete EJB to CDI conversion guide |
| [jms-to-smallrye.md](references/jms-to-smallrye.md) | JMS/MDB to SmallRye Reactive Messaging |
| [jpa-to-quarkus-persistence.md](references/jpa-to-quarkus-persistence.md) | JPA persistence context migration |
| [jsf-migration-patterns.md](references/jsf-migration-patterns.md) | JSF→MyFaces and JSF→Qute migration options |
| [pattern-remote-ejb-limitation.md](references/pattern-remote-ejb-limitation.md) | Remote EJB limitations and alternatives |
| [phase0-detection-flags.md](references/phase0-detection-flags.md) | Feature detection and conditional logic |
| [phases-detail.md](references/phases-detail.md) | Detailed phase-by-phase migration guide |
| [quarkus-extension-catalog.md](references/quarkus-extension-catalog.md) | Extension selection and dependency mapping |
| [security-migration.md](references/security-migration.md) | Security migration and SECURITY-NOTES.md spec |
| [troubleshooting-pitfalls.md](references/troubleshooting-pitfalls.md) | Troubleshooting guide and common migration pitfalls |
| [worked-examples-complete.md](references/worked-examples-complete.md) | Complete before/after migration examples |

## Known Limitations & Blockers

**Pre-Migration Blockers:**
- **REMOTE_EJB_DETECTED** — @Remote interfaces not supported → [migration strategies](references/pattern-remote-ejb-limitation.md)
- **EJB2_ENTITY_BEAN_FOUND** — EJB 2.x Entity Beans (`extends EntityBean`) are not supported in Quarkus and cannot be automatically migrated → HALT
- **ELYTRON_SECURITY_DOMAIN** — WildFly Elytron security domain configurations and EJB security context propagation are not portable; require redesign with quarkus-security/JWT → see [security-migration.md](references/security-migration.md)
- **JNDI_LOOKUPS** — InitialContext.lookup() fails at runtime. This is a **resolvable blocker**: must be resolved before proceeding, but has clear resolution paths. For static config lookups → replace with `@ConfigProperty`. For dynamic lookups that cannot be statically replaced → replace method body with `throw new UnsupportedOperationException("JNDI lookups not supported in Quarkus. Use @ConfigProperty.")`. See [troubleshooting-pitfalls.md](references/troubleshooting-pitfalls.md) for the pattern.
- **JAVAX_PINNED_BINARY_DEPENDENCY** — applies ONLY when BOTH hold: (1) the project is a genuine JavaEE/JakartaEE app (has EE APIs: Servlet, JAX-RS, EJB, JPA, JMS, CDI, JSF), AND (2) a **binary** dependency (no source in the repo) pins `javax.*` with no `jakarta.*` release. Resolution is a MUST-try sequence — do NOT halt until (a) and (b) are both exhausted:
  - (a) Find a `jakarta.*`-based replacement or newer version of the library.
  - (b) **MANDATORY before halting** — apply the Eclipse Transformer (`org.eclipse.transformer`) to rewrite `javax.*`->`jakarta.*` in the artifact at build time (see [namespace-migration guidance](references/troubleshooting-pitfalls.md)). A binary jar being javax-pinned is NOT sufficient reason to halt; the Transformer resolves the large majority of these cases (e.g. `rapla`/`restinject`).
  - (c) ONLY if (a) and (b) both genuinely fail, HALT and emit `BLOCKERS.md`.
  - Scope guard: this is about a *specific javax-pinned binary blocking an in-progress namespace migration*. **The mere presence of an unfamiliar or third-party dependency is NOT a blocker and MUST NOT downgrade a migratable JavaEE app to NOT_APPLICABLE / WRONG_FRAMEWORK.** If namespace migration has already started, prefer completing it via the Transformer over halting. Before declaring completion, verify with `grep -rc 'javax\.' src/`.

## Rollback & Abort
When Phase 0 detects a hard blocker (REMOTE_EJB_DETECTED, EJB2_ENTITY_BEAN_FOUND, ELYTRON_SECURITY_DOMAIN):
1. Emit `BLOCKERS.md` in the project root listing each blocker with detection evidence
2. Do NOT modify any source files
3. Exit with status code 1
4. User reviews BLOCKERS.md and decides: manual redesign OR skip app

For mid-migration failures:
- `git restore .` reverts all changes to pre-migration state
- Re-run with updated TD version after fixing the pattern in SKILL.md

## Exit Status
**0**: Success | **1**: Blocker detected (BLOCKERS.md emitted) | **NO_OP**: Already Quarkus | **WRONG_FRAMEWORK**: Spring Boot

**Non-Goals (require separate handling):**
- EJB Remote/IIOP, JCA Resource Adapters, EJB 2.x Entity Beans
- CDI Portable Extensions → must convert to Quarkus BuildExtension  
- XA Distributed Transactions, JSP compilation (use Facelets)
- HA Singleton (MSC/WildFly clustering) — replace with @Scheduled + external leader election (e.g., Kubernetes Lease)
- EJB Security Context Propagation across Remote EJB calls — see [pattern-remote-ejb-limitation.md](references/pattern-remote-ejb-limitation.md)
- Automatic BMT→CMT conversion — UserTransaction is preserved as-is when used

**ArC CDI Restrictions:**
- Normal-scoped beans need no-arg constructor
- @ApplicationScoped beans must be thread-safe
- See [ArC limitations](references/arc-limitations.md) for complete list

## Exit Criteria Checklist

**MANDATORY — All must pass:**
- [ ] `./mvnw clean verify` succeeds (compile + tests)
- [ ] No `javax.*` EE imports in src/main/java/ (excluding Java SE)
- [ ] javax.* check: `grep -rn --include='*.java' '^import javax\.' src/main/java/ src/test/java/ | grep -v 'javax\.sql\|javax\.crypto\|javax\.security\.auth\|javax\.net'` must return empty
- [ ] No EJB annotations: `grep -rn '@Stateless\|@EJB\|@TransactionAttribute' src/main/java/ | grep -v '^[[:space:]]*\(//\|\*\)'` returns empty (comment-filtered — Javadoc/`MIGRATION:` lines are not real annotations)
- [ ] No app server descriptors (persistence.xml, jboss-web.xml)
- [ ] No redundant web.xml or beans.xml (Quarkus uses application.properties and implicit CDI bean discovery)
- [ ] Dockerfile present & valid (STATIC, mandatory): `test -f src/main/docker/Dockerfile.jvm` and it references a Quarkus base image + copies `quarkus-app/` layers (verify by parse, NOT by building)
- [ ] `application.properties` present with a health/HTTP config (STATIC, mandatory)
- [ ] No JNDI lookups: `grep -rn 'InitialContext\|context\.lookup' src/main/java/ | grep -v '^[[:space:]]*\(//\|\*\)'` returns empty (comment-filtered)
- [ ] Clean build artifacts: delete any `.bak`/`.orig` files under `src/` before final scan

**Phase 0 HALT scoring:** If Phase 0 detects a hard blocker and correctly emits `BLOCKERS.md` without modifying source, the migration scores as **Exit Status 1** (blocker detected) — this is a CORRECT outcome, NOT a failure.

**ENVIRONMENT-DEPENDENT (non-blocking — mark NON-APPLICABLE if tooling absent):**
- [ ] Docker build: `docker build -f src/main/docker/Dockerfile.jvm -t test .` — run ONLY if a Docker/Podman CLI is available; if not, mark `NON-APPLICABLE (no container runtime)`, do NOT mark PARTIAL/FAIL
- [ ] Health check UP: `curl localhost:8080/q/health` (localhost only — not an external fetch; environment-dependent check) — run ONLY if the app can be started in the environment; else mark `NON-APPLICABLE (no runtime)`
- [ ] JSF namespace check: `grep -rn "jakarta\.face\." src/ | grep -v "jakarta\.faces\."` returns empty
- [ ] Deployment manifests (Helm/K8s) reference Quarkus image, correct ports, and health probes
- [ ] No WildFly/JBoss/quickstart references in deployment manifests: `grep -rn -E "wildfly|jboss|quickstart" charts/ k8s/ deploy/ helm/ 2>/dev/null` returns empty
- [ ] Helm/K8s manifests updated

**OVERALL STATUS rule (for `validation_summary.md`):**
- `COMPLETE` = every **applicable** criterion PASSes. A criterion that cannot be executed because the
  required tooling is absent (e.g. no Docker/Podman CLI, no runtime to start the app) counts as
  **NON-APPLICABLE**, NOT as PARTIAL or FAIL.
- Only emit `PARTIAL` when an **applicable, executable** criterion did not pass.
- Never let an environment-dependent check (Docker build, health curl) drag the overall status below
  COMPLETE. List it explicitly under a `NON-APPLICABLE CRITERIA` heading with the reason.
- Rationale: the benchmark scoring sandbox has no Docker CLI; treating container/runtime checks as hard
  exit criteria produces false PARTIAL results even when the migration is fully correct.

## Migration Phases — Checklist Format

**Phase 0: Project Analysis (ALWAYS)**
- [ ] **Complete ALL blocker scans before emitting BLOCKERS.md and halting** — do NOT short-circuit on the first hard blocker. A comprehensive BLOCKERS.md enables developers to plan all remediation in one pass.
- [ ] Scan for REMOTE_EJB_DETECTED → HALT if found
- [ ] Scan for EJB2_ENTITY_BEAN_FOUND: `grep -rn 'extends javax.ejb.EntityBean\|implements EntityBean' src/` → HALT if found (not supported)
- [ ] Scan for CDI Portable Extensions: `grep -rn 'implements Extension\|META-INF/services/javax.enterprise.inject.spi.Extension' src/` → WARNING if found (may need manual review)
- [ ] Scan for CONVERSATION_SCOPED: `grep -rn '@ConversationScoped' src/main/java/` → if found: set CONVERSATION_SCOPED_FALLBACK=true → Phase 2 will apply @SessionScoped fallback and emit warning to MIGRATION-WARNINGS.md
- [ ] Scan for ELYTRON_SECURITY_DOMAIN: `grep -rn 'security-domain\|@SecurityDomain\|Elytron\|EJBContext.*runAs\|SessionContext.*runAs' src/` AND `grep -l '<security-domain>\|security-domain=' jboss-ejb3.xml jboss-web.xml jboss-app.xml 2>/dev/null` → if found: **HALT** + emit BLOCKERS.md entry (EJB security context propagation not portable — requires redesign with quarkus-security/JWT; see [security-migration.md](references/security-migration.md))
- [ ] Set feature flags: EJB_NEEDED, JPA_NEEDED, JMS_NEEDED, JSF_NEEDED, SECURITY_NEEDED
- [ ] Detect JPA_NEEDED: `grep -rn '@Entity\|@PersistenceContext\|javax\.persistence\|jakarta\.persistence' src/main/java/` OR `find src/ -name 'persistence.xml'` → if found: set JPA_NEEDED=true
- [ ] Scan for BATCH_NEEDED: grep -rn 'jakarta.batch.api\|batch.xml\|@Named.*ItemReader\|@Named.*ItemProcessor\|@Named.*ItemWriter' src/ → if found: set BATCH_NEEDED=true → Phase 3 batch migration path (see references/batch-jberet-fallback.md)
- [ ] Scan for JNDI_LOOKUPS: `grep -rn 'new InitialContext\|context\.lookup' src/main/java/ | grep -v '^[[:space:]]*\(//\|\*\)'` (comment-filtered — migration `MIGRATION:`/Javadoc lines produce false positives) → if found: **resolve before proceeding** (replace with @ConfigProperty or UnsupportedOperationException — see [troubleshooting-pitfalls.md](references/troubleshooting-pitfalls.md))
- [ ] **MIGRATABILITY GUARD (positive check takes precedence over halt heuristics):** if the project uses ANY Jakarta/JavaEE API — `javax.servlet`/`jakarta.servlet`, `javax.ws.rs`/`jakarta.ws.rs`, `javax.ejb`, `javax.persistence`/`jakarta.persistence`, `javax.jms`, `javax.enterprise`/CDI, JSF — then it IS a migration target. Proceed with migration. Do NOT classify it as NOT_APPLICABLE just because it is packaged as a library, has an unfamiliar dependency, or lacks a `main()`. NOT_APPLICABLE is reserved for projects with ZERO EE APIs (pure Java SE) or non-Java-EE frameworks (Spring Boot → WRONG_FRAMEWORK).
- [ ] NO_OP check: if pom.xml already contains quarkus-bom AND no javax.* imports remain → skip migration, emit NO_OP status | Spring detection: if pom.xml contains spring-boot-starter-* → skip (wrong skill), emit WRONG_FRAMEWORK status
- Full details → [phases-detail.md](references/phases-detail.md)

**Phase 1: Build System & Namespace (ALWAYS)**  
- [ ] Add Quarkus BOM, change packaging WAR→JAR
- [ ] Build tool: if Gradle detected → use `./gradlew build` / `./gradlew test` equivalents; Gradle multi-module: see ear-consolidation.md. If BUILD COMMAND=mvn but no pom.xml exists (Gradle-only project): do NOT create a pom.xml. Verify via `./gradlew build` or static inspection (grep scans). See [troubleshooting-pitfalls.md](references/troubleshooting-pitfalls.md).
- [ ] Gradle + EAR multi-module: known high-iteration scenario (may require 3-4 passes). See ear-consolidation.md for details.
- [ ] Multi-module EAR/WAR consolidation: merge ejb-jar + war modules into single Quarkus module; remove ear packaging
- [ ] Namespace migration: javax.* → jakarta.* (EE packages only)
- [ ] Extract persistence.xml/web.xml → application.properties
- [ ] **IMPORTANT: Do NOT delete `src/main/resources/import.sql` or `data.sql`** — these are Quarkus-native data initialization files loaded by Hibernate ORM at startup. Only remove persistence.xml, web.xml, jboss-*.xml, beans.xml.
- [ ] Derive JDBC extension from persistence.xml: parse `jta-data-source` JNDI name or `javax.persistence.jdbc.driver` → add `quarkus-jdbc-{h2|postgresql|mysql|mssql|oracle}` in the same task that adds the Quarkus BOM
- [ ] Remove src/main/webapp/WEB-INF/web.xml (Quarkus uses application.properties)
- [ ] Remove META-INF/beans.xml or replace with empty <beans/> (Quarkus uses annotated bean discovery by default)
- [ ] Build passes: `./mvnw clean compile -Dmaven.test.skip=true`

**Phase 2: Core Migration (CONDITIONAL - if EJB_NEEDED or JPA_NEEDED)**
- [ ] EJB→CDI: @Stateless→@ApplicationScoped, @EJB→@Inject
- [ ] @Stateful → @SessionScoped: add `implements Serializable` + `private static final long serialVersionUID = 1L`
- [ ] @Remove → remove annotation only (method becomes regular business method; do NOT convert to @PreDestroy)
- [ ] If CONVERSATION_SCOPED_FALLBACK=true: Emit WARNING in MIGRATION-WARNINGS.md: "@ConversationScoped→@SessionScoped may cause state-leak across browser tabs in multi-step wizards. Manual review required for conversation boundaries."
- [ ] Validate @SessionScoped Serializable: `grep -rn '@SessionScoped' src/main/java/ | xargs grep -L 'implements Serializable'` must return empty
- [ ] JPA: @PersistenceContext→@Inject EntityManager  
- [ ] If import.sql or data.sql exists, verify table names match @Table(name=...) annotations or set `quarkus.hibernate-orm.physical-naming-strategy=org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl` in application.properties  
- [ ] JAX-RS: minimal changes, remove Application class. When removing @ApplicationPath class with a NON-ROOT custom path, add `quarkus.rest.path=/your-path` (RESTEasy Reactive) or `quarkus.resteasy.path=/your-path` (Classic) to application.properties. For root path "/", delete the class with no property addition.
- [ ] Security: @RolesAllowed works unchanged
- [ ] Build passes: `./mvnw clean compile`
- Detailed patterns → [ejb-to-cdi-mapping.md](references/ejb-to-cdi-mapping.md)

**Phase 3: Messaging & Security (CONDITIONAL — if JMS_NEEDED or SECURITY_NEEDED)**
- [ ] JMS destinations: Replace @Resource ConnectionFactory/@Resource Queue → quarkus-messaging-amqp or quarkus-messaging-kafka
- [ ] MDB → @Incoming/@Outgoing channels with SmallRye Reactive Messaging
- [ ] Add @Blocking on REST endpoints calling JMS/messaging (recurring fix)
- [ ] Validation: `grep -rn "@Channel\|JMSContext\|ConnectionFactory" src/main/java/` — if found in @Path classes without @Blocking → add @io.smallrye.common.annotation.Blocking
- [ ] Messaging validation: `grep -rn '@MessageDriven\|javax\.jms\|jakarta\.jms' src/main/java/ | grep -v '^[[:space:]]*\(//\|\*\)'` must return empty after migration (comment-filtered)
- [ ] Security: migrate web.xml security-constraint → @RolesAllowed + quarkus-elytron-security-properties-file

**Phase 4: Testing & UI (ALWAYS)**
- [ ] Test migration: @RunWith(Arquillian)→@QuarkusTest
- [ ] Remove all @Deployment/ShrinkWrap methods; for JMS/DB integration tests, add testcontainers (ActiveMQ Artemis, PostgreSQL) via @QuarkusTestResource
- [ ] JSF migration: MyFaces extension or JSF→Qute  
- [ ] JSF resources: `mkdir -p src/main/resources/META-INF/resources/` then move src/main/webapp/* (xhtml, images, css) → src/main/resources/META-INF/resources/
- [ ] Move faces-config.xml → src/main/resources/META-INF/
- [ ] JSF validation: `find src/main/webapp -name '*.xhtml' 2>/dev/null` should return empty
- [ ] Tests pass: `./mvnw clean test`
- [ ] Test count validation: `./mvnw test` must report non-zero test count (if original had tests)
- Test migration → [arquillian-to-quarkustest.md](references/arquillian-to-quarkustest.md)
- JSF patterns → [jsf-migration-patterns.md](references/jsf-migration-patterns.md)
- Note: `mockito-junit-jupiter` is NOT managed by the Quarkus BOM — retain explicit `<version>` tag.

**Phase 5: Deployment & Verification (ALWAYS)**
- [ ] `mkdir -p src/main/docker/` then generate Dockerfile.jvm, add health endpoints
- [ ] Docker builds and container starts
- [ ] Helm/K8s chart scan: `find charts/ k8s/ deploy/ helm/ -name '*.yaml' 2>/dev/null | xargs grep -l -E "wildfly|jboss|quickstart|/eap[0-9]" 2>/dev/null` OR `grep -rn -E "wildfly|jboss|quickstart|/eap[0-9]" charts/ k8s/ deploy/ helm/ README.md 2>/dev/null` — Update image refs, containerPort (8080), JAVA_OPTS, probes (/health → /q/health/live), Chart.yaml `home` and `sources` fields (strip WildFly/JBoss quickstart URLs), README badges referencing old app-server repos
- [ ] Generate SECURITY-NOTES.md listing pre-existing vulnerabilities detected during migration
- [ ] Final validation: `/q/health` returns UP
- Container setup → [phases-detail.md](references/phases-detail.md#phase-5-deployment--verification-always)

## Migration Patterns — Summary Table

| Pattern | Complexity | Outcome | Details |
|---|---|---|---|
| EJB→CDI | Medium | @Stateless→@ApplicationScoped | [ejb-to-cdi-mapping.md](references/ejb-to-cdi-mapping.md) |
| JPA Persistence | Low | @PersistenceContext→@Inject | [jpa-to-quarkus-persistence.md](references/jpa-to-quarkus-persistence.md) |
| JMS→SmallRye | Medium | @MessageDriven→@Incoming | [jms-to-smallrye.md](references/jms-to-smallrye.md) |
| Security Migration | Low | @RolesAllowed unchanged | [security-migration.md](references/security-migration.md) |
| JSF→MyFaces | Low | Minimal changes, MyFaces extension | [jsf-migration-patterns.md](references/jsf-migration-patterns.md) |
| JSF→Qute | High | Template rewrite required | [jsf-migration-patterns.md](references/jsf-migration-patterns.md) |
| Remote EJB | **BLOCKER** | Must replace with REST/gRPC | [pattern-remote-ejb-limitation.md](references/pattern-remote-ejb-limitation.md) |
| Arquillian Tests | Medium | @RunWith→@QuarkusTest | [arquillian-to-quarkustest.md](references/arquillian-to-quarkustest.md) |

## Compatibility Matrix

Quarkus 3.33.x LTS | JDK 17/21 | Maven 3.8+ | Hibernate ORM 6.x | Jakarta EE 10
See [references/compatibility-matrix.md](references/compatibility-matrix.md) for full matrix and version-sensitive extensions.

## Reference Dispatch

Load reference files on demand when encountering these signals:

| Signal in Source Code | Reference File |
|---|---|
| Phase 0 flag detection, pom.xml dep scanning | [phase0-detection-flags.md](references/phase0-detection-flags.md) |
| @Stateless, @EJB, @TransactionAttribute | [ejb-to-cdi-mapping.md](references/ejb-to-cdi-mapping.md) |
| ArC proxy errors, CDI extensions | [arc-limitations.md](references/arc-limitations.md) |
| persistence.xml, @PersistenceContext | [jpa-to-quarkus-persistence.md](references/jpa-to-quarkus-persistence.md) |
| @MessageDriven, JMSContext, @Incoming | [jms-to-smallrye.md](references/jms-to-smallrye.md) |
| @RolesAllowed, SessionContext, SECURITY-NOTES.md | [security-migration.md](references/security-migration.md) |
| .xhtml, @Named, @ViewScoped, JSF backing beans | [jsf-migration-patterns.md](references/jsf-migration-patterns.md) |
| @Remote EJB interfaces | [pattern-remote-ejb-limitation.md](references/pattern-remote-ejb-limitation.md) |
| @RunWith(Arquillian), @Deployment | [arquillian-to-quarkustest.md](references/arquillian-to-quarkustest.md) |
| Extension selection, dependency mapping | [quarkus-extension-catalog.md](references/quarkus-extension-catalog.md) |
| Complete worked examples | [worked-examples-complete.md](references/worked-examples-complete.md) |

## Common Pitfalls

See [references/troubleshooting-pitfalls.md](references/troubleshooting-pitfalls.md) for troubleshooting guide, bean management issues, threading problems, and migration mistakes.

## Tips & Best Practices

- [2025-05] **Key Version:** Always set `<quarkus.platform.version>` to the latest 3.33.x patch (LTS). Individual Quarkiverse extensions (like quarkus-cxf) may have their own version numbers that are compatible.
- [2025-05] @ApplicationScoped beans with mutable state → use @RequestScoped | JNDI lookups: `grep -rn 'InitialContext\|context\.lookup' src/main/java/ | grep -v '^[[:space:]]*\(//\|\*\)'` → must return empty (comment-filtered)
- [2025-05] JSF namespace: `grep -rn "jakarta\.face\." src/` → fix typos (missing 's') | @RolesAllowed needs security extension to work (not silently ignored)
- [2025-05] Use `quarkus.arc.remove-unused-beans=false` during migration, re-enable after | Remove static persistence.xml parsers → causes NPE in Quarkus
- [2025-05] Prefer constructor injection over @Inject on fields — improves testability and avoids ArC proxy issues on final/private fields
- [2025-05] ServiceLoader SPI → CDI @StaticInitSafe bean (required for Quarkus build-time initialization)
- [2025-06] NEVER delete a test whose target class still exists after migration. If a test mocks a removed JCA/EJB interface but the implementation survives as a CDI bean, REWRITE the mock against the CDI type. Deletion only when target class was removed.
- [2025-06] Post-migration check: compare test count before vs. after (`grep -rc '@Test' src/test`); a drop must be justified by removed target classes, not by convenience.
- [2026-07] `mockito-junit-jupiter` is NOT managed by the Quarkus BOM — always specify explicit `<version>` in pom.xml (e.g. `<version>5.11.0</version>`).
- [2026-07] When setting `quarkus.http.root-path` (from jboss-web.xml context-root), health endpoints move to `{root-path}/q/health/live`. To keep health at `/q/health`, set `quarkus.http.non-application-root-path=/q` explicitly.
- [2026-07] web.xml `<env-entry>` JNDI path ≠ `@ConfigProperty` key name. Grep source for `@ConfigProperty(name=...)` usages to discover actual key names before populating application.properties.

## Validation Commands

**Build Verification:**
```bash
./mvnw clean verify  # Must pass
# Or for Gradle: ./gradlew build
```

**Legacy Content Scans (must return ZERO results):**
```bash
# EJB annotations (comment-filtered — exclude Javadoc/MIGRATION comment lines)
grep -rn '@Stateless\|@EJB\|@TransactionAttribute' src/main/java/ | grep -v '^[[:space:]]*\(//\|\*\)'

# JNDI lookups (comment-filtered)
grep -rn 'InitialContext\|context\.lookup' src/main/java/ | grep -v '^[[:space:]]*\(//\|\*\)'

# App server descriptors
find . -name 'persistence.xml' -o -name 'jboss-web.xml' -o -name '*-ds.xml'

# javax.* imports (both main and test sources)
grep -rn --include='*.java' '^import javax\.' src/main/java/ src/test/java/ | grep -v 'javax\.sql\|javax\.crypto\|javax\.security\.auth\|javax\.net'
```

**Container Verification (localhost only — environment-dependent):**
```bash
docker build -f src/main/docker/Dockerfile.jvm -t test . && docker run -d -p 8080:8080 test && curl http://localhost:8080/q/health  # Must return {"status":"UP"}
```

Complete validation checklist → [phase0-detection-flags.md](references/phase0-detection-flags.md)
