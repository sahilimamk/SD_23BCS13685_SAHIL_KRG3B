# Low-Level Design (LLD) — E-Commerce Inventory & Order System

---

## 1. Class Diagrams (OOP + SOLID Principles)

### 1.1 User Domain

```
┌─────────────────────────────┐
│           User              │
├─────────────────────────────┤
│ - userId: UUID              │
│ - name: String              │
│ - email: String             │
│ - passwordHash: String      │
│ - phoneNumber: String       │
│ - address: Address          │
│ - createdAt: Timestamp      │
├─────────────────────────────┤
│ + register(): void          │
│ + login(): JWT              │
│ + updateProfile(): void     │
└─────────────────────────────┘

┌─────────────────────────────┐
│           Address           │
├─────────────────────────────┤
│ - street: String            │
│ - city: String              │
│ - state: String             │
│ - pincode: String           │
│ - country: String           │
└─────────────────────────────┘
```

### 1.2 Product Domain

```
┌──────────────────────────────┐
│          Product             │
├──────────────────────────────┤
│ - productId: UUID            │
│ - name: String               │
│ - description: String        │
│ - price: Decimal             │
│ - categoryId: UUID           │
│ - sellerId: UUID             │
│ - imageUrls: List<String>    │
│ - tags: List<String>         │
├──────────────────────────────┤
│ + getDetails(): ProductDTO   │
│ + updatePrice(): void        │
└──────────────────────────────┘

┌──────────────────────────────┐
│         Inventory            │
├──────────────────────────────┤
│ - inventoryId: UUID          │
│ - productId: UUID            │
│ - quantity: Integer          │
│ - reservedQty: Integer       │
│ - warehouseId: UUID          │
├──────────────────────────────┤
│ + reserve(qty): Boolean      │
│ + release(qty): void         │
│ + deduct(qty): void          │
│ + getAvailable(): Integer    │
└──────────────────────────────┘
```

### 1.3 Order Domain

```
┌──────────────────────────────┐
│            Order             │
├──────────────────────────────┤
│ - orderId: UUID              │
│ - userId: UUID               │
│ - items: List<OrderItem>     │
│ - totalPrice: Decimal        │
│ - status: OrderStatus        │
│ - paymentId: UUID            │
│ - createdAt: Timestamp       │
├──────────────────────────────┤
│ + place(): void              │
│ + cancel(): void             │
│ + getStatus(): OrderStatus   │
└──────────────────────────────┘

        <<enum>>
┌──────────────────────────────┐
│         OrderStatus          │
├──────────────────────────────┤
│ PENDING                      │
│ CONFIRMED                    │
│ PROCESSING                   │
│ SHIPPED                      │
│ DELIVERED                    │
│ CANCELLED                    │
│ FAILED                       │
└──────────────────────────────┘

┌──────────────────────────────┐
│          OrderItem           │
├──────────────────────────────┤
│ - orderItemId: UUID          │
│ - productId: UUID            │
│ - quantity: Integer          │
│ - priceAtPurchase: Decimal   │
└──────────────────────────────┘
```

### 1.4 Cart Domain

```
┌──────────────────────────────┐
│             Cart             │
├──────────────────────────────┤
│ - cartId: UUID               │
│ - userId: UUID               │
│ - items: List<CartItem>      │
│ - updatedAt: Timestamp       │
├──────────────────────────────┤
│ + addItem(item): void        │
│ + removeItem(productId): void│
│ + clear(): void              │
│ + getTotal(): Decimal        │
└──────────────────────────────┘

┌──────────────────────────────┐
│           CartItem           │
├──────────────────────────────┤
│ - productId: UUID            │
│ - quantity: Integer          │
│ - addedAt: Timestamp         │
└──────────────────────────────┘
```

### 1.5 Payment Domain

