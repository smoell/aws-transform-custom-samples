# Worked Examples: Before/After Code

## 1. JAX-RS Servlet → REST

### Before: WebServlet
```java
@WebServlet(urlPatterns = "/hello")
public class HelloServlet extends HttpServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) {
        resp.getWriter().println("Hello " + req.getParameter("name"));
    }
}
```

### After: JAX-RS
```java
@Path("/hello")
public class HelloResource {
    @GET
    public String hello(@QueryParam("name") String name) {
        return "Hello " + name;
    }
}
```

## 2. EJB+JPA → CDI+Panache

### Before: Stateless EJB
```java
@Stateless
public class CustomerService {
    @PersistenceContext
    EntityManager em;
    
    @EJB
    private OrderService orderService;
    
    public Customer findById(Long id) {
        return em.find(Customer.class, id);
    }
}
```

### After: CDI
```java
@ApplicationScoped
@Transactional
public class CustomerService {
    @Inject
    EntityManager em;
    
    @Inject
    OrderService orderService;
    
    public Customer findById(Long id) {
        return em.find(Customer.class, id);
    }
}
```

## 3. MDB → @Incoming

### Before: Message-Driven Bean
```java
@MessageDriven(activationConfig = {
    @ActivationConfigProperty(propertyName = "destination", propertyValue = "java:/jms/queue/orders")
})
public class OrderProcessorMDB implements MessageListener {
    public void onMessage(Message message) {
        // Process order
    }
}
```

### After: SmallRye Reactive Messaging
```java
@ApplicationScoped
public class OrderProcessor {
    @Incoming("orders")
    @Blocking
    public void processOrder(String orderData) {
        // Process order
    }
}
```

## Additional Examples

### 4. Arquillian → QuarkusTest
```java
// Before: Arquillian
@RunWith(Arquillian.class)
public class CustomerServiceTest {
    @Deployment
    public static WebArchive createDeployment() {
        return ShrinkWrap.create(WebArchive.class)
            .addClass(CustomerService.class);
    }
    
    @EJB
    CustomerService customerService;
    
    @Test
    public void testFindCustomer() {
        Customer customer = customerService.findById(1L);
        assertNotNull(customer);
    }
}

// After: QuarkusTest
@QuarkusTest
public class CustomerServiceTest {
    @Inject
    CustomerService customerService;
    
    @Test
    public void testFindCustomer() {
        Customer customer = customerService.findById(1L);
        assertNotNull(customer);
    }
}
```

### 5. JNDI Lookup → CDI Injection
```java
// Before: JNDI
public class OrderController {
    private OrderService orderService;
    
    @PostConstruct
    public void init() {
        InitialContext ctx = new InitialContext();
        orderService = (OrderService) ctx.lookup("java:app/OrderService");
    }
}

// After: CDI
@ApplicationScoped
public class OrderController {
    @Inject
    OrderService orderService;
}
```