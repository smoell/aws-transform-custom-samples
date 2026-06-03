# JavaEE Security → Quarkus Security Reference

> Reference for Phase 3 Step 12: Security migration (conditional on SECURITY_NEEDED flag).
> See also: https://quarkus.io/guides/security-overview

## Annotation Mapping

| JavaEE/JakartaEE Annotation | Quarkus Equivalent | Action |
|---|---|---|
| `@RolesAllowed({"admin","user"})` | `@RolesAllowed({"admin","user"})` | NO CHANGE — same annotation from `jakarta.annotation.security` |
| `@PermitAll` | `@PermitAll` | NO CHANGE — same annotation |
| `@DenyAll` | `@DenyAll` | NO CHANGE — same annotation |
| `@DeclareRoles({"admin","user"})` | REMOVE | Quarkus does not require role pre-declaration |
| `@RunAs("SYSTEM")` | `SecurityIdentity` augmentation | Implement `SecurityIdentityAugmentor` (see below) |
| `@RolesAllowed` on EJB | `@RolesAllowed` on CDI bean/JAX-RS resource | Same annotation, same package — just works |

### Important Notes

- `@RolesAllowed`, `@PermitAll`, `@DenyAll` work on both JAX-RS resources (`@Path` classes) and CDI beans (`@ApplicationScoped` classes) in Quarkus.
- Role checking is performed by the Quarkus security subsystem at runtime — the bean does not need to be aware of which identity provider is in use.
- To enable annotation-based security on CDI beans (non-JAX-RS), no additional configuration is needed in Quarkus (unlike Spring which requires `@EnableMethodSecurity`).

## web.xml security-constraint → application.properties

### Before — web.xml

```xml
<web-app>
    <security-constraint>
        <web-resource-collection>
            <web-resource-name>Admin Area</web-resource-name>
            <url-pattern>/admin/*</url-pattern>
            <http-method>GET</http-method>
            <http-method>POST</http-method>
            <http-method>PUT</http-method>
            <http-method>DELETE</http-method>
        </web-resource-collection>
        <auth-constraint>
            <role-name>ADMIN</role-name>
        </auth-constraint>
    </security-constraint>

    <security-constraint>
        <web-resource-collection>
            <web-resource-name>API Access</web-resource-name>
            <url-pattern>/api/*</url-pattern>
        </web-resource-collection>
        <auth-constraint>
            <role-name>USER</role-name>
            <role-name>ADMIN</role-name>
        </auth-constraint>
    </security-constraint>

    <security-constraint>
        <web-resource-collection>
            <web-resource-name>Public Resources</web-resource-name>
            <url-pattern>/public/*</url-pattern>
            <url-pattern>/health</url-pattern>
        </web-resource-collection>
        <!-- No auth-constraint = permit all -->
    </security-constraint>

    <login-config>
        <auth-method>BASIC</auth-method>
        <realm-name>MyApp Realm</realm-name>
    </login-config>

    <security-role>
        <role-name>ADMIN</role-name>
    </security-role>
    <security-role>
        <role-name>USER</role-name>
    </security-role>
</web-app>
```

### After — application.properties

```properties
# Authentication mechanism
quarkus.http.auth.basic=true

# Policy: admin-only
quarkus.http.auth.policy.admin-policy.roles-allowed=ADMIN

# Policy: authenticated users (USER or ADMIN)
quarkus.http.auth.policy.user-policy.roles-allowed=USER,ADMIN

# Policy: public access (no authentication needed)
quarkus.http.auth.policy.public-policy.roles-allowed=**

# Permission: /admin/* → requires ADMIN role
quarkus.http.auth.permission.admin-area.paths=/admin/*
quarkus.http.auth.permission.admin-area.policy=admin-policy
quarkus.http.auth.permission.admin-area.methods=GET,POST,PUT,DELETE

# Permission: /api/* → requires USER or ADMIN role
quarkus.http.auth.permission.api-access.paths=/api/*
quarkus.http.auth.permission.api-access.policy=user-policy

# Permission: /public/*, /health → public access
quarkus.http.auth.permission.public-resources.paths=/public/*,/health
quarkus.http.auth.permission.public-resources.policy=permit
# (built-in "permit" policy — no custom definition needed)

# Permission: Quarkus management endpoints
quarkus.http.auth.permission.health.paths=/q/*
quarkus.http.auth.permission.health.policy=permit
```

