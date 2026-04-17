# Inventory Service

Stock management and reservation engine for the [Order Management Platform](https://github.com/soltyDude/order-management-infra). Handles atomic stock reservations with optimistic locking, Redis caching for hot-path reads, and automatic expiry of stale reservations.

## Status: In Development

**MVP planned: Phase 3 of the [project roadmap](https://github.com/soltyDude/order-management-infra#roadmap).**

Architecture, database schema, and API contract are fully designed and documented. Implementation begins after [payment-service](https://github.com/soltyDude/payment-service) Phase 2 completion.

## Responsibility

- Track stock levels per product (`totalStock = availableStock + reservedStock`)
- Reserve stock atomically with optimistic locking and retry
- Release reservations on saga compensation (payment failed, user cancel)
- Expire stale reservations via scheduled job (15 min TTL)
- Cache stock levels in Redis for sub-millisecond reads
- Publish inventory events for saga orchestration

## Patterns

| Pattern | Why |
|---------|-----|
| **Optimistic Locking** | Multiple orders may reserve the same product simultaneously. `@Version` on Product entity detects conflicts at commit time — no DB-level locks. Spring Retry handles `OptimisticLockException` (3 attempts, exponential backoff). |
| **Cache-Aside** | Stock level queries are the hot path. Redis cache with 60s TTL. Read: check cache → miss → DB → cache. Write mutations (reserve, release, update) evict the cache explicitly. |
| **Scheduled Jobs** | Reservations have a 15 min TTL. If the saga doesn't complete, `ReservationExpiryJob` automatically releases stock and publishes an event. Runs every 1 minute. |
| **Outbox** | Inventory events saved in the same transaction as stock mutations. Poller publishes to Kafka. No dual-write risk. |

## Planned API

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/inventory/{productId}` | Stock level (Redis cached, 60s TTL) |
| GET | `/api/v1/inventory` | Product catalog with stock levels (admin only) |
| POST | `/api/v1/inventory/reserve` | Reserve stock (optimistic locking + retry) |
| POST | `/api/v1/inventory/release` | Release reservation (idempotent compensation) |
| POST | `/api/v1/inventory/confirm` | Confirm reservation (prevents expiry) |
| PUT | `/api/v1/inventory/{productId}/stock` | Update total stock (admin only) |

### Example: Reserve Stock

```bash
curl -X POST http://localhost:8082/api/v1/inventory/reserve \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <jwt>" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "orderId": "d290f1ee-6c54-4b01-90e6-d701748f0851",
    "items": [
      { "productId": "7c9e6679-7425-40de-944b-e07fc1f90ae7", "quantity": 2 },
      { "productId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890", "quantity": 1 }
    ]
  }'
```

**Response (201):**
```json
{
  "reservationId": "res_xyz789abc012",
  "orderId": "d290f1ee-6c54-4b01-90e6-d701748f0851",
  "status": "ACTIVE",
  "items": [
    { "productId": "7c9e6679-...", "quantity": 2, "reserved": true },
    { "productId": "a1b2c3d4-...", "quantity": 1, "reserved": true }
  ],
  "expiresAt": "2025-01-15T10:45:00Z",
  "createdAt": "2025-01-15T10:30:03Z"
}
```

Reservation is all-or-nothing: if any item has insufficient stock, nothing is reserved (422 with `failedItems` details).

> Primary reservation flow goes through Kafka (`inventory.reserve.requested` from order-service), not REST. REST endpoints exist for admin operations and testing.

### Reservation Lifecycle

```
ACTIVE ──► CONFIRMED    saga completed, order confirmed
ACTIVE ──► RELEASED     compensation: payment failed / user cancel
ACTIVE ──► EXPIRED      ReservationExpiryJob (TTL 15 min)
```

All terminal. Reservation timing is designed so it never expires during normal saga flow:

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Saga max duration | ~30s | 3 retries × 2s backoff × 2 steps + buffers |
| Reservation TTL | 15 min | 30× buffer over saga timeout |
| Expiry job interval | 1 min | Scans `WHERE expires_at < NOW() AND status = 'ACTIVE'` |

## Planned Database Schema

Five tables in `inventory_db` (port 5434):

| Table | Purpose |
|-------|---------|
| `products` | Product catalog with `@Version` for optimistic locking. `available_quantity + reserved_quantity = total`. Partial index for low-stock queries |
| `stock_reservations` | Reservation header. ID format: `res_` + nanoid. TTL via `expires_at` column |
| `stock_reservation_items` | Per-product reserved quantities within a reservation |
| `outbox_events` | Transactional outbox for Kafka publishing |
| `processed_events` | Consumer idempotency |

```
products (1) ──────────── (N) stock_reservation_items  [FK: product_id]
stock_reservations (1) ── (N) stock_reservation_items  [FK: reservation_id, CASCADE]
```

### Stock Operations

**Reserve:** `available -= quantity`, `reserved += quantity`, version incremented. On `OptimisticLockException` → retry up to 3 times.

**Release (compensation):** `available += quantity`, `reserved -= quantity`. Idempotent — releasing an already-released or expired reservation returns current state.

**Admin stock update:** Sets new total. Constraint: `newTotal >= reservedStock` (cannot reduce below active reservations).

## Redis Cache

| Key Pattern | Value | TTL | Eviction |
|-------------|-------|-----|----------|
| `stock:{productId}` | StockDto as JSON | 60s | On any stock mutation (reserve, release, confirm, admin update) |

Cache-aside: reads tolerate 60s staleness (acceptable for product page display). Reservation decisions always read from the database with optimistic lock — never from cache.

## Planned Package Structure

```
com.example.inventoryservice/
├── api/
│   ├── controller/          InventoryController
│   ├── dto/
│   │   ├── request/         ReserveStockRequest, ReleaseStockRequest, UpdateStockRequest
│   │   └── response/        StockDto, ReservationDto, ReleaseResultDto, InventorySummaryDto
│   └── exception/           GlobalExceptionHandler, InsufficientStockException
├── domain/
│   ├── model/               Product (@Version), StockReservation, StockReservationItem, ReservationStatus
│   ├── event/               InventoryReservedEvent, InventoryFailedEvent, InventoryReleasedEvent
│   └── service/             InventoryService (queries + cache), ReservationService (reserve/release/confirm/expire)
├── infrastructure/
│   ├── kafka/
│   │   ├── producer/        OutboxPoller
│   │   └── consumer/        ReserveStockConsumer, ReleaseStockConsumer
│   ├── persistence/
│   │   ├── entity/          OutboxEvent, ProcessedEvent
│   │   ├── repository/      ProductRepository, StockReservationRepository
│   │   └── mapper/          InventoryMapper
│   ├── cache/               StockCacheService (Redis get/put/evict)
│   ├── scheduler/           ReservationExpiryJob (@Scheduled, every 1 min)
│   └── config/              KafkaConfig, RedisConfig, SecurityConfig
└── InventoryServiceApplication.java
```

## Kafka Topics

**Consumes (commands from order-service):**
| Topic | Action |
|-------|--------|
| `inventory.reserve.requested` | Reserve stock, publish result |
| `inventory.release.requested` | Release reservation (compensation) |

**Produces (results back to order-service):**
| Topic | When |
|-------|------|
| `inventory.reserved` | Stock successfully reserved |
| `inventory.failed` | Insufficient stock (with `failedItems` details) |
| `inventory.released` | Reservation released |

## Tech Stack

Java 17 · Spring Boot 3.x · PostgreSQL 16 · Redis 7 · Flyway · Maven

## Related

- [order-management-infra](https://github.com/soltyDude/order-management-infra) — project overview, Docker Compose, documentation
- [order-service](https://github.com/soltyDude/order-service) — saga orchestrator (produces commands for this service)
- [Full API Contract](https://github.com/soltyDude/order-management-infra/blob/main/docs/api-contract-full.md)
