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
| Tests not found | Check test class naming | Rename `*IT.java` → `*Test.java` |
| Native build failures | Missing reflection config | Add `@RegisterForReflection` |
| import.sql table errors | Table name mismatch | Set physical-naming-strategy |
| @ConversationScoped failure | ArC doesn't support it | Use `@SessionScoped` + `implements Serializable` |
| UnnecessaryStubbingException | MockitoExtension strict mode | Remove unreachable stubs or use `lenient()` — see arquillian-to-quarkustest.md Section 9 |
| TransactionRequiredException on read | @Transactional missing | Apply class-level @Transactional (not just write methods) |
| Uni<T> NPE in tests | Mockito default returns null for Uni | Stub with `Uni.createFrom().item(...)` |
| quarkus-junit5 relocation warning | Quarkus 3.31+ renamed artifact | Use `quarkus-junit` (renamed from quarkus-junit5) |
| @ConfigProperty null in unit test | No CDI container in plain unit test | Use reflective injection or convert to @QuarkusTest |
| Gradle + mvn BUILD COMMAND mismatch | `ls pom.xml` absent + `ls build.gradle` present | Do NOT create pom.xml — use `./gradlew build` |
| `find` exit code false positive | `find ... -delete` always exits 0 | Use `find ... -print` first, verify, then delete |
| Missing `gradlew` script | `ls gradlew` absent | Run `gradle wrapper --gradle-version=8.10` — see below |
| Micrometer MeterRegistry NPE in tests | @Mock MeterRegistry → concrete methods unmocked | Use `new SimpleMeterRegistry()` |
| `find` with `-o` matches unexpected files | Operator precedence | Parenthesize: `\( -name '*.bak' -o -name '*.orig' \)` |
| Mockito 5 / JDK 17+ IllegalAccessError | Dynamic agent loading disabled | Add surefire argLine — see below |
| .bak files left after grep-edit | Worker edit left backup | Run full sweep — see below |
| 415 Unsupported Media Type | Missing JSON serialization extension | Add `quarkus-rest-jackson` — see below |
| REST base path silently wrong | Wrong property key for extension | Check extension: `quarkus-rest` → `quarkus.rest.path`; `quarkus-resteasy` → `quarkus.resteasy.path` |
| Hibernate dialect version-mismatch error | Explicit dialect set alongside db-kind | Remove `quarkus.hibernate-orm.dialect` — auto-detects from db-kind |

## 415 Unsupported Media Type — Missing quarkus-rest-jackson

**Problem**: REST endpoints returning complex types (non-String, non-Response, non-primitive) silently produce 415 or empty responses without a JSON serialization extension.

**Detection**: Any `@Path` endpoint with return type that is a POJO/DTO/List<T>/Map:
```bash
grep -rn '@GET\|@POST\|@PUT' src/main/java/ | head -20
# Check return types — if non-String/Response → need jackson
```

**Fix**: Add to pom.xml:
```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-rest-jackson</artifactId>
</dependency>
```

**LRA companion requirement**: `quarkus-narayana-lra` ALWAYS requires both `quarkus-rest-jackson` AND `quarkus-rest-client` as mandatory companions.

## quarkus.rest.path Silent Failure

**Problem**: Using `quarkus.rest.path` when the project uses `quarkus-resteasy` (Classic), or vice versa, silently has no effect — base path reverts to `/` without error.

**Diagnostic**: Check which extension is in pom.xml:
```bash
grep -n 'quarkus-rest\b\|quarkus-resteasy\b' pom.xml
```

**Rule**: `quarkus-rest` → `quarkus.rest.path`; `quarkus-resteasy` → `quarkus.resteasy.path`.

## Hibernate Dialect Auto-Detection

**Problem**: Setting explicit `quarkus.hibernate-orm.dialect` when `quarkus.datasource.db-kind` is already configured causes version-mismatch validation errors in Hibernate ORM 6.

**Fix**: Remove the `quarkus.hibernate-orm.dialect` line entirely. Hibernate ORM 6 auto-detects from `db-kind`. The only case where explicit dialect is needed: test override when main config sets an explicit dialect.

## Surefire argLine for Mockito 5 / JDK 17+

**Fix**: Add to pom.xml `maven-surefire-plugin` configuration:
```xml
<plugin>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <argLine>@{argLine} -XX:+EnableDynamicAgentLoading</argLine>
    </configuration>
</plugin>
```

## .bak/.orig File Cleanup

**Fix**: After any batch edit operation, proactively delete backup files:
```bash
# Preview what will be deleted
find . -not -path './target/*' -not -path './build/*' \( -name '*.bak' -o -name '*.orig' \) -print

# Delete
find . -not -path './target/*' -not -path './build/*' \( -name '*.bak' -o -name '*.orig' \) -delete

# Verify empty
find . -not -path './target/*' -not -path './build/*' \( -name '*.bak' -o -name '*.orig' \) -print
```

**Best practice**: Run this cleanup after EVERY batch of file modifications, not just at the end of migration. Scope includes root-level files (pom.xml.bak, build.gradle.bak).

## JNDI Lookup Resolution Patterns

### Pattern 1: Static Config Lookups → @ConfigProperty
```java
// BEFORE: @Resource(lookup = "java:comp/env/config/emailHost")
// AFTER: @ConfigProperty(name = "app.email.host") String emailHost;
```
**Key**: JNDI path ≠ @ConfigProperty key. Grep source for actual key names.

### Pattern 2: Managed Resource JNDI Lookups → @Inject CDI Bean
```java
// BEFORE: InitialContext ctx = new InitialContext(); DataSource ds = (DataSource) ctx.lookup(...)
// AFTER: @Inject DataSource ds;
```

