# Application Properties Checklist

## MicroProfile Telemetry → Quarkus OpenTelemetry

- Remove WildFly MicroProfile OpenTelemetry subsystem config remnants
- All `otel.*` properties must be prefixed with `quarkus.` → `quarkus.otel.*`
- `@WithSpan`, `@SpanAttribute` annotations work unchanged (require `quarkus-opentelemetry` extension)
- Phase 5 validation: `grep -rn '^otel\.' src/main/resources/application.properties` must return empty

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

### Transaction Manager (quarkus-narayana-jta)
```properties
# JTA Configuration
quarkus.transaction-manager.enable-recovery=false
quarkus.transaction-manager.default-transaction-timeout=60s

# Optional: Object Store
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