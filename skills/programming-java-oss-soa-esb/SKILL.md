---
name: programming-java-oss-soa-esb
description: Expert guidance on implementing Service-Oriented Architecture (SOA) and Enterprise Service Bus (ESB) patterns for Enterprise Application Integration (EAI) using open-source Java technologies — Spring Boot, Apache Camel, Apache Artemis, Apache ZooKeeper, Hazelcast, and related OSS. Covers SOA design principles (Erl), Enterprise Integration Patterns (Hohpe & Woolf), canonical data models, service contract design, message bus architecture, protocol mediation, content-based routing, service orchestration and choreography, distributed coordination and leader election, shared-memory caches and in-memory data grids, canonical microservice layers, anti-corruption layers, service versioning, and production deployment topologies. This skill focuses on architecture-level guidance — how to compose the OSS stack into a coherent SOA/ESB platform — and refers to adjacent skills for implementation-level detail on individual technologies. Use this skill when the user is designing, architecting, or reviewing an enterprise integration platform, a message-driven SOA, an ESB layer, a canonical service bus, or a microservice integration backbone using Java OSS. Also use when the user asks about SOA principles, ESB patterns, canonical models, service contracts, protocol mediation, message routing topologies, or distributed coordination in a Java context, even if they don't explicitly say "SOA" or "ESB."
---

This skill guides the architecture and implementation of Service-Oriented and Enterprise Service Bus architectures for Enterprise Application Integration using open-source Java technologies. It focuses on *how to compose* the OSS stack — Spring Boot, Apache Camel, Apache Artemis, Apache ZooKeeper, Hazelcast, and others — into a coherent integration platform.

**Adjacent skills for implementation detail:**
- **`programming-java-spring-framework`** — Deep guidance on Spring Boot, Spring Data, Spring Security, Spring configuration
- **`programming-java-apache-camel`** — Deep guidance on Camel routes, EIPs, components, testing, error handling

This skill operates at the architecture level: choosing patterns, designing service layers, configuring infrastructure topology, and establishing conventions. When you need implementation-level code, consult the adjacent skills and return here for architectural guidance.

---

## Part I: SOA Design Principles

The eight principles from Thomas Erl's *SOA Principles of Service Design* form the design philosophy for any service-oriented system. These principles apply whether you are building SOAP services, REST APIs, message-driven microservices, or hybrid architectures.

### The Eight Principles

**1. Standardised Service Contract**
Every service exposes a formal, explicit contract that defines its interface. In Java OSS:
- REST APIs: OpenAPI/Swagger specifications. Generate from code or design contract-first.
- JMS services: Documented message schemas (JSON Schema, Avro, Protobuf). Published in a shared schema registry.
- Camel routes: REST DSL with `apiContextPath` for auto-generated OpenAPI docs.
- *Implementation:* Define DTOs/schemas in a shared Maven module. Consumers depend on the contract artifact, not the service implementation.

**2. Loose Coupling**
Services minimise dependencies on each other. Changes to one service do not cascade to others.
- Communicate via messages on a bus (Artemis JMS queues/topics), not via direct synchronous calls.
- Use a **canonical data model** as the lingua franca — services translate to/from canonical form at their boundaries.
- Never expose internal entities through service interfaces. Use DTOs and anti-corruption layers.
- *Implementation:* Camel routes with `marshal`/`unmarshal` at on-ramp and off-ramp. Spring Boot services behind Camel REST DSL or JMS consumers.

**3. Service Abstraction**
Hide implementation detail. Consumers see only the contract, not the technology, database, or internal structure.
- The ESB mediates: consumers address logical destinations (`jms:queue:order-service`), not physical hosts.
- Version the contract, not the implementation. Internal refactoring must not break consumers.
- *Implementation:* Camel's component model provides protocol abstraction. A route can switch from HTTP to JMS without changing business logic.

