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

## Test Preservation — Never Delete Surviving Tests
Rule: a test may only be deleted if its **target class no longer exists** after migration.
If a test mocks a removed JCA/EJB interface but the implementation survives as a CDI bean,
**rewrite the mock against the CDI type** — do not delete the test.
- Example (Cloud-Connectors, benchmark 2026-07-16): `MQTTResourceAdapterTest` (4 methods) and
  `MQTTWorkTest` (3 methods) were deleted while `MQTTResourceAdapter`/`MQTTWork` still exist as
  CDI beans/Runnables. This violates the plan's own "mock new CDI-based classes" instruction and
  scores as a harmful code change.
- Check: `grep -rc '@Test' src/test` before vs. after — any drop must map to a removed target class.

## Incomplete Namespace Migration — javax-pinned Binary Dependency
Symptom: after migration, `grep -rc 'javax\.' src/` still returns many hits (e.g. rapla:
505 `javax.inject`, 156 `javax.ws.rs`, 98 `javax.servlet`) because a binary dependency with no
`jakarta.*` release (e.g. `restinject`) forces `javax.*`.
Fix priority — (b) is MANDATORY before halting; a javax-pinned binary is NOT sufficient reason to halt:

**(a) Replacement** — check for a jakarta-based release or newer major version of the library first.

**(b) Eclipse Transformer (resolves the majority of cases, incl. rapla/restinject)** — rewrite
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

### Quick Fixes
- **ArC bean discovery**: Use `-Dquarkus.arc.unremovable-types` to debug missing beans
- **Schema issues**: Set `quarkus.hibernate-orm.database.generation=drop-and-create` for dev
- **JMS blocking**: Always add `@Blocking` to REST endpoints that call JMS/messaging