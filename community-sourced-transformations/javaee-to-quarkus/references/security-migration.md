# Security Migration Reference

## SECURITY-NOTES.md Specification

### Purpose
Document pre-existing security vulnerabilities detected during JavaEE→Quarkus migration (NOT introduced by the migration itself).

### Trigger Conditions
Generate SECURITY-NOTES.md when ANY pattern is detected during Phase 2/3 code scanning:
- **Path Traversal**: `../`, `..\\`, URL path manipulation patterns
- **Hardcoded Credentials**: passwords, API keys, tokens in source code
- **Deprecated Crypto**: MD5, SHA1, DES, RC4, weak SSL/TLS configurations
- **Insecure Deserialization**: ObjectInputStream, readObject() without validation

### File Format
```markdown
# Security Notes

| Severity | Location | Issue | Recommendation |
|----------|----------|-------|----------------|
| HIGH | src/main/java/com/example/FileUtil.java:23 | Path traversal | Validate file paths |
| MEDIUM | src/main/java/com/example/Config.java:15 | Hardcoded password | Use @ConfigProperty |
| LOW | src/main/java/com/example/HashUtil.java:8 | Deprecated crypto MD5 | Use SHA-256 |
```

## JavaEE Security → Quarkus Security Migration

### Annotation Mapping
| JavaEE Annotation | Quarkus Equivalent | Action |
|---|---|---|
| `@RolesAllowed` | `@RolesAllowed` | NO CHANGE |
| `@DenyAll` | `@DenyAll` | NO CHANGE |
| `@PermitAll` | `@PermitAll` | NO CHANGE |
| `@RunAs("role")` | Not supported | Remove, use programmatic security |

### Configuration Migration
```properties
# Replace web.xml security-constraint
quarkus.security.users.embedded.enabled=true
quarkus.security.users.embedded.users.alice=alice
quarkus.security.users.embedded.roles.alice=admin,user
```

### JASPIC Migration Pattern
```java
// Before: ServerAuthModule
public class CustomAuthModule implements ServerAuthModule {
    // Complex JASPIC implementation
}

// After: Custom SecurityIdentityAugmentor  
@ApplicationScoped
public class CustomSecurityAugmentor implements SecurityIdentityAugmentor {
    @Override
    public Uni<SecurityIdentity> augment(SecurityIdentity identity, AuthenticationRequestContext context) {
        return Uni.createFrom().item(identity);
    }
}
```

### Security Context Migration
```java
// Before: EJB SessionContext
@Resource SessionContext sessionContext;
Principal p = sessionContext.getCallerPrincipal();

// After: Quarkus SecurityIdentity
@Inject SecurityIdentity securityIdentity;
String name = securityIdentity.getPrincipal().getName();
```