**4. Service Reusability**
Design services to be reused across multiple contexts, not tailored to a single consumer.
- **Entity services** (Customer, Order, Product) provide CRUD and query over core entities.
- **Utility services** (Email, Notification, Audit, File Transfer) provide common infrastructure.
- **Task/process services** orchestrate entity and utility services for specific business processes.
- *Implementation:* Package reusable services as independent Spring Boot applications. Expose via JMS and REST. Avoid consumer-specific logic.

**5. Service Autonomy**
Each service controls its own runtime environment and data.
- Own database per service (or at minimum, own schema). No shared mutable state between services.
- Own deployment — each service is an independent Spring Boot JAR with its own lifecycle.
- *Implementation:* Separate Maven modules/repos per service. Independent CI/CD pipelines. Artemis queues per service.

**6. Service Statelessness**
Services do not hold conversational state between invocations.
- State lives in the message (headers, body), in the database, or in a distributed cache (Hazelcast/Redis) — not in the service JVM.
- Enables horizontal scaling: any instance can handle any request.
- *Implementation:* Spring Boot stateless services. Session state in Hazelcast if needed. Camel routes are inherently stateless per exchange.

**7. Service Discoverability**
Services can be found and understood through metadata.
- OpenAPI specs published to a central registry or developer portal.
- JMS queue/topic naming conventions documented and enforced.
- Service catalogue (even a wiki or Git repo) listing all services, their contracts, owners, and SLAs.
- *Implementation:* Camel REST DSL auto-generates OpenAPI. Spring Boot Actuator `/info` endpoint carries service metadata.

**8. Service Composability**
Services can be assembled into larger composite processes.
- Orchestration: A process service calls entity services in sequence (Camel route or saga).
- Choreography: Services react to events on the bus independently (pub/sub via Artemis topics).
- *Implementation:* Camel routes compose services. Artemis topics for event-driven choreography. Camel saga EIP for distributed transactions.

---

## Part II: ESB Architecture with Java OSS

### The Logical ESB Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    SERVICE CONSUMERS                         │
│  (Web apps, mobile, partner systems, internal services)     │
└─────────────────┬───────────────────────────────────────────┘
                  │  REST / SOAP / JMS / gRPC
┌─────────────────▼───────────────────────────────────────────┐
│                   ON-RAMP (Ingress)                          │
│  • Protocol mediation (HTTP → JMS, SOAP → JSON)             │
│  • Authentication / authorisation                           │
│  • Schema validation                                        │
│  • Canonical model transformation                           │
│  (Apache Camel routes + Spring Security)                    │
└─────────────────┬───────────────────────────────────────────┘
                  │  Canonical messages
┌─────────────────▼───────────────────────────────────────────┐
│                   MESSAGE BUS                                │
│  • Reliable, persistent messaging                           │
│  • Queue-based (point-to-point) and topic-based (pub/sub)   │
│  • Dead letter queues for failed messages                   │
│  (Apache Artemis)                                           │
└─────────────────┬───────────────────────────────────────────┘
                  │  Canonical messages
┌─────────────────▼───────────────────────────────────────────┐
│                 MEDIATION / ROUTING                          │
│  • Content-based routing                                    │
│  • Message enrichment, splitting, aggregation               │
│  • Orchestration of service calls                           │
│  • Error handling, retry, circuit breaker                   │
│  (Apache Camel EIPs)                                        │
└─────────────────┬───────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────┐
│                 SERVICE PROVIDERS                            │
│  • Entity services (Spring Boot + Spring Data)              │
│  • Utility services (Email, Audit, File Transfer)           │
│  • Legacy adapters (Camel components)                       │
│  • External system connectors                               │
└─────────────────────────────────────────────────────────────┘

  CROSS-CUTTING:
  ┌─────────────────────────────────────────────────────────┐
  │ Coordination: Apache ZooKeeper (leader election, config)│
  │ Shared State:  Hazelcast (distributed cache, locks)     │
  │ Monitoring:    Spring Boot Actuator + Micrometer         │
  │ Security:      Spring Security + OAuth2/JWT              │
  └─────────────────────────────────────────────────────────┘
