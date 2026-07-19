# Application Properties Checklist

## MicroProfile Telemetry → Quarkus OpenTelemetry

- Remove WildFly MicroProfile OpenTelemetry subsystem config remnants
- All `otel.*` properties must be prefixed with `quarkus.` → `quarkus.otel.*`
- `@WithSpan`, `@SpanAttribute` annotations work unchanged (require `quarkus-opentelemetry` extension)
- Phase 5 validation: `grep -rn '^otel\.' src/main/resources/application.properties` must return empty

## web.xml env-entry Migration

When migrating `<env-entry>` elements from `web.xml`, do NOT use the JNDI path as the property key.

### Procedure

1. **Identify env-entries** in web.xml:
```xml
<env-entry>
    <env-entry-name>config/emailHost</env-entry-name>
    <env-entry-type>java.lang.String</env-entry-type>
    <env-entry-value>smtp.example.com</env-entry-value>
</env-entry>
```

2. **Grep source for actual @ConfigProperty key names**:
```bash
grep -rn '@ConfigProperty' src/main/java/
```
This reveals the actual keys the code expects, e.g. `@ConfigProperty(name = "app.email.host")`.

3. **Map env-entry-value to the correct key** in application.properties:
```properties
# WRONG — JNDI path as key (will not be found by code)
config.emailHost=smtp.example.com

# CORRECT — matches @ConfigProperty(name=...) in source
app.email.host=smtp.example.com
```

4. **If no @ConfigProperty exists yet** (code used `@Resource(lookup=...)`): choose a meaningful key name following the `app.` prefix convention, then update the Java code to use `@ConfigProperty(name = "chosen.key")`.

## HTTP Root Path and Health Endpoints

When migrating `jboss-web.xml` context-root or `<context-root>` from `application.xml`:

```properties
# Sets application root path (equivalent to context-root)
quarkus.http.root-path=/myapp
```

**Impact on health endpoints**: Setting `quarkus.http.root-path` shifts ALL paths including non-application endpoints. Health moves to `/myapp/q/health/live` instead of `/q/health/live`.

**To keep health at `/q/health` regardless of root-path**:
```properties
quarkus.http.root-path=/myapp
quarkus.http.non-application-root-path=/q
```

**K8s probe update**: If probes were `httpGet.path: /q/health/live`, update to `/{root-path}/q/health/live` — or set `non-application-root-path` and keep probes unchanged.

## REST Path Configuration (Application Class Removal)

When removing a JAX-RS `Application` subclass with `@ApplicationPath`, the base path must be preserved in application.properties. The correct property depends on which REST extension is in pom.xml:

| REST Extension in pom.xml | Property Key | Example |
|---|---|---|
| `quarkus-rest` (RESTEasy Reactive) | `quarkus.rest.path` | `quarkus.rest.path=/api` |
| `quarkus-resteasy` (RESTEasy Classic) | `quarkus.resteasy.path` | `quarkus.resteasy.path=/api` |

**WARNING**: Using the wrong property key silently has no effect — the base path reverts to `/` without any error message. Always verify which REST extension is in pom.xml before setting the property.

**Detection**: Check pom.xml for `quarkus-rest` vs `quarkus-resteasy` (or their older names: `quarkus-resteasy-reactive` → `quarkus-rest`).

```java
// BEFORE — Application class with custom path
@ApplicationPath("/api")
public class JaxRsApplication extends Application {}

// AFTER — Remove class entirely, add to application.properties:
// quarkus.rest.path=/api       (if using quarkus-rest)
// quarkus.resteasy.path=/api   (if using quarkus-resteasy)
```

## Required Properties by Extension

### Datasource (quarkus-jdbc-*)
```properties
# Minimum required
quarkus.datasource.db-kind=postgresql
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/mydb
quarkus.datasource.username=user
quarkus.datasource.password=pass

# Hibernate ORM
quarkus.hibernate-orm.database.generation=drop-and-create
quarkus.hibernate-orm.log.sql=false
```

