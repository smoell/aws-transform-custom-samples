# EJB → CDI (ArC) Mapping Reference

> Reference for Phase 2: Core Migration — EJB to Quarkus CDI bean conversion.

## @TransactionAttribute → @Transactional Mapping

| EJB TransactionAttribute | Quarkus @Transactional | Notes |
|--------------------------|----------------------|-------|
| REQUIRED (default) | @Transactional (or omit) | Default in both |
| REQUIRED (method-level, when class-level @Transactional present) | REMOVE | Redundant — class-level already provides REQUIRED |
| REQUIRES_NEW | @Transactional(REQUIRES_NEW) | |
| MANDATORY | @Transactional(MANDATORY) | |
| SUPPORTS | See SUPPORTS Rule below | Context-dependent |
| NOT_SUPPORTED | @Transactional(NOT_SUPPORTED) | Suspends outer TX |
| NEVER | @Transactional(NEVER) | |

**Phase 2 validation:** `grep -rn '@TransactionAttribute' src/main/java/` must return empty after migration.

**Class-level @Transactional guidance:** Apply class-level `@Transactional` ONLY when the original `@Stateless` bean uses EntityManager/persistence operations. Quarkus Hibernate ORM requires an active transaction for `EntityManager.find()` and `TypedQuery` operations — not just writes. Beans WITHOUT EntityManager that merely delegate to other services do NOT need @Transactional.

---

## ⚠️ BMT EXCEPTION — @TransactionManagement(BEAN)

**Beans with `@TransactionManagement(TransactionManagementType.BEAN)` must NOT receive class-level `@Transactional`.** They manage their own transaction boundaries via `UserTransaction`.

**Action**: Drop `@TransactionManagement(BEAN)` annotation entirely. Keep `@Inject UserTransaction`. Do NOT add `@Transactional` at class or method level.

```java
// AFTER (Quarkus) — NO @Transactional on class
@ApplicationScoped
public class BatchService {
    @Inject UserTransaction utx;
    // ... manual begin/commit/rollback
}
```

**Detection**: `grep -rn '@TransactionManagement' src/main/java/` — if found with `BEAN`, apply BMT pattern.

---

## SUPPORTS Rule (Authoritative — v2.2 Rewrite)

Two mutually exclusive rules based on whether the class carries class-level `@Transactional`:

### Rule A: Class HAS class-level @Transactional

**Every `@TransactionAttribute(SUPPORTS)` method annotation is unconditionally redundant — REMOVE it regardless of EntityManager usage.**

**Rationale**: Under Hibernate ORM 6 (Quarkus 3.x), ALL EntityManager operations (including reads like `find()`, `createQuery().getResultList()`) require an active transaction. With class-level `@Transactional` (REQUIRED) in place, a transaction is always present. Adding `@Transactional(TxType.SUPPORTS)` would allow the method to run WITHOUT a transaction when called outside a TX context — which would break any EntityManager usage. Therefore SUPPORTS is both redundant (when TX present) and dangerous (when TX absent). Remove unconditionally.

```java
// BEFORE (JavaEE)
@Stateless
public class CatalogService {
    @PersistenceContext private EntityManager em;

    @TransactionAttribute(TransactionAttributeType.SUPPORTS)
    public List<Product> findAll() { return em.createQuery(...).getResultList(); }

    @TransactionAttribute(TransactionAttributeType.SUPPORTS)
    public String getServiceName() { return "catalog"; }  // no EntityManager
}

// AFTER (Quarkus) — REMOVE all SUPPORTS, let class-level REQUIRED apply
@ApplicationScoped
@Transactional
public class CatalogService {
    @Inject EntityManager em;

    public List<Product> findAll() { return em.createQuery(...).getResultList(); }

    public String getServiceName() { return "catalog"; }
}
```

### Rule B: Class does NOT have class-level @Transactional

**Only remove `@TransactionAttribute(SUPPORTS)` when the method does NOT use EntityManager.** If the method uses EntityManager, map to `@Transactional(TxType.SUPPORTS)` — the caller is responsible for providing the transaction context.

