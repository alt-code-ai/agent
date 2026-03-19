---
name: programming-java-apache-camel
description: Expert guidance and automation for programming in Java with the Apache Camel integration framework, with a focus on Spring Boot integration. Covers route building (Java DSL, YAML DSL), Enterprise Integration Patterns (EIPs), CamelContext lifecycle, component configuration, error handling (Dead Letter Channel, onException, retries), data transformation (marshal/unmarshal, type converters, data formats), testing (MockEndpoint, AdviceWith, ProducerTemplate, camel-test-spring-junit5), and production deployment. Provides deep guidance on core distributed systems components: JMS, HTTP, REST, master/singleton routes, ZooKeeper leader election, and all 300+ official Camel components. Synthesises techniques from the official Apache Camel documentation, Camel in Action (Ibsen & Anstey), and Enterprise Integration Patterns (Hohpe & Woolf). Use this skill whenever the user is building, debugging, configuring, or architecting a Camel application — integration routes, message-driven microservices, protocol bridges, data pipelines, ETL flows, or any Java project using Apache Camel with Spring Boot. Also use when the user asks about Camel routes, EIPs, Camel components, message routing, message transformation, or integration patterns, even if they don't explicitly mention "Camel." Downstream projects (Camel K, Karavan, JBoss Fuse) are out of scope — only the core Camel monorepo is covered.
---

This skill guides Java development with the Apache Camel integration framework, focused on Spring Boot as the runtime. It provides idiomatic patterns, production-ready configurations, and deep guidance on Camel's core architecture and its 300+ components.

The user may be:

1. **Building** — creating integration routes, connecting systems, transforming messages
2. **Debugging** — diagnosing route failures, message transformation issues, component configuration problems
3. **Architecting** — designing integration topology, choosing components, planning error handling and resilience
4. **Testing** — writing unit and integration tests for Camel routes
5. **Operating** — deploying Camel applications, monitoring routes, managing lifecycle

Default to **Camel 4.x** with **Spring Boot 3.x** unless the user specifies otherwise. Camel 4.x requires Java 17+ and uses the `jakarta` namespace.

---

## Part I: Core Architecture

### CamelContext

The `CamelContext` is the runtime container that manages the lifecycle of all routes, components, endpoints, and type converters. In Spring Boot, it is auto-configured by `camel-spring-boot-starter`.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.apache.camel.springboot</groupId>
    <artifactId>camel-spring-boot-starter</artifactId>
    <version>${camel.version}</version>
</dependency>
```

```yaml
# application.yml
camel:
  springboot:
    name: my-integration-service
    main-run-controller: true   # Keep the app alive (no web dependency needed)
  component:
    jms:
      connection-factory: "#connectionFactory"  # Reference a Spring bean
```

Spring Boot auto-discovers any `RouteBuilder` beans in the application context and adds their routes to the `CamelContext`.

### Routes

A route is a pipeline from a source (consumer endpoint) through processing steps to one or more destination (producer) endpoints. Routes implement Enterprise Integration Patterns.

```java
@Component
public class OrderRoute extends RouteBuilder {

