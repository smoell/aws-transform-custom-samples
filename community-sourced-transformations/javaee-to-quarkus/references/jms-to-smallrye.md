# JMS/MDB → Quarkus Messaging Reference

> Reference for Phase 3 Step 14: Messaging migration (conditional on JMS_NEEDED or MDB_NEEDED flags).
> See also: https://quarkus.io/guides/jms and https://quarkus.io/guides/amqp

**Artifact naming**: For Quarkus 3.9+, use the shortened artifact IDs `quarkus-messaging-amqp` and `quarkus-messaging-kafka` (renamed from `quarkus-smallrye-reactive-messaging-amqp`/`-kafka`). Old names still work via Maven relocation but produce warnings.

## Decision Tree

```
Is the MDB pattern simple? (single queue/topic consumer, no selectors, no XA)
├── YES → Option B: SmallRye Reactive Messaging (recommended for cloud-native)
│         ├── Target is cloud/Kubernetes? → quarkus-messaging-amqp or quarkus-messaging-kafka
│         └── Keeping Artemis broker? → quarkus-messaging-amqp (AMQP 1.0 protocol to Artemis)
└── NO (complex JMS: request-reply, selectors, XA, temporary queues)
    └── Option A: quarkus-artemis-jms (keep JMS API, minimal code change)
```

### Recommendation Matrix

| Scenario | Option | Reasoning |
|---|---|---|
| Simple MDBs, fire-and-forget | B (Reactive) | Cleaner code, cloud-native fit |
| Request-reply (temp queues) | A (JMS API) | No request-reply built into reactive messaging |
| Message selectors (`type='ORDER'`) | A (JMS API) | Selectors not supported in SmallRye |
| XA distributed TX (JMS + DB) | A (JMS API) | Reactive messaging has no XA |
| Migrating to Kafka later | B (Reactive) | SmallRye supports AMQP + Kafka |
| Many simple MDBs | B (Reactive) | Dramatic code reduction |
| ObjectMessage / MapMessage | A (JMS API) | Complex message types easier with JMS API |

---

## Option A — Keep JMS API (quarkus-artemis-jms)

### Dependency & Configuration

```xml
<dependency><groupId>io.quarkiverse.artemis</groupId><artifactId>quarkus-artemis-jms</artifactId></dependency>
```

```properties
quarkus.artemis.url=tcp://localhost:61616
quarkus.artemis.username=admin
quarkus.artemis.password=[REDACTED_PASSWORD]
# Named connection (multiple brokers): quarkus.artemis."inventory".url=tcp://inventory-broker:61616
```

### @MessageDriven MDB → JMS Consumer Bean

```java
// BEFORE (JavaEE MDB)
@MessageDriven(activationConfig = {
    @ActivationConfigProperty(propertyName = "destinationType", propertyValue = "javax.jms.Queue"),
    @ActivationConfigProperty(propertyName = "destination", propertyValue = "java:/jms/queue/OrderQueue"),
    @ActivationConfigProperty(propertyName = "acknowledgeMode", propertyValue = "Auto-acknowledge")
})
public class OrderMessageHandler implements MessageListener {
    @EJB private OrderService orderService;
    @Override public void onMessage(Message message) {
        try {
            if (message instanceof TextMessage tm) orderService.processOrder(tm.getText());
        } catch (Exception e) { throw new RuntimeException("Failed to process order message", e); }
    }
}

// AFTER (Quarkus — JMS API consumer, @Scheduled polling pattern)
@ApplicationScoped
public class OrderMessageHandler {
    @Inject ConnectionFactory connectionFactory;
    @Inject OrderService orderService;

    @Scheduled(every = "1s")
    void pollMessages() {
        try (JMSContext context = connectionFactory.createContext(Session.AUTO_ACKNOWLEDGE)) {
            JMSConsumer consumer = context.createConsumer(context.createQueue("OrderQueue"));
            Message message;
            while ((message = consumer.receiveNoWait()) != null) {
                if (message instanceof TextMessage tm) orderService.processOrder(tm.getText());
            }
        } catch (Exception e) { /* log error */ }
    }
}
```

For a continuous blocking consumer instead of polling, submit a `consumeMessages()` loop to a single-thread `ExecutorService` from `void onStart(@Observes StartupEvent ev)`, using `consumer.receive()` (blocking) inside a `try (JMSContext ...)`.

### JMS Producer

```java
// AFTER (Quarkus — injected ConnectionFactory)
@ApplicationScoped
public class NotificationSender {
    @Inject ConnectionFactory connectionFactory;
    public void sendNotification(String orderId, String message) {
        try (JMSContext context = connectionFactory.createContext(Session.AUTO_ACKNOWLEDGE)) {
            context.createProducer().setProperty("orderId", orderId)
                .send(context.createQueue("NotificationQueue"), message);
        }
    }
}
```

### JNDI Queue/Topic Lookup → Direct Name

