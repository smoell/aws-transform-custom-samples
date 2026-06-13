# JSF Migration Patterns

## Option 1: JSF Preservation with MyFaces

### Use Case
Preserve existing JSF applications with minimal changes while gaining Quarkus benefits.

### Dependencies
```xml
<dependency>
    <groupId>org.apache.myfaces.core.extensions.quarkus</groupId>
    <artifactId>myfaces-quarkus</artifactId>
    <version>4.0.2</version>
</dependency>
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-undertow</artifactId>
</dependency>
```

### Configuration
```properties
# application.properties
quarkus.myfaces.build-time-module-validation=false
javax.faces.STATE_SAVING_METHOD=server
javax.faces.DEFAULT_SUFFIX=.xhtml
```

### Backing Bean Migration
```java
// Before: JSF @ManagedBean
@ManagedBean
@ViewScoped
public class CustomerController implements Serializable {
    // JSF lifecycle methods remain unchanged
}

// After: CDI Bean (minimal changes)
@Named
@ViewScoped  
public class CustomerController implements Serializable {
    // Same JSF lifecycle methods work
}
```

## Option 2: JSF → Qute Template Migration

### Architecture Differences
- **JSF**: Server-side component tree, stateful backing beans, complex lifecycle
- **Qute**: Stateless template rendering, JAX-RS endpoints, build-time compilation

### Core Component Mappings

#### Form Submission
```html
<!-- Before: JSF -->
<h:form>
    <h:inputText value="#{customer.name}" />
    <h:commandButton action="#{customerController.save}" />
</h:form>

<!-- After: Qute + JAX-RS -->
<form action="/customers" method="post">
    <input name="name" value="{customer.name}" />
    <button type="submit">Save</button>
</form>
```

#### Data Table
```html
<!-- Before: JSF -->
<h:dataTable value="#{customers}" var="c">
    <h:column>
        <h:outputText value="#{c.name}" />
    </h:column>
</h:dataTable>

<!-- After: Qute -->
<table>
{#for customer in customers}
    <tr><td>{customer.name}</td></tr>
{/for}
</table>
```

### REST Controller Pattern
```java
// Replace JSF backing bean with JAX-RS
@Path("/customers")
public class CustomerResource {
    @GET
    @Produces(MediaType.TEXT_HTML)
    public TemplateInstance list() {
        return customerList.data("customers", Customer.listAll());
    }
    
    @POST
    @Consumes(MediaType.APPLICATION_FORM_URLENCODED)
    public Response create(@FormParam("name") String name) {
        Customer.persist(new Customer(name));
        return Response.seeOther(URI.create("/customers")).build();
    }
}
```