    @Override
    public void configure() {
        // Error handling for all routes in this builder
        errorHandler(deadLetterChannel("jms:queue:dead-letter")
            .maximumRedeliveries(3)
            .redeliveryDelay(1000)
            .retryAttemptedLogLevel(LoggingLevel.WARN));

        from("jms:queue:incoming-orders")
            .routeId("process-orders")
            .log("Received order: ${body}")
            .unmarshal().json(JsonLibrary.Jackson, Order.class)
            .choice()
                .when(simple("${body.priority} == 'HIGH'"))
                    .to("direct:high-priority")
                .when(simple("${body.priority} == 'NORMAL'"))
                    .to("direct:normal-priority")
                .otherwise()
                    .to("direct:low-priority")
            .end()
            .marshal().json(JsonLibrary.Jackson)
            .to("jms:queue:processed-orders");
    }
}
```

### Exchanges and Messages

Every message flowing through a route is wrapped in an `Exchange`. The exchange carries:

- **In Message** — the current message (body + headers)
- **Out Message** — (deprecated pattern; use in-only in modern Camel)
- **Exchange Properties** — metadata scoped to the exchange (survives routing, not sent to endpoints)
- **Exchange Pattern** — `InOnly` (one-way) or `InOut` (request-reply)

**Body:** The message payload. Can be any Java object. Camel's type converter system automatically converts between types.

**Headers:** Key-value metadata. Used for routing decisions, component-specific parameters (e.g., `CamelHttpMethod`, `JMSReplyTo`), and passing data between processors.

**Properties:** Exchange-scoped state. Unlike headers, properties are not sent to external endpoints.

### Endpoints and URIs

Every component exposes endpoints via URIs:

```
component:path[?options]
```

Examples:
```
jms:queue:orders                        # JMS queue
jms:topic:notifications                 # JMS topic
http:api.example.com/orders             # HTTP producer
rest:get:orders/{id}                    # REST consumer
file:/data/inbox?noop=true              # File consumer (don't move files)
timer:heartbeat?period=30000            # Timer fires every 30s
direct:processOrder                     # Synchronous in-memory
seda:asyncProcess?concurrentConsumers=5 # Async in-memory with thread pool
kafka:orders?brokers=localhost:9092     # Kafka
sql:SELECT * FROM orders?dataSource=#ds # SQL polling
```

### Component Configuration

Components can be configured at three levels:

**1. In the URI (endpoint level):**
```java
from("jms:queue:orders?concurrentConsumers=5&acknowledgementModeName=CLIENT_ACKNOWLEDGE")
```

**2. In application.yml (component level):**
```yaml
camel:
  component:
    jms:
      connection-factory: "#connectionFactory"
      concurrent-consumers: 5
      acknowledgement-mode-name: CLIENT_ACKNOWLEDGE
```

**3. In Java (component bean):**
```java
@Bean
JmsComponent jms(ConnectionFactory connectionFactory) {
    JmsComponent component = JmsComponent.jmsComponentAutoAcknowledge(connectionFactory);
    component.setConcurrentConsumers(5);
    return component;
}
```

---

## Part II: Enterprise Integration Patterns

Camel implements most patterns from Hohpe & Woolf's *Enterprise Integration Patterns*. These are the building blocks of all integration routes.

### Message Routing Patterns

| Pattern | DSL | Purpose |
|---------|-----|---------|
| **Content-Based Router** | `choice().when().otherwise()` | Route to different destinations based on message content |
| **Message Filter** | `filter()` | Drop messages that don't match a predicate |
| **Recipient List** | `recipientList()` | Route to a dynamic list of destinations |
| **Splitter** | `split()` | Break a message into parts, process each independently |
| **Aggregator** | `aggregate()` | Combine multiple messages into one |
| **Dynamic Router** | `dynamicRouter()` | Route step-by-step, with each step determining the next |
| **Routing Slip** | `routingSlip()` | Route through a pre-defined sequence of endpoints |
| **Multicast** | `multicast()` | Send the same message to multiple destinations in parallel |
| **Wire Tap** | `wireTap()` | Send a copy to a secondary destination without affecting the main flow |
| **Load Balancer** | `loadBalance()` | Distribute messages across multiple endpoints |

### Message Transformation Patterns

| Pattern | DSL | Purpose |
|---------|-----|---------|
| **Content Enricher** | `enrich()` / `pollEnrich()` | Add data from an external source |
| **Message Translator** | `transform()` / `process()` | Transform message structure/format |
| **Marshal/Unmarshal** | `marshal()` / `unmarshal()` | Serialise/deserialise (JSON, XML, CSV, Avro, Protobuf) |
| **Claim Check** | `claimCheck()` | Temporarily store payload; retrieve later |

### Error Handling Patterns

| Pattern | DSL | Purpose |
|---------|-----|---------|
| **Dead Letter Channel** | `deadLetterChannel()` | Route failed messages to an error queue |
| **Retry / Redelivery** | `.maximumRedeliveries()` | Retry failed processing with configurable delay |
| **On Exception** | `onException()` | Handle specific exception types with custom logic |
| **Circuit Breaker** | `.circuitBreaker()` (Resilience4j) | Prevent cascading failures |
| **Throttler** | `throttle()` | Limit message throughput |

### Error Handling Configuration

```java
@Component
public class MyRoutes extends RouteBuilder {
    @Override
    public void configure() {
        // Global error handler for all routes in this builder
        errorHandler(deadLetterChannel("jms:queue:errors")
            .maximumRedeliveries(5)
            .redeliveryDelay(2000)
            .backOffMultiplier(2)       // Exponential backoff
            .retryAttemptedLogLevel(LoggingLevel.WARN)
            .useOriginalMessage());     // Send original message to DLQ

        // Specific exception handling
        onException(ValidationException.class)
            .handled(true)              // Don't propagate further
            .maximumRedeliveries(0)     // No retries for validation errors
            .to("jms:queue:validation-errors");

        onException(ConnectException.class)
            .maximumRedeliveries(10)
            .redeliveryDelay(5000)
            .logRetryAttempted(true);

        from("jms:queue:incoming")
            .routeId("main-route")
            .to("direct:process");
    }
}
```

**Error handling rules:**
- `errorHandler()` applies to all routes in that `RouteBuilder`.
- `onException()` takes precedence over `errorHandler()` for matching exception types.
- `handled(true)` means Camel considers the exception resolved — the original caller won't see it.
- `continued(true)` means Camel continues processing the route as if no error occurred.
- `useOriginalMessage()` sends the original input to the DLQ, not the partially-processed version.

---

## Part III: Distributed Systems Components (Deep Dive)

For the complete catalogue of all 300+ Camel components, see **`references/component-catalogue.md`**.

### JMS (`camel-jms`)

The most important component for enterprise messaging. Works with any JMS-compliant broker (ActiveMQ, Artemis, IBM MQ, TIBCO).

```yaml
# application.yml for ActiveMQ Artemis
spring:
  artemis:
    broker-url: tcp://localhost:61616
    user: admin
    password: admin

