# Quarkus Extension Catalog Reference

> Reference for selecting appropriate Quarkus extensions during migration.
> All extensions below are compatible with Quarkus 3.x BOM (`io.quarkus.platform:quarkus-bom`).
> Extensions managed by the BOM do not need explicit `<version>` — only community/Quarkiverse extensions require version.

## Status Legend

- ✅ **Stable** — Platform extension, production-ready, fully supported
- 🟡 **Preview** — Platform extension but API may change; suitable for production with caution
- 🔴 **Community** — Quarkiverse or third-party; not part of core platform BOM (requires explicit version)

---

## Core / CDI

| JavaEE Spec | Quarkus Extension | Maven artifactId | Status | Notes |
|---|---|---|---|---|
| CDI (Contexts & Dependency Injection) | ArC (build-time CDI) | `quarkus-arc` | ✅ Stable | Included by default in all Quarkus apps; see `references/arc-limitations.md` |
| EJB (Enterprise JavaBeans) | No direct equivalent | — | — | Migrate to CDI beans; see `references/ejb-to-cdi-mapping.md` |

## REST / Web

| JavaEE Spec | Quarkus Extension | Maven artifactId | Status | Notes |
|---|---|---|---|---|
| JAX-RS (REST) | RESTEasy Reactive | `quarkus-rest` | ✅ Stable | **Recommended**. Reactive + blocking support. Formerly `quarkus-resteasy-reactive` |
| JAX-RS (REST, classic) | RESTEasy Classic | `quarkus-resteasy` | ✅ Stable | Legacy path; use `quarkus-rest` for new migrations |
| JAX-RS + JSON | RESTEasy Reactive Jackson | `quarkus-rest-jackson` | ✅ Stable | **Detection rule**: If ANY REST endpoint returns/consumes non-String/Response/primitive types → include this extension. Without it, 415 Unsupported Media Type or empty responses occur silently. |
| JAX-RS + JSON-B | RESTEasy Reactive JSON-B | `quarkus-rest-jsonb` | ✅ Stable | Alternative to Jackson |
| Servlet API | Undertow | `quarkus-undertow` | ✅ Stable | Only if Servlet API required; prefer JAX-RS |
| WebSocket (JSR 356) | WebSockets | `quarkus-websockets` | ✅ Stable | JSR 356 compatible |
| WebSocket (next-gen) | WebSockets Next | `quarkus-websockets-next` | ✅ Stable | Quarkus-native API; recommended for new code |
| SOAP / JAX-WS | Apache CXF | `quarkus-cxf` | 🔴 Community | `io.quarkiverse.cxf:quarkus-cxf`; requires explicit version |

## Persistence / JPA

| JavaEE Spec | Quarkus Extension | Maven artifactId | Status | Notes |
|---|---|---|---|---|
| JPA (Hibernate ORM) | Hibernate ORM | `quarkus-hibernate-orm` | ✅ Stable | Hibernate 6.x bundled |
| JPA + Active Record | Hibernate ORM Panache | `quarkus-hibernate-orm-panache` | ✅ Stable | Optional; simplifies CRUD |
| JPA + Kotlin | Hibernate ORM Panache Kotlin | `quarkus-hibernate-orm-panache-kotlin` | ✅ Stable | Kotlin coroutine support |
| JDBC — PostgreSQL | JDBC PostgreSQL | `quarkus-jdbc-postgresql` | ✅ Stable | |
| JDBC — MySQL | JDBC MySQL | `quarkus-jdbc-mysql` | ✅ Stable | |
| JDBC — MariaDB | JDBC MariaDB | `quarkus-jdbc-mariadb` | ✅ Stable | |
| JDBC — H2 | JDBC H2 | `quarkus-jdbc-h2` | ✅ Stable | Dev/test only recommended |
| JDBC — Oracle | JDBC Oracle | `quarkus-jdbc-oracle` | ✅ Stable | |
| JDBC — SQL Server | JDBC MSSQL | `quarkus-jdbc-mssql` | ✅ Stable | |
| JDBC — DB2 | JDBC DB2 | `quarkus-jdbc-db2` | ✅ Stable | |
| JDBC — Derby | JDBC Derby | `quarkus-jdbc-derby` | ✅ Stable | |
| Connection Pool | Agroal | `quarkus-agroal` | ✅ Stable | Included transitively by JDBC extensions |
| Hibernate Envers (Audit) | Hibernate Envers | `quarkus-hibernate-envers` | ✅ Stable | `@Audited` works as-is |
| Hibernate Search | Hibernate Search + Elasticsearch | `quarkus-hibernate-search-orm-elasticsearch` | ✅ Stable | |
| Flyway (schema migration) | Flyway | `quarkus-flyway` | ✅ Stable | |
| Liquibase (schema migration) | Liquibase | `quarkus-liquibase` | ✅ Stable | |

