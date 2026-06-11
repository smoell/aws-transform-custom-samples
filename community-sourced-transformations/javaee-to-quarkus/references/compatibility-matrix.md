# Compatibility Matrix

## Core Platform Compatibility

| Component | Version | JDK Support | Notes |
|---|---|---|---|
| Quarkus | 3.33.x LTS | JDK 17/21 | Recommended LTS line |
| Maven | 3.8+ | Required | Build tool minimum |
| Hibernate ORM | 6.x | Auto-managed | Via quarkus-hibernate-orm |
| Jakarta EE | 10 | Namespace support | javax.* → jakarta.* |
| MyFaces | 4.0.2 | Via myfaces-quarkus | JSF support |
| CXF | 3.33.6 | Via quarkiverse-cxf | SOAP/JAX-WS |
| Artemis | 2.x | Via SmallRye Messaging | JMS replacement |

## Version-Sensitive Extensions

| Extension | Quarkus Version | Special Notes |
|---|---|---|
| quarkus-jberet | 3.36+ required | Use CDI fallback for 3.33.x |
| quarkiverse-cxf | 3.33.6 exact | SOAP/JAX-WS compatibility |
| myfaces-quarkus | 4.0.2 | JSF preservation |
| quarkus-kafka | Any 3.33.x | SmallRye Kafka integration |

## Database Compatibility

| Database | JDBC Driver | Quarkus Extension | Notes |
|---|---|---|---|
| PostgreSQL | 42.x | quarkus-jdbc-postgresql | Recommended |
| MySQL | 8.0.x | quarkus-jdbc-mysql | Full compatibility |
| H2 | 2.x | quarkus-jdbc-h2 | Development only |
| Oracle | 21c+ | quarkus-jdbc-oracle | Enterprise |
| DB2 | 11.5+ | quarkus-jdbc-db2 | IBM environments |

## Container Platform Support

| Platform | Base Image | JVM Mode | Native Mode |
|---|---|---|---|
| OpenShift | UBI8 OpenJDK 17 | ✅ | ✅ |
| Kubernetes | Eclipse Temurin | ✅ | ✅ |
| Docker | Official OpenJDK | ✅ | ✅ |
| Podman | Compatible images | ✅ | ✅ |

## Version Maintenance Schedule

**LTS Line:** Quarkus 3.33.x (current)
- **Support Period:** Through Q2 2025
- **Next LTS:** Quarkus 3.8.x (check roadmap)
- **Critical Updates:** Monthly patch releases

**Extension Updates:**
- **CXF:** Tied to Quarkus version (3.33.6 for 3.33.x)
- **MyFaces:** Independent versioning (4.0.2 stable)
- **JBeret:** Version-sensitive (requires 3.36+)

## Migration Path Compatibility

| Source Platform | Quarkus 3.33.x | Migration Effort |
|---|---|---|
| WildFly 26+ | ✅ Full | Low-Medium |
| JBoss EAP 7/8 | ✅ Full | Low-Medium |
| Payara 5/6 | ✅ Partial | Medium |
| GlassFish 6/7 | ✅ Partial | Medium |
| WebLogic 12c/14c | ⚠️ Limited | High |
| Liberty | ✅ Good | Low-Medium |

**Legend:**
- ✅ Full: Complete API compatibility
- ✅ Partial: Most APIs supported, some rework needed
- ⚠️ Limited: Significant rework required

## Version-Sensitive Extensions
| Extension | Tested Version | Notes |
|-----------|----------------|-------|
| myfaces-quarkus | 4.0.2 | For Quarkus 3.33.x LTS |
| quarkus-cxf | 3.33.6 | quarkiverse-cxf |
| quarkus-jberet | Check compat | Limited; see batch-jberet-fallback.md |
| quarkus-narayana-lra | Bundled in Quarkus BOM | |

## Version Maintenance
Centralize Quarkus version in one place. When LTS advances, update only the BOM version.
Versioned anchor: `<quarkus.platform.version>3.33.2</quarkus.platform.version>`