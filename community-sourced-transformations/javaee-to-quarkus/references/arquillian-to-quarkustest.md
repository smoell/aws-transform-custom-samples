# Arquillian → @QuarkusTest Reference

> Reference for Phase 4 Step 15: Test framework migration.
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
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-junit5</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-junit5-mockito</artifactId>
    <scope>test</scope>
</dependency>
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
import org.jboss.shrinkwrap.api.asset.EmptyAsset;
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
            .addAsResource("test-data.sql", "import.sql")
            .addAsWebInfResource(EmptyAsset.INSTANCE, "beans.xml");
    }

    @Inject
    private OrderService orderService;

    @Test
    public void testCreateOrder() {
        OrderRequest request = new OrderRequest("item-1", 2);
        Order order = orderService.createOrder(request);
        assertNotNull(order);
        assertNotNull(order.getId());
        assertEquals("PENDING", order.getStatus());
    }

    @Test
    public void testFindOrder() {
        Order order = orderService.findById(1L);
        assertNotNull(order);
        assertEquals("Test Order", order.getName());
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
        OrderRequest request = new OrderRequest("item-1", 2);
        Order order = orderService.createOrder(request);
        assertNotNull(order);
        assertNotNull(order.getId());
        assertEquals("PENDING", order.getStatus());
    }

    @Test
    public void testFindOrder() {
        Order order = orderService.findById(1L);
        assertNotNull(order);
        assertEquals("Test Order", order.getName());
    }
}
```

**Key changes**:
- `@RunWith(Arquillian.class)` → `@QuarkusTest`
- Delete `@Deployment` method entirely
- `org.junit.Test` → `org.junit.jupiter.api.Test`
- `Assert.*` → `Assertions.*`
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
        return ShrinkWrap.create(WebArchive.class)
            .addPackages(true, "com.example")
            .addAsResource("META-INF/persistence.xml");
    }

    @ArquillianResource
    private URL deploymentUrl;

    @Test
    public void testGetOrder() {
        Client client = ClientBuilder.newClient();
        Response response = client.target(deploymentUrl.toURI())
            .path("/api/orders/1")
            .request(MediaType.APPLICATION_JSON)
            .get();

        assertEquals(200, response.getStatus());
        Order order = response.readEntity(Order.class);
        assertNotNull(order);
        assertEquals(Long.valueOf(1L), order.getId());
        client.close();
    }

    @Test
    public void testCreateOrder() {
        Client client = ClientBuilder.newClient();
        OrderRequest request = new OrderRequest("item-1", 2);

        Response response = client.target(deploymentUrl.toURI())
            .path("/api/orders")
            .request(MediaType.APPLICATION_JSON)
            .post(Entity.json(request));

        assertEquals(201, response.getStatus());
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
        given()
            .when().get("/api/orders/1")
            .then()
            .statusCode(200)
            .body("id", equalTo(1))
            .body("status", notNullValue());
    }

    @Test
    public void testCreateOrder() {
        given()
            .contentType("application/json")
            .body("""
                {"itemId": "item-1", "quantity": 2}
                """)
            .when().post("/api/orders")
            .then()
            .statusCode(201)
            .body("id", notNullValue())
            .body("status", equalTo("PENDING"));
    }

    @Test
    public void testNotFound() {
        given()
            .when().get("/api/orders/99999")
            .then()
            .statusCode(404);
    }
}
```

**Key changes**:
- No URL management — REST Assured auto-configures base URL from test server
- Fluent assertion API replaces manual Response parsing
- `@TestHTTPEndpoint(OrderResource.class)` can set base path per test class

---

## 3. Database Test — @TestTransaction

### Before (Arquillian with test data in @Deployment)

```java
@RunWith(Arquillian.class)
public class OrderRepositoryIT {

    @Deployment
    public static WebArchive createDeployment() {
        return ShrinkWrap.create(WebArchive.class)
            .addPackages(true, "com.example")
            .addAsResource("META-INF/persistence.xml")
            .addAsResource("test-data.sql", "import.sql")
            .addAsWebInfResource(EmptyAsset.INSTANCE, "beans.xml");
    }

    @Inject
    private EntityManager em;

    @Test
    public void testFindByStatus() {
        // test-data.sql inserted sample data
        List<Order> orders = em.createQuery(
            "SELECT o FROM Order o WHERE o.status = :status", Order.class)
            .setParameter("status", "ACTIVE")
            .getResultList();
        assertEquals(3, orders.size());
    }
}
```

