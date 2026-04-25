# Scalability & Reliability — E-Commerce Inventory & Order System

---

## 1. Load Balancing Strategy

### API Gateway as Load Balancer
The API Gateway (e.g., AWS API Gateway + ALB, or Nginx) acts as the single entry point and distributes incoming requests across service instances using a **Round Robin** strategy by default, with health-check-aware routing to avoid sending traffic to unhealthy pods.

### Per-Service Load Balancing
Each microservice runs as multiple replicas behind an internal load balancer (Kubernetes Service + kube-proxy or AWS ALB Target Group).

**Strategy by service type:**

| Service | Strategy | Reason |
|---------|----------|--------|
| User Service | Round Robin | Stateless, all instances equal |
| Search Service | Round Robin | Stateless reads |
| Product Service | Round Robin | Stateless reads |
| Cart Service | Sticky Sessions (optional) | Cart state in DB, not memory — Round Robin works too |
| Checkout Service | Least Connections | Heavy operations, avoid overloading one pod |
| Payment Service | Round Robin | Stateless gateway proxy |
| Inventory Service | Least Connections | Critical writes, spread load carefully |

### Health Checks
- **Liveness probe**: HTTP GET `/health/live` — restarts the pod if it fails
- **Readiness probe**: HTTP GET `/health/ready` — removes pod from load balancer rotation until warm

---

## 2. Horizontal vs Vertical Scaling

### Approach: Horizontal Scaling (Primary Strategy)

All microservices are designed to be **stateless** so they can be horizontally scaled by adding more instances. No service holds state in memory that another instance would be unaware of — all state lives in the database, Redis, or Kafka.

**Horizontal Scaling per service:**

| Service | Trigger to Scale Out | Metric |
|---------|----------------------|--------|
| User Service | CPU > 60% | Kubernetes HPA |
| Search Service | Request queue depth | Elasticsearch read replicas too |
| Checkout Service | p99 latency > 500ms | HPA on CPU + custom metric |
| Inventory Service | Concurrent write rate | Manual scaling + pessimistic locking |
| Kafka Consumers | Consumer lag > threshold | KEDA (Kubernetes Event-Driven Autoscaling) |

**Vertical Scaling (Secondary / DB Layer):**
Databases (MySQL, PostgreSQL) are vertically scaled first before sharding because horizontal DB scaling is complex. Large DB instances handle high-write workloads. Vertical scaling has a ceiling though, so read replicas and sharding are planned for beyond a threshold (see Section 4).

---

## 3. Caching Strategy

### 3.1 Redis — Application-Level Cache

Redis is used as the primary cache layer, deployed as a Redis Cluster for HA.

| Data Cached | TTL | Strategy | Reason |
|-------------|-----|----------|--------|
| Product details | 10 min | Cache-aside | Read-heavy, rarely changes |
| Category tree | 30 min | Cache-aside | Very stable |
| User profile | 5 min | Cache-aside | Moderate read frequency |
| Inventory count (approx.) | 30 sec | Write-through | Fast stock checks during listing |
| Search results (top queries) | 2 min | Cache-aside | ES is expensive for popular queries |
| JWT blacklist (logout) | Token TTL | Set membership | Fast auth check |
| Idempotency keys | 24 hr | Set + GET | Prevent duplicate checkout |
| Rate limit counters | 1 min window | Sliding window | Token bucket in Redis |

**Cache-aside pattern:**
```
1. Request arrives
2. Check Redis → HIT → return cached data
3. MISS → query DB → store in Redis → return
```

**Cache invalidation:**
Product and inventory cache is invalidated on write via CDC events consumed from Kafka. When a product price changes in MySQL, the CDC pipeline publishes an event; a cache-invalidation consumer deletes the Redis key.

### 3.2 CDN — Static Asset Caching

Product images stored in **AWS S3** are served via **CloudFront CDN**.
- Images are cached at edge nodes globally (TTL: 7 days)
- URLs include a content hash; when images change, URLs change, avoiding stale cache
- This removes image-serving load entirely from the application servers

---

## 4. Database Scaling

### 4.1 Read Replicas (Primary Strategy)

All databases use a **Primary + Read Replica** setup:
- Writes go to the primary instance only
- Reads (product listing, order history, search fallback) go to read replicas
- MySQL replication lag is monitored; queries with consistency requirements (post-checkout) read from primary

**Replica count by load:**

| DB | Replicas |
|----|----------|
| User DB (MySQL) | 1 read replica |
| Product DB (MySQL) | 2 read replicas |
| Order DB (MySQL) | 1 read replica |
| Inventory DB (PostgreSQL) | 1 read replica (reads only for reporting) |

### 4.2 Sharding (Future Scale)

When a single MySQL primary cannot handle write throughput:

**Order DB — Range-based sharding on `created_at` (time-based partitioning):**
- Shard 1: Orders from 2023
- Shard 2: Orders from 2024
- Shard 3: Orders from 2025+
- Old shards become read-only archives
- Trade-off: Range shards can cause hot spots during current period; mitigate with monthly partitions

**Product DB — Hash-based sharding on `product_id`:**
- Hash(product_id) % N → assigns to shard
- Uniform distribution of products across shards
- Trade-off: Cross-shard queries (e.g., category browse) require scatter-gather; solved by Elasticsearch for search

### 4.3 Elasticsearch Scaling

Elasticsearch is scaled horizontally by adding data nodes.
- Each index is configured with **2 primary shards and 1 replica** per shard
- Product index replicas serve read traffic; primary shards handle indexing
- CDC pipeline (Debezium) feeds near-real-time updates from MySQL → Kafka → ES

