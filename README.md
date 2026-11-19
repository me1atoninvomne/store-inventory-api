# Store Inventory API

A REST API for managing stock in a small shop or warehouse, built with **Java 17** and **Spring Boot 3**.

The idea comes from my own retail work: receiving deliveries, checking them against the invoice,
selling goods and keeping track of what needs to be re-ordered. This project turns that daily
process into a small backend service.

## Features

- **Products** — create, read, update and delete products (SKU, name, category, price, stock, minimum stock)
- **Sales** — selling decreases stock; selling more than is available is rejected (`409 Conflict`)
- **Low-stock report** — lists products at or below their minimum stock level
- **Deliveries** — receive goods against an invoice:
  - received quantities are added to stock
  - each line is compared with the invoice and **discrepancies** (missing / extra items) are reported
  - the same invoice cannot be received twice; unknown or duplicated SKUs reject the whole delivery
- **Validation & errors** — prices with max. 2 decimals, stock limited to 1 000 000 per product;
  every error is returned as the same JSON shape (`status`, `message`, `details`)

## Tech stack

| Area        | Technology                                  |
|-------------|---------------------------------------------|
| Language    | Java 17                                     |
| Framework   | Spring Boot 3 (Web, Data JPA, Validation)   |
| Database    | H2 in-memory (demo data in `data.sql`)      |
| Build       | Gradle (wrapper included)                   |
| Tests       | JUnit 5, Mockito, AssertJ, Spring MockMvc   |

## Run it

Requirements: JDK 17 or newer.

```bash
./gradlew bootRun          # starts the API on http://localhost:8080
./gradlew test             # runs unit + integration tests
```

On Windows use `gradlew.bat` instead of `./gradlew`.

## Endpoints

| Method | Path                         | Description                               |
|--------|------------------------------|-------------------------------------------|
| GET    | `/api/products`              | List products (optional `?category=`)     |
| GET    | `/api/products/{id}`         | Get one product                           |
| GET    | `/api/products/low-stock`    | Products that need re-ordering            |
| POST   | `/api/products`              | Create a product                          |
| PUT    | `/api/products/{id}`         | Update a product                          |
| DELETE | `/api/products/{id}`         | Delete a product                          |
| POST   | `/api/products/{id}/sell`    | Sell items (decreases stock)              |
| POST   | `/api/deliveries`            | Receive a delivery against an invoice     |
| GET    | `/api/deliveries`            | List received deliveries                  |
| GET    | `/api/deliveries/{id}`       | Get one delivery with its lines           |

### Example: receive a delivery

```bash
curl -X POST http://localhost:8080/api/deliveries \
  -H "Content-Type: application/json" \
  -d '{
        "supplier": "FreshFarm",
        "invoiceNumber": "INV-2026-001",
        "items": [
          {"sku": "MILK-1L",   "invoicedQuantity": 24, "receivedQuantity": 20},
          {"sku": "BREAD-500", "invoicedQuantity": 10, "receivedQuantity": 10}
        ]
      }'
```

Response (shortened):

```json
{
  "invoiceNumber": "INV-2026-001",
  "status": "DISCREPANCY",
  "discrepancyCount": 1,
  "items": [
    {"sku": "MILK-1L",   "invoicedQuantity": 24, "receivedQuantity": 20, "difference": -4},
    {"sku": "BREAD-500", "invoicedQuantity": 10, "receivedQuantity": 10, "difference": 0}
  ]
}
```

## Project structure

```
src/main/java/com/avlastimov/inventory
├── product/    Product entity, repository, service, controller, DTOs
├── delivery/   Delivery + DeliveryItem entities, receiving logic, controller
└── common/     Error handling (GlobalExceptionHandler, ApiError), Clock bean
```

Design decisions:

- **Layers**: controller (HTTP) → service (business rules, transactions) → repository (database).
- **DTOs** (`record` classes) are used for requests and responses instead of exposing JPA entities.
- **Business rules live in the entities** (`Product.sell()`, `Product.receive()`), so they are easy to unit-test.
- **One transaction per delivery**: if any line is invalid, nothing is saved and stock is not changed.
- **`BigDecimal` for prices** to avoid floating-point rounding errors.
- A `Clock` bean is injected so tests can use a fixed time.

## Testing

- `ProductTest` — pure unit tests of stock rules
- `ProductServiceTest`, `DeliveryServiceTest` — service logic with mocked repositories (Mockito)
- `ApiIntegrationTest` — starts the full application and calls the real endpoints with MockMvc

Manual and API test documentation (test plan, test cases, Postman collection) is in the separate
repository [inventory-api-qa](https://github.com/me1atoninvomne/inventory-api-qa).

## Possible improvements

Authentication, pagination, a persistent database (PostgreSQL) with Flyway migrations,
OpenAPI/Swagger documentation, Docker image.