```

### The Technology Stack

| Layer | Technology | Role |
|-------|-----------|------|
| **Message Bus** | Apache Artemis | Persistent JMS messaging (queues + topics), high availability, clustering |
| **Integration Routes** | Apache Camel (Spring Boot) | Protocol mediation, transformation, routing, EIPs, error handling |
| **Service Runtime** | Spring Boot | Service hosting, dependency injection, configuration, REST, data access |
| **Coordination** | Apache ZooKeeper | Leader election, distributed configuration, singleton route control |
| **Shared State** | Hazelcast (or Redis, Infinispan) | Distributed caching, distributed locks, session state, idempotent repositories |
| **Security** | Spring Security + OAuth2 | Authentication, authorisation, token validation |
| **Observability** | Spring Boot Actuator + Micrometer | Health checks, metrics, distributed tracing |

---

## Part III: The Canonical Data Model

The canonical data model (CDM) is the single most important architectural decision in an ESB. It defines the common language spoken on the bus.

### Why a Canonical Model

Without a CDM, every system-to-system integration requires a bespoke transformation. With N systems, you need up to N×(N−1) transformations. With a CDM, each system needs only 2 transformations: native → canonical (on-ramp) and canonical → native (off-ramp). N systems = 2N transformations.

### Designing the CDM

1. **Identify core business entities.** Order, Customer, Product, Invoice, Payment — the nouns of your business.
2. **Define canonical schemas.** Use a technology-neutral format (JSON Schema, Avro, Protobuf). Store in a shared Git repo or schema registry.
3. **Include envelope metadata.** Every canonical message should carry: `messageId`, `correlationId`, `timestamp`, `sourceSystem`, `messageType`, `version`.
4. **Version from day one.** Use schema versioning (e.g., `com.example.canonical.order.v2`). Support backward-compatible evolution.
5. **Keep it business-focused.** The CDM represents business concepts, not the internal structure of any one system.

### Implementation Pattern

```java
// Canonical message envelope
public record CanonicalMessage<T>(
    String messageId,
    String correlationId,
    Instant timestamp,
    String sourceSystem,
    String messageType,
    int version,
    T payload
) {}

// Canonical Order entity
public record CanonicalOrder(
    String orderId,
    String customerId,
    List<CanonicalOrderLine> lines,
    BigDecimal totalAmount,
    String currency,
    OrderStatus status
) {}
```

```java
// On-ramp: System A native format → Canonical
@Component
public class SystemAOnRamp extends RouteBuilder {
    @Override
    public void configure() {
        from("jms:queue:systemA.orders.inbound")
            .routeId("systemA-onramp")
            .unmarshal().json(JsonLibrary.Jackson, SystemAOrder.class)
            .bean("systemAMapper", "toCanonical")   // Anti-corruption layer
            .marshal().json(JsonLibrary.Jackson)
            .to("jms:queue:canonical.orders");       // Canonical bus
    }
}

// Off-ramp: Canonical → System B native format
@Component
public class SystemBOffRamp extends RouteBuilder {
    @Override
    public void configure() {
        from("jms:queue:canonical.orders.systemB")
            .routeId("systemB-offramp")
            .unmarshal().json(JsonLibrary.Jackson, CanonicalOrder.class)
            .bean("systemBMapper", "fromCanonical")  // Anti-corruption layer
            .marshal().json(JsonLibrary.Jackson)
            .to("http:systemB.internal:8080/api/orders");
    }
}
```

---

## Part IV: Message Bus — Apache Artemis

Apache Artemis is the message backbone of the ESB. It provides persistent, transactional, high-availability messaging.

### Deployment Topologies

**Standalone (Development/Small):**
Single broker. Simple. No HA.

**Primary/Backup with Shared Storage (Production):**
Two brokers share a journal (NFS, SAN, or shared database). Primary serves clients; backup takes over if primary fails. ZooKeeper coordinates failover.

```
Primary (active) ──── Shared Journal ──── Backup (passive)
         └───── ZooKeeper Ensemble ─────┘
