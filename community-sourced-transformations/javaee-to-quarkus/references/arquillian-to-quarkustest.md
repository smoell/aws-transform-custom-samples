# Arquillian → @QuarkusTest Reference

> Reference for Phase 4 Step 16: Test framework migration.
> See also: https://quarkus.io/guides/getting-started-testing

## Dependencies

Remove Arquillian dependencies and add Quarkus test dependencies:

```xml
<!-- REMOVE all of these -->
<dependency><groupId>org.jboss.arquillian</groupId><artifactId>arquillian-bom</artifactId></dependency>
<dependency><groupId>org.jboss.arquillian.junit</groupId><artifactId>arquillian-junit-container</artifactId></dependency>
<dependency><groupId>org.jboss.arquillian.container</groupId><artifactId>arquillian-weld-ee-embedded-1.1</artifactId></dependency>
<dependency><groupId>org.jboss.shrinkwrap</groupId><artifactId>shrinkwrap-api</artifactId></dependency>
<dependency><groupId>org.jboss.shrinkwrap.resolver</groupId><artifactId>shrinkwrap-resolver-impl-maven</artifactId></dependency>

<!-- ADD these -->
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-junit5</artifactId><scope>test</scope></dependency>
<dependency><groupId>io.rest-assured</groupId><artifactId>rest-assured</artifactId><scope>test</scope></dependency>
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-junit5-mockito</artifactId><scope>test</scope></dependency>
```

Also delete: `arquillian.xml`, `test-ds.xml`, and any Arquillian container adapter JARs.

## Annotation Mapping

| Arquillian | Quarkus | Notes |
|---|---|---|
| `@RunWith(Arquillian.class)` | `@QuarkusTest` | Class-level annotation; JUnit 5 only |
| `@Deployment` + `ShrinkWrap` | REMOVE entirely | Quarkus manages full app lifecycle |
| `@Inject` in test | `@Inject` in test | Works natively — no special setup |
| `@ArquillianResource URL` | `@TestHTTPResource` | Injects test server URL |
| `@RunAsClient` | Default behavior | `@QuarkusTest` runs as client by default |
| `@InSequence(n)` | `@TestMethodOrder(OrderAnnotation.class)` + `@Order(n)` | JUnit 5 ordering |
| `arquillian.xml` config | `application.properties` (`%test` profile) | Test-specific config |

---

## 1. Typical Integration Test — Before/After

### Before (Arquillian)

```java
import org.jboss.arquillian.container.test.api.Deployment;
import org.jboss.arquillian.junit.Arquillian;
import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.WebArchive;
import org.junit.Test;
import org.junit.runner.RunWith;
import javax.inject.Inject;
import static org.junit.Assert.*;

@RunWith(Arquillian.class)
public class OrderServiceIT {

    @Deployment
    public static WebArchive createDeployment() {
        return ShrinkWrap.create(WebArchive.class, "test.war")
            .addPackages(true, "com.example.orders")
            .addAsResource("META-INF/persistence.xml")
            .addAsResource("test-data.sql", "import.sql");
    }

    @Inject
    private OrderService orderService;

    @Test
    public void testCreateOrder() {
        Order order = orderService.createOrder(new OrderRequest("item-1", 2));
        assertNotNull(order);
        assertEquals("PENDING", order.getStatus());
    }
}
```

### After (Quarkus)

```java
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
public class OrderServiceIT {

    // No @Deployment needed — Quarkus boots the full application

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

**Key changes**:
- `@RunWith(Arquillian.class)` → `@QuarkusTest`; delete `@Deployment` method entirely
- `org.junit.Test` → `org.junit.jupiter.api.Test`; `Assert.*` → `Assertions.*`
- Test data: move `import.sql` to `src/test/resources/import.sql`

---

## 2. REST API Test — Before/After

### Before (Arquillian + JAX-RS Client)

```java
@RunWith(Arquillian.class)
@RunAsClient
public class OrderResourceIT {
    @Deployment(testable = false)
    public static WebArchive createDeployment() {
        return ShrinkWrap.create(WebArchive.class).addPackages(true, "com.example");
    }

    @ArquillianResource
    private URL deploymentUrl;

