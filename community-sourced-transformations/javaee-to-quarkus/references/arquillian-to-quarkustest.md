# Arquillian → @QuarkusTest Reference

> Reference for Phase 4 Step 16: Test framework migration.
> See also: https://quarkus.io/guides/getting-started-testing

Sections: Dependencies · Annotation Mapping · (1) Integration Test · (2) REST API Test · (3) @TestTransaction · (4) @InjectMock · (5) @TestProfile · (6) TestContainers · (7) Native Mode · (8) Continuous Testing · (9) MockitoExtension Strict Stubbing · (10) Assertion Arg Order · (11) assertThrows Isolation · (12) @ConfigProperty in Unit Tests · (13) @TestSecurity Class-Level · (14) REQUIRES_NEW Isolation · (15) H2 Test Datasource · (16) Uni Mock NPE · (17) Effectively-Final in assertThrows · (18) Mixed @Test(expected)+try/catch · (19) Qute Mock Chain · (20) Dual root/rest Path · (21) Empty @BeforeEach · (22) *IT→*Test Rename · (23) All-@Inject Audit · (24) @BeforeEach Config Audit · (25) Generic Mock Type · Migration Checklist · JUnit 4→5 Quick Reference

---

## Dependencies

```xml
<!-- REMOVE -->
<dependency><groupId>org.jboss.arquillian</groupId><artifactId>arquillian-bom</artifactId></dependency>
<dependency><groupId>org.jboss.arquillian.junit</groupId><artifactId>arquillian-junit-container</artifactId></dependency>
<dependency><groupId>org.jboss.arquillian.container</groupId><artifactId>arquillian-weld-ee-embedded-1.1</artifactId></dependency>
<dependency><groupId>org.jboss.shrinkwrap</groupId><artifactId>shrinkwrap-api</artifactId></dependency>
<dependency><groupId>org.jboss.shrinkwrap.resolver</groupId><artifactId>shrinkwrap-resolver-impl-maven</artifactId></dependency>

<!-- ADD (all BOM-managed — no explicit <version>) -->
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-junit</artifactId><scope>test</scope></dependency><!-- use quarkus-junit, NOT quarkus-junit5 (deprecated since 3.31) -->
<dependency><groupId>io.rest-assured</groupId><artifactId>rest-assured</artifactId><scope>test</scope></dependency>
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-junit-mockito</artifactId><scope>test</scope></dependency>
```

Also delete: `arquillian.xml`, `test-ds.xml`, Arquillian container adapter JARs.

**Mockito BOM status (Quarkus 3.33.2+):** Both `mockito-core` and `mockito-junit-jupiter` are BOM-managed. Omit explicit `<version>` — specifying one causes downgrade conflicts. Verify: `mvn dependency:resolve | grep mockito`.

## Annotation Mapping

| Arquillian | Quarkus | Notes |
|---|---|---|
| `@RunWith(Arquillian.class)` | `@QuarkusTest` | Class-level; JUnit 5 only |
| `@Deployment` + `ShrinkWrap` | REMOVE entirely | Quarkus manages full app lifecycle |
| `@Inject` in test | `@Inject` in test | Works natively |
| `@ArquillianResource URL` | `@TestHTTPResource` | Injects test server URL |
| `@RunAsClient` | Default behavior | `@QuarkusTest` runs as client by default |
| `@InSequence(n)` | `@TestMethodOrder(OrderAnnotation.class)` + `@Order(n)` | JUnit 5 ordering |
| `arquillian.xml` config | `application.properties` (`%test` profile) | Test-specific config |

---

## 1. Typical Integration Test

```java
// BEFORE (Arquillian): @RunWith(Arquillian.class) + @Deployment ShrinkWrap.create(...) + @Inject
// AFTER (Quarkus):
@QuarkusTest
public class OrderServiceTest {  // renamed from *IT — see Section 22
    @Inject
    OrderService orderService;

    @Test
    public void testCreateOrder() {
        Order order = orderService.createOrder(new OrderRequest("item-1", 2));
        assertNotNull(order);
        assertEquals("PENDING", order.getStatus());
    }
}
```

**Key changes**: `@RunWith(Arquillian.class)` → `@QuarkusTest`; delete `@Deployment`; `org.junit.Test` → `org.junit.jupiter.api.Test`; `Assert.*` → `Assertions.*`; rename `*IT.java` → `*Test.java`.

## 2. REST API Test (REST Assured)