```
        <<interface>>
┌──────────────────────────────┐
│       PaymentProcessor       │
├──────────────────────────────┤
│ + charge(req): PaymentResult │
│ + refund(txnId): Boolean     │
└──────────────────────────────┘
           ▲         ▲
           │         │
┌──────────┴──┐  ┌───┴──────────┐
│  Razorpay   │  │   Stripe     │
│  Processor  │  │  Processor   │
└─────────────┘  └──────────────┘

┌──────────────────────────────┐
│          Payment             │
├──────────────────────────────┤
│ - paymentId: UUID            │
│ - orderId: UUID              │
│ - userId: UUID               │
│ - amount: Decimal            │
│ - status: PaymentStatus      │
│ - gateway: String            │
│ - txnRef: String             │
│ - createdAt: Timestamp       │
└──────────────────────────────┘
```

---

## 2. Database Schema

### 2.1 User Service — MySQL

```sql
CREATE TABLE users (
  user_id       CHAR(36)     PRIMARY KEY,
  name          VARCHAR(100) NOT NULL,
  email         VARCHAR(150) NOT NULL UNIQUE,
  password_hash VARCHAR(255) NOT NULL,
  phone_number  VARCHAR(15),
  created_at    TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
  updated_at    TIMESTAMP    ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE user_addresses (
  address_id CHAR(36)     PRIMARY KEY,
  user_id    CHAR(36)     NOT NULL,
  street     VARCHAR(255),
  city       VARCHAR(100),
  state      VARCHAR(100),
  pincode    VARCHAR(10),
  country    VARCHAR(50),
  is_default BOOLEAN      DEFAULT FALSE,
  FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Indexes
CREATE INDEX idx_users_email ON users(email);
```

### 2.2 Product Service — MySQL + Elasticsearch

```sql
CREATE TABLE products (
  product_id  CHAR(36)       PRIMARY KEY,
  name        VARCHAR(255)   NOT NULL,
  description TEXT,
  price       DECIMAL(10,2)  NOT NULL,
  category_id CHAR(36),
  seller_id   CHAR(36),
  is_active   BOOLEAN        DEFAULT TRUE,
  created_at  TIMESTAMP      DEFAULT CURRENT_TIMESTAMP,
  updated_at  TIMESTAMP      ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE product_images (
  image_id    CHAR(36)     PRIMARY KEY,
  product_id  CHAR(36)     NOT NULL,
  s3_url      TEXT         NOT NULL,
  is_primary  BOOLEAN      DEFAULT FALSE,
  FOREIGN KEY (product_id) REFERENCES products(product_id)
);

CREATE TABLE categories (
  category_id   CHAR(36)    PRIMARY KEY,
  name          VARCHAR(100),
  parent_id     CHAR(36),   -- for nested categories
  FOREIGN KEY (parent_id) REFERENCES categories(category_id)
);

-- Indexes
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_seller   ON products(seller_id);
```

### 2.3 Inventory Service — PostgreSQL

```sql
CREATE TABLE inventory (
  inventory_id  UUID           PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id    UUID           NOT NULL UNIQUE,
  quantity      INTEGER        NOT NULL DEFAULT 0,
  reserved_qty  INTEGER        NOT NULL DEFAULT 0,
  warehouse_id  UUID,
  updated_at    TIMESTAMP      DEFAULT now()
);

-- Computed available = quantity - reserved_qty
-- Indexed for fast product lookups
CREATE INDEX idx_inventory_product ON inventory(product_id);
```

### 2.4 Cart Service — PostgreSQL

```sql
CREATE TABLE carts (
  cart_id     UUID       PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID       NOT NULL UNIQUE,
  created_at  TIMESTAMP  DEFAULT now(),
  updated_at  TIMESTAMP  DEFAULT now()
);

CREATE TABLE cart_items (
  cart_item_id  UUID           PRIMARY KEY DEFAULT gen_random_uuid(),
  cart_id       UUID           NOT NULL,
  product_id    UUID           NOT NULL,
  quantity      INTEGER        NOT NULL,
  added_at      TIMESTAMP      DEFAULT now(),
  FOREIGN KEY (cart_id) REFERENCES carts(cart_id),
  UNIQUE(cart_id, product_id)
);

CREATE INDEX idx_cart_user ON carts(user_id);
```

