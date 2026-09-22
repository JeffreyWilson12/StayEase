<p align="center">
  <img src="frontend/public/favicon.svg" alt="StayEase logo" width="72" height="72" />
</p>

<h1 align="center">StayEase</h1>

<p align="center">
  <strong>Smart Hotel Management &amp; Reservation Platform</strong><br />
  A full-stack web application for running hotel operations: reservations, rooms, guests, staff and payments.
</p>

<p align="center">
  React 19 · Material UI · Spring Boot 4 · Java 17 · MySQL 8 · Docker
</p>

![StayEase dashboard](docs/screenshots/dashboard.png)

---

## Overview

StayEase is a hotel operations workspace. Front-desk and management staff use it to book rooms, keep guest profiles, track room availability, record payments and see what is happening across their properties on a single dashboard.

The project is a React single-page application talking to a Spring Boot REST API backed by MySQL. It runs locally with one `docker compose up` command.

## Features

**Operations dashboard**
- Occupancy rate, arrivals and departures today, active stays and collected revenue
- Room-status breakdown, reservations by status and by check-in month
- Upcoming arrivals (next 7 days), latest reservations and a payments summary
- Every figure is calculated from live API data; charts can be switched to an accessible table view

**Reservations**
- Create, edit and delete bookings with guest, room, handling employee, dates, guest count and status
- The server rejects overlapping bookings for the same room and rooms under maintenance; the form warns about overlaps before submitting
- Check-out must be after check-in, and guests must be at least 1
- Shows the number of nights and the estimated room charge
- Status counters double as quick filters; search by guest, room or reservation number

**Rooms and hotels**
- Room inventory with type, floor, nightly rate and status (Available / Reserved / Occupied / Maintenance)
- Filter rooms by hotel, type and status
- Hotel cards with star rating, contact details, room count, availability and occupancy

**Guests, staff and payments**
- Guest profiles with contact details, location and reservation history (stay count, last stay)
- Employee directory with position, hire date and tenure, salary and contact details
- Payment records per reservation (method, status, amount, date) with collected / pending / refunded totals
- Payment form lists only reservations that can be paid and suggests the room charge for the stay
- Shared address book showing which guests, hotels and employees use each address

**Across the app**
- Consistent design system (navy and gold hospitality theme) built on MUI theming
- Responsive layout: collapsible sidebar on desktop, drawer navigation and card lists on phones, full-screen dialogs on small screens
- Loading skeletons, empty states, inline error states with retry, and success / failure notifications for every create, update and delete
- Client-side validation that mirrors the server's bean-validation rules; server error messages are shown inside the dialog that caused them

> StayEase records payments manually. It does not process cards or integrate with a payment gateway.

## Screenshots

| Sign in | Reservations |
| --- | --- |
| ![Sign in](docs/screenshots/login.png) | ![Reservations](docs/screenshots/reservations.png) |

| New reservation | Rooms |
| --- | --- |
| ![Reservation dialog](docs/screenshots/reservation-dialog.png) | ![Rooms](docs/screenshots/rooms.png) |

| Hotels | Payments |
| --- | --- |
| ![Hotels](docs/screenshots/hotels.png) | ![Payments](docs/screenshots/payments.png) |

**Mobile (390 px)**

![Mobile dashboard and reservations](docs/screenshots/mobile.png)

## Technology Stack

| Layer | Technologies |
| --- | --- |
| Frontend | React 19, Vite 8, JavaScript (JSX), Material UI 6, MUI X Data Grid, MUI X Charts, React Router 7, Axios |
| Backend | Java 17, Spring Boot 4, Spring Web MVC, Spring Data JPA / Hibernate, Bean Validation, Spring Security Crypto (BCrypt), springdoc-openapi (Swagger UI), Lombok, Maven |
| Database | MySQL 8.4 |
| Delivery | Docker, Docker Compose, Nginx (serves the production build) |
| Quality | JUnit 5, Mockito, AssertJ (backend); oxlint (frontend) |

## Architecture

```text
Browser ── React SPA (Vite build, served by Nginx on :3000)
              │  pages → hooks (useApiData, useDialogForm) → services → Axios
              ▼
           REST API /api/**  (Spring Boot on :8080)
              │  controllers → services (business rules) → JPA repositories
              ▼
           MySQL 8 (hotelmanagementdb)
```

- **Frontend.** Each module (reservations, rooms, …) has a page that loads its data with `useApiData`, filters it client-side, and renders shared building blocks: `DataTable`, `FilterBar`, `StatusSummary`, `FormDialog`, `ConfirmDialog`. Forms keep their validation schema and API mapping in a `*FormModel.js` file next to the component.
- **Backend.** Controllers validate request DTOs with `@Valid`. Services enforce business rules (no overlapping reservations, one payment per reservation, no payments for cancelled reservations, rooms under maintenance cannot be booked). A global exception handler turns errors into consistent JSON (`timestamp`, `status`, `error`, `message`).

## Project Structure