```java
@QuarkusTest
public class OrderResourceTest {
    @Test
    public void testGetOrder() {
        given().when().get("/api/orders/1")
            .then().statusCode(200).body("id", equalTo(1));
    }
}
```

No `@Deployment`, no `@ArquillianResource URL`, no JAX-RS Client boilerplate. When `quarkus.http.root-path` is set, REST Assured base URL auto-includes it — do NOT add root-path manually (see Section 20).

## 3. Database Test — @TestTransaction

```java
@QuarkusTest
@TestTransaction  // each test's TX rolled back after
public class OrderRepositoryTest {
    @Inject EntityManager em;

    @BeforeEach
    public void setupData() { em.persist(new Order("order-1", "ACTIVE")); em.flush(); }

    @Test
    public void testFindByStatus() {
        List<Order> orders = em.createQuery(
            "SELECT o FROM Order o WHERE o.status = :status", Order.class)
            .setParameter("status", "ACTIVE").getResultList();
        assertEquals(1, orders.size());
    }
}
```

Alternative: static `import.sql` in `src/test/resources/` with `schema-management.strategy=drop-and-create`.

## 4. Mocking — @InjectMock

```java
@QuarkusTest
public class OrderResourceTest {
    @InjectMock PaymentService paymentService;

    @Test
    public void testPaymentFailure() {
        when(paymentService.charge(any())).thenThrow(new PaymentException("Declined"));
        given().contentType("application/json").body("""{"itemId":"item-1","quantity":1}""")
            .when().post("/api/orders").then().statusCode(402);
        verify(paymentService).charge(any());
    }
}
```

`@InjectMock` replaces the real CDI bean class-wide (mocks reset between tests). Spy: `@InjectSpy`.

## 5. Test Profiles — @TestProfile

```java
public class MockExternalServiceProfile implements QuarkusTestProfile {
    @Override public Map<String,String> getConfigOverrides() {
        return Map.of("external.service.url", "http://localhost:8089/mock");
    }
}
@QuarkusTest @TestProfile(MockExternalServiceProfile.class)
public class ExternalIntegrationTest { ... }
```

Alternatively override `getEnabledAlternatives()` to swap in `@Alternative @Priority` mock beans.

## 6. TestContainers Integration

**Dev Services (zero-config, preferred):**
```properties
# src/test/resources/application.properties
quarkus.datasource.db-kind=postgresql
quarkus.datasource.devservices.enabled=true
quarkus.hibernate-orm.schema-management.strategy=drop-and-create
```
Quarkus auto-starts containers for DBs and brokers with Dev Services support.

**Explicit QuarkusTestResource:**
```java
public class PostgresResource implements QuarkusTestResourceLifecycleManager {
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");
    @Override public Map<String,String> start() {
        postgres.start();
        return Map.of("quarkus.datasource.jdbc.url", postgres.getJdbcUrl());
    }
    @Override public void stop() { postgres.stop(); }
}
// Apply: @QuarkusTest @QuarkusTestResource(PostgresResource.class)
```

## 7. Native Mode Testing — @QuarkusIntegrationTest

```java
@QuarkusIntegrationTest
public class NativeOrderResourceIT {
    @Test public void testHealth() { given().when().get("/q/health").then().statusCode(200); }
}
```

`@Inject` does NOT work (runs outside app JVM); only HTTP testing. File naming: `*IT.java` + maven-failsafe-plugin. Run with `./mvnw verify -Pnative`.

## 8. Continuous Testing

```bash
./mvnw quarkus:dev   # 'r' re-run, 'o' output, 'p' pause
```
Only tests affected by changed source re-run, in the dev-mode JVM — replaces Arquillian's deploy-test-undeploy cycle.

## 9. MockitoExtension Strict Stubbing (JUnit 4 → 5)

JUnit 4 `MockitoJUnitRunner` = **lenient** (unused stubs ignored). JUnit 5 `@ExtendWith(MockitoExtension.class)` = **strict** (unused stubs throw `UnnecessaryStubbingException`).

### Strict-Stubbing Decision Table