    @Test
    public void testGetOrder() {
        Client client = ClientBuilder.newClient();
        Response response = client.target(deploymentUrl.toURI())
            .path("/api/orders/1").request(MediaType.APPLICATION_JSON).get();
        assertEquals(200, response.getStatus());
        client.close();
    }
}
```

### After (Quarkus + REST Assured)

```java
import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;
import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

@QuarkusTest
public class OrderResourceIT {

    // No @Deployment, no @ArquillianResource URL, no JAX-RS Client boilerplate

    @Test
    public void testGetOrder() {
        given().when().get("/api/orders/1")
            .then().statusCode(200)
            .body("id", equalTo(1)).body("status", notNullValue());
    }

    @Test
    public void testCreateOrder() {
        given().contentType("application/json")
            .body("""{"itemId": "item-1", "quantity": 2}""")
            .when().post("/api/orders")
            .then().statusCode(201).body("id", notNullValue());
    }

    @Test
    public void testNotFound() {
        given().when().get("/api/orders/99999").then().statusCode(404);
    }
}
```

**Key changes**: No URL management — REST Assured auto-configures base URL. Fluent assertion API replaces manual Response parsing. `@TestHTTPEndpoint(OrderResource.class)` can set base path per test class.

**Path configuration with root-path** (confirmed by [Quarkus GitHub #28001](https://github.com/quarkusio/quarkus/issues/28001)): When `quarkus.http.root-path=/myapp` is set, Quarkus auto-configures `RestAssured.basePath=/myapp`. Test request paths must be relative to root-path — include the REST path but NOT root-path. Example: with `root-path=/app` and `rest.path=/api`, use `.post("/api/orders")`, not `.post("/app/api/orders")`.

For non-application-root-path endpoints (e.g., `/q/health`), use `@TestHTTPResource("/q/health")` URL injection and pass the URL to `RestAssured.get(url)` to bypass basePath:

```java
@TestHTTPResource("/q/health")
URL healthUrl;

@Test
void testHealthEndpoint() {
    RestAssured.get(healthUrl).then().statusCode(200);
}
```

---

## 3. Database Test — @TestTransaction

Arquillian tests that loaded data via `@Deployment` + `test-data.sql` migrate to one of two patterns.

### Option 1: @TestTransaction (rolled back after each test)

```java
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.TestTransaction;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
@TestTransaction  // each test runs in a TX that is rolled back after
public class OrderRepositoryIT {

    @Inject
    EntityManager em;

    @BeforeEach
    public void setupData() {
        // Inserted data is rolled back after each test
        em.persist(new Order("order-1", "ACTIVE"));
        em.persist(new Order("order-2", "ACTIVE"));
        em.persist(new Order("order-3", "ACTIVE"));
        em.flush();
    }

    @Test
    public void testFindByStatus() {
        List<Order> orders = em.createQuery(
            "SELECT o FROM Order o WHERE o.status = :status", Order.class)
            .setParameter("status", "ACTIVE").getResultList();
        assertEquals(3, orders.size());
    }
}
```

### Option 2: import.sql for static test data

```properties
# src/test/resources/application.properties
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:testdb
quarkus.hibernate-orm.database.generation=drop-and-create
quarkus.hibernate-orm.sql-load-script=import.sql
```

Place static INSERTs in `src/test/resources/import.sql`.

---

## 4. Mocking — @InjectMock

Replaces Arquillian manual `@Alternative` mocks registered in `beans.xml`.

```java
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.InjectMock;
import org.junit.jupiter.api.Test;
import static org.mockito.Mockito.*;
import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

@QuarkusTest
public class OrderResourceIT {

    @InjectMock  // replaces the real CDI bean with a Mockito mock
    PaymentService paymentService;

    @Test
    public void testCreateOrderWithPaymentFailure() {
        when(paymentService.charge(any())).thenThrow(new PaymentException("Declined"));
        given().contentType("application/json")
            .body("""{"itemId": "item-1", "quantity": 1}""")
            .when().post("/api/orders")
            .then().statusCode(402).body("error", equalTo("PAYMENT_DECLINED"));
        verify(paymentService).charge(any());
    }
}
```

**Notes**:
- `@InjectMock` replaces the real CDI bean for the entire test class; mocks reset between tests automatically.
- For partial mock (real bean with spy capabilities): use `@InjectSpy` — then `verify(bean).method(any())` confirms the real method was called.

---

## 5. Test Profiles — @TestProfile

### Per-test configuration with QuarkusTestProfile

```java
import io.quarkus.test.junit.QuarkusTestProfile;
import java.util.Map;

