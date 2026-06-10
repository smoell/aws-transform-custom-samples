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

## Quick Reference

**Dependency Mapping:**
| JavaEE/Jakarta | Quarkus Extension | Notes |
|---|---|---|
| EJB | quarkus-arc | @Stateless → @ApplicationScoped |
| Stateful EJB | quarkus-arc | @Stateful → @SessionScoped, conversational state preserved |
| JAX-RS | quarkus-rest | Minimal changes |
| Servlet | quarkus-rest | @WebServlet → @Path + @GET/POST |
| JPA | quarkus-hibernate-orm | @PersistenceContext → @Inject |
| JMS/MDB | quarkus-smallrye-reactive-messaging | MDB → @Incoming/@Outgoing channels, reactive messaging |
| JSF | quarkus-undertow + myfaces-quarkus | See [JSF→MyFaces pattern](references/pattern-jsf-myfaces.md) |
| Batch | quarkus-jberet | JSR 352 → JBeret, ItemReader/Processor/Writer preserved, javax.batch→jakarta.batch |
| Security | quarkus-security | @RolesAllowed unchanged |
| JASPIC | quarkus-security | ServerAuthModule → properties-file authentication, @RolesAllowed preserved |
| Health | quarkus-smallrye-health | Auto /q/health endpoints |
| OpenTelemetry | quarkus-opentelemetry | MicroProfile compatible, quarkus.otel.* config |
| REST Client | quarkus-rest-client | MicroProfile REST Client, quarkus.rest-client.* config |
| SOAP/JAX-WS | quarkiverse-cxf | @WebService unchanged, quarkus.cxf.endpoint.* config |
| LRA | quarkus-narayana-lra | @LRA, @Compensate, @Complete unchanged |
| WebSocket | quarkus-websockets | @ServerEndpoint unchanged |
| Testcontainers | quarkus-test + testcontainers | Arquillian→@QuarkusTest, embedded containers for integration tests |
| JTA | quarkus-narayana-jta | @Transactional preserved, Narayana JTA, ACID properties maintained |

**Common Patterns:**
- EJB→CDI: @Stateless → @ApplicationScoped + @Transactional, @EJB → @Inject
- JPA: @PersistenceContext → @Inject EntityManager  
- JAX-RS: @Path, @GET, @POST stay unchanged
- Servlet→JAX-RS: @WebServlet → @Path, doGet()/doPost() → @GET/@POST methods
- JSF: Use MyFaces extension → [details](references/pattern-jsf-myfaces.md)
- Remote EJB: Not supported → [migration strategies](references/pattern-remote-ejb-limitation.md)

## Known Limitations & Blockers

**Pre-Migration Blockers:**
- **REMOTE_EJB_DETECTED** — @Remote interfaces not supported → [migration strategies](references/pattern-remote-ejb-limitation.md)
- **JNDI_LOOKUPS** — InitialContext.lookup() fails at runtime → replace with @ConfigProperty

**Non-Goals (require separate handling):**
- EJB Remote/IIOP, JCA Resource Adapters, EJB 2.x Entity Beans
- CDI Portable Extensions → must convert to Quarkus BuildExtension  
- XA Distributed Transactions, JSP compilation (use Facelets)

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
- [ ] Set feature flags: EJB_NEEDED, JMS_NEEDED, JSF_NEEDED, SECURITY_NEEDED
- [ ] Check for JNDI lookups → P0 blocker
- Full details → [phases-detail.md](references/phases-detail.md)

**Phase 1: Build System & Namespace (ALWAYS)**  
- [ ] Add Quarkus BOM, change packaging WAR→JAR
- [ ] Namespace migration: javax.* → jakarta.* (EE packages only)
- [ ] Extract persistence.xml/web.xml → application.properties
- [ ] Build passes: `./mvnw clean compile -Dmaven.test.skip=true`

**Phase 2: Core Migration (CONDITIONAL - if EJB detected)**
- [ ] EJB→CDI: @Stateless→@ApplicationScoped, @EJB→@Inject
- [ ] JPA: @PersistenceContext→@Inject EntityManager  
- [ ] JAX-RS: minimal changes, remove Application class
- [ ] Security: @RolesAllowed works unchanged
- [ ] Build passes: `./mvnw clean compile`
- Detailed patterns → [ejb-to-cdi-mapping.md](references/ejb-to-cdi-mapping.md)

**Phase 4: Testing & UI (ALWAYS)**
- [ ] Test migration: @RunWith(Arquillian)→@QuarkusTest
- [ ] JSF migration: MyFaces extension or JSF→Qute  
- [ ] Tests pass: `./mvnw clean test`
- Test migration → [arquillian-to-quarkustest.md](references/arquillian-to-quarkustest.md)
- JSF patterns → [jsf-to-qute.md](references/jsf-to-qute.md)

**Phase 5: Deployment & Verification (ALWAYS)**
- [ ] Generate Dockerfile.jvm, add health endpoints
- [ ] Docker builds and container starts
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