camel:
  component:
    jms:
      connection-factory: "#jmsConnectionFactory"
      concurrent-consumers: 5
      max-concurrent-consumers: 10
      acknowledgement-mode-name: CLIENT_ACKNOWLEDGE
      transacted: true
      cache-level-name: CACHE_CONSUMER
```

**Key patterns:**

```java
// Queue consumer
from("jms:queue:orders?concurrentConsumers=5")
    .to("direct:processOrder");

// Topic subscriber (durable)
from("jms:topic:events?durableSubscriptionName=myApp&clientId=myApp-1")
    .to("direct:handleEvent");

// Request-Reply (InOut)
from("direct:getPrice")
    .to(ExchangePattern.InOut, "jms:queue:price-service?requestTimeout=30000");

// Transacted route (commit/rollback with broker)
from("jms:queue:payments?transacted=true")
    .transacted()
    .to("bean:paymentService")
    .to("jms:queue:confirmations");

// Selector (filter at broker level)
from("jms:queue:events?selector=type='ORDER'")
    .to("direct:orderEvents");
```

### HTTP / HTTPS (`camel-http`)

```java
// HTTP producer (call external API)
from("direct:callApi")
    .setHeader(Exchange.HTTP_METHOD, constant("POST"))
    .setHeader(Exchange.CONTENT_TYPE, constant("application/json"))
    .to("http:api.example.com/orders?bridgeEndpoint=true");

// With timeout and error handling
from("direct:callApi")
    .to("http:api.example.com/orders"
        + "?httpClient.connectTimeout=5000"
        + "&httpClient.socketTimeout=30000"
        + "&throwExceptionOnFailure=true");
```

For exposing REST APIs, use the REST DSL (see below) rather than the HTTP component.

### REST DSL (`camel-rest`)

```java
@Component
public class OrderRestRoute extends RouteBuilder {
    @Override
    public void configure() {
        restConfiguration()
            .component("servlet")           // Use Spring MVC servlet
            .bindingMode(RestBindingMode.json)
            .dataFormatProperty("prettyPrint", "true")
            .apiContextPath("/api-doc")
            .apiProperty("api.title", "Order API")
            .apiProperty("api.version", "1.0");

        rest("/api/v1/orders")
            .get()
                .to("direct:listOrders")
            .get("/{id}")
                .to("direct:getOrder")
            .post()
                .type(CreateOrderRequest.class)
                .to("direct:createOrder")
            .put("/{id}")
                .type(UpdateOrderRequest.class)
                .to("direct:updateOrder")
            .delete("/{id}")
                .to("direct:deleteOrder");
    }
}
```

### Master / Singleton Routes (`camel-master`)

Ensures only one instance of a route runs across a cluster. Critical for consumers that must not run concurrently (e.g., polling a legacy system).

```java
// Only one instance across the cluster will consume from this JMS queue
from("master:my-lock:jms:queue:legacy-system")
    .routeId("singleton-legacy-consumer")
    .to("direct:processLegacyMessage");