| JavaEE JNDI Lookup | Quarkus Equivalent |
|---|---|
| `@Resource(lookup="java:/jms/queue/OrderQueue")` | `context.createQueue("OrderQueue")` — name directly |
| `@Resource(lookup="java:/jms/topic/Events")` | `context.createTopic("Events")` — name directly |
| `@Resource(lookup="java:/ConnectionFactory")` | `@Inject ConnectionFactory` — auto-configured from `quarkus.artemis.*` |

---

## Option B — SmallRye Reactive Messaging

### Dependencies

```xml
<!-- AMQP (Artemis, RabbitMQ w/ AMQP 1.0) -->
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-messaging-amqp</artifactId></dependency>
<!-- OR Kafka -->
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-messaging-kafka</artifactId></dependency>
```

### @MessageDriven → @Incoming

The BEFORE is the same MDB as Option A. The Quarkus form:

```java
@ApplicationScoped
public class OrderMessageHandler {
    @Inject OrderService orderService;

    @Incoming("orders-in")
    public void onMessage(String orderPayload) {
        orderService.processOrder(orderPayload);
    }
}
```

Remove `@MessageDriven`, `implements MessageListener`, and all `@ActivationConfigProperty`.

### JMS Send → Emitter or @Outgoing

**Pattern 1: Emitter (imperative send — most common for migration)**

```java
@ApplicationScoped
public class NotificationSender {
    @Inject @Channel("notifications-out") Emitter<String> notificationEmitter;
    public void sendNotification(String orderId) { notificationEmitter.send(orderId); }
}
```

**Pattern 2: @Outgoing (reactive stream — for processing pipelines)**

```java
@ApplicationScoped
public class OrderProcessor {
    @Incoming("orders-in")
    @Outgoing("processed-orders-out")
    public String processOrder(String rawOrder) { return enrichOrder(rawOrder); }
}
```

### Channel Configuration (AMQP)

```properties
mp.messaging.incoming.orders-in.connector=smallrye-amqp
mp.messaging.incoming.orders-in.address=OrderQueue
mp.messaging.incoming.orders-in.durable=true
mp.messaging.incoming.orders-in.container-id=order-consumer

mp.messaging.outgoing.notifications-out.connector=smallrye-amqp
mp.messaging.outgoing.notifications-out.address=NotificationQueue
mp.messaging.outgoing.notifications-out.durable=true

amqp-host=localhost
amqp-port=5672
amqp-username=admin
amqp-password=[REDACTED_PASSWORD]
```

### Channel Configuration (Kafka)

```properties
mp.messaging.incoming.orders-in.connector=smallrye-kafka
mp.messaging.incoming.orders-in.topic=orders
mp.messaging.incoming.orders-in.group.id=order-service
mp.messaging.incoming.orders-in.auto.offset.reset=earliest
mp.messaging.incoming.orders-in.value.deserializer=org.apache.kafka.common.serialization.StringDeserializer

mp.messaging.outgoing.notifications-out.connector=smallrye-kafka
mp.messaging.outgoing.notifications-out.topic=notifications
mp.messaging.outgoing.notifications-out.value.serializer=org.apache.kafka.common.serialization.StringSerializer

kafka.bootstrap.servers=localhost:9092
```

### Topic Consumer (Pub/Sub)

A durable Topic MDB migrates to `@Incoming("order-events")` with:

```properties
mp.messaging.incoming.order-events.connector=smallrye-amqp
mp.messaging.incoming.order-events.address=OrderEvents
mp.messaging.incoming.order-events.durable=true
mp.messaging.incoming.order-events.container-id=order-audit-sub
mp.messaging.incoming.order-events.broadcast=true
```

### Structured Message Types (JSON Objects)

SmallRye auto-(de)serializes JSON to/from a POJO: `@Incoming("orders-in") public void onMessage(OrderEvent event)` and `Emitter<OrderEvent>`. Configure the object (de)serializer (Kafka):

```properties
mp.messaging.incoming.orders-in.value.deserializer=io.quarkus.kafka.client.serialization.ObjectMapperDeserializer
mp.messaging.outgoing.order-events-out.value.serializer=io.quarkus.kafka.client.serialization.ObjectMapperSerializer
```

---

## Transaction Semantics

### JMS Transacted Sessions → Acknowledgment Strategies

| JavaEE JMS Transaction Mode | SmallRye Equivalent |
|---|---|
| `Session.AUTO_ACKNOWLEDGE` | Default — acked after method returns successfully |
| `Session.CLIENT_ACKNOWLEDGE` | Accept `Message<T>`, call `message.ack()` manually |
| `Session.SESSION_TRANSACTED` | `@Acknowledgment(Strategy.POST_PROCESSING)` — ack after chain completes |
| XA (JMS + DB) | **NOT SUPPORTED** — use Option A (JMS API) |

### Acknowledgment Patterns

