# interview-relational-data-api

> A production-grade RESTful data management service built with NestJS 11, Prisma ORM 6, and PostgreSQL 14, featuring OpenAPI/Swagger interactive documentation, declarative DTO validation, and containerized database infrastructure.

**Inception (First Commit):** `2025-01-20` • **Batch / Horizon:** `Batch 2024–2025` • **Architecture Domain:** `Relational Backend / API Data Modeling`

---

## 📌 Architecture & Overview

`JobInterview_api-data` (internally titled `planetwatch`) is a backend technical interview assessment designed to implement a robust, maintainable, and type-safe REST API for persistent key-value configuration data and environment variable retrieval.

The application follows the classic **Layered Clean Architecture** pattern enforced by NestJS modular dependency injection:

```
[ HTTP Client / Swagger UI ]
              │
              ▼
   ┌────────────────────────────────────────────────────────┐
   │                  HTTP Request Pipeline                 │
   │  ┌───────────────────────┐  ┌───────────────────────┐  │
   │  │ ValidationPipe        │  │ GlobalExceptionFilter │  │
   │  │ (Whitelist/Transform) │  │ (Timestamp, Path, Msg)│  │
   │  └───────────┬───────────┘  └───────────────────────┘  │
   └──────────────┼─────────────────────────────────────────┘
                  ▼
   ┌────────────────────────────────────────────────────────┐
   │              DataModule / DataController               │
   │  (Routes: /data, /data/:name, /data/env)               │
   └──────────────────────┬─────────────────────────────────┘
                          ▼
   ┌────────────────────────────────────────────────────────┐
   │                      DataService                       │
   │  (Business Logic, Conflict/NotFound Exception Mapping) │
   └──────────────┬───────────────────────────┬─────────────┘
                  │                           │
                  ▼                           ▼
   ┌─────────────────────────────┐  ┌───────────────────────┐
   │        PrismaService        │  │     ConfigService     │
   │  (PrismaClient PostgreSQL)  │  │ (Process Environment) │
   └──────────────┬──────────────┘  └───────────────────────┘
                  ▼
   ┌─────────────────────────────┐
   │     PostgreSQL Database     │
   │      (Table: DataTable)     │
   └─────────────────────────────┘
```

### Context & Architectural Highlights

* **Context:** Built as a backend technical assessment simulating a configuration and metadata ingestion microservice for the PlanetWatch ecosystem.
* **Strict Layer Separation:** Decouples HTTP request routing ([`DataController`](src/modules/data/data.controller.ts)) from database querying and exception translation ([`DataService`](src/modules/data/data.service.ts)) and data persistence abstraction ([`PrismaService`](src/prisma/prisma.service.ts)).
* **Resilient Startup & Teardown:** Connects to PostgreSQL directly during bootstrapping with explicit error trapping (`prismaService.$connect()`), aborting immediately on connection failure and enabling graceful teardown via `app.enableShutdownHooks()`.
* **Standardized Error Handling:** Employs a centralized [`GlobalExceptionFilter`](src/filters/http-exception.filter.ts) ensuring all errors—whether HTTP status exceptions or unexpected runtime failures—are logged with timestamps and returned in a unified JSON envelope.
* **Declarative Request Validation:** All incoming mutations are validated against Data Transfer Objects ([`CreateDataDto`](src/modules/data/dto/create-data.dto.ts) and [`UpdateDataDto`](src/modules/data/dto/update-data.dto.ts)) utilizing `class-validator` and `class-transformer` with strict stripping of non-whitelisted properties.

---

## 🛠 Tech Stack

* **Runtime & Framework:** Node.js (LTS v20 / v22), TypeScript 5.7.3, NestJS 11.0.2 (`@nestjs/core`, `@nestjs/common`, `@nestjs/platform-express`, `@nestjs/config`, `@nestjs/mapped-types`)
* **Database & ORM:** PostgreSQL 14 (Alpine), Prisma ORM 6.2.1 (`@prisma/client`, `prisma`)
* **API Documentation:** Swagger / OpenAPI 3.0 (`@nestjs/swagger`, `swagger-ui-express`)
* **Validation & Transformation:** `class-validator` 0.14.1, `class-transformer` 0.5.1
* **Testing & Tooling:** Jest 29.7.0, Supertest 7.0.0, ts-jest 29.2.5, ESLint 9.18.0, Prettier 3.4.2
* **Containerization:** Docker & Docker Compose (PostgreSQL 14 + pgAdmin 4)

---

## ✨ Features & Endpoints