```java
// Class WITHOUT class-level @Transactional (e.g., pure delegation bean)
@ApplicationScoped
public class UtilityService {
    // Method using EntityManager → keep SUPPORTS mapping
    @Transactional(TxType.SUPPORTS)
    public Config loadConfig(EntityManager em) { ... }

    // Method NOT using EntityManager → remove entirely (no annotation needed)
    public String formatName(String input) { return input.trim(); }
}
```

---

## beans.xml Interceptor/Alternative Migration

When `beans.xml` contains `<interceptors>`, `<alternatives>`, or `<decorators>` with non-empty content, these declarations must be migrated BEFORE deleting beans.xml:

| beans.xml Element | Quarkus CDI Migration |
|---|---|
| `<interceptors><class>com.example.LogInterceptor</class></interceptors>` | Add `@jakarta.interceptor.Interceptor` + `@jakarta.annotation.Priority(value)` to the interceptor class |
| `<alternatives><class>com.example.MockService</class></alternatives>` | Add `@jakarta.enterprise.inject.Alternative` + `@jakarta.annotation.Priority(value)` to the alternative class |
| `<decorators><class>com.example.LoggingDecorator</class></decorators>` | Add `@jakarta.decorator.Decorator` + `@jakarta.annotation.Priority(value)` to the decorator class |

```java
// AFTER: self-activating via @Priority (no beans.xml needed)
@Interceptor
@Priority(Interceptor.Priority.APPLICATION)
public class LogInterceptor {
    @AroundInvoke
    public Object log(InvocationContext ctx) throws Exception { ... }
}
```

**Rule**: After migrating all interceptors/alternatives/decorators to use `@Priority`, delete beans.xml.

---

## EJB Stereotype Mapping

| EJB Annotation | Quarkus CDI Equivalent | Notes |
|---|---|---|
| `@Stateless` | `@ApplicationScoped` + `@Transactional` (if uses EntityManager) | Class-level `@Transactional` only for beans with persistence |
| `@Stateful` | `@SessionScoped` | Evaluate if stateful pattern is still needed; bean MUST be `Serializable` |
| `@Singleton` (javax.ejb) | `@ApplicationScoped` | Do NOT use `jakarta.inject.Singleton` (not a normal scope, cannot be intercepted) |
| `@Singleton` + `@Startup` | `@ApplicationScoped` + `void onStart(@Observes StartupEvent ev)` | Startup observer replaces `@PostConstruct` eager init — see Test Compatibility below |
| `@Singleton` + `@Lock(READ)` | `@ApplicationScoped` + `ReadWriteLock` | Manual concurrency control required |
| `@MessageDriven` | See `references/jms-to-smallrye.md` | Defer to messaging migration reference |
| `@LocalBean` | REMOVE | All CDI beans are "local" — no equivalent needed |
| `@Local` | REMOVE | No local/remote distinction in CDI |
| `@Remote` | REMOVE — redesign to REST/gRPC | No remote interface support in Quarkus |
| `@Remove` | REMOVE annotation only | Method becomes regular business method — see below |
| `@Asynchronous` | Uni<T> with Mutiny worker pool | Future<T>→Uni<T>, void→Uni<Void> — see Example 4 below |
| `@TransactionManagement(BEAN)` | DROP annotation; do NOT add @Transactional; keep `@Inject UserTransaction` | See BMT EXCEPTION above |

### @Remove (on @Stateful EJBs)

- **Action**: Simply remove the `@Remove` annotation. The method becomes a regular business method.
- **Do NOT** convert to `@PreDestroy`. `@PreDestroy` fires on bean destruction (session expiry), not after a specific business call.

### @PostConstruct → StartupEvent: Test Compatibility Pattern

When migrating `@Singleton` + `@Startup` + `@PostConstruct` to Quarkus, **keep the original `init()` method name and visibility unchanged**. Add `onStart(@Observes StartupEvent ev)` as a delegator that calls `init()`.

