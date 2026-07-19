# Troubleshooting & Common Pitfalls

## Symptom → Diagnostic → Fix Table

| Symptom | Diagnostic | Fix |
|---|---|---|
| Bean proxy errors | `grep -rn 'final class' src/` | Add `quarkus.arc.transform-unproxyable-classes=true` |
| Missing beans at runtime | Check build logs for "removed bean" | Set `quarkus.arc.remove-unused-beans=false` |
| JNDI lookup failures | `grep -rn 'InitialContext' src/` | Replace with `@Inject` or `@ConfigProperty` |
| JMS/REST blocking issues | Check REST endpoints calling JMS | Add `@Blocking` annotation |
| JSF navigation broken | `grep -rn 'jakarta\.face\.' src/` | Fix typos: `jakarta.face` → `jakarta.faces` |
| Security annotations ignored | Missing quarkus-security extension | Add `quarkus-elytron-security-properties-file` |
| Tests not found | Check test class naming | Rename `*Test.java` or configure surefire |
| Native build failures | Missing reflection config | Add `@RegisterForReflection` |
| import.sql table errors | Table name mismatch | Set `quarkus.hibernate-orm.physical-naming-strategy=org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl` |
| @ConversationScoped failure | ArC doesn't support it | Use `@SessionScoped` + `implements Serializable` |
| UnnecessaryStubbingException | MockitoExtension strict mode | Remove unreachable stubs — see arquillian-to-quarkustest.md Section 9 |
| TransactionRequiredException on read | @Transactional missing | Apply class-level @Transactional (not just write methods) |
| Uni<T> NPE in tests | Mockito default returns null for Uni | Stub with `Uni.createFrom().item(...)` — see below |
| quarkus-junit5 relocation warning | Quarkus 3.31+ renamed artifact | Use `quarkus-junit` to suppress warning — see below |
| @ConfigProperty null in unit test | No CDI container in plain unit test | Use reflective injection or convert to @QuarkusTest — see arquillian-to-quarkustest.md Section 11 |
| Gradle + mvn BUILD COMMAND mismatch | `ls pom.xml` absent + `ls build.gradle` present | Do NOT create pom.xml — verify via `./gradlew build` or static inspection. See below |
| `find` exit code false positive | `find ... -delete` always exits 0 | Use `find ... -print` first, then `find ... -delete`; verify with re-run of print |

## JNDI Lookup Resolution Patterns

### Pattern 1: Static Config Lookups → @ConfigProperty

For JNDI lookups that retrieve configuration values (datasource URLs, hostnames, feature flags):

```java
// BEFORE
@Resource(lookup = "java:comp/env/config/emailHost")
private String emailHost;

// or programmatic:
InitialContext ctx = new InitialContext();
String emailHost = (String) ctx.lookup("java:comp/env/config/emailHost");

// AFTER
@ConfigProperty(name = "app.email.host")
String emailHost;
```

**Key**: the JNDI path (e.g. `config/emailHost`) does NOT equal the `@ConfigProperty` key. Grep source for `@ConfigProperty(name=...)` usages to discover actual key names used in code, then map the JNDI values to those keys in `application.properties`.

### Pattern 2: Unmigratable Dynamic JNDI Lookups → UnsupportedOperationException

For JNDI lookup methods that perform dynamic lookups (variable lookup paths, complex service resolution) where static replacement is not feasible:

```java
// BEFORE
public Object lookupService(String jndiName) {
    InitialContext ctx = new InitialContext();
    return ctx.lookup(jndiName);
}

// AFTER — replace body with UnsupportedOperationException
public Object lookupService(String jndiName) {
    throw new UnsupportedOperationException(
        "JNDI lookups not supported in Quarkus. Use @ConfigProperty or @Inject instead. "
        + "Attempted lookup: " + jndiName);
}
```

**Why UnsupportedOperationException**: it extends `RuntimeException`, which preserves existing `@Test(expected = RuntimeException.class)` test contracts. Tests that previously caught exceptions from JNDI failures continue to pass. The method signature (including checked exceptions) remains unchanged.

**When to use**: Only when the JNDI lookup cannot be replaced with a static `@ConfigProperty` or `@Inject`. This is a compile-and-test-passing placeholder that clearly communicates the limitation.

## Uni<T> Null Mock Pitfall

**Problem**: Mockito's default return value for `Uni<T>` is `null`. Calling `.subscribe().with()` or `.await().indefinitely()` on `null` throws `NullPointerException`.

**Symptom**: NPE in test code at a line like `service.someMethod().await().indefinitely()` where `someMethod()` returns `Uni<T>` and is mocked but not stubbed.

**Fix**: Always stub `Uni`-returning methods explicitly:
```java
// For Uni<SomeObject>
when(service.findItem(any())).thenReturn(Uni.createFrom().item(new Item("test")));

// For Uni<Void>
when(service.deleteItem(any())).thenReturn(Uni.createFrom().voidItem());

// For Uni that should fail
when(service.findItem(any())).thenReturn(Uni.createFrom().failure(new NotFoundException()));
```

**When this occurs**: After converting `@Asynchronous` EJB methods from `Future<T>` to `Uni<T>`, any test that mocks those methods must be updated to stub the Uni return value.

## quarkus-junit5 Artifact Relocation (Quarkus 3.31+)

In Quarkus 3.31+, `quarkus-junit5` was relocated to `quarkus-junit`. Both artifact IDs work, but using `quarkus-junit5` produces a harmless Maven relocation warning in the build output:

```
[WARNING] The artifact io.quarkus:quarkus-junit5 has been relocated to io.quarkus:quarkus-junit
```