### Policy Mapping Rules

| web.xml Pattern | application.properties |
|---|---|
| `<auth-constraint>` with roles | `quarkus.http.auth.policy.<name>.roles-allowed=ROLE1,ROLE2` |
| No `<auth-constraint>` (permit all) | `.policy=permit` (built-in) |
| Empty `<auth-constraint>` (deny all) | `.policy=deny` (built-in) |
| `<http-method>` restrictions | `.methods=GET,POST` |
| `<auth-method>BASIC</auth-method>` | `quarkus.http.auth.basic=true` |
| `<auth-method>FORM</auth-method>` | `quarkus.http.auth.form.enabled=true` |
| `<transport-guarantee>CONFIDENTIAL` | `quarkus.http.ssl.certificate.*` + `quarkus.http.insecure-requests=redirect` |

### Form-Based Authentication

```properties
# Replaces <auth-method>FORM</auth-method> + <form-login-config>
quarkus.http.auth.form.enabled=true
quarkus.http.auth.form.login-page=/login.html
quarkus.http.auth.form.error-page=/login-error.html
quarkus.http.auth.form.landing-page=/index.html
quarkus.http.auth.form.timeout=PT30M
quarkus.http.auth.form.cookie-name=quarkus-credential
```

## Identity Provider Patterns

### JDBC Realm (Database Users)

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-elytron-security-jdbc</artifactId>
</dependency>
```

```properties
# JDBC identity provider configuration
quarkus.datasource.db-kind=postgresql
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/myapp

quarkus.security.jdbc.enabled=true
quarkus.security.jdbc.principal-query.sql=SELECT password, role FROM users WHERE username=?
quarkus.security.jdbc.principal-query.clear-text-password.password-index=1
quarkus.security.jdbc.principal-query.attribute-mappings.0.index=2
quarkus.security.jdbc.principal-query.attribute-mappings.0.to=groups
```

For bcrypt-hashed passwords:
```properties
quarkus.security.jdbc.principal-query.sql=SELECT password FROM users WHERE username=?
quarkus.security.jdbc.principal-query.bcrypt-password.password-index=1
quarkus.security.jdbc.principal-query.bcrypt-password.iteration-count-index=2
quarkus.security.jdbc.principal-query.bcrypt-password.salt-index=3
```

### LDAP

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-elytron-security-ldap</artifactId>
</dependency>
```

```properties
quarkus.security.ldap.enabled=true
quarkus.security.ldap.dir-context.url=ldap://ldap.example.com:389
quarkus.security.ldap.dir-context.principal=cn=admin,dc=example,dc=com
quarkus.security.ldap.dir-context.password=admin_password

quarkus.security.ldap.identity-mapping.rdn-identifier=uid
quarkus.security.ldap.identity-mapping.search-base-dn=ou=users,dc=example,dc=com

quarkus.security.ldap.identity-mapping.attribute-mappings.0.from=cn
quarkus.security.ldap.identity-mapping.attribute-mappings.0.to=groups
quarkus.security.ldap.identity-mapping.attribute-mappings.0.filter=(member=uid={0},ou=users,dc=example,dc=com)
quarkus.security.ldap.identity-mapping.attribute-mappings.0.filter-base-dn=ou=groups,dc=example,dc=com
```

### OIDC (Keycloak, Auth0, Okta, Cognito)

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-oidc</artifactId>
</dependency>
```

```properties
# Keycloak example
quarkus.oidc.auth-server-url=https://keycloak.example.com/realms/myapp
quarkus.oidc.client-id=my-app-client
quarkus.oidc.credentials.secret=client-secret-here

