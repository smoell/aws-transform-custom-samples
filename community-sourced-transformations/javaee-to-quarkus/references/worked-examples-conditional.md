# Worked Examples — Conditional Migrations

> Complete before/after code examples for conditional phases (JMS, Security, SOAP, JSF, Testing).
> Each example shows the FULL source class for both JavaEE and Quarkus.

---

## Example 1: MDB → SmallRye @Incoming

### Before (JavaEE — Message-Driven Bean)

```java
package com.example.messaging;

import javax.ejb.MessageDriven;
import javax.ejb.ActivationConfigProperty;
import javax.ejb.EJB;
import javax.ejb.TransactionAttribute;
import javax.ejb.TransactionAttributeType;
import javax.jms.Message;
import javax.jms.MessageListener;
import javax.jms.TextMessage;
import javax.jms.JMSException;
import java.util.logging.Level;
import java.util.logging.Logger;

@MessageDriven(activationConfig = {
    @ActivationConfigProperty(
        propertyName = "destinationType",
        propertyValue = "javax.jms.Queue"),
    @ActivationConfigProperty(
        propertyName = "destination",
        propertyValue = "java:/jms/queue/OrderProcessingQueue"),
    @ActivationConfigProperty(
        propertyName = "acknowledgeMode",
        propertyValue = "Auto-acknowledge"),
    @ActivationConfigProperty(
        propertyName = "maxSession",
        propertyValue = "5")
})
@TransactionAttribute(TransactionAttributeType.REQUIRED)
public class OrderProcessingMDB implements MessageListener {

    private static final Logger LOG = Logger.getLogger(OrderProcessingMDB.class.getName());

    @EJB
    private OrderService orderService;

    @EJB
    private NotificationService notificationService;

    @Override
    public void onMessage(Message message) {
        try {
            if (message instanceof TextMessage) {
                TextMessage textMessage = (TextMessage) message;
                String orderId = textMessage.getText();
                String priority = message.getStringProperty("priority");

                LOG.info("Processing order: " + orderId + " with priority: " + priority);

                OrderResult result = orderService.processOrder(orderId);

                if (result.isSuccess()) {
                    notificationService.sendConfirmation(orderId, result.getTrackingNumber());
                } else {
                    notificationService.sendFailureAlert(orderId, result.getErrorMessage());
                }
            } else {
                LOG.warning("Received non-text message, ignoring");
            }
        } catch (JMSException e) {
            LOG.log(Level.SEVERE, "Error processing JMS message", e);
            throw new RuntimeException("Failed to process order message", e);
        }
    }
}
```

### After (Quarkus — SmallRye Reactive Messaging)

```java
package com.example.messaging;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.context.control.ActivateRequestContext;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.eclipse.microprofile.reactive.messaging.Incoming;
import org.eclipse.microprofile.reactive.messaging.Message;
import org.jboss.logging.Logger;
import java.util.concurrent.CompletionStage;

@ApplicationScoped
public class OrderProcessingConsumer {

    private static final Logger LOG = Logger.getLogger(OrderProcessingConsumer.class);

    @Inject
    OrderService orderService;

    @Inject
    NotificationService notificationService;

    @Incoming("order-processing")
    @ActivateRequestContext  // required if any injected bean is @RequestScoped
    @Transactional
    public CompletionStage<Void> onMessage(Message<String> message) {
        String orderId = message.getPayload();
        // Access metadata (AMQP application properties)
        String priority = message.getMetadata()
            .get(io.smallrye.reactive.messaging.amqp.IncomingAmqpMetadata.class)
            .map(meta -> meta.getProperties().getString("priority"))
            .orElse("NORMAL");

        LOG.infof("Processing order: %s with priority: %s", orderId, priority);

        try {
            OrderResult result = orderService.processOrder(orderId);

            if (result.isSuccess()) {
                notificationService.sendConfirmation(orderId, result.getTrackingNumber());
            } else {
                notificationService.sendFailureAlert(orderId, result.getErrorMessage());
            }

            return message.ack();  // acknowledge on success
        } catch (Exception e) {
            LOG.errorf(e, "Error processing order: %s", orderId);
            return message.nack(e);  // negative ack → dead-letter queue
        }
    }
}
```

### application.properties

