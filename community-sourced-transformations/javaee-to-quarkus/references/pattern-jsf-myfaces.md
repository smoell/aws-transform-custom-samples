# JSF → MyFaces Quarkus Extension Pattern

## Overview
The Apache MyFaces Quarkus Extension provides JSF 4.0 compatibility for Quarkus applications, enabling minimal-change migration of existing JSF applications while gaining Quarkus benefits.

## Use Case
- Preserve existing JSF applications with minimal code changes
- Maintain JSF development patterns and team knowledge
- Migrate to Quarkus without UI rewrite
- Bridge solution while planning long-term modernization

## Dependencies

### Required Extensions
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

### Optional JSF Components
```xml
<!-- For PrimeFaces components -->
<dependency>
    <groupId>org.primefaces</groupId>
    <artifactId>primefaces</artifactId>
    <version>12.0.0</version>
</dependency>

<!-- For OmniFaces utilities -->
<dependency>
    <groupId>org.omnifaces</groupId>
    <artifactId>omnifaces</artifactId>
    <version>4.0</version>
</dependency>
```

## Resource Migration

### Web Resources Location
Move all web resources from WAR structure to Quarkus resource layout:

```
# Before (WAR)
src/main/webapp/
  ├── index.xhtml
  ├── WEB-INF/
  │   ├── web.xml
  │   └── faces-config.xml
  └── resources/
      ├── css/
      └── js/

# After (Quarkus)
src/main/resources/META-INF/resources/
  ├── index.xhtml
  └── resources/
      ├── css/
      └── js/
src/main/resources/META-INF/
  └── faces-config.xml
src/main/resources/META-INF/
  └── web.xml (minimal)
```