public class MockExternalServiceProfile implements QuarkusTestProfile {
    @Override
    public Map<String, String> getConfigOverrides() {
        return Map.of(
            "external.service.url", "http://localhost:8089/mock",
            "quarkus.datasource.jdbc.url", "jdbc:h2:mem:mocktest"
        );
    }

    @Override
    public String getConfigProfile() {
        return "mockexternal";  // activates %mockexternal.* properties
    }
}
```

```java
@QuarkusTest
@TestProfile(MockExternalServiceProfile.class)
public class ExternalIntegrationIT {
    @Test
    public void testWithMockedExternalService() { /* runs with overridden config */ }
}
```

### Alternative CDI Beans per Profile

Override `getEnabledAlternatives()` to return `Set.of(MockPaymentService.class)`, where the mock is an `@Alternative @Priority(1) @ApplicationScoped` subclass of the real service.

---

## 6. TestContainers Integration

### Dependency

```xml
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-test-containers</artifactId><scope>test</scope></dependency>
<dependency><groupId>org.testcontainers</groupId><artifactId>postgresql</artifactId><scope>test</scope></dependency>
```

### QuarkusTestResource for PostgreSQL

```java
import io.quarkus.test.common.QuarkusTestResourceLifecycleManager;
import org.testcontainers.containers.PostgreSQLContainer;
import java.util.Map;

public class PostgresResource implements QuarkusTestResourceLifecycleManager {

    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
        .withDatabaseName("testdb").withUsername("test").withPassword("test");

    @Override
    public Map<String, String> start() {
        postgres.start();
        return Map.of(
            "quarkus.datasource.jdbc.url", postgres.getJdbcUrl(),
            "quarkus.datasource.username", postgres.getUsername(),
            "quarkus.datasource.password", postgres.getPassword()
        );
    }

    @Override
    public void stop() { postgres.stop(); }
}
```

Apply with `@QuarkusTestResource(PostgresResource.class)` on the test class.

### Dev Services (Zero-Config TestContainers)

Quarkus can auto-start containers without explicit TestResource configuration:

```properties
# src/test/resources/application.properties — just set db-kind
quarkus.datasource.db-kind=postgresql
quarkus.datasource.devservices.enabled=true
# No jdbc.url, username, or password needed — auto-configured
quarkus.hibernate-orm.database.generation=drop-and-create
```

This replaces the entire `QuarkusTestResource` pattern for databases and brokers with Dev Services support (PostgreSQL, MySQL, MariaDB, Kafka, Artemis, Redis, etc.).

---

## 7. Native Mode Testing — @QuarkusIntegrationTest

`@QuarkusIntegrationTest` runs tests against the built artifact (uber-jar or native binary) — not in JVM test mode:

```java
import io.quarkus.test.junit.QuarkusIntegrationTest;
import org.junit.jupiter.api.Test;
import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