**Fix**: For new migrations targeting Quarkus 3.31+, use `quarkus-junit` directly. For existing projects, this warning is cosmetic and does not affect functionality.

## Test Preservation — Never Delete Surviving Tests
Rule: a test may only be deleted if its **target class no longer exists** after migration.
If a test mocks a removed JCA/EJB interface but the implementation survives as a CDI bean,
**rewrite the mock against the CDI type** — do not delete the test.
- Check: `grep -rc '@Test' src/test` before vs. after — any drop must map to a removed target class.

## Incomplete Namespace Migration — javax-pinned Binary Dependency
Symptom: after migration, `grep -rc 'javax\.' src/` still returns many hits because a binary dependency with no `jakarta.*` release forces `javax.*`.
Fix priority — (b) is MANDATORY before halting; a javax-pinned binary is NOT sufficient reason to halt:

**(a) Replacement** — check for a jakarta-based release or newer major version of the library first.

**(b) Eclipse Transformer (resolves the majority of cases)** — rewrite
`javax.*`->`jakarta.*` in the offending artifact at build time. Maven plugin approach:
```xml
<plugin>
  <groupId>org.eclipse.transformer</groupId>
  <artifactId>transformer-maven-plugin</artifactId>
  <version>0.5.0</version>
  <executions>
    <execution>
      <id>jakarta-transform</id>
      <phase>generate-sources</phase>
      <goals><goal>run</goal></goals>
      <configuration>
        <rules><jakartaDefaults>true</jakartaDefaults></rules>
        <artifact>
          <groupId>org.rapla</groupId><artifactId>restinject</artifactId><version>PINNED</version>
        </artifact>
      </configuration>
    </execution>
  </executions>
</plugin>
```
Then depend on the transformed artifact (classifier `jakarta`) instead of the original. Verify with
`grep -rc 'javax\.' src/` (source) AND confirm the transformed jar no longer exposes `javax.*` on the
compile classpath. CLI alternative: `java -jar org.eclipse.transformer.cli.jar restinject.jar restinject-jakarta.jar`.

**(c) HALT** — ONLY if (a) has no jakarta release AND (b) fails (e.g. reflection-heavy artifact the
Transformer cannot rewrite). Then emit `BLOCKERS.md` documenting both attempts.

Never ship a partially-migrated namespace (mixed javax/jakarta) — but note: a clean Phase-0 halt on an
app that WAS migratable via the Transformer is also a failure. Exhaust (b) first.

## Multi-module EAR Test Consolidation
Tests in `ear/ejb-tests` submodules must be moved to the consolidated module's `src/test/java/` or they are silently dropped during EAR→single-module consolidation. Always verify test count post-migration matches pre-migration count.

## Common Migration Pitfalls

### Bean Management Issues
- **Bean removed by unused-bean optimization** → Set `quarkus.arc.remove-unused-beans=false` during migration
- **Final class cannot be proxied** → Add `quarkus.arc.transform-unproxyable-classes=true`
- **Missing no-arg constructor** → Normal-scoped beans need no-arg constructor for ArC proxies

### Threading Problems
- **@ApplicationScoped with mutable state** → Use `@RequestScoped` for per-request state
- **Self-invocation bypasses interceptors** → Extract to separate bean for `@Transactional`

### Migration Mistakes
- **Incomplete namespace migration** → Mixed `javax.*`/`jakarta.*` causes compilation errors
- **JNDI lookups remain** → `InitialContext` fails at Quarkus runtime
- **@ConversationScoped loss** → ArC fallback to `@SessionScoped` may cause state-leak across browser tabs
- **import.sql failures** → Verify table names match `@Table(name=...)` annotations
- **@Remove → @PreDestroy (WRONG)** → Simply remove the @Remove annotation; do not convert to @PreDestroy

### Quick Fixes
- **ArC bean discovery**: Use `-Dquarkus.arc.unremovable-types` to debug missing beans
- **Schema issues**: Set `quarkus.hibernate-orm.database.generation=drop-and-create` for dev
- **JMS blocking**: Always add `@Blocking` to REST endpoints that call JMS/messaging

## Gradle Project + Maven BUILD COMMAND Mismatch

**Symptom**: `mvn clean test` fails with "no POM" or "Could not find pom.xml" on a Gradle-only project.

**Diagnostic**:
```bash
ls pom.xml    # absent
ls build.gradle  # present (or build.gradle.kts)
```

**Resolution:**
1. **NEVER create a `pom.xml`** — this introduces a parallel build system that conflicts with the correct Gradle build and is always wrong.
2. Attempt `./gradlew build` or `./gradlew test` if a Gradle wrapper is available.
3. If Gradle wrapper is unavailable or version mismatch prevents execution, use **static verification**: grep scans for `javax.*` imports, EJB annotations, missing `application.properties`.
4. Report the environment mismatch in the output — do NOT treat as a migration failure.
5. **Debugger constraint**: the debugger role must also follow this constraint. Never create `pom.xml` to fix a Gradle project's `mvn` failure.

## find Exit-Code Idiom

**Problem**: `find ... -delete` always exits 0 even if no files match. This makes it unreliable for verifying file removal.

**Correct pattern:**
```bash
# Step 1: Preview what will be deleted
find src/main/webapp -name '*.xhtml' -print

# Step 2: Delete only if Step 1 showed results
find src/main/webapp -name '*.xhtml' -delete

# Step 3: Verify deletion succeeded
find src/main/webapp -name '*.xhtml' -print  # must return empty
```

Do NOT rely on the exit code of `find ... -delete` to determine if files were present or successfully removed.
