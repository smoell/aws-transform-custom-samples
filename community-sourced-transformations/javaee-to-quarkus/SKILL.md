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
| Stateful EJB | quarkus-arc | @Stateful → @SessionScoped, conversational state preserved |
| JAX-RS | quarkus-rest | Minimal changes |
| Servlet | quarkus-rest | @WebServlet → @Path + @GET/POST |
| JPA | quarkus-hibernate-orm | @PersistenceContext → @Inject |
| JMS/MDB | quarkus-smallrye-reactive-messaging | MDB → @Incoming/@Outgoing channels, reactive messaging; requires @Blocking on REST endpoints calling JMS/messaging |
| JSF | quarkus-undertow + myfaces-quarkus | See [JSF→MyFaces pattern](references/pattern-jsf-myfaces.md) |
| Batch | quarkus-jberet (limited compat) — fallback: CDI-based @ApplicationScoped beans with manual ItemReader/Processor/Writer composition | See references/batch-jberet-fallback.md |
| Security | quarkus-security | @RolesAllowed unchanged |
| JASPIC | quarkus-security | ServerAuthModule → properties-file authentication, @RolesAllowed preserved |
| Health | quarkus-smallrye-health | Auto /q/health endpoints |
| OpenTelemetry | quarkus-opentelemetry | MicroProfile compatible, quarkus.otel.* config |
| REST Client | quarkus-rest-client | MicroProfile REST Client, quarkus.rest-client.* config |
| SOAP/JAX-WS | quarkiverse-cxf | @WebService unchanged, quarkus.cxf.endpoint.* config |
| @Schedule (EJB Timer) | quarkus-scheduler | @Schedule → @Scheduled; @Singleton timer bean → @ApplicationScoped |
| EE Concurrency (ManagedExecutorService) | quarkus-smallrye-context-propagation | @Resource ManagedExecutorService → @Inject ManagedExecutor |
| LRA | quarkus-narayana-lra | @LRA, @Compensate, @Complete unchanged |
| WebSocket | quarkus-websockets | @ServerEndpoint unchanged |
| Testcontainers | quarkus-test + testcontainers | Arquillian→@QuarkusTest, embedded containers for integration tests |
| JTA | quarkus-narayana-jta | @Transactional preserved, Narayana JTA, ACID properties maintained |
| BMT | quarkus-narayana-jta | @Inject UserTransaction works in Quarkus via quarkus-narayana-jta; prefer @Transactional unless explicit boundary control needed |

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
| [jsf-to-qute.md](references/jsf-to-qute.md) | JSF to Qute template migration |
| [pattern-jsf-myfaces.md](references/pattern-jsf-myfaces.md) | JSF preservation with MyFaces extension |
| [pattern-remote-ejb-limitation.md](references/pattern-remote-ejb-limitation.md) | Remote EJB limitations and alternatives |
| [phase0-detection-flags.md](references/phase0-detection-flags.md) | Feature detection and conditional logic |
| [phases-detail.md](references/phases-detail.md) | Detailed phase-by-phase migration guide |
| [quarkus-extension-catalog.md](references/quarkus-extension-catalog.md) | Extension selection and dependency mapping |
| [security-to-quarkus-security.md](references/security-to-quarkus-security.md) | Security configuration migration |
| [troubleshooting.md](references/troubleshooting.md) | Symptom diagnosis and fix table |
| [worked-examples-conditional.md](references/worked-examples-conditional.md) | Complete migration examples |
| [worked-examples.md](references/worked-examples.md) | Before/after code examples |

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
- Bean-Managed Transactions (UserTransaction) — prefer @Transactional; preserve UserTransaction only when business logic explicitly requires manual demarcation

**ArC CDI Restrictions:**
- Normal-scoped beans need no-arg constructor
- @ApplicationScoped beans must be thread-safe
- See [ArC limitations](references/arc-limitations.md) for complete list

## Exit Criteria Checklist