* **Data Entry Ingestion & Conflict Prevention:** Upserts structured configuration records into PostgreSQL with unique name enforcement and Prisma `P2002` to HTTP `409 Conflict` mapping.
* **Partial Updates via Mapped Types:** Allows dynamic partial patching of `DataValue` or `DataRequired` flags using `UpdateDataDto` derived via `PartialType`.
* **Safe Deletions:** Deletes records by natural key (`DataName`) with automatic Prisma `P2025` to HTTP `404 Not Found` handling.
* **Environment Configuration Querying:** Exposes an interface to look up runtime environment variables via `ConfigService`.
* **Self-Documenting API:** Interactive Swagger documentation accessible at `/api`.

### API Routes & Schema Reference

| Method | Endpoint | Description | Request Body / Params | Status Responses |
|---|---|---|---|---|
| `POST` | `/data` | Creates a new record in `DataTable`. | `{ "name": string, "value": string, "required": boolean }` | `201 Created`<br>`400 Bad Request`<br>`409 Conflict` |
| `GET` | `/data` | Retrieves all records from `DataTable`. | *None* | `200 OK`<br>`500 Internal Error` |
| `PATCH` | `/data/:name` | Partially updates value or required state by name. | Param: `:name`<br>Body: `{ "value"?: string, "required"?: boolean }` | `200 OK`<br>`400 Bad Request`<br>`404 Not Found` |
| `DELETE` | `/data/:name` | Deletes a record by name. | Param: `:name` | `200 OK`<br>`404 Not Found` |
| `GET` | `/data/env` | Reads a runtime environment variable by key. | Query: `?key=<variable_name>` | `200 OK`<br>`400 Bad Request`<br>`404 Not Found` |
| `GET` | `/api` | OpenAPI / Swagger interactive documentation UI. | *None* | `200 OK` |

#### Data Entity Schema (`prisma/schema.prisma`)

```prisma
model DataTable {
  id           Int      @id @default(autoincrement())
  DataName     String   @unique
  DataValue    String
  DataRequired Boolean
  createdAt    DateTime @default(now())
}
```

---

## 🚀 Getting Started

### Prerequisites

* **Node.js:** `>= 20.17.0` (LTS v22 recommended)
* **Package Manager:** `pnpm` (`>= 9.0.0`) or `npm`
* **Container Engine:** Docker & Docker Compose

---

### Installation

1. **Clone the repository:**
   ```bash
   git clone https://github.com/fabogit/interview-relational-data-api.git
   cd interview-relational-data-api
   ```

2. **Install project dependencies:**
   ```bash
   pnpm install
   # or: npm install
   ```

---

### Environment Configuration

The repository uses two environment configuration files: one for the NestJS application and one for the Dockerized database infrastructure.

1. **Configure Application Environment (`.env`):**
   Copy the sample environment file to `.env`:
   ```bash
   cp ".env copy" .env
   ```

   Ensure `.env` matches your local database settings:
   ```env
   DATABASE_URL="postgresql://myuser:mypassword@localhost:5432/mydatabase?schema=public"
   PORT=3000
   VARIABLE="a string"
   ```

2. **Verify Docker Environment (`docker/docker.env`):**
   ```env
   POSTGRES_USER=myuser
   POSTGRES_PASSWORD=mypassword
   POSTGRES_DB=mydatabase
   PGADMIN_DEFAULT_EMAIL=admin@admin.com
   PGADMIN_DEFAULT_PASSWORD=admin
   POSTGRES_PORT=5432
   PGADMIN_PORT=5050
   ```

---

### Running External Dependencies (PostgreSQL & pgAdmin)

Spin up the containerized database services using Docker Compose:

```bash
# Start PostgreSQL (port 5432) and pgAdmin (port 5050)
pnpm run docker:up
# or: docker-compose -f ./docker/docker-compose.yaml --env-file ./docker/docker.env up -d
```

To stop containers and preserve volume data:
```bash
pnpm run docker:down
```

---

### Database Migrations & Prisma Generation

Generate the Prisma client and apply existing database migrations:

```bash
# Generate the strongly-typed Prisma Client
npx prisma generate

# Apply migrations to PostgreSQL
npx prisma migrate dev
```

*(Optional)* Launch Prisma Studio to inspect the database via browser:
```bash
npx prisma studio
```

---

### Running the Application

```bash
# Development mode with Hot-Reload / Watch
pnpm run start:dev

# Standard start
pnpm run start

# Production build and execution
pnpm run build
pnpm run start:prod
```

Once running:
* **API Server:** `http://localhost:3000`
* **Swagger API Documentation:** `http://localhost:3000/api`
* **pgAdmin Dashboard:** `http://localhost:5050` (Login: `admin@admin.com` / `admin`)

---

### Running Tests

```bash
# Execute unit and service-level test suites
pnpm run test

# Run tests in watch mode
pnpm run test:watch

# Generate test coverage reports
pnpm run test:cov

# Execute End-to-End (E2E) integration tests
pnpm run test:e2e
```

---

## 📄 License & Author

Developed by [@fabogit](https://github.com/fabogit).