# For service-to-service (bearer token only):
quarkus.oidc.application-type=service

# For web apps (authorization code flow):
quarkus.oidc.application-type=web-app
quarkus.oidc.roles.source=realm
```

Auth0 example:
```properties
quarkus.oidc.auth-server-url=https://your-tenant.auth0.com/
quarkus.oidc.client-id=your-client-id
quarkus.oidc.credentials.secret=your-client-secret
quarkus.oidc.application-type=service
quarkus.oidc.token.audience=https://your-api-identifier
```

### Properties File (Dev/Test Only)

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-elytron-security-properties-file</artifactId>
</dependency>
```

```properties
# Embedded users for development/testing
quarkus.security.users.embedded.enabled=true
quarkus.security.users.embedded.plain-text=true
quarkus.security.users.embedded.users.admin=admin123
quarkus.security.users.embedded.users.user1=user123
quarkus.security.users.embedded.roles.admin=ADMIN,USER
quarkus.security.users.embedded.roles.user1=USER
```

### Custom Identity Provider (Replaces JAAS LoginModule)

```java
// BEFORE (JAAS LoginModule)
public class CustomLoginModule implements LoginModule {
    private Subject subject;
    private CallbackHandler callbackHandler;

    @Override
    public void initialize(Subject subject, CallbackHandler handler, ...) {
        this.subject = subject;
        this.callbackHandler = handler;
    }

    @Override
    public boolean login() throws LoginException {
        NameCallback nameCallback = new NameCallback("Username:");
        PasswordCallback passwordCallback = new PasswordCallback("Password:", false);
        callbackHandler.handle(new Callback[]{nameCallback, passwordCallback});

        String username = nameCallback.getName();
        char[] password = passwordCallback.getPassword();
        // validate against database/service...
        return validated;
    }

    @Override
    public boolean commit() throws LoginException {
        subject.getPrincipals().add(new SimplePrincipal(username));
        subject.getPrincipals().add(new SimpleGroup("Roles", "ADMIN"));
        return true;
    }
}

// AFTER (Quarkus IdentityProvider)
import io.quarkus.security.identity.IdentityProvider;
import io.quarkus.security.identity.AuthenticationRequestContext;
import io.quarkus.security.identity.SecurityIdentity;
import io.quarkus.security.identity.request.UsernamePasswordAuthenticationRequest;
import io.quarkus.security.runtime.QuarkusPrincipal;
import io.quarkus.security.runtime.QuarkusSecurityIdentity;
import io.smallrye.mutiny.Uni;

@ApplicationScoped
public class CustomIdentityProvider
        implements IdentityProvider<UsernamePasswordAuthenticationRequest> {

    @Inject
    UserRepository userRepository;

    @Override
    public Class<UsernamePasswordAuthenticationRequest> getRequestType() {
        return UsernamePasswordAuthenticationRequest.class;
    }

    @Override
    public Uni<SecurityIdentity> authenticate(
            UsernamePasswordAuthenticationRequest request,
            AuthenticationRequestContext context) {

        return context.runBlocking(() -> {
            String username = request.getUsername();
            String password = new String(request.getPassword().getPassword());

            // Validate against database (replaces LoginModule.login())
            UserEntity user = userRepository.findByUsername(username);
            if (user == null || !BCrypt.checkpw(password, user.getPasswordHash())) {
                throw new AuthenticationFailedException("Invalid credentials");
            }

            // Build identity (replaces LoginModule.commit())
            return QuarkusSecurityIdentity.builder()
                .setPrincipal(new QuarkusPrincipal(username))
                .addRoles(user.getRoles())  // Set<String> of role names
                .build();
        });
    }
}
```

## Programmatic Security Migration

### EJBContext / SessionContext → SecurityContext