@QuarkusIntegrationTest
public class NativeOrderResourceIT {
    @Test
    public void testHealthEndpoint() {
        given().when().get("/q/health").then().statusCode(200).body("status", equalTo("UP"));
    }
}
```

**Key differences from `@QuarkusTest`**:
- `@Inject` does NOT work (test runs outside the application JVM)
- Only HTTP-based testing (REST Assured, WebSocket clients)
- Tests the actual packaged artifact — catches native image issues
- Use with `./mvnw verify -Pnative`

**File naming convention**: place in `src/test/java` with `*IT.java` suffix; configure `maven-failsafe-plugin` with `integration-test` + `verify` goals.

---

## 8. Continuous Testing (quarkus:test mode)

Quarkus provides a built-in continuous testing mode that auto-runs affected tests when source files change:

```bash
./mvnw quarkus:dev     # dev mode; press 'r' re-run, 'o' toggle output, 'p' pause
./mvnw quarkus:test    # standalone continuous test run
```

**Behavior**: only tests affected by the changed source re-run (incremental), in the same JVM as dev mode — extremely fast feedback loop, replaces Arquillian's deploy-test-undeploy cycle.

**Configuration**:
```properties
quarkus.test.continuous-testing=enabled
quarkus.test.include-pattern=.*IT
quarkus.test.exclude-pattern=.*NativeIT
```

---

## 9. MockitoExtension Strict Stubbing (JUnit 4 → 5)

When migrating from JUnit 4 `MockitoJUnitRunner` to JUnit 5 `@ExtendWith(MockitoExtension.class)`, tests may fail with `UnnecessaryStubbingException` — a behavioral difference:

- **JUnit 4 MockitoJUnitRunner** = **lenient** stubbing (unused stubs silently ignored)
- **JUnit 5 MockitoExtension** = **strict** stubbing (unused stubs throw `UnnecessaryStubbingException`)

### Detection

After replacing the runner, run tests. Any `UnnecessaryStubbingException` identifies stubs set up but never invoked.

### Common Cause 1: Removed EJB/JMS Layer

When replacing an EJB/JMS layer with CDI mocks, `@BeforeEach` stubs that exclusively set up the removed layer throw the exception. If ALL stubs in a `@BeforeEach` targeted the removed layer, **delete the entire `@BeforeEach` method** — do not add `@MockitoSettings(LENIENT)`. Move failure-scenario stubs into the individual test methods that need them.

### Common Cause 2: Validation Guards

A test stubs a collaborator, but the code-under-test throws before reaching the stubbed call:

```java
// Service: throws on null BEFORE reaching inventoryService.reserve(...)
public Order createOrder(OrderRequest request) {
    if (request == null || request.getItemId() == null)
        throw new ValidationException("Invalid request");
    return inventoryService.reserve(request.getItemId());
}

// Test — the when() on inventoryService is unnecessary for the null scenario
@Test
void testCreateOrder_nullRequest() {
    when(inventoryService.reserve(any())).thenReturn(new Item());  // UNNECESSARY — remove
    assertThrows(ValidationException.class, () -> service.createOrder(null));
}
```

### Fix: Remove Unreachable Stubs

Delete stubs that set up a method unreachable due to an early exit (null check, validation throw, conditional return). The fixed test keeps only the `assertThrows`.

### Anti-Pattern: Do NOT Apply Class-Wide Lenient Mode

```java
// DO NOT DO THIS — hides dead test code and masks broken assumptions
@MockitoSettings(strictness = Strictness.LENIENT)
@ExtendWith(MockitoExtension.class)
class OrderServiceTest { ... }
```

Fix each stub individually instead.

### Proactive Checklist After Adding MockitoExtension

1. For each test class with `@ExtendWith(MockitoExtension.class)`:
2. Delete `@BeforeEach` methods whose stubs all target removed EJB/JMS collaborators.
3. Identify test methods verifying error/validation paths (null args, blank inputs, invalid state).
4. Check whether those stub collaborator calls that occur AFTER the validation guard.
5. Remove those stubs — they are never consumed.
6. Run `./mvnw test` — no `UnnecessaryStubbingException` should remain.

### Dependency Note

`mockito-junit-jupiter` (provides `MockitoExtension`) is **NOT** managed by the Quarkus BOM. Always specify an explicit version:

```xml
<dependency>
    <groupId>org.mockito</groupId><artifactId>mockito-junit-jupiter</artifactId>
    <version>5.11.0</version><scope>test</scope>
