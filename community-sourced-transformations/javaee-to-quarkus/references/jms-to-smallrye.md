# JMS/MDB → Quarkus Messaging Reference

> Reference for Phase 3 Step 13: Messaging migration (conditional on JMS_NEEDED or MDB_NEEDED flags).
> See also: https://quarkus.io/guides/jms and https://quarkus.io/guides/amqp

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

| Scenario | Recommended Option | Reasoning |
|---|---|---|
| Simple MDBs, fire-and-forget | Option B (Reactive) | Cleaner code, better cloud-native fit |
| Request-reply pattern (temp queues) | Option A (JMS API) | Reactive messaging has no request-reply built-in |
| Message selectors (`selector = "type = 'ORDER'"`) | Option A (JMS API) | Selectors not directly supported in SmallRye |
| XA distributed transactions (JMS + DB in one TX) | Option A (JMS API) | Reactive messaging does not support XA |
| Migrating to Kafka in the future | Option B (Reactive) | SmallRye supports both AMQP and Kafka connectors |
| Many MDBs with simple logic | Option B (Reactive) | Dramatic code reduction |
| JMS ObjectMessage / MapMessage | Option A (JMS API) | Complex message types easier with JMS API |
| Cloud-native event-driven architecture | Option B (Kafka) | Kafka is the standard for event streaming |

---

## Option A — Keep JMS API (quarkus-artemis-jms)

### Dependencies

```xml
<dependency>
    <groupId>io.quarkiverse.artemis</groupId>
    <artifactId>quarkus-artemis-jms</artifactId>
</dependency>
```

### application.properties Configuration

```properties
# Artemis broker connection
quarkus.artemis.url=tcp://localhost:61616
quarkus.artemis.username=admin
quarkus.artemis.password=admin

# Named connection (if multiple brokers)
quarkus.artemis."inventory".url=tcp://inventory-broker:61616
quarkus.artemis."inventory".username=inv_user
quarkus.artemis."inventory".password=inv_pass
```

### Before/After: @MessageDriven MDB → JMS Consumer Bean

```java
// BEFORE (JavaEE MDB)
import javax.ejb.MessageDriven;
import javax.ejb.ActivationConfigProperty;
import javax.jms.Message;
import javax.jms.MessageListener;
import javax.jms.TextMessage;
import javax.ejb.EJB;

@MessageDriven(activationConfig = {
    @ActivationConfigProperty(
        propertyName = "destinationType",
        propertyValue = "javax.jms.Queue"),
    @ActivationConfigProperty(
        propertyName = "destination",
        propertyValue = "java:/jms/queue/OrderQueue"),
    @ActivationConfigProperty(
        propertyName = "acknowledgeMode",
        propertyValue = "Auto-acknowledge"),
    @ActivationConfigProperty(
        propertyName = "maxSession",
        propertyValue = "10")
})
public class OrderMessageHandler implements MessageListener {

    @EJB
    private OrderService orderService;

    @Override
    public void onMessage(Message message) {
        try {
            if (message instanceof TextMessage) {
                String payload = ((TextMessage) message).getText();
                orderService.processOrder(payload);
            }
        } catch (Exception e) {
            throw new RuntimeException("Failed to process order message", e);
        }
    }
}
```

```java
// AFTER (Quarkus — JMS API consumer with scheduled polling)
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.jms.*;
import io.quarkus.runtime.StartupEvent;
import jakarta.enterprise.event.Observes;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

@ApplicationScoped
public class OrderMessageHandler {

    @Inject
    ConnectionFactory connectionFactory;

    @Inject
    OrderService orderService;

    private final ExecutorService executor = Executors.newSingleThreadExecutor();

    void onStart(@Observes StartupEvent ev) {
        executor.submit(this::consumeMessages);
    }

    private void consumeMessages() {
        try (JMSContext context = connectionFactory.createContext(Session.AUTO_ACKNOWLEDGE)) {
            Queue queue = context.createQueue("OrderQueue");
            JMSConsumer consumer = context.createConsumer(queue);

            while (true) {
                Message message = consumer.receive();
                if (message instanceof TextMessage textMsg) {
                    String payload = textMsg.getText();
                    orderService.processOrder(payload);
                }
            }
        } catch (Exception e) {
            // log and potentially restart
            throw new RuntimeException("JMS consumer failed", e);
        }
    }
}
```

**Alternative — simpler pattern using @Scheduled for polling:**

