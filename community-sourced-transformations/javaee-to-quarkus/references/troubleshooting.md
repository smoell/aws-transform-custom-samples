# Troubleshooting Guide

## Symptom → Diagnostic → Fix Table

| Symptom | Diagnostic Command | Fix |
|---|---|---|
| Bean proxy errors | `grep -rn 'final class' src/` | Add `quarkus.arc.transform-unproxyable-classes=true` |
| Missing beans at runtime | Check build logs for "removed bean" | Set `quarkus.arc.remove-unused-beans=false` |
| JNDI lookup failures | `grep -rn 'InitialContext' src/` | Replace with `@Inject` or `@ConfigProperty` |
| JMS/REST blocking issues | Check REST endpoints calling JMS | Add `@Blocking` annotation |
| JSF navigation broken | `grep -rn 'jakarta\.face\.' src/` | Fix typos: `jakarta.face` → `jakarta.faces` |
| Security annotations ignored | Check for quarkus-security extension | Add `quarkus-elytron-security-properties-file` |
| Tests not found | Check test class naming | Rename `*Test.java` or configure surefire |
| Native build failures | Check reflection configuration | Add `@RegisterForReflection` or config |

## Common Quick-Fixes

### Bean Discovery Issues
```bash
# Check what beans were removed
mvn quarkus:info -Dquarkus.arc.unremovable-types=**

# Enable all beans during migration  
quarkus.arc.remove-unused-beans=false
```

### Proxy Creation Problems
```properties
# Transform final classes to be proxyable
quarkus.arc.transform-unproxyable-classes=true
```

### Database Schema Issues
```properties
# Recreate schema on startup (development only)
quarkus.hibernate-orm.database.generation=drop-and-create
```

### JMS Blocking Issues
```java
// Add @Blocking to REST endpoints that call JMS
@GET
@Blocking
public String processOrder() {
    jmsProducer.send("order-queue", orderData);
    return "processed";
}
```

### Security Configuration Missing
```properties
# Enable security extension
quarkus.security.users.embedded.enabled=true
quarkus.security.users.embedded.users.admin=password
quarkus.security.users.embedded.roles.admin=admin
```

## Diagnostic Commands

### Build Analysis
```bash
# Check extension compatibility
mvn quarkus:list-extensions | grep -i <extension-name>

# Validate configuration
mvn quarkus:info

# Check for unused dependencies
mvn dependency:analyze
```

### Runtime Debugging
```bash
# Enable CDI debugging
-Dquarkus.log.category."io.quarkus.arc".level=DEBUG

# Check health endpoints
curl http://localhost:8080/q/health

# View configuration
curl http://localhost:8080/q/info
```

### Migration Validation
```bash
# Scan for legacy patterns
grep -rn '@Stateless\|@EJB' src/main/java/
grep -rn 'InitialContext\|context\.lookup' src/
find . -name 'persistence.xml' -o -name 'web.xml'
```