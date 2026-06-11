# Multi-module EAR Consolidation

## Problem
Java EE applications often use multi-module EAR structure:
```
CustomerApp.ear
├── CustomerEJB.jar    (business logic)  
├── CustomerWeb.war    (web tier)
└── META-INF/application.xml
```

Quarkus uses single-module JAR packaging.

## Consolidation Strategy

### Step 1: Analyze Module Dependencies
```bash
# Check module structure
find . -name "pom.xml" -exec grep -l "packaging.*ear\|packaging.*ejb" {} \;

# Map dependencies between modules
mvn dependency:tree
```

### Step 2: Create Target Structure
```
quarkus-app/
├── pom.xml                    (Quarkus packaging=jar)
├── src/main/java/            (merged from all modules)
├── src/main/resources/       (configurations merged)
└── src/test/java/            (tests consolidated)
```

### Step 3: Source Consolidation
```bash
# Merge source directories
cp -r CustomerEJB/src/main/java/* quarkus-app/src/main/java/
cp -r CustomerWeb/src/main/java/* quarkus-app/src/main/java/
cp -r CustomerWeb/src/main/webapp/WEB-INF/web.xml quarkus-app/src/main/resources/
```

## Worked Examples

### Example 1: DayTrader 7 (#2)
**Before (Multi-module):**
- `daytrader-ee7-ejb` (EJB business logic)
- `daytrader-ee7-web` (JSF web interface)  
- `daytrader-ee7-ear` (packaging)

**After (Single module):**
- All EJB classes → `src/main/java/com/ibm/websphere/samples/daytrader/`
- JSF pages → `src/main/resources/META-INF/resources/`
- Single `pom.xml` with Quarkus BOM

### Example 2: ejb-multi-server (#9)
**Before (Multi-module EAR):**
```
ejb-multi-server/
├── client/                 (remote client)
├── ear/                   (EAR packaging)
├── ejb/                   (EJB implementation)
├── web/                   (web tier)
└── server-side/           (server utilities)
```

**After (Consolidated):**
- Remote EJB → REST endpoints
- EJB business logic → CDI beans
- Web tier integrated directly
- Single deployment unit

## Configuration Merge

### application.xml → application.properties
```xml
<!-- Before: META-INF/application.xml -->
<application>
    <module>
        <ejb>CustomerEJB.jar</ejb>
    </module>
    <module>
        <web>
            <web-uri>CustomerWeb.war</web-uri>
            <context-root>/customer</context-root>
        </web>
    </module>
</application>
```

```properties
# After: application.properties
quarkus.http.root-path=/customer
quarkus.arc.remove-unused-beans=false
```

### web.xml → application.properties
```xml
<!-- Before: WEB-INF/web.xml -->
<web-app>
    <context-param>
        <param-name>javax.faces.PROJECT_STAGE</param-name>
        <param-value>Production</param-value>
    </context-param>
</web-app>
```

```properties
# After: application.properties  
quarkus.faces.project-stage=Production
```

## Dependency Resolution

### Before (Parent POM):
```xml
<modules>
    <module>ejb-module</module>
    <module>web-module</module>
    <module>ear-module</module>
</modules>
```

### After (Single POM):
```xml
<dependencies>
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-arc</artifactId>
    </dependency>
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-undertow</artifactId>
    </dependency>
    <!-- Merge all module dependencies here -->
</dependencies>
```

## Testing Consolidation

### Before (Module-specific tests):
- `CustomerEJB/src/test/` → EJB unit tests
- `CustomerWeb/src/test/` → Web integration tests
- `ear-module/` → Arquillian deployment tests

### After (Unified testing):
```java
@QuarkusTest
public class CustomerServiceIT {
    // Tests that cover both EJB logic and web endpoints
    
    @Test
    public void testEndToEndFlow() {
        // Test complete user journey
    }
}
```

## Common Pitfalls

1. **Circular Dependencies**: Check for module circular references
2. **Resource Conflicts**: Watch for duplicate resource files
3. **Class Name Conflicts**: Resolve duplicate class names across modules
4. **Context Paths**: Update hardcoded module paths in code

## Validation Checklist

- [ ] All source code merged without conflicts
- [ ] No circular dependencies in merged code
- [ ] All tests execute successfully  
- [ ] Application context paths work correctly
- [ ] No missing configuration properties
- [ ] Build produces single JAR artifact
- [ ] Docker image builds and runs
- [ ] Health checks pass