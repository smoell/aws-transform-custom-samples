# ArC Build-Time CDI Limitations Reference

> Understanding Quarkus ArC (build-time CDI) vs full CDI. See https://quarkus.io/guides/cdi-reference

ArC resolves bean discovery/wiring at **build time** (unlike runtime Weld/OWB) — faster startup, lower memory, but restrictions on dynamic CDI features.

## NOT SUPPORTED in ArC

### Portable Extensions (`javax.enterprise.inject.spi.Extension`)
Silently ignored — no lifecycle events fire; beans it would register are missing → `UnsatisfiedResolutionException` (or no error at all). **Fix**: convert to a Quarkus Build Extension with `@BuildStep` + `SyntheticBeanBuildItem`:
```java
public class CustomExtensionProcessor {
    @BuildStep @Record(ExecutionTime.RUNTIME_INIT)
    SyntheticBeanBuildItem registerMyService(CustomRecorder recorder) {
        return SyntheticBeanBuildItem.configure(MyService.class)
            .scope(ApplicationScoped.class).runtimeValue(recorder.createMyService()).done();
    }
}
```
Build extensions need a multi-module `-deployment`/`-runtime` setup. For simpler cases use `@Produces` methods in an `@ApplicationScoped` config bean.

### bean-discovery-mode="all"
Ignored — only classes with a bean-defining annotation (`@ApplicationScoped`, `@RequestScoped`, `@Singleton`, `@Dependent`, …) are discovered → `UnsatisfiedResolutionException: No bean found for type X`. **Fix**: add explicit scope annotations, or `quarkus.arc.unremovable-types=com.example.legacy.**`.

### @Specializes
Build fails (`DeploymentException: Unsupported feature: @Specializes`) or is ignored (→ `AmbiguousResolutionException`). **Fix**: use `@Alternative` + `@Priority(1)` on the overriding subclass instead.

### @New Qualifier
Not recognized — resolves to the normal scoped bean (no error, wrong semantics). **Fix**: use `@Dependent` explicitly or inject `Instance<T>` and `.get()`/`.destroy()` per lookup.

### Private Interceptors
Build fails — ArC proxies cannot override private methods (`DeploymentException: Interceptor binding applied to a private method`). **Fix**: make the intercepted method at least package-private:
```java
@ApplicationScoped
public class MyService {
    @Transactional void doWork() { }  // package-private (was private)
}
```

### @Vetoed on Producers
Ignored — the producer still participates (no error, active when it should be excluded). **Fix**: remove the producer, or guard with `@IfBuildProfile`/`@UnlessBuildProfile`.

## LIMITED SUPPORT

### @Decorator
**Works**: single decorated type with `@Delegate`. **Doesn't**: multiple decorated types, complex generic delegate types, `@Dependent` decorators with `@PreDestroy`. Must be enabled via `@Priority` — ArC does NOT read `<decorators>` from beans.xml.
```java
@Decorator @Priority(10)
public class LoggingNotificationDecorator implements NotificationSender {
    @Inject @Delegate @Any NotificationSender delegate;
    @Override public void send(String m) { Log.info("Sending: " + m); delegate.send(m); }
}
```

### @Observes(notifyObserver = IF_EXISTS)
Default reception (ALWAYS) works. `Reception.IF_EXISTS` (deliver only if a bean instance already exists) is NOT guaranteed — ArC may deliver regardless. **Workaround**: use standard `@Observes` and check context activity manually.

## Common Migration Pitfalls

### Unannotated Beans from bean-discovery-mode="all"
Legacy `all`-mode beans without scope annotations are not found post-migration → `UnsatisfiedResolutionException`. **Fix**: add `@ApplicationScoped` (shared) or `@Dependent` (not shared) to each; or `quarkus.arc.unremovable-types=com.example.legacy.**`; or `@Unremovable` for programmatic-only lookups.

### @Inject on Private Fields in Superclasses
May not inject (especially native mode) → `NullPointerException` (works JVM, fails native). **Fix**: use `protected`/package-private, or constructor injection (best).

### @Transactional Self-Invocation Proxy Bypass
Interceptors apply ONLY through the CDI proxy (external call). Self-calls (`this.method()`) bypass them — a `@Transactional(REQUIRES_NEW)` helper called internally silently runs in the caller's TX. **Fix**: extract to a separate bean:
```java
// WRONG: this.audit(order) from processOrder() — REQUIRES_NEW ignored
// CORRECT: @Inject AuditService auditService; auditService.audit(order); // via proxy → REQUIRES_NEW works
@ApplicationScoped
public class AuditService {
    @Inject EntityManager em;
    @Transactional(TxType.REQUIRES_NEW) public void audit(Order o) { em.persist(new AuditEntry(o)); }
}
```
**Test isolation pitfall**: a `@Transactional(REQUIRES_NEW)` helper in `@BeforeEach` commits in its own TX and survives `@TestTransaction` rollback. Fix with an `@AfterEach` `UserTransaction` bulk DELETE.

### Circular Dependencies
Mutual injection → build-time `CircularDependencyException`. **Fix**: break the cycle with lazy `Instance<T>` or `Provider<T>` `.get()`, or event-based decoupling (`Event<T>.fire()` + `@Observes`).

### Beans Referenced Only via Programmatic Lookup
ArC removes beans with no static injection point — a bean only accessed via `CDI.current().select(...)` / `Arc.container().instance(...)` gets removed → `UnsatisfiedResolutionException`. **Fix**: `@Unremovable` on the class; or `quarkus.arc.unremovable-types=...`; or replace `CDI.current()` with an injected `@Any Instance<T>` (static injection point retains the bean).

### @Observes with Qualifiers Across Packages
Custom qualifier with restricted access → observer not invoked. **Fix**: qualifier annotation must be `public` and in an accessible package; observer qualifier must match the producer exactly.

### Final Classes and Methods
Normal-scoped beans are proxied via subclassing — final classes/methods can't be subclassed → `DeploymentException: Bean class is final and cannot be proxied`. **Fix**: remove `final` (preferred); or `quarkus.arc.transform-unproxyable-classes=true` (bytecode-removes final at build time); or `@Dependent` scope (no proxy).

## Configuration Properties (`quarkus.arc.*`)

| Property | Default | Description |
|---|---|---|
| `remove-unused-beans` | `true` | Removes beans with no injection point. `false` during migration; `fwk` = only framework beans. |
| `unremovable-types` | (none) | Glob patterns never removed, e.g. `com.example.legacy.**`. |
| `detect-unused-false-positives` | `false` | Warns when unused-removal may have dropped programmatically-looked-up beans. Enable during migration. |
| `transform-unproxyable-classes` | `true` | Auto-removes `final` from classes/methods needing proxying. |
| `auto-inject-fields` | `true` | Auto-adds `@Inject` to unassigned non-static fields. |
| `config-properties-default-naming-strategy` | `verbatim` | `@ConfigProperties` mapping strategy. |
| `dev-mode.monitoring-enabled` | `true` | CDI bean monitoring in dev UI (`/q/dev-ui/arc`). |
| `test.disable-application-lifecycle-observers` | `false` | Disables `@Observes StartupEvent`/`ShutdownEvent` in tests. |
| `selected-alternatives` | (none) | Activate alternatives by class name (vs `@Priority`). |
| `exclude-types` | (none) | Glob patterns excluded from discovery entirely. |

**Migration phase** (relaxed): `remove-unused-beans=false`, `detect-unused-false-positives=true`, `transform-unproxyable-classes=true`. **After stabilization** (stricter): `remove-unused-beans=true`, `detect-unused-false-positives=false`.
