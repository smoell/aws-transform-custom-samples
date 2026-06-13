# EJB → CDI (ArC) Mapping Reference

> Reference for Phase 2: Core Migration — EJB to Quarkus CDI bean conversion.

## EJB Stereotype Mapping

| EJB Annotation | Quarkus CDI Equivalent | Notes |
|---|---|---|
| `@Stateless` | `@ApplicationScoped` + `@Transactional` | Add `@Transactional` ONLY if bean performs persistence operations |
| `@Stateful` | `@SessionScoped` | Evaluate if stateful pattern is still needed; bean MUST be `Serializable` |
| `@Singleton` (javax.ejb) | `@ApplicationScoped` | Do NOT use `jakarta.inject.Singleton` (not a normal scope, cannot be intercepted) |
| `@Singleton` + `@Startup` | `@ApplicationScoped` + `void onStart(@Observes StartupEvent ev)` | Startup observer replaces `@PostConstruct` eager init |
| `@Singleton` + `@Lock(READ)` | `@ApplicationScoped` + `ReadWriteLock` | Manual concurrency control required (see examples below) |
| `@MessageDriven` | See `references/jms-to-smallrye.md` | Defer to messaging migration reference |
| `@LocalBean` | REMOVE | All CDI beans are "local" — no equivalent needed |
| `@Local` | REMOVE | No local/remote distinction in CDI |
| `@Remote` | REMOVE — redesign to REST/gRPC | No remote interface support in Quarkus |

### @Stateful Caveats

- **Serialization**: `@SessionScoped` beans MUST implement `Serializable` for HTTP session serialization.
- **ArC proxy requirement**: normal-scoped beans (`@SessionScoped`) require a no-arg constructor (can be package-private).
- **Cloud-native consideration**: stateful beans are problematic for horizontal scaling. Prefer externalizing state to Redis/database and using `@ApplicationScoped` stateless beans.
- **@PrePassivate / @PostActivate**: DELETE these lifecycle methods entirely — no Quarkus equivalent exists for stateful passivation/activation.

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
- **Self-invocation**: unlike EJB where container intercepts all calls, `@Transactional` only works when called through the CDI proxy (external call). Self-calls (`this.method()`) bypass the interceptor. Use `Arc.container().instance(MyBean.class).get().method()` for self-invocation if needed.

## Injection Migration

| EJB Pattern | Quarkus CDI Equivalent | Notes |
|---|---|---|
| `@EJB private MyService svc;` | `@Inject MyService svc;` | Simple replacement |
| `@EJB(beanName="orderSvc")` | `@Inject @Named("orderSvc") MyService svc;` | Add `@Named("orderSvc")` on the target bean class |
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
    private final PaymentService paymentService;

    OrderService() {} // package-private no-arg constructor for ArC proxy

    @Inject
    public OrderService(InventoryService inventoryService, PaymentService paymentService) {
        this.inventoryService = inventoryService;
        this.paymentService = paymentService;
    }
}
```

**Note**: If the bean has only ONE constructor, `@Inject` on the constructor is optional — ArC infers it.

## Worked Examples

### Example 1: Simple @Stateless Service Bean

```java
// BEFORE (JavaEE)
import javax.ejb.Stateless;
import javax.ejb.EJB;
import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;

@Stateless
public class OrderService {
    @PersistenceContext
    private EntityManager em;

    @EJB
    private InventoryService inventoryService;

    public Order createOrder(OrderRequest request) {
        inventoryService.reserve(request.getItemId(), request.getQuantity());
        Order order = new Order(request);
        em.persist(order);
        return order;
    }

    public Order findById(Long id) {
        return em.find(Order.class, id);
    }
}
```

```java
// AFTER (Quarkus)
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

@ApplicationScoped
@Transactional  // class-level — all methods run in a transaction
public class OrderService {
    @Inject
    EntityManager em;

    private final InventoryService inventoryService;

    OrderService() {}

    @Inject
    public OrderService(InventoryService inventoryService) {
        this.inventoryService = inventoryService;
    }

    public Order createOrder(OrderRequest request) {
        inventoryService.reserve(request.getItemId(), request.getQuantity());
        Order order = new Order(request);
        em.persist(order);
        return order;
    }