```text
.
├── backend/                     Spring Boot REST API
│   ├── src/main/java/com/hotelmanagement/
│   │   ├── config/              CORS, BCrypt, OpenAPI configuration
│   │   ├── controller/          REST controllers (/api/...)
│   │   ├── dto/                 request / response DTOs with validation
│   │   ├── enums/               RoomStatus, ReservationStatus, PaymentMethod, ...
│   │   ├── exception/           GlobalExceptionHandler, error types
│   │   ├── model/               JPA entities
│   │   ├── repository/          Spring Data repositories
│   │   └── service/             business logic (interfaces + impl/)
│   ├── src/test/java/...        unit tests
│   └── Dockerfile
├── frontend/                    React single-page app
│   ├── src/
│   │   ├── api/                 Axios instance
│   │   ├── auth/                session helpers and route guards
│   │   ├── components/          layout shell, shared UI (common/), module components
│   │   ├── constants/           enum values and display labels
│   │   ├── hooks/               useApiData, useForm, useDialogForm, useDialogState
│   │   ├── pages/               one page per module + auth pages
│   │   ├── services/            API calls per resource, dashboard metrics
│   │   ├── theme/               design tokens and MUI theme
│   │   └── utils/               formatting, validation, error messages
│   ├── nginx.conf
│   └── Dockerfile
├── database/HotelManagementDB.sql   schema + sample data (loaded on first Docker start)
├── docs/screenshots/
├── installer/                   optional Windows launcher / Inno Setup script
├── docker-compose.yml
└── .env.example
```

## Authentication

- **Sign up** (`POST /api/auth/register`) creates a user with the `USER` role. Passwords are hashed with BCrypt and never stored or returned in plain text. E-mail addresses are normalised to lower case.
- **Sign in** (`POST /api/auth/login`) verifies the password against the BCrypt hash. Wrong e-mail and wrong password return the same `401` message, so the endpoint does not reveal which accounts exist.
- On success the frontend stores the user's public profile (id, name, e-mail, role) in `sessionStorage`. Application routes are guarded on the client and redirect to the sign-in page when nobody is signed in. Signing out clears the session.

