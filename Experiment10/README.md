# E-Commerce Inventory & Order System
### System Design — Lab Mini Project

---

## Project Overview

This project presents a complete system design for a large-scale E-Commerce platform inspired by Amazon and Flipkart. The system handles the full customer journey — from user registration and product discovery to cart management, checkout, inventory reservation, and payment processing.

The architecture follows a **microservices model** with event-driven communication via Apache Kafka, a CDC-based search sync pipeline, and layered caching for performance. The design prioritizes availability and partition tolerance (AP system) with eventual consistency where acceptable, and strong consistency only where money and inventory are involved.

---

## System Architecture (Summary)

The system is composed of the following core microservices, all exposed through a central API Gateway:

| Service | Responsibility |
|---------|---------------|
| **User Service** | Registration, login, JWT issuance |
| **Search Service** | Full-text product search via Elasticsearch |
| **Product Service** | Product CRUD, image management (S3) |
| **Cart Service** | Per-user cart with live pricing |
| **Checkout Service** | Inventory reservation + order creation |
| **Order Status Service** | Order lifecycle management |
| **Inventory Service** | Stock tracking with atomic updates |
| **Payment Service** | Gateway integration + webhook handling |

Communication:
- **Synchronous**: REST over HTTP (client-facing)
- **Asynchronous**: Apache Kafka (inter-service events: order placed, payment confirmed, stock updated)
- **Search Sync**: Debezium CDC → Kafka → Elasticsearch (near real-time)

---

## Assumptions

1. The system serves a single country/region initially (India); multi-region is a future concern.
2. Each product has a single seller (marketplace with one inventory pool per product).
3. Users are pre-authenticated via JWT; OAuth2 / social login is out of scope for this design.
4. Payment is handled by a third-party gateway (Razorpay); PCI-DSS compliance is delegated to the gateway.
5. All services run in Kubernetes (EKS or GKE) and are independently deployable.
6. Flash sale scenarios are expected; the inventory layer is designed to handle spike traffic.
7. Email/SMS notifications for order updates are sent asynchronously and are not in the critical path.
8. Product images are uploaded by sellers via a separate Seller Portal (out of scope here).
9. Fraud detection is handled by the payment gateway, not this system.
10. Prices are stored and served in INR; multi-currency is a future extension.

---

## Tech Stack

| Layer | Technology | Justification |
|-------|-----------|---------------|
| **API Gateway** | AWS API Gateway / Kong | Rate limiting, auth, routing, load balancing — offloads cross-cutting concerns from services |
| **Services** | Node.js (Express) or Java (Spring Boot) | Node.js for I/O-heavy services (Search, Cart); Java for transaction-heavy services (Checkout, Inventory) |
| **User DB** | MySQL | Structured relational data with strict constraints; ACID transactions for user accounts |
| **Product DB** | MySQL | Relational schema with category hierarchy; well-supported CDC tooling with Debezium |
| **Order DB** | MySQL | ACID transactions critical; familiar tooling; schema is structured and relationally consistent |
| **Inventory DB** | PostgreSQL | `SELECT FOR UPDATE` row locking is more predictable; native UUID and JSONB support |
| **Cart DB** | PostgreSQL | Clean row-level locking for concurrent cart updates |
| **Payment DB** | MySQL | Simple append-only writes; consistent with team tooling |
| **Search** | Elasticsearch | Industry-standard full-text search; supports fuzzy matching, faceting, relevance scoring |
| **Cache** | Redis Cluster | Sub-millisecond reads; atomic operations (INCR/DECR) for inventory counters; supports rate limiting |
| **Message Broker** | Apache Kafka | High-throughput, durable, ordered event streaming; supports consumer groups for independent downstream services |
| **CDC Pipeline** | Debezium | Zero-impact change capture directly from MySQL binlog; no application-level code changes needed |
| **Object Storage** | AWS S3 | Scalable, cheap storage for product images |
| **CDN** | AWS CloudFront | Low-latency global image delivery; removes load from origin |
| **Containerization** | Docker + Kubernetes | Consistent deployments; HPA for auto-scaling based on metrics |
| **Monitoring** | Prometheus + Grafana | Service metrics, latency dashboards, alerting |
| **Logging** | ELK Stack | Centralized log aggregation across all services |

