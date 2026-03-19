# Enterprise Integration Patterns Reference

Detailed reference for all EIPs implemented by Apache Camel, with Java DSL examples and key configuration options. Based on patterns from Hohpe & Woolf's *Enterprise Integration Patterns*.

## Table of Contents

1. [Message Routing](#1-message-routing)
2. [Message Transformation](#2-message-transformation)
3. [Message Channel](#3-message-channel)
4. [Message Endpoint](#4-message-endpoint)
5. [System Management](#5-system-management)
6. [Error Handling](#6-error-handling)

---

## 1. Message Routing

### Content-Based Router

Route messages to different destinations based on message content.

```java
from("direct:orders")
    .choice()
        .when(simple("${header.region} == 'US'"))
            .to("direct:us-processing")
        .when(simple("${header.region} == 'EU'"))
            .to("direct:eu-processing")
        .when(jsonpath("$.priority == 'URGENT'"))
            .to("direct:urgent-processing")
        .otherwise()
            .to("direct:default-processing")
    .end()
    .to("direct:audit-log");  // Continues after choice
```

### Message Filter

Drop messages that don't match a predicate.

```java
from("jms:queue:events")
    .filter(simple("${header.type} == 'ORDER'"))
        .to("direct:orderEvents");
    // Messages not matching the filter are silently dropped

// Filter with bean predicate
from("jms:queue:events")
    .filter().method("eventFilter", "isRelevant")
        .to("direct:process");
```

### Recipient List

Route to a dynamically computed list of destinations.

```java
// Header contains comma-separated list of destinations
from("direct:distribute")
    .recipientList(header("destinations"))
    .delimiter(",")
    .parallelProcessing()
    .stopOnException();

// Computed via bean
from("direct:distribute")
    .recipientList().method("routingService", "getRecipients");
```

### Splitter

Split a composite message into individual parts.

```java
// Split JSON array
from("direct:batchOrders")
    .split(jsonpath("$.orders"))
    .streaming()                     // Process one at a time (low memory)
    .parallelProcessing()            // Process parts concurrently
    .to("direct:processSingleOrder");

// Split by line
from("file:/data/inbox")
    .split(body().tokenize("\n"))
    .to("direct:processLine");

// Split with aggregation of results
from("direct:process")
    .split(body(), new MyAggregationStrategy())
    .to("direct:processItem")
    .end()
    .to("direct:handleAggregatedResult");
```

### Aggregator

Combine multiple related messages into a single message.

```java
from("jms:queue:order-items")
    .aggregate(header("orderId"), new OrderAggregationStrategy())
    .completionSize(10)              // Complete after 10 messages
    .completionTimeout(30000)        // Or after 30 seconds
    .completionPredicate(simple("${body.size} >= ${header.expectedCount}"))
    .eagerCheckCompletion()
    .to("direct:processCompleteOrder");
```

```java
public class OrderAggregationStrategy implements AggregationStrategy {
    @Override
    public Exchange aggregate(Exchange oldExchange, Exchange newExchange) {
        if (oldExchange == null) {
            List<OrderItem> items = new ArrayList<>();
            items.add(newExchange.getIn().getBody(OrderItem.class));
            newExchange.getIn().setBody(items);
            return newExchange;
        }
        List<OrderItem> items = oldExchange.getIn().getBody(List.class);
        items.add(newExchange.getIn().getBody(OrderItem.class));
        return oldExchange;
    }
}
```

**Aggregation completion conditions:**
- `completionSize(N)` — after N messages in the group
- `completionTimeout(ms)` — after ms milliseconds of inactivity
- `completionInterval(ms)` — periodic completion every ms
- `completionPredicate(pred)` — when predicate matches
- `completionFromBatchConsumer()` — when batch consumer signals end of batch
- Multiple conditions can be combined (OR logic)

### Dynamic Router

Route step-by-step, where each step determines the next destination.

```java
from("direct:start")
    .dynamicRouter(method("dynamicRouterBean", "route"));

@Component("dynamicRouterBean")
public class DynamicRouterBean {
    public String route(@Header("step") Integer step, @Body String body) {
        if (step == null) return "direct:step1";
        if (step == 1) return "direct:step2";
        if (step == 2) return "direct:step3";
        return null;  // null = stop routing
    }
}
```

### Routing Slip

Route through a pre-determined sequence of endpoints stored in a header.

```java
// Header "routingSlip" contains: "direct:validate,direct:enrich,direct:save"
from("direct:start")
    .routingSlip(header("routingSlip")).delimiter(",");
```

### Multicast

Send the same message to multiple destinations (optionally in parallel).

```java
from("direct:distribute")
    .multicast()
    .parallelProcessing()
    .stopOnException()
    .to("direct:audit", "direct:analytics", "direct:notification");

// With aggregation of responses
from("direct:scatter-gather")
    .multicast(new BestPriceAggregator())
    .parallelProcessing()
    .to("direct:vendorA", "direct:vendorB", "direct:vendorC")
    .end()
    .to("direct:handleBestPrice");
```

### Wire Tap

Send a copy of the message to a secondary destination without affecting the main flow.

```java
from("direct:orders")
    .wireTap("jms:queue:audit")
    .to("direct:processOrder");   // Main flow continues unaffected

// Wire tap with modified copy
from("direct:orders")
    .wireTap("jms:queue:audit")
        .newExchangeBody(simple("Audit: ${body}"))
    .end()
    .to("direct:processOrder");
```

### Load Balancer

Distribute messages across multiple endpoints.

```java
// Round-robin
from("direct:distribute")
    .loadBalance().roundRobin()
    .to("http:server1/api", "http:server2/api", "http:server3/api");

// Failover (try next on failure)
from("direct:distribute")
    .loadBalance().failover(IOException.class)
    .to("http:primary/api", "http:backup/api");

// Random
from("direct:distribute")
    .loadBalance().random()
    .to("http:server1/api", "http:server2/api");

// Weighted round-robin
from("direct:distribute")
    .loadBalance().weighted(true, "3,2,1")
    .to("http:server1/api", "http:server2/api", "http:server3/api");
```

### Throttler

Limit message throughput.

```java
// Max 100 messages per second
from("jms:queue:events")
    .throttle(100).timePeriodMillis(1000)
    .to("direct:process");

// Dynamic throttle rate from header
from("jms:queue:events")
    .throttle(header("maxRate"))
    .to("direct:process");
```

### Saga (Long-Running Transactions)

Distributed saga pattern with compensating actions.

```java
from("direct:placeOrder")
    .saga()
        .propagation(SagaPropagation.REQUIRES_NEW)
        .compensation("direct:cancelOrder")
        .completion("direct:confirmOrder")
        .to("direct:reserveInventory")
        .to("direct:processPayment")
        .to("direct:scheduleShipping");
```

---

## 2. Message Transformation

### Content Enricher

Add data from an external source to the message.

```java
// Enrich from a producer endpoint (e.g., HTTP call)
from("direct:orders")
    .enrich("http:customer-service/api/customers/${header.customerId}",
        new CustomerEnrichmentStrategy());

// Poll enrich (from a consumer endpoint, e.g., file)
from("direct:orders")
    .pollEnrich("file:/data/config?fileName=settings.json", 5000)  // 5s timeout
    .to("direct:process");
```

```java
public class CustomerEnrichmentStrategy implements AggregationStrategy {
    @Override
    public Exchange aggregate(Exchange original, Exchange enrichment) {
        Order order = original.getIn().getBody(Order.class);
        Customer customer = enrichment.getIn().getBody(Customer.class);
        order.setCustomerName(customer.getName());
        return original;
    }
}
```

### Claim Check

Store payload temporarily; retrieve later (useful when calling services that don't need the full payload).

```java
from("direct:process")
    .claimCheck(ClaimCheckOperation.Push)    // Store current body
    .transform(simple("${header.orderId}"))  // Replace body with just the ID
    .to("direct:lookupDetails")              // External call
    .claimCheck(ClaimCheckOperation.Pop);    // Restore original body
```

### Message Translator

Transform message content using processor, bean, or expression.

```java
// Simple expression transform
from("direct:input")
    .transform(simple("Hello ${body}"))
    .to("mock:result");

// Bean transform
from("direct:input")
    .transform().method("orderMapper", "toDto")
    .to("direct:output");

// Processor transform
from("direct:input")
    .process(exchange -> {
        Order order = exchange.getIn().getBody(Order.class);
        exchange.getIn().setBody(new OrderSummary(order.getId(), order.getTotal()));
    })
    .to("direct:output");
```

---

## 3. Message Channel

### Point-to-Point Channel

One consumer receives each message. Implemented by JMS queues, SEDA, Direct.

```java
from("jms:queue:orders").to("direct:process");  // Only one consumer gets each message
```

### Publish-Subscribe Channel

All subscribers receive every message. Implemented by JMS topics, Multicast.

```java
from("jms:topic:events").to("direct:handle");  // All subscribers get every message
```

### Dead Letter Channel

Messages that fail processing after all retries go to an error destination.

```java
errorHandler(deadLetterChannel("jms:queue:dead-letters")
    .maximumRedeliveries(5)
    .redeliveryDelay(2000)
    .backOffMultiplier(2)
    .useOriginalMessage()
    .logExhausted(true)
    .logExhaustedMessageHistory(true));
```

### Guaranteed Delivery

Ensure messages are not lost via transacted JMS sessions.

```java
from("jms:queue:payments?transacted=true")
    .transacted("PROPAGATION_REQUIRED")
    .to("bean:paymentProcessor")
    .to("jms:queue:confirmations");
```

---

## 4. Message Endpoint

### Polling Consumer

Periodically polls a resource for new messages.

```java
from("file:/data/inbox?delay=5000")        // Poll every 5 seconds
    .to("direct:process");

from("sql:SELECT * FROM orders WHERE status='NEW'?consumer.delay=10000")
    .to("direct:process");
```

### Event-Driven Consumer

Reacts to messages as they arrive (push-based).

```java
from("jms:queue:events")      // JMS listener — event-driven
    .to("direct:handle");

from("kafka:events")          // Kafka consumer — event-driven
    .to("direct:handle");
```

### Idempotent Consumer

Prevent duplicate message processing.

```java
from("jms:queue:orders")
    .idempotentConsumer(header("JMSMessageID"),
        MemoryIdempotentRepository.memoryIdempotentRepository(1000))
    .to("direct:process");
```

### Service Activator

Invoke a service in response to a message (the most common pattern).

```java
from("jms:queue:requests")
    .bean("orderService", "process")   // Service activator
    .to("jms:queue:responses");
```

---

## 5. System Management

### Control Bus

Manage routes at runtime.

```java
// Start a route
template.sendBody("controlbus:route?routeId=myRoute&action=start", null);

// Stop a route
template.sendBody("controlbus:route?routeId=myRoute&action=stop", null);

// Get route status
String status = template.requestBody(
    "controlbus:route?routeId=myRoute&action=status", null, String.class);
```

### Detour

Conditionally route through additional processing.

```java
from("direct:orders")
    .choice()
        .when(simple("${properties:detour.enabled} == 'true'"))
            .to("direct:auditStep")
    .end()
    .to("direct:normalProcessing");
```

### Log

Diagnostic logging at any point in a route.

```java
from("jms:queue:orders")
    .log(LoggingLevel.INFO, "Received: ${body}")
    .to("direct:process")
    .log("Processed order ${header.orderId} in ${header.CamelTimerPeriod}ms");
```

---

## 6. Error Handling

### onException

Handle specific exception types with targeted recovery.

```java
onException(ValidationException.class)
    .handled(true)
    .maximumRedeliveries(0)
    .setHeader(Exchange.HTTP_RESPONSE_CODE, constant(400))
    .setBody(simple("Validation error: ${exception.message}"));

onException(IOException.class)
    .maximumRedeliveries(5)
    .redeliveryDelay(3000)
    .backOffMultiplier(2)
    .retryAttemptedLogLevel(LoggingLevel.WARN);

onException(Exception.class)
    .handled(true)
    .to("jms:queue:errors")
    .log(LoggingLevel.ERROR, "Unhandled: ${exception.message}");
```

### doTry / doCatch / doFinally

Java try-catch-finally semantics in a route.

```java
from("direct:process")
    .doTry()
        .to("http:external-api/process")
    .doCatch(HttpOperationFailedException.class)
        .log(LoggingLevel.WARN, "HTTP call failed: ${exception.message}")
        .to("direct:fallback")
    .doFinally()
        .to("direct:audit")
    .end();
```

### Error Handler Types

| Type | Use Case |
|---|---|
| `defaultErrorHandler()` | Default — propagates exception to caller |
| `deadLetterChannel("uri")` | Routes failed messages to error queue |
| `noErrorHandler()` | Disable error handling (for transacted routes) |
| `transactionErrorHandler()` | JTA transaction-aware error handling |

### Redelivery Policy Options

| Option | Description |
|---|---|
| `maximumRedeliveries(N)` | Max retry attempts |
| `redeliveryDelay(ms)` | Delay between retries |
| `backOffMultiplier(N)` | Multiply delay after each retry |
| `maximumRedeliveryDelay(ms)` | Cap on delay growth |
| `collisionAvoidanceFactor(0.15)` | Randomise delay to avoid thundering herd |
| `retryAttemptedLogLevel(WARN)` | Log level for retry attempts |
| `logExhausted(true)` | Log when retries exhausted |
| `useOriginalMessage()` | Send original (not modified) message to DLQ |
| `asyncDelayedRedelivery()` | Non-blocking delay between retries |
