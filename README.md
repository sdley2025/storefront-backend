# Store Front Backend / Spring Boot Starter

![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.1.0-brightgreen.svg)
![Java](https://img.shields.io/badge/Java-25-orange.svg)
![Maven](https://img.shields.io/badge/Maven-wrapper-blue.svg)
![License](https://img.shields.io/badge/License-MIT-yellow.svg)

A Spring Boot storefront backend with a product catalog, user registration and JWT authentication, shopping carts, orders, and Stripe Checkout. It uses MySQL and Flyway for persistence and schema migrations.

## Features

- Product catalog with category filtering
- User registration, login, refresh tokens, and role-protected administration
- Shopping carts and order creation
- Stripe Checkout sessions and signed webhook handling
- MySQL persistence with versioned Flyway migrations
- Optional development seed data
- Public Thymeleaf landing page and OpenAPI/Swagger UI

## Tech stack

| Area | Technology |
| --- | --- |
| Runtime | Java 25, Spring Boot 4.1 |
| Build | Maven Wrapper |
| Persistence | Spring Data JPA, MySQL 8.4 |
| Schema migrations | Flyway |
| Security | Spring Security, JWT, BCrypt |
| Payments | Stripe Java SDK |
| API documentation | springdoc OpenAPI / Swagger UI |
| Web | Spring MVC, Thymeleaf |

## Project structure

```text
src/
├── main/
│   ├── java/sn/sdley/springbootstarter0/
│   │   ├── admin/       # Admin-only endpoints
│   │   ├── auth/        # JWT login, refresh, and security configuration
│   │   ├── carts/       # Cart entities, services, DTOs, and endpoints
│   │   ├── common/      # Home page, logging, and API error handling
│   │   ├── config/      # OpenAPI setup and optional data seeding
│   │   ├── orders/      # Orders, order items, and payment status
│   │   ├── payments/    # Checkout and Stripe webhook integration
│   │   ├── products/    # Products, categories, and catalog endpoints
│   │   └── users/       # Users, profiles, addresses, and user endpoints
│   └── resources/
│       ├── db/migration/ # Flyway SQL migrations
│       ├── templates/    # Public Thymeleaf landing page
│       └── application.yml
└── test/java/            # Unit, MVC-slice, and MySQL integration tests
```

## Requirements

- JDK 25
- Docker Compose (or a separately managed MySQL 8.4 database)
- Stripe account and Stripe CLI for payment testing

## Run locally

1. Copy `.env.example` to `.env`. For the included Compose database, keep `DB_USERNAME=app_user`, set `DB_PASSWORD=app_password`, and set `FLYWAY_USER=root` and `FLYWAY_PASSWORD=root`. Replace the Stripe and JWT placeholders with your own test credentials and a generated JWT secret. `.env` is git-ignored.
2. Start the MySQL database:

   ```bash
   docker compose up -d mysql
   ```

3. Set your Stripe secret API key in `STRIPE_SECRET_KEY`. Set the webhook secret after starting Stripe CLI as described in [Testing Stripe webhooks](#testing-stripe-webhooks).
4. Start the application:

   ```bash
   ./mvnw spring-boot:run
   ```

Flyway applies the migrations at startup. The public Thymeleaf landing page is at <http://localhost:8080/> (also available at `/index.html`). The Swagger UI at <http://localhost:8080/swagger-ui/index.html> and OpenAPI specification at <http://localhost:8080/v3/api-docs> are public; protected API operations still require a JWT.

To run the tests:

```bash
./mvnw test
```

The full test suite uses Testcontainers to start a disposable MySQL 8.4 database, so Docker must be running. Unit and MVC-slice tests do not need the local application database or Stripe credentials.

## Configuration

The application imports an optional `.env` file from the project root. These settings can also be supplied as environment variables.

| Variable | Purpose | Default |
| --- | --- | --- |
| `APP_NAME` | Spring application name | `spring-boot-starter-0` |
| `DB_URL` | JDBC URL for MySQL | Local `app_db` database |
| `DB_USERNAME` | MySQL username | `app_user` |
| `DB_PASSWORD` | MySQL password | `app_password` |
| `JPA_DATABASE_PLATFORM` | Hibernate database dialect | MySQL dialect |
| `JWT_SECRET` | Secret used to sign JWTs | Development placeholder; replace it |
| `JWT_ACCESS_TOKEN_EXPIRATION` | Access-token lifetime in seconds | `900` |
| `JWT_REFRESH_TOKEN_EXPIRATION` | Refresh-token lifetime in seconds | `604800` |
| `APP_SEED_ENABLED` | Enable sample data on an empty database | `false` |
| `FLYWAY_USER` | Database user used by startup Flyway migrations | `root` in dev; required in prod |
| `FLYWAY_PASSWORD` | Database password used by startup Flyway migrations | `root` in dev; required in prod |
| `STRIPE_SECRET_KEY` | Stripe API secret key | Placeholder; replace it |
| `STRIPE_WEBHOOK_SECRET_KEY` | Stripe webhook signing secret | Placeholder; replace it |
| `WEBSITE_URL` | Frontend base URL used for Checkout return URLs | `http://localhost:4200` |

Use test-mode Stripe credentials for development. Never commit `.env`, production credentials, or real secrets. Generate a unique strong `JWT_SECRET` and use a secret manager for deployed environments.

Startup migrations use `spring.flyway.user` and `spring.flyway.password`, configured through `FLYWAY_USER` and `FLYWAY_PASSWORD`. The `root` defaults are only for the included local development database; supply both variables in production using a dedicated migration account with only the privileges its migrations need. The Maven `flyway:*` commands are configured separately in `pom.xml` and currently use `DB_URL`, `DB_USERNAME`, and `DB_PASSWORD`.

When `APP_SEED_ENABLED=true`, sample categories, products, users, profiles, addresses, and wishlists are created only if the related tables have no existing data. Seeded accounts are development-only and must not be used in production.

## API overview

All API endpoints require authentication unless noted otherwise. Include the JWT access token in the `Authorization` header for protected endpoints. The landing page, API documentation, carts, and Stripe webhooks are accessible without JWT.

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/`, `/index.html` | Public Thymeleaf landing page |
| POST | `/auth/login` | Authenticate and issue an access token and refresh-token cookie |
| POST | `/auth/refresh` | Refresh the access token using the refresh-token cookie |
| GET | `/auth/me` | Get the authenticated user's profile |
| GET, POST | `/users` | List users or register a user (`POST` is public) |
| GET, PUT, DELETE | `/users/{id}` | Read, update, or delete a user |
| POST | `/users/{id}/change-password` | Change a user's password |
| GET, POST | `/products` | List products or create a product |
| GET, PUT, DELETE | `/products/{id}` | Read, update, or delete a product |
| GET | `/products?categoryId={id}` | Filter products by category |
| POST | `/carts` | Create a cart (public) |
| GET | `/carts/{cartId}` | Retrieve a cart (public) |
| POST | `/carts/{cartId}/items` | Add a product to a cart (public) |
| PUT | `/carts/{cartId}/items/{productId}` | Update a cart item's quantity (public) |
| DELETE | `/carts/{cartId}/items/{productId}` | Remove an item (public) |
| DELETE | `/carts/{cartId}/items` | Clear a cart (public) |
| POST | `/checkout` | Create an order and Stripe Checkout session |
| POST | `/checkout/webhook` | Receive Stripe webhook events (public; signature-verified) |
| GET | `/orders` | List orders |
| GET | `/orders/{orderId}` | Retrieve an order |
| GET | `/admin/hello` | Example admin-only endpoint |

Product listing accepts an optional `categoryId` query parameter. The `admin` endpoints require the `ADMIN` role. Cart endpoints are currently public and use UUID cart IDs; do not treat an unguessable cart ID as a replacement for authorization in a production system.

## Checkout flow

Create a cart, add products, then call `POST /checkout` with its ID:

```json
{
  "cartId": "your-cart-uuid"
}
```

Checkout requires an authenticated user and a non-empty cart. The response contains an `orderId` and Stripe Checkout URL. The app creates an order, redirects the customer to Stripe, and relies on verified Stripe webhook events to update the order payment status.

## Testing Stripe webhooks

For local webhook testing, start the app with your Stripe test secret key configured. In another terminal, forward Stripe CLI events to the webhook endpoint:

```bash
stripe login
stripe listen --forward-to localhost:8080/checkout/webhook
```

Stripe CLI prints a webhook signing secret beginning with `whsec_`. Set that value as `STRIPE_WEBHOOK_SECRET_KEY` in `.env`, then restart the app.

To simulate a successful payment-intent event with an order ID in its metadata, run:

```bash
stripe trigger payment_intent.succeeded \
  --add "payment_intent:metadata[orderId]=1"
```

Stripe CLI will set up and run the `payment_intent` fixture and report when the trigger succeeds. This creates a test event; it does not charge a real payment method. The `orderId` metadata must refer to an existing order for the webhook handler to update its status to `PAID`. Change `1` to the ID of the order you are testing.

Typical CLI output looks like this:

```text
Setting up fixture for: payment_intent
Running fixture for: payment_intent
Trigger succeeded! Check dashboard for event details.
```

## Database migrations

Flyway migration files are in `src/main/resources/db/migration/`. They run automatically when the application starts. To inspect or apply migrations manually:

```bash
./mvnw flyway:info
./mvnw flyway:migrate
```

Avoid `flyway:clean` except against disposable databases; it drops the schema.

## Deploying to Railway

Add a MySQL service to the project, then set these variables on the application service (replace `MySQL` with your database service name if it differs):

| Variable | Value |
| --- | --- |
| `SPRING_PROFILES_ACTIVE` | `prod` |
| `DB_URL` | `jdbc:mysql://${{MySQL.MYSQLHOST}}:${{MySQL.MYSQLPORT}}/${{MySQL.MYSQLDATABASE}}` |
| `DB_USERNAME` | `${{MySQL.MYSQLUSER}}` |
| `DB_PASSWORD` | `${{MySQL.MYSQLPASSWORD}}` |
| `FLYWAY_USER` | `${{MySQL.MYSQLUSER}}` (or a dedicated migration user) |
| `FLYWAY_PASSWORD` | `${{MySQL.MYSQLPASSWORD}}` |
| `JWT_SECRET`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET_KEY`, `WEBSITE_URL` | Production values |

Do not build `DB_URL` from Railway's `mysql://user:pass@host/db` URL variables: they are not JDBC URLs, and an unresolved reference leaves `DB_URL` empty, which fails startup with "Failed to determine a suitable driver class". The application listens on Railway's `PORT` (defaults to `8080`).

## Security notes

- Passwords supplied during user registration are encoded with BCrypt.
- Keep `JWT_SECRET`, Stripe keys, database credentials, and webhook signing secrets out of source control.
- Use Stripe test-mode credentials for testing; configure production secrets and HTTPS separately before deployment.
- Review authorization for user, product, order, and cart operations before exposing the service publicly.
