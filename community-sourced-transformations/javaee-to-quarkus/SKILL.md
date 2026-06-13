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

**Version**: 1.0 — Targets Quarkus 3.33.x LTS (review for 3.34+) | **Category**: javaee-to-quarkus

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

## Quick Reference

**Dependency Mapping:**
| JavaEE/Jakarta | Quarkus Extension | Notes |
|---|---|---|
| EJB | quarkus-arc | @Stateless → @ApplicationScoped |
| Stateful EJB | quarkus-arc | @Stateful → @SessionScoped — **session-scoped fallback; semantics differ from EJB @Stateful** (HTTP session lifecycle, shared across browser tabs; @ConversationScoped not supported in ArC; see ejb-to-cdi-mapping.md for state-leak warning) |
| JAX-RS | quarkus-rest | Minimal changes |
| Servlet | quarkus-rest | @WebServlet → @Path + @GET/POST |
| JPA | quarkus-hibernate-orm | @PersistenceContext → @Inject |
| JMS/MDB | quarkus-smallrye-reactive-messaging | MDB → @Incoming/@Outgoing channels, reactive messaging; requires @Blocking on REST endpoints calling JMS/messaging |
| JSF | quarkus-undertow + myfaces-quarkus | See [JSF→MyFaces pattern](references/pattern-jsf-myfaces.md) |
| Batch | quarkus-jberet (limited compat) — fallback: CDI-based @ApplicationScoped beans with manual ItemReader/Processor/Writer composition | See references/batch-jberet-fallback.md |
| Security | quarkus-security | @RolesAllowed unchanged |
| JASPIC | quarkus-security | ServerAuthModule → properties-file authentication, @RolesAllowed preserved |
| HttpAuthenticationMechanism | quarkus-security | Custom mechanism via HttpAuthenticationMechanism interface (same SPI, works natively) |
| Health | quarkus-smallrye-health | Auto /q/health endpoints |
| OpenTelemetry | quarkus-opentelemetry | MicroProfile compatible, quarkus.otel.* config |
| REST Client | quarkus-rest-client | MicroProfile REST Client, quarkus.rest-client.* config |
| SOAP/JAX-WS | quarkus-cxf | @WebService unchanged, quarkus.cxf.endpoint.* config |
| @Schedule (EJB Timer) | quarkus-scheduler | @Schedule → @Scheduled; @Singleton timer bean → @ApplicationScoped |
| EE Concurrency (ManagedExecutorService) | quarkus-smallrye-context-propagation | @Resource ManagedExecutorService → @Inject ManagedExecutor |
| LRA | quarkus-narayana-lra | @LRA, @Compensate, @Complete unchanged |
| WebSocket | quarkus-websockets | @ServerEndpoint unchanged |
| Testcontainers | quarkus-test + testcontainers | Arquillian→@QuarkusTest, embedded containers for integration tests |
| JTA | quarkus-narayana-jta | @Transactional preserved, Narayana JTA, ACID properties maintained |
| BMT | quarkus-narayana-jta | @Inject UserTransaction works in Quarkus via quarkus-narayana-jta; prefer @Transactional unless explicit boundary control needed; configure quarkus.transaction-manager.object-store-directory for crash recovery |

**Common Patterns:**
- EJB→CDI: @Stateless → @ApplicationScoped + @Transactional, @EJB → @Inject
- @Stateful EJB → @SessionScoped (requires Serializable; no @ConversationScoped in ArC — use @SessionScoped as fallback)
- @Singleton EJB → @ApplicationScoped (use @Startup observer for eager init; @ConcurrencyManagement not portable)
- JPA: @PersistenceContext → @Inject EntityManager  
- JAX-RS: @Path, @GET, @POST stay unchanged
- Servlet→JAX-RS: @WebServlet → @Path, doGet()/doPost() → @GET/@POST methods
- JSF: Use MyFaces extension → [details](references/pattern-jsf-myfaces.md)
- Remote EJB: Not supported → [migration strategies](references/pattern-remote-ejb-limitation.md)

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
- **ELYTRON_SECURITY_DOMAIN** — WildFly Elytron security domain configurations and EJB security context propagation are not portable; require redesign with quarkus-security/JWT → see pattern-remote-ejb-limitation.md
- **JNDI_LOOKUPS** — InitialContext.lookup() fails at runtime → replace with @ConfigProperty

## Rollback & Abort
When Phase 0 detects a hard blocker (REMOTE_EJB_DETECTED, ELYTRON_SECURITY_DOMAIN):
1. Emit `BLOCKERS.md` in the project root listing each blocker with detection evidence
2. Do NOT modify any source files
3. Exit with status code 1
4. User reviews BLOCKERS.md and decides: manual redesign OR skip app

For mid-migration failures:
- `git restore .` reverts all changes to pre-migration state
- Re-run with updated TD version after fixing the pattern in SKILL.md