    public Order findById(Long id) {
        return em.find(Order.class, id);
    }
}
```

### Example 2: @Stateless with Method-Level @TransactionAttribute

```java
// BEFORE (JavaEE)
import javax.ejb.Stateless;
import javax.ejb.TransactionAttribute;
import javax.ejb.TransactionAttributeType;

@Stateless
public class PaymentService {
    @EJB
    private AuditService auditService;

    // Default REQUIRED — joins existing transaction
    public void processPayment(Payment payment) {
        // payment logic
    }

    @TransactionAttribute(TransactionAttributeType.REQUIRES_NEW)
    public void recordAudit(String action, String details) {
        // always in its own transaction (survives caller rollback)
        auditService.log(action, details);
    }

    @TransactionAttribute(TransactionAttributeType.NOT_SUPPORTED)
    public PaymentStatus checkExternalStatus(String paymentRef) {
        // no transaction needed — external HTTP call
        return externalGateway.getStatus(paymentRef);
    }
}
```

```java
// AFTER (Quarkus)
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.transaction.Transactional.TxType;

@ApplicationScoped
@Transactional  // default REQUIRED for all methods
public class PaymentService {
    private final AuditService auditService;

    PaymentService() {}

    @Inject
    public PaymentService(AuditService auditService) {
        this.auditService = auditService;
    }

    // Inherits class-level @Transactional (REQUIRED)
    public void processPayment(Payment payment) {
        // payment logic
    }

    @Transactional(TxType.REQUIRES_NEW)
    public void recordAudit(String action, String details) {
        // always in its own transaction
        auditService.log(action, details);
    }

    @Transactional(TxType.NOT_SUPPORTED)
    public PaymentStatus checkExternalStatus(String paymentRef) {
        // suspends any active transaction
        return externalGateway.getStatus(paymentRef);
    }
}
```

### Example 3: @Singleton with @Lock(READ/WRITE) → ReadWriteLock

```java
// BEFORE (JavaEE)
import javax.ejb.Singleton;
import javax.ejb.Startup;
import javax.ejb.Lock;
import javax.ejb.LockType;
import javax.ejb.ConcurrencyManagement;
import javax.ejb.ConcurrencyManagementType;
import javax.annotation.PostConstruct;

@Singleton
@Startup
@ConcurrencyManagement(ConcurrencyManagementType.CONTAINER)
public class ConfigCache {
    private Map<String, String> cache;

    @PostConstruct
    public void init() {
        cache = loadFromDatabase();
    }

    @Lock(LockType.READ)
    public String getValue(String key) {
        return cache.get(key);
    }

    @Lock(LockType.WRITE)
    public void setValue(String key, String value) {
        cache.put(key, value);
    }

    @Lock(LockType.WRITE)
    public void refresh() {
        cache = loadFromDatabase();
    }
}
```

```java
// AFTER (Quarkus)
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import io.quarkus.runtime.StartupEvent;
import java.util.Map;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

@ApplicationScoped
public class ConfigCache {
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private Map<String, String> cache;

    void onStart(@Observes StartupEvent ev) {
        cache = loadFromDatabase();
    }

    public String getValue(String key) {
        lock.readLock().lock();
        try {
            return cache.get(key);
        } finally {
            lock.readLock().unlock();
        }
    }

    public void setValue(String key, String value) {
        lock.writeLock().lock();
        try {
            cache.put(key, value);
        } finally {
            lock.writeLock().unlock();
        }
    }