```

**Symmetric Cluster (High Throughput):**
Multiple live brokers. Messages distributed across the cluster. Each broker has its own backup.

### Queue Design Conventions

| Convention | Pattern | Example |
|---|---|---|
| **Service input** | `svc.<service>.<operation>` | `svc.order.create` |
| **Canonical bus** | `canonical.<entity>` | `canonical.orders` |
| **Dead letter** | `dlq.<original-queue>` | `dlq.svc.order.create` |
| **Events (topics)** | `event.<entity>.<action>` | `event.order.created` |
| **System-specific** | `<system>.<entity>.<direction>` | `erp.orders.inbound` |

### Spring Boot Configuration

```yaml
spring:
  artemis:
    mode: native
    broker-url: tcp://artemis-primary:61616?ha=true&retryInterval=1000&retryIntervalMultiplier=2&maxRetryInterval=30000&reconnectAttempts=-1
    user: ${ARTEMIS_USER}
    password: ${ARTEMIS_PASS}
    pool:
      enabled: true
      max-connections: 10
```

### Transactional Messaging with Camel

```java
from("jms:queue:svc.payment.process?transacted=true")
    .transacted("PROPAGATION_REQUIRED")
    .bean("paymentService", "process")
    .to("jms:queue:event.payment.processed");
// Commit: both consume and produce succeed atomically
// Rollback: message returns to source queue for redelivery
```

---

## Part V: Distributed Coordination — ZooKeeper

ZooKeeper provides the distributed coordination primitives that make the ESB cluster-safe.

### Use Cases in the ESB

| Use Case | Pattern | Implementation |
|---|---|---|
| **Singleton route** | Only one instance of a route runs across the cluster | Camel `master:` or `zookeeper-master:` component |
| **Leader election** | One service instance is the "leader" for coordination tasks | ZooKeeper ephemeral sequential nodes |
| **Distributed config** | Shared configuration across all instances | ZK nodes watched by services |
| **Service registry** | Services register themselves; consumers discover them | ZK ephemeral nodes |

### Singleton Routes (Critical for Polling Consumers)

Many integrations poll legacy systems that cannot handle concurrent readers. Only one instance should poll at a time, with automatic failover.

```java
// Only one instance across the cluster polls the legacy system
from("zookeeper-master:legacy-poll:sql:SELECT * FROM legacy.outbox WHERE sent=false?dataSource=#legacyDs&consumer.delay=5000")
    .routeId("legacy-poller-singleton")
    .bean("legacyProcessor", "process")
    .to("jms:queue:canonical.orders");
```

```yaml
camel:
  component:
    zookeeper-master:
      zookeeper-url: zk1:2181,zk2:2181,zk3:2181
      base-path: /esb/master