> **Scope:** this is front-end session state only. The API does not issue or verify tokens, and the resource endpoints (`/api/rooms`, `/api/reservations`, …) are not protected server-side. See [Security](#security) and [Future Improvements](#future-improvements).

## Database

MySQL database `hotelmanagementdb`. Main tables:

| Table | Purpose |
| --- | --- |
| `address` | Shared addresses referenced by guests, hotels and employees |
| `customer` | Guests (shown as **Guests** in the UI) |
| `hotel` | Properties |
| `room` | Rooms per hotel: type, floor, price, status |
| `employee` | Staff: position, salary, hire date |
| `reservation` | Guest + room + handling employee + dates, guest count and status |
| `payment` | One payment per reservation (deleted with its reservation) |
| `users` | Application accounts (BCrypt password hashes) |

`database/HotelManagementDB.sql` creates the schema and loads sample data. All monetary values (room rates, salaries, payment amounts) are stored and displayed in Indian rupees (₹, INR). Docker Compose runs it automatically the first time the MySQL volume is created. The dump also contains `invoice`, `service` and `reservation_service` tables that the application does not use yet.

## API

All endpoints are under `http://localhost:8080/api`.

| Resource | Endpoints |
| --- | --- |
| Auth | `POST /auth/register`, `POST /auth/login` |
| Customers (guests) | `GET/POST /customers`, `GET/PUT/DELETE /customers/{id}`, `GET /customers/search?lastName=` |
| Addresses | `GET/POST /addresses`, `GET/PUT/DELETE /addresses/{id}` |
| Hotels | `GET/POST /hotels`, `GET/PUT/DELETE /hotels/{id}`, `GET /hotels/stars/{stars}`, `GET /hotels/search?name=` |
| Rooms | `GET/POST /rooms`, `GET/PUT/DELETE /rooms/{id}`, `GET /rooms/hotel/{hotelId}`, `/rooms/status/{status}`, `/rooms/type/{type}` |
| Employees | `GET/POST /employees`, `GET/PUT/DELETE /employees/{id}`, `GET /employees/lastname/{lastName}`, `/employees/position/{position}` |
| Reservations | `GET/POST /reservations`, `GET/PUT/DELETE /reservations/{id}`, `GET /reservations/customer/{id}`, `/room/{id}`, `/employee/{id}`, `/status/{status}` |
| Payments | `GET/POST /payments`, `GET/PUT/DELETE /payments/{id}`, `GET /payments/reservation/{id}`, `/status/{status}`, `/method/{method}` |

Error responses share one format:

```json
{ "timestamp": "2026-09-22T23:37:28", "status": 400, "error": "Bad Request",
  "message": "Check-out date must be after check-in date." }
```

| Status | When |
| --- | --- |
| `400` | Validation failure, business-rule violation, malformed JSON or unknown enum value |
| `401` | Invalid sign-in |
| `404` | Record not found |
| `409` | Database constraint conflict, e.g. deleting a hotel whose rooms have reservations |
| `500` | Unexpected error (details are logged server-side, not returned) |

## Installation

### Prerequisites

- **Docker route (recommended):** Docker Desktop with Docker Compose
- **Manual route:** Java 17, Node.js 20.19+ (or 22.12+), MySQL 8

### Environment Variables

Copy `.env.example` to `.env` in the project root and set your own values. `.env` is git-ignored and must never be committed.

| Variable | Used by | Description |
| --- | --- | --- |
| `MYSQL_DATABASE` | db, backend | Database name (`hotelmanagementdb`) |
| `MYSQL_ROOT_PASSWORD` | db | MySQL root password for the container |
| `SPRING_DATASOURCE_USERNAME` | backend | Database user (`root` in the default setup) |
| `SPRING_DATASOURCE_PASSWORD` | backend | Database password |
| `APP_CORS_ALLOWED_ORIGINS` | backend (optional) | Comma-separated front-end origins; defaults to `http://localhost:3000,http://localhost:5173` |
| `VITE_API_BASE_URL` | frontend build (optional) | API base URL; defaults to `http://localhost:8080/api` (see `frontend/.env.example`) |

## Docker

```bash
cp .env.example .env        # then edit the passwords
docker compose up --build -d
```

| Service | URL |
| --- | --- |
| Frontend (Nginx) | http://localhost:3000 |
| Backend API | http://localhost:8080/api |
| Swagger UI | http://localhost:8080/swagger-ui/index.html |
| MySQL | localhost:3307 |

Open http://localhost:3000, create an account on the sign-up page, and sign in.

```bash
docker compose ps           # container status
docker compose logs -f backend
docker compose down         # stop (keeps the database volume)
docker compose down -v      # stop AND delete the database volume (irreversible)
```

The Compose project is named `stayease`, so its database volume is `stayease_mysql_data`. A volume created by an earlier version of this repository under a different project name is not reused, so the first start initialises a fresh database from the SQL file.

## Running Locally

**Database.** Create `hotelmanagementdb` in MySQL 8 and import `database/HotelManagementDB.sql`. Alternatively, start only the Docker database with `docker compose up -d db`, which listens on port 3307.

**Backend**

```bash
cd backend
cp src/main/resources/application-example.properties src/main/resources/application.properties
# edit the datasource URL / username / password (application.properties is git-ignored)
./mvnw spring-boot:run            # Windows: mvnw.cmd spring-boot:run
```

**Frontend**

```bash
cd frontend
npm install
npm run dev                       # http://localhost:5173
```

## API Documentation

With the backend running, interactive OpenAPI docs are available at:

- Swagger UI: http://localhost:8080/swagger-ui/index.html
- OpenAPI JSON: http://localhost:8080/v3/api-docs

## Security

Measures in place:

- BCrypt password hashing; passwords are never returned by the API
- Bean validation on every write endpoint, mirrored by client-side validation
- Generic error message for `500` responses (stack traces and SQL details are only logged)
- Constraint violations mapped to a friendly `409` without leaking SQL
- Uniform sign-in failure message (no account enumeration through error text)
- CORS limited to configured front-end origins
- Credentials supplied through environment variables; `.env` and `application.properties` are git-ignored and excluded from Docker build contexts
- Nginx sends `X-Content-Type-Options`, `X-Frame-Options` and `Referrer-Policy` headers
- React escapes all rendered data; no `dangerouslySetInnerHTML`

Known gaps (do not deploy to the public internet as-is):

- REST endpoints are **not** authenticated or authorised server-side; the client-side route guard is a UX feature, not access control
- No rate limiting on sign-in
- The Docker setup uses the MySQL `root` account and plain HTTP

## Testing

```bash
# Backend unit tests (no database needed)
cd backend && ./mvnw test

# Include the full Spring context test against a running database
SPRING_DATASOURCE_URL="jdbc:mysql://localhost:3307/hotelmanagementdb" \
SPRING_DATASOURCE_USERNAME=root SPRING_DATASOURCE_PASSWORD=... ./mvnw test

# Frontend lint and production build
cd frontend && npm run lint && npm run build
```

Backend tests cover the reservation rules (check-out after check-in, overlap detection including back-to-back stays and cancelled bookings, maintenance rooms), the payment rules (duplicate payment, cancelled reservation), authentication (hashing, e-mail normalisation, duplicate account, wrong password) and request validation. There are no automated frontend tests yet.

## Future Improvements

- Token-based authentication (e.g. Spring Security + JWT) with role-based authorisation on the API
- Server-side pagination and filtering for large datasets
- Frontend component and end-to-end tests
- Guest detail view with full stay and payment history
- Automatic room-status updates from check-in / check-out
- Invoices and extra services (tables already exist in the schema)
- Database migrations with Flyway instead of a SQL dump

## Author

**Yannis Drougas** — original Hotel Management System.

StayEase is the redesigned and extended version of that project.
