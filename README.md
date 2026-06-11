# Bakery Delivery Platform

A small but thoughtfully designed REST API for a fictional bakery client. Built to demonstrate clean architecture, Test-Driven Development, and SOLID principles — the same engineering practices used in professional software delivery.

Demo link- https://drive.google.com/file/d/1E35v40SwdueayrbtQ7BW7bYJyIoZ04lN/view?usp=sharing

---

## The Client Brief

> "We run a local bakery and need a system so customers can order online and track their deliveries. We're worried the old system is a mess and we want something we can change easily as the business grows."

This is a common consulting scenario. The technical choices below are driven by the client's actual needs, not technology preference.

---

## User Stories

These acceptance criteria drove the tests before a single line of implementation was written.

| Story | Acceptance Criteria |
|---|---|
| Place an order | Customer submits items + address → receives confirmed order with total |
| Out of stock | Unavailable product → clear error, no partial order |
| Track delivery | Customer queries order ID → gets current status |
| Cancel order | Customer cancels before dispatch → notified of cancellation |
| Wrong customer cancels | Different customer ID → forbidden, order unchanged |
| Skip status step | Confirm → dispatch without preparing → rejected |

---

## Architecture

This project follows **Clean Architecture** (Robert C. Martin). The core rule: **dependencies point inward only**.

```
Infrastructure  →  Application  →  Domain
(Spring, DB)       (Use Cases)     (Pure Java)
```

### Why this matters for the client

The bakery might switch from email notifications to WhatsApp. Or from H2 to PostgreSQL. Or add a mobile app. With this architecture, the business logic (domain + application layers) never changes when the delivery mechanism changes. Only infrastructure adapters are swapped.

### Package structure

```
com.bakery/
├── domain/                    ← Core business logic. Zero framework imports.
│   ├── model/                 ←   Order, Product, OrderItem, OrderStatus
│   ├── repository/            ←   Repository interfaces (NOT implementations)
│   └── service/               ←   Notification interface
├── application/               ← Orchestration. Depends only on domain.
│   └── usecase/               ←   PlaceOrder, TrackDelivery, CancelOrder
└── infrastructure/            ← Framework-specific. Depends on application + domain.
    ├── web/                   ←   REST controllers, DTOs, exception handler
    ├── persistence/           ←   In-memory repository implementations
    └── notification/          ←   Console notification (swap for email/SMS)
```

---

## SOLID Principles — Where They Appear

| Principle | Where applied |
|---|---|
| **S** — Single Responsibility | Each use case class has exactly one reason to change |
| **O** — Open/Closed | New notification channels = new class, no changes to domain |
| **L** — Liskov Substitution | `InMemoryOrderRepository` is fully substitutable for any future JPA implementation |
| **I** — Interface Segregation | `OrderRepository` and `ProductRepository` are separate interfaces |
| **D** — Dependency Inversion | `PlaceOrderUseCase` depends on `OrderRepository` interface, not a Spring bean |

---

## Test Strategy

Tests are written **test-first** (Red → Green → Refactor). Each test describes a behaviour, not a method.

```
src/test/
├── domain/
│   └── OrderTest.java          ← 14 unit tests, no mocks needed (pure domain)
└── application/
    ├── PlaceOrderUseCaseTest.java   ← 7 tests, mocked repos and notifier
    └── CancelOrderUseCaseTest.java  ← 3 tests
```

Run tests:
```bash
mvn test
```

---

## Running Locally

```bash
# Clone and run
git clone https://github.com/your-username/bakery-delivery
cd bakery-delivery
mvn spring-boot:run
```

The app starts on `http://localhost:8080`. Products are auto-seeded on startup.

### Example API calls

```bash
# List products
curl http://localhost:8080/api/products

# Place an order (use a product ID from the list above)
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "customer-1",
    "deliveryAddress": "42 Bread Lane, London",
    "items": [{ "productId": "<product-id-here>", "quantity": 2 }]
  }'

# Track an order
curl http://localhost:8080/api/orders/<order-id>

# Cancel an order
curl -X DELETE http://localhost:8080/api/orders/<order-id> \
  -H "X-Customer-Id: customer-1"
```

---

## Design Decisions and Trade-offs

**Why no JPA annotations on domain models?**
JPA annotations would make the domain dependent on Hibernate. If we ever switch ORMs or go event-sourced, the domain would need to change. Instead, persistence adapters translate between domain objects and JPA entities.

**Why are use cases not Spring `@Service` beans?**
Marking them `@Service` would couple the application layer to Spring's DI container. Wiring them manually in `BakeryApplication.java` keeps use cases fully testable without starting a Spring context.

**Why `record` for commands and DTOs?**
Java records are immutable by default. A command object that can be mutated after construction is a bug waiting to happen.

**What would change in a production system?**
- Replace `InMemoryOrderRepository` with a JPA implementation
- Replace `ConsoleNotificationService` with an email/SMS adapter
- Add an `OrderEntity` JPA class to avoid polluting the domain model
- Add authentication (Spring Security) to validate `X-Customer-Id`

---

## CI/CD

Every push to `main` runs the full test suite via GitHub Actions. See `.github/workflows/ci.yml`.
