# Yazh Digital Studio — Instagram DM Booking Bot V1

A local-first booking dashboard and DM simulator for Yazh Digital Studio. V1 runs without an Instagram account or Meta credentials. The simulator drives the same persisted conversation state machine that a future official Meta webhook can call.

## 1. Project overview

- Manage service offerings, appointments, customers, and available studio times.
- Test a complete booking as a customer in the Instagram-style local simulator.
- Seeded development services, people, times, and example bookings are created on first API start.
- PostgreSQL is the system of record; outgoing Instagram messages are mocked into the local conversation store.
- Instagram webhook endpoints deliberately return `501 Not Implemented` until official Meta verification and event handling are configured.

## 2. Architecture

```text
YazhBookingBot/
├── backend/
│   ├── YazhBookingBot.Api/             ASP.NET Core controllers, middleware, Swagger, composition root
│   ├── YazhBookingBot.Application/     Booking contracts, store interfaces, conversation state machine
│   ├── YazhBookingBot.Domain/          EF-independent entities and enums
│   ├── YazhBookingBot.Infrastructure/  EF Core / PostgreSQL persistence and mock messaging
│   └── YazhBookingBot.Tests/           State-machine unit tests and API integration test
├── frontend/yazh-booking-dashboard/   Angular standalone dashboard and chat simulator
├── docker-compose.yml
└── README.md
```

The domain layer does not depend on EF Core. API controllers accept request DTOs, and the application state machine uses `IBookingStore`. `IInstagramMessagingService` and `IStoryInteractionMapper` are the integration seams for official messaging and story interactions.

Active bookings are protected in two places: the application checks the slot inside a serializable database transaction, and PostgreSQL has a unique partial index on `BusinessSlotId` for Pending and Confirmed bookings. If the slot loses a race while the customer is reviewing, the bot returns to slot selection.

## 3. Requirements

- .NET 8 SDK
- Node.js 20+ and npm (or pnpm)
- PostgreSQL 15+ for direct local runs
- Docker Desktop / Docker Compose v2 (optional)

## 4. Installation

From the repository root, copy `.env.example` to `.env` and provide local values. `DATABASE_CONNECTION_STRING` is required for a direct backend run. The API does not contain a default database password.

PowerShell example:

```powershell
Copy-Item .env.example .env
$env:DATABASE_CONNECTION_STRING = "Host=localhost;Port=5432;Database=yazh_booking;Username=postgres;Password=<your-local-password>"
```

Do not commit `.env`. For Docker Compose, set both `POSTGRES_PASSWORD` and `DATABASE_CONNECTION_STRING` in `.env`; the database hostname inside the connection string must be `db`.

## 5. PostgreSQL setup

Create a database named `yazh_booking` and a local development user. The database can be created with your PostgreSQL administration tool or:

```sql
CREATE DATABASE yazh_booking;
```

Point `DATABASE_CONNECTION_STRING` to it. For example, the connection string format is `Host=localhost;Port=5432;Database=yazh_booking;Username=postgres;Password=...`.

## 6. Environment variables

| Variable | Purpose |
| --- | --- |
| `DATABASE_CONNECTION_STRING` | Required Npgsql connection string. |
| `POSTGRES_PASSWORD` | Docker Compose database password. |
| `ASPNETCORE_ENVIRONMENT` | Set to `Development` locally to enable Swagger UI. |

`appsettings.json` contains non-secret logging and CORS settings. Add additional dashboard origins under `Cors:AllowedOrigins` for a deployed frontend.

## 7. Database schema and migration commands

The PostgreSQL initial migration is checked in under `backend/YazhBookingBot.Infrastructure/Persistence/Migrations`. On API start, EF Core applies pending migrations and then seeds the development data.

Install the EF command-line tool if needed and create later migrations from the repository root:

```powershell
dotnet tool install --global dotnet-ef --version 8.0.11
dotnet ef migrations add AddYourChange --project backend/YazhBookingBot.Infrastructure --startup-project backend/YazhBookingBot.Api
dotnet ef database update --project backend/YazhBookingBot.Infrastructure --startup-project backend/YazhBookingBot.Api
```

Set `DATABASE_CONNECTION_STRING` before running `database update`. The integration-test host uses `EnsureCreatedAsync` only for its isolated SQLite database; normal development, Docker, and production use versioned PostgreSQL migrations.

## 8. Start the backend

```powershell
$env:DATABASE_CONNECTION_STRING = "Host=localhost;Port=5432;Database=yazh_booking;Username=postgres;Password=<your-local-password>"
$env:ASPNETCORE_ENVIRONMENT = "Development"
dotnet run --project backend/YazhBookingBot.Api
```

The API listens on the URL printed by ASP.NET Core (normally `http://localhost:5080` when using Docker, or an assigned local port from `launchSettings.json`). Swagger is available at `/swagger` in Development.

## 9. Start Angular frontend

```powershell
cd frontend/yazh-booking-dashboard
npm install
npm start
```

The dev server runs at `http://localhost:4200`. Its proxy forwards `/api` requests to `http://localhost:5080`; set a different backend address in `proxy.conf.json` if needed. With Docker Compose, use the combined frontend/API routing described below.

## 10. Use the Chat Simulator