```java
// BEFORE (JavaEE — EJB/SessionContext)
import javax.annotation.Resource;
import javax.ejb.SessionContext;
import javax.ejb.Stateless;

@Stateless
public class AccountService {
    @Resource
    private SessionContext ctx;

    public void performAction() {
        // Get current user
        String username = ctx.getCallerPrincipal().getName();

        // Check role
        if (ctx.isCallerInRole("ADMIN")) {
            // admin-specific logic
        }
    }
}

// AFTER (Quarkus — SecurityContext or SecurityIdentity)
import jakarta.inject.Inject;
import jakarta.ws.rs.core.SecurityContext;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class AccountService {
    @Inject
    SecurityContext securityContext;

    public void performAction() {
        // Get current user
        String username = securityContext.getUserPrincipal().getName();

        // Check role
        if (securityContext.isUserInRole("ADMIN")) {
            // admin-specific logic
        }
    }
}
```

### Alternative: SecurityIdentity (Quarkus-native API, more features)

```java
import io.quarkus.security.identity.SecurityIdentity;
import jakarta.inject.Inject;

@ApplicationScoped
public class AccountService {
    @Inject
    SecurityIdentity identity;

    public void performAction() {
        // Get current user
        String username = identity.getPrincipal().getName();

        // Check role
        if (identity.hasRole("ADMIN")) {
            // admin-specific logic
        }

        // Check if authenticated at all
        if (identity.isAnonymous()) {
            throw new NotAuthorizedException("Login required");
        }

        // Get all roles
        Set<String> roles = identity.getRoles();

        // Access additional attributes (e.g., from OIDC token)
        String email = identity.getAttribute("email");
    }
}
```

### Mapping Table

| JavaEE Programmatic Security | Quarkus Equivalent |
|---|---|
| `SessionContext.getCallerPrincipal().getName()` | `securityContext.getUserPrincipal().getName()` or `identity.getPrincipal().getName()` |
| `SessionContext.isCallerInRole("ADMIN")` | `securityContext.isUserInRole("ADMIN")` or `identity.hasRole("ADMIN")` |
| `EJBContext.getCallerPrincipal()` | `securityContext.getUserPrincipal()` or `identity.getPrincipal()` |
| `HttpServletRequest.getUserPrincipal()` | `securityContext.getUserPrincipal()` |
| `HttpServletRequest.isUserInRole("X")` | `securityContext.isUserInRole("X")` |
| `HttpServletRequest.login(user, pass)` | Custom `HttpAuthenticationMechanism` (see below) |
| `HttpServletRequest.logout()` | Invalidate session / revoke token |

### HttpServletRequest.login() → Custom Authentication Mechanism

```java
// BEFORE (JavaEE — programmatic login)
@WebServlet("/login")
public class LoginServlet extends HttpServlet {
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) {
        String username = req.getParameter("username");
        String password = req.getParameter("password");
        req.login(username, password);
        resp.sendRedirect("/dashboard");
    }
}

// AFTER (Quarkus — form-based auth handled by framework, OR custom mechanism)
// Option 1: Use built-in form auth (application.properties):
// quarkus.http.auth.form.enabled=true

// Option 2: Custom REST login endpoint
@Path("/auth")
@ApplicationScoped
public class AuthResource {
    @Inject
    SecurityIdentity identity;

    @POST
    @Path("/login")
    @Consumes(MediaType.APPLICATION_FORM_URLENCODED)
    public Response login(@FormParam("username") String username,
                          @FormParam("password") String password) {
        // With basic auth enabled, credentials are validated automatically
        // if Authorization header is present. For custom login flows,
        // implement HttpAuthenticationMechanism.
        if (identity.isAnonymous()) {
            return Response.status(Response.Status.UNAUTHORIZED).build();
        }
        return Response.ok().build();
    }
}
```

### @RunAs → SecurityIdentity Augmentation