### 4.4 Connection Pooling

All services use a connection pool (e.g., HikariCP for Java, pg-pool for Node.js) to avoid the overhead of creating DB connections per request.
- Pool size tuned to: `(core_count * 2) + effective_disk_spindles`
- Connection pool exhaustion is monitored and triggers horizontal scaling of the service

---

## 5. Failure Handling

### 5.1 Retries with Exponential Backoff

All service-to-service HTTP calls and Kafka consumers implement retry with exponential backoff:

```
Attempt 1: immediate
Attempt 2: 1s delay
Attempt 3: 2s delay
Attempt 4: 4s delay (max retries = 4)
→ If still failing, route to Dead Letter Queue (DLQ)
```

This prevents retry storms during transient failures.

### 5.2 Circuit Breaker Pattern

Implemented using **Resilience4j** (Java) or **opossum** (Node.js) at service call boundaries.

**Example — Checkout Service calling Inventory Service:**

| State | Behaviour |
|-------|-----------|
| CLOSED | Normal — all calls pass through |
| OPEN | Triggered when >50% calls fail in 10s window — fast-fail with fallback response |
| HALF-OPEN | After 30s, allow limited calls; if they succeed, circuit closes again |

**Fallback responses:**
- Inventory check failure → return "item may be unavailable, try again" (don't block checkout silently)
- Payment gateway timeout → return pending status, webhook confirms later

### 5.3 Kafka Dead Letter Queue (DLQ)

Messages that fail processing after max retries are sent to a DLQ topic (e.g., `order.events.dlq`).
- An ops team monitors and replays DLQ messages after root cause resolution
- Prevents data loss for critical events (order placed, payment confirmed)

### 5.4 Idempotent Consumers

All Kafka consumers are idempotent — processing the same message twice produces the same result.
- Inventory deduction checks if the `orderId` has already been processed before deducting stock
- Prevents double-deduction on consumer restart/rebalance

### 5.5 Database Transactions for Inventory

The checkout flow uses a **PostgreSQL transaction with pessimistic locking** for inventory:

```sql
BEGIN;
SELECT quantity, reserved_qty FROM inventory
  WHERE product_id = $1 FOR UPDATE;  -- row-level lock
-- Check available = quantity - reserved_qty >= requested_qty
UPDATE inventory SET reserved_qty = reserved_qty + $qty
  WHERE product_id = $1;
COMMIT;
```

This prevents two simultaneous checkouts from over-reserving the same product (race condition / overselling).

### 5.6 Graceful Degradation

| Failure Scenario | Degraded Response |
|-----------------|-------------------|
| Elasticsearch down | Fall back to MySQL LIKE query for search (slower but available) |
| Redis down | Bypass cache, read directly from DB (higher latency, system stays up) |
| Payment gateway timeout | Mark order as PENDING; webhook resolves it asynchronously |
| CDC pipeline lagging | Product listing shows slightly stale data; eventual consistency acceptable |

---

## 6. Bottlenecks & Optimizations

### Bottleneck 1 — Inventory Check During Checkout (Most Critical)

**Problem:** Every checkout makes a real-time DB write to reserve inventory. At high concurrency (flash sales), this creates lock contention on the inventory table.

**Solution:**
1. Use **Redis** to maintain a real-time stock counter (pre-loaded from DB at sale start).
2. `DECRBY inventory:{productId} {qty}` in Redis is atomic — no locking needed.
3. If Redis count goes negative, deny the request (oversell prevention without DB lock).
4. Asynchronously sync Redis count back to DB via a background job every few seconds.

Trade-off: During a Redis failure, we fall back to DB with locking (slower but safe).

---

### Bottleneck 2 — Search at Scale

**Problem:** MySQL LIKE queries on millions of products are O(N) and cannot scale for full-text search.

**Solution:** Elasticsearch handles all search queries. MySQL remains the source of truth. CDC pipeline keeps ES in sync within seconds of a product update.

Optimization: Popular search terms are cached in Redis for 2 minutes to skip ES entirely for the long tail of repetitive queries.

---

### Bottleneck 3 — Order History Queries

**Problem:** `SELECT * FROM orders WHERE user_id = X` becomes slow when a user has thousands of orders and the table has millions of rows.

**Solution:**
1. Composite index on `(user_id, created_at DESC)` for pagination.
2. Return only summary fields (orderId, status, totalAmount, createdAt) in the list — full detail fetched on demand.
3. Older orders (> 1 year) archived to a cold storage table or S3-backed data warehouse.

---

### Bottleneck 4 — Kafka Consumer Lag During Order Spikes

**Problem:** During a flash sale, the order placement rate can spike 100x. Kafka consumers (Inventory, Order DB writer) may lag.

**Solution:**
1. Scale Kafka consumers horizontally via KEDA (Kubernetes Event-Driven Autoscaler) — consumer count scales with topic lag.
2. Increase Kafka partition count for `order.events` topic to allow more parallel consumers.
3. Pre-warm consumers before a known sale event.

---

### General Optimizations

| Optimization | Impact |
|-------------|--------|
| DB connection pooling | Reduces connection setup latency per request |
| CDN for product images | Removes ~60% of bandwidth from origin servers |
| Pagination on all list APIs | Prevents full table scans for listing endpoints |
| Async order confirmation email | Removes email latency from critical checkout path |
| Lazy loading of cart prices | Cart add is fast; prices recalculated only at checkout |
