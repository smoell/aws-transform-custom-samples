# EJB → CDI (ArC) Mapping Reference

> Reference for Phase 2: Core Migration — EJB to Quarkus CDI bean conversion.

## @TransactionAttribute → @Transactional Mapping

| EJB TransactionAttributeType | Quarkus `@Transactional` | Behavior |
|---|---|---|
| `REQUIRED` (default) | `@Transactional` or `@Transactional(TxType.REQUIRED)` | Join existing TX or create new one |
| `REQUIRES_NEW` | `@Transactional(TxType.REQUIRES_NEW)` | Always create a new TX, suspend existing |
| `MANDATORY` | `@Transactional(TxType.MANDATORY)` | Must run within existing TX, throws if none |
| `NOT_SUPPORTED` | `@Transactional(TxType.NOT_SUPPORTED)` | Suspend existing TX, run without TX |
| `SUPPORTS` | `@Transactional(TxType.SUPPORTS)` | Use existing TX if present, else non-transactional |
| `NEVER` | `@Transactional(TxType.NEVER)` | Must NOT run within a TX, throws if one exists |

**Phase 2 validation:** `grep -rn '@TransactionAttribute' src/main/java/` must return empty after migration.

### Transaction Notes

- `@Transactional` import: `jakarta.transaction.Transactional` (NOT the Spring one); `TxType` import: `jakarta.transaction.Transactional.TxType`.
- **Class-level `@Transactional`**: apply when the original `@Stateless` bean uses EntityManager in multiple methods (read AND write) — preserves EJB default REQUIRED semantics. Quarkus Hibernate ORM requires an active transaction for `EntityManager.find()` and `TypedQuery` operations, not just writes. Class-level applies to ALL public methods; method-level overrides it (same precedence as EJB).
- **Rollback**: `@Transactional` rolls back on `RuntimeException` by default. For checked exceptions: `@Transactional(rollbackOn = MyCheckedException.class)`.
- **Self-invocation**: `@Transactional` only works when called through the CDI proxy (external call). Self-calls (`this.method()`) bypass the interceptor — see Self-Invocation Gotcha below.

## EJB Stereotype Mapping

| EJB Annotation | Quarkus CDI Equivalent | Notes |
|---|---|---|
| `@Stateless` | `@ApplicationScoped` + `@Transactional` | Class-level `@Transactional` for beans using EntityManager |
| `@Stateful` | `@SessionScoped` | Evaluate if stateful pattern is still needed; bean MUST be `Serializable` |
| `@Singleton` (javax.ejb) | `@ApplicationScoped` | Do NOT use `jakarta.inject.Singleton` (not a normal scope, cannot be intercepted) |
| `@Singleton` + `@Startup` | `@ApplicationScoped` + `void onStart(@Observes StartupEvent ev)` | Startup observer replaces `@PostConstruct` eager init — see Test Compatibility below |
| `@Singleton` + `@Lock(READ)` | `@ApplicationScoped` + `ReadWriteLock` | Manual concurrency control required (see Example 3) |
| `@MessageDriven` | See `references/jms-to-smallrye.md` | Defer to messaging migration reference |
| `@LocalBean` / `@Local` | REMOVE | All CDI beans are "local" — no equivalent needed |
| `@Remote` | REMOVE — redesign to REST/gRPC | No remote interface support in Quarkus |
| `@Remove` | REMOVE annotation only | Method becomes regular business method — see below |

### @Remove (on @Stateful EJBs)

`@Remove` signals the EJB container to destroy a `@Stateful` bean after the method completes. In Quarkus with `@SessionScoped`:

- **Action**: Simply remove the `@Remove` annotation — the method becomes a regular business method.
- **Do NOT** convert to `@PreDestroy` (which fires on session expiry, not after a specific business call).
- **Lifecycle**: `@SessionScoped` bean lifecycle is managed by HTTP session expiry. When the session times out, the bean is destroyed.

```java
// BEFORE                          // AFTER
@Stateful                          @SessionScoped
public class ShoppingCart {        public class ShoppingCart implements Serializable {
    @Remove                            public void checkout() {
    public void checkout() { ... }         // bean lives until HTTP session expires
}                                      }
                                   }
```

### @PostConstruct → StartupEvent: Test Compatibility Pattern

When migrating `@Singleton` + `@Startup` + `@PostConstruct`, **keep the original `init()` method name and visibility unchanged**. Add `onStart(@Observes StartupEvent ev)` as a delegator that calls `init()`. This preserves unit-test compatibility — tests call `bean.init()` directly without a `StartupEvent`.

