# RabbitMQ Basics

## Part 1: Introduction to RabbitMQ

### Overview

RabbitMQ is a message broker that facilitates communication between distributed systems. It implements the **AMQP (Advanced Message Queuing Protocol)** and is widely used for **event-driven applications, microservices communication, and job processing**. 

In this document, we will explore RabbitMQ’s **architecture, best practices, and a comparison with Kafka and Azure Service Bus**.

### Why RabbitMQ?

RabbitMQ is designed to provide:

- **Reliable message delivery** with acknowledgments and retries.
- **Flexible routing** via exchanges and queues.
- **Scalability** using clustering and high-availability setups.
- **Support for multiple messaging patterns**, including pub-sub, request-response, and event-driven processing.

## Architecture

We run a **3-node RabbitMQ cluster** behind a **load balancer** to ensure **high availability and fault tolerance**.

Here’s how RabbitMQ processes messages:

1. **Producers** send messages to an **exchange**.
2. **Exchanges** route messages to the appropriate **queues** based on binding rules.
3. **Queues** store messages until they are processed by **consumers**.
4. **Consumers** receive messages and process them based on application logic.

### Exchange Types

RabbitMQ supports different **exchange types** that determine how messages are routed:

#### **Direct Exchange**
- Routes messages to queues based on an **exact match** with the routing key.
- Example use case: Task distribution where each message reaches **only one queue**.

#### **Fanout Exchange**
- Broadcasts messages to **all bound queues**, ignoring routing keys.
- Example use case: **Event broadcasting** (e.g., system-wide notifications).

#### **Topic Exchange**
- Routes messages based on **pattern-matching routing keys** (`*` matches one word, `#` matches multiple words).
- Example use case: **Logging systems** routing logs by severity and source.

#### **Headers Exchange**
- Routes messages based on **message headers** rather than routing keys.
- Example use case: **Routing based on metadata** (e.g., region, priority).

### Durable Quorum Queues and Federated Deployment

- **Durable, quorum queues** are the preferred means of queueing within RabbitMQ, ensuring message persistence and reliability.
- We use a **Federated deployment** for:
  - **Cross-region messaging**
  - **Disaster recovery** in case of node failures.

## Comparing RabbitMQ, Kafka, and Azure Service Bus

| Feature            | RabbitMQ                         | Kafka                            | Azure Service Bus            |
|-------------------|--------------------------------|--------------------------------|------------------------------|
| **Use Case**      | Event-driven apps, RPC, job queues | High-throughput event streaming | Cloud-based messaging       |
| **Message Retention** | Transient unless persisted | Log-based retention | Built-in durability         |
| **Scalability**   | Horizontally scalable, clustering required | Massively scalable partitions | Managed scaling |
| **Ordering** | Per-queue message ordering | Partition-based ordering | FIFO & partitioned queues |
| **Delivery Guarantee** | At-most-once, at-least-once | At-least-once | At-least-once, exactly-once |

## Streaming in RabbitMQ

RabbitMQ introduced **streaming** capabilities to handle high-throughput, log-based message retention, similar to Kafka. It provides:
- **Efficient handling of large message volumes**
- **Persistent message storage** without requiring consumers to acknowledge messages immediately
- **Consumer offsets**, allowing messages to be replayed from a specified point

Unlike traditional queues, **streams** are designed for scenarios where **multiple consumers** need access to the same messages **in order**, making them ideal for event sourcing and real-time data processing.

## Quorum Queues vs. Mirrored Classic Queues

RabbitMQ provides two high-availability queue types: **Quorum Queues** and **Mirrored Classic Queues**.

### **Quorum Queues**
- Based on [**Raft consensus algorithm**](raft.github.io).
- Messages are replicated **to multiple nodes**, ensuring high availability.
- No risk of "split-brain" issues seen in mirrored queues.
- **Best for: Long-lived, durable, and high-reliability applications**.

### **Mirrored Classic Queues**
- Messages are replicated to **designated mirror nodes**.
- If the primary node fails, a mirror node takes over.
- Can suffer from **split-brain scenarios**, causing inconsistencies.
- **Deprecated in favor of Quorum Queues** in modern RabbitMQ setups.

### **Tabular Comparison**
| Feature           | Quorum Queues                   | Mirrored Classic Queues        |
|------------------|--------------------------------|--------------------------------|
| **Replication**   | Raft-based consensus         | Designated mirror nodes      |
| **Split-Brain Risk** | No                           | Yes                            |
| **Failover Mechanism** | Automatic leader election  | Mirror node promotion         |
| **Performance**   | Slightly higher overhead      | Observed 300% improvement |
| **Deprecation**   | Recommended                  | Deprecated                     |

For **high-reliability deployments**, **Quorum Queues** are the preferred choice due to their superior consistency and fault tolerance. Mirrored Classic Queues are DEPRECATED. This means they are no longer supported at all.

## Deployment Environments

We maintain **three environments** to ensure stability and proper testing:

- **Development (Dev)** – For local testing and feature development.
- **Quality Assurance (QA)** – For integration and performance testing.
- **Production (Prod)** – Live environment for serving business applications.

## Onboarding Process

New users or services onboarded to RabbitMQ must:

1. **Be added to FIM/MIM groups** for access control.
2. **Have a dedicated Virtual Host (VHost)** created to isolate workloads.

## Cluster State Monitoring & Disaster Recovery

To maintain high availability, we have a **daily job** that:

- Tracks the state of our RabbitMQ cluster.
- Ensures consistency across nodes.
- Can provision a **new cluster** automatically if necessary.

## Best Practices for Running RabbitMQ

- **Use durable, quorum queues** to persist messages in case of crashes.
- **Optimize prefetch settings** to improve consumer performance.
- **Monitor cluster health** using **Prometheus and Grafana**.
- **Leverage dead-letter queues (DLQs)** to handle failed messages gracefully.

### Optimizing Prefetch Settings

**Prefetch settings** determine how many messages a consumer can fetch from a queue before acknowledging previous messages. Optimizing prefetch settings can significantly improve RabbitMQ performance.

#### Steps to Optimize:

1. **Understand Consumer Workload**
   - If consumers process messages quickly, a higher prefetch value can reduce latency.
   - If message processing time varies, a lower prefetch value prevents message pile-up.

2. **Set the Prefetch Value Properly**
   - Use `basic.qos(prefetch_count=N)` where `N` is the optimal number of unacknowledged messages a consumer can hold.
   - A value of `1` ensures strict round-robin dispatching but may lower throughput.
   - A higher value (`10-100`) improves efficiency when consumers are fast.

3. **Test and Adjust**
   - Monitor message queue depth and consumer throughput.
   - Adjust prefetch dynamically based on workload analysis.

4. **Enable Fair Dispatch**
   - Setting `basic.qos(prefetch_count=1)` ensures consumers only receive a new message when they finish the previous one, preventing slow consumers from blocking others.

5. **Use Consumer Auto-Scaling**
   - If message backlogs increase, dynamically scale consumers rather than just increasing prefetch count.

## Next in the Series

**Part 2: Common Problems and Their Solutions**

This includes onboarding, increased CPU/memory usage, and Queue Length longer than 10k messages.