```properties
# Incoming channel: order processing queue
mp.messaging.incoming.order-processing.connector=smallrye-amqp
mp.messaging.incoming.order-processing.address=OrderProcessingQueue
mp.messaging.incoming.order-processing.durable=true
mp.messaging.incoming.order-processing.failure-strategy=dead-letter-queue
mp.messaging.incoming.order-processing.dead-letter-queue.queue=OrderProcessingQueue-DLQ

# AMQP broker
amqp-host=localhost
amqp-port=5672
amqp-username=admin
amqp-password=admin
```

---

## Example 2: JMS Queue Producer → Emitter Pattern

### Before (JavaEE — JMS Producer)

```java
package com.example.messaging;

import javax.ejb.Stateless;
import javax.annotation.Resource;
import javax.inject.Inject;
import javax.jms.ConnectionFactory;
import javax.jms.JMSContext;
import javax.jms.JMSException;
import javax.jms.Queue;
import javax.jms.TextMessage;
import java.util.logging.Level;
import java.util.logging.Logger;

@Stateless
public class OrderEventPublisher {

    private static final Logger LOG = Logger.getLogger(OrderEventPublisher.class.getName());

    @Resource(lookup = "java:/jms/queue/OrderEventsQueue")
    private Queue orderEventsQueue;

    @Resource(lookup = "java:/jms/queue/AuditQueue")
    private Queue auditQueue;

    @Inject
    private JMSContext jmsContext;

    public void publishOrderCreated(String orderId, String customerId, double total) {
        try {
            String payload = String.format(
                "{\"event\":\"ORDER_CREATED\",\"orderId\":\"%s\",\"customerId\":\"%s\",\"total\":%.2f}",
                orderId, customerId, total);

            TextMessage message = jmsContext.createTextMessage(payload);
            message.setStringProperty("eventType", "ORDER_CREATED");
            message.setStringProperty("orderId", orderId);

            jmsContext.createProducer()
                .setDeliveryMode(javax.jms.DeliveryMode.PERSISTENT)
                .send(orderEventsQueue, message);

            LOG.info("Published ORDER_CREATED event for order: " + orderId);
        } catch (JMSException e) {
            LOG.log(Level.SEVERE, "Failed to publish order event", e);
            throw new RuntimeException("Event publishing failed", e);
        }
    }

    public void publishAuditEvent(String action, String userId, String details) {
        String payload = String.format(
            "{\"action\":\"%s\",\"userId\":\"%s\",\"details\":\"%s\",\"timestamp\":%d}",
            action, userId, details, System.currentTimeMillis());

        jmsContext.createProducer().send(auditQueue, payload);
    }
}
```

### After (Quarkus — SmallRye Emitter)

```java
package com.example.messaging;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.reactive.messaging.Channel;
import org.eclipse.microprofile.reactive.messaging.Emitter;
import org.eclipse.microprofile.reactive.messaging.Message;
import org.eclipse.microprofile.reactive.messaging.Metadata;
import io.smallrye.reactive.messaging.amqp.OutgoingAmqpMetadata;
import org.jboss.logging.Logger;

@ApplicationScoped
public class OrderEventPublisher {

    private static final Logger LOG = Logger.getLogger(OrderEventPublisher.class);

    @Inject
    @Channel("order-events-out")
    Emitter<String> orderEventsEmitter;

    @Inject
    @Channel("audit-out")
    Emitter<String> auditEmitter;

    public void publishOrderCreated(String orderId, String customerId, double total) {
        String payload = String.format(
            "{\"event\":\"ORDER_CREATED\",\"orderId\":\"%s\",\"customerId\":\"%s\",\"total\":%.2f}",
            orderId, customerId, total);

        // Attach AMQP application properties (equivalent to JMS message properties)
        OutgoingAmqpMetadata metadata = OutgoingAmqpMetadata.builder()
            .withApplicationProperty("eventType", "ORDER_CREATED")
            .withApplicationProperty("orderId", orderId)
            .withDurable(true)
            .build();

        orderEventsEmitter.send(Message.of(payload, Metadata.of(metadata)));

        LOG.infof("Published ORDER_CREATED event for order: %s", orderId);
    }

    public void publishAuditEvent(String action, String userId, String details) {
        String payload = String.format(
            "{\"action\":\"%s\",\"userId\":\"%s\",\"details\":\"%s\",\"timestamp\":%d}",
            action, userId, details, System.currentTimeMillis());

        auditEmitter.send(payload);
    }
}
```

### application.properties