## Validation

| JavaEE Spec | Quarkus Extension | Maven artifactId | Status | Notes |
|---|---|---|---|---|
| Bean Validation (JSR 380) | Hibernate Validator | `quarkus-hibernate-validator` | ✅ Stable | Jakarta Validation 3.0. **Must be added explicitly** — NOT a transitive dependency of `quarkus-hibernate-orm`. Trigger: `@NotNull`, `@Size`, `@Email`, `@Positive`, `@Min`, `@Max`, `@Pattern`, `@Valid` in source. |

## Messaging

| JavaEE Spec | Quarkus Extension | Maven artifactId | Status | Notes |
|---|---|---|---|---|
| JMS (keep API) | Artemis JMS | `quarkus-artemis-jms` | 🔴 Community | `io.quarkiverse.artemis:quarkus-artemis-jms`; requires version |
| JMS → AMQP (reactive) | SmallRye Reactive Messaging AMQP | `quarkus-messaging-amqp` | ✅ Stable | MicroProfile Reactive Messaging with AMQP 1.0 |
| JMS → Kafka (reactive) | SmallRye Reactive Messaging Kafka | `quarkus-messaging-kafka` | ✅ Stable | MicroProfile Reactive Messaging with Kafka |
| JMS → RabbitMQ | SmallRye Reactive Messaging RabbitMQ | `quarkus-messaging-rabbitmq` | ✅ Stable | Native RabbitMQ protocol |

## Security

| JavaEE Spec | Quarkus Extension | Maven artifactId | Status | Notes |
|---|---|---|---|---|
| Java EE Security (core) | Quarkus Security | `quarkus-security` | ✅ Stable | Core security framework |
| JDBC Realm | Elytron Security JDBC | `quarkus-elytron-security-jdbc` | ✅ Stable | Database-backed identity store |
| LDAP | Elytron Security LDAP | `quarkus-elytron-security-ldap` | ✅ Stable | LDAP identity store |
| OIDC / OAuth2 | OIDC | `quarkus-oidc` | ✅ Stable | Keycloak, Auth0, Okta, Cognito |
| JWT (MicroProfile) | SmallRye JWT | `quarkus-smallrye-jwt` | ✅ Stable | MicroProfile JWT RBAC |
| Properties file (dev) | Elytron Security Properties | `quarkus-elytron-security-properties-file` | ✅ Stable | Dev/test user store |

## JSON / Serialization

| JavaEE Spec | Quarkus Extension | Maven artifactId | Status | Notes |
|---|---|---|---|---|
| JSON-B (Jakarta JSON Binding) | JSON-B | `quarkus-jsonb` | ✅ Stable | Alternative to Jackson |
| JSON-P (Jakarta JSON Processing) | JSON-P | `quarkus-jsonp` | ✅ Stable | Low-level JSON API |
| Jackson (default JSON) | Jackson | `quarkus-jackson` | ✅ Stable | Default with `quarkus-rest` |
| JAXB (XML Binding) | JAXB | `quarkus-jaxb` | ✅ Stable | `jakarta.xml.bind` support |

## Mail / Communication

| JavaEE Spec | Quarkus Extension | Maven artifactId | Status | Notes |
|---|---|---|---|---|
| JavaMail (jakarta.mail) | Mailer | `quarkus-mailer` | ✅ Stable | Reactive + blocking; replaces JavaMail API |

## Scheduling / Batch

| JavaEE Spec | Quarkus Extension | Maven artifactId | Status | Notes |
|---|---|---|---|---|
| EJB Timer / @Schedule | Scheduler | `quarkus-scheduler` | ✅ Stable | `@Scheduled(cron=...)`, `@Scheduled(every="5m")` |
| JSR 352 Batch | JBeret | `quarkus-jberet` | 🔴 Community | `io.quarkiverse.jberet:quarkus-jberet`; preserves JSR 352 API |

## Caching

