# SOA and EAI Architecture Patterns

Detailed reference for Service-Oriented Architecture and Enterprise Application Integration patterns, with Java OSS implementation guidance using Spring Boot, Apache Camel, Apache Artemis, ZooKeeper, and Hazelcast.

## Table of Contents

1. [Canonical Data Model](#1-canonical-data-model)
2. [Protocol Mediation](#2-protocol-mediation)
3. [Message Broker](#3-message-broker)
4. [Content-Based Router](#4-content-based-router)
5. [Scatter-Gather](#5-scatter-gather)
6. [Anti-Corruption Layer](#6-anti-corruption-layer)
7. [Service Registry](#7-service-registry)
8. [Saga / Compensation](#8-saga--compensation)
9. [Event Sourcing](#9-event-sourcing)
10. [CQRS](#10-cqrs)
11. [Transactional Outbox](#11-transactional-outbox)
12. [Idempotent Receiver](#12-idempotent-receiver)
13. [Wire Tap / Audit](#13-wire-tap--audit)
14. [Claim Check](#14-claim-check)
15. [Message Expiration](#15-message-expiration)
16. [Competing Consumers](#16-competing-consumers)
17. [Singleton Service](#17-singleton-service)
18. [Distributed Lock](#18-distributed-lock)

---

## 1. Canonical Data Model

**Problem:** N systems each have their own data formats. Point-to-point integration requires N×(N−1) transformations.

**Solution:** Define a single canonical schema for each business entity. Each system transforms to/from canonical at the boundary (on-ramp/off-ramp). The bus carries only canonical messages.

**Java OSS Implementation:**
- Define canonical types as Java records or POJOs in a shared Maven module (`canonical-model`).
- Use JSON Schema, Avro, or Protobuf for cross-language compatibility.
- Camel `marshal()`/`unmarshal()` at on-ramp and off-ramp routes.
- Version schemas in a shared Git repo or Confluent Schema Registry.

**When to use:** Always, in any ESB or integration platform with more than two systems.

**When to avoid:** Prototyping with only two systems — the overhead may not justify itself yet. But plan for it.

---

## 2. Protocol Mediation

**Problem:** Systems use different communication protocols (REST, SOAP, JMS, FTP, gRPC). You need them to communicate without coupling each to the other's protocol.

**Solution:** The ESB accepts messages in the producer's native protocol and delivers them in the consumer's native protocol. The message bus in the middle is protocol-agnostic.

**Java OSS Implementation:**
```java
// REST → JMS mediation
from("rest:post:/api/orders")
    .routeId("rest-to-jms-mediation")
    .unmarshal().json(JsonLibrary.Jackson, ExternalOrder.class)
    .bean("orderMapper", "toCanonical")
    .marshal().json(JsonLibrary.Jackson)
    .to("jms:queue:canonical.orders");

// JMS → SOAP mediation
from("jms:queue:canonical.orders.erp")
    .routeId("jms-to-soap-mediation")
    .unmarshal().json(JsonLibrary.Jackson, CanonicalOrder.class)
    .bean("erpMapper", "fromCanonical")
    .marshal().jaxb("com.example.erp.model")
    .to("cxf:http://erp.internal:8080/ws/orders?serviceClass=OrderService");

// FTP → JMS mediation
from("sftp:partner.example.com/outbox?username=${ftp.user}&password=${ftp.pass}&delay=60000")
    .routeId("ftp-to-jms-mediation")
    .unmarshal().csv()
    .split(body())
    .bean("csvMapper", "toCanonical")
    .marshal().json(JsonLibrary.Jackson)
    .to("jms:queue:canonical.invoices");
```

**When to use:** Any time two systems speak different protocols.

---

## 3. Message Broker

**Problem:** Direct point-to-point connections between services create tight coupling, poor resilience, and complex topology.

**Solution:** Route all inter-service communication through a central message broker. Producers send to the broker; consumers receive from the broker. Neither knows about the other.

**Java OSS Implementation:**
- Apache Artemis as the JMS broker.
- Queues for point-to-point (each message consumed by one service).
- Topics for publish-subscribe (each message delivered to all subscribers).
- Dead letter queues for failed messages.
- Message persistence for guaranteed delivery.

**Topology:**
```yaml
# Artemis broker.xml (HA with shared store)
<ha-policy>
  <shared-store>
    <primary>
      <failover-on-shutdown>true</failover-on-shutdown>
    </primary>
  </shared-store>
</ha-policy>
```

---

## 4. Content-Based Router

**Problem:** Incoming messages must be routed to different destinations based on their content.

**Solution:** Inspect message content (headers, body fields) and route to the appropriate destination.

**Java OSS Implementation:**
```java
from("jms:queue:canonical.orders")
    .routeId("order-router")
    .choice()
        .when(jsonpath("$.region == 'APAC'"))
            .to("jms:queue:regional.orders.apac")
        .when(jsonpath("$.region == 'EMEA'"))
            .to("jms:queue:regional.orders.emea")
        .when(jsonpath("$.region == 'AMER'"))
            .to("jms:queue:regional.orders.amer")
        .otherwise()
            .to("jms:queue:regional.orders.default")
    .end();
```

---

## 5. Scatter-Gather

**Problem:** You need to request data from multiple services and combine the results.

**Solution:** Send the request to multiple providers in parallel, wait for all responses (or timeout), and aggregate.

**Java OSS Implementation:**
```java
from("direct:getPriceQuotes")
    .routeId("scatter-gather-pricing")
    .multicast(new BestPriceAggregator())
        .parallelProcessing()
        .timeout(5000)                         // 5 second timeout
        .to("direct:vendorA-price",
            "direct:vendorB-price",
            "direct:vendorC-price")
    .end()
    .to("direct:handleBestPrice");
```

---

## 6. Anti-Corruption Layer

**Problem:** External systems have data models that don't match your canonical domain. Direct integration would corrupt your domain model.

**Solution:** Place a translation layer at the boundary that converts between the external model and your canonical model. The rest of your system only sees canonical types.

**Java OSS Implementation:**
- One mapper class per external system, per entity.
- Mappers are Spring `@Component` beans, called from Camel routes via `.bean()`.
- Mapper lives in the integration module, not the domain module.
- Unit-test mappers with real sample data from the external system.

```java
@Component("erpOrderMapper")
public class ErpOrderMapper {
    public CanonicalOrder toCanonical(ErpPurchaseOrder erp) {
        return new CanonicalOrder(
            erp.getDocNum(), erp.getCardCode(),
            mapLines(erp.getDocumentLines()),
            erp.getDocTotal(), mapCurrency(erp.getDocCur()),
            mapStatus(erp.getDocStatus()));
    }
}
```

---

## 7. Service Registry

**Problem:** Consumers need to discover available services and their endpoints without hardcoding.

**Solution:** Services register themselves in a central registry. Consumers query the registry to discover service locations.

**Java OSS Implementation:**
- **ZooKeeper:** Services create ephemeral nodes at `/services/<name>/<instance-id>` with connection details. Consumers watch the parent node for changes.
- **Consul:** `camel-consul` component for health-checked service discovery.
- **Spring Cloud (if applicable):** Eureka or Consul via Spring Cloud Discovery.
- **Simpler alternative:** DNS-based discovery via Kubernetes services or load balancer.

---

## 8. Saga / Compensation

**Problem:** A business process spans multiple services, each with its own database. A traditional distributed transaction (2PC) is impractical in a microservices/SOA environment.

**Solution:** Execute the process as a sequence of local transactions. If any step fails, execute compensating transactions to undo the completed steps.

**Java OSS Implementation:**
```java
from("jms:queue:svc.order.fulfill")
    .routeId("order-fulfillment-saga")
    .saga()
        .propagation(SagaPropagation.REQUIRES_NEW)
        .compensation("direct:cancelReservation")
        .completion("direct:orderFulfilled")
    .to("direct:reserveInventory")
    .to("direct:chargePayment")
    .to("direct:scheduleShipping");

from("direct:cancelReservation")
    .bean("inventoryService", "cancelReservation")
    .bean("paymentService", "refund");
```

**When to use:** Multi-service business processes where atomicity across services is required.

**When to avoid:** Single-service operations (use local transactions). Operations where eventual consistency is unacceptable.

---

## 9. Event Sourcing

**Problem:** You need a complete audit trail of all state changes, and the ability to reconstruct state at any point in time.

**Solution:** Store every state change as an immutable event in an append-only log. Current state is derived by replaying events.

**Java OSS Implementation:**
- Publish events to Artemis topics: `event.order.created`, `event.order.updated`, `event.order.cancelled`.
- Store events in an event store (database table or Kafka topic).
- Build read-side projections by consuming the event stream.

---

## 10. CQRS (Command Query Responsibility Segregation)

**Problem:** Read and write workloads have different scaling and consistency requirements.

**Solution:** Separate the write model (handles commands, enforces invariants) from the read model (optimised for queries). Synchronise via events.

**Java OSS Implementation:**
- **Write side:** Spring Boot service receiving commands via JMS queue. Writes to primary database. Publishes events to Artemis topic.
- **Read side:** Camel route consumes events from topic. Updates denormalised read-optimised store (Redis, Elasticsearch, or RDBMS view).
- Hazelcast or Redis as the read cache for low-latency queries.

---

## 11. Transactional Outbox

**Problem:** You need to update a database and publish a message atomically. But the database and message broker are different systems — if the app crashes between the two, you get inconsistency.

**Solution:** Write the event to an "outbox" table in the same database transaction as the business data. A separate process (relay) polls the outbox and publishes to the message broker.

**Java OSS Implementation:**
```java
// Write service — single database transaction
@Transactional
public Order createOrder(CreateOrderRequest req) {
    Order order = orderRepository.save(toEntity(req));
    outboxRepository.save(new OutboxEvent(
        "order.created", order.getId().toString(),
        objectMapper.writeValueAsString(toCanonical(order))));
    return order;
}

// Outbox relay — Camel singleton route polls the outbox table
from("zookeeper-master:outbox-relay:sql:SELECT * FROM outbox WHERE published=false ORDER BY created_at?dataSource=#ds&consumer.delay=1000")
    .routeId("outbox-relay")
    .setHeader("JMSType", simple("${body[event_type]}"))
    .setBody(simple("${body[payload]}"))
    .to("jms:topic:event.${header.JMSType}")
    .transform(simple("UPDATE outbox SET published=true WHERE id=${body[id]}"))
    .to("sql:dummy?dataSource=#ds");
```

---

## 12. Idempotent Receiver

**Problem:** At-least-once delivery (standard in JMS) means a message may be delivered more than once. Processing it twice causes duplicates.

**Solution:** Track message IDs. If a message has already been processed, skip it.

**Java OSS Implementation:**
```java
// Hazelcast-backed idempotent repository (survives restarts, shared across instances)
from("jms:queue:canonical.orders")
    .idempotentConsumer(
        header("JMSMessageID"),
        HazelcastIdempotentRepository.hazelcastIdempotentRepository(hazelcast, "processed-messages"))
    .to("direct:processOrder");

// JDBC-backed (for persistence without Hazelcast)
from("jms:queue:canonical.orders")
    .idempotentConsumer(
        header("JMSMessageID"),
        new JdbcMessageIdRepository(dataSource, "message_id_repo"))
    .to("direct:processOrder");
```

---

## 13. Wire Tap / Audit

**Problem:** You need an audit trail of all messages flowing through the bus, without affecting the main processing flow.

**Solution:** Copy each message to an audit destination asynchronously.

```java
from("jms:queue:canonical.orders")
    .wireTap("jms:queue:audit.all-messages")
    .to("direct:processOrder");
```

---

## 14. Claim Check

**Problem:** Large payloads (files, images, large documents) should not flow through the message bus — they bloat the broker and slow processing.

**Solution:** Store the large payload externally (S3, filesystem, database). Pass only a reference (claim check) on the bus. The consumer retrieves the payload using the reference.

```java
// Store payload, pass reference
from("rest:post:/api/documents")
    .bean("storageService", "store")          // Returns claim check ID
    .setBody(simple("${header.claimCheckId}"))
    .to("jms:queue:canonical.documents");     // Only the ID flows on the bus

// Retrieve payload
from("jms:queue:document-processor")
    .bean("storageService", "retrieve")       // Fetches by claim check ID
    .to("direct:processDocument");
```

---

## 15. Message Expiration

**Problem:** Some messages are time-sensitive. If not processed within a window, they should be discarded or rerouted.

**Solution:** Set TTL (time-to-live) on messages. Expired messages go to an expiry address.

```java
from("direct:sendTimeSensitive")
    .setHeader("JMSExpiration", constant(300000))  // 5 minutes
    .to("jms:queue:time-sensitive-requests");
```

Artemis can route expired messages to a configured expiry address for monitoring.

---

## 16. Competing Consumers

**Problem:** A single consumer cannot keep up with the message volume.

**Solution:** Multiple consumer instances read from the same queue. The broker distributes messages round-robin.

```java
// Configure concurrent consumers
from("jms:queue:canonical.orders?concurrentConsumers=5&maxConcurrentConsumers=20")
    .to("direct:processOrder");
```

Combined with horizontal scaling (multiple Spring Boot instances), this provides elastic throughput.

---

## 17. Singleton Service

**Problem:** Some operations (batch jobs, polling legacy systems, report generation) must run on exactly one instance in the cluster.

**Solution:** Use distributed leader election to designate one instance as the singleton.

```java
// ZooKeeper-based singleton
from("zookeeper-master:daily-report:timer:report?period=86400000")
    .routeId("daily-report-singleton")
    .bean("reportService", "generateDailyReport");

// Hazelcast-based singleton (alternative)
FencedLock lock = hazelcast.getCPSubsystem().getLock("report-lock");
```

---

## 18. Distributed Lock

**Problem:** Multiple instances must coordinate access to a shared resource (file, external API with rate limits, database migration).

**Solution:** Acquire a distributed lock before accessing the resource. Release when done.

| Implementation | Technology | Consistency |
|---|---|---|
| Hazelcast `FencedLock` (CP subsystem) | Hazelcast | Strong (Raft) |
| ZooKeeper ephemeral sequential node | ZooKeeper | Strong (ZAB) |
| Redis `SETNX` + TTL | Redis (Redisson) | Weak (single-master) |
| Database advisory lock | PostgreSQL/MySQL | Strong (single-master) |

```java
// Hazelcast CP FencedLock (recommended for Java-native stacks)
FencedLock lock = hazelcast.getCPSubsystem().getLock("migration-lock");
lock.lock();
try {
    migrationService.run();
} finally {
    lock.unlock();
}
```