```properties
# Outgoing channel: order events
mp.messaging.outgoing.order-events-out.connector=smallrye-amqp
mp.messaging.outgoing.order-events-out.address=OrderEventsQueue
mp.messaging.outgoing.order-events-out.durable=true

# Outgoing channel: audit
mp.messaging.outgoing.audit-out.connector=smallrye-amqp
mp.messaging.outgoing.audit-out.address=AuditQueue
mp.messaging.outgoing.audit-out.durable=true
```

---

## Example 3: JAAS-Secured REST Endpoint → Quarkus Security

### Before (JavaEE — JAAS + EJB Security)

```java
package com.example.api;

import javax.annotation.security.DeclareRoles;
import javax.annotation.security.PermitAll;
import javax.annotation.security.RolesAllowed;
import javax.ejb.EJB;
import javax.ejb.Stateless;
import javax.ws.rs.*;
import javax.ws.rs.core.Context;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.Response;
import javax.ws.rs.core.SecurityContext;
import java.util.List;

@Path("/users")
@Stateless
@DeclareRoles({"ADMIN", "USER", "MANAGER"})
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class UserResource {

    @EJB
    private UserService userService;

    @Context
    private SecurityContext securityContext;

    @GET
    @RolesAllowed({"ADMIN", "MANAGER"})
    public Response listUsers(@QueryParam("active") Boolean active) {
        List<UserDTO> users;
        if (Boolean.TRUE.equals(active)) {
            users = userService.findActiveUsers();
        } else {
            users = userService.findAllUsers();
        }
        return Response.ok(users).build();
    }

    @GET
    @Path("/me")
    @RolesAllowed({"ADMIN", "USER", "MANAGER"})
    public Response getCurrentUser() {
        String username = securityContext.getUserPrincipal().getName();
        UserDTO user = userService.findByUsername(username);
        if (user == null) {
            return Response.status(Response.Status.NOT_FOUND).build();
        }
        return Response.ok(user).build();
    }

    @POST
    @RolesAllowed("ADMIN")
    public Response createUser(CreateUserRequest request) {
        if (!securityContext.isUserInRole("ADMIN")) {
            return Response.status(Response.Status.FORBIDDEN).build();
        }
        UserDTO created = userService.createUser(request);
        return Response.status(Response.Status.CREATED).entity(created).build();
    }

    @DELETE
    @Path("/{id}")
    @RolesAllowed("ADMIN")
    public Response deleteUser(@PathParam("id") Long id) {
        String currentUser = securityContext.getUserPrincipal().getName();
        userService.deleteUser(id, currentUser);
        return Response.noContent().build();
    }

    @GET
    @Path("/health")
    @PermitAll
    public Response healthCheck() {
        return Response.ok("{\"status\":\"UP\"}").build();
    }
}
```

### After (Quarkus)

```java
package com.example.api;

import jakarta.annotation.security.PermitAll;
import jakarta.annotation.security.RolesAllowed;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import io.quarkus.security.identity.SecurityIdentity;
import org.jboss.logging.Logger;
import java.util.List;

@Path("/users")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class UserResource {

    private static final Logger LOG = Logger.getLogger(UserResource.class);

    @Inject
    UserService userService;

    @Inject
    SecurityIdentity identity;

    @GET
    @RolesAllowed({"ADMIN", "MANAGER"})
    public Response listUsers(@QueryParam("active") Boolean active) {
        List<UserDTO> users;
        if (Boolean.TRUE.equals(active)) {
            users = userService.findActiveUsers();
        } else {
            users = userService.findAllUsers();
        }
        return Response.ok(users).build();
    }

    @GET
    @Path("/me")
    @RolesAllowed({"ADMIN", "USER", "MANAGER"})
    public Response getCurrentUser() {
        String username = identity.getPrincipal().getName();
        UserDTO user = userService.findByUsername(username);
        if (user == null) {
            return Response.status(Response.Status.NOT_FOUND).build();
        }
        return Response.ok(user).build();
    }

    @POST
    @RolesAllowed("ADMIN")
    public Response createUser(CreateUserRequest request) {
        // @RolesAllowed already enforces ADMIN — no need for manual check
        UserDTO created = userService.createUser(request);
        return Response.status(Response.Status.CREATED).entity(created).build();
    }

    @DELETE
    @Path("/{id}")
    @RolesAllowed("ADMIN")
    public Response deleteUser(@PathParam("id") Long id) {
        String currentUser = identity.getPrincipal().getName();
        userService.deleteUser(id, currentUser);
        return Response.noContent().build();
    }

    @GET
    @Path("/health")
    @PermitAll
    public Response healthCheck() {
        return Response.ok("{\"status\":\"UP\"}").build();
    }
}
```