```java
// BEFORE (JavaEE — @RunAs for elevated privileges)
@Stateless
@RunAs("SYSTEM")
public class BatchJobService {
    @Resource
    SessionContext ctx;
    // All methods execute with SYSTEM role, regardless of caller
}

// AFTER (Quarkus — SecurityIdentityAugmentor)
import io.quarkus.security.identity.SecurityIdentityAugmentor;
import io.quarkus.security.identity.AuthenticationRequestContext;
import io.quarkus.security.identity.SecurityIdentity;
import io.quarkus.security.runtime.QuarkusSecurityIdentity;
import io.smallrye.mutiny.Uni;

@ApplicationScoped
public class SystemRoleAugmentor implements SecurityIdentityAugmentor {

    @Override
    public Uni<SecurityIdentity> augment(SecurityIdentity identity,
                                          AuthenticationRequestContext context) {
        // Add SYSTEM role to specific service accounts
        if ("batch-service".equals(identity.getPrincipal().getName())) {
            return Uni.createFrom().item(
                QuarkusSecurityIdentity.builder(identity)
                    .addRole("SYSTEM")
                    .build()
            );
        }
        return Uni.createFrom().item(identity);
    }
}
```

For per-method elevation (running a block as a different identity):

```java
@ApplicationScoped
public class BatchJobService {
    @Inject
    SecurityIdentity currentIdentity;

    public void runAsSystem(Runnable action) {
        QuarkusSecurityIdentity systemIdentity = QuarkusSecurityIdentity.builder()
            .setPrincipal(new QuarkusPrincipal("SYSTEM"))
            .addRole("SYSTEM")
            .build();

        // Execute action with elevated identity
        // Note: This requires quarkus-security extension
        currentIdentity.runAs(systemIdentity, action);
    }
}
```

## Secured REST Endpoint — Complete Before/After

### Before (JavaEE — EJB + JAX-RS with Container Security)

```java
// web.xml provides URL-level security
// @DeclareRoles declares valid roles
// @RolesAllowed on EJB methods

import javax.annotation.security.DeclareRoles;
import javax.annotation.security.RolesAllowed;
import javax.annotation.security.PermitAll;
import javax.ejb.Stateless;
import javax.ejb.EJB;
import javax.ws.rs.*;
import javax.ws.rs.core.Context;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.Response;
import javax.ws.rs.core.SecurityContext;

@Path("/orders")
@Stateless
@DeclareRoles({"ADMIN", "USER", "MANAGER"})
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class OrderResource {

    @EJB
    private OrderService orderService;

    @Context
    private SecurityContext securityContext;

    @GET
    @RolesAllowed({"USER", "ADMIN"})
    public Response listOrders() {
        String user = securityContext.getUserPrincipal().getName();
        List<Order> orders;
        if (securityContext.isUserInRole("ADMIN")) {
            orders = orderService.findAll();
        } else {
            orders = orderService.findByUser(user);
        }
        return Response.ok(orders).build();
    }

    @GET
    @Path("/{id}")
    @RolesAllowed({"USER", "ADMIN"})
    public Response getOrder(@PathParam("id") Long id) {
        Order order = orderService.findById(id);
        if (order == null) {
            return Response.status(Response.Status.NOT_FOUND).build();
        }
        return Response.ok(order).build();
    }

    @POST
    @RolesAllowed({"USER", "ADMIN"})
    public Response createOrder(OrderRequest request) {
        String user = securityContext.getUserPrincipal().getName();
        Order order = orderService.create(request, user);
        return Response.status(Response.Status.CREATED).entity(order).build();
    }

    @DELETE
    @Path("/{id}")
    @RolesAllowed("ADMIN")
    public Response deleteOrder(@PathParam("id") Long id) {
        orderService.delete(id);
        return Response.noContent().build();
    }

    @GET
    @Path("/public/status")
    @PermitAll
    public Response getStatus() {
        return Response.ok(Map.of("status", "operational")).build();
    }
}
```

### After (Quarkus)