```java
// AFTER (Quarkus) — keep init() as-is, add delegator
@ApplicationScoped
public class ConfigCache {
    // Remove @PostConstruct — StartupEvent observer triggers init
    public void init() {          // name and visibility preserved for tests
        cache = loadFromDatabase();
    }
    void onStart(@Observes StartupEvent ev) {
        init();                   // delegator — tests call init() directly
    }
}
```

**Rules:**
1. **Keep `init()` method name** — renaming to `onStart()` breaks all unit-test `setUp()` methods that call `init()`.
2. **Remove `@PostConstruct`** from `init()` — rely solely on the StartupEvent observer.
3. **If `@PostConstruct` is retained alongside `onStart()`**, both fire on startup (double-invocation). Acceptable ONLY if `init()` is idempotent; otherwise remove `@PostConstruct`.
4. **Unit tests**: call `bean.init()` directly — no `StartupEvent` parameter needed.

### @Schedules (Plural) → Multiple @Scheduled Methods

EJB `@Schedules({@Schedule(...), @Schedule(...)})` combines multiple cron expressions on one method. Quarkus `@Scheduled` accepts only ONE cron/every expression per annotation ([Quarkus Scheduler Reference](https://quarkus.io/guides/scheduler-reference)). Extract the body to a private helper, then create N `@Scheduled` methods each calling the helper:

```java
// BEFORE                                       // AFTER
@Schedules({                                    @Scheduled(cron = "0 0 9 ? * MON")
    @Schedule(dayOfWeek = "Mon", hour = "9"),   void weeklyStartReport() { doWeeklyReport(); }
    @Schedule(dayOfWeek = "Fri", hour = "17")
})                                              @Scheduled(cron = "0 0 17 ? * FRI")
public void weeklyBoundaryReport(Timer t) {     void weeklyEndReport() { doWeeklyReport(); }
    generateWeeklyReport();
}                                               private void doWeeklyReport() { generateWeeklyReport(); }
```

### @Stateful Caveats

- **Serialization**: `@SessionScoped` beans MUST implement `Serializable`; all injected fields must be Serializable too.
- **ArC proxy**: normal-scoped beans require a no-arg constructor (can be package-private).
- **Cloud-native**: stateful beans are problematic for horizontal scaling. Prefer externalizing state to Redis/database with `@ApplicationScoped` stateless beans.
- **@PrePassivate / @PostActivate**: DELETE entirely — no Quarkus equivalent for stateful passivation/activation.

## Injection Migration

| EJB Pattern | Quarkus CDI Equivalent | Notes |
|---|---|---|
| `@EJB private MyService svc;` | `@Inject MyService svc;` | Simple replacement |
| `@EJB(beanName="orderSvc")` | `@Inject @Named("orderSvc") MyService svc;` | Add `@Named` on target bean class |
| `@EJB(lookup="java:global/...")` | `@Inject MyService svc;` | Remove JNDI lookup — CDI discovers by type |
| `@EJB(lookup="...")` with ambiguity | `@Inject @MyQualifier MyService svc;` | Create custom `@Qualifier` to disambiguate |
| `@Resource SessionContext ctx` | `@Inject SecurityContext ctx;` | See security migration reference |
| `@Resource TimerService ts` | `@Inject Scheduler scheduler;` | Quarkus scheduler API |
| `@Resource(lookup="java:comp/env/...")` | `@ConfigProperty(name="key")` | Config property injection |

### Constructor Injection (Preferred)

```java
@ApplicationScoped
public class OrderService {
    private final InventoryService inventoryService;
    OrderService() {}  // package-private no-arg constructor for ArC proxy

    @Inject
    public OrderService(InventoryService inventoryService) {
        this.inventoryService = inventoryService;
    }
}
```

**Note**: If the bean has only ONE constructor, `@Inject` on it is optional — ArC infers it.

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
    public Order findById(Long id) { return em.find(Order.class, id); }
}

// AFTER (Quarkus)
@ApplicationScoped
@Transactional  // class-level — all methods run in a transaction (read AND write)
public class OrderService {
    @Inject EntityManager em;
    private final InventoryService inventoryService;
    OrderService() {}
    @Inject public OrderService(InventoryService inv) { this.inventoryService = inv; }

    public Order createOrder(OrderRequest request) {
        inventoryService.reserve(request.getItemId(), request.getQuantity());
        Order order = new Order(request);
        em.persist(order);
        return order;
    }
    public Order findById(Long id) { return em.find(Order.class, id); }
}
```

### Example 2: Method-Level @TransactionAttribute

```java
// BEFORE: @Stateless with mixed transaction attributes
@Stateless
public class PaymentService {
    public void processPayment(Payment p) { /* default REQUIRED */ }

    @TransactionAttribute(TransactionAttributeType.REQUIRES_NEW)
    public void recordAudit(String action, String details) { auditService.log(action, details); }

    @TransactionAttribute(TransactionAttributeType.NOT_SUPPORTED)
    public PaymentStatus checkExternalStatus(String ref) { return externalGateway.getStatus(ref); }
}

// AFTER
@ApplicationScoped
@Transactional  // default REQUIRED for all methods
public class PaymentService {
    public void processPayment(Payment p) { /* inherits class-level REQUIRED */ }

    @Transactional(TxType.REQUIRES_NEW)
    public void recordAudit(String action, String details) { auditService.log(action, details); }

    @Transactional(TxType.NOT_SUPPORTED)
    public PaymentStatus checkExternalStatus(String ref) { return externalGateway.getStatus(ref); }
}
```

### Example 3: @Singleton with @Lock(READ/WRITE) → ReadWriteLock

`@Lock` has no ArC equivalent — replace with `java.util.concurrent.locks.ReadWriteLock`:

```java
// AFTER (Quarkus)
@ApplicationScoped
public class ConfigCache {
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private Map<String, String> cache;

    void onStart(@Observes StartupEvent ev) { cache = loadFromDatabase(); }

    public String getValue(String key) {
        lock.readLock().lock();
        try { return cache.get(key); } finally { lock.readLock().unlock(); }
    }
    public void setValue(String key, String value) {
        lock.writeLock().lock();
        try { cache.put(key, value); } finally { lock.writeLock().unlock(); }
    }
    public void refresh() {
        lock.writeLock().lock();
        try { cache = loadFromDatabase(); } finally { lock.writeLock().unlock(); }
    }
}
```

**Lock ordering**: keep `init()`/refresh helper lock-free; all locking lives in the public API methods (a non-reentrant `ReadWriteLock` would deadlock if `init()` re-acquired). For `@Singleton` with NO `@Lock` (ConcurrencyManagement.BEAN), just use `@ApplicationScoped` with no lock — beans are not thread-safe by default.

### Example 4: @Asynchronous Method → CompletionStage/Uni

```java
// BEFORE: @Asynchronous Future<String> / void fire-and-forget

// AFTER — Option A: Mutiny Uni
@ApplicationScoped
public class NotificationService {
    public Uni<String> sendEmail(String to, String subject, String body) {
        return Uni.createFrom().item(() -> emailGateway.send(to, subject, body))
            .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }
}

// AFTER — Option B: CompletionStage with ManagedExecutor
@ApplicationScoped
public class NotificationService {
    @Inject ManagedExecutor executor;
    public CompletionStage<String> sendEmail(String to, String subject, String body) {
        return executor.supplyAsync(() -> emailGateway.send(to, subject, body));
    }
}
```

**CRITICAL**: If the async method accesses `@RequestScoped` beans, annotate it with `@ActivateRequestContext` — ArC does not propagate request context to worker threads automatically.

**Cross-task dependency**: When converting `Future<T>` → `Uni<T>`, scan all callers for `Future<T>` declarations. Callers must be updated in the same task or scheduled AFTER the callee task.

### Example 5: EJB Timer (@Schedule) → Quarkus @Scheduled

```java
// AFTER (Quarkus)
@ApplicationScoped
public class ReportScheduler {
    @Scheduled(cron = "0 0 2 * * ?")   // daily at 2:00 AM
    public void dailyReport() { generateDailyReport(); }

    @Scheduled(every = "15m")          // every 15 minutes
    public void healthCheck() { checkSystemHealth(); }

    @Scheduled(cron = "0 0 9 ? * MON") // @Schedules split into separate methods
    public void weeklyStartReport() { generateWeeklyReport(); }
    @Scheduled(cron = "0 0 17 ? * FRI")
    public void weeklyEndReport() { generateWeeklyReport(); }
}
```

**Notes**: Remove the `Timer` parameter (Quarkus does not pass timer info). Remove `persistent = false` (Quarkus scheduled tasks are non-persistent by default). Cron format is Quartz-style `second minute hour dayOfMonth month dayOfWeek`. Add the `quarkus-scheduler` extension.

### Example 6: @ApplicationException → @Transactional rollbackOn + ExceptionMapper

```java
// Exception class: DELETE the @ApplicationException annotation (no Quarkus equivalent)
public class InsufficientFundsException extends Exception { /* fields, ctor, getters */ }

// Service — rollback=true equivalent declared on @Transactional
@ApplicationScoped
public class AccountService {
    @Inject EntityManager em;

    @Transactional(rollbackOn = InsufficientFundsException.class)
    public void withdraw(Long accountId, BigDecimal amount) throws InsufficientFundsException {
        Account account = em.find(Account.class, accountId);
        if (account.getBalance().compareTo(amount) < 0)
            throw new InsufficientFundsException(account.getBalance(), amount);
        account.debit(amount);
    }
}

// JAX-RS ExceptionMapper — maps exception to HTTP response
@Provider
public class InsufficientFundsExceptionMapper implements ExceptionMapper<InsufficientFundsException> {
    @Override
    public Response toResponse(InsufficientFundsException e) {
        return Response.status(Response.Status.CONFLICT)
            .entity(Map.of("error", "INSUFFICIENT_FUNDS",
                           "balance", e.getBalance(), "requested", e.getAmount()))
            .build();
    }
}
```

**@ApplicationException mapping rules**:
- `rollback = true` → `@Transactional(rollbackOn = MyException.class)` on the throwing method
- `rollback = false` (default) → `@Transactional(dontRollbackOn = MyException.class)`, or simply remove annotation
- `inherited = true` (default) → apply to all methods that throw the exception or subclasses
- The exception class itself: DELETE `@ApplicationException` — no CDI/Quarkus equivalent

## ArC-Specific Caveats

### What Works Fine

`@PostConstruct`, `@PreDestroy` (normal scopes), `@Inject` (field/constructor/setter), `@Produces`/`@Disposes`, `@Interceptor`/`@InterceptorBinding` (use `@Priority` for ordering — beans.xml `<alternatives>`/`<interceptors>` NOT read), `@Observes`/`@ObservesAsync`, `@Alternative` + `@Priority`, `@Vetoed`, `@Transactional` — all fully supported.

### What Requires Attention

| Feature | Status | Workaround |
|---|---|---|
| `@PreDestroy` on `@Dependent` | ⚠ Not guaranteed | Use `@ApplicationScoped` or explicit cleanup |
| `@LocalBean` | ❌ Not needed | Remove — all CDI beans are "local" |
| Remote interfaces (`@Remote`) | ❌ Not supported | Redesign to REST (`quarkus-rest`) or gRPC (`quarkus-grpc`) |
| `@Lock(READ/WRITE)` | ❌ No equivalent | Use `ReadWriteLock` manually (Example 3) |
| `@AccessTimeout` | ❌ No equivalent | Implement timeout logic manually or reactive `@Timeout` |
| `@ConcurrencyManagement(CONTAINER)` | ❌ No equivalent | CDI beans not thread-safe by default — add manual synchronization |
| `@DependsOn` | ❌ No equivalent | Use `@Observes StartupEvent` with `@Priority` for ordering |
| Bean-managed transactions (BMT) | ⚠ Possible | Inject `UserTransaction` or use `QuarkusTransaction.begin()/commit()` |
| `@RunAs` | ❌ No direct equivalent | Use `SecurityIdentityAugmentor` — see security reference |
| `@Remove` | ❌ No equivalent | Remove annotation — method becomes regular business method |

### Constructor Injection & Native Image

Constructor injection is preferred (testability, immutability, reliable native-image handling). Normal-scoped beans (`@ApplicationScoped`, `@RequestScoped`, `@SessionScoped`) **require a no-arg constructor** (package-private is fine) for ArC proxy generation. `@Dependent` beans do not (no proxy). For Kotlin, use the `kotlin-noarg` compiler plugin for normal scopes.

### Self-Invocation Gotcha

In EJB the container intercepts ALL calls including self-calls. In CDI, interceptors (`@Transactional`, custom) only apply through the **proxy** (external call):

```java
@ApplicationScoped
@Transactional
public class OrderService {
    public void createOrder(OrderRequest req) { doCreate(req); }  // proxied — TX applied

    @Transactional(TxType.REQUIRES_NEW)
    public void doCreate(OrderRequest req) {
        // WARNING: when called via self-invocation from createOrder(),
        // REQUIRES_NEW is IGNORED — uses the existing transaction
    }
}
```

**Solutions**: (1) extract the method to a separate bean (preferred), (2) `Arc.container().instance(MyBean.class).get().method()`, or (3) inject `Instance<MyBean>` and call through it.

## @ConversationScoped → @SessionScoped Fallback

Quarkus ArC does NOT support `@ConversationScoped`. Use `@SessionScoped` as fallback: bean must implement `Serializable`, all injected fields Serializable. State is isolated per HTTP session. For fine-grained conversation control use `@ViewScoped` (JSF) or explicit client-side state.

## @Stateful EJB — Non-HTTP Client Warning

`@Stateful` → `@SessionScoped` works for HTTP-triggered flows only. For remote callers, scheduled jobs, or background processes: use `@ApplicationScoped` + explicit state map, or a request-scoped correlation ID. Do NOT use `@SessionScoped` for non-HTTP scenarios.