```

---

## Part VI: Shared State — Distributed Caching

### Cache Selection Guide

| Technology | Best For | Spring Boot Starter |
|---|---|---|
| **Hazelcast** | Distributed data structures, locks, near-cache, embedded mode, Java-native | `hazelcast-spring` |
| **Redis** | Simple key-value caching, pub/sub, widely deployed | `spring-boot-starter-data-redis` |
| **Infinispan** | Clustered caching, JCache standard, Red Hat ecosystem | `infinispan-spring-boot3-starter-embedded` |
| **Caffeine** | Local (non-distributed) L1 cache, ultra-fast, JVM-only | `com.github.ben-manes.caffeine:caffeine` |
| **Apache Ignite** | Distributed compute + cache, SQL over cache, heavy-duty IMDG | `ignite-spring-boot-autoconfigure` |

### Hazelcast in the ESB

Hazelcast provides embedded clustering — no external server required. Each Spring Boot service JVM joins the Hazelcast cluster and shares state.

**Common use cases:**
- **Idempotent repository:** Prevent duplicate message processing across instances.
- **Distributed locks:** Coordinate access to shared resources (e.g., singleton batch jobs).
- **Near-cache:** Cache frequently accessed reference data locally with cluster-wide invalidation.
- **Distributed map:** Share session state, routing tables, or circuit breaker state.

```java
@Configuration
public class HazelcastConfig {
    @Bean
    Config hazelcastConfig() {
        Config config = new Config();
        config.setClusterName("esb-cluster");

        // Distributed map for idempotent message IDs
        MapConfig idempotentMap = new MapConfig("idempotent-repo")
            .setTimeToLiveSeconds(3600)     // Expire after 1 hour
            .setMaxIdleSeconds(1800)
            .setEvictionConfig(new EvictionConfig()
                .setMaxSizePolicy(MaxSizePolicy.PER_NODE)
                .setSize(100000));
        config.addMapConfig(idempotentMap);

        // Reference data cache with near-cache
        NearCacheConfig nearCacheConfig = new NearCacheConfig()
            .setTimeToLiveSeconds(300)
            .setMaxIdleSeconds(60)
            .setInvalidateOnChange(true);    // Invalidate when another node updates
        MapConfig refDataMap = new MapConfig("reference-data")
            .setNearCacheConfig(nearCacheConfig);
        config.addMapConfig(refDataMap);

        return config;
    }
}
```

```java
// Camel idempotent consumer backed by Hazelcast
@Autowired HazelcastInstance hazelcast;

from("jms:queue:canonical.orders")
    .idempotentConsumer(
        header("JMSMessageID"),
        HazelcastIdempotentRepository.hazelcastIdempotentRepository(
            hazelcast, "idempotent-repo"))
    .to("direct:processOrder");
```

### CP Subsystem (Strong Consistency)

For distributed locks and leader election where correctness matters more than availability, use Hazelcast's CP subsystem (based on Raft consensus):

```java
// Distributed lock
FencedLock lock = hazelcast.getCPSubsystem().getLock("batch-job-lock");
if (lock.tryLock(5, TimeUnit.SECONDS)) {
    try {
        // Only one instance executes this
        batchService.runDailyReport();
    } finally {
        lock.unlock();
    }
}
```

---

## Part VII: Service Layer Patterns

### The Three Service Layers (Erl)

| Layer | Purpose | Examples | Communication |
|-------|---------|---------|---------------|
| **Entity Services** | CRUD and query over core business entities | OrderService, CustomerService, ProductService | Synchronous (REST) + Events (JMS topics) |
| **Utility Services** | Cross-cutting infrastructure capabilities | EmailService, AuditService, FileTransferService, NotificationService | Async (JMS queues) |
| **Process/Task Services** | Orchestrate entity and utility services for business processes | OrderFulfillmentProcess, CustomerOnboarding | Camel orchestration routes |

### Anti-Corruption Layer Pattern

Protect your canonical model from the specifics of external systems:

```java
// Anti-corruption layer: maps between external system model and canonical model
@Component
public class ErpOrderMapper {

    public CanonicalOrder toCanonical(ErpOrderRecord erp) {
        return new CanonicalOrder(
            erp.getDocNum(),                                     // Different field name
            erp.getBpCode(),                                     // Different concept name
            erp.getLines().stream().map(this::mapLine).toList(),
            erp.getDocTotal(),
            mapCurrency(erp.getCurrCode()),                      // Different currency coding
            mapStatus(erp.getDocStatus())                        // Different status enum
        );
    }