```java
@ApplicationScoped
public class OrderMessageHandler {

    @Inject
    ConnectionFactory connectionFactory;

    @Inject
    OrderService orderService;

    @Scheduled(every = "1s")
    void pollMessages() {
        try (JMSContext context = connectionFactory.createContext(Session.AUTO_ACKNOWLEDGE)) {
            Queue queue = context.createQueue("OrderQueue");
            JMSConsumer consumer = context.createConsumer(queue);

            Message message;
            while ((message = consumer.receiveNoWait()) != null) {
                if (message instanceof TextMessage textMsg) {
                    orderService.processOrder(textMsg.getText());
                }
            }
        } catch (Exception e) {
            // log error
        }
    }
}
```

### Before/After: JMS Producer

```java
// BEFORE (JavaEE JMS Producer)
import javax.ejb.Stateless;
import javax.annotation.Resource;
import javax.jms.*;

@Stateless
public class NotificationSender {

    @Resource(lookup = "java:/jms/queue/NotificationQueue")
    private Queue notificationQueue;

    @Resource(lookup = "java:/ConnectionFactory")
    private ConnectionFactory connectionFactory;

    public void sendNotification(String orderId, String message) {
        try (Connection conn = connectionFactory.createConnection();
             Session session = conn.createSession(false, Session.AUTO_ACKNOWLEDGE)) {

            MessageProducer producer = session.createProducer(notificationQueue);
            TextMessage textMessage = session.createTextMessage();
            textMessage.setText(message);
            textMessage.setStringProperty("orderId", orderId);
            producer.send(textMessage);
        } catch (JMSException e) {
            throw new RuntimeException("Failed to send notification", e);
        }
    }
}
```

```java
// AFTER (Quarkus — JMS API with injected ConnectionFactory)
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.jms.*;

@ApplicationScoped
public class NotificationSender {

    @Inject
    ConnectionFactory connectionFactory;

    public void sendNotification(String orderId, String message) {
        try (JMSContext context = connectionFactory.createContext(Session.AUTO_ACKNOWLEDGE)) {
            Queue queue = context.createQueue("NotificationQueue");
            context.createProducer()
                .setProperty("orderId", orderId)
                .send(queue, message);
        }
    }
}
```

### JNDI Queue/Topic Lookup → Direct Name

| JavaEE JNDI Lookup | Quarkus Equivalent |
|---|---|
| `@Resource(lookup="java:/jms/queue/OrderQueue")` | `context.createQueue("OrderQueue")` — use the destination name directly |
| `@Resource(lookup="java:/jms/topic/Events")` | `context.createTopic("Events")` — use the topic name directly |
| `@Resource(lookup="java:/ConnectionFactory")` | `@Inject ConnectionFactory` — auto-configured from `quarkus.artemis.*` |

---

## Option B — SmallRye Reactive Messaging

### Dependencies

```xml
<!-- For AMQP broker (Artemis, RabbitMQ with AMQP 1.0 plugin) -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-messaging-amqp</artifactId>
</dependency>

<!-- OR for Kafka -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-messaging-kafka</artifactId>
</dependency>
```

### Before/After: @MessageDriven → @Incoming

```java
// BEFORE (JavaEE MDB)
@MessageDriven(activationConfig = {
    @ActivationConfigProperty(propertyName = "destinationType",
        propertyValue = "javax.jms.Queue"),
    @ActivationConfigProperty(propertyName = "destination",
        propertyValue = "java:/jms/queue/OrderQueue")
})
public class OrderMessageHandler implements MessageListener {
    @EJB
    private OrderService orderService;

    @Override
    public void onMessage(Message message) {
        try {
            TextMessage textMsg = (TextMessage) message;
            orderService.processOrder(textMsg.getText());
        } catch (JMSException e) {
            throw new RuntimeException(e);
        }
    }
}

// AFTER (Quarkus — SmallRye Reactive Messaging)
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.reactive.messaging.Incoming;

@ApplicationScoped
public class OrderMessageHandler {

    @Inject
    OrderService orderService;

    @Incoming("orders-in")
    public void onMessage(String orderPayload) {
        orderService.processOrder(orderPayload);
    }
}
```

### Before/After: JMS Send → @Outgoing or Emitter

**Pattern 1: Emitter (imperative send — most common for migration)**

```java
// BEFORE (JavaEE JMS producer)
@Stateless
public class NotificationSender {
    @Resource(lookup = "java:/jms/queue/NotificationQueue")
    private Queue queue;

    @Inject
    private JMSContext jmsContext;

    public void sendNotification(String orderId) {
        jmsContext.createProducer().send(queue, orderId);
    }
}

// AFTER (Quarkus — Emitter)
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.reactive.messaging.Channel;
import org.eclipse.microprofile.reactive.messaging.Emitter;

@ApplicationScoped
public class NotificationSender {

    @Inject
    @Channel("notifications-out")
    Emitter<String> notificationEmitter;

    public void sendNotification(String orderId) {
        notificationEmitter.send(orderId);
    }
}
```

