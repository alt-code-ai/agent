# Apache Camel Component Catalogue

Complete catalogue of official Apache Camel components from the core monorepo. Grouped by category. Each entry shows: component name, artifact, URI format, consumer(C)/producer(P) support, and brief description.

For full configuration options for any component, consult the official docs at `https://camel.apache.org/components/4.x/<component>-component.html`.

---

## Core Components (27)

Built into `camel-core` or its immediate modules. No additional dependency for most.

| Component | Artifact | URI | C/P | Description |
|---|---|---|---|---|
| **Bean** | camel-bean | `bean:beanName?method=foo` | P | Invoke methods on Spring/registry beans |
| **Class** | camel-bean | `class:com.example.MyClass?method=foo` | P | Invoke methods by class name |
| **Browse** | camel-browse | `browse:name` | C | Inspect received messages (debugging) |
| **Control Bus** | camel-controlbus | `controlbus:route?action=start&routeId=foo` | P | Manage routes at runtime |
| **Data Format** | camel-dataformat | `dataformat:jackson:marshal` | P | Use data format as component |
| **Dataset** | camel-dataset | `dataset:name` | C/P | Load/soak testing |
| **Direct** | camel-direct | `direct:name` | C/P | Synchronous in-memory call (same CamelContext) |
| **Dynamic Router** | camel-dynamic-router | `dynamic-router:channel` | C | Dynamic step-by-step routing |
| **Exec** | camel-exec | `exec:command` | P | Execute OS commands |
| **File** | camel-file | `file:/path?options` | C/P | Read/write local files |
| **Kamelet** | camel-kamelet | `kamelet:name` | C/P | Reusable route templates |
| **Language** | camel-language | `language:simple:${body}` | P | Evaluate expressions |
| **Log** | camel-log | `log:category?level=INFO` | P | Log message data |
| **Mock** | camel-mock | `mock:name` | C/P | Test expectations |
| **Ref** | camel-ref | `ref:beanName` | C/P | Dynamic endpoint lookup from registry |
| **REST** | camel-rest | `rest:get:path` | C/P | REST DSL endpoints |
| **REST API** | camel-rest | `rest-api:name` | C | OpenAPI spec exposure |
| **Scheduler** | camel-scheduler | `scheduler:name?delay=5000` | C | Periodic trigger (ScheduledExecutorService) |
| **SEDA** | camel-seda | `seda:name?concurrentConsumers=5` | C/P | Async in-memory queue (same JVM) |
| **Stream** | camel-stream | `stream:in` / `stream:out` | C/P | System in/out/err |
| **Stub** | camel-stub | `stub:anyUri` | C/P | Stub out endpoints for dev/test |
| **Timer** | camel-timer | `timer:name?period=5000` | C | Periodic message generation |
| **Validator** | camel-validator | `validator:schema.xsd` | P | XML Schema validation |
| **XSLT** | camel-xslt | `xslt:transform.xsl` | P | XML transformation |
| **XSLT Saxon** | camel-xslt-saxon | `xslt-saxon:transform.xsl` | P | XSLT 2.0/3.0 via Saxon |

---

## Messaging Components

| Component | Artifact | URI | C/P | Description |
|---|---|---|---|---|
| **JMS** | camel-jms | `jms:queue:name` / `jms:topic:name` | C/P | JMS 2.0/3.0 (any JMS broker) |
| **ActiveMQ 5.x** | camel-activemq | `activemq:queue:name` | C/P | ActiveMQ Classic specific |
| **ActiveMQ 6.x** | camel-activemq6 | `activemq6:queue:name` | C/P | ActiveMQ 6+ |
| **AMQP** | camel-amqp | `amqp:queue:name` | C/P | AMQP 1.0 protocol (Qpid JMS) |
| **Kafka** | camel-kafka | `kafka:topic?brokers=host:9092` | C/P | Apache Kafka |
| **MQTT 5** | camel-paho-mqtt5 | `paho-mqtt5:topic` | C/P | MQTT 5.0 via Eclipse Paho |
| **RabbitMQ** | camel-rabbitmq | `rabbitmq:exchange?queue=name` | C/P | RabbitMQ (native AMQP 0.9.1) |
| **Spring RabbitMQ** | camel-spring-rabbitmq | `spring-rabbitmq:exchange` | C/P | RabbitMQ via Spring AMQP |
| **NATS** | camel-nats | `nats:topic` | C/P | NATS messaging |
| **Pulsar** | camel-pulsar | `pulsar:topic` | C/P | Apache Pulsar |
| **Google Pub/Sub** | camel-google-pubsub | `google-pubsub:project:topic` | C/P | GCP Pub/Sub |
| **AWS SQS** | camel-aws2-sqs | `aws2-sqs:queue` | C/P | Amazon SQS |
| **AWS SNS** | camel-aws2-sns | `aws2-sns:topic` | P | Amazon SNS |
| **Azure Service Bus** | camel-azure-servicebus | `azure-servicebus:queue` | C/P | Azure Service Bus |
| **Stomp** | camel-stomp | `stomp:destination` | C/P | STOMP protocol |