    public void refresh() {
        lock.writeLock().lock();
        try {
            cache = loadFromDatabase();
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

**Alternative** — for simple cases without `@Lock` (ConcurrencyManagement.BEAN or no concurrency needs):

```java
// If @Singleton had no @Lock annotations (bean-managed or single-threaded access):
@ApplicationScoped
public class SimpleCache {
    void onStart(@Observes StartupEvent ev) {
        // init logic
    }
    // no lock needed — @ApplicationScoped beans are not thread-safe by default
    // (same as ConcurrencyManagementType.BEAN)
}
```

### Example 4: @Asynchronous Method → CompletionStage/Uni

```java
// BEFORE (JavaEE)
import javax.ejb.Stateless;
import javax.ejb.Asynchronous;
import java.util.concurrent.Future;
import javax.ejb.AsyncResult;

@Stateless
public class NotificationService {
    @Asynchronous
    public Future<String> sendEmail(String to, String subject, String body) {
        // long-running email send operation
        String messageId = emailGateway.send(to, subject, body);
        return new AsyncResult<>(messageId);
    }

    @Asynchronous
    public void sendSmsFireAndForget(String phone, String message) {
        // fire-and-forget — caller doesn't wait
        smsGateway.send(phone, message);
    }
}
```

```java
// AFTER (Quarkus — Option A: Mutiny Uni)
import jakarta.enterprise.context.ApplicationScoped;
import io.smallrye.mutiny.Uni;
import io.smallrye.mutiny.infrastructure.Infrastructure;

@ApplicationScoped
public class NotificationService {
    public Uni<String> sendEmail(String to, String subject, String body) {
        return Uni.createFrom().item(() -> {
            // long-running email send operation
            return emailGateway.send(to, subject, body);
        }).runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }

    public Uni<Void> sendSmsFireAndForget(String phone, String message) {
        return Uni.createFrom().voidItem()
            .onItem().invoke(() -> smsGateway.send(phone, message))
            .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }
}
```

```java
// AFTER (Quarkus — Option B: CompletionStage with ManagedExecutor)
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.context.ManagedExecutor;
import java.util.concurrent.CompletionStage;

@ApplicationScoped
public class NotificationService {
    @Inject
    ManagedExecutor executor;

    public CompletionStage<String> sendEmail(String to, String subject, String body) {
        return executor.supplyAsync(() -> {
            return emailGateway.send(to, subject, body);
        });
    }

    public CompletionStage<Void> sendSmsFireAndForget(String phone, String message) {
        return executor.runAsync(() -> {
            smsGateway.send(phone, message);
        });
    }
}
```

**CRITICAL**: If the async method accesses `@RequestScoped` beans, annotate the method with `@ActivateRequestContext` — ArC does not propagate request context to worker threads automatically.

### Example 5: EJB Timer (@Schedule) → Quarkus @Scheduled

```java
// BEFORE (JavaEE)
import javax.ejb.Singleton;
import javax.ejb.Schedule;
import javax.ejb.Schedules;
import javax.ejb.Timer;

@Singleton
public class ReportScheduler {
    @Schedule(hour = "2", minute = "0", persistent = false)
    public void dailyReport(Timer timer) {
        generateDailyReport();
    }

    @Schedule(hour = "*", minute = "*/15", persistent = false)
    public void healthCheck(Timer timer) {
        checkSystemHealth();
    }

    @Schedules({
        @Schedule(dayOfWeek = "Mon", hour = "9"),
        @Schedule(dayOfWeek = "Fri", hour = "17")
    })
    public void weeklyBoundaryReport(Timer timer) {
        generateWeeklyReport();
    }
}
```

```java
// AFTER (Quarkus)
import jakarta.enterprise.context.ApplicationScoped;
import io.quarkus.scheduler.Scheduled;

@ApplicationScoped
public class ReportScheduler {
    @Scheduled(cron = "0 0 2 * * ?")  // daily at 2:00 AM
    public void dailyReport() {
        generateDailyReport();
    }

    @Scheduled(every = "15m")  // every 15 minutes
    public void healthCheck() {
        checkSystemHealth();
    }

    // @Schedules with multiple crons → separate methods
    @Scheduled(cron = "0 0 9 ? * MON")  // Monday at 9:00 AM
    public void weeklyStartReport() {
        generateWeeklyReport();
    }

    @Scheduled(cron = "0 0 17 ? * FRI")  // Friday at 5:00 PM
    public void weeklyEndReport() {
        generateWeeklyReport();
    }
}
```

**Notes**:
- Remove `Timer` parameter from method signature (Quarkus does not pass timer info).
- Remove `persistent = false` (Quarkus scheduled tasks are non-persistent by default).
- `@Schedules` (plural) → split into separate `@Scheduled` methods delegating to shared logic.
- Quarkus cron format: `second minute hour dayOfMonth month dayOfWeek` (Quartz-style).
- Add `quarkus-scheduler` extension to pom.xml.

### Example 6: @ApplicationException → ExceptionMapper

```java
// BEFORE (JavaEE)
import javax.ejb.ApplicationException;

@ApplicationException(rollback = true)
public class InsufficientFundsException extends Exception {
    private final BigDecimal balance;
    private final BigDecimal amount;

    public InsufficientFundsException(BigDecimal balance, BigDecimal amount) {
        super("Insufficient funds: balance=" + balance + ", requested=" + amount);
        this.balance = balance;
        this.amount = amount;
    }

    public BigDecimal getBalance() { return balance; }
    public BigDecimal getAmount() { return amount; }
}

// EJB that throws it:
@Stateless
public class AccountService {
    public void withdraw(Long accountId, BigDecimal amount) throws InsufficientFundsException {
        Account account = em.find(Account.class, accountId);
        if (account.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException(account.getBalance(), amount);
        }
        account.debit(amount);
    }
}
```

```java
// AFTER (Quarkus) — Exception class (remove @ApplicationException)
public class InsufficientFundsException extends Exception {
    private final BigDecimal balance;
    private final BigDecimal amount;

    public InsufficientFundsException(BigDecimal balance, BigDecimal amount) {
        super("Insufficient funds: balance=" + balance + ", requested=" + amount);
        this.balance = balance;
        this.amount = amount;
    }

    public BigDecimal getBalance() { return balance; }
    public BigDecimal getAmount() { return amount; }
}

// Service — rollback=true equivalent: declare in @Transactional
@ApplicationScoped
public class AccountService {
    @Inject
    EntityManager em;

    @Transactional(rollbackOn = InsufficientFundsException.class)
    public void withdraw(Long accountId, BigDecimal amount) throws InsufficientFundsException {
        Account account = em.find(Account.class, accountId);
        if (account.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException(account.getBalance(), amount);
        }
        account.debit(amount);
    }
}

// JAX-RS ExceptionMapper — maps exception to HTTP response
import jakarta.ws.rs.core.Response;
import jakarta.ws.rs.ext.ExceptionMapper;
import jakarta.ws.rs.ext.Provider;

@Provider
public class InsufficientFundsExceptionMapper implements ExceptionMapper<InsufficientFundsException> {
    @Override
    public Response toResponse(InsufficientFundsException e) {
        return Response.status(Response.Status.CONFLICT)
            .entity(Map.of(
                "error", "INSUFFICIENT_FUNDS",
                "balance", e.getBalance(),
                "requested", e.getAmount()
            ))
            .build();
    }
}
```

**@ApplicationException mapping rules**:
- `rollback = true` → `@Transactional(rollbackOn = MyException.class)` on the throwing method
- `rollback = false` (default) → `@Transactional(dontRollbackOn = MyException.class)` if parent would rollback, or simply remove annotation
- `inherited = true` (default) → apply `rollbackOn`/`dontRollbackOn` to all methods that throw the exception or subclasses
- The exception class itself: DELETE `@ApplicationException` annotation — it has no CDI/Quarkus equivalent

## ArC-Specific Caveats

### What Works Fine

| Feature | Status | Notes |
|---|---|---|
| `@PostConstruct` | ✅ Fully supported | Works on all scoped beans |
| `@PreDestroy` | ✅ Supported for normal scopes | Works for `@ApplicationScoped`, `@RequestScoped`, `@SessionScoped` |
| `@Inject` (field, constructor, setter) | ✅ Fully supported | Constructor injection preferred |
| `@Produces` / `@Disposes` | ✅ Fully supported | Producer methods/fields work as in full CDI |
| `@Interceptor` / `@InterceptorBinding` | ✅ Fully supported | Must use `@Priority` for ordering (no beans.xml) |
| `@Observes` / `@ObservesAsync` | ✅ Fully supported | Event system works as expected |
| `@Alternative` + `@Priority` | ✅ Fully supported | beans.xml `<alternatives>` NOT read — use `@Priority` |
| `@Vetoed` | ✅ Fully supported | Excludes bean from discovery |
| `@Transactional` | ✅ Fully supported | Interceptor-based, requires normal scope |

### What Requires Attention

| Feature | Status | Workaround |
|---|---|---|
| `@PreDestroy` on `@Dependent` | ⚠ Not guaranteed | Use `@ApplicationScoped` or explicit cleanup |
| `@LocalBean` | ❌ Not needed | Remove — all CDI beans are "local" |
| Remote interfaces (`@Remote`) | ❌ Not supported | Redesign to REST (`quarkus-rest`) or gRPC (`quarkus-grpc`) |
| `@Lock(READ/WRITE)` | ❌ No equivalent | Use `java.util.concurrent.locks.ReadWriteLock` manually |
| `@AccessTimeout` | ❌ No equivalent | Implement timeout logic manually or use `@Timeout` with reactive |
| `@ConcurrencyManagement(CONTAINER)` | ❌ No equivalent | CDI beans are not thread-safe by default — add manual synchronization |
| `@DependsOn` | ❌ No equivalent | Use `@Observes StartupEvent` with `@Priority` for ordering |
| Bean-managed transactions (BMT) | ⚠ Possible | Inject `UserTransaction` or use `QuarkusTransaction.begin()/commit()` |
| `@RunAs` | ❌ No direct equivalent | Use `SecurityIdentityAugmentor` — see security reference |

### Constructor Injection & Native Image

- **Constructor injection is preferred** for testability and immutability.
- For native image compilation: classes with constructor injection are reliably handled. Field injection works too, but constructor injection avoids potential reflection issues in edge cases.
- Normal-scoped beans (`@ApplicationScoped`, `@RequestScoped`, `@SessionScoped`) **require a no-arg constructor** (can be package-private) for ArC proxy generation:

```java
@ApplicationScoped
public class MyService {
    private final Dependency dep;

    MyService() {} // required for ArC proxy — package-private is fine

    @Inject
    public MyService(Dependency dep) {
        this.dep = dep;
    }
}
```

- `@Dependent` scoped beans do NOT require a no-arg constructor (no proxy generated).
- If using Kotlin: ArC handles Kotlin `data class` injection, but the no-arg constructor requirement still applies for normal scopes (use `kotlin-noarg` compiler plugin or `@io.quarkus.arc.Unremovable` to prevent removal).

### Self-Invocation Gotcha

In EJB, the container intercepts ALL method calls (including self-calls within the same bean). In CDI, interceptors (`@Transactional`, custom interceptors) only apply when the call goes through the **proxy** (external call).

```java
@ApplicationScoped
@Transactional
public class OrderService {
    public void createOrder(OrderRequest req) {
        // This DOES have @Transactional applied (external call through proxy)
        doCreate(req);
    }

    @Transactional(TxType.REQUIRES_NEW)
    public void doCreate(OrderRequest req) {
        // WARNING: if called from createOrder() above (self-call),
        // REQUIRES_NEW is IGNORED — uses the existing transaction
        // To force a new transaction, call through the proxy:
        // Arc.container().instance(OrderService.class).get().doCreate(req);
    }
}
```

**Solutions for self-invocation**:
1. Extract method to a separate bean (preferred — clean architecture)
2. Use `Arc.container().instance(MyBean.class).get().method()` (programmatic proxy access)
3. Inject `Instance<MyBean>` and call through it

## @ConversationScoped → @SessionScoped Fallback
Quarkus ArC does NOT support @ConversationScoped. Use @SessionScoped as fallback:
- Bean must implement `java.io.Serializable`
- All injected fields must be Serializable
- State collision risk: different users share session-scoped state only within their own HTTP session
- For fine-grained conversation control: use @ViewScoped (JSF) or explicit client-side state

## @Stateful EJB — Non-HTTP Client Warning
@Stateful → @SessionScoped works for HTTP-triggered flows only. For remote callers, scheduled jobs, or background processes: use @ApplicationScoped + explicit state map, or CDI @ConversationScoped emulation via a request-scoped correlation ID. Do NOT use @SessionScoped for non-HTTP scenarios.