Open **Chat Simulator** in the sidebar. Select Arun, Priya, or Karthik from the seeded inbox, or choose **New chat** to create a local test customer. Send `Hi`, then use the quick replies or type the same answers yourself.

The state machine supports service, available date, available slot, name, Indian mobile phone, optional notes, confirmation, `back`, and `cancel`. A booking is committed only after the customer confirms. The normalized phone is stored as ten digits. The chat options and step are persisted, so a page refresh does not lose the conversation.

## 11. API documentation

Swagger: `http://localhost:5080/swagger` in Development (or `http://localhost:4200/api` for routed API calls when Compose is running).

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET, POST | `/api/services` | List and create services. `?activeOnly=true` filters customer menu choices. |
| PUT, DELETE | `/api/services/{id}` | Update, delete, or deactivate a service with history. |
| GET, POST | `/api/slots` | List and create studio times. Supports `from`, `to`, and `availableOnly` filters. |
| POST | `/api/slots/generate` | Generate consecutive slots for a date and working window. |
| PUT, DELETE | `/api/slots/{id}` | Update availability/time or remove an empty slot. |
| GET | `/api/bookings` | List bookings; supports `date` and `status` filters. |
| GET | `/api/bookings/{id}` | Read booking details. |
| POST | `/api/bookings` | Create a pending admin booking. Active slot conflicts return 409. |
| PUT | `/api/bookings/{id}/confirm` | Confirm a pending booking. |
| PUT | `/api/bookings/{id}/cancel` | Cancel an active booking. |
| PUT | `/api/bookings/{id}/complete` | Mark a booking complete. |
| GET | `/api/customers` | List customers with booking history. |
| GET | `/api/customers/{id}` | Read one customer and booking history. |
| POST | `/api/chat/message` | Process one simulator DM through the state machine. |
| GET | `/api/chat/sessions` | List local simulator conversations. |
| GET | `/api/chat/session/{customerId}` | Read one local simulator conversation. |
| GET | `/api/dashboard` | Dashboard counts. |
| GET, POST | `/api/instagram/webhook` | Explicit `501` placeholder for future Meta setup. |

Example first DM:

```http
POST /api/chat/message
Content-Type: application/json

{ "customerId": "test-user-001", "message": "Hi" }
```

The response contains `reply`, `options`, `step`, and the local customer/conversation IDs. Send the selected option text as the next `message`.

## 12. Testing

Run backend tests with:

```powershell
dotnet test backend/YazhBookingBot.Tests/YazhBookingBot.Tests.csproj
```

The unit suite exercises customer creation, service/date/slot selection, invalid choices, invalid phone numbers, normalization, back/cancel, confirmation, and competing confirmation attempts. The integration test runs the main API chat flow against an isolated SQLite test database and verifies that the same active slot cannot be booked twice. PostgreSQL's filtered unique index is specific to the production provider and should also be exercised in a PostgreSQL-backed CI job.

Build the API with `dotnet build backend/YazhBookingBot.Api/YazhBookingBot.Api.csproj`. Build Angular with `npm run build` from `frontend/yazh-booking-dashboard`.

## 13. Docker setup

Copy `.env.example` to `.env`, set `POSTGRES_PASSWORD`, and set `DATABASE_CONNECTION_STRING` to a container-local connection such as `Host=db;Port=5432;Database=yazh_booking;Username=postgres;Password=<same-password>`.

```powershell
docker compose up --build
```

- Dashboard: `http://localhost:4200`
- API: `http://localhost:5080`
- Swagger: `http://localhost:5080/swagger`
- PostgreSQL: `localhost:5432`

Compose waits for PostgreSQL health before starting the API and persists data in the `yazh-postgres-data` volume.

## 14. Future Instagram integration

`IInstagramMessagingService` abstracts text, quick reply, and button messages. `MockInstagramMessagingService` writes outgoing mock messages to the simulator's conversation store. `InstagramWebhookController` is a visible unconfigured boundary; it responds with `501` rather than claiming delivery or verification. `IStoryInteractionMapper` maps future story taps/replies to the same customer key used by `ConversationEngine`.

## 15. Meta Developer setup requirements

Before implementing the official adapter, create a Meta developer app, configure an Instagram professional account and messaging permissions, complete any required app review/business verification, configure a public HTTPS webhook URL, and obtain/manage access tokens through a secret store. Add webhook challenge verification, request signature validation, event idempotency, rate-limit handling, and token refresh/rotation. Keep credentials out of source control and out of the local simulator.

## 16. Production deployment notes

- Replace the V1 `EnsureCreated` bootstrap with versioned EF migrations before production rollout.
- Put PostgreSQL behind private networking, require TLS where supported, and store connection strings in the deployment secret manager.
- Add real admin authentication/authorization before exposing management endpoints publicly. The API defines a named `Admin` policy seam; it is not an authentication implementation.
- Set CORS to only the production dashboard origin, configure HTTPS and forwarded headers, and keep Swagger disabled outside Development unless access-controlled.
- Add backups, migration roll-forward/rollback procedures, structured log shipping, metrics, health/readiness probes, and PostgreSQL-backed integration tests.
- Keep the mock messaging adapter selected only for local/test deployments; register a Meta adapter only after the official integration is implemented and verified.