| Return type of stubbed method | Situation | Action |
|---|---|---|
| **void** (`doNothing().when(m).v()`) | Always | **DELETE unconditionally** — void `doNothing()` stubs cause the exception and have no effect |
| **Uni<Void>** | Needed by some tests | `lenient().when(m.method()).thenReturn(Uni.createFrom().voidItem())` |
| Object-returning | Used by ALL tests | Plain `when()` in `@BeforeEach` — correct |
| Object-returning | Used by SOME tests | `lenient().when()` in `@BeforeEach` |
| Object-returning | Used by ONE test | Move stub into that test method |
| Any | ALL stubs target removed EJB/JMS layer | Delete entire `@BeforeEach` |
| Any | Stub unreachable (guard throws first) | Remove stub from that test |

**Partial-use definition**: a `@BeforeEach` stub is "used-by-all" ONLY if EVERY test reaches the stubbed call. If ANY test completes without invoking it → partial-use → `lenient().when()`.

**Instance<T> mocking** (common after EJB→CDI):
```java
lenient().when(instance.iterator()).thenAnswer(inv -> List.of(myImpl).iterator());
lenient().when(instance.get()).thenReturn(myImpl);
```

**Common cause — validation guard**: test stubs a collaborator, but code throws before reaching it:
```java
// WRONG: when(inventoryService.reserve(any()))... is unreachable when input null
// FIXED: drop the stub, keep only assertThrows(ValidationException.class, () -> service.createOrder(null));
```

**Common cause — removed EJB/JMS layer**: if ALL `@BeforeEach` stubs targeted the removed layer, delete the whole `@BeforeEach`.

**@Transactional on @BeforeEach/@AfterEach is silently ignored** in `@QuarkusTest`. Use `@Inject UserTransaction` with explicit `begin()`/`commit()` for transactional setup.

**Anti-pattern**: do NOT apply class-wide `@MockitoSettings(strictness = Strictness.LENIENT)`. Fix each stub.

**Checklist**: (1) delete `@BeforeEach` stubs for removed EJB/JMS; (2) for error-path tests, remove stubs set up after the guard; (3) `./mvnw test` → no `UnnecessaryStubbingException`.

## 10. JUnit 4→5 Assertion Argument Order

JUnit 4 puts message FIRST; JUnit 5 moves it LAST. Both compile silently when args share a type → silent bug.

| JUnit 4 (message FIRST) | JUnit 5 (message LAST) |
|---|---|
| `assertEquals("msg", exp, act)` | `assertEquals(exp, act, "msg")` |
| `assertNotNull("msg", obj)` | `assertNotNull(obj, "msg")` |
| `assertNull("msg", obj)` | `assertNull(obj, "msg")` |
| `assertTrue("msg", cond)` | `assertTrue(cond, "msg")` |
| `assertFalse("msg", cond)` | `assertFalse(cond, "msg")` |
| `assertSame("msg", exp, act)` | `assertSame(exp, act, "msg")` |
| `assertNotSame("msg", exp, act)` | `assertNotSame(exp, act, "msg")` |

**⚠️ 2-arg exemption**: 2-arg overloads (no String message) are identical in JUnit 4/5 — do NOT reorder. Only 3-arg calls change.

```bash
# Detect message-first (string literal as first arg)
grep -rn --include='*.java' 'assertEquals("\|assertNotNull("\|assertNull("\|assertTrue("\|assertFalse("\|assertSame("\|assertNotSame("' src/test/
```

## 11. assertThrows Lambda Isolation

Lambda body must contain ONLY the single expected-throwing call; extract setup before it.
```java
// WRONG: assertThrows(X.class, () -> { when(mock.validate(any())).thenReturn(false); service.process(in); });
// CORRECT:
when(mock.validate(any())).thenReturn(false);
assertThrows(ValidationException.class, () -> service.process(input));
```
**Capture for property assertions**: `SomeException ex = assertThrows(SomeException.class, () -> service.process(bad)); assertEquals("INVALID_FORMAT", ex.getErrorCode());`

**Dead-code removal on `@Test(expected=...)` → assertThrows conversion**: remove unreachable stubs before the throw and dead post-throw statements.

**MINIMIZE CHANGES for try/catch/fail**: do NOT convert when the catch block asserts on **multiple** exception fields — the conversion loses clarity. Only convert when the sole intent is verifying a throw.

## 12. CDI @ConfigProperty in Unit Tests

`@ConfigProperty` fields are `null`/`0` in plain unit tests (no CDI container); `defaultValue` is applied only by the MicroProfile runtime. Fixes: (A) reflective field injection in `@BeforeEach`; (B) convert to `@QuarkusTest`; (C) field initializer matching `defaultValue` (CDI overwrites it, plain `new` keeps fallback).
```java
@BeforeEach void setup() throws Exception {
    Field f = MyService.class.getDeclaredField("timeout"); f.setAccessible(true); f.set(service, 30);
}
```

