# Phase 0 Detection Flags — JavaEE/JakartaEE → Quarkus

> Reference file for the `javaee-to-quarkus` transformation definition.
> Loaded by Phase 0 (Project Analysis) to set feature flags that control which subsequent phases execute.

---

## 1. Core Execution Flags

These flags determine which transformation phases run. Phases are skipped when their corresponding flag is `false`.

| Flag | Detection Rule | Triggers Phase |
|------|---------------|----------------|
| `EJB_NEEDED` | Any Java file contains `@Stateless`, `@Stateful`, `@Singleton` (from `javax.ejb`/`jakarta.ejb`), OR `ejb-jar.xml` exists in `META-INF/` | Phase 2 (EJB→CDI) |
| `MDB_NEEDED` | Any Java file contains `@MessageDriven` (from `javax.ejb`/`jakarta.ejb`), OR `ejb-jar.xml` contains `<message-driven>` | Phase 3 (Messaging) |
| `JMS_NEEDED` | `@JMSDestinationDefinition`, `javax.jms.*`/`jakarta.jms.*` imports, OR `@MessageDriven` present | Phase 3 (Messaging) |
| `SECURITY_NEEDED` | `<security-constraint>` in `web.xml`, `@RolesAllowed`, `@DeclareRoles`, `@RunAs`, `<security-domain>` in `jboss-web.xml`/`glassfish-web.xml`, OR `@LoginConfig` | Phase 3 (Security) |
| `JSF_NEEDED` | `.xhtml` files in `src/main/webapp/`, OR `javax.faces.*`/`jakarta.faces.*` imports, OR `faces-config.xml` exists | Phase 4 (UI) |
| `HAS_DOTTED_NAMED_BEANS` | Any `@Named` annotation contains a dot in the value: `grep -rn '@Named(".*\\..*")' src/main/java/` | Phase 4 — rename to camelCase before JSF pages will work |
| `SOAP_NEEDED` | `@WebService`, `@WebMethod`, `javax.xml.ws.*`/`jakarta.xml.ws.*` imports, OR WSDL files in `src/main/resources/` | Phase 3 (SOAP→REST or quarkus-cxf) |
| `BATCH_NEEDED` | `javax.batch.*`/`jakarta.batch.*` imports, OR `META-INF/batch-jobs/` directory exists | Phase 4 (Batch) |
| `WEBSOCKET_NEEDED` | `@ServerEndpoint`, `javax.websocket