### 2.5 Order Service — MySQL

```sql
CREATE TABLE orders (
  order_id    CHAR(36)       PRIMARY KEY,
  user_id     CHAR(36)       NOT NULL,
  total_price DECIMAL(10,2)  NOT NULL,
  status      ENUM('PENDING','CONFIRMED','PROCESSING',
                   'SHIPPED','DELIVERED','CANCELLED','FAILED')
              DEFAULT 'PENDING',
  payment_id  CHAR(36),
  created_at  TIMESTAMP      DEFAULT CURRENT_TIMESTAMP,
  updated_at  TIMESTAMP      ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
  order_item_id      CHAR(36)      PRIMARY KEY,
  order_id           CHAR(36)      NOT NULL,
  product_id         CHAR(36)      NOT NULL,
  quantity           INTEGER       NOT NULL,
  price_at_purchase  DECIMAL(10,2) NOT NULL,
  FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

-- Indexes
CREATE INDEX idx_orders_user_id    ON orders(user_id);
CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_order_items_prod  ON order_items(product_id);
```

### 2.6 Payment Service — MySQL

```sql
CREATE TABLE payments (
  payment_id  CHAR(36)      PRIMARY KEY,
  order_id    CHAR(36)      NOT NULL,
  user_id     CHAR(36)      NOT NULL,
  amount      DECIMAL(10,2) NOT NULL,
  status      ENUM('INITIATED','SUCCESS','FAILED','REFUNDED')
              DEFAULT 'INITIATED',
  gateway     VARCHAR(50),
  txn_ref     VARCHAR(255),
  created_at  TIMESTAMP     DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_payments_order  ON payments(order_id);
CREATE INDEX idx_payments_user   ON payments(user_id);
```

---

## 3. Relationships & Indexing Strategy

| Relationship                      | Type          | Index                     |
|-----------------------------------|---------------|---------------------------|
| User → Orders                     | One-to-Many   | `orders.user_id`          |
| Order → OrderItems                | One-to-Many   | `order_items.order_id`    |
| Product → Inventory               | One-to-One    | `inventory.product_id`    |
| User → Cart                       | One-to-One    | `carts.user_id`           |
| Cart → CartItems                  | One-to-Many   | `cart_items.cart_id`      |
| Order → Payment                   | One-to-One    | `payments.order_id`       |
| Product → Elasticsearch (CDC)     | Async Sync    | ES `product_id` field     |

**Indexing Decisions:**
- `user_id` indexed on orders for fast order history queries
- `product_id` indexed on inventory for checkout stock checks
- `email` unique-indexed on users for O(1) login lookup
- `product_id` on Elasticsearch for full-text search across name, description, tags
- `order_id` on payments for payment status checks during order updates

---

## 4. Sequence Diagrams

### 4.1 User Login & JWT Generation

```
Client          API Gateway       User Service      MySQL (User DB)
  │                  │                  │                  │
  │  POST /login     │                  │                  │
  │─────────────────>│                  │                  │
  │                  │  Validate & fwd  │                  │
  │                  │─────────────────>│                  │
  │                  │                  │  SELECT by email │
  │                  │                  │─────────────────>│
  │                  │                  │   User record    │
  │                  │                  │<─────────────────│
  │                  │                  │  bcrypt compare  │
  │                  │                  │  Generate JWT    │
  │                  │   JWT token      │                  │
  │                  │<─────────────────│                  │
  │  200 OK + JWT    │                  │                  │
  │<─────────────────│                  │                  │
```

### 4.2 Add to Cart → Checkout → Payment Flow