### Pattern 3: Unmigratable Dynamic JNDI → UnsupportedOperationException
Only when static replacement is not feasible.

## Uni<T> Null Mock Pitfall

**Fix**: Always stub `Uni`-returning methods explicitly:
```java
when(service.findItem(any())).thenReturn(Uni.createFrom().item(new Item("test")));
when(service.deleteItem(any())).thenReturn(Uni.createFrom().voidItem());
```

## quarkus-junit Artifact (Quarkus 3.31+)

In Quarkus 3.31+, `quarkus-junit5` was renamed to `quarkus-junit` (Maven relocation in place). Use `quarkus-junit` directly to suppress the relocation warning. Source: [Quarkus 3.31 release blog](https://quarkus.io/blog/quarkus-3-31-released/).

## Test Preservation — Never Delete Surviving Tests
Rule: a test may only be deleted if its **target class no longer exists** after migration.
- Check: `grep -rc '@Test' src/test` before vs. after — any drop must map to a removed target class.

## Incomplete Namespace Migration — javax-pinned Binary Dependency
Fix priority — (b) is MANDATORY before halting:
**(a) Replacement** — check for jakarta-based release.
**(b) Eclipse Transformer** — rewrite javax→jakarta in the offending artifact at build time.
**(c) HALT** — ONLY if (a) and (b) fail.

## Multi-module EAR Test Consolidation
Tests in `ear/ejb-tests` submodules must be moved to the consolidated module's `src/test/java/`.

## Common Migration Pitfalls

### Bean Management Issues
- **Bean removed by unused-bean optimization** → Set `quarkus.arc.remove-unused-beans=false`
- **Final class cannot be proxied** → Add `quarkus.arc.transform-unproxyable-classes=true`
- **Missing no-arg constructor** → Normal-scoped beans need no-arg constructor for ArC proxies

### Threading Problems
- **@ApplicationScoped with mutable state** → Use `@RequestScoped` for per-request state
- **Self-invocation bypasses interceptors** → Extract to separate bean for `@Transactional`

### Migration Mistakes
- **Incomplete namespace migration** → Mixed `javax.*`/`jakarta.*` causes compilation errors
- **JNDI lookups remain** → `InitialContext` fails at Quarkus runtime
- **@Remove → @PreDestroy (WRONG)** → Simply remove the @Remove annotation
- **import.sql failures** → Verify table names match `@Table(name=...)` annotations

### Quick Fixes
- **ArC bean discovery**: Use `-Dquarkus.arc.unremovable-types` to debug missing beans
- **Schema issues**: Set `quarkus.hibernate-orm.schema-management.strategy=drop-and-create` for dev
- **JMS blocking**: Always add `@Blocking` to REST endpoints that call JMS/messaging

## Gradle Project + Maven BUILD COMMAND Mismatch

**Resolution**: NEVER create a `pom.xml` for a Gradle-only project. Use `./gradlew build` or static verification.

## Missing Gradle Wrapper (gradlew)

**Resolution:** Run `gradle wrapper --gradle-version=8.10` using system Gradle. Do NOT use curl/wget.

## Micrometer MeterRegistry — Never @Mock

**Fix**: Replace `@Mock MeterRegistry` with `new SimpleMeterRegistry()` in all unit tests.

## Version Lookups — Never curl/wget

The target version is `3.33.2` (LTS). Never invoke `curl`, `wget`, or any HTTP client — the denylist fires on the command name regardless of arguments and hard-terminates the pipeline.

## Cross-Task Field Rename — setField() String Staleness

After renaming `@Inject` fields, update string literals in `ReflectionTestUtils.setField(service, "oldName", mock)`.

## Comment False-Positives in Validation Greps

MIGRATION comments quoting literal annotation names (`@EJB`, `@Stateless`) cause false-positive failures. Rewrite comments to use prose descriptions.

## H2 Test Override Dialect Interaction

**Problem**: Main config sets explicit dialect. Test sets `db-kind=h2` without overriding dialect. Result: SQL syntax errors.

**Fix**: When main config has explicit dialect, test override MUST set BOTH:
```properties
%test.quarkus.datasource.db-kind=h2
%test.quarkus.hibernate-orm.dialect=org.hibernate.dialect.H2Dialect
```

## Inline Fully-Qualified Annotations (@javax.X.Y)

**Detection**: `grep -rn '\bjavax\.' src/ | grep -v 'javax\.sql\|javax\.crypto\|javax\.security\.auth\|javax\.net\|javax\.naming' | grep -v '^\s*//' | grep -v '^\s*\*'`
**Fix**: Replace each `@javax.X.Y` with `@jakarta.X.Y`. Also catches non-annotation type references (field declarations, catch clauses, casts).

## javax.annotation.security Namespace Migration

`javax.annotation.security.RolesAllowed` must migrate to `jakarta.annotation.security.RolesAllowed`. Easy to miss because @RolesAllowed "works unchanged" semantically — but the import still changes.

## Gradle-Specific Pitfalls

### testcontainers scope
In Gradle, `testImplementation` is the correct scope for testcontainers dependencies. Do NOT use `implementation` — testcontainers should not be in the production classpath.

### enforcedPlatform dual-scope
When using `enforcedPlatform("io.quarkus.platform:quarkus-bom:...")`, it must appear in BOTH `implementation` and `testImplementation` configurations, or use a top-level `dependencies { constraints { ... } }` block. Missing from test scope causes version conflicts for test-only dependencies.

### Deprecated sourceCompatibility
Use `java { sourceCompatibility = JavaVersion.VERSION_17 }` in `build.gradle`. The older `sourceCompatibility = '17'` string form may cause Quarkus plugin validation warnings.