| JavaEE Spec | Quarkus Extension | Maven artifactId | Status | Notes |
|---|---|---|---|---|
| JCache (JSR 107) | Cache | `quarkus-cache` | ✅ Stable | Annotation-based caching (`@CacheResult`, `@CacheInvalidate`) |
| Infinispan | Infinispan Client | `quarkus-infinispan-client` | ✅ Stable | Distributed cache |
| Redis | Redis Client | `quarkus-redis-client` | ✅ Stable | Redis data store |

## Concurrency / Reactive

| JavaEE Spec | Quarkus Extension | Maven artifactId | Status | Notes |
|---|---|---|---|---|
| Concurrency Utilities (JSR 236) | Virtual Threads | `quarkus-virtual-threads` | ✅ Stable | Java 21+; `@RunOnVirtualThread` |
| Reactive Streams | Mutiny | `quarkus-mutiny` | ✅ Stable | Bundled; Uni/Multi reactive types |
| ManagedExecutor | SmallRye Context Propagation | `quarkus-smallrye-context-propagation` | ✅ Stable | MicroProfile Context Propagation |

## Configuration

| JavaEE Spec | Quarkus Extension | Maven artifactId | Status | Notes |
|---|---|---|---|---|
| MicroProfile Config | Built-in | (included in all Quarkus apps) | ✅ Stable | `@ConfigProperty`; no extra dep needed |
| Config from Vault | Vault | `quarkus-vault` | 🔴 Community | `io.quarkiverse.vault:quarkus-vault` |
| Config from AWS | AWS SSM/Secrets | `quarkus-amazon-ssm` | 🔴 Community | `io.quarkiverse.amazonservices:quarkus-amazon-ssm` |

## Observability

| JavaEE Spec | Quarkus Extension | Maven artifactId | Status | Notes |
|---|---|---|---|---|
| Health Check (MicroProfile) | SmallRye Health | `quarkus-smallrye-health` | ✅ Stable | `/q/health`, `/q/health/live`, `/q/health/ready` |
| Metrics (MicroProfile) | Micrometer | `quarkus-micrometer-registry-prometheus` | ✅ Stable | `/q/metrics`; Prometheus format |
| OpenTelemetry | OpenTelemetry | `quarkus-opentelemetry` | ✅ Stable | Distributed tracing |
| Logging (JBoss Logging) | Built-in | (included in all Quarkus apps) | ✅ Stable | JBoss Logging + JUL bridge |
| Logging (SLF4J) | Logging SLF4J | `quarkus-logging-slf4j` | ✅ Stable | Optional; only if SLF4J API preferred |

## Fault Tolerance

| JavaEE Spec | Quarkus Extension | Maven artifactId | Status | Notes |
|---|---|---|---|---|
| MicroProfile Fault Tolerance | SmallRye Fault Tolerance | `quarkus-smallrye-fault-tolerance` | ✅ Stable | `@Retry`, `@CircuitBreaker`, `@Timeout`, `@Fallback`, `@Bulkhead` |

## REST Client

| JavaEE Spec | Quarkus Extension | Maven artifactId | Status | Notes |
|---|---|---|---|---|
| JAX-RS Client | REST Client Reactive | `quarkus-rest-client-reactive` | ✅ Stable | MicroProfile REST Client; type-safe |
| JAX-RS Client (classic) | REST Client | `quarkus-rest-client` | ✅ Stable | Older API; prefer reactive variant |

## OpenAPI / Documentation

| JavaEE Spec | Quarkus Extension | Maven artifactId | Status | Notes |
|---|---|---|---|---|
| MicroProfile OpenAPI | SmallRye OpenAPI | `quarkus-smallrye-openapi` | ✅ Stable | Auto-generates OpenAPI spec; Swagger UI at `/q/swagger-ui` |

## Templating / UI

| JavaEE Spec | Quarkus Extension | Maven artifactId | Status | Notes |
|---|---|---|---|---|
| JSF (Facelets) → Qute | Qute | `quarkus-rest-qute` | ✅ Stable | Quarkus-native template engine |
| JSF → Qute Web | Qute Web | `quarkus-qute-web` | 🔴 Community | `io.quarkiverse.qute.web:quarkus-qute-web`; file-based routing for .html templates |
| JSF (preserve) | MyFaces | `myfaces-quarkus` | 🔴 Community | `org.apache.myfaces.core.extensions.quarkus:myfaces-quarkus`; limited native support |

## Containerization / Deployment