## 13. @TestSecurity CLASS-level Placement

If ANY `@BeforeEach` invokes a `@RolesAllowed`-protected method (e.g. data setup via service layer), `@TestSecurity` MUST be at CLASS level — method-level does not cover `@BeforeEach`:
```java
@QuarkusTest
@TestSecurity(user = "admin", roles = "admin")   // covers @BeforeEach
public class OrderServiceTest {
    @BeforeEach void setup() { orderService.createOrder(...); }  // runs with admin identity
}
```
Method-level `@TestSecurity` can still override for specific tests.

## 14. REQUIRES_NEW Data Isolation with UserTransaction

```java
@Inject UserTransaction utx; @Inject EntityManager em;

@Test void testBatchCreatesAuditInNewTx() throws Exception {
    utx.begin();
    batchService.processBatch(items);  // internally REQUIRES_NEW for audit
    utx.rollback();                    // roll back outer — audit should survive
    utx.begin();
    long n = em.createQuery("SELECT COUNT(a) FROM AuditLog a", Long.class).getSingleResult();
    utx.commit();
    assertTrue(n > 0, "Audit in REQUIRES_NEW survives outer rollback");
}
```

## 15. H2 Test Datasource Override Template

If main `application.properties` sets an explicit `quarkus.hibernate-orm.dialect`, the test override MUST set BOTH `db-kind=h2` AND `dialect=H2Dialect` — changing `db-kind` alone does not cancel an explicit dialect.
```properties
%test.quarkus.datasource.db-kind=h2
%test.quarkus.datasource.jdbc.url=jdbc:h2:mem:test;DB_CLOSE_DELAY=-1
%test.quarkus.hibernate-orm.dialect=org.hibernate.dialect.H2Dialect
%test.quarkus.hibernate-orm.schema-management.strategy=drop-and-create
```
```xml
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-jdbc-h2</artifactId><scope>test</scope></dependency>
```
Use `schema-management.strategy` (not deprecated `database.generation`). If main config omits dialect (auto-detect), test override only needs `db-kind=h2`. If `import.sql` supplies test data, omit explicit ID columns (H2 IDENTITY collision).

## 16. Uni Mock Null-Return NPE

Mockito's default return for `Uni<T>` is `null` → `.subscribe()`/`.await()` NPEs. Always stub explicitly:
```java
when(service.findItem(any())).thenReturn(Uni.createFrom().item(new Item("test")));
when(service.deleteItem(any())).thenReturn(Uni.createFrom().voidItem());
when(service.findItem(any())).thenReturn(Uni.createFrom().failure(new NotFoundException()));
```

## 17. Effectively-Final Variables in assertThrows

Variables captured by the lambda must be effectively final. Field access (populated in `@BeforeEach`) is safe; local vars need single assignment:
```java
final String input = buildInput();
assertThrows(ValidationException.class, () -> service.process(input));
```

## 18. Mixed @Test(expected) + try/catch Idioms

Convert to `assertThrows` ONLY when the catch does simple type-checking:
```java
// BEFORE: @Test(expected=ValidationException.class) { try { service.process(bad); } catch (IOException e) { fail(...); } }
// AFTER:
@Test void testInvalid() { assertThrows(ValidationException.class, () -> service.process(badInput)); }
```
If the catch asserts on multiple exception properties, KEEP the try/catch (see Section 11 MINIMIZE CHANGES).

## 19. Qute Template Mock Chain

Mock the full fluent chain for injected `io.quarkus.qute.Template`:
```java
@InjectMock Template emailTemplate;
@BeforeEach void setup() {
    TemplateInstance instance = mock(TemplateInstance.class);
    when(emailTemplate.data(anyString(), any())).thenReturn(instance);
    when(instance.data(anyString(), any())).thenReturn(instance);
    when(instance.render()).thenReturn("<html>rendered</html>");
}
```

## 20. REST Assured Dual root/rest Path

