# Batch Processing: JBeret Fallback Pattern

## Decision Tree

**Primary Path: quarkus-jberet**
1. Add `quarkus-jberet` extension
2. Attempt build: `./mvnw clean compile`
3. If successful → continue with standard JBeret migration

**Fallback Trigger Signals:**
- **ClassNotFoundException** during build for batch-related classes
- **NoSuchMethodError** at runtime for JBeret APIs
- **UnsatisfiedLinkError** or native linking issues

**Fallback Path: CDI-based Manual Implementation**
When quarkus-jberet fails, replace with CDI-based batch processing (see below).

## Problem
`quarkus-jberet` has limited compatibility with newer Quarkus versions and may fail during migration.

## CDI-Based Fallback Solution

When `quarkus-jberet` is incompatible, use CDI-based batch processing:

### Pattern: Manual Batch Composition

> **⚠️ CDI Self-Invocation Warning**: `@Transactional` (and all CDI interceptors) only applies when a method is called through the CDI proxy — i.e., from an external bean. Self-calls within the same bean (`this.method()`) bypass the interceptor entirely. Place `@Transactional` on the cross-bean method calls (reader/writer), NOT on the launcher's internal orchestration methods.

```java
@ApplicationScoped
public class BatchProcessor {
    
    @Inject
    DatabaseItemReader itemReader;
    
    @Inject 
    DataProcessor itemProcessor;
    
    @Inject
    DatabaseItemWriter itemWriter;
    
    public void processChunk(List<Object> items) {
        List<Object> processedItems = new ArrayList<>();
        for (Object item : items) {
            Object processed = itemProcessor.processItem(item);
            if (processed != null) {
                processedItems.add(processed);
            }
        }
        itemWriter.writeItems(processedItems);  // @Transactional on writer (cross-bean call → proxy intercepts)
    }
    
    @Scheduled(every = "PT1H") // Run hourly
    public void runBatch() {
        List<Object> chunk;
        while ((chunk = itemReader.readItems(100)) != null && !chunk.isEmpty()) {
            processChunk(chunk);
        }
    }
}
```

### Component Implementation

```java
@ApplicationScoped
public class DatabaseItemReader {
    @Inject
    EntityManager em;
    
    private int currentOffset = 0;
    private final int chunkSize = 100;
    
    @Transactional  // cross-bean call — proxy intercepts correctly
    public List<Object> readItems(int count) {
        List<Object> items = em.createQuery("SELECT e FROM Entity e ORDER BY e.id")
                               .setFirstResult(currentOffset)
                               .setMaxResults(count)
                               .getResultList();
        if (items.isEmpty()) {
            return null;  // signal end-of-data
        }
        currentOffset += items.size();  // advance by actual result count, not requested count
        return items;
    }
    
    public void reset() {
        currentOffset = 0;  // call before each job execution for re-runnability
    }
}

@ApplicationScoped  
public class DataProcessor {
    public Object processItem(Object item) {
        // Transform/validate item
        return item; // transformed
    }
}

@ApplicationScoped
public class DatabaseItemWriter {
    @Inject
    EntityManager em;
    
    @Transactional  // cross-bean call — proxy intercepts correctly
    public void writeItems(List<Object> items) {
        for (Object item : items) {
            em.persist(item);
        }
    }
}
```

## Worked Example: WildFly batch-processing (#11)

The WildFly `batch-processing` quickstart was migrated using this pattern:

**Before (JSR 352 Job XML):**
```xml
<job id="chunk-simple" xmlns="http://xmlns.jcp.org/xml/ns/javaee">
    <step id="chunkStep">
        <chunk item-count="3">
            <reader ref="simpleItemReader"/>
            <processor ref="simpleItemProcessor"/>
            <writer ref="simpleItemWriter"/>
        </chunk>
    </step>
</job>
```

**After (CDI Composition):**
```java
@ApplicationScoped
public class ChunkSimpleBatch {
    @Inject SimpleItemReader reader;
    @Inject SimpleItemWriter writer;

    @Scheduled(cron = "0 0 2 * * ?") // 2 AM daily
    public void executeJob() {
        List<Object> chunk;
        while ((chunk = reader.readItems(3)) != null && !chunk.isEmpty()) {
            writer.writeItems(chunk);  // @Transactional on writer bean
        }
    }
}
```

## Key Benefits

1. **No XML Configuration**: Pure CDI approach
2. **Quarkus Native**: Works with native compilation
3. **Transactional Control**: Explicit @Transactional on cross-bean calls (proxy-intercepted)
4. **Scheduling Integration**: Use @Scheduled for triggers
5. **Testability**: Standard CDI testing patterns

## Trade-offs

- **Manual Implementation**: More boilerplate than JSR 352
- **No Job Repository**: Must implement job tracking if needed
- **Limited Restart**: No built-in restart capabilities

## When to Use

- `quarkus-jberet` fails to compile/run
- Batch job is simple (single-step processing)
- Native compilation required
- Prefer explicit over framework magic
