# Application Properties Checklist

## MicroProfile Telemetry → Quarkus OpenTelemetry

- Remove WildFly MicroProfile OpenTelemetry subsystem config remnants
- All `otel.*` properties must be prefixed with `quarkus.` → `quarkus.otel.*`
- `@WithSpan`, `@SpanAttribute` annotations work unchanged (require `quarkus-opentelemetry` extension)
- Phase 5 validation: `grep -rn '^otel\.' src/main/resources/application.properties` must return empty
- Test profile: add `%test.quarkus.otel.enabled=false` to prevent OpenTelemetry noise in test output

## web.xml env-entry Migration

When migrating `<env-entry>` elements from `web.xml`, do NOT use the JNDI path as the property key.

### Procedure

1. **Identify env-entries** in web.xml
2. **Grep source for actual @ConfigProperty key names**: `grep -rn '@ConfigProperty' src/main/java/`
3. **Map env-entry-value to the correct key** in application.properties
4. **If no @ConfigProperty exists yet** (code used `@Resource(lookup=...)`): choose a meaningful key name following the `app.` prefix convention, then update the Java code.

## HTTP Root Path and Health Endpoints

When migrating context-root from ANY app-server descriptor:

```properties
# Sets application root path (equivalent to context-root) — ONLY if non-root
quarkus.http.root-path=/myapp
# ALWAYS pair with non-application-root-path
quarkus.http.non-application-root-path=/q
```

**Guard**: If original context root is `/` (root), do NOT set `quarkus.http.root-path` (default is already root).

**Impact on health endpoints**: Setting `quarkus.http.root-path` shifts ALL paths. Health moves to `/myapp/q/health/live`. Pairing with `quarkus.http.non-application-root-path=/q` keeps health at `/q/health`.

**K8s probe update**: Probes remain `httpGet.path: /q/health/live` when `non-application-root-path=/q` is set.

## REST Path Configuration (Application Class Removal)

When removing a JAX-RS `Application` subclass with `@ApplicationPath`, the base path must be preserved. The correct property depends on which REST extension is in pom.xml:

| REST Extension in pom.xml | Property Key | Example |
|---|---|---|
| `quarkus-rest` (RESTEasy Reactive) | `quarkus.rest.path` | `quarkus.rest.path=/api` |
| `quarkus-resteasy` (RESTEasy Classic) | `quarkus.resteasy.path` | `quarkus.resteasy.path=/api` |

**WARNING**: Using the wrong property key silently has no effect — the base path reverts to `/` without any error message.

**@ApplicationPath("/") guard**: If original `@ApplicationPath` is `"/"` or empty string → do NOT write any rest.path property. The default is already root.

## Required Properties by Extension

### Datasource (quarkus-jdbc-*)
```properties
quarkus.datasource.db-kind=postgresql
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/mydb
quarkus.datasource.username=user
quarkus.datasource.password=pass
quarkus.hibernate-orm.schema-management.strategy=drop-and-create
# (deprecated alias: quarkus.hibernate-orm.database.generation — still functional but emits startup WARNING since Quarkus 3.23)
quarkus.hibernate-orm.log.sql=false
```

**NEVER set `quarkus.hibernate-orm.dialect` when `db-kind` is configured** — Hibernate ORM 6 auto-detects dialect. An explicit dialect causes version-mismatch validation errors.

### Multi-Datasource Named Configuration

```properties
# Default datasource
quarkus.datasource.db-kind=postgresql
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/orders
quarkus.datasource.username=order_user
quarkus.datasource.password=order_pass

# Named datasource: "inventory"
quarkus.datasource."inventory".db-kind=postgresql
quarkus.datasource."inventory".jdbc.url=jdbc:postgresql://localhost:5432/inventory
quarkus.datasource."inventory".username=inv_user
quarkus.datasource."inventory".password=inv_pass
quarkus.hibernate-orm."inventory".datasource=inventory
quarkus.hibernate-orm."inventory".packages=com.example.inventory.model
```

### Transaction Manager (quarkus-narayana-jta)
```properties
quarkus.transaction-manager.enable-recovery=false
quarkus.transaction-manager.default-transaction-timeout=60s
```

### SmallRye Reactive Messaging (JMS → Messaging)

**ATOMIC CREATION RULE**: Generate the entire `application.properties` file in one write operation. Incremental "append later" patterns cause silent configuration inconsistencies that surface as test failures.

```properties
mp.messaging.outgoing.orders-out.connector=smallrye-in-memory
mp.messaging.incoming.orders-in.connector=smallrye-in-memory

# Test profile — in-memory connector (MANDATORY for tests without broker)
%test.mp.messaging.incoming.orders-in.connector=smallrye-in-memory
%test.mp.messaging.outgoing.orders-out.connector=smallrye-in-memory
```

### Bidirectional SmallRye Channel Configuration

When a channel name is used by BOTH a producer and consumer, BOTH directions MUST be configured. Missing one side causes silent channel resolution failure at startup.

**Exception**: In-memory connectors used only for intra-app communication (test scenarios where producer and consumer are in the same app) do not require both directions to have external connector config.

