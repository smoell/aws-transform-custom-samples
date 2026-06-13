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