---

## HTTP / REST / Web Service Components

| Component | Artifact | URI | C/P | Description |
|---|---|---|---|---|
| **HTTP** | camel-http | `http:host/path` | P | HTTP/HTTPS client (Apache HttpClient) |
| **Netty HTTP** | camel-netty-http | `netty-http:host:port/path` | C/P | HTTP via Netty (high perf) |
| **Undertow** | camel-undertow | `undertow:http://host:port/path` | C/P | HTTP via Undertow |
| **Platform HTTP** | camel-platform-http | `platform-http:/path` | C | Reuse runtime's HTTP server |
| **Servlet** | camel-servlet | `servlet:path` | C | Servlet-based HTTP consumer |
| **CXF** | camel-cxf-soap | `cxf:address?serviceClass=X` | C/P | SOAP web services |
| **CXF-RS** | camel-cxf-rest | `cxfrs:address` | C/P | JAX-RS REST services |
| **gRPC** | camel-grpc | `grpc:host:port/service` | C/P | gRPC protocol |
| **GraphQL** | camel-graphql | `graphql:url` | P | GraphQL client |
| **WebSocket** | camel-websocket | `websocket:host:port/path` | C/P | WebSocket |

---

## Database / Data Components

| Component | Artifact | URI | C/P | Description |
|---|---|---|---|---|
| **SQL** | camel-sql | `sql:query?dataSource=#ds` | C/P | SQL queries via JDBC |
| **JDBC** | camel-jdbc | `jdbc:dataSource` | P | Raw JDBC execution |
| **JPA** | camel-jpa | `jpa:entityClass` | C/P | JPA entity operations |
| **MongoDB** | camel-mongodb | `mongodb:connection?database=X&collection=Y` | C/P | MongoDB operations |
| **Cassandra CQL** | camel-cassandraql | `cql:host/keyspace` | C/P | Cassandra queries |
| **Elasticsearch** | camel-elasticsearch | `elasticsearch:cluster` | P | Elasticsearch operations |
| **Redis** | camel-spring-redis | `spring-redis:host:port` | C/P | Redis via Spring Data |
| **InfluxDB** | camel-influxdb2 | `influxdb2:url` | P | InfluxDB 2.x |
| **HDFS** | camel-hdfs | `hdfs:hostName:port/path` | C/P | Hadoop HDFS |
| **Couchbase** | camel-couchbase | `couchbase:url` | C/P | Couchbase operations |
| **MyBatis** | camel-mybatis | `mybatis:statementName` | C/P | MyBatis SQL mapping |

---

## File / FTP / Storage Components

| Component | Artifact | URI | C/P | Description |
|---|---|---|---|---|
| **File** | camel-file | `file:/path` | C/P | Local filesystem |
| **FTP** | camel-ftp | `ftp:host/path` | C/P | FTP protocol |
| **FTPS** | camel-ftp | `ftps:host/path` | C/P | FTP over SSL/TLS |
| **SFTP** | camel-ftp | `sftp:host/path` | C/P | SFTP (SSH File Transfer) |
| **AWS S3** | camel-aws2-s3 | `aws2-s3:bucket` | C/P | Amazon S3 |
| **Azure Blob** | camel-azure-storage-blob | `azure-storage-blob:account/container` | C/P | Azure Blob Storage |
| **Google Cloud Storage** | camel-google-storage | `google-storage:bucket` | C/P | GCP Cloud Storage |
| **MinIO** | camel-minio | `minio:bucket` | C/P | MinIO object storage |

---

## Cloud Provider Components

### AWS
| Component | Artifact | Description |
|---|---|---|
| **AWS S3** | camel-aws2-s3 | Object storage |
| **AWS SQS** | camel-aws2-sqs | Message queuing |
| **AWS SNS** | camel-aws2-sns | Push notifications |
| **AWS Lambda** | camel-aws2-lambda | Serverless functions |
| **AWS DynamoDB** | camel-aws2-ddb | DynamoDB operations |
| **AWS Kinesis** | camel-aws2-kinesis | Stream processing |
| **AWS SES** | camel-aws2-ses | Email sending |
| **AWS Secrets Manager** | camel-aws-secrets-manager | Secret retrieval |
| **AWS CloudWatch** | camel-aws-cloudwatch | Metrics/logging |
| **AWS EventBridge** | camel-aws2-eventbridge | Event bus |
| **AWS Step Functions** | camel-aws2-step-functions | Workflow orchestration |

### Azure
| Component | Artifact | Description |
|---|---|---|
| **Azure Service Bus** | camel-azure-servicebus | Messaging |
| **Azure Blob Storage** | camel-azure-storage-blob | Object storage |
| **Azure Queue Storage** | camel-azure-storage-queue | Queue storage |
| **Azure Event Hubs** | camel-azure-eventhubs | Event streaming |
| **Azure CosmosDB** | camel-azure-cosmosdb | Document database |
| **Azure Key Vault** | camel-azure-key-vault | Secret management |