With `quarkus.http.root-path=/app` and `quarkus.rest.path=/api`: REST Assured `basePath=/app`; request paths include the REST path but NOT root-path (`.get("/api/orders")` → hits `/app/api/orders`). Confirmed by [Quarkus #28001](https://github.com/quarkusio/quarkus/issues/28001). Non-application paths (`/q/health`): use `@TestHTTPResource("/q/health")` URL injection to bypass basePath.

## 21. Empty @BeforeEach Cleanup

After removing all stubs from a `@BeforeEach` (strict-stubbing fixes / removed EJB layer), delete the empty method — do not leave `@BeforeEach void setup() {}`.

## 22. *IT.java → *Test.java Mandatory Rename

| Current | Target exists? | Action |
|---|---|---|
| `*IT.java` | no `*Test.java` for same class | rename → `*Test.java` |
| `*IT.java` | `*Test.java` exists | rename → `*IntegrationTest.java` |
| `*Test.java` | — | skip |

Maven Surefire excludes `*IT.java` — migrated tests would be silently never run by `mvn test`. **Exception**: keep `*IT.java` ONLY for `@QuarkusIntegrationTest` (native, Failsafe).

## 23. All-@Inject-Field Audit

After migrating to `@ExtendWith(MockitoExtension.class)` / `@QuarkusTest`, verify EVERY `@Inject` field in the production class has a corresponding `@Mock`/`@InjectMock`/`@Inject`/reflective injection. Missing → NPE where production code uses the field. Common miss: fields added during EJB→CDI (e.g. new `@Inject EntityManager`) with no mock in existing tests.

## 24. @BeforeEach Config Injection Audit

When `@BeforeEach` injects config/state, check it does not invalidate negative-path assertions. Fix: move config injection into the tests that need it, or set `null` explicitly in the negative test.

## 25. Generic Mock Type Resolution (Emitter<T>)

For generic types like `Emitter<T>`, Mockito resolves by **field name** (type erasure). Multiple `@Mock Emitter<?>` fields must match the production field names exactly, else silent null injection:
```java
// production: @Channel("orders-out") Emitter<String> orderEmitter; @Channel("audit-out") Emitter<String> auditEmitter;
@Mock Emitter<String> orderEmitter;   // names MUST match
@Mock Emitter<String> auditEmitter;
@InjectMocks OrderService service;
```

---

## Migration Checklist

| Step | Action | Verify |
|---|---|---|
| 1 | Remove Arquillian deps | `grep -rn "arquillian\|shrinkwrap" pom.xml` empty |
| 2 | Add `quarkus-junit` + `rest-assured` (BOM-managed) | Compile passes |
| 3 | `@RunWith(Arquillian.class)` → `@QuarkusTest` | `grep -rn "Arquillian" src/test/` empty |
| 4 | Delete ALL `@Deployment` methods | `grep -rn "ShrinkWrap\|@Deployment" src/test/` empty |
| 5 | `@ArquillianResource URL` → `@TestHTTPResource` | no URL management |
| 6 | JAX-RS Client → REST Assured | cleaner tests |
| 7 | JUnit 4→5 assertions (swap message — 3-arg ONLY) | imports updated |
| 8 | `src/test/resources/application.properties` H2 config | test DS configured |
| 9 | Delete `arquillian.xml`, `test-ds.xml` | removed |
| 10 | Remove unnecessary stubs (strict mode) | no `UnnecessaryStubbingException` |
| 11 | All-@Inject-field audit | every @Inject has a test mock |
| 12 | Rename `*IT.java` → `*Test.java` | tests appear in surefire |
| 13 | `./mvnw clean test` | ALL pass |

## JUnit 4 → 5 Quick Reference

| JUnit 4 | JUnit 5 | Notes |
|---|---|---|
| `@RunWith(...)` | `@ExtendWith(...)` | not needed with `@QuarkusTest` |
| `@RunWith(MockitoJUnitRunner.class)` | `@ExtendWith(MockitoExtension.class)` | strict stubbing — Section 9 |
| `@Test` (org.junit) | `@Test` (org.junit.jupiter.api) | different import |
| `@Before` / `@After` | `@BeforeEach` / `@AfterEach` | |
| `@BeforeClass` / `@AfterClass` | `@BeforeAll` / `@AfterAll` | |
| `@Ignore` | `@Disabled` | do NOT add as migration shortcut — fix the test |
| `@Test(expected=X.class)` | `assertThrows(X.class, () -> {...})` | lambda = only the throwing call (Section 11) |
| `Assert.assertEquals(msg, exp, act)` | `assertEquals(exp, act, msg)` | message LAST — 3-arg ONLY |
| `@Rule ExpectedException` | `assertThrows(...)` | |
| `@Category(...)` | `@Tag(...)` | |