### application.properties (security config)

```properties
# Basic auth for this example
quarkus.http.auth.basic=true

# Embedded users (replace with quarkus-oidc or quarkus-elytron-security-jdbc in production)
quarkus.security.users.embedded.enabled=true
quarkus.security.users.embedded.plain-text=true
quarkus.security.users.embedded.users.admin=admin123
quarkus.security.users.embedded.users.manager1=mgr123
quarkus.security.users.embedded.users.user1=user123
quarkus.security.users.embedded.roles.admin=ADMIN,USER
quarkus.security.users.embedded.roles.manager1=MANAGER,USER
quarkus.security.users.embedded.roles.user1=USER
```

---

## Example 4: JAX-WS @WebService → quarkus-cxf

### Before (JavaEE — JAX-WS SOAP Service)

```java
package com.example.soap;

import javax.ejb.Stateless;
import javax.jws.WebMethod;
import javax.jws.WebParam;
import javax.jws.WebResult;
import javax.jws.WebService;
import javax.jws.soap.SOAPBinding;
import javax.xml.ws.WebServiceException;
import javax.ejb.EJB;

@WebService(
    serviceName = "InventoryService",
    portName = "InventoryPort",
    targetNamespace = "http://example.com/inventory"
)
@SOAPBinding(style = SOAPBinding.Style.DOCUMENT)
@Stateless
public class InventoryWebService {

    @EJB
    private InventoryRepository inventoryRepository;

    @WebMethod(operationName = "checkStock")
    @WebResult(name = "stockLevel")
    public int checkStock(
            @WebParam(name = "productId") String productId,
            @WebParam(name = "warehouseId") String warehouseId) {

        if (productId == null || productId.isEmpty()) {
            throw new WebServiceException("productId is required");
        }

        return inventoryRepository.getStockLevel(productId, warehouseId);
    }

    @WebMethod(operationName = "reserveStock")
    @WebResult(name = "reservationId")
    public String reserveStock(
            @WebParam(name = "productId") String productId,
            @WebParam(name = "quantity") int quantity,
            @WebParam(name = "orderId") String orderId) {

        if (quantity <= 0) {
            throw new WebServiceException("Quantity must be positive");
        }

        return inventoryRepository.reserve(productId, quantity, orderId);
    }

    @WebMethod(operationName = "releaseReservation")
    public void releaseReservation(
            @WebParam(name = "reservationId") String reservationId) {

        inventoryRepository.release(reservationId);
    }
}
```

### After (Quarkus — CXF Extension)

```java
package com.example.soap;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.jws.WebMethod;
import jakarta.jws.WebParam;
import jakarta.jws.WebResult;
import jakarta.jws.WebService;
import jakarta.jws.soap.SOAPBinding;
import jakarta.xml.ws.WebServiceException;

@WebService(
    serviceName = "InventoryService",
    portName = "InventoryPort",
    targetNamespace = "http://example.com/inventory"
)
@SOAPBinding(style = SOAPBinding.Style.DOCUMENT)
@ApplicationScoped
public class InventoryWebService {

    @Inject
    InventoryRepository inventoryRepository;

    @WebMethod(operationName = "checkStock")
    @WebResult(name = "stockLevel")
    public int checkStock(
            @WebParam(name = "productId") String productId,
            @WebParam(name = "warehouseId") String warehouseId) {

        if (productId == null || productId.isEmpty()) {
            throw new WebServiceException("productId is required");
        }

        return inventoryRepository.getStockLevel(productId, warehouseId);
    }

    @WebMethod(operationName = "reserveStock")
    @WebResult(name = "reservationId")
    public String reserveStock(
            @WebParam(name = "productId") String productId,
            @WebParam(name = "quantity") int quantity,
            @WebParam(name = "orderId") String orderId) {

        if (quantity <= 0) {
            throw new WebServiceException("Quantity must be positive");
        }

        return inventoryRepository.reserve(productId, quantity, orderId);
    }

    @WebMethod(operationName = "releaseReservation")
    public void releaseReservation(
            @WebParam(name = "reservationId") String reservationId) {

        inventoryRepository.release(reservationId);
    }
}
```

