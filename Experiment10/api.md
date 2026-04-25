# API Design — E-Commerce Inventory & Order System

---

## 1. API Design Principles

- **RESTful**: Resources are nouns, HTTP verbs define actions
- **Versioned**: All endpoints prefixed with `/api/v1/`
- **Stateless**: Auth via JWT in `Authorization: Bearer <token>` header
- **Consistent Response Envelope**: All responses follow a standard JSON wrapper
- **Idempotency**: Mutation endpoints support idempotency keys to prevent duplicate side effects

### Standard Response Envelope

```json
{
  "success": true,
  "data": { ... },
  "error": null,
  "meta": {
    "requestId": "req_abc123",
    "timestamp": "2025-04-24T10:00:00Z"
  }
}
```

### Standard Error Response

```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "PRODUCT_NOT_FOUND",
    "message": "No product found with the given ID",
    "details": {}
  },
  "meta": {
    "requestId": "req_abc123",
    "timestamp": "2025-04-24T10:00:00Z"
  }
}
```

---

## 2. Authentication & Authorization

All protected routes require the header:
```
Authorization: Bearer <JWT>
```

JWT Payload:
```json
{
  "userId": "uuid",
  "email": "user@example.com",
  "role": "customer",
  "iat": 1714000000,
  "exp": 1714086400
}
```

---

## 3. REST API Endpoints

---

### 3.1 User Service

#### POST `/api/v1/users/register`
Register a new user.

**Request:**
```json
{
  "name": "Rahul Sharma",
  "email": "rahul@example.com",
  "password": "Secret@123",
  "phoneNumber": "9876543210"
}
```

**Response — 201 Created:**
```json
{
  "success": true,
  "data": {
    "userId": "550e8400-e29b-41d4-a716-446655440000",
    "email": "rahul@example.com",
    "name": "Rahul Sharma"
  }
}
```

**Errors:**
| Code | HTTP | Meaning |
|------|------|---------|
| `EMAIL_ALREADY_EXISTS` | 409 | Duplicate email |
| `VALIDATION_ERROR` | 400 | Invalid fields |

---

#### POST `/api/v1/users/login`
Authenticate and receive JWT.

**Request:**
```json
{
  "email": "rahul@example.com",
  "password": "Secret@123"
}
```

**Response — 200 OK:**
```json
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expiresIn": 86400
  }
}
```

**Errors:**
| Code | HTTP | Meaning |
|------|------|---------|
| `INVALID_CREDENTIALS` | 401 | Wrong email/password |

---

#### GET `/api/v1/users/me`
🔒 Auth required. Returns current user profile.

**Response — 200 OK:**
```json
{
  "success": true,
  "data": {
    "userId": "550e8400-...",
    "name": "Rahul Sharma",
    "email": "rahul@example.com",
    "phoneNumber": "9876543210",
    "addresses": []
  }
}
```

---

### 3.2 Product & Search Service

#### GET `/api/v1/products`
List products with pagination and filters.

**Query Params:**
| Param | Type | Description |
|-------|------|-------------|
| `page` | int | Page number (default: 1) |
| `limit` | int | Items per page (default: 20, max: 50) |
| `categoryId` | UUID | Filter by category |
| `minPrice` | decimal | Minimum price |
| `maxPrice` | decimal | Maximum price |

**Response — 200 OK:**
```json
{
  "success": true,
  "data": {
    "products": [
      {
        "productId": "...",
        "name": "iPhone 16",
        "price": 79999.00,
        "imageUrl": "https://cdn.example.com/iphone16.jpg",
        "inStock": true
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 340,
      "totalPages": 17
    }
  }
}
```

---

#### GET `/api/v1/products/{productId}`
Get full product details.

**Path Param:** `productId` — UUID

**Response — 200 OK:**
```json
{
  "success": true,
  "data": {
    "productId": "...",
    "name": "iPhone 16",
    "description": "Apple iPhone 16, 256GB...",
    "price": 79999.00,
    "category": "Electronics",
    "images": ["https://cdn.example.com/img1.jpg"],
    "availableQty": 42
  }
}
```

**Errors:**
| Code | HTTP | Meaning |
|------|------|---------|
| `PRODUCT_NOT_FOUND` | 404 | Invalid productId |

---

#### GET `/api/v1/search?q={query}`
Full-text search via Elasticsearch.

**Query Params:**
| Param | Type | Description |
|-------|------|-------------|
| `q` | string | Search query (required) |
| `page` | int | Page number |
| `limit` | int | Page size |

**Response — 200 OK:**
```json
{
  "success": true,
  "data": {
    "results": [
      { "productId": "...", "name": "iPhone 16", "price": 79999.00, "score": 0.98 }
    ],
    "total": 15
  }
}
```

---

### 3.3 Cart Service

#### GET `/api/v1/cart`
🔒 Auth required. Returns current user's cart.

**Response — 200 OK:**
```json
{
  "success": true,
  "data": {
    "cartId": "...",
    "items": [
      {
        "productId": "...",
        "name": "iPhone 16",
        "quantity": 1,
        "currentPrice": 79999.00
      }
    ],
    "totalAmount": 79999.00
  }
}
```

**Note:** `currentPrice` is fetched live from Product DB, not the price at time of add.

---

#### POST `/api/v1/cart/items`
🔒 Auth required. Add or update item in cart.

**Request:**
```json
{
  "productId": "550e8400-...",
  "quantity": 2
}
```