```java
// AFTER (Quarkus) — CORRECT: keep init() as-is, add delegator
@ApplicationScoped
public class ConfigCache {
    public void init() {  // name and visibility preserved for tests
        cache = loadFromDatabase();
    }

    void onStart(@Observes StartupEvent ev) {
        init();  // delegator — tests call init() directly
    }
}
```

**Rules:**
1. **Keep `init()` method name** — do NOT rename. Renaming breaks unit test setUp() methods.
2. **Remove `@PostConstruct` annotation** from `init()` — do NOT migrate to `jakarta.annotation.PostConstruct`. This supersedes the generic javax→jakarta namespace rule.
3. **If both retained**, both fire on startup (double-invocation). Remove `@PostConstruct` unless init() is idempotent.

### @Schedules (Plural) → Multiple @Scheduled Methods

EJB `@Schedules({@Schedule(...), @Schedule(...)})` → extract body to private helper, create N `@Scheduled` methods each calling the helper:

```java
// AFTER (Quarkus)
@Scheduled(cron = "0 0 9 ? * MON")
void weeklyStartReport() { doWeeklyBoundaryReport(); }

@Scheduled(cron = "0 0 17 ? * FRI")
void weeklyEndReport() { doWeeklyBoundaryReport(); }

private void doWeeklyBoundaryReport() { generateWeeklyReport(); }
```

### EE Concurrency: ContextService.createContextualProxy() Removal

- **Action**: Remove the `createContextualProxy()` call entirely. `ManagedExecutor` in Quarkus propagates context automatically.
- **Detection**: `grep -rn 'createContextualProxy\|ContextService' src/main/java/`

### @Stateful Caveats

- **Serialization**: `@SessionScoped` beans MUST implement `Serializable` for HTTP session serialization.
- **ArC proxy requirement**: normal-scoped beans require no-arg constructor (can be package-private).
- **Cloud-native**: Prefer externalizing state to Redis/database and using `@ApplicationScoped` stateless beans.
- **@PrePassivate / @PostActivate**: DELETE entirely — no Quarkus equivalent.

## Transaction Attribute Mapping

| EJB TransactionAttributeType | Quarkus Equivalent | Behavior |
|---|---|---|
| `REQUIRED` (default) | `@Transactional` or `@Transactional(TxType.REQUIRED)` | Join existing TX or create new one |
| `REQUIRES_NEW` | `@Transactional(TxType.REQUIRES_NEW)` | Always create a new TX, suspend existing |
| `MANDATORY` | `@Transactional(TxType.MANDATORY)` | Must run within existing TX, throws if none |
| `NOT_SUPPORTED` | `@Transactional(TxType.NOT_SUPPORTED)` | Suspend existing TX, run without TX |
| `SUPPORTS` | `@Transactional(TxType.SUPPORTS)` | Use existing TX if present, otherwise non-transactional |
| `NEVER` | `@Transactional(TxType.NEVER)` | Must NOT run within a TX, throws if one exists |

### Transaction Notes

- `@Transactional` import: `jakarta.transaction.Transactional` (NOT `org.springframework.transaction.annotation.Transactional`)
- `TxType` import: `jakarta.transaction.Transactional.TxType`
- Class-level `@Transactional` applies to ALL public methods (same as EJB `@Stateless` default `REQUIRED`)
- Method-level `@Transactional` overrides class-level (same precedence as EJB)
- **Rollback behavior**: `@Transactional` rolls back on `RuntimeException` by default. For checked exceptions: `@Transactional(rollbackOn = MyCheckedException.class)`
- **Self-invocation**: `@Transactional` only works through CDI proxy (external call). Self-calls bypass interceptor. Use separate bean or `Arc.container().instance(MyBean.class).get().method()`.

## Injection Migration

