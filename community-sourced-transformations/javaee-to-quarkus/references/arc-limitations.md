# ArC Build-Time CDI Limitations Reference

> Reference for understanding Quarkus ArC (build-time CDI) vs full CDI specification compatibility.
> See also: https://quarkus.io/guides/cdi-reference

## Overview

Quarkus ArC is a build-time CDI implementation. Unlike Weld (JBoss/WildFly) or OWB (TomEE) which resolve beans at runtime, ArC performs bean discovery, resolution, and wiring at build time. This produces faster startup and lower memory but imposes restrictions on dynamic CDI features.

## NOT SUPPORTED in ArC

### Portable Extensions (`javax.enterprise.inject.spi.Extension`)

**What happens if used**: The extension class is silently ignored. No `ProcessAnnotatedType`, `AfterBeanDiscovery`, or other lifecycle events fire. Beans registered by the extension will be missing at runtime.

**Error you'll see**: `UnsatisfiedResolutionException` for beans that were supposed to be registered by the extension; or no error at all — the extension simply does nothing.

**Workaround**: Convert to a Quarkus Build Extension using `@BuildStep`:

```java
// BEFORE (CDI Portable Extension)
public class CustomExtension implements Extension {
    public void afterBeanDiscovery(@Observes AfterBeanDiscovery abd) {
        abd.addBean()
            .types(MyService.class)
            .scope(ApplicationScoped.class)
            .createWith(ctx -> new MyServiceImpl());
    }
}
// Registered via META-INF/services/javax.enterprise.inject.spi.Extension

// AFTER (Quarkus Build Extension)
// In a separate extension module (src/main/java in a -deployment artifact)
import io.quarkus.arc.deployment.SyntheticBeanBuildItem;
import io.quarkus.deployment.annotations.BuildStep;
import io.quarkus.deployment.annotations.Record;
import io.quarkus.deployment.annotations.ExecutionTime;

public class CustomExtensionProcessor {
    @BuildStep
    @Record(ExecutionTime.RUNTIME_INIT)
    SyntheticBeanBuildItem registerMyService(CustomRecorder recorder) {
        return SyntheticBeanBuildItem.configure(MyService.class)
            .scope(ApplicationScoped.class)
            .runtimeValue(recorder.createMyService())
            .done();
    }
}
```

**Note**: Build extensions require a multi-module Maven setup (`-deployment` and `-runtime` modules). For simpler cases, use `@Produces` methods in an `@ApplicationScoped` configuration bean instead.

### bean-discovery-mode="all" (Implicit Bean Archives)

**What happens if used**: beans.xml with `bean-discovery-mode="all"` is ignored. Only classes with a bean-defining annotation (`@ApplicationScoped`, `@RequestScoped`, `@Singleton`, `@Dependent`, etc.) are discovered.

**Error you'll see**: `UnsatisfiedResolutionException: No bean found for type X` for classes that have no CDI scope annotation but were previously discovered by the "all" mode.

**Workaround**: Add explicit scope annotations to all beans, or use `quarkus.arc.unremovable-types` config:

```java
// Option 1: Add explicit annotation (preferred)
@ApplicationScoped  // was implicitly discovered in "all" mode
public class HelperService {
    // ...
}

// Option 2: If many unannotated beans, use config
// application.properties:
// quarkus.arc.unremovable-types=com.example.legacy.**
```

### @Specializes

**What happens if used**: Build fails with `jakarta.enterprise.inject.spi.DeploymentException` or the `@Specializes` annotation is silently ignored depending on the ArC version.

**Error you'll see**: `DeploymentException: Unsupported feature: @Specializes` or `AmbiguousResolutionException` if both beans are discovered.

**Workaround**: Use `@Alternative` + `@Priority` to override the base bean:

```java
// BEFORE (@Specializes)
@ApplicationScoped
public class DefaultNotifier {
    public void notify(String msg) { /* default impl */ }
}

@Specializes
@ApplicationScoped
public class SmsNotifier extends DefaultNotifier {
    @Override
    public void notify(String msg) { /* SMS impl */ }
}

// AFTER (@Alternative + @Priority)
@ApplicationScoped
public class DefaultNotifier {
    public void notify(String msg) { /* default impl */ }
}

@Alternative
@Priority(1)
@ApplicationScoped
public class SmsNotifier extends DefaultNotifier {
    @Override
    public void notify(String msg) { /* SMS impl */ }
}
```