```java
import jakarta.annotation.security.RolesAllowed;
import jakarta.annotation.security.PermitAll;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import io.quarkus.security.identity.SecurityIdentity;

@Path("/orders")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class OrderResource {

    @Inject
    OrderService orderService;

    @Inject
    SecurityIdentity identity;

    @GET
    @RolesAllowed({"USER", "ADMIN"})
    public Response listOrders() {
        String user = identity.getPrincipal().getName();
        List<Order> orders;
        if (identity.hasRole("ADMIN")) {
            orders = orderService.findAll();
        } else {
            orders = orderService.findByUser(user);
        }
        return Response.ok(orders).build();
    }

    @GET
    @Path("/{id}")
    @RolesAllowed({"USER", "ADMIN"})
    public Response getOrder(@PathParam("id") Long id) {
        Order order = orderService.findById(id);
        if (order == null) {
            return Response.status(Response.Status.NOT_FOUND).build();
        }
        return Response.ok(order).build();
    }

    @POST
    @RolesAllowed({"USER", "ADMIN"})
    public Response createOrder(OrderRequest request) {
        String user = identity.getPrincipal().getName();
        Order order = orderService.create(request, user);
        return Response.status(Response.Status.CREATED).entity(order).build();
    }

    @DELETE
    @Path("/{id}")
    @RolesAllowed("ADMIN")
    public Response deleteOrder(@PathParam("id") Long id) {
        orderService.delete(id);
        return Response.noContent().build();
    }

    @GET
    @Path("/public/status")
    @PermitAll
    public Response getStatus() {
        return Response.ok(Map.of("status", "operational")).build();
    }
}
```

### Key Changes Summary

| Change | Explanation |
|---|---|
| `@Stateless` → `@ApplicationScoped` | EJB removed — CDI bean |
| `@DeclareRoles(...)` → REMOVED | Not needed in Quarkus |
| `@EJB` → `@Inject` | CDI injection |
| `@Context SecurityContext` → `@Inject SecurityIdentity` | Quarkus-native API (or keep `@Context SecurityContext` — both work) |
| `securityContext.isUserInRole(...)` → `identity.hasRole(...)` | Quarkus API (or keep JAX-RS `SecurityContext`) |
| web.xml `<security-constraint>` → `application.properties` | HTTP policy config |

## Testing Secured Endpoints

```java
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.*;

@QuarkusTest
public class OrderResourceTest {

    @Test
    @TestSecurity(user = "admin", roles = "ADMIN")
    public void testAdminCanDeleteOrder() {
        given()
            .when().delete("/orders/1")
            .then()
            .statusCode(204);
    }

    @Test
    @TestSecurity(user = "user1", roles = "USER")
    public void testUserCannotDeleteOrder() {
        given()
            .when().delete("/orders/1")
            .then()
            .statusCode(403);
    }

    @Test
    public void testUnauthenticatedAccessDenied() {
        given()
            .when().get("/orders")
            .then()
            .statusCode(401);
    }

    @Test
    @TestSecurity(user = "anyone", roles = {})
    public void testPublicEndpoint() {
        given()
            .when().get("/orders/public/status")
            .then()
            .statusCode(200);
    }
}
```

**Note**: `@TestSecurity` bypasses the real authentication mechanism and injects a synthetic identity — no need for test users in properties files or a running Keycloak.

## Extension Selection Guide

| Original Auth Mechanism | Recommended Quarkus Extension |
|---|---|
| Database users table (JDBC realm) | `quarkus-elytron-security-jdbc` |
| LDAP directory | `quarkus-elytron-security-ldap` |
| Keycloak / OAuth2 / OIDC | `quarkus-oidc` |
| Custom JAAS LoginModule | `quarkus-security` + custom `IdentityProvider` |
| Certificate/mTLS | `quarkus-security` + `quarkus-vertx-http` (built-in) |
| Properties file (dev) | `quarkus-elytron-security-properties-file` |
| JWT tokens (MicroProfile) | `quarkus-smallrye-jwt` |
| JASPIC ServerAuthModule | `quarkus-security` + custom `HttpAuthenticationMechanism` |