| EJB Pattern | Quarkus CDI Equivalent | Notes |
|---|---|---|
| `@EJB private MyService svc;` | `@Inject MyService svc;` | Simple replacement |
| `@EJB(beanName="orderSvc")` | `@Inject @Named("orderSvc") MyService svc;` | Add `@Named("orderSvc")` on target bean |
| `@EJB(lookup="java:global/...")` | `@Inject MyService svc;` | Remove JNDI lookup — CDI discovers by type |
| `@EJB(lookup="...")` with ambiguity | `@Inject @MyQualifier MyService svc;` | Create custom `@Qualifier` |
| `@Resource SessionContext ctx` | `@Inject SecurityContext ctx;` | See security migration reference |
| `@Resource TimerService ts` | `@Inject Scheduler scheduler;` | Quarkus scheduler API |
| `@Resource(lookup="java:comp/env/...")` | `@ConfigProperty(name="key")` | Config property injection |

### Constructor Injection (Preferred)

```java
@ApplicationScoped
public class OrderService {
    private final InventoryService inventoryService;

    OrderService() {} // package-private no-arg constructor for ArC proxy

    @Inject
    public OrderService(InventoryService inventoryService) {
        this.inventoryService = inventoryService;
    }
}
```

## Worked Examples

### Example 1: Simple @Stateless Service Bean

```java
// BEFORE (JavaEE)
@Stateless
public class OrderService {
    @PersistenceContext private EntityManager em;
    @EJB private InventoryService inventoryService;

    public Order createOrder(OrderRequest request) {
        inventoryService.reserve(request.getItemId(), request.getQuantity());
        Order order = new Order(request);
        em.persist(order);
        return order;
    }
}

// AFTER (Quarkus)
@ApplicationScoped
@Transactional  // class-level — bean uses EntityManager
public class OrderService {
    @Inject EntityManager em;
    @Inject InventoryService inventoryService;

    public Order createOrder(OrderRequest request) {
        inventoryService.reserve(request.getItemId(), request.getQuantity());
        Order order = new Order(request);
        em.persist(order);
        return order;
    }
}
```

### Example 2: @Stateless with Method-Level @TransactionAttribute

```java
// BEFORE (JavaEE)
@Stateless
public class PaymentService {
    @EJB private AuditService auditService;

    public void processPayment(Payment payment) { /* payment logic */ }

    @TransactionAttribute(TransactionAttributeType.REQUIRES_NEW)
    public void recordAudit(String action, String details) {
        auditService.log(action, details);
    }

    @TransactionAttribute(TransactionAttributeType.NOT_SUPPORTED)
    public PaymentStatus checkExternalStatus(String paymentRef) {
        return externalGateway.getStatus(paymentRef);
    }
}

// AFTER (Quarkus)
@ApplicationScoped
@Transactional  // default REQUIRED for all methods
public class PaymentService {
    @Inject AuditService auditService;

    public void processPayment(Payment payment) { /* payment logic */ }

    @Transactional(TxType.REQUIRES_NEW)
    public void recordAudit(String action, String details) {
        auditService.log(action, details);
    }

    @Transactional(TxType.NOT_SUPPORTED)
    public PaymentStatus checkExternalStatus(String paymentRef) {
        return externalGateway.getStatus(paymentRef);
    }
}
```

### Example 3: @Singleton with @Lock(READ/WRITE) → ReadWriteLock

```java
// AFTER (Quarkus)
@ApplicationScoped
public class ConfigCache {
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private Map<String, String> cache;

    public void init() { cache = loadFromDatabase(); }
    void onStart(@Observes StartupEvent ev) { init(); }

    public String getValue(String key) {
        lock.readLock().lock();
        try { return cache.get(key); } finally { lock.readLock().unlock(); }
    }

    public void refresh() {
        lock.writeLock().lock();
        try { cache = loadFromDatabase(); } finally { lock.writeLock().unlock(); }
    }
}
```

### Example 4: @Asynchronous Method → Uni<T>