### @New Qualifier

**What happens if used**: `@New` qualifier is not recognized. The injection point resolves to the normal bean (ignoring `@New` semantics).

**Error you'll see**: No compile error, but runtime behavior differs — you get the existing scoped instance instead of a new dependent one.

**Workaround**: Use `@Dependent` scope explicitly or inject `Instance<T>` for per-lookup creation:

```java
// BEFORE (@New)
@Inject @New
private MyProcessor processor;

// AFTER (Instance<T> for per-lookup)
@Inject
Instance<MyProcessor> processorInstance;

public void process() {
    MyProcessor processor = processorInstance.get();
    try {
        processor.execute();
    } finally {
        processorInstance.destroy(processor);
    }
}
```

### Private Interceptors

**What happens if used**: Build fails. ArC generates subclasses/proxies for intercepted beans and cannot override private methods.

**Error you'll see**: `DeploymentException: Interceptor binding applied to a private method` or a build-time error about non-proxyable class.

**Workaround**: Make interceptor methods at least package-private:

```java
// BEFORE (private interceptor method — fails)
@ApplicationScoped
public class MyService {
    @Transactional
    private void doWork() { /* ... */ }
}

// AFTER (package-private — works)
@ApplicationScoped
public class MyService {
    @Transactional
    void doWork() { /* ... */ }  // package-private
}
```

### @Vetoed on Producers

**What happens if used**: `@Vetoed` on producer methods or producer fields is ignored. The producer still participates in bean discovery.

**Error you'll see**: No error — the producer is active when it was expected to be excluded.

**Workaround**: Remove the producer entirely, or guard it with `@IfBuildProfile` / `@UnlessBuildProfile`:

```java
// BEFORE (@Vetoed on producer — ignored by ArC)
@Vetoed
@Produces
@ApplicationScoped
public DataSource createDataSource() { /* ... */ }

// AFTER (conditional with build profile)
@Produces
@ApplicationScoped
@IfBuildProfile("dev")
public DataSource createDevDataSource() { /* ... */ }
```

## LIMITED SUPPORT

### @Decorator

**What works**: Basic decorator patterns with a single decorated type and `@Delegate` injection point.

**What doesn't work**:
- Decorators with multiple decorated types (implements multiple interfaces)
- Complex generic type resolution on the `@Delegate` point
- Decorators on `@Dependent` beans with `@PreDestroy` logic

**Conditions for breakage**: If the decorator implements more than one bean type or uses complex parameterized types on the delegate.

**Supported pattern**:

```java
// Interface
public interface NotificationSender {
    void send(String message);
}

// Base bean
@ApplicationScoped
public class EmailSender implements NotificationSender {
    @Override
    public void send(String message) {
        // send email
    }
}

// Decorator (SUPPORTED — single interface, simple delegate)
@Decorator
@Priority(10)
public class LoggingNotificationDecorator implements NotificationSender {
    @Inject
    @Delegate
    @Any
    NotificationSender delegate;

    @Override
    public void send(String message) {
        Log.info("Sending notification: " + message);
        delegate.send(message);
    }
}
```

**Note**: Decorators must be enabled via `@Priority` — ArC does NOT read `<decorators>` from beans.xml.

### @Observes(notifyObserver = IF_EXISTS)

**What works**: `@Observes` with default reception (ALWAYS) works perfectly.

**What doesn't work**: `Reception.IF_EXISTS` — which in full CDI only delivers the event if a bean instance already exists in the current context. ArC may deliver the event regardless.

**Conditions for breakage**: When you rely on conditional delivery based on whether a `@RequestScoped` or `@SessionScoped` bean has already been instantiated in the current context.

**Supported pattern**:

```java
// SUPPORTED — standard observer
public void onOrderCreated(@Observes OrderCreatedEvent event) {
    // always invoked when event fires
}

// RISKY — IF_EXISTS behavior not guaranteed
public void onOrderCreated(@Observes(notifyObserver = Reception.IF_EXISTS) OrderCreatedEvent event) {
    // may be invoked even if bean instance doesn't exist yet
}

// WORKAROUND — check manually
public void onOrderCreated(@Observes OrderCreatedEvent event) {
    if (isContextActive()) {
        // process event
    }
}
```