**Response — 200 OK:**
```json
{
  "success": true,
  "data": { "message": "Cart updated" }
}
```

---

#### DELETE `/api/v1/cart/items/{productId}`
🔒 Auth required. Remove a product from cart.

**Response — 200 OK:**
```json
{
  "success": true,
  "data": { "message": "Item removed" }
}
```

---

### 3.4 Checkout & Order Service

#### POST `/api/v1/checkout`
🔒 Auth required. Initiates checkout — reserves inventory and creates an order.

**Idempotency-Key header required** (see Section 5).

**Request:**
```json
{
  "addressId": "...",
  "paymentMethod": "RAZORPAY"
}
```

**Response — 201 Created:**
```json
{
  "success": true,
  "data": {
    "orderId": "...",
    "totalAmount": 79999.00,
    "paymentUrl": "https://razorpay.com/pay/order_xyz",
    "status": "PENDING"
  }
}
```

**Errors:**
| Code | HTTP | Meaning |
|------|------|---------|
| `OUT_OF_STOCK` | 409 | Inventory insufficient |
| `CART_EMPTY` | 400 | No items to checkout |
| `DUPLICATE_REQUEST` | 409 | Idempotency key already processed |

---

#### GET `/api/v1/orders`
🔒 Auth required. List all orders for current user.

**Query Params:** `page`, `limit`, `status`

**Response — 200 OK:**
```json
{
  "success": true,
  "data": {
    "orders": [
      {
        "orderId": "...",
        "status": "DELIVERED",
        "totalAmount": 79999.00,
        "createdAt": "2025-04-01T10:00:00Z",
        "itemCount": 1
      }
    ]
  }
}
```

---

#### GET `/api/v1/orders/{orderId}`
🔒 Auth required. Get detailed order info.

**Response — 200 OK:**
```json
{
  "success": true,
  "data": {
    "orderId": "...",
    "status": "SHIPPED",
    "items": [
      { "productId": "...", "name": "iPhone 16", "quantity": 1, "priceAtPurchase": 79999.00 }
    ],
    "totalAmount": 79999.00,
    "payment": {
      "paymentId": "...",
      "gateway": "RAZORPAY",
      "status": "SUCCESS"
    },
    "shippingAddress": { ... },
    "createdAt": "2025-04-01T10:00:00Z"
  }
}
```

---

#### DELETE `/api/v1/orders/{orderId}`
🔒 Auth required. Cancel an order (only if status is PENDING or CONFIRMED).

**Response — 200 OK:**
```json
{
  "success": true,
  "data": { "message": "Order cancelled", "status": "CANCELLED" }
}
```

**Errors:**
| Code | HTTP | Meaning |
|------|------|---------|
| `ORDER_NOT_CANCELLABLE` | 400 | Order already shipped/delivered |

---

### 3.5 Payment Service

#### POST `/api/v1/payments/webhook`
Webhook endpoint called by payment gateway (Razorpay/Stripe) after payment attempt.

> This endpoint is public but verified via HMAC signature.

**Request:**
```json
{
  "orderId": "...",
  "txnRef": "pay_xyz",
  "status": "SUCCESS",
  "amount": 79999.00,
  "signature": "hmac_sha256_digest"
}
```

**Response — 200 OK:**
```json
{ "received": true }
```

---

## 4. HTTP Status Codes

| Code | Usage |
|------|-------|
| `200` | Successful GET, PUT, DELETE |
| `201` | Resource created (POST) |
| `400` | Bad request / validation error |
| `401` | Missing or invalid JWT |
| `403` | Forbidden (accessing another user's resource) |
| `404` | Resource not found |
| `409` | Conflict (duplicate, out of stock) |
| `429` | Rate limit exceeded |
| `500` | Internal server error |
| `503` | Service temporarily unavailable |

---

## 5. Versioning Strategy

**URI-based versioning** is used: `/api/v1/`, `/api/v2/`, etc.

**Rationale:**
- Explicit and easy to route at the API Gateway level
- Clients always know what contract they're on
- Old versions can be deprecated gracefully without breaking existing consumers

**Deprecation Policy:**
- Old versions remain active for 6 months post new version release
- `Sunset` response header is sent on deprecated endpoints: `Sunset: Sat, 1 Nov 2025 00:00:00 GMT`

---

## 6. Idempotency Handling

Mutation endpoints that can cause financial or inventory side effects require an `Idempotency-Key` header:

```
POST /api/v1/checkout
Idempotency-Key: client-generated-uuid-v4
```

**How it works:**
1. On first request, the key is stored in Redis with a TTL of 24 hours along with the response.
2. If the same key arrives again (due to network retry), the stored response is returned directly — no duplicate order or charge is created.
3. Keys expire after 24 hours.

**Applicable endpoints:**
- `POST /api/v1/checkout`
- `POST /api/v1/payments/initiate` (if exposed)

---

## 7. Rate Limiting Strategy

Rate limiting is enforced at the **API Gateway** level using a **Token Bucket** algorithm (via Redis).

| Endpoint Type | Limit |
|---------------|-------|
| Public endpoints (search, product listing) | 100 req/min per IP |
| Authenticated endpoints (cart, orders) | 60 req/min per userId |
| Checkout endpoint | 10 req/min per userId |
| Payment webhook | No rate limit (IP allowlist instead) |

**Response when limit exceeded:**
```
HTTP 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1714000060
```

**Spillover handling:** If Redis is unavailable, the gateway falls back to an in-memory counter with a conservative limit to prevent complete outage.