### application.properties

```properties
# CXF SOAP endpoint configuration
quarkus.cxf.endpoint."/inventory".implementor=com.example.soap.InventoryWebService
quarkus.cxf.endpoint."/inventory".features=org.apache.cxf.ext.logging.LoggingFeature

# WSDL available at: http://localhost:8080/services/inventory?wsdl
quarkus.cxf.path=/services
```

**Key changes**: `@Stateless` → `@ApplicationScoped`, `@EJB` → `@Inject`, `javax.jws.*` → `jakarta.jws.*`. The `@WebService`/`@WebMethod` annotations remain identical.

---

## Example 5: JSF Managed Bean → CDI Bean + Qute TemplateInstance

### Before (JavaEE — JSF Backing Bean + XHTML)

```java
package com.example.web;

import javax.annotation.PostConstruct;
import javax.faces.application.FacesMessage;
import javax.faces.context.FacesContext;
import javax.faces.view.ViewScoped;
import javax.inject.Inject;
import javax.inject.Named;
import java.io.Serializable;
import java.util.List;

@Named("productBean")
@ViewScoped
public class ProductBean implements Serializable {

    private static final long serialVersionUID = 1L;

    @Inject
    private ProductService productService;

    private List<Product> products;
    private Product selectedProduct;
    private String searchTerm;

    @PostConstruct
    public void init() {
        products = productService.findAll();
    }

    public String search() {
        if (searchTerm != null && !searchTerm.isEmpty()) {
            products = productService.searchByName(searchTerm);
        } else {
            products = productService.findAll();
        }
        return null; // stay on same page
    }

    public String viewProduct(Long productId) {
        selectedProduct = productService.findById(productId);
        if (selectedProduct == null) {
            FacesContext.getCurrentInstance().addMessage(null,
                new FacesMessage(FacesMessage.SEVERITY_ERROR, "Product not found", null));
            return null;
        }
        return "product-detail?faces-redirect=true&id=" + productId;
    }

    public String deleteProduct(Long productId) {
        productService.delete(productId);
        products = productService.findAll();
        FacesContext.getCurrentInstance().addMessage(null,
            new FacesMessage(FacesMessage.SEVERITY_INFO, "Product deleted", null));
        return null;
    }

    // Getters and setters
    public List<Product> getProducts() { return products; }
    public void setProducts(List<Product> products) { this.products = products; }
    public String getSearchTerm() { return searchTerm; }
    public void setSearchTerm(String searchTerm) { this.searchTerm = searchTerm; }
    public Product getSelectedProduct() { return selectedProduct; }
}
```

**JSF Template — `src/main/webapp/products.xhtml`:**
```xml
<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:h="http://xmlns.jcp.org/jsf/html"
      xmlns:f="http://xmlns.jcp.org/jsf/core">
<h:body>
    <h:form>
        <h:inputText value="#{productBean.searchTerm}" />
        <h:commandButton value="Search" action="#{productBean.search}" />
    </h:form>
    <h:dataTable value="#{productBean.products}" var="p">
        <h:column><f:facet name="header">Name</f:facet>#{p.name}</h:column>
        <h:column><f:facet name="header">Price</f:facet>#{p.price}</h:column>
        <h:column>
            <h:link value="View" outcome="product-detail"><f:param name="id" value="#{p.id}"/></h:link>
            <h:commandButton value="Delete" action="#{productBean.deleteProduct(p.id)}" />
        </h:column>
    </h:dataTable>
</h:body>
</html>
```

### After (Quarkus — JAX-RS Resource + Qute Template)

```java
package com.example.web;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import io.quarkus.qute.Template;
import io.quarkus.qute.TemplateInstance;
import java.net.URI;
import java.util.List;

@Path("/products")
@ApplicationScoped
public class ProductResource {

    @Inject
    ProductService productService;

    @Inject
    Template products;  // injects templates/products.html

    @Inject
    Template productDetail;  // injects templates/productDetail.html

    @GET
    @Produces(MediaType.TEXT_HTML)
    public TemplateInstance listProducts(@QueryParam("search") String searchTerm) {
        List<Product> productList;
        if (searchTerm != null && !searchTerm.isEmpty()) {
            productList = productService.searchByName(searchTerm);
        } else {
            productList = productService.findAll();
        }
        return products
            .data("products", productList)
            .data("searchTerm", searchTerm);
    }

    @GET
    @Path("/{id}")
    @Produces(MediaType.TEXT_HTML)
    public Response viewProduct(@PathParam("id") Long id) {
        Product product = productService.findById(id);
        if (product == null) {
            return Response.status(Response.Status.NOT_FOUND)
                .entity(products.data("products", productService.findAll())
                    .data("error", "Product not found")
                    .data("searchTerm", null))
                .build();
        }
        return Response.ok(productDetail.data("product", product)).build();
    }

    @POST
    @Path("/{id}/delete")
    @Produces(MediaType.TEXT_HTML)
    public Response deleteProduct(@PathParam("id") Long id) {
        productService.delete(id);
        return Response.seeOther(URI.create("/products")).build();
    }
}
```