### After — Option 1: @TestTransaction (rolled back after each test)

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
        // Insert test data — will be rolled back after each test
        em.persist(new Order("order-1", "ACTIVE"));
        em.persist(new Order("order-2", "ACTIVE"));
        em.persist(new Order("order-3", "ACTIVE"));
        em.persist(new Order("order-4", "COMPLETED"));
        em.flush();
    }

    @Test
    public void testFindByStatus() {
        List<Order> orders = em.createQuery(
            "SELECT o FROM Order o WHERE o.status = :status", Order.class)
            .setParameter("status", "ACTIVE")
            .getResultList();
        assertEquals(3, orders.size());
    }
}
```

### After — Option 2: import.sql for static test data

```properties
# src/test/resources/application.properties
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:testdb
quarkus.hibernate-orm.database.generation=drop-and-create
quarkus.hibernate-orm.sql-load-script=import.sql
```

Place `src/test/resources/import.sql`:
```sql
INSERT INTO orders (id, name, status) VALUES (1, 'Test Order 1', 'ACTIVE');
INSERT INTO orders (id, name, status) VALUES (2, 'Test Order 2', 'ACTIVE');
INSERT INTO orders (id, name, status) VALUES (3, 'Test Order 3', 'ACTIVE');
INSERT INTO orders (id, name, status) VALUES (4, 'Test Order 4', 'COMPLETED');
```

---

## 4. Mocking — @InjectMock

### Before (Arquillian alternatives / manual mocks)

```java
@RunWith(Arquillian.class)
public class OrderResourceIT {

    @Deployment
    public static WebArchive createDeployment() {
        return ShrinkWrap.create(WebArchive.class)
            .addPackages(true, "com.example")
            .addClass(MockPaymentService.class)  // manual mock alternative
            .addAsWebInfResource(EmptyAsset.INSTANCE, "beans.xml");
    }

    // Manual CDI @Alternative mock registered in beans.xml
}
```

### After (Quarkus @InjectMock)

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
        // Setup mock behavior
        when(paymentService.charge(any())).thenThrow(new PaymentException("Declined"));

        given()
            .contentType("application/json")
            .body("""
                {"itemId": "item-1", "quantity": 1, "paymentInfo": {"card": "4111..."}}
                """)
            .when().post("/api/orders")
            .then()
            .statusCode(402)
            .body("error", equalTo("PAYMENT_DECLINED"));

        verify(paymentService).charge(any());
    }

    @Test
    public void testCreateOrderSuccess() {
        when(paymentService.charge(any())).thenReturn(new PaymentResult("txn-123"));

        given()
            .contentType("application/json")
            .body("""
                {"itemId": "item-1", "quantity": 1, "paymentInfo": {"card": "4111..."}}
                """)
            .when().post("/api/orders")
            .then()
            .statusCode(201);
    }
}
```

**Notes**:
- `@InjectMock` replaces the real CDI bean for the entire test class.
- Mocks are reset between tests automatically.
- For spy behavior (partial mock): use `@InjectSpy`.
- For mock that applies to a single test: setup expectations in the test method.

### Mocking with @InjectSpy (partial mock)

```java
@QuarkusTest
public class OrderServiceIT {

    @InjectSpy
    OrderService orderService;  // real bean with spy capabilities

    @Test
    public void testOrderCreationCallsValidation() {
        given()
            .contentType("application/json")
            .body("""{"itemId": "item-1", "quantity": 1}""")
            .when().post("/api/orders")
            .then()
            .statusCode(201);

        // Verify the real method was called
        verify(orderService).validateOrder(any());
    }
}
```

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
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;

@QuarkusTest
@TestProfile(MockExternalServiceProfile.class)
public class ExternalIntegrationIT {

    @Test
    public void testWithMockedExternalService() {
        // Test runs with overridden config
    }
}
```

### Alternative CDI Beans per Profile

```java
public class MockServiceProfile implements QuarkusTestProfile {

    @Override
    public Set<Class<?>> getEnabledAlternatives() {
        return Set.of(MockPaymentService.class);
    }
}

// The mock alternative
@Alternative
@Priority(1)
@ApplicationScoped
public class MockPaymentService extends PaymentService {
    @Override
    public PaymentResult charge(PaymentInfo info) {
        return new PaymentResult("mock-txn-001");
    }
}
```

---

## 6. TestContainers Integration

### Dependency

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-test-containers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
```

### QuarkusTestResource for PostgreSQL

```java
import io.quarkus.test.common.QuarkusTestResource;
import io.quarkus.test.common.QuarkusTestResourceLifecycleManager;
import org.testcontainers.containers.PostgreSQLContainer;
import java.util.Map;

public class PostgresResource implements QuarkusTestResourceLifecycleManager {

    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

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
    public void stop() {
        postgres.stop();
    }
}
```

```java
@QuarkusTest
@QuarkusTestResource(PostgresResource.class)
public class OrderRepositoryIT {

    @Inject
    OrderRepository orderRepository;

    @Test
    @TestTransaction
    public void testPersistOrder() {
        Order order = new Order("test-item", "ACTIVE");
        orderRepository.persist(order);
        assertNotNull(order.getId());
    }
}
```