```java
// BEFORE (JavaEE)
@Stateless
public class NotificationService {
    @Asynchronous
    public Future<String> sendEmail(String to, String subject, String body) {
        String messageId = emailGateway.send(to, subject, body);
        return new AsyncResult<>(messageId);
    }
}

// AFTER (Quarkus — Mutiny Uni)
@ApplicationScoped
public class NotificationService {
    public Uni<String> sendEmail(String to, String subject, String body) {
        return Uni.createFrom().item(() -> emailGateway.send(to, subject, body))
            .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }
}
```

**CRITICAL**: Mutiny Uni is lazy — calling without `.subscribe()` or `.await()` is a no-op. Update ALL callers.

### Example 5: EJB Timer (@Schedule) → Quarkus @Scheduled

```java
// AFTER (Quarkus)
@ApplicationScoped
public class ReportScheduler {
    @Scheduled(cron = "0 0 2 * * ?")
    public void dailyReport() { generateDailyReport(); }
}
```

### Example 6: @ApplicationException → ExceptionMapper

```java
// BEFORE: @ApplicationException(rollback = true)
// AFTER: @Transactional(rollbackOn = MyException.class) on throwing method
//        + @Provider ExceptionMapper for HTTP response mapping
//        DELETE @ApplicationException annotation from exception class
```

## ArC-Specific Caveats

| Feature | Status | Workaround |
|---|---|---|
| `@PreDestroy` on `@Dependent` | ⚠ Not guaranteed | Use `@ApplicationScoped` or explicit cleanup |
| Remote interfaces (`@Remote`) | ❌ Not supported | Redesign to REST or gRPC |
| `@Lock(READ/WRITE)` | ❌ No equivalent | Use `ReadWriteLock` manually |
| `@AccessTimeout` | ❌ No equivalent | Implement timeout manually |
| `@ConcurrencyManagement(CONTAINER)` | ❌ No equivalent | Add manual synchronization |
| `@DependsOn` | ❌ No equivalent | Use `@Observes StartupEvent` + `@Priority` |
| `@RunAs` | ❌ No equivalent | Use `SecurityIdentityAugmentor` |

### Self-Invocation Gotcha

In CDI, interceptors (`@Transactional`) only apply through the **proxy** (external call). Self-calls bypass. Solutions:
1. Extract method to separate bean (preferred)
2. Use `Arc.container().instance(MyBean.class).get().method()`

## Updating Call Sites of Migrated @Asynchronous Methods (Uni Caller-Side)

After migrating `@Asynchronous` methods from `Future<T>` to `Uni<T>`, ALL callers must be updated:
- Fire-and-forget: add `.subscribe().with(v -> {}, e -> LOG.warn(...))` 
- Blocking: `.await().indefinitely()` (caller must be `@Blocking`)
- JAX-RS endpoint returning Uni: framework subscribes automatically

## @ConversationScoped → @SessionScoped Fallback
Quarkus ArC does NOT support @ConversationScoped. Use @SessionScoped as fallback with `implements Serializable`.

## Edge Cases — Additional Rules

### @Resource → @ConfigProperty — Preserve Integer Wrapper Type

When converting `@Resource` to `@ConfigProperty`, preserve wrapper types (`Integer` not `int`) for null-check and Mockito compatibility.

### @Singleton + @Startup: StartupEvent OVERRIDES @PostConstruct

Migration order: (1) Remove `@PostConstruct`; (2) Add `void onStart(@Observes StartupEvent ev) { init(); }`. Do NOT also migrate @PostConstruct to jakarta — it should be gone entirely.

### Orphaned Import Cleanup After JNDI Block Deletion

After deleting JNDI init blocks, check ALL imports at file top. Delete orphaned imports (InitialContext, NamingException, PostConstruct if method deleted).

### CDI Self-Invocation Proxy Bypass for @TransactionAttribute

When a method annotated `REQUIRES_NEW` or `NOT_SUPPORTED` is called via `this.method()`, extract to separate bean to ensure proxy invocation.

### ContextService.createContextualProxy() — ALL Sites Must Be Removed

Detection: `grep -rn 'createContextualProxy' src/main/java/`. Remove each proxy wrapping while preserving task submission logic.