**Non-Goals (require separate handling):**
- EJB Remote/IIOP, JCA Resource Adapters, EJB 2.x Entity Beans
- CDI Portable Extensions → must convert to Quarkus BuildExtension  
- XA Distributed Transactions, JSP compilation (use Facelets)
- HA Singleton (MSC/WildFly clustering) — replace with @Scheduled + external leader election (e.g., Kubernetes Lease)
- EJB Security Context Propagation across Remote EJB calls — see pattern-remote-ejb-limitation.md
- Automatic BMT→CMT conversion — UserTransaction is preserved as-is when used

**ArC CDI Restrictions:**
- Normal-scoped beans need no-arg constructor
- @ApplicationScoped beans must be thread-safe
- See [ArC limitations](references/arc-limitations.md) for complete list

## Exit Criteria Checklist

**MANDATORY — All must pass:**
- [ ] `./mvnw clean verify` succeeds (compile + tests)
- [ ] No `javax.*` EE imports in src/main/java/ (excluding Java SE)
- [ ] javax.* check: `grep -rn '^import javax\.' src/main/java/ | grep -v 'javax\.sql\|javax\.crypto\|javax\.security\.auth\|javax\.net\|javax\.naming'` must return empty
- [ ] No EJB annotations (@Stateless, @EJB, @TransactionAttribute)
- [ ] No app server descriptors (persistence.xml, jboss-web.xml)
- [ ] No redundant web.xml or beans.xml (Quarkus uses application.properties and implicit CDI bean discovery)
- [ ] Docker builds: `docker build -f src/main/docker/Dockerfile.jvm -t test .`
- [ ] Health check UP: `curl localhost:8080/q/health`
- [ ] No JNDI lookups: `grep -rn 'InitialContext\|context\.lookup' src/` returns empty
- [ ] JSF namespace check: `grep -rn "jakarta\.face\." src/ | grep -v "jakarta\.faces\."` returns empty
- [ ] Deployment manifests (Helm/K8s) reference Quarkus image, correct ports, and health probes
- [ ] Helm/K8s manifests updated

## Migration Phases — Checklist Format

**Phase 0: Project Analysis (ALWAYS)**
- [ ] Scan for REMOTE_EJB_DETECTED → HALT if found
- [ ] Scan for CONVERSATION_SCOPED: `grep -rn '@ConversationScoped' src/main/java/` → if found: set CONVERSATION_SCOPED_FALLBACK=true → Phase 2 will apply @SessionScoped fallback and emit warning to MIGRATION-WARNINGS.md
- [ ] Scan for ELYTRON_SECURITY_DOMAIN: `grep -rn 'security-domain\|@SecurityDomain\|Elytron\|SecurityContext.*runAs' src/` AND `grep -l '<security-domain>\|security-domain=' jboss-ejb3.xml jboss-web.xml jboss-app.xml 2>/dev/null` → if found: **HALT** + emit BLOCKERS.md entry (EJB security context propagation not portable — requires redesign with quarkus-security/JWT; see pattern-remote-ejb-limitation.md)
- [ ] Set feature flags: EJB_NEEDED, JMS_NEEDED, JSF_NEEDED, SECURITY_NEEDED
- [ ] Scan for BATCH_NEEDED: grep -rn 'jakarta.batch.api\|batch.xml\|@Named.*ItemReader\|@Named.*ItemProcessor\|@Named.*ItemWriter' src/ → if found: set BATCH_NEEDED=true → Phase 3 batch migration path
- [ ] NO_OP check: if pom.xml already contains quarkus-bom AND no javax.* imports remain → skip migration, emit NO_OP status | Spring detection: if pom.xml contains spring-boot-starter-* → skip (wrong skill), emit WRONG_FRAMEWORK status
- [ ] Check for JNDI lookups → P0 blocker
- Full details → [phases-detail.md](references/phases-detail.md)

**Phase 1: Build System & Namespace (ALWAYS)**  
- [ ] Add Quarkus BOM, change packaging WAR→JAR
- [ ] Build tool: if Gradle detected → use `./gradlew build` / `./gradlew test` equivalents; Gradle multi-module: see ear-consolidation.md
- [ ] Gradle + EAR multi-module: known high-iteration scenario (may require 3-4 passes). See ear-consolidation.md for details.
- [ ] Multi-module EAR/WAR consolidation: merge ejb-jar + war modules into single Quarkus module; remove ear packaging
- [ ] Namespace migration: javax.* → jakarta.* (EE packages only)
- [ ] Extract persistence.xml/web.xml → application.properties
- [ ] Build passes: `./mvnw clean compile -Dmaven.test.skip=true`