```java
// Auto-acknowledge (default) — acked when method returns; NACKed (redelivered) on exception
@Incoming("orders-in")
public void onMessage(String payload) { orderService.process(payload); }

// Manual acknowledge — custom error handling
@Incoming("orders-in")
public CompletionStage<Void> onMessage(Message<String> message) {
    try { orderService.process(message.getPayload()); return message.ack(); }
    catch (Exception e) { return message.nack(e); }  // → dead-letter or redelivery
}

// Post-processing — ack only after downstream @Outgoing confirms
@Incoming("orders-in") @Outgoing("processed-out")
@Acknowledgment(Acknowledgment.Strategy.POST_PROCESSING)
public String processAndForward(String payload) { return transform(payload); }
```

### XA Transactions (JMS + Database)

In JavaEE, MDBs can participate in XA transactions (message consumption + DB write in one distributed TX). SmallRye Reactive Messaging does NOT support XA. Solutions:

1. **Option A (JMS API)** — `quarkus-artemis-jms` with `UserTransaction` for JMS-local transactions.
2. **Idempotent consumer** — accept at-least-once delivery, process idempotently:
```java
@Incoming("orders-in")
@Transactional
public void onMessage(String orderId) {
    if (!orderService.isProcessed(orderId)) orderService.processOrder(orderId);
}
```
3. **Outbox pattern** — write to DB + outbox table in one TX, relay outbox to broker separately.

### Dead-Letter Queue

```properties
# AMQP
mp.messaging.incoming.orders-in.failure-strategy=dead-letter-queue
mp.messaging.incoming.orders-in.dead-letter-queue.queue=orders-dlq
# Kafka: .failure-strategy=dead-letter-queue + .dead-letter-queue.topic=orders-dlq
```

---

## Migration Checklist

| Step | Action |
|---|---|
| 1 | Identify all `@MessageDriven` classes + activation config |
| 2 | Choose Option A or B per MDB (decision tree) |
| 3 | Map JNDI destination names to direct queue/topic names |
| 4 | Remove `@MessageDriven`, `implements MessageListener`, `@ActivationConfigProperty` |
| 5 | Add `@ApplicationScoped` + `@Incoming` (B) or JMS consumer (A) |
| 6 | Map producers to `Emitter<T>` (B) or injected `ConnectionFactory` (A) |
| 7 | Configure channels in `application.properties` (+ `%test` in-memory — see below) |
| 8 | Verify `./mvnw clean compile` passes |
| 9 | Test: send a message, verify consumer processes it |

## Test Profile Configuration (In-Memory Connector)

For every `mp.messaging` channel, add a `%test` override using the in-memory connector. **MANDATORY** when no AMQP/Kafka broker is available in tests — otherwise `@QuarkusTest` startup fails with connection-refused to localhost:5672/9092.

```xml
<dependency><groupId>io.smallrye.reactive</groupId>
  <artifactId>smallrye-reactive-messaging-in-memory</artifactId><scope>test</scope></dependency>
```

```properties
# Override EVERY channel for tests
%test.mp.messaging.incoming.orders-in.connector=smallrye-in-memory
%test.mp.messaging.outgoing.notifications-out.connector=smallrye-in-memory
%test.mp.messaging.incoming.order-events.connector=smallrye-in-memory
```

**Rule**: for each `mp.messaging.incoming/outgoing.<channel>`, add `%test.mp.messaging.<direction>.<channel>.connector=smallrye-in-memory`.

Source: [SmallRye Reactive Messaging Testing Guide](https://smallrye.io/smallrye-reactive-messaging/4.24.0/concepts/testing/)

### Test Example

```java
@QuarkusTest
public class OrderMessageHandlerTest {
    @Inject @Channel("orders-in") Emitter<String> testEmitter;
    @Inject InMemoryConnector connector;

    @Test
    public void testMessageProcessing() {
        testEmitter.send("test-order-123");
        // verify processing (database, mock, etc.)
    }
}
```

---

## Message Properties / Headers Warning

`Emitter<String>` carries only the payload. JMS properties set via `setStringProperty()`/`getStringProperty()` have **no automatic equivalent** in SmallRye. Options:

1. **Encode in JSON payload** — include former header fields as JSON properties.
2. **Use `Message<T>` with OutgoingAmqpMetadata** (AMQP only):
```java
emitter.send(Message.of(payload)
    .addMetadata(OutgoingAmqpMetadata.builder()
        .withApplicationProperties(Map.of("orderId", orderId)).build()));
```

**Migration action**: at every `setStringProperty()`/`getStringProperty()` call site, add a `// MIGRATION: JMS property — encoded in payload` comment and choose option (1) or (2).

## Durable Subscription Mapping

| JMS ActivationConfig Property | SmallRye AMQP Config |
|---|---|
| `subscriptionDurability=Durable` | `mp.messaging.incoming.<channel>.durable=true` |
| `subscriptionName=my-sub` | `mp.messaging.incoming.<channel>.container-id=my-sub` |
| `clientId=my-client` | `.client-id=my-client` (Kafka: `group.id`) |
| `messageSelector="type='ORDER'"` | Not supported — use Option A (JMS API) or filter in code |