## Common Migration Pitfalls

### Unannotated Beans from bean-discovery-mode="all"

**Problem**: Legacy apps using `bean-discovery-mode="all"` in beans.xml have many classes without CDI scope annotations that were still discovered and injectable. After migration to Quarkus, these beans are not found.

**Symptom**: `jakarta.enterprise.inject.UnsatisfiedResolutionException: No bean found for type com.example.MyHelper`

**Solutions** (choose one):

```java
// Solution 1: Add @ApplicationScoped to every bean (preferred — explicit)
@ApplicationScoped
public class MyHelper { /* ... */ }

// Solution 2: Use @Dependent if bean should not be shared
@Dependent
public class MyHelper { /* ... */ }
```

```properties
# Solution 3: Config-based (for large legacy codebases with many unannotated beans)
quarkus.arc.unremovable-types=com.example.legacy.**
```

```java
// Solution 4: If bean is only looked up programmatically
@Unremovable
@ApplicationScoped
public class MyHelper { /* ... */ }
```

### @Inject on Private Fields in Superclasses

**Problem**: ArC may not inject private fields declared in superclasses, especially in native image mode where reflection access is restricted.

**Symptom**: `NullPointerException` at runtime on fields that should have been injected; works in JVM mode but fails in native.

**Solution**: Use protected or package-private visibility, or switch to constructor injection:

```java
// BEFORE (may fail in native mode)
public abstract class BaseService {
    @Inject
    private Logger logger;  // private in superclass — risky
}

// AFTER (safe for native image)
public abstract class BaseService {
    @Inject
    protected Logger logger;  // protected — ArC can access

    // OR — constructor injection (best)
    private final Logger logger;

    protected BaseService() {}

    @Inject
    protected BaseService(Logger logger) {
        this.logger = logger;
    }
}
```

### Circular Dependencies

**Problem**: Two beans that inject each other cause a `CircularDependencyException` at build time (ArC detects cycles during wiring).

**Symptom**: `DeploymentException: Circular dependency detected: BeanA -> BeanB -> BeanA`

**Solutions**:

```java
// Solution 1: Use Instance<T> (lazy lookup — breaks the cycle)
@ApplicationScoped
public class ServiceA {
    @Inject
    Instance<ServiceB> serviceBInstance;

    public void doWork() {
        serviceBInstance.get().process();  // resolved lazily
    }
}

// Solution 2: Use Provider<T>
@ApplicationScoped
public class ServiceA {
    @Inject
    Provider<ServiceB> serviceBProvider;

    public void doWork() {
        serviceBProvider.get().process();
    }
}

// Solution 3: Event-based decoupling (preferred for loose coupling)
@ApplicationScoped
public class ServiceA {
    @Inject
    Event<ProcessingRequest> event;

    public void doWork() {
        event.fire(new ProcessingRequest(data));
    }
}

@ApplicationScoped
public class ServiceB {
    public void onProcessing(@Observes ProcessingRequest req) {
        process(req);
    }
}
```

### Beans Referenced Only via Programmatic Lookup (CDI.current())

**Problem**: ArC's unused bean removal optimization removes beans that have no static injection points. If a bean is only accessed via `CDI.current().select(MyBean.class).get()` or `Arc.container().instance(MyBean.class).get()`, ArC considers it unused and removes it.

**Symptom**: `UnsatisfiedResolutionException` at runtime, or `Arc.container().instance(X)` returns an unsatisfied `InstanceHandle`.

**Solutions**:

```java
// Solution 1: @Unremovable annotation (on the bean class)
@Unremovable
@ApplicationScoped
public class DynamicallyLoadedService {
    // ArC will NOT remove this bean even without static injection points
}

// Solution 2: Configuration-based
// application.properties:
// quarkus.arc.unremovable-types=com.example.DynamicallyLoadedService

// Solution 3: Replace CDI.current() with @Inject Instance<T>
@ApplicationScoped
public class ServiceRegistry {
    @Inject
    @Any
    Instance<MyPlugin> plugins;  // static injection point — bean is retained

    public MyPlugin getPlugin(String name) {
        return plugins.stream()
            .filter(p -> p.getName().equals(name))
            .findFirst()
            .orElseThrow();
    }
}
```