**Qute Template — `src/main/resources/templates/products.html`:**
```html
<!DOCTYPE html>
<html>
<head><title>Products</title></head>
<body>
    <h1>Products</h1>

    {#if error}
    <div class="error">{error}</div>
    {/if}

    <form action="/products" method="GET">
        <input type="text" name="search" value="{searchTerm ?: ''}" placeholder="Search..." />
        <button type="submit">Search</button>
    </form>

    <table>
        <thead>
            <tr><th>Name</th><th>Price</th><th>Actions</th></tr>
        </thead>
        <tbody>
            {#for p in products}
            <tr>
                <td>{p.name}</td>
                <td>{p.price}</td>
                <td>
                    <a href="/products/{p.id}">View</a>
                    <form action="/products/{p.id}/delete" method="POST" style="display:inline">
                        <button type="submit">Delete</button>
                    </form>
                </td>
            </tr>
            {#else}
            <tr><td colspan="3">No products found</td></tr>
            {/for}
        </tbody>
    </table>
</body>
</html>
```

---

## Example 6: Arquillian Integration Test → @QuarkusTest

### Before (Arquillian)

```java
package com.example.test;

import com.example.api.OrderResource;
import com.example.model.Order;
import com.example.service.OrderService;
import org.jboss.arquillian.container.test.api.Deployment;
import org.jboss.arquillian.container.test.api.RunAsClient;
import org.jboss.arquillian.junit.Arquillian;
import org.jboss.arquillian.junit.InSequence;
import org.jboss.arquillian.test.api.ArquillianResource;
import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.asset.EmptyAsset;
import org.jboss.shrinkwrap.api.spec.WebArchive;
import org.junit.Assert;
import org.junit.Test;
import org.junit.runner.RunWith;

import javax.inject.Inject;
import javax.ws.rs.client.Client;
import javax.ws.rs.client.ClientBuilder;
import javax.ws.rs.client.Entity;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.Response;
import java.net.URL;

@RunWith(Arquillian.class)
public class OrderResourceIT {

    @Deployment
    public static WebArchive createDeployment() {
        return ShrinkWrap.create(WebArchive.class, "order-test.war")
            .addPackages(true, "com.example")
            .addAsResource("META-INF/persistence.xml")
            .addAsResource("test-data.sql", "import.sql")
            .addAsWebInfResource(EmptyAsset.INSTANCE, "beans.xml")
            .addAsWebInfResource("jboss-web.xml");
    }

    @ArquillianResource
    private URL deploymentUrl;

    @Inject
    private OrderService orderService;

    private static Long createdOrderId;

    @Test
    @InSequence(1)
    public void testServiceInjection() {
        Assert.assertNotNull("OrderService should be injected", orderService);
    }

    @Test
    @InSequence(2)
    @RunAsClient
    public void testCreateOrder() throws Exception {
        Client client = ClientBuilder.newClient();
        try {
            String json = "{\"itemId\":\"WIDGET-001\",\"quantity\":5,\"customerId\":\"CUST-123\"}";

            Response response = client.target(deploymentUrl.toURI())
                .path("/api/orders")
                .request(MediaType.APPLICATION_JSON)
                .post(Entity.json(json));

            Assert.assertEquals("Should return 201", 201, response.getStatus());

            Order order = response.readEntity(Order.class);
            Assert.assertNotNull("Order should have an ID", order.getId());
            Assert.assertEquals("PENDING", order.getStatus());
            createdOrderId = order.getId();
        } finally {
            client.close();
        }
    }

    @Test
    @InSequence(3)
    @RunAsClient
    public void testGetOrder() throws Exception {
        Client client = ClientBuilder.newClient();
        try {
            Response response = client.target(deploymentUrl.toURI())
                .path("/api/orders/" + createdOrderId)
                .request(MediaType.APPLICATION_JSON)
                .get();

            Assert.assertEquals("Should return 200", 200, response.getStatus());

            Order order = response.readEntity(Order.class);
            Assert.assertEquals(createdOrderId, order.getId());
            Assert.assertEquals("WIDGET-001", order.getItemId());
        } finally {
            client.close();
        }
    }

    @Test
    @InSequence(4)
    @RunAsClient
    public void testGetNonExistentOrder() throws Exception {
        Client client = ClientBuilder.newClient();
        try {
            Response response = client.target(deploymentUrl.toURI())
                .path("/api/orders/99999")
                .request(MediaType.APPLICATION_JSON)
                .get();

            Assert.assertEquals("Should return 404", 404, response.getStatus());
        } finally {
            client.close();
        }
    }

    @Test
    @InSequence(5)
    @RunAsClient
    public void testListOrders() throws Exception {
        Client client = ClientBuilder.newClient();
        try {
            Response response = client.target(deploymentUrl.toURI())
                .path("/api/orders")
                .request(MediaType.APPLICATION_JSON)
                .get();

            Assert.assertEquals("Should return 200", 200, response.getStatus());
            String body = response.readEntity(String.class);
            Assert.assertTrue("Should contain at least one order", body.contains("WIDGET-001"));
        } finally {
            client.close();
        }
    }
}
```