**MANDATORY — All must pass:**
- [ ] `./mvnw clean verify` succeeds (compile + tests)
- [ ] No `javax.*` EE imports in src/main/java/ (excluding Java SE)
- [ ] No EJB annotations (@Stateless, @EJB, @TransactionAttribute)
- [ ] No app server descriptors (persistence.xml, jboss-web.xml)
- [ ] Docker builds: `docker build -f src/main/docker/Dockerfile.jvm -t test .`
- [ ] Health check UP: `curl localhost:8080/q/health`
- [ ] No JNDI lookups: `grep -rn 'InitialContext\|context\.lookup' src/` returns empty
- [ ] JSF namespace check: `grep -rn "jakarta\.face\." src/ | grep -v "jakarta\.faces\."` returns empty

## Migration Phases — Checklist Format

**Phase 0: Project Analysis (ALWAYS)**
- [ ] Scan for REMOTE_EJB_DETECTED → HALT if found
- [ ] Scan for ELYTRON_SECURITY_DOMAIN: grep -rn 'security-domain\|@SecurityDomain\|Elytron' src/ -- → if found: HALT + emit blocker (see pattern-remote-ejb-limitation.md)
- [ ] Set feature flags: EJB_NEEDED, JMS_NEEDED, JSF_NEEDED, SECURITY_NEEDED
- [ ] Scan for BATCH_NEEDED: grep -rn 'jakarta.batch.api\|batch.xml\|@Named.*ItemReader\|@Named.*ItemProcessor\|@Named.*ItemWriter' src/ → if found: set BATCH_NEEDED=true → Phase 3 batch migration path
- [ ] NO_OP check: if pom.xml already contains quarkus-bom AND no javax.* imports remain → skip migration, emit NO_OP status
- [ ] Spring detection: if pom.xml contains spring-boot-starter-* → skip (wrong skill), emit WRONG_FRAMEWORK status
- [ ] Check for JNDI lookups → P0 blocker
- Full details → [phases-detail.md](references/phases-detail.md)

**Phase 1: Build System & Namespace (ALWAYS)**  
- [ ] Add Quarkus BOM, change packaging WAR→JAR
- [ ] Build tool: if Gradle detected → use `./gradlew build` / `./gradlew test` equivalents; Gradle multi-module: see ear-consolidation.md
- [ ] Multi-module EAR/WAR consolidation: merge ejb-jar + war modules into single Quarkus module; remove ear packaging
- [ ] Namespace migration: javax.* → jakarta.* (EE packages only)
- [ ] Extract persistence.xml/web.xml → application.properties
- [ ] Build passes: `./mvnw clean compile -Dmaven.test.skip=true`

**Phase 2: Core Migration (CONDITIONAL - if EJB detected)**
- [ ] EJB→CDI: @Stateless→@ApplicationScoped, @EJB→@Inject
- [ ] @Stateful → @SessionScoped: add `implements Serializable` + `private static final long serialVersionUID = 1L`
- [ ] JPA: @PersistenceContext→@Inject EntityManager  
- [ ] JAX-RS: minimal changes, remove Application class
- [ ] Security: @RolesAllowed works unchanged
- [ ] Build passes: `./mvnw clean compile`
- Detailed patterns → [ejb-to-cdi-mapping.md](references/ejb-to-cdi-mapping.md)

**Phase 3: Messaging & Security (CONDITIONAL — if JMS_NEEDED or SECURITY_NEEDED)**
- [ ] JMS destinations: Replace @Resource ConnectionFactory/@Resource Queue → quarkus-messaging-artemis or quarkus-messaging-kafka
- [ ] MDB → @Incoming/@Outgoing channels with SmallRye Reactive Messaging
- [ ] Add @Blocking on REST endpoints calling JMS/messaging (recurring fix)
- [ ] Validation: grep -rn "@Channel\|JMSContext\|ConnectionFactory" src/main/java/ — if found in @Path classes without @Blocking → add @io.smallrye.common.annotation.Blocking
- [ ] Security: if ELYTRON_SECURITY_DOMAIN detected → document as Non-Goal (pattern-remote-ejb-limitation.md)
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
- [ ] Scan for charts/**/values.yaml, k8s/**/*.yaml for wildfly|jboss image references — update to Quarkus image
- [ ] Update livenessProbe/readinessProbe paths: /health → /q/health/live and /q/health/ready
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