```

The `master` component requires a `CamelClusterService` implementation. Options:

| Implementation | Dependency | Backend |
|---|---|---|
| ZooKeeper | `camel-zookeeper-master` | Apache ZooKeeper |
| Consul | `camel-consul` | HashiCorp Consul |
| Kubernetes | `camel-kubernetes` | Kubernetes ConfigMaps/Leases |
| File-based | `camel-file` | Shared filesystem (dev only) |

### ZooKeeper Integration

**ZooKeeper Master (`camel-zookeeper-master`):**

```java
// Leader election via ZooKeeper
from("zookeeper-master:my-app:jms:queue:singleton-queue")
    .to("direct:process");
```

```yaml
camel:
  component:
    zookeeper-master:
      zookeeper-url: zk1:2181,zk2:2181,zk3:2181
      base-path: /camel/master
```

**ZooKeeper Component (`camel-zookeeper`):**

```java
// Read ZooKeeper node
from("zookeeper:localhost:2181/config/app?repeat=true")
    .log("Config changed: ${body}");

// Write to ZooKeeper node
from("direct:updateConfig")
    .to("zookeeper:localhost:2181/config/app?createMode=PERSISTENT");
```

**ZooKeeper Route Policy:**

```java
ZooKeeperRoutePolicy policy = new ZooKeeperRoutePolicy(
    "zookeeper:localhost:2181/camel/leader", 1);
from("jms:queue:work")
    .routePolicy(policy)
    .to("direct:process");
```

---

## Part IV: Data Transformation

### Data Formats

```java
// JSON (Jackson)
from("jms:queue:orders")
    .unmarshal().json(JsonLibrary.Jackson, Order.class)
    .process(exchange -> {
        Order order = exchange.getIn().getBody(Order.class);
        order.setStatus("PROCESSING");
    })
    .marshal().json(JsonLibrary.Jackson)
    .to("jms:queue:processed");

// XML (JAXB)
from("file:/data/inbox")
    .unmarshal().jaxb("com.example.model")
    .to("direct:process");

// CSV
from("file:/data/reports")
    .unmarshal().csv()
    .split(body())
    .to("direct:processRow");

// Avro, Protobuf, YAML, CBOR — same pattern
```

### Type Converters

Camel automatically converts between types when possible:

```java
// Camel converts String body to byte[] for JMS
from("direct:send")
    .setBody(constant("hello"))   // String
    .to("jms:queue:test");        // Auto-converts to byte[]

// Explicit conversion
from("direct:process")
    .convertBodyTo(String.class)
    .to("log:output");
```

### Processors and Beans

```java
// Inline processor
from("direct:process")
    .process(exchange -> {
        String body = exchange.getIn().getBody(String.class);
        exchange.getIn().setBody(body.toUpperCase());
    })
    .to("mock:result");

// Bean method call
from("direct:process")
    .bean(OrderService.class, "validate")
    .bean(OrderService.class, "enrich")
    .bean(OrderService.class, "save");

// Spring bean reference
from("direct:process")
    .bean("orderService", "process");
```

---

## Part V: Testing

### Spring Boot Test Setup

```xml
<dependency>
    <groupId>org.apache.camel.springboot</groupId>
    <artifactId>camel-test-spring-junit5-starter</artifactId>
    <version>${camel.version}</version>
    <scope>test</scope>
</dependency>
```

### Basic Route Test

```java
@CamelSpringBootTest
@SpringBootTest
@EnableAutoConfiguration
class OrderRouteTest {

    @Autowired
    ProducerTemplate producerTemplate;

    @EndpointInject("mock:result")
    MockEndpoint mockResult;

    @Configuration
    static class TestConfig {
        @Bean
        RouteBuilder testRoute() {
            return new RouteBuilder() {
                @Override
                public void configure() {
                    from("direct:test")
                        .transform(simple("Hello ${body}"))
                        .to("mock:result");
                }
            };
        }
    }

    @Test
    void shouldTransformMessage() throws Exception {
        mockResult.expectedMessageCount(1);
        mockResult.expectedBodiesReceived("Hello World");

        producerTemplate.sendBody("direct:test", "World");

        mockResult.assertIsSatisfied();
    }
}
```

### AdviceWith (Intercept and Replace Endpoints)

Replace real endpoints with mocks during testing without changing the route:

```java
@CamelSpringBootTest
@SpringBootTest
@UseAdviceWith   // Prevents routes from starting automatically
class OrderRouteAdviceTest {

    @Autowired CamelContext camelContext;
    @Autowired ProducerTemplate producerTemplate;
    @EndpointInject("mock:jms:queue:processed") MockEndpoint mockProcessed;

