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

JASPIC `ServerAuthModule` maps to Quarkus `HttpAuthenticationMechanism` (full class rewrite). This is a transitive dependency of `quarkus-rest` — no extra pom.xml entry needed.

```java
// BEFORE: ServerAuthModule (JASPIC)
public class CustomAuthModule implements ServerAuthModule {
    @Override
    public AuthStatus validateRequest(MessageInfo info, Subject clientSubject, Subject serviceSubject) {
        // Extract credentials from request, validate, populate subject
    }
    // ... initialize(), getSupportedMessageTypes(), etc.
}

// AFTER: HttpAuthenticationMechanism (Quarkus Security)
@ApplicationScoped
public class CustomAuthMechanism implements HttpAuthenticationMechanism {
    @Override
    public Uni<SecurityIdentity> authenticate(RoutingContext context, IdentityProviderManager identityProviderManager) {
        // Extract credentials from context.request(), call identityProviderManager.authenticate()
        String token = context.request().getHeader("Authorization");
        if (token == null) {
            return Uni.createFrom().nullItem(); // no credentials → skip this mechanism
        }
        return identityProviderManager.authenticate(new TokenAuthenticationRequest(token));
    }

    @Override
    public Uni<ChallengeData> getChallenge(RoutingContext context) {
        // Return 401 challenge (equivalent to SEND_CONTINUE in JASPIC)
        return Uni.createFrom().item(new ChallengeData(401, "WWW-Authenticate", "Bearer"));
    }

    @Override
    public Set<Class<? extends AuthenticationRequest>> getCredentialTypes() {
        return Set.of(TokenAuthenticationRequest.class);
    }
}
```

For simpler cases (augmenting an existing identity with extra roles/attributes):
```java
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