## Examples

See [references/worked-examples.md](references/worked-examples.md) for full before/after code examples:
- Servlet → JAX-RS REST resource
- EJB+JPA → CDI+Panache  
- MDB → @Incoming reactive messaging

## Troubleshooting

See [references/troubleshooting.md](references/troubleshooting.md) for the symptom → diagnostic → fix table.
Common quick-fixes: add @Blocking for JMS in REST; check ArC bean discovery with -Dquarkus.arc.unremovable-types; set quarkus.hibernate-orm.database.generation=drop-and-create for schema issues.

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
| @RolesAllowed, SessionContext | [security-to-quarkus-security.md](references/security-to-quarkus-security.md) |
| .xhtml, @Named, @ViewScoped, JSF backing beans | [jsf-to-qute.md](references/jsf-to-qute.md) |
| JSF preservation with minimal changes | [pattern-jsf-myfaces.md](references/pattern-jsf-myfaces.md) |
| @Remote EJB interfaces | [pattern-remote-ejb-limitation.md](references/pattern-remote-ejb-limitation.md) |
| @RunWith(Arquillian), @Deployment | [arquillian-to-quarkustest.md](references/arquillian-to-quarkustest.md) |
| Extension selection, dependency mapping | [quarkus-extension-catalog.md](references/quarkus-extension-catalog.md) |
| Complete worked examples | [worked-examples-conditional.md](references/worked-examples-conditional.md) |

## Common Pitfalls

**Bean Management:**
- Bean removed by unused-bean optimization → set `quarkus.arc.remove-unused-beans=false` during migration
- Final class cannot be proxied → add `quarkus.arc.transform-unproxyable-classes=true`
- Missing no-arg constructor → normal-scoped beans need no-arg constructor for ArC proxies

**Threading Issues:**
- @ApplicationScoped with mutable state → use @RequestScoped for per-request state
- Self-invocation bypasses interceptors → extract to separate bean for @Transactional

**Migration Mistakes:**  
- Incomplete namespace migration → mixed javax.*/jakarta.* causes compilation errors
- JNDI lookups remain → InitialContext fails at Quarkus runtime

## Tips & Best Practices

**Key Version:** Always set `<quarkus.platform.version>3.33.2</quarkus.platform.version>` (LTS)

**Critical Checks:**
- @ApplicationScoped beans with mutable state → use @RequestScoped
- JNDI lookups: `grep -rn 'InitialContext\|context\.lookup' src/` → must return empty
- JSF namespace: `grep -rn "jakarta\.face\." src/` → fix typos (missing 's')
- @RolesAllowed needs security extension to work (not silently ignored)

**Migration Efficiency:**
- Use `quarkus.arc.remove-unused-beans=false` during migration, re-enable after
- Scan for `Arc.container()` service locator anti-pattern → replace with @Inject  
- Remove static persistence.xml parsers → causes NPE in Quarkus
- JMS in REST endpoints needs @Blocking annotation
- Prefer constructor injection over @Inject on fields — improves testability and avoids ArC proxy issues on final/private fields

**Development Tools:**
- Quarkus Dev UI: `/q/dev-ui` for bean inspection and config debugging
- Debug logging: `quarkus.log.category."io.quarkus.arc".level=DEBUG` for CDI issues

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

# JSF namespace typos
grep -rn "jakarta\.face\." src/ | grep -v "jakarta\.faces\."
```

**Container Verification:**
```bash
docker build -f src/main/docker/Dockerfile.jvm -t test .
docker run -d -p 8080:8080 test
curl http://localhost:8080/q/health  # Must return {"status":"UP"}
```

Complete validation checklist → [phase0-detection-flags.md](references/phase0-detection-flags.md)