</dependency>
```

---

## 10. JUnit 4→5 Assertion Argument Order

**Critical mechanical fix**: JUnit 4 puts the message String as the FIRST argument; JUnit 5 moves it to the LAST position. Applies to ALL 3-argument assertion methods.

### Before/After Table

| JUnit 4 (message FIRST) | JUnit 5 (message LAST) |
|---|---|
| `assertEquals("msg", expected, actual)` | `assertEquals(expected, actual, "msg")` |
| `assertNotNull("msg", object)` | `assertNotNull(object, "msg")` |
| `assertNull("msg", object)` | `assertNull(object, "msg")` |
| `assertTrue("msg", condition)` | `assertTrue(condition, "msg")` |
| `assertFalse("msg", condition)` | `assertFalse(condition, "msg")` |
| `assertSame("msg", expected, actual)` | `assertSame(expected, actual, "msg")` |
| `assertArrayEquals("msg", expected, actual)` | `assertArrayEquals(expected, actual, "msg")` |

2-argument assertions (no message) are unchanged.

### Detection Command

```bash
grep -rn --include='*.java' 'assert\(Not\)\?Null\|assert\(True\|False\|Equals\)' src/test/ | grep '".*"\s*,'
```

### WARNING: Silent Semantic Bugs

When both arguments are the same type (e.g., both String), swapping the order **compiles without error** but tests the wrong thing:

```java
// JUnit 4
assertEquals("Expected ACTIVE status", "ACTIVE", order.getStatus());
// CORRECT JUnit 5 migration — message moves LAST:
assertEquals("ACTIVE", order.getStatus(), "Expected ACTIVE status");
```

Leaving the JUnit-4 order in JUnit 5 would assert `expected="Expected ACTIVE status", actual="ACTIVE"` — a silent bug.

---

## 11. CDI @ConfigProperty in Unit Tests

CDI `@ConfigProperty` fields are `null` (or `0` for primitives) in plain unit tests where no CDI container runs (tests using `new MyClass()` or `@ExtendWith(MockitoExtension.class)` without `@QuarkusTest`). The `defaultValue` attribute is only applied by the MicroProfile Config runtime — NOT in plain unit tests.

**Fix Option A — reflective field injection in `@BeforeEach`** (unit tests):
```java
@BeforeEach
void setup() throws Exception {
    Field f = MyService.class.getDeclaredField("timeout");
    f.setAccessible(true);
    f.set(service, 30);
}
```

**Fix Option B — convert to `@QuarkusTest`** (integration tests verifying config-dependent behavior): `@Inject MyService service;` — `@ConfigProperty` is applied by the runtime.

**Fix Option C — field initializer matching defaultValue** (see troubleshooting-pitfalls.md): initialize the field to the same literal as `defaultValue`; CDI overwrites it after construction, plain `new` keeps the fallback.

---

## Migration Checklist

| Step | Action | Verify |
|---|---|---|
| 1 | Remove Arquillian dependencies from pom.xml | `grep -rn "arquillian\|shrinkwrap" pom.xml` empty |
| 2 | Add `quarkus-junit5` + `rest-assured` | Compile passes |
| 3 | Replace `@RunWith(Arquillian.class)` with `@QuarkusTest` | `grep -rn "Arquillian" src/test/` empty |
| 4 | Delete ALL `@Deployment` methods | `grep -rn "ShrinkWrap\|@Deployment" src/test/` empty |
| 5 | Replace `@ArquillianResource URL` with `@TestHTTPResource` | No URL management code |
| 6 | Convert JAX-RS Client calls to REST Assured | Cleaner test code |
| 7 | Migrate JUnit 4 → 5 assertions (swap message arg) | `org.junit.Test` → `org.junit.jupiter.api.Test` |
| 8 | Create `src/test/resources/application.properties` with H2 config | Test datasource configured |
| 9 | Delete `arquillian.xml`, `test-ds.xml` | Files removed |
| 10 | Remove unnecessary stubs (MockitoExtension strict mode) | No `UnnecessaryStubbingException` |
| 11 | Run `./mvnw clean test` | ALL tests pass |

## JUnit 4 → 5 Quick Reference

| JUnit 4 | JUnit 5 | Notes |
|---|---|---|
| `@RunWith(...)` | `@ExtendWith(...)` | Not needed with `@QuarkusTest` |
| `@RunWith(MockitoJUnitRunner.class)` | `@ExtendWith(MockitoExtension.class)` | Strict stubbing — see Section 9 |
| `@Test` (org.junit) | `@Test` (org.junit.jupiter.api) | Different import |
| `@Before` / `@After` | `@BeforeEach` / `@AfterEach` | |
| `@BeforeClass` / `@AfterClass` | `@BeforeAll` / `@AfterAll` | |
| `@Ignore` | `@Disabled` | Do NOT add as migration shortcut — fix the test |
| `@Test(expected=X.class)` | `assertThrows(X.class, () -> {...})` | |
| `Assert.assertEquals(msg, exp, act)` | `assertEquals(exp, act, msg)` | Message moves LAST — see Section 10 |
| `@Rule ExpectedException` | `assertThrows(...)` | |
| `@Category(...)` | `@Tag(...)` | |