    @Test
    void shouldRouteHighPriorityOrders() throws Exception {
        AdviceWith.adviceWith(camelContext, "process-orders", a -> {
            // Replace the JMS producer with a mock
            a.weaveByToUri("jms:queue:processed-orders")
                .replace().to("mock:jms:queue:processed");
            // Replace the JMS consumer with a direct endpoint
            a.replaceFromWith("direct:start");
        });

        camelContext.start();

        mockProcessed.expectedMessageCount(1);
        producerTemplate.sendBody("direct:start", "{\"priority\":\"HIGH\",\"item\":\"widget\"}");
        mockProcessed.assertIsSatisfied();
    }
}
```

**Testing rules:**
- Use `@UseAdviceWith` when you need to modify routes before they start.
- Use `MockEndpoint` to set expectations on messages received.
- Use `ProducerTemplate` to send test messages into routes.
- Use `AdviceWith.replaceFromWith()` to replace real consumers (JMS, Kafka) with `direct:` endpoints.
- Use `AdviceWith.weaveByToUri().replace()` to intercept producers.
- Always call `mockEndpoint.assertIsSatisfied()` to verify expectations.

---

## Part VI: Production Configuration

### Graceful Shutdown

```yaml
camel:
  springboot:
    main-run-controller: true
    duration-max-seconds: -1       # Run indefinitely
    shutdown-timeout: 30           # Wait 30s for in-flight exchanges
    shutdown-routes-in-reverse-order: true
```

### Route Management

```java
// Start/stop routes programmatically
camelContext.getRouteController().startRoute("my-route");
camelContext.getRouteController().stopRoute("my-route");

// Suspend/resume (gentler — completes in-flight)
camelContext.getRouteController().suspendRoute("my-route");
camelContext.getRouteController().resumeRoute("my-route");
```

### JMX and Monitoring

```yaml
camel:
  springboot:
    jmx-enabled: true
    jmx-management-statistics-level: Extended
  component:
    metrics:
      enabled: true
```

Combine with Spring Boot Actuator for health checks:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,camelroutes
  endpoint:
    health:
      show-details: always
```

Camel auto-registers health checks for each route and component.

### Idempotent Consumer

Prevent duplicate message processing:

```java
from("jms:queue:orders")
    .idempotentConsumer(
        header("JMSMessageID"),
        MemoryIdempotentRepository.memoryIdempotentRepository(1000))
    .to("direct:process");

// JDBC-backed for persistence across restarts
from("jms:queue:orders")
    .idempotentConsumer(
        header("OrderId"),
        new JdbcMessageIdRepository(dataSource, "idempotent_repo"))
    .to("direct:process");
```

---

## Part VII: Common Patterns and Recipes

### File-based Integration

```java
// Poll directory, process XML files, move to done/error
from("file:/data/inbox?include=.*\\.xml&move=../done&moveFailed=../error")
    .unmarshal().jaxb("com.example.model")
    .to("direct:process");
```

### REST API Proxy / Gateway

```java
from("rest:get:proxy/{path}")
    .toD("http:backend-service:8080/${header.path}?bridgeEndpoint=true");
```

### Scheduled Polling + Aggregation

```java
from("sql:SELECT * FROM events WHERE processed=false?dataSource=#ds&consumer.delay=5000")
    .aggregate(constant(true), new GroupedBodyAggregationStrategy())
    .completionSize(100)
    .completionTimeout(10000)
    .marshal().json(JsonLibrary.Jackson)
    .to("http:analytics-service.example.com/batch");
```

### Circuit Breaker (Resilience4j)

```java
from("direct:callExternalService")
    .circuitBreaker()
        .resilience4jConfiguration()
            .failureRateThreshold(50)
            .waitDurationInOpenState(10000)
            .slidingWindowSize(10)
        .end()
        .to("http:external-service.example.com/api")
    .onFallback()
        .setBody(constant("{\"status\":\"fallback\"}"))
    .end();
```

---

## Reference Files

- **`references/component-catalogue.md`** — Complete catalogue of all 300+ official Apache Camel components, grouped by category (Core, Messaging, HTTP/REST, Data, Cloud, Social, IoT, etc.). Each entry includes: artifact name, URI format, brief description, and whether it supports consumer/producer/both. Use when choosing a component or looking up its artifact dependency.

- **`references/eip-reference.md`** — Detailed reference for all Enterprise Integration Patterns implemented by Camel, with Java DSL examples, configuration options, and usage guidance for each pattern.