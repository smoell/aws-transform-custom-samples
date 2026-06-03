# JSF → Qute Templates Reference

> Reference for Phase 4: UI migration (conditional on JSF_NEEDED flag).

## Architecture Differences

**JSF Model**: Server-side component tree with lifecycle phases (Restore View → Apply Request → Process Validations → Update Model → Invoke Application → Render Response), stateful backing beans, navigation rules, and complex client-server synchronization.

**Qute Model**: Stateless template rendering engine, JAX-RS endpoints handle HTTP requests directly, templates are compiled at build-time for type safety, minimal server-side state.

## Migration Patterns

### Core Component Mappings

#### Form Submission
**JSF**:
```xml
<h:form>
    <h:inputText value="#{userBean.name}" />
    <h:commandButton action="#{userBean.save}" value="Save" />
</h:form>
```

**Qute + JAX-RS**:
```html
<form action="/users" method="POST">
    <input name="name" value="{user.name}" />
    <button type="submit">Save</button>
</form>
```
```java
@POST @Path("/users") @Consumes("application/x-www-form-urlencoded")
public TemplateInstance save(@FormParam("name") String name) {
    userService.save(name);
    return Templates.users(userService.findAll());
}
```

#### Data Tables
**JSF**:
```xml
<h:dataTable value="#{userBean.users}" var="user">
    <h:column>
        <h:outputText value="#{user.name}" />
    </h:column>
</h:dataTable>
```

**Qute**:
```html
<table>
{#for user in users}
    <tr><td>{user.name}</td></tr>
{/for}
</table>
```

#### Text Output
**JSF**: `<h:outputText value="#{user.name}" />` → **Qute**: `{user.name}`

### Template Composition

#### Include/Composition
**JSF**:
```xml
<ui:composition template="/WEB-INF/templates/layout.xhtml">
    <ui:define name="content">Page content</ui:define>
</ui:composition>
```

**Qute**:
```html
{#include layout}
    {#content}Page content{/content}
{/include}
```

#### Dynamic Includes
**JSF**: `<ui:include src="#{dynamicPage}" />` → **Qute**: `{#include {dynamicTemplate}}`

### Backing Bean Migration

#### @ManagedBean/@Named → CDI + @CheckedTemplate
**JSF**:
```java
@ManagedBean @RequestScoped
public class UserBean {
    private String name;
    public String save() { return "success"; }
    // getters/setters
}
```

**Quarkus**:
```java
@ApplicationScoped
public class UserResource {
    
    @CheckedTemplate
    public static class Templates {
        public static native TemplateInstance users(List<User> users);
        public static native TemplateInstance userForm(User user);
    }
    
    @GET @Path("/users")
    public TemplateInstance list() {
        return Templates.users(userService.findAll());
    }
    
    @POST @Path("/users")
    public TemplateInstance save(@FormParam("name") String name) {
        userService.save(name);
        return Templates.users(userService.findAll());
    }
}
```

### Navigation Rules

#### faces-config.xml → JAX-RS Routing
**JSF**:
```xml
<navigation-rule>
    <from-view-id>/user/edit.xhtml</from-view-id>
    <navigation-case>
        <from-outcome>success</from-outcome>
        <to-view-id>/user/list.xhtml</to-view-id>
    </navigation-case>
</navigation-rule>
```

**JAX-RS**:
```java
@POST @Path("/users/{id}/edit")
public Response update(@PathParam("id") Long id, @FormParam("name") String name) {
    userService.update(id, name);
    return Response.seeOther(URI.create("/users")).build(); // redirect
}
```

### AJAX → htmx/Fragments

#### JSF AJAX
**JSF**:
```xml
<h:commandButton value="Update">
    <f:ajax execute="@form" render="result" />
</h:commandButton>
<h:panelGroup id="result">#{bean.result}</h:panelGroup>
```

**htmx + Qute Fragment**:
```html
<button hx-post="/update" hx-target="#result">Update</button>
<div id="result">{#fragment result}{resultText}{/fragment}</div>
```
```java
@POST @Path("/update")
public TemplateInstance updateFragment() {
    return Templates.result(processUpdate());
}
```

### Validation

#### JSF Bean Validation → JAX-RS + Bean Validation
**JSF**: Works automatically with `<h:message for="field" />`

**JAX-RS**:
```java
@POST @Path("/users")
public TemplateInstance save(@Valid User user) {
    // ValidationException automatically handled by Quarkus
    return Templates.users(userService.save(user));
}
```

## Common Migration Steps

1. **Convert .xhtml to .html**: Remove JSF namespaces, convert components to HTML + Qute expressions
2. **Extract data logic**: Move from backing bean methods to JAX-RS endpoints
3. **Replace navigation**: Convert outcomes to HTTP redirects or template responses
4. **Handle forms**: Convert h:form to HTML forms, update endpoints to handle form data
5. **Convert AJAX**: Replace f:ajax with htmx or fetch() + fragment rendering

## PrimeFaces/RichFaces Alternatives

Common component replacements:
- **p:dataTable** → Qute {#for} + CSS framework (Bootstrap, Tailwind)
- **p:dialog** → HTML `<dialog>` element + JavaScript
- **p:calendar** → HTML5 `<input type="date">` or third-party picker
- **p:autoComplete** → HTML5 `<datalist>` or JavaScript autocomplete
- **p:fileUpload** → HTML5 `<input type="file">` + multipart handling

## Alternative: MyFaces Quarkus Extension

For complex JSF applications, consider the MyFaces Quarkus extension which preserves JSF API compatibility:

```xml
<dependency>
    <groupId>org.apache.myfaces.core.extensions.quarkus</groupId>
    <artifactId>myfaces-quarkus</artifactId>
</dependency>
```

This allows incremental migration while maintaining existing JSF components and lifecycle.