### Minimal web.xml
Create a minimal `web.xml` for FacesServlet mapping:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee 
         https://jakarta.ee/xml/ns/jakartaee/web-app_5_0.xsd"
         version="5.0">
    
    <servlet>
        <servlet-name>FacesServlet</servlet-name>
        <servlet-class>jakarta.faces.webapp.FacesServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>FacesServlet</servlet-name>
        <url-pattern>*.jsf</url-pattern>
    </servlet-mapping>
    <servlet-mapping>
        <servlet-name>FacesServlet</servlet-name>
        <url-pattern>/faces/*</url-pattern>
    </servlet-mapping>
    
    <!-- JSF Configuration -->
    <context-param>
        <param-name>jakarta.faces.PROJECT_STAGE</param-name>
        <param-value>Production</param-value>
    </context-param>
    <context-param>
        <param-name>jakarta.faces.FACELETS_SKIP_COMMENTS</param-name>
        <param-value>true</param-value>
    </context-param>
</web-app>
```

## Namespace Migration

### Critical: Jakarta EE Namespace
Ensure ALL JSF templates use correct Jakarta namespace URIs:

```xml
<!-- CORRECT -->
xmlns:h="jakarta.faces.html"
xmlns:f="jakarta.faces.core" 
xmlns:ui="jakarta.faces.facelets"
xmlns:c="jakarta.tags.core"

<!-- WRONG - Will cause silent failures -->
xmlns:h="jakarta.face.html"    <!-- Missing 's' -->
xmlns:f="javax.faces.core"     <!-- Old namespace -->
```

**Critical Check**: Run this validation before proceeding:
```bash
grep -rn "jakarta\.face\." src/ | grep -v "jakarta\.faces\."
```
Must return ZERO results. Any matches will cause JSF components to fail silently.

### faces-config.xml Namespace
Update to Jakarta EE 4.0 schema:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<faces-config xmlns="https://jakarta.ee/xml/ns/jakartaee"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee 
              https://jakarta.ee/xml/ns/jakartaee/web-facesconfig_4_0.xsd"
              version="4.0">
    
    <!-- Your JSF configuration -->
</faces-config>
```

## CDI Integration

### Managed Bean Migration
JSF managed beans integrate seamlessly with ArC CDI:

```java
// Before: JSF Managed Bean
@ManagedBean
@SessionScoped
public class UserController implements Serializable {
    private String username;
    // ...
}

// After: CDI Bean (works with MyFaces)
@Named
@SessionScoped  
public class UserController implements Serializable {
    private String username;
    // ...
}
```

### Scopes Mapping
JSF scopes work with ArC proxies:

| JSF Scope | CDI Scope | Notes |
|-----------|-----------|-------|
| `@RequestScoped` | `@RequestScoped` | Direct mapping |
| `@ViewScoped` | `@ViewScoped` | MyFaces provides this |
| `@SessionScoped` | `@SessionScoped` | Serializable required |
| `@ApplicationScoped` | `@ApplicationScoped` | Singleton behavior |
| `@ConversationScoped` | `@ConversationScoped` | For multi-page flows |

### Injection Points
CDI injection works in JSF backing beans:

```java
@Named
@RequestScoped
public class OrderController {
    
    @Inject
    OrderService orderService;  // CDI injection
    
    @Inject
    EntityManager em;           // JPA injection
    
    @ConfigProperty("app.order.max-items")
    int maxItems;               // Config injection
    
    public void processOrder() {
        Order order = orderService.createOrder();
        // Process with injected services
    }
}
```

## Packaging Changes

### From WAR to JAR
Update Maven packaging:

```xml
<!-- Before -->
<packaging>war</packaging>

<!-- After -->  
<packaging>jar</packaging>

<!-- Remove WAR plugin -->
<!-- <plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-war-plugin</artifactId>
</plugin> -->

<!-- Add Quarkus plugin -->
<plugin>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-maven-plugin</artifactId>
    <version>${quarkus.platform.version}</version>
    <extensions>true</extensions>
    <executions>
        <execution>
            <goals>
                <goal>build</goal>
                <goal>generate-code</goal>
                <goal>generate-code-tests</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

## Configuration

### JSF Parameters
MyFaces configuration uses standard JSF context parameters in `web.xml`, NOT `application.properties`:

```xml
<!-- In web.xml, not application.properties -->
<context-param>
    <param-name>jakarta.faces.PROJECT_STAGE</param-name>
    <param-value>Development</param-value>
</context-param>
<context-param>
    <param-name>jakarta.faces.FACELETS_DEVELOPMENT</param-name>
    <param-value>true</param-value>
</context-param>
<context-param>
    <param-name>org.apache.myfaces.REFRESH_TRANSIENT_BUILD_ON_PSS</param-name>
    <param-value>auto</param-value>
</context-param>
```

**Note**: Quarkus-style `quarkus.myfaces.*` properties are NOT supported.

### Undertow Configuration
Configure Undertow via `application.properties`:

```properties
# HTTP configuration
quarkus.http.port=8080
quarkus.http.host=0.0.0.0

# Session configuration
quarkus.http.session.timeout=30M

# Static resource caching
quarkus.http.static-resources."/*".caching-enabled=true
```

## Migration Steps

### Step 1: Add Dependencies
Add MyFaces and Undertow extensions to your project.

### Step 2: Move Resources
Relocate web resources from `src/main/webapp/` to `src/main/resources/META-INF/resources/`.

### Step 3: Update Namespaces
- Fix all Jakarta namespace URIs in templates
- Update faces-config.xml schema
- Run validation check for namespace typos

### Step 4: Migrate Managed Beans
Convert `@ManagedBean` to `@Named` with appropriate CDI scopes.

### Step 5: Configure Servlet Mapping
Create minimal web.xml with FacesServlet mappings.

### Step 6: Update Build Configuration
Change packaging to JAR and configure Quarkus plugin.

### Step 7: Test SessionScoped Functionality
Verify session-scoped beans work correctly with Undertow sessions.

## Testing

### Development Mode
```bash
./mvnw quarkus:dev
```

Access JSF pages at `http://localhost:8080/faces/yourpage.jsf`

### Integration Tests
```java
@QuarkusTest
public class JSFIntegrationTest {
    
    @Test
    public void testJsfPageRenders() {
        given()
            .when().get("/faces/index.jsf")
            .then()
                .statusCode(200)
                .body(containsString("JSF Content"));
    }
}
```

## Limitations

### GraalVM Native
- **No native compilation support** for JSF applications
- MyFaces uses extensive runtime reflection
- Only JVM mode is supported
- Use `src/main/docker/Dockerfile.jvm` only

### Performance
- Larger memory footprint compared to pure Quarkus REST
- Slower startup than Qute-based solutions
- JSF lifecycle adds processing overhead

### Cloud-Native Features
- Limited reactive programming support
- Stateful session management complicates horizontal scaling
- Server-side rendering less suitable for microservices

## Best Practices

### Migration Strategy
1. Use MyFaces extension as a bridge solution
2. Plan eventual migration to Qute or modern frontend
3. Avoid adding new JSF components during migration
4. Minimize PrimeFaces/RichFaces dependencies

### Session Management
```java
// Minimize session state
@Named
@ViewScoped  // Prefer ViewScoped over SessionScoped
public class OrderController implements Serializable {
    // Keep state minimal and serializable
}
```

### Error Handling
```java
@Named
@ApplicationScoped
public class GlobalExceptionHandler {
    
    public void handleException(ComponentSystemEvent event) {
        // Custom error handling
    }
}
```

## Troubleshooting

### Common Issues

**Empty Pages/No JSF Components**
- Check namespace URIs (jakarta.faces.*, not jakarta.face.*)
- Verify FacesServlet mapping in web.xml
- Ensure resources are in META-INF/resources/

**CDI Injection Failures**
- Add no-arg constructor to normal-scoped beans
- Verify proper CDI scope annotations
- Check for serializable requirements on session-scoped beans

**Build Failures**
- Ensure JAR packaging (not WAR)
- Remove WAR plugin from pom.xml
- Verify Quarkus plugin configuration

**Performance Issues**
- Review JSF PROJECT_STAGE setting
- Minimize view state size
- Consider client-side state saving

This pattern provides a low-risk migration path for JSF applications while maintaining functional parity and allowing teams to migrate to Quarkus without rewriting their user interface.