**Pattern 2: @Outgoing (reactive stream — for processing pipelines)**

```java
// Reactive stream that transforms incoming messages and forwards them
@ApplicationScoped
public class OrderProcessor {

    @Incoming("orders-in")
    @Outgoing("processed-orders-out")
    public String processOrder(String rawOrder) {
        // transform and forward
        return enrichOrder(rawOrder);
    }
}
```

### application.properties — Channel Configuration (AMQP)

```properties
# Incoming channel: consumes from "OrderQueue" on Artemis
mp.messaging.incoming.orders-in.connector=smallrye-amqp
mp.messaging.incoming.orders-in.address=OrderQueue
mp.messaging.incoming.orders-in.durable=true
mp.messaging.incoming.orders-in.container-id=order-consumer

# Outgoing channel: produces to "NotificationQueue"
mp.messaging.outgoing.notifications-out.connector=smallrye-amqp
mp.messaging.outgoing.notifications-out.address=NotificationQueue
mp.messaging.outgoing.notifications-out.durable=true

# AMQP broker connection
amqp-host=localhost
amqp-port=5672
amqp-username=admin
amqp-password=admin
```

### application.properties — Channel Configuration (Kafka)

```properties
# Incoming channel: consumes from "orders" topic
mp.messaging.incoming.orders-in.connector=smallrye-kafka
mp.messaging.incoming.orders-in.topic=orders
mp.messaging.incoming.orders-in.group.id=order-service
mp.messaging.incoming.orders-in.auto.offset.reset=earliest
mp.messaging.incoming.orders-in.value.deserializer=org.apache.kafka.common.serialization.StringDeserializer

# Outgoing channel: produces to "notifications" topic
mp.messaging.outgoing.notifications-out.connector=smallrye-kafka
mp.messaging.outgoing.notifications-out.topic=notifications
mp.messaging.outgoing.notifications-out.value.serializer=org.apache.kafka.common.serialization.StringSerializer

# Kafka broker connection
kafka.bootstrap.servers=localhost:9092
```

### Topic Consumer (Pub/Sub Pattern)

```java
// BEFORE (JavaEE — Topic MDB)
@MessageDriven(activationConfig = {
    @ActivationConfigProperty(propertyName = "destinationType",
        propertyValue = "javax.jms.Topic"),
    @ActivationConfigProperty(propertyName = "destination",
        propertyValue = "java:/jms/topic/OrderEvents"),
    @ActivationConfigProperty(propertyName = "subscriptionDurability",
        propertyValue = "Durable"),
    @ActivationConfigProperty(propertyName = "subscriptionName",
        propertyValue = "order-audit-sub")
})
public class OrderAuditListener implements MessageListener {
    @Override
    public void onMessage(Message message) { /* ... */ }
}

// AFTER (Quarkus — SmallRye Reactive Messaging)
@ApplicationScoped
public class OrderAuditListener {

    @Incoming("order-events")
    public void onOrderEvent(String event) {
        // process event
    }
}
```

```properties
# Topic subscription (AMQP)
mp.messaging.incoming.order-events.connector=smallrye-amqp
mp.messaging.incoming.order-events.address=OrderEvents
mp.messaging.incoming.order-events.durable=true
mp.messaging.incoming.order-events.container-id=order-audit-sub
mp.messaging.incoming.order-events.broadcast=true
```

### Structured Message Types (JSON Objects)

```java
// Consuming structured messages (auto-deserialization)
@ApplicationScoped
public class OrderMessageHandler {

    @Incoming("orders-in")
    public void onMessage(OrderEvent event) {
        // SmallRye auto-deserializes JSON → OrderEvent
        processOrder(event.getOrderId(), event.getAction());
    }
}

// Producing structured messages
@ApplicationScoped
public class OrderPublisher {

    @Inject
    @Channel("order-events-out")
    Emitter<OrderEvent> emitter;

    public void publishOrderCreated(String orderId) {
        emitter.send(new OrderEvent(orderId, "CREATED"));
    }
}
```

```properties
# JSON serialization config (Kafka)
mp.messaging.incoming.orders-in.value.deserializer=io.quarkus.kafka.client.serialization.ObjectMapperDeserializer
mp.messaging.outgoing.order-events-out.value.serializer=io.quarkus.kafka.client.serialization.ObjectMapperSerializer
```

---

## Transaction Semantics

### JMS Transacted Sessions → Acknowledgment Strategies