### Dev Services (Zero-Config TestContainers)

Quarkus can auto-start containers without explicit TestResource configuration:

```properties
# src/test/resources/application.properties
# Just set db-kind — Quarkus auto-starts a PostgreSQL container!
quarkus.datasource.db-kind=postgresql
quarkus.datasource.devservices.enabled=true
# No jdbc.url, username, or password needed — auto-configured
quarkus.hibernate-orm.database.generation=drop-and-create
```

This replaces the entire `QuarkusTestResource` pattern for databases and message brokers that have Dev Services support (PostgreSQL, MySQL, MariaDB, Kafka, Artemis, Redis, etc.).

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
        given()
            .when().get("/q/health")
            .then()
            .statusCode(200)
            .body("status", equalTo("UP"));
    }

    @Test
    public void testGetOrders() {
        given()
            .when().get("/api/orders")
            .then()
            .statusCode(200)
            .body("size()", greaterThanOrEqualTo(0));
    }
}
```

**Key differences from `@QuarkusTest`**:
- `@Inject` does NOT work (test runs outside the application JVM)
- Only HTTP-based testing (REST Assured, WebSocket clients)
- Tests the actual packaged artifact — catches native image issues
- Use with `./mvnw verify -Pnative` to test the native binary

**File naming convention**: place in `src/test/java` with `*IT.java` suffix. Configure maven-failsafe-plugin:

```xml
<plugin>
    <artifactId>maven-failsafe-plugin</artifactId>
    <version>3.2.5</version>
    <executions>
        <execution>
            <goals>
                <goal>integration-test</goal>
                <goal>verify</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

---

## 8. Continuous Testing (quarkus:test mode)

Quarkus provides a built-in continuous testing mode that auto-runs affected tests when source files change:

```bash
# Start dev mode with continuous testing
./mvnw quarkus:dev
# Press 'r' in the console to re-run all tests
# Press 'o' to toggle test output
# Press 'p' to pause/resume continuous testing
```

Or run tests standalone:
```bash
./mvnw quarkus:test
```

**Behavior**:
- Only tests affected by the changed source file re-run (incremental)
- Tests run in the same JVM as dev mode — extremely fast feedback loop
- Replaces Arquillian's slow deploy-test-undeploy cycle

**Configuration**:
```properties
# Enable/disable continuous testing on startup
quarkus.test.continuous-testing=enabled

# Include/exclude test patterns
quarkus.test.include-pattern=.*IT
quarkus.test.exclude-pattern=.*NativeIT
```

---

## Migration Checklist

| Step | Action | Verify |
|---|---|---|
| 1 | Remove Arquillian dependencies from pom.xml | `grep -rn "arquillian\|shrinkwrap" pom.xml` returns empty |
| 2 | Add `quarkus-junit5` + `rest-assured` dependencies | Compile passes |
| 3 | Replace `@RunWith(Arquillian.class)` with `@QuarkusTest` on all test classes | `grep -rn "Arquillian" src/test/` returns empty |
| 4 | Delete ALL `@Deployment` methods | `grep -rn "ShrinkWrap\|@Deployment" src/test/` returns empty |
| 5 | Replace `@ArquillianResource URL` with `@TestHTTPResource` | No URL management code |
| 6 | Convert JAX-RS Client calls to REST Assured | Cleaner test code |
| 7 | Migrate JUnit 4 → JUnit 5 assertions | `org.junit.Test` → `org.junit.jupiter.api.Test` |
| 8 | Create `src/test/resources/application.properties` with H2 config | Test datasource configured |
| 9 | Delete `arquillian.xml`, `test-ds.xml` | Files removed |
| 10 | Run `./mvnw clean test` | ALL tests pass |

## JUnit 4 → 5 Quick Reference

| JUnit 4 | JUnit 5 | Notes |
|---|---|---|
| `@RunWith(...)` | `@ExtendWith(...)` | Not needed with `@QuarkusTest` |
| `@Test` (org.junit) | `@Test` (org.junit.jupiter.api) | Different import |
| `@Before` | `@BeforeEach` | |
| `@After` | `@AfterEach` | |
| `@BeforeClass` | `@BeforeAll` | |
| `@Ignore` | `@Disabled` | Do NOT add `@Disabled` as migration shortcut — fix the test |
| `@Test(expected=X.class)` | `assertThrows(X.class, () -> {...})` | |
| `Assert.assertEquals(msg, exp, act)` | `assertEquals(exp, act, msg)` | Message moves to LAST position |
| `Assert.assertTrue(msg, cond)` | `assertTrue(cond, msg)` | Message moves to LAST position |
| `@Rule ExpectedException` | `assertThrows(...)` | |
| `@Category(...)` | `@Tag(...)` | |