    public ErpOrderRecord fromCanonical(CanonicalOrder order) {
        // Reverse mapping
    }
}
```

**Rules:**
- One ACL per external system, per direction (inbound/outbound).
- ACL lives in the integration layer (Camel route), not in the service layer.
- ACL is the *only* place that knows about the external system's data format.
- Test ACLs thoroughly with realistic sample data from the external system.

### Service Contract Versioning

| Strategy | Approach | When to Use |
|----------|----------|-------------|
| **URI versioning** | `/api/v1/orders`, `/api/v2/orders` | REST APIs with breaking changes |
| **Header versioning** | `Accept: application/vnd.myapp.v2+json` | REST APIs needing clean URLs |
| **Schema versioning** | Avro/Protobuf schema evolution, canonical schema `version` field | JMS/event messages |
| **Queue versioning** | `svc.order.create.v1`, `svc.order.create.v2` | Message bus migrations |
| **Backward-compatible evolution** | Add fields (never remove/rename) | Preferred default — no version bump needed |

**Golden rule:** Prefer backward-compatible evolution. Only create new versions when breaking changes are unavoidable.

### Orchestration vs. Choreography

| | Orchestration | Choreography |
|---|---|---|
| **Control** | Central coordinator (Camel route) directs the flow | Services react to events independently |
| **Coupling** | Higher (coordinator knows all participants) | Lower (services only know about events) |
| **Visibility** | Flow is visible in one place | Flow is distributed across services |
| **Error handling** | Centralised (saga/compensation in the orchestrator) | Distributed (each service handles its own) |
| **Best for** | Complex business processes with ordering, compensation, human tasks | Simple event-driven flows, notifications, denormalisation |

**Implementation:**

```java
// ORCHESTRATION — Camel saga with compensation
from("jms:queue:svc.order.fulfill")
    .routeId("order-fulfillment-orchestration")
    .saga()
        .compensation("direct:cancelOrder")
    .to("direct:reserveInventory")
    .to("direct:chargePayment")
    .to("direct:scheduleShipping")
    .to("jms:queue:event.order.fulfilled");

// CHOREOGRAPHY — Services react to events independently
// Order Service publishes:
from("direct:orderCreated")
    .to("jms:topic:event.order.created");

// Inventory Service subscribes:
from("jms:topic:event.order.created")
    .routeId("inventory-reserve-on-order")
    .bean("inventoryService", "reserve");

// Notification Service subscribes:
from("jms:topic:event.order.created")
    .routeId("notify-on-order")
    .bean("notificationService", "sendOrderConfirmation");
```

---

## Part VIII: Production Deployment Topology

### Minimal HA Topology

```
Load Balancer (nginx / HAProxy)
      │
      ├── ESB Instance 1 (Spring Boot + Camel)
      ├── ESB Instance 2 (Spring Boot + Camel)
      └── ESB Instance 3 (Spring Boot + Camel)
            │
            ▼
      Artemis Cluster
      ├── Broker 1 (primary) + Broker 1-backup
      └── Broker 2 (primary) + Broker 2-backup
            │
            ▼
      ZooKeeper Ensemble
      ├── ZK Node 1
      ├── ZK Node 2
      └── ZK Node 3
            │
            ▼
      Hazelcast Cluster (embedded in ESB instances)
```

### Deployment Checklist

- [ ] Artemis configured for HA (primary/backup or symmetric cluster)
- [ ] ZooKeeper ensemble has 3+ nodes (odd number for quorum)
- [ ] Singleton routes use `zookeeper-master:` or `master:` component
- [ ] Dead letter queues configured for all service queues
- [ ] Idempotent consumers configured for at-least-once delivery
- [ ] Canonical model schemas versioned and published
- [ ] ACL mappers tested with real data samples from each external system
- [ ] Health checks exposed via Actuator for all services
- [ ] Distributed tracing enabled (OpenTelemetry)
- [ ] Graceful shutdown configured (Camel + Spring Boot)
- [ ] Alerting on DLQ depth, consumer lag, route failures

---

## Reference Files

- **`references/soa-patterns.md`** — Detailed reference for SOA and EAI architecture patterns: Canonical Data Model, Protocol Mediation, Content-Based Routing, Scatter-Gather, Message Broker, Service Registry, Saga/Compensation, Event Sourcing, CQRS, and others. Each pattern includes: problem, solution, Java OSS implementation, and when to use/avoid.

For implementation-level guidance on individual technologies, consult:
- **Skill: `programming-java-spring-framework`** — Spring Boot, Spring Data, Spring Security
- **Skill: `programming-java-apache-camel`** — Camel routes, EIPs, components, testing