### After (Quarkus — @QuarkusTest + REST Assured)

```java
package com.example.test;

import com.example.service.OrderService;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.TestTransaction;
import jakarta.inject.Inject;
import org.junit.jupiter.api.MethodOrderer;
import org.junit.jupiter.api.Order;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.TestMethodOrder;

import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
public class OrderResourceIT {

    // No @Deployment — Quarkus manages the full app lifecycle
    // No @ArquillianResource URL — REST Assured auto-configures

    @Inject
    OrderService orderService;

    static Long createdOrderId;

    @Test
    @Order(1)
    public void testServiceInjection() {
        assertNotNull(orderService, "OrderService should be injected");
    }

    @Test
    @Order(2)
    public void testCreateOrder() {
        createdOrderId = given()
            .contentType("application/json")
            .body("""
                {"itemId":"WIDGET-001","quantity":5,"customerId":"CUST-123"}
                """)
            .when()
            .post("/api/orders")
            .then()
            .statusCode(201)
            .body("id", notNullValue())
            .body("status", equalTo("PENDING"))
            .body("itemId", equalTo("WIDGET-001"))
            .extract()
            .jsonPath().getLong("id");
    }

    @Test
    @Order(3)
    public void testGetOrder() {
        given()
            .when()
            .get("/api/orders/" + createdOrderId)
            .then()
            .statusCode(200)
            .body("id", equalTo(createdOrderId.intValue()))
            .body("itemId", equalTo("WIDGET-001"))
            .body("quantity", equalTo(5));
    }

    @Test
    @Order(4)
    public void testGetNonExistentOrder() {
        given()
            .when()
            .get("/api/orders/99999")
            .then()
            .statusCode(404);
    }

    @Test
    @Order(5)
    public void testListOrders() {
        given()
            .when()
            .get("/api/orders")
            .then()
            .statusCode(200)
            .body("size()", greaterThanOrEqualTo(1))
            .body("itemId", hasItem("WIDGET-001"));
    }
}
```

### Test application.properties (`src/test/resources/application.properties`)

```properties
# Test datasource (replaces persistence.xml + test-data.sql in @Deployment)
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
quarkus.hibernate-orm.database.generation=drop-and-create
quarkus.hibernate-orm.sql-load-script=import.sql
```

**Key changes summary**:
- `@RunWith(Arquillian.class)` → `@QuarkusTest`
- `@Deployment` ShrinkWrap → DELETED (no deployment descriptor needed)
- `@ArquillianResource URL` + JAX-RS Client → REST Assured (auto-configured URL)
- `@InSequence(n)` → `@TestMethodOrder` + `@Order(n)`
- `Assert.assertEquals(msg, expected, actual)` → `assertEquals(expected, actual)` or Hamcrest matchers
- `@RunAsClient` → default behavior (REST Assured is always a client)
- Test data in `@Deployment` → `import.sql` or `@TestTransaction` + `@BeforeEach`
