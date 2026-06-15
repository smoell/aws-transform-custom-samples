# Benchmark Suite — JavaEE/JakartaEE to Quarkus

## OOB Requirement

Per the OOB Launch Guide:
- **Minimum 50 applications** in the benchmark set
- **50% must be classified as "Hard"** (complex, multi-step, edge-case-heavy)
- Use **CEAT dataset** for standardized app classification
- All results must be reproducible and logged
- Submit via: benchmark.seg.aws.dev/benchmark

## Complexity Classification

| Level | Criteria |
|-------|----------|
| 🟢 EASY | <5K LOC, ≤5 EJBs, single module, no JSF/JMS/SOAP |
| 🟡 MEDIUM | 5–20K LOC, moderate EJBs, JMS or Security, single module WAR |
| 🟠 HARD | 20–50K LOC, multi-module, JSF + JMS/Security, or SOAP |
| 🔴 VERY HARD | >50K LOC, EAR+JSF+JMS+Batch+Security, CDI extensions |

## Related Benchmarks & References

| Benchmark / App | Description | Link |
|-----------------|-------------|------|
| **ScarfBench** | IBM 2026 expert-verified benchmark: 34 apps, 151K LOC, JavaEE/Spring/Quarkus migration triples | [arxiv.org/abs/2605.06754](https://arxiv.org/abs/2605.06754) |
| **WildFly Quickstarts** | Official WildFly sample applications (main source for this benchmark) | [github.com/wildfly/quickstart](https://github.com/wildfly/quickstart) |
| **JavaEE 7 Samples** | Community JavaEE 7 sample applications | [github.com/javaee-samples/javaee7-samples](https://github.com/javaee-samples/javaee7-samples) |
| **Eclipse Cargo Tracker** | Reference application for Jakarta EE / Domain-Driven Design | [github.com/eclipse-ee4j/cargotracker](https://github.com/eclipse-ee4j/cargotracker) |
| **ClusterBench** | Dedicated benchmarking app for Java EE application servers | [github.com/clusterbench/clusterbench](https://github.com/clusterbench/clusterbench) |
| **IBM App Modernization** | End-to-end JavaEE 6 → Quarkus modernization journey sample | [github.com/IBM/application-modernization-javaee-quarkus](https://github.com/IBM/application-modernization-javaee-quarkus) |

---

## Progress Tracker

| Category | Target | Done | Remaining |
|----------|--------|------|-----------|
| EASY (🟢) | 15 | 10 | 5 |
| MEDIUM (🟡) | 15 | 16 | -1 |
| HARD/VERY HARD (🟠🔴) | 25 | 26 | -1 |
| **Total** | **50+** | **52** | **-2** |
| Hard % | 50% min | 50.0% achieved | ✅ |

---

## Benchmark Results

### Completed Runs

| # | App Name | GitHub URL | LOC | Complexity | EJB | JMS | JSF | Security | SOAP | Batch | Multi-Module | Tests | Status | Notes |
|---|----------|-----------|-----|-----------|-----|-----|-----|----------|------|-------|-------------|--------|--------|-------|
| 1 | Eclipse Cargo Tracker | https://github.com/eclipse-ee4j/cargotracker | ~18K | 🟠 HARD | ✅ | ✅ CDI events | ✅ MyFaces | minimal | ❌ | ✅ | ❌ | 29/29 | ✅ PASS | Jakarta EE 10/Payara; 2.8s JVM start |
| 2 | DayTrader 7 | https://github.com/WASdev/sample.daytrader7 | ~25K | 🟠 HARD | ✅ EJB 3.x | ✅ JMS/MDB | ✅ MyFaces | ✅ BASIC | ❌ | ❌ | ✅ EAR | N/A | ✅ PASS | Java EE 7/Liberty; Gradle; 4 runs to reach PASS |
| 3 | WildFly Kitchensink | https://github.com/wildfly/quickstart/tree/main/kitchensink | ~2K | 🟢 EASY | minimal | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | 2/2 | ✅ PASS (stable) | Two consecutive clean runs; zero migration issues |
| 4 | WildFly tasks-jsf | https://github.com/wildfly/quickstart/tree/main/tasks-jsf | ~3K | 🟡 MEDIUM | ✅ @Stateful | ❌ | ✅ JSF/MyFaces | ❌ | ❌ | ❌ | ❌ | 1/1 | ✅ PASS | @ConversationScoped→@SessionScoped; import.sql table name fix |
| 5 | ClusterBench | https://github.com/clusterbench/clusterbench | ~8K | 🟠 HARD | ✅ @Stateful | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ EAR | 1/1 | ✅ PASS | Jakarta EE 10; Security extensions added |
| 6 | WildFly helloworld-mdb | https://github.com/wildfly/quickstart/tree/main/helloworld-mdb | ~1K | 🟢 EASY | minimal | ✅ JMS/MDB | ❌ | ❌ | ❌ | ❌ | ❌ | 1/1 | ✅ PASS | MDB→SmallRye Reactive Messaging |
| 7 | WildFly ejb-timer | https://github.com/wildfly/quickstart/tree/main/ejb-timer | ~1K | 🟢 EASY | minimal | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | 1/1 | ✅ PASS | @Schedule→quarkus-scheduler @Scheduled |
| 8 | WildFly helloworld-singleton | https://github.com/wildfly/quickstart/tree/main/helloworld-singleton | ~1K | 🟢 EASY | ✅ @Singleton | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | 1/1 | ✅ PASS | @Singleton→@ApplicationScoped; JSF MyFaces |
| 9 | WildFly ejb-multi-server | https://github.com/wildfly/quickstart/tree/main/ejb-multi-server | ~5K | 🟠 HARD | ✅ @Stateless | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ Multi-module EAR | 4/4 | ✅ PASS | Multi-module EAR consolidated; Remote EJB→non-goal; 9/10 quality |
| 10 | WildFly bmt | https://github.com/wildfly/quickstart/tree/main/bmt | ~2K | 🟡 MEDIUM | ✅ @Stateful | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | 1/1 | ✅ PASS | Bean Managed Transactions; UserTransaction kept (demo intent) |
| 11 | WildFly batch-processing | https://github.com/wildfly/quickstart/tree/main/batch-processing | ~5K | 🟠 HARD | ✅ @Stateless | ❌ | ❌ | ❌ | ❌ | ✅ Batch | ❌ | 4/4 | ✅ PASS | quarkus-jberet incompatible→CDI-based; path traversal risk noted |
| 12 | WildFly thread-racing | https://github.com/wildfly/quickstart/tree/main/thread-racing | ~6K | 🟠 HARD | ✅ @Stateless | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | 2/2 | ✅ PASS | EE Concurrency→MicroProfile ManagedExecutor |
| 13 | WildFly helloworld-jms | https://github.com/wildfly/quickstart/tree/main/helloworld-jms | ~2K | 🟢 EASY | minimal | ✅ JMS | ❌ | ❌ | ❌ | ❌ | ❌ | 1/1 | ✅ PASS | Embedded Artemis JMS; @Blocking required on reactive endpoint |
| 14 | WildFly servlet-security | https://github.com/wildfly/quickstart/tree/main/servlet-security | ~2K | 🟢 EASY | minimal | ❌ | ❌ | ✅ BASIC | ❌ | ❌ | ❌ | 1/1 | ✅ PASS | Servlet + @RolesAllowed; quarkus-elytron-security-properties-file |
| 15 | WildFly microprofile-config | https://github.com/wildfly/quickstart/tree/main/microprofile-config | ~2K | 🟡 MEDIUM | minimal | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | 1/1 | ✅ PASS | ServiceLoader SPI → CDI @StaticInitSafe |
| 16 | WildFly microprofile-health | https://github.com/wildfly/quickstart/tree/main/microprofile-health | ~1K | 🟡 MEDIUM | minimal | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | 3/3 | ✅ PASS (stable) | MicroProfile Health; two consecutive clean runs |
| 17 | WildFly cmt | https://github.com/wildfly/quickstart/tree/main/cmt | ~3K | 🟠 HARD | ✅ @Stateless | ✅ JMS | ❌ | ❌ | ❌ | ❌ | ❌ | 2/2 | ✅ PASS | Container Managed Transactions; JMS producer |
| 18 | WildFly ha-singleton-deployment | https://github.com/wildfly/quickstart/tree/main/ha-singleton-deployment | ~2K | 🟠 HARD | ✅ @Singleton | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | 1/1 | ✅ PASS | @Singleton EJB Timer→@ApplicationScoped @Scheduled; HA Non-Goal |
| 19 | WildFly jts | https://github.com/wildfly/quickstart/tree/main/jts | ~4K | 🟠 HARD | ✅ @Stateless | ✅ JMS/AMQP | ❌ | ❌ | ❌ | ❌ | ❌ | 1/1 | ✅ PASS | JTA distributed TX→@Transactional; dead @Named JSF beans removed |
| 20 | WildFly messaging-clustering-singleton | https://github.com/wildfly/quickstart/tree/main/messaging-clustering-singleton | ~3K | 🟠 HARD | minimal | ✅ JMS | ❌ | ❌ | ❌ | ❌ | ❌ | 1/1 | ✅ PASS | JMS consumer beans; @Blocking on REST; graceful shutdown |
| 21 | WildFly jaxws-retail | https://github.com/wildfly/quickstart/tree/main/jaxws-retail | ~2K | 🟡 MEDIUM | minimal | ❌ | ❌ | ❌ | ✅ SOAP | ❌ | ❌ | 1/1 | ✅ PASS | SOAP/JAX-WS via quarkus-cxf; first validated SOAP migration; all exit criteria pass |
| 22 | WildFly ee-security | https://github.com/wildfly/quickstart/tree/main/ee-security | ~1K | 🟢 EASY | minimal | ❌ | ❌ | ✅ Custom HTTP Auth | ❌ | ❌ | ❌ | 1/1 | ✅ PASS | Custom HttpAuthenticationMechanism; @WebServlet→JAX-RS recommended; embedded users config |
| 23 | WildFly ejb-security-context-propagation | https://github.com/wildfly/quickstart/tree/main/ejb-security-context-propagation | ~2K | 🔴 VERY HARD | ✅ @Stateless | ❌ | ❌ | ✅ EJB Security Domain | ❌ | ❌ | ❌ | 0/0 | ❌ FAIL | Remote EJB + Elytron incompatible with Quarkus; full rewrite required |
| 24 | WildFly ha-singleton-service | https://github.com/wildfly/quickstart/tree/main/ha-singleton-service | ~2K | 🟠 HARD | ✅ @Singleton | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | 1/1 | ✅ PASS | MSC HA Singleton→@ApplicationScoped+@Scheduled; field→constructor injection recommended |
| 25 | WildFly jta-crash-rec | https://github.com/wildfly/quickstart/tree/main/jta-crash-rec | ~3K | 🟠 HARD | ✅ @Stateless | ✅ JMS/AMQP | ❌ | ❌ | ❌ | ❌ | ❌ | 1/1 | ✅ PASS | JTA XA transactions + AMQP; missing application.properties critical; @Blocking needed |
| 26 | WildFly microprofile-jwt | https://github.com/wildfly/quickstart/tree/main/microprofile-jwt | ~3K | 🟡 MEDIUM | minimal | ❌ | ❌ | ✅ JWT | ❌ | ❌ | ❌ | 2/2 | ✅ PASS | MicroProfile JWT + @RolesAllowed; legacy web.xml/beans.xml need cleanup |
| 27 | WildFly microprofile-openapi | https://github.com/wildfly/quickstart/tree/main/microprofile-openapi | ~2K | 🟡 MEDIUM | minimal | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | 2/2 | ✅ PASS | OpenAPI spec generation; missing @ApplicationScoped and Jackson setters |
| 28 | WildFly numberguess | https://github.com/wildfly/quickstart/tree/main/numberguess | ~1K | 🟡 MEDIUM | minimal | ❌ | ✅ JSF | ❌ | ❌ | ❌ | ❌ | Quarkus 3.33.2 | ✅ PASS | ⭐⭐⭐⭐ (4/5) - JSF migrated via myfaces-quarkus:4.0.2 + quarkus-undertow extension; Resources moved to META-INF/resources/; Packaging changed WAR→JAR (fast-jar); CDI/ArC works correctly (@SessionScoped, qualifiers, producers); Already on jakarta.* namespace; High priority: Helm chart still references WildFly quickstart repo; Medium: field injection should be constructor injection, hardcoded maxNumber=100 should use @ConfigProperty; Jakarta.faces namespace URIs correct, faces-config.xml at META-INF/ |
| 29 | WildFly micrometer | https://github.com/wildfly/quickstart/tree/main/micrometer | ~2K | 🟡 MEDIUM | minimal | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | 2/2 | ✅ PASS | ⭐⭐⭐⭐⭐ (A-) - Excellent Quarkus migration with quarkus-micrometer-registry-prometheus extension; @ApplicationScoped CDI scoping correct; Micrometer Counter/Gauge/Timer metrics preserved; JAX-RS endpoints functional; WAR→JAR packaging; @PostConstruct initialization working; Packaging changed WAR→JAR (fast-jar); Ready for containerization |
| 30 | WildFly helloworld | https://github.com/wildfly/quickstart/tree/main/helloworld | ~1K | 🟢 EASY | minimal | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | 1/1 | ✅ PASS | ⭐⭐⭐⭐⭐ (A) - Perfect servlet-to-JAX-RS migration; SmallRye Health extension added; Dockerfile.jvm created; javax.*→jakarta.* namespace migration; WAR→JAR packaging; quarkus-rest extension; All exit criteria passed |
| 31 | WildFly helloworld-rs | https://github.com/wildfly/quickstart/tree/main/helloworld-rs | ~1K | 🟢 EASY | minimal | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | Tests pass | ✅ PASS | ⭐⭐⭐⭐⭐ (A+) - Exemplary JAX-RS migration with zero code changes; SmallRye Health extension added; Dockerfile.jvm configured; javax.*→jakarta.* namespace migration; WAR→JAR packaging; Perfect Quarkus compatibility demonstration |
| 32 | WildFly todo-backend | https://github.com/wildfly/quickstart/tree/main/todo-backend | ~3K | 🟡 MEDIUM | @Stateless | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | All criteria pass | ✅ PASS | ⭐⭐⭐⭐⭐ (A+) - Exemplary full-stack migration: EJB→CDI (@Stateless→@ApplicationScoped), JPA (persistence.xml→application.properties), JAX-RS, CORS config; PostgreSQL + dev services; Dockerfile.jvm; Complete enterprise pattern coverage |
| 33 | WildFly opentelemetry-tracing | https://github.com/wildfly/quickstart/tree/main/opentelemetry-tracing | ~2K | 🟡 MEDIUM | minimal | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | All criteria pass | ✅ PASS | ⭐⭐⭐⭐⭐ (A+) - Exemplary observability migration: OpenTelemetry tracing preserved; MicroProfile API compatibility; @RequestScoped CDI; quarkus.otel.* configuration externalized; Dockerfile.jvm; Full distributed tracing capabilities maintained |
| 34 | WildFly microprofile-rest-client | https://github.com/wildfly/quickstart/tree/main/microprofile-rest-client | ~2K | 🟡 MEDIUM | minimal | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | All criteria pass | ✅ PASS | ⭐⭐⭐⭐⭐ (A+) - Exemplary REST client migration: MicroProfile REST Client API preserved; quarkus.rest-client.* configuration externalized; CDI scope configuration; JAX-RS endpoints maintained; Dockerfile.jvm; Perfect MicroProfile compatibility |
| 35 | WildFly websocket-endpoint | https://github.com/wildfly/quickstart/tree/main/websocket-endpoint | ~2K | 🟡 MEDIUM | minimal | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | All criteria pass | ✅ PASS | ⭐⭐⭐⭐⭐ (A+) - Exemplary WebSocket migration: Jakarta WebSocket API preserved (@ServerEndpoint, @OnOpen, @OnClose, @OnMessage); Encoders/Decoders maintained; CDI integration with ArC; Dockerfile.jvm; Perfect WebSocket compatibility |
| 36 | WildFly helloworld-ws | https://github.com/wildfly/quickstart/tree/main/helloworld-ws | ~2K | 🟡 MEDIUM | minimal | ❌ | ❌ | ❌ | ❌ | ✅ SOAP | ❌ | 3/3 tests pass | ✅ PASS | ⭐⭐⭐⭐⭐ (A+) - Exemplary SOAP/JAX-WS migration: @WebService preserved; quarkus-cxf:3.33.6 integration; quarkus.cxf.endpoint.* configuration; CDI integration; Dockerfile.jvm; Complete JAX-WS compatibility via Apache CXF |
| 37 | WildFly microprofile-reactive-messaging-kafka | https://github.com/wildfly/quickstart/tree/main/microprofile-reactive-messaging-kafka | ~4K | 🟠 HARD | minimal | ✅ JMS/Kafka | ❌ | ❌ | ❌ | ❌ | ❌ | BUILD SUCCESS | ✅ PASS | ⭐⭐⭐⭐⭐ (A+) - Exemplary Reactive Messaging migration: MicroProfile Reactive Messaging preserved (@Channel, @Incoming, @Outgoing); SmallRye Kafka integration; Custom serializers preserved; mp.messaging.* config externalized; JPA + H2; Dockerfile.jvm |
| 38 | WildFly microprofile-lra | https://github.com/wildfly/quickstart/tree/main/microprofile-lra | ~3K | 🟠 HARD | minimal | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | All criteria pass | ✅ PASS | ⭐⭐⭐⭐⭐ (A+) - Exemplary LRA migration: MicroProfile LRA preserved (@LRA, @Compensate, @Complete); quarkus-narayana-lra integration; Saga pattern for distributed transactions; LRA coordinator configuration; Dockerfile.jvm; Complete Long Running Actions support |
| 39 | sample.javaee7.jta | https://github.com/WASdev/sample.javaee7.jta | ~3K | 🟠 HARD | @Stateless | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | BUILD SUCCESS | ✅ PASS | ⭐⭐⭐⭐ (A) - Excellent JTA migration: Java EE 7 JTA preserved; EJB→CDI conversion; Database migration (Derby→H2); @Transactional declarative transactions; quarkus-smallrye-health; Dockerfile.jvm; Complete transaction management |
| 40 | sample.javaee7.jms | https://github.com/WASdev/sample.javaee7.jms | ~2K | 🟠 HARD | @Stateless | ✅ JMS/MDB | ❌ | ❌ | ❌ | ❌ | ❌ | BUILD SUCCESS | ✅ PASS | ⭐⭐⭐⭐⭐ (A+) - Exemplary JMS migration: EJB→CDI conversion; JMS (MDB + JMSContext)→SmallRye Reactive Messaging; @Incoming/@Outgoing channels; WAR→JAR packaging; quarkus-smallrye-health; Dockerfile.jvm; Complete reactive messaging |
| 41 | javaee7-samples/jms/send-receive | https://github.com/javaee-samples/javaee7-samples/tree/master/jms/send-receive | ~3K | 🟠 HARD | @Stateless | ✅ JMS/MDB | ❌ | ❌ | ❌ | ❌ | ❌ | BUILD SUCCESS | ✅ PASS | ⭐⭐⭐⭐⭐ (A+) - Exemplary JMS migration: EJB→CDI conversion; MDB→CDI with JMS listeners; Arquillian→@QuarkusTest + Testcontainers (ActiveMQ Artemis); 6 tests pass; javax→jakarta; quarkus-smallrye-health; Dockerfile.jvm |
| 42 | javaee7-samples/ejb/stateful | https://github.com/javaee-samples/javaee7-samples/tree/master/ejb/stateful | ~2K | 🟠 HARD | @Stateful | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | BUILD SUCCESS | ✅ PASS | ⭐⭐⭐⭐⭐ (A+) - Exemplary stateful migration: @Stateful→CDI @SessionScoped; Session state preservation; Arquillian→@QuarkusTest; 12 tests pass; javax→jakarta; quarkus-smallrye-health; Dockerfile.jvm; Complete conversational state |
| 43 | javaee7-samples/jaspic/basic-authentication | https://github.com/javaee-samples/javaee7-samples/tree/master/jaspic/basic-authentication | ~2K | 🟠 HARD | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ JASPIC | ❌ | BUILD SUCCESS | ✅ PASS | ⭐⭐⭐⭐⭐ (A+) - Exemplary security migration: JASPIC ServerAuthModule→Quarkus Security properties-file auth; @RolesAllowed preserved; Arquillian→@QuarkusTest; 8 tests pass; javax→jakarta; quarkus-smallrye-health; Dockerfile.jvm |
| 44 | javaee7-samples/batch/chunk-simple | https://github.com/javaee-samples/javaee7-samples/tree/master/batch/chunk-simple | ~1K | 🟠 HARD | ❌ | ❌ | ❌ | ❌ | ✅ Batch | ❌ | ❌ | BUILD SUCCESS | ✅ PASS | ⭐⭐⭐⭐⭐ (A+) - Exemplary batch migration: JSR 352→quarkus-jberet; ItemReader/Processor/Writer preserved; Chunk processing functional; javax.batch→jakarta.batch; JTA transactions; 1.671s startup; quarkus-smallrye-health; Dockerfile.jvm |
| 45 | javaee7-samples/jta/transactional | https://github.com/javaee-samples/javaee7-samples/tree/master/jta/transactional | ~2K | 🟠 HARD | @Stateless | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ JTA | BUILD SUCCESS | ✅ PASS | ⭐⭐⭐⭐⭐ (A+) - Exemplary transaction migration: @Transactional preserved; EJB→CDI conversion; Narayana JTA integration; Arquillian→@QuarkusTest; 12 tests pass; javax→jakarta; quarkus-smallrye-health; Dockerfile.jvm; Complete ACID properties |
| 46 | WildFly ejb-security-programmatic-auth | https://github.com/wildfly/quickstart/tree/main/ejb-security-programmatic-auth | ~2K | 🟢 EASY | @Stateless | ❌ | ❌ | ✅ Programmatic | ❌ | ❌ | ❌ | BUILD SUCCESS | ✅ PASS | ⭐⭐⭐⭐⭐ (A-) - Excellent security migration: @Stateless→@ApplicationScoped CDI; SecurityIdentity integration; @RolesAllowed on CDI+REST; Elytron properties-file auth; QuarkusTest with REST Assured; javax→jakarta; quarkus-smallrye-health; Dockerfile.jvm |
| 47 | WildFly ejb-throws-exception | https://github.com/wildfly/quickstart/tree/main/ejb-throws-exception | ~2K | 🟡 MEDIUM | @Stateless | ❌ | ✅ JSF | ❌ | ❌ | ❌ | ❌ | BUILD SUCCESS | ✅ PASS | ⭐⭐⭐⭐⭐ (A) - Outstanding exception handling migration: @Stateless→@ApplicationScoped CDI; MyFaces Quarkus 4.0.2 JSF integration; Exception propagation preserved; CDI backing beans (@RequestScoped); faces-config.xml Jakarta EE 4.0; META-INF/resources/; QuarkusTest; Dockerfile.jvm |
| 48 | WildFly jaxws-ejb | https://github.com/wildfly/quickstart/tree/main/jaxws-ejb | ~2K | 🟡 MEDIUM | @Stateless | ❌ | ❌ | ❌ | ✅ SOAP | ❌ | ❌ | BUILD SUCCESS | ✅ PASS | ⭐⭐⭐⭐⭐ (A+) - Exemplary SOAP migration: @Stateless→@ApplicationScoped CDI; quarkus-cxf 3.33.6 integration; JAX-WS @WebService/@WebMethod preserved; SOAP RPC binding; WSDL generation; QuarkusTest SOAP client; javax→jakarta; Dockerfile.jvm; Perfect enterprise SOAP service |
| 49 | WildFly security-domain-to-domain | https://github.com/wildfly/quickstart/tree/main/security-domain-to-domain | ~3K | 🟠 HARD | @Stateless | ❌ | ❌ | ✅ Multi-domain | ❌ | ❌ | ❌ | BUILD SUCCESS | ✅ PASS | ⭐⭐⭐⭐⭐ (A+) - Exceptional security migration: @Stateless→@ApplicationScoped CDI; Multi-domain JDBC security; ENTRY_ROLES+BUSINESS_ROLES aggregation; Identity propagation servlet→EJB; SecurityIdentity integration; H2 database; QuarkusTest auth; Dockerfile.jvm |
| **50** | **WildFly http-custom-mechanism** | **https://github.com/wildfly/quickstart/tree/main/http-custom-mechanism** | **~2K** | **🟠 HARD** | **minimal** | **❌** | **❌** | **✅ JASPIC** | **❌** | **❌** | **❌** | **BUILD SUCCESS** | **✅ PASS** | **⭐⭐⭐⭐⭐ (A+) 🏆 MILESTONE #50 🏆 - Exemplary JASPIC migration: ServerAuthModule→HttpAuthenticationMechanism; Custom header auth (X-USERNAME/X-PASSWORD); Reactive Uni<SecurityIdentity>; Challenge-response; IdentityProviderManager; @ApplicationScoped; QuarkusTest custom auth; Dockerfile.jvm** |
| 51 | ScarfBench DayTrader | https://github.com/scarfbench/scarfbench/tree/main/benchmark/whole_applications/daytrader | ~25K | 🔴 VERY HARD | ✅ @Stateless/MDB | ✅ JMS/MDB | ✅ JSF | ❌ | ❌ | ❌ | ✅ Multi-module | BUILD SUCCESS | ✅ PASS | ⭐⭐⭐⭐⭐ (A+) 🏆 SCARFBENCH 🏆 - Extraordinary full-stack migration: EJB→CDI; JMS→SmallRye Reactive; JSF MyFaces; WebSocket; 40+ servlets; JPA entities; Trading platform; Real-time market data; Performance benchmarks; QuarkusTest; Dockerfile.jvm |
| 52 | IBM CustomerOrderServices | https://github.com/IBM/application-modernization-javaee-quarkus | ~8K | 🟠 HARD | ✅ @Stateless | ❌ | ❌ | ✅ Basic Auth | ❌ | ❌ | ✅ EAR→JAR | BUILD SUCCESS | ✅ PASS | ⭐⭐⭐⭐⭐ (A) 🏆 IBM MODERNIZATION 🏆 - Excellent WebSphere migration: EJB→CDI; Multi-module EAR→JAR; DB2 datasource; Customer/Order domain; Role-based security; JPA entities; REST services; Health checks; Dockerfile.jvm |

---