### @Observes with Qualifiers Across Packages

**Problem**: When observer methods use custom qualifier annotations defined in a different package, the qualifier may not be visible if it has restricted access.

**Symptom**: Observer method not invoked for qualified events.

**Solution**: Ensure qualifier annotations are `public` and in an accessible package:

```java
// Qualifier annotation — MUST be public
@Qualifier
@Retention(RUNTIME)
@Target({METHOD, FIELD, PARAMETER, TYPE})
public @interface Premium { }  // must be public

// Event producer
@Inject
@Premium
Event<OrderEvent> premiumOrderEvent;

// Observer — qualifier must match exactly
public void onPremiumOrder(@Observes @Premium OrderEvent event) {
    // processes only @Premium-qualified events
}
```

### Final Classes and Methods

**Problem**: ArC generates subclass proxies for normal-scoped beans. Final classes or classes with final methods cannot be subclassed.

**Symptom**: `DeploymentException: Bean class is final and cannot be proxied` or `java.lang.Error: Unresolved compilation problem` at build time.

**Solutions**:

```java
// Solution 1: Remove final keyword (preferred)
@ApplicationScoped
public class MyService {  // was: public final class MyService
    public void process() { /* ... */ }  // was: public final void process()
}

// Solution 2: Enable class transformation in config
// application.properties:
// quarkus.arc.transform-unproxyable-classes=true
// (ArC will bytecode-transform the class to remove final at build time)

// Solution 3: Use @Dependent scope (no proxy generated)
@Dependent  // no proxy needed — injected as direct instance
public final class MyHelper {
    // final class is fine with @Dependent
}
```

## Configuration Properties

Key `quarkus.arc.*` properties that affect CDI behavior during migration:

| Property | Default | Description |
|---|---|---|
| `quarkus.arc.remove-unused-beans` | `true` | Removes beans with no injection point. Set to `false` during migration to avoid missing-bean surprises. Set to `fwk` to only remove beans from frameworks. |
| `quarkus.arc.unremovable-types` | (none) | Glob patterns for bean types that should never be removed. Example: `com.example.legacy.**` |
| `quarkus.arc.detect-unused-false-positives` | `false` | Logs warnings when unused bean removal may have removed beans that are actually needed (programmatic lookup). Enable during migration for debugging. |
| `quarkus.arc.transform-unproxyable-classes` | `true` | Automatically removes `final` from bean classes/methods that need proxying. Disable if you need to preserve final semantics for correctness. |
| `quarkus.arc.auto-inject-fields` | `true` | Automatically adds `@Inject` to non-static fields with no value assignment in bean classes. Useful for legacy code but can cause unexpected injections. |
| `quarkus.arc.config-properties-default-naming-strategy` | `verbatim` | Naming strategy for `@ConfigProperties` mapping. |
| `quarkus.arc.dev-mode.monitoring-enabled` | `true` | Enables CDI bean monitoring in dev mode UI (`/q/dev-ui/arc`). |
| `quarkus.arc.test.disable-application-lifecycle-observers` | `false` | Disables `@Observes StartupEvent`/`ShutdownEvent` in tests. |
| `quarkus.arc.selected-alternatives` | (none) | Activate alternatives by class name (alternative to `@Priority`). Example: `com.example.MockService` |
| `quarkus.arc.exclude-types` | (none) | Glob patterns for types to exclude from discovery entirely. |

### Recommended Migration Configuration

During active migration, use these settings to avoid false positives:

```properties
# Relaxed settings for migration phase (revert to defaults after stabilization)
quarkus.arc.remove-unused-beans=false
quarkus.arc.detect-unused-false-positives=true
quarkus.arc.transform-unproxyable-classes=true
```

After migration is stable:

```properties
# Production settings (stricter)
quarkus.arc.remove-unused-beans=true
quarkus.arc.detect-unused-false-positives=false
quarkus.arc.transform-unproxyable-classes=true
```