---

## Trade-offs Taken

### 1. Eventual Consistency in Search vs. Immediate Consistency
**Decision:** Product changes are synced to Elasticsearch asynchronously via CDC (delay of ~1-5 seconds).
**Trade-off:** A newly updated price may not appear in search results instantly. Acceptable because the authoritative price is always fetched from the Product DB at checkout — the search result is a guide, not a contract.

### 2. Optimistic Inventory Display vs. Strict Real-Time Stock
**Decision:** Product listing pages show approximate stock (Redis counter), not a live DB query.
**Trade-off:** A product may appear "in stock" when it's actually just sold out in the last few seconds. This is resolved at checkout with a real-time DB check. The UX risk is a late-stage "out of stock" message, which is acceptable and common on real platforms.

### 3. Microservices vs. Monolith
**Decision:** Microservices architecture chosen.
**Trade-off:** Higher operational complexity (service discovery, distributed tracing, Kafka setup). Justified because each service has a different scaling requirement — Inventory and Checkout need aggressive scaling during sales while User Service does not. A monolith would force over-provisioning.

### 4. Kafka vs. Direct REST Calls Between Services
**Decision:** Order events flow through Kafka instead of direct service calls.
**Trade-off:** Adds Kafka as an infrastructure dependency and introduces processing latency (~100-200ms). Justified because it completely decouples services — if Inventory Service is temporarily down during checkout, the event is queued and processed when it recovers. No data loss, no cascading failure.

### 5. PostgreSQL for Inventory vs. MySQL (used elsewhere)
**Decision:** PostgreSQL chosen for Inventory for its superior row-level locking semantics with `SELECT FOR UPDATE SKIP LOCKED`.
**Trade-off:** Operational heterogeneity (two DB technologies). Justified because inventory accuracy is the most critical write path — PostgreSQL's locking behavior is more predictable under high concurrency than MySQL's InnoDB for this specific access pattern.

---

## Future Improvements

1. **Recommendation Engine** — Collaborative filtering or ML-based recommendations ("Customers who bought X also bought Y") using order history data piped into a data warehouse.

2. **Multi-Region Deployment** — Active-active multi-region setup with geo-routing for latency reduction and disaster recovery. Requires solving cross-region DB consistency.

3. **Seller Portal** — A separate microservice for sellers to manage their products, pricing, and inventory with role-based access control.

4. **Real-Time Order Tracking** — WebSocket-based order tracking pushed to the client as order status changes via Kafka events.

5. **Flash Sale Infrastructure** — A dedicated sale scheduler service that pre-warms Redis inventory counters, disables CDC sync temporarily to reduce DB load, and enables queue-based virtual waiting rooms.

6. **GraphQL API Layer** — Replace REST for the frontend-facing API to allow clients to fetch exactly the data they need, reducing over-fetching on mobile devices.

7. **Fraud Detection Service** — ML-based order anomaly detection (unusual bulk orders, address mismatches) before payment initiation.

8. **Returns & Refunds Flow** — Order reversal, inventory restocking, and payment refund orchestration as a separate service.

9. **Data Warehouse** — Pipe order, click, and search events into a data warehouse (Redshift / BigQuery) for business intelligence and reporting.

10. **Service Mesh** — Introduce Istio or Linkerd for mTLS between services, distributed tracing (Jaeger), and traffic management without code changes.

---

## Repository Structure

```
/ecommerce-system-design
├── HLD.md        ← High-Level Design, architecture diagram, data flow
├── LLD.md        ← Class diagrams, DB schema, sequence diagrams, design patterns
├── api.md        ← REST endpoints, request/response formats, status codes, rate limiting
├── scaling.md    ← Load balancing, caching, DB scaling, failure handling, bottlenecks
└── README.md     ← This file — overview, assumptions, tech stack, trade-offs
```

---

## Author

System Design Lab Mini Project — E-Commerce Inventory & Order System
Submitted: 24th April 2025
