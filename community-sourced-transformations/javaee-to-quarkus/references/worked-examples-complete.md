# Complete Migration Examples

## Simple Patterns

### JAX-RS Servlet → REST
```java
// Before: WebServlet
@WebServlet(urlPatterns = "/hello")
public class HelloServlet extends HttpServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) {
        resp.getWriter().println("Hello " + req.getParameter("name"));
    }
}

// After: JAX-RS
@Path("/hello")
public class HelloResource {
    @GET
    public String hello(@QueryParam("name") String name) {
        return "Hello " + name;
    }
}
```

### EJB+JPA → CDI+Panache
```java
// Before: EJB
@Stateless
public class CustomerService {
    @PersistenceContext EntityManager em;
    
    public List<Customer> findAll() {
        return em.createQuery("SELECT c FROM Customer c").getResultList();
    }
}

// After: CDI+Panache
@ApplicationScoped
@Transactional
public class CustomerService {
    public List<Customer> findAll() {
        return Customer.listAll();
    }
}
```

## Complete Application Migration

### E-Commerce Application: WildFly → Quarkus

**Original Structure:**
- WAR deployment with EJBs, JPA, JAX-RS, JSF
- Uses WildFly datasource, JMS queues
- Form-based authentication

**Migration Steps Applied:**

**Phase 0: Analysis**
```bash
# Detected features
EJB_NEEDED=true (found @Stateless, @MessageDriven)
JMS_NEEDED=true (found @Resource Queue)
JSF_NEEDED=true (found .xhtml files)
SECURITY_NEEDED=true (found @RolesAllowed)
```

**Phase 1: Build System**
```xml
<!-- Added to pom.xml -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>io.quarkus.platform</groupId>
      <artifactId>quarkus-bom</artifactId>
      <version>3.33.2</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-arc</artifactId>
  </dependency>
  <dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-hibernate-orm-panache</artifactId>
  </dependency>
  <dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-rest</artifactId>
  </dependency>
</dependencies>
```

**Phase 2: EJB Migration**
```java
// Before: Order Service EJB
@Stateless
@TransactionAttribute(TransactionAttributeType.REQUIRED)
public class OrderService {
    @PersistenceContext
    private EntityManager em;
    
    @EJB
    private InventoryService inventory;
    
    public void createOrder(Order order) {
        inventory.reserveItems(order.getItems());
        em.persist(order);
    }
}

// After: CDI Bean
@ApplicationScoped
@Transactional
public class OrderService {
    @Inject
    EntityManager em;
    
    @Inject
    InventoryService inventory;
    
    public void createOrder(Order order) {
        inventory.reserveItems(order.getItems());
        em.persist(order);
    }
}
```

**Final Result:**
- Build time: 15s → 8s
- Memory footprint: 512MB → 128MB
- Startup time: 45s → 2.3s
- All 47 integration tests passing