### Multi-Datasource Named Configuration

When persistence.xml has multiple `<persistence-unit>` elements or the app uses multiple JNDI datasources:

```properties
# Default datasource (unnamed — injected with plain @Inject EntityManager)
quarkus.datasource.db-kind=postgresql
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/orders
quarkus.datasource.username=order_user
quarkus.datasource.password=order_pass

# Named datasource: "inventory"
quarkus.datasource."inventory".db-kind=postgresql
quarkus.datasource."inventory".jdbc.url=jdbc:postgresql://localhost:5432/inventory
quarkus.datasource."inventory".username=inv_user
quarkus.datasource."inventory".password=inv_pass

# Map named persistence unit to named datasource
quarkus.hibernate-orm."inventory".datasource=inventory
quarkus.hibernate-orm."inventory".packages=com.example.inventory.model
```

Injection in code:
```java
@Inject EntityManager defaultEm;                        // default
@Inject @PersistenceUnit("inventory") EntityManager em; // named
```

### Transaction Manager (quarkus-narayana-jta)
```properties
# JTA Configuration
quarkus.transaction-manager.enable-recovery=false
quarkus.transaction-manager.default-transaction-timeout=60s

# Optional: Object Store (for crash recovery with BMT)
quarkus.transaction-manager.object-store.directory=tm-object-store
```

### Artemis JMS (quarkus-artemis-jms)
```properties
# Embedded Artemis
quarkus.artemis.enabled=true
quarkus.artemis.embedded.enabled=true
quarkus.artemis.embedded.queues=orders,notifications
quarkus.artemis.embedded.topics=market-updates

# External Artemis  
quarkus.artemis.url=tcp://localhost:61616
quarkus.artemis.username=artemis
quarkus.artemis.password=artemis
```

### SmallRye Reactive Messaging (JMS → Messaging)
```properties
# Channel → Queue mapping
mp.messaging.outgoing.orders-out.connector=smallrye-in-memory
mp.messaging.incoming.orders-in.connector=smallrye-in-memory

# For external Artemis
mp.messaging.outgoing.orders.connector=smallrye-artemis
mp.messaging.outgoing.orders.address=orders-queue
mp.messaging.outgoing.orders.host=localhost
mp.messaging.outgoing.orders.port=61616

# Test profile — in-memory connector (MANDATORY for tests without broker)
%test.mp.messaging.incoming.orders-in.connector=smallrye-in-memory
%test.mp.messaging.outgoing.orders-out.connector=smallrye-in-memory
# Add %test override for EVERY channel defined above
```

### Kafka (quarkus-kafka)
```properties
# Kafka basics
kafka.bootstrap.servers=localhost:9092

# Channel configuration
mp.messaging.outgoing.orders.connector=smallrye-kafka
mp.messaging.outgoing.orders.topic=orders-topic
mp.messaging.outgoing.orders.key.serializer=org.apache.kafka.common.serialization.StringSerializer
mp.messaging.outgoing.orders.value.serializer=org.apache.kafka.common.serialization.StringSerializer
```

### OpenTelemetry (quarkus-opentelemetry)
```properties
# Basic tracing
quarkus.otel.enabled=true
quarkus.otel.exporter.otlp.traces.endpoint=http://localhost:4317
quarkus.otel.service.name=my-service
quarkus.otel.exporter.otlp.traces.protocol=grpc

# Jaeger (alternative)
quarkus.jaeger.service-name=my-service
quarkus.jaeger.sampler-type=const
quarkus.jaeger.sampler-param=1
```

### CXF SOAP (quarkiverse-cxf)
```properties
# Endpoint configuration
quarkus.cxf.endpoint."/hello".implementor=com.example.HelloServiceImpl
quarkus.cxf.endpoint."/hello".wsdl-path=META-INF/wsdl/hello.wsdl

# Client configuration
quarkus.cxf.client.hello.client-endpoint-url=http://localhost:8080/soap/hello
quarkus.cxf.client.hello.wsdl-url=http://localhost:8080/soap/hello?wsdl
```

