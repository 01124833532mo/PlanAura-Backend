# Planura — Backend

The API that powers **Planura**, an AI-powered event planning and vendor
booking marketplace. Built with ASP.NET Core using Clean Architecture.

> Looking for the client app? See [PlanAura-Frontend](../../../PlanAura-Frontend).

---

## What this API does

Planura connects people planning an event (weddings, engagements, birthdays,
corporate events) with verified vendors (wedding halls, photographers, DJs,
caterers, decorators, makeup artists). This service is the backend behind:

- **Client & vendor accounts** — registration, login, role-based access
  (`client`, `vendor`, `admin`) via ASP.NET Identity and JWT.
- **Vendor onboarding & verification** — vendors register with business
  details, ID documents and portfolio images; an admin reviews and
  approves or rejects each submission before the vendor can accept bookings.
- **Vendor packages & availability** — vendors publish priced packages and
  keep a live availability calendar.
- **Bookings & deposits** — a booking authorizes a deposit on the client's
  card, the vendor accepts, the remainder is charged automatically ahead of
  the event, with a grace period and admin-reviewed cancellations/refunds
  if something goes wrong. See [`Deposit_Lifecycle.md`](./Deposit_Lifecycle.md)
  for the full state machine.
- **Admin tools** — vendor verification review, account suspension, and
  (in progress) a dashboard — see [`AdminDashboardCompletionReport.md`](./AdminDashboardCompletionReport.md).

For a detailed, code-verified snapshot of what is implemented versus
planned, see [`PROJECT_STATUS.md`](./PROJECT_STATUS.md).

---

## Tech stack

| Layer | Technology |
|---|---|
| Framework | ASP.NET Core Web API (C#) |
| Architecture | Clean Architecture (Domain / Application / Infrastructure / API, split into separate class libraries) |
| Database | SQL Server |
| ORM | Entity Framework Core (code-first migrations) |
| Data access | Generic repository + Unit of Work, Specification pattern |
| Auth | ASP.NET Core Identity + JWT bearer tokens |
| Object mapping | AutoMapper |
| Background jobs | Hangfire (deposit remainder charging, grace-period expiry) |
| Payments | Stripe (deposit authorization, remainder capture, refunds, webhooks) |
| API docs | Swagger / OpenAPI |

---

## Solution structure

```
Planura.sln
├── Planura.Core.Domain                   # Entities, enums, constants, repository interfaces
├── Planura.Core.Application               # DTOs, service interfaces + implementations,
│                                           # specifications, AutoMapper profiles
├── Planura.Core.Application.Abstraction   # Thin interfaces shared across layers (e.g. IAttachmentService)
├── Planura.Infrastructure                 # JWT/token service, current-user service, file storage,
│                                           # authorization handlers, DI wiring
├── Planura.Infrastructure.Persistence     # DbContext, EF Core configurations, migrations,
│                                           # UnitOfWork implementation, identity/role seeding
├── Planura.Apis.Controller                # MVC controllers and their request/response models
├── Planura.Apis                           # ASP.NET Core host — Program.cs, Swagger, middleware
├── Planura.Shared                         # Cross-cutting error types and response shapes
└── Planura.Tests                          # Test project
```

Dependencies flow inward: `Planura.Core.Domain` has no dependency on any
other layer; `Planura.Apis` depends on everything below it.

---

## Getting started

### Prerequisites

- [.NET SDK](https://dotnet.microsoft.com/download) (matching the version targeted by `Planura.sln`)
- SQL Server (local instance or container)
- A Stripe account (test-mode secret/publishable keys) for the payment flow

### Clone & restore

```bash
git clone https://github.com/PlanAuraa/PlanAura-Backend.git
cd PlanAura-Backend
dotnet restore Planura.sln
```

### Configure

Add your local settings to `Planura.Apis/appsettings.Development.json`
(create it if it doesn't exist — it should not be committed):

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=.;Database=PlanuraDb;Trusted_Connection=True;TrustServerCertificate=True"
  },
  "Jwt": {
    "Key": "your-local-signing-key",
    "Issuer": "Planura",
    "Audience": "PlanuraClient",
    "ExpiryMinutes": 60
  },
  "Stripe": {
    "SecretKey": "sk_test_...",
    "PublishableKey": "pk_test_...",
    "WebhookSecret": "whsec_..."
  },
  "Booking": {
    "DepositPercentage": 20,
    "FullPaymentThresholdDays": 7,
    "RemainderChargeLeadDays": 4,
    "GracePeriodDays": 2,
    "RemainderChargingTimeoutMinutes": 60
  }
}
```

> Never commit real connection strings, JWT signing keys or Stripe secret
> keys. Use `sk_test_...` / `pk_test_...` Stripe keys for local development.

### Apply migrations

```bash
dotnet ef database update --project Planura.Infrastructure.Persistence --startup-project Planura.Apis
```

Identity roles (`admin`, `vendor`, `client`) are seeded automatically on
startup.

### Run the API

```bash
dotnet run --project Planura.Apis
```

Swagger UI is available at the root or `/swagger` once the app is running,
listing all controllers and endpoints.

### Run tests

```bash
dotnet test Planura.sln
```

---

## Authentication & authorization

- **Login/registration**: `POST api/auth/register/client`, `POST api/auth/register/vendor`
  (multipart/form-data — vendor registration includes ID documents and
  portfolio images), `POST api/auth/login`, `GET api/auth/me`.
- **Tokens**: JWT bearer tokens carrying the user id, email, name, one role
  claim per assigned role, and — for vendors — a `vendor_id` claim. Token
  validity is re-checked against the user's active status on every request.
- **Policies**: `ClientOnly`, `VendorOnly`, `AdminOnly`, and `ApprovedVendor`
  (requires the vendor role *and* a verification status of `verified` or
  `trusted`, re-checked on every request — approval takes effect
  immediately, no re-login required).

---

## Background jobs

Two Hangfire jobs run hourly against the booking/deposit state machine:

- `deposit-remainder-charge` — charges the remaining balance ahead of the event.
- `deposit-remainder-grace-expiry` — routes unpaid, grace-expired bookings to admin review.

See [`Deposit_Lifecycle.md`](./Deposit_Lifecycle.md) for the full flow,
states and configuration.

---

## Branching & contributing

- `develop` is the active integration branch — branch off it for new work.
- Run `dotnet build` and `dotnet test` locally before opening a pull request.
- Follow the existing Clean Architecture boundaries — domain logic stays
  free of infrastructure concerns; new external integrations go in
  `Planura.Infrastructure`.
- Keep commits scoped and use clear messages (e.g. `feat: vendor profile endpoint`,
  `fix: remainder charge retry`).

---

## Team

| Role | Name |
|---|---|
| Team Lead | Mohamed Wahba |
| Team Member | Mohamed Hamdy |
| Team Member | Mahmoud Rehan |
| Team Member | Ibrahim Mohamed |
| Team Member | Doaa Ahmed |

---

## License

No license has been set for this project yet.
