# Security Migration Reference

## SECURITY-NOTES.md Specification

### Purpose
Document pre-existing security vulnerabilities detected during JavaEE→Quarkus migration (NOT introduced by the migration itself).

### Trigger Conditions
Generate SECURITY-NOTES.md when ANY pattern is detected during Phase 2/3 code scanning:
- **Path Traversal**: `../`, `..\\`, URL path manipulation patterns
- **Hardcoded Credentials**: passwords, API keys, tokens in source code
- **Deprecated Crypto**: API-boundary patterns (see detection commands below)
- **Insecure Deserialization**: ObjectInputStream, readObject() without validation

### Detection Commands for Deprecated Crypto

Use API-boundary patterns to avoid false positives on common identifiers:
```bash
# Correct — matches cryptographic API usage only
grep -rn 'getInstance.*"DES"\|getInstance.*"MD5"\|getInstance.*"SHA-1"\|MessageDigest\.getInstance\|Cipher\.getInstance' src/

# WRONG — bare 'DES' or 'SHA1' matches field names like 'description', 'getDescription'
# Do NOT use: grep -rn 'DES\|SHA1' src/
```

### File Format
```markdown
# Security Notes

| Severity | Location | Issue | Recommendation |
|----------|----------|-------|----------------|
| HIGH | src/main/java/com/example/FileUtil.java:23 | Path traversal | Validate file paths |
| MEDIUM | src/main/java/com/example/Config.java:15 | Hardcoded password | Use @ConfigProperty |
| LOW | src/main/java/com/example/HashUtil.java:8 | Deprecated crypto MD5 | Use SHA-256 |
```

### NO_OP Case — SECURITY-NOTES.md Template

When Phase 0 exits NO_OP (project is already Quarkus), SECURITY-NOTES.md MUST still be generated. All four security scans MUST execute before emitting the CLEAN template. If no vulnerabilities are detected:

```markdown
# Security Notes

No pre-existing security vulnerabilities detected during migration analysis.

## Scan Summary
- Path traversal patterns: CLEAN
- Hardcoded credentials: CLEAN
- Deprecated cryptography: CLEAN
- Insecure deserialization: CLEAN
```

### Provenance Field for Migrated Credentials

When credentials are found in files that are deleted during migration (e.g., `*-ds.xml`, `persistence.xml`), the SECURITY-NOTES.md entry MUST include a provenance field showing origin:

```markdown
| MEDIUM | src/main/resources/application.properties:5 | Hardcoded password | Location: my-app-ds.xml (deleted during migration) → application.properties. Use @ConfigProperty + env variable |
```

## JavaEE Security → Quarkus Security Migration

### Annotation Mapping
| JavaEE Annotation | Quarkus Equivalent | Action |
|---|---|---|
| `@RolesAllowed` | `@RolesAllowed` | NO CHANGE (import changes javax→jakarta) |
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

### jboss-web.xml security-domain → Quarkus HTTP Auth

`<security-domain>` in jboss-web.xml configures the **HTTP authentication realm** — it is NOT the EJB security context propagation blocker (which only applies to jboss-ejb3.xml). Map it as follows:

```properties
# application.properties
quarkus.http.auth.basic=true
quarkus.security.users.embedded.enabled=true
quarkus.security.users.embedded.plain-text=true
quarkus.security.users.embedded.users.alice=password
quarkus.security.users.embedded.roles.alice=admin,user
```

This is a **non-blocker** migration — proceed normally.

### JASPIC Migration Pattern

JASPIC `ServerAuthModule` maps to Quarkus `HttpAuthenticationMechanism` (full class rewrite). This is a transitive dependency of `quarkus-rest` — no extra pom.xml entry needed.

**Mandatory abstract methods**: `authenticate()` and `getChallenge()`. `getCredentialTypes()` is optional.

```java
// AFTER: HttpAuthenticationMechanism (Quarkus Security)
@ApplicationScoped
public class CustomAuthMechanism implements HttpAuthenticationMechanism {
    @Override
    public Uni<SecurityIdentity> authenticate(RoutingContext context, IdentityProviderManager identityProviderManager) {
        String rawToken = context.request().getHeader("Authorization");
        if (rawToken == null) {
            return Uni.createFrom().nullItem();
        }
        TokenCredential credential = new TokenCredential(rawToken, "Bearer");
        return identityProviderManager.authenticate(new TokenAuthenticationRequest(credential));
    }

    @Override
    public Uni<ChallengeData> getChallenge(RoutingContext context) {
        return Uni.createFrom().item(new ChallengeData(401, "WWW-Authenticate", "Bearer"));
    }
}
```

**Key API notes**:
- Use `TokenCredential(token, "Bearer")` — NOT the non-existent `BearerTokenCredential` class.
- Use `new TokenAuthenticationRequest(credential)` with a `TokenCredential` argument.

For simpler cases (augmenting an existing identity):
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