### Google Cloud
| Component | Artifact | Description |
|---|---|---|
| **Google Pub/Sub** | camel-google-pubsub | Messaging |
| **Google Cloud Storage** | camel-google-storage | Object storage |
| **Google BigQuery** | camel-google-bigquery | Analytics |
| **Google Cloud Functions** | camel-google-functions | Serverless |
| **Google Secret Manager** | camel-google-secret-manager | Secret management |

---

## Clustering / Coordination

| Component | Artifact | URI | Description |
|---|---|---|---|
| **Master** | camel-master | `master:name:endpoint` | Singleton route via cluster service |
| **ZooKeeper Master** | camel-zookeeper-master | `zookeeper-master:name:endpoint` | Singleton route via ZooKeeper |
| **ZooKeeper** | camel-zookeeper | `zookeeper:host:port/path` | ZK node read/write |
| **Consul** | camel-consul | `consul:category` | Consul KV, health, agent |
| **Kubernetes** | camel-kubernetes | `kubernetes-pods:url` | K8s resource management |
| **etcd** | camel-etcd3 | `etcd3:path` | etcd v3 key-value |
| **Infinispan** | camel-infinispan | `infinispan:cache` | Distributed cache |
| **Hazelcast** | camel-hazelcast | `hazelcast-map:name` | Distributed data structures |
| **Ignite** | camel-ignite | `ignite-cache:name` | Apache Ignite |

---

## Data Formats

Camel supports marshal/unmarshal for these formats:

| Format | Artifact | DSL |
|---|---|---|
| **JSON (Jackson)** | camel-jackson | `.json(JsonLibrary.Jackson)` |
| **JSON (Gson)** | camel-gson | `.json(JsonLibrary.Gson)` |
| **JSON (Jsonb)** | camel-jsonb | `.json(JsonLibrary.Jsonb)` |
| **XML (JAXB)** | camel-jaxb | `.jaxb("package")` |
| **XML (Jackson)** | camel-jacksonxml | `.jacksonXml()` |
| **CSV** | camel-csv | `.csv()` |
| **Avro** | camel-avro | `.avro()` |
| **Protobuf** | camel-protobuf | `.protobuf()` |
| **YAML** | camel-snakeyaml | `.yaml()` |
| **CBOR** | camel-cbor | `.cbor()` |
| **Bindy (fixed/CSV/KVP)** | camel-bindy | `.bindy(BindyType.Csv, MyModel.class)` |
| **Flatpack** | camel-flatpack | `.flatpack()` |
| **HL7** | camel-hl7 | `.hl7()` |
| **FHIR JSON/XML** | camel-fhir | FHIR healthcare |
| **Thrift** | camel-thrift | `.thrift()` |
| **ASN.1** | camel-asn1 | `.asn1()` |
| **Zip/GZip** | camel-zipfile | `.zipFile()` / `.gzipDeflater()` |
| **Tar** | camel-tarfile | `.tarFile()` |
| **Base64** | camel-base64 | `.base64()` |

---

## Expression Languages

| Language | Artifact | Example |
|---|---|---|
| **Simple** | (core) | `${body}`, `${header.orderId}`, `${exchangeProperty.key}` |
| **JSONPath** | camel-jsonpath | `$.store.book[0].title` |
| **XPath** | camel-xpath | `/order/@id` |
| **Groovy** | camel-groovy | Groovy scripts |
| **MVEL** | camel-mvel | MVEL expressions |
| **OGNL** | camel-ognl | OGNL expressions |
| **JQ** | camel-jq | jq JSON queries |
| **DataSonnet** | camel-datasonnet | DataSonnet transformation |
| **XSLT** | camel-xslt | XML transformation |
| **Constant** | (core) | `.constant("fixed value")` |

---

## Other Notable Components

| Category | Components |
|---|---|
| **Email** | camel-mail (SMTP/IMAP/POP3), camel-aws2-ses |
| **Scheduling** | camel-quartz, camel-timer, camel-scheduler, camel-cron |
| **Social** | camel-telegram, camel-slack, camel-twitter |
| **AI/ML** | camel-langchain4j-chat, camel-langchain4j-embeddings, camel-djl |
| **Observability** | camel-micrometer, camel-opentelemetry, camel-tracing |
| **Security** | camel-crypto, camel-xmlsecurity, camel-jasypt |
| **Scripting** | camel-groovy, camel-joor (Java), camel-python |
| **Networking** | camel-netty, camel-mina, camel-ssh |
| **SAP** | camel-sap-netweaver |
| **Salesforce** | camel-salesforce |
| **Git** | camel-git |
| **GitHub** | camel-github |
| **Docker** | camel-docker |
| **LDAP** | camel-ldap |
| **DNS** | camel-dns |
| **SNMP** | camel-snmp |
| **Velocity** | camel-velocity (templates) |
| **Freemarker** | camel-freemarker (templates) |
