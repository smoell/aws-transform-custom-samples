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
| `WEBSOCKET_NEEDED` | `@ServerEndpoint`, `javax.websocket.*`/`jakarta.websocket.*` imports | Phase 3 (WebSocket) |
| `JAXRS_PRESENT` | `@Path`, `javax.ws.rs.*`/`jakarta.ws.rs.*` imports | Phase 1 (add quarkus-rest) |
| `JPA_NEEDED` | `@Entity`, `@PersistenceContext`, `javax.persistence.*`/`jakarta.persistence.*` imports, OR `persistence.xml` exists | Phase 1 (add hibernate-orm) |
| `HAS_BEAN_VALIDATION` | `@Valid`, `@NotNull`, `@Size`, `javax.validation.*`/`jakarta.validation.*` imports | Phase 1 (add hibernate-validator) |
| `IS_MULTI_MODULE` | Multiple `src/main/java` trees (EAR/EJB/WAR structure) | Phase 1 (consolidation required) |

## 2. Detailed Detection Commands

### Namespace Migration Detection
```bash
# Detect javax.* EE imports (excluding Java SE packages)
grep -rn "import javax\." src/main/java/ | grep -v "javax\.crypto\|javax\.net\|javax\.sql\|javax\.security\.cert\|javax\.swing\|javax\.xml\.parsers\|javax\.xml\.transform"

# Find fully-qualified javax references in code (after import migration)
grep -rn "javax\." src/ | grep -v "^.*import " 
```

### EJB Detection
```bash
# Detect EJB annotations
grep -rn "@Stateless\|@Stateful\|@MessageDriven\|@EJB\|@TransactionAttribute" src/main/java/

# Find EJB deployment descriptors
find src/ -name "ejb-jar.xml"
```

### Configuration File Detection  
```bash
# Find application server-specific descriptors
find src/ -name "jboss-*.xml" -o -name "glassfish-*.xml" -o -name "sun-*.xml" -o -name "ibm-*.xml" -o -name "persistence.xml" -o -name "beans.xml" -o -name "web.xml" -o -name "arquillian.xml"
```

### Migration Completeness Validation
```bash
# Verify no Arquillian tests remain
grep -rn "Arquillian" src/test/

# Verify no ShrinkWrap imports remain  
grep -rn "shrinkwrap\|ShrinkWrap" src/test/

# Verify no JUnit 4 tests remain
grep -rn "import org.junit.Test" src/test/

# Check for anti-patterns
grep -rn "Arc\.container\(\)" src/main/java/
grep -rn "private static.*DataSource\|private static.*initialized\b" src/main/java/

# Verify JSF namespace correctness
grep -rn "jakarta\.face\." src/ | grep -v "jakarta\.faces\."
```

## 3. Health Check Commands

### Build Validation
```bash
# Verify clean compilation
./mvnw clean compile -Dmaven.test.skip=true

# Verify all tests pass
./mvnw clean test

# Check dependency conflicts
./mvnw dependency:tree | grep -i "jboss\|wildfly\|glassfish\|javax.ejb\|javax.faces"
```

### Runtime Validation  
```bash
# Verify health endpoint
curl -s http://localhost:8080/q/health | grep -q '"status": "UP"'

# Check startup time
grep "Quarkus started in" logs/application.log
```