| JavaEE Spec | Quarkus Extension | Maven artifactId | Status | Notes |
|---|---|---|---|---|
| Docker image build | Container Image Docker | `quarkus-container-image-docker` | ✅ Stable | Builds Docker image during `mvn package` |
| Jib image build | Container Image Jib | `quarkus-container-image-jib` | ✅ Stable | Daemonless container build |
| Kubernetes manifests | Kubernetes | `quarkus-kubernetes` | ✅ Stable | Generates `kubernetes.yml` at build time |
| OpenShift manifests | OpenShift | `quarkus-openshift` | ✅ Stable | Generates OpenShift deployment config |
| AWS Lambda | AWS Lambda | `quarkus-amazon-lambda` | ✅ Stable | Serverless deployment |

## Testing

| JavaEE Spec | Quarkus Extension | Maven artifactId | Status | Notes |
|---|---|---|---|---|
| Arquillian → QuarkusTest | JUnit 5 | `quarkus-junit` | ✅ Stable | Core test framework (formerly quarkus-junit; Maven relocation in place) |
| REST testing | REST Assured | `rest-assured` (io.rest-assured) | ✅ Stable | BOM-managed; fluent HTTP testing |
| CDI mocking | JUnit 5 Mockito | `quarkus-junit-mockito` | ✅ Stable | `@InjectMock`, `@InjectSpy` (formerly quarkus-junit-mockito) |
| Test H2 database | Test H2 | `quarkus-test-h2` | ✅ Stable | H2 for integration tests |
| TestContainers | Test Containers | `quarkus-test-containers` | ✅ Stable | TestContainers lifecycle management |
| Code coverage | JaCoCo | `quarkus-jacoco` | ✅ Stable | Coverage reports |
| Security testing | Security Test | `quarkus-test-security` | ✅ Stable | `@TestSecurity` for role injection. Required when tests use `@TestSecurity(user="...", roles="...")` — not transitive from quarkus-security. |

## gRPC

| JavaEE Spec | Quarkus Extension | Maven artifactId | Status | Notes |
|---|---|---|---|---|
| EJB Remote → gRPC | gRPC | `quarkus-grpc` | ✅ Stable | Alternative to EJB Remote interfaces |

---

## Minimum Version Notes

| Extension | Minimum Quarkus Version | Notes |
|---|---|---|
| `quarkus-rest` (replaces `quarkus-resteasy-reactive`) | 3.9+ | Renamed in 3.9; older versions use `quarkus-resteasy-reactive` |
| `quarkus-websockets-next` | 3.2+ | Available since 3.2 |
| `quarkus-virtual-threads` | 3.6+ | Requires Java 21+ runtime |
| `quarkus-messaging-amqp` (replaces `quarkus-smallrye-reactive-messaging-amqp`) | 3.9+ | Shortened artifactId in 3.9 |
| `quarkus-messaging-kafka` (replaces `quarkus-smallrye-reactive-messaging-kafka`) | 3.9+ | Shortened artifactId in 3.9 |
| `quarkus-rest-client-reactive` (replaces `quarkus-rest-client-reactive-jackson`) | 3.9+ | Consolidated in 3.9 |

---

## Community Extensions (Quarkiverse) — Require Explicit Version

These are NOT managed by the Quarkus platform BOM — add explicit `<version>`:

```xml
<!-- Apache CXF (SOAP) -->
<dependency>
    <groupId>io.quarkiverse.cxf</groupId>
    <artifactId>quarkus-cxf</artifactId>
    <version>3.17.4</version>
</dependency>

<!-- Artemis JMS -->
<dependency>
    <groupId>io.quarkiverse.artemis</groupId>
    <artifactId>quarkus-artemis-jms</artifactId>
    <version>3.4.0</version>
</dependency>

<!-- JBeret (JSR 352 Batch) -->
<dependency>
    <groupId>io.quarkiverse.jberet</groupId>
    <artifactId>quarkus-jberet</artifactId>
    <version>2.4.0</version>
</dependency>

<!-- MyFaces (JSF) -->
<dependency>
    <groupId>org.apache.myfaces.core.extensions.quarkus</groupId>
    <artifactId>myfaces-quarkus</artifactId>
    <version>4.0.2</version>
</dependency>

<!-- Vault (secrets) -->
<dependency>
    <groupId>io.quarkiverse.vault</groupId>
    <artifactId>quarkus-vault</artifactId>
    <version>4.0.0</version>
</dependency>
```

**Note**: Always check [Quarkiverse Hub](https://quarkiverse.io/) or Maven Central for the latest compatible version with your Quarkus BOM version.