**Detection command**:
```bash
grep -rn '@Incoming\|@Outgoing\|@Channel' src/main/java/ | grep -oP '"[^"]+"' | sort -u
```

### OpenTelemetry (quarkus-opentelemetry)
```properties
quarkus.otel.enabled=true
quarkus.otel.exporter.otlp.traces.endpoint=http://localhost:4317
quarkus.otel.service.name=my-service
%test.quarkus.otel.enabled=false
```

### CXF SOAP (quarkiverse-cxf)
```properties
quarkus.cxf.endpoint."/hello".implementor=com.example.HelloServiceImpl
quarkus.cxf.endpoint."/hello".wsdl-path=META-INF/wsdl/hello.wsdl
```

## Security Extensions

### Basic Authentication (quarkus-elytron-security-properties-file)
```properties
quarkus.http.auth.basic=true
quarkus.security.users.embedded.enabled=true
quarkus.security.users.embedded.plain-text=true
quarkus.security.users.embedded.users.alice=password
quarkus.security.users.embedded.roles.alice=admin,user
```

## CDI/ArC Configuration

```properties
# During migration - prevent bean removal
quarkus.arc.remove-unused-beans=false
quarkus.arc.transform-unproxyable-classes=true
```

## Health & Monitoring

`quarkus-smallrye-health` is self-activating — no enable/disable property exists. Simply adding the extension to pom.xml activates `/q/health`, `/q/health/live`, and `/q/health/ready` endpoints.

```properties
# Allow unauthenticated access to health endpoints
quarkus.http.auth.permission.health.paths=/q/*
quarkus.http.auth.permission.health.policy=permit
```

## Common Migration Mappings

### persistence.xml → application.properties
```xml
<!-- Before: META-INF/persistence.xml -->
<persistence-unit name="primary" transaction-type="JTA">
    <jta-data-source>java:jboss/datasources/MyAppDS</jta-data-source>
</persistence-unit>
```

```properties
# After: application.properties
quarkus.datasource.db-kind=postgresql
quarkus.hibernate-orm.schema-management.strategy=none
quarkus.hibernate-orm.log.sql=true
```

## REST Client configKey Alignment

When using `@RegisterRestClient(configKey = "my-service")`, the `application.properties` key uses the **configKey** value, not the interface FQCN:

```properties
# Correct — uses configKey
quarkus.rest-client.my-service.url=http://host:8080
quarkus.rest-client.my-service.scope=jakarta.inject.Singleton

# WRONG — uses interface class name (only works without configKey)
# quarkus.rest-client."com.example.MyClient".url=http://host:8080
```

## microprofile-config.properties Migration

If the source project has `src/main/resources/META-INF/microprofile-config.properties`, merge its content into `application.properties`. Quarkus supports both files but `application.properties` takes precedence — maintaining both causes confusion.

## H2 Test Datasource Override

When the MAIN `application.properties` has an explicit `quarkus.hibernate-orm.dialect`, the test override MUST set BOTH `db-kind=h2` AND `dialect=H2Dialect`. Changing `db-kind` alone does NOT cancel an explicit dialect setting.

```properties
# src/test/resources/application.properties
%test.quarkus.datasource.db-kind=h2
%test.quarkus.datasource.jdbc.url=jdbc:h2:mem:test;DB_CLOSE_DELAY=-1
%test.quarkus.hibernate-orm.dialect=org.hibernate.dialect.H2Dialect
%test.quarkus.hibernate-orm.schema-management.strategy=drop-and-create
```

**Property key**: Use `quarkus.hibernate-orm.schema-management.strategy` (not the deprecated `database.generation`).

**Also add to pom.xml:**
```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-jdbc-h2</artifactId>
    <scope>test</scope>
</dependency>
```

**When override is NOT needed**: If main config omits `quarkus.hibernate-orm.dialect` (relying on auto-detection from db-kind), the test override only needs `%test.quarkus.datasource.db-kind=h2`.

## Payara JDBC Pool Attribute Mapping

| Payara Attribute | Quarkus Property | Notes |
|---|---|---|
| `steady-pool-size` | `quarkus.datasource.jdbc.min-size` | Minimum pool connections |
| `max-pool-size` | `quarkus.datasource.jdbc.max-size` | Maximum pool connections |
| `idle-timeout-in-seconds` | `quarkus.datasource.jdbc.idle-removal-interval` | Append `S` for seconds |
| `max-wait-time-in-millis` | `quarkus.datasource.jdbc.acquisition-timeout` | Convert to seconds with `S` suffix |
| `pool-resize-quantity` | (no equivalent) | Agroal manages pool growth automatically |

## JMS Extension Clarification

| Migration Path | Extension | Use When |
|---|---|---|
| Reactive migration (recommended) | `quarkus-messaging-amqp` or `-kafka` | MDB → @Incoming/@Outgoing channels |
| JMS API preserved | `quarkus-artemis-jms` + `quarkus-artemis-embedded` (test) | Must keep `ConnectionFactory`/`JMSContext` API |

## Validation Commands

```bash
./mvnw quarkus:list-extensions
./mvnw quarkus:info
./mvnw quarkus:dev
```