```
Client     API GW   Cart Svc   Inventory Svc   Checkout Svc   Payment Svc   Kafka   Order DB
  │          │          │             │               │              │          │        │
  │ ADD ITEM │          │             │               │              │          │        │
  │─────────>│─────────>│             │               │              │          │        │
  │          │          │─────────────────────────────│              │          │        │
  │ 200 OK   │          │  Check live price from Prod DB             │          │        │
  │<─────────│          │             │               │              │          │        │
  │          │          │             │               │              │          │        │
  │ CHECKOUT │          │             │               │              │          │        │
  │─────────>│──────────────────────────────────────>│              │          │        │
  │          │          │             │  reserve(qty) │              │          │        │
  │          │          │             │<──────────────│              │          │        │
  │          │          │             │  reserved OK  │              │          │        │
  │          │          │             │──────────────>│              │          │        │
  │          │          │             │               │  initiate    │          │        │
  │          │          │             │               │─────────────>│          │        │
  │          │          │             │               │ Gateway call │          │        │
  │          │          │             │               │              │──────>3rd party   │
  │          │          │             │               │  SUCCESS     │          │        │
  │          │          │             │               │<─────────────│          │        │
  │          │          │             │               │  publish event          │        │
  │          │          │             │               │─────────────────────────>        │
  │          │          │             │ deduct(qty)   │ order.created│  consume │        │
  │          │          │             │<──────────────│──────────────│──────────│───────>│
  │ 200 OK   │          │             │               │              │          │        │
  │<─────────│          │             │               │              │          │        │
```

### 4.3 Product Search via Elasticsearch

```
Client     API GW    Search Svc    Elasticsearch    Product Svc (CDC)
  │           │           │               │                 │
  │ GET /search?q=iPhone  │               │                 │
  │──────────>│──────────>│               │                 │
  │           │           │  ES query     │                 │
  │           │           │──────────────>│                 │
  │           │           │  product_ids  │                 │
  │           │           │<──────────────│                 │
  │  product list         │               │                 │
  │<──────────│           │               │                 │
  │           │           │               │                 │
  │           │ (Background CDC sync)      │                 │
  │           │           │               │  Product DB     │
  │           │           │               │<────────────────│ (Debezium connector
  │           │           │               │  upsert doc     │  watches MySQL binlog)
  │           │           │               │─────────────────│
```

---

## 5. Design Patterns Used

### 5.1 Strategy Pattern — Payment Processing
The `PaymentProcessor` interface is implemented by `RazorpayProcessor` and `StripeProcessor`. The `PaymentService` selects the right implementation at runtime based on the user's region or the gateway configured.

**Why:** New payment gateways can be added without modifying existing code (Open/Closed Principle).

### 5.2 Singleton Pattern — Kafka Producer
The Kafka producer client is initialized once at application startup and reused across all services. Creating a new producer per request is expensive; a singleton avoids connection overhead.

### 5.3 Observer Pattern (via Kafka) — Order & Inventory Sync
When an order is placed, the `CheckoutService` publishes an event to Kafka. The `InventoryConsumer` and `OrderConsumer` subscribe independently. This decouples services — Checkout does not directly call Inventory or Order DB.

### 5.4 Repository Pattern — Data Access Layer
Each service has a dedicated repository class (e.g., `OrderRepository`, `ProductRepository`) that abstracts all database queries. Business logic never writes SQL directly; it calls repository methods. This makes testing and DB swaps easier.

### 5.5 Factory Pattern — Order Status Transitions
An `OrderStatusFactory` determines valid next states from the current state. For example, `CONFIRMED → PROCESSING` is valid, but `DELIVERED → PENDING` is not. This centralizes transition logic and avoids scattered if-else checks.

---

## 6. SOLID Principles Applied

| Principle | How it's applied |
|-----------|-----------------|
| **S** — Single Responsibility | Each microservice (User, Cart, Order, Payment) has one clear domain responsibility |
| **O** — Open/Closed | PaymentProcessor interface allows new gateways without changing existing code |
| **L** — Liskov Substitution | RazorpayProcessor and StripeProcessor are interchangeable via PaymentProcessor interface |
| **I** — Interface Segregation | Search interface is separate from Product CRUD; not forced into one interface |
| **D** — Dependency Inversion | Services depend on abstractions (interfaces/repositories), not concrete DB clients |