| JavaEE JMS Transaction Mode | SmallRye Reactive Messaging Equivalent |
|---|---|
| `Session.AUTO_ACKNOWLEDGE` | Default — message acked after method returns successfully |
| `Session.CLIENT_ACKNOWLEDGE` | Accept `Message<T>` parameter, call `message.ack()` manually |
| `Session.SESSION_TRANSACTED` (local TX) | `@Acknowledgment(Strategy.POST_PROCESSING)` — ack after processing chain completes |
| XA Transaction (JMS + DB) | **NOT SUPPORTED** in reactive messaging — use Option A (JMS API) |

### Acknowledgment Patterns

```java
// Auto-acknowledge (default) — message acked when method returns without exception
@Incoming("orders-in")
public void onMessage(String payload) {
    orderService.process(payload);
    // auto-acked on success; NACKed (redelivered) on exception
}

// Manual acknowledge — for custom error handling
@Incoming("orders-in")
public CompletionStage<Void> onMessage(Message<String> message) {
    try {
        orderService.process(message.getPayload());
        return message.ack();  // explicit acknowledge
    } catch (Exception e) {
        return message.nack(e);  // negative acknowledge → dead-letter or redelivery
    }
}

// Post-processing acknowledgment — ack after entire chain completes
@Incoming("orders-in")
@Outgoing("processed-out")
@Acknowledgment(Acknowledgment.Strategy.POST_PROCESSING)
public String processAndForward(String payload) {
    // ack only when downstream (@Outgoing) confirms receipt
    return transform(payload);
}
```

### XA Transactions (JMS + Database)

**Problem**: In JavaEE, MDBs can participate in XA transactions — the JMS message consumption and database write happen in the same distributed transaction. If the DB write fails, the message is not consumed.

**Quarkus limitation**: SmallRye Reactive Messaging does NOT support XA/distributed transactions.

**Solutions**:
1. **Option A (JMS API)** — use `quarkus-artemis-jms` with `UserTransaction` for JMS-local transactions
2. **Idempotent consumer pattern** — process messages idempotently, accept at-least-once delivery:
```java
@ApplicationScoped
public class OrderMessageHandler {

    @Inject
    OrderService orderService;

    @Incoming("orders-in")
    @Transactional
    public void onMessage(String orderId) {
        // Idempotent: check if already processed
        if (!orderService.isProcessed(orderId)) {
            orderService.processOrder(orderId);
        }
        // Message acked automatically after successful return
    }
}
```
3. **Outbox pattern** — write to DB + outbox table in single TX, relay outbox to broker separately

### Dead-Letter Queue Configuration

```properties
# AMQP — dead-letter on failure
mp.messaging.incoming.orders-in.failure-strategy=dead-letter-queue
mp.messaging.incoming.orders-in.dead-letter-queue.queue=orders-dlq

# Kafka — dead-letter topic
mp.messaging.incoming.orders-in.failure-strategy=dead-letter-queue
mp.messaging.incoming.orders-in.dead-letter-queue.topic=orders-dlq
mp.messaging.incoming.orders-in.dead-letter-queue.value.serializer=org.apache.kafka.common.serialization.StringSerializer
```

---

## Migration Checklist

| Step | Action |
|---|---|
| 1 | Identify all `@MessageDriven` classes and their activation config |
| 2 | Choose Option A or B per MDB (use decision tree above) |
| 3 | Map JNDI destination names to direct queue/topic names |
| 4 | Remove `@MessageDriven`, `implements MessageListener`, `@ActivationConfigProperty` |
| 5 | Add `@ApplicationScoped` + `@Incoming("channel")` (Option B) or JMS consumer (Option A) |
| 6 | Map JMS producers to `Emitter<T>` (Option B) or injected `ConnectionFactory` (Option A) |
| 7 | Configure channels in `application.properties` |
| 8 | Verify: `./mvnw clean compile` passes |
| 9 | Test: send a message to the queue/topic, verify consumer processes it |

## Testing Messaging

```java
// Test with InMemory connector (no broker needed)
@QuarkusTest
public class OrderMessageHandlerTest {

    @Inject
    @Channel("orders-in")
    Emitter<String> testEmitter;

    @Inject
    InMemoryConnector connector;

    @Test
    public void testMessageProcessing() {
        // Send test message
        testEmitter.send("test-order-123");

        // Verify processing (check database, mock, etc.)
        // ...
    }
}
```

```properties
# Test profile — use in-memory connector (no real broker)
%test.mp.messaging.incoming.orders-in.connector=smallrye-in-memory
%test.mp.messaging.outgoing.notifications-out.connector=smallrye-in-memory
```
