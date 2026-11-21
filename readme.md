# Spring Boot Microservices Online Shop

A backend application that demonstrates an online shop built with **Java 21**, **Spring Boot 3**, and a microservices architecture. It supports a product catalog, inventory checks, order placement, and asynchronous email notifications.

The project contains **four business services and an API gateway**, organized into five Maven modules. The development stack runs using Docker Compose without Kubernetes.

## Architecture

```text
                    Swagger UI / API client
                             |
                             v
                    API gateway :9000
                  Keycloak JWT validation
                    /        |         \
                   v         v          v
               Product   Inventory    Order
                  |          ^           |
                  v          |           v
               MongoDB       └── REST   MySQL
                                          |
                                          v
                                  Kafka + Schema Registry
                                          |
                                          v
                                 Notification service
                                          |
                                          v
                                       Mailpit
```

The gateway is the main entry point. The order service calls inventory synchronously over REST and publishes order events to Kafka for asynchronous notification processing.

## Technology stack

| Area | Technologies |
|---|---|
| Language and framework | Java 21, Spring Boot 3.2.4 |
| Build | Maven multi-module project |
| Gateway | Spring Cloud Gateway MVC, Spring Cloud 2023.0.1 |
| REST APIs | Spring Web, Spring REST client |
| API documentation | Springdoc OpenAPI, Swagger UI |
| Authentication | Keycloak, OAuth2 resource server, JWT bearer tokens |
| Product persistence | MongoDB, Spring Data MongoDB |
| Order and inventory persistence | MySQL, Spring Data JPA, Hibernate |
| Database migrations | Flyway |
| Messaging | Apache Kafka, ZooKeeper |
| Event serialization | Apache Avro, Confluent Schema Registry |
| Email | Spring Mail, Mailpit |
| Resilience | Resilience4j circuit breakers and retries, HTTP timeouts |
| Operations | Spring Boot Actuator, Docker health checks |
| Testing | JUnit, Spring Boot Test, Testcontainers, WireMock, REST Assured |
| Deployment | Docker, multi-stage Dockerfile, Docker Compose |

## Services

### Product service

Manages products stored in MongoDB.

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/product` | Create a product |
| `GET` | `/api/product` | List products |

A product contains an ID, name, description, SKU code, and price. Spring Data MongoDB provides the persistence layer.

### Inventory service

Checks stock availability using MySQL.

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/inventory?skuCode=...&quantity=...` | Check whether the requested quantity is available |

Flyway creates the inventory table and adds four sample SKUs: `iphone_15`, `pixel_8`, `galaxy_24`, and `oneplus_12`. Each starts with a quantity of `100`.

> **Current scope:** The service checks inventory but does not reserve or deduct stock.

### Order service

Places orders using MySQL and communicates with inventory and Kafka.

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/order` | Check stock and place an order |

An order request includes a SKU, unit price, quantity, and customer details. The service:

1. Calls the inventory service over REST.
2. Generates an order number when stock is reported available.
3. Calculates the total price and saves the order in MySQL.
4. Publishes an `OrderPlacedEvent` to the `order-placed` Kafka topic.

The event contains the order number and customer email, first name, and last name.

### Notification service

Consumes `order-placed` events from Kafka and sends order-confirmation emails using Spring Mail.

In the local Compose stack, emails are captured by **Mailpit** and can be inspected at [http://localhost:8025](http://localhost:8025).

### API gateway

The gateway is the entry point for product, inventory, and order API requests. It:

- Routes requests to the appropriate service.
- Validates Keycloak JWT bearer tokens.
- Aggregates three OpenAPI definitions in one Swagger UI.
- Applies circuit-breaker filters and provides a fallback route.

Swagger is available at [http://localhost:9000/swagger-ui.html](http://localhost:9000/swagger-ui.html).

## Order flow

```text
Client
  → API gateway
  → Order service
  → Inventory service
  → MySQL
  → Kafka
  → Notification service
  → Mailpit
```

The gateway authenticates the API request. The order service checks inventory, saves the order if stock is reported available, and publishes an event. The notification service consumes that event and prepares a confirmation email.

## Authentication and Swagger

Keycloak issues JWT access tokens. The gateway validates those tokens before allowing API requests. Swagger documentation is public, while business API operations require authentication.

To test an API:

1. Obtain a Keycloak access token for the configured realm.
2. Open the gateway Swagger UI and select a service definition.
3. Click **Authorize** and paste the token without the `Bearer` prefix.
4. Use **Try it out** and **Execute** on an operation.

Tokens expire; obtain and authorize a fresh token when needed. The current setup uses manual token entry rather than a Swagger-to-Keycloak browser login.

## Data and messaging

| Data | Technology | Purpose |
|---|---|---|
| Products | MongoDB | Stores product documents |
| Inventory | MySQL | Stores SKU quantities |
| Orders | MySQL | Stores orders and calculated totals |
| Order events | Kafka | Transfers order events to notification service |
| Event schemas | Avro and Schema Registry | Defines and serializes event data |
| Test emails | Mailpit | Captures local confirmation emails |

Flyway manages the MySQL schemas and inventory seed data. Named Docker volumes retain database data across ordinary container shutdowns.

## Resilience and operations

- **Circuit breakers:** Configured for gateway routes and inventory calls.
- **Retries:** Configured for inventory calls made by the order service.
- **Timeouts:** Configured for HTTP communication and gateway resilience.
- **Fallbacks:** The gateway has a service-unavailable fallback; the inventory client can return `false` when a stock check fails.
- **Health:** Spring Boot Actuator exposes health information, and Compose defines health checks for several infrastructure services.

## Docker Compose stack

The development stack contains **13 containers**:

| Group | Containers |
|---|---|
| Application — 5 | API gateway, product, inventory, order, notification |
| Data — 3 | MongoDB, application MySQL, Keycloak MySQL |
| Identity — 1 | Keycloak |
| Messaging — 3 | Kafka broker, ZooKeeper, Schema Registry |
| Email testing — 1 | Mailpit |

A shared multi-stage Dockerfile builds the application modules. Services communicate over the Compose network.

### Run locally

**Requirement:** Docker with the Compose plugin.

From the project directory:

```powershell
Copy-Item .env.example .env
docker compose up --build -d
```

View container status and logs:

```powershell
docker compose ps -a
docker compose logs -f
```

| Interface | Address |
|---|---|
| Swagger UI | [http://localhost:9000/swagger-ui.html](http://localhost:9000/swagger-ui.html) |
| Keycloak | [http://localhost:8181](http://localhost:8181) |
| Mailpit | [http://localhost:8025](http://localhost:8025) |

Stop the stack with:

```powershell
docker compose down
```

Named database volumes remain after shutdown. The sample credentials, local URLs, and Keycloak development mode are intended for local development only.

## Testing

The project includes Spring Boot tests and dependencies for Testcontainers, WireMock, and REST Assured. It also contains focused tests for Swagger definitions and gateway security.

Run the Maven test and verification lifecycle with:

```powershell
mvn verify
```

The test suite requires further attention: for example, the order success test omits customer details that the current order service accesses. The Dockerfile builds application images with tests skipped.

## Current limitations

- Inventory is **checked but not reserved or deducted**.
- Saving an order and publishing its Kafka event are **not atomic**.
- Input validation and error handling are limited; an inventory service failure can be treated as unavailable stock.
- The Compose stack uses **development-oriented configuration** and is not production-ready as supplied.
- The project does not include a frontend, payment processing, or Kubernetes deployment.
- Throughput, latency, scalability, and uptime improvements have **not been benchmarked**.