## Security Extensions

### Basic Authentication (quarkus-elytron-security-properties-file)
```properties
# Enable basic auth
quarkus.http.auth.basic=true

# Embedded users (development only)
quarkus.security.users.embedded.enabled=true
quarkus.security.users.embedded.plain-text=true
quarkus.security.users.embedded.users.alice=password
quarkus.security.users.embedded.roles.alice=admin,user

# File-based users (production)
quarkus.security.users.file.enabled=true
quarkus.security.users.file.users=users.properties
quarkus.security.users.file.roles=roles.properties
```

### JDBC Security (quarkus-elytron-security-jdbc)
```properties
# JDBC identity store
quarkus.security.jdbc.enabled=true
quarkus.security.jdbc.principal-query.sql=SELECT password FROM users WHERE username=?
quarkus.security.jdbc.principal-query.clear-password-mapper.enabled=true
quarkus.security.jdbc.principal-query.clear-password-mapper.password-index=1

# Role mapping
quarkus.security.jdbc.principal-query.roles.sql=SELECT role FROM user_roles WHERE username=?
quarkus.security.jdbc.principal-query.roles.attribute-mappings.0.index=1
quarkus.security.jdbc.principal-query.roles.attribute-mappings.0.to=groups
```

## CDI/ArC Configuration

### Essential CDI Settings
```properties
# During migration - prevent bean removal
quarkus.arc.remove-unused-beans=false

# Enable proxy transformation for final classes
quarkus.arc.transform-unproxyable-classes=true

# Debug CDI issues
quarkus.log.category."io.quarkus.arc".level=DEBUG
```

## Health & Monitoring

### SmallRye Health (quarkus-smallrye-health)
```properties
# Health checks (automatically enabled)
quarkus.health.extensions.enabled=true

# Custom health check paths
quarkus.http.auth.permission.health.paths=/q/*
quarkus.http.auth.permission.health.policy=permit
```

### Micrometer (quarkus-micrometer)
```properties
# Prometheus metrics
quarkus.micrometer.export.prometheus.enabled=true
quarkus.micrometer.export.prometheus.path=/q/metrics
```

## JTA Minimal Configuration

### Basic JTA Setup
```properties
# Enable JTA (usually automatic with JPA)
quarkus.transaction-manager.enable-recovery=false

# Transaction timeout
quarkus.transaction-manager.default-transaction-timeout=60s

# Database-specific settings
quarkus.datasource.jdbc.transactions=xa
```

## Common Migration Mappings

### persistence.xml → application.properties
```xml
<!-- Before: META-INF/persistence.xml -->
<persistence-unit name="customerPU">
    <jta-data-source>java:/datasources/CustomerDS</jta-data-source>
    <properties>
        <property name="hibernate.dialect" value="org.hibernate.dialect.PostgreSQLDialect"/>
        <property name="hibernate.show_sql" value="true"/>
    </properties>
</persistence-unit>
```

```properties
# After: application.properties
quarkus.datasource.db-kind=postgresql
quarkus.hibernate-orm.database.generation=none
quarkus.hibernate-orm.log.sql=true
quarkus.hibernate-orm.dialect=org.hibernate.dialect.PostgreSQLDialect
```

### web.xml → application.properties
```xml
<!-- Before: WEB-INF/web.xml -->
<context-param>
    <param-name>javax.faces.PROJECT_STAGE</param-name>
    <param-value>Production</param-value>
</context-param>
<context-param>
    <param-name>javax.faces.FACELETS_SKIP_COMMENTS</param-name>
    <param-value>true</param-value>
</context-param>
```

```properties
# After: application.properties
quarkus.faces.project-stage=Production
quarkus.faces.facelets-skip-comments=true
```

## Validation Commands

```bash
# Check property syntax
./mvnw quarkus:list-extensions

# Validate configuration
./mvnw quarkus:info

# Test with dev mode
./mvnw quarkus:dev
```