**Phase 2: Core Migration (CONDITIONAL - if EJB detected)**
- [ ] EJB→CDI: @Stateless→@ApplicationScoped, @EJB→@Inject
- [ ] @Stateful → @SessionScoped: add `implements Serializable` + `private static final long serialVersionUID = 1L`
- [ ] If CONVERSATION_SCOPED_FALLBACK=true: Emit WARNING in MIGRATION-WARNINGS.md: "@ConversationScoped→@SessionScoped may cause state-leak across browser tabs in multi-step wizards. Manual review required for conversation boundaries."
- [ ] Validate @SessionScoped Serializable: `grep -rn '@SessionScoped' src/main/java/ | xargs grep -L 'implements Serializable'` must return empty
- [ ] JPA: @PersistenceContext→@Inject EntityManager  
- [ ] If import.sql or data.sql exists, verify table names match @Table(name=...) annotations or set `quarkus.hibernate-orm.physical-naming-strategy=org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl` in application.properties  
- [ ] JAX-RS: minimal changes, remove Application class
- [ ] Security: @RolesAllowed works unchanged
- [ ] Build passes: `./mvnw clean compile`
- Detailed patterns → [ejb-to-cdi-mapping.md](references/ejb-to-cdi-mapping.md)

**Phase 3: Messaging & Security (CONDITIONAL — if JMS_NEEDED or SECURITY_NEEDED)**
- [ ] JMS destinations: Replace @Resource ConnectionFactory/@Resource Queue → quarkus-messaging-artemis or quarkus-messaging-kafka
- [ ] MDB → @Incoming/@Outgoing channels with SmallRye Reactive Messaging
- [ ] Add @Blocking on REST endpoints calling JMS/messaging (recurring fix)
- [ ] Validation: grep -rn "@Channel\|JMSContext\|ConnectionFactory" src/main/java/ — if found in @Path classes without @Blocking → add @io.smallrye.common.annotation.Blocking
- [ ] Security: migrate web.xml security-constraint → @RolesAllowed + quarkus-elytron-security-properties-file

**Phase 4: Testing & UI (ALWAYS)**
- [ ] Test migration: @RunWith(Arquillian)→@QuarkusTest
- [ ] Remove all @Deployment/ShrinkWrap methods; for JMS/DB integration tests, add testcontainers (ActiveMQ Artemis, PostgreSQL) via @QuarkusTestResource
- [ ] JSF migration: MyFaces extension or JSF→Qute  
- [ ] Tests pass: `./mvnw clean test`
- Test migration → [arquillian-to-quarkustest.md](references/arquillian-to-quarkustest.md)
- JSF patterns → [jsf-to-qute.md](references/jsf-to-qute.md)

**Phase 5: Deployment & Verification (ALWAYS)**
- [ ] Generate Dockerfile.jvm, add health endpoints
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
| Security Migration | Low | @RolesAllowed unchanged | [security-to-quarkus-security.md](references/security-to-quarkus-security.md) |
| JSF→MyFaces | Low | Minimal changes, MyFaces extension | [pattern-jsf-myfaces.md](references/pattern-jsf-myfaces.md) |
| JSF→Qute | High | Template rewrite required | [jsf-to-qute.md](references/jsf-to-qute.md) |
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

**Key Version:** Always set `<quarkus.platform.version>3.33.2</quarkus.platform.version>` (LTS). Individual Quarkiverse extensions (like quarkus-cxf) may have their own version numbers that are compatible.

**Critical Checks:**
- @ApplicationScoped beans with mutable state → use @RequestScoped | JNDI lookups: `grep -rn 'InitialContext\|context\.lookup' src/` → must return empty
- JSF namespace: `grep -rn "jakarta\.face\." src/` → fix typos (missing 's') | @RolesAllowed needs security extension to work (not silently ignored)

**Migration Efficiency:**
- Use `quarkus.arc.remove-unused-beans=false` during migration, re-enable after | Remove static persistence.xml parsers → causes NPE in Quarkus
- Prefer constructor injection over @Inject on fields — improves testability and avoids ArC proxy issues on final/private fields
- ServiceLoader SPI → CDI @StaticInitSafe bean (required for Quarkus build-time initialization)

## Validation Commands

**Build Verification:**
```bash
./mvnw clean verify  # Must pass
```

**Legacy Content Scans (must return ZERO results):**
```bash
# EJB annotations
grep -rn '@Stateless\|@EJB\|@TransactionAttribute' src/main/java/

# JNDI lookups  
grep -rn 'InitialContext\|context\.lookup' src/main/java/

# App server descriptors
find . -name 'persistence.xml' -o -name 'jboss-web.xml' -o -name '*-ds.xml'
```

**Container Verification:**
```bash
docker build -f src/main/docker/Dockerfile.jvm -t test . && docker run -d -p 8080:8080 test && curl http://localhost:8080/q/health  # Must return {"status":"UP"}
```

Complete validation checklist → [phase0-detection-flags.md](references/phase0-detection-flags.md)