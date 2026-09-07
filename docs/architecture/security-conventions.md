# Security Conventions

Explicit security decisions for this project. `security-reviewer` and
`design-reviewer` enforce these; `spec-writer` cites them; `dotnet-implementor`
implements to them.

> **These are intentional training-project decisions, not inferred universal
> ASP.NET Core defaults.** A Story may deviate only through a resolved Open
> Decision approved by a human.

## Training-project security policy

```yaml
authentication_model: cookie/session-based for initial MVP unless a Story explicitly introduces bearer tokens
password_hashing: BCrypt (BCrypt.Net-Next)
password_min_length: 12
password_max_length: 72
password_requirements:
  - at least one uppercase letter
  - at least one lowercase letter
  - at least one digit
  - at least one special character
csrf:
  browser_session_endpoints: enabled (antiforgery tokens)
  stateless_api_endpoints: requires explicit architecture decision
database_admin_ui:
  enabled: false
secrets:
  committed_to_repository: forbidden
schema_mutation:
  allowed:
    - EF Core Migrations (`dotnet ef migrations add`, applied via `Database.Migrate()`)
  forbidden:
    - `Database.EnsureCreated()` / `EnsureDeleted()` against the file database
    - raw `ExecuteSqlRaw` schema changes
```

## SC-1 Passwords

- Hash with `BCrypt.Net.BCrypt.HashPassword` (project-documented work factor,
  minimum cost `12`) behind an `IPasswordHasher` abstraction that lives in
  the `Security` namespace. A no-op / plaintext hasher is forbidden.
- Enforce the `password_*` policy above during request validation (a custom
  `ValidationAttribute` in `Validation`) **and** re-check in the Service
  before hashing.
- Plaintext passwords: accepted only in the inbound request DTO; never
  persisted, never logged, never returned, never placed on a response DTO.
- The hash is stored in `password_hash` (see `persistence-conventions.md`
  PC-9) and is never returned by any endpoint.

## SC-2 Roles

- `Customer` — default role on registration.
- `Admin` — administrative operations.
- Role claim values are `Customer` / `Admin` (`ClaimTypes.Role`).
- Default account state on registration: enabled (unless a Story's approved
  design says otherwise).

## SC-3 Authentication

- ASP.NET Core Cookie Authentication (`AddAuthentication().AddCookie(...)`),
  session-based. No ASP.NET Core Identity — sign-in is issued manually via
  `HttpContext.SignInAsync` with a `ClaimsPrincipal` built from the loaded
  account.
- A sign-in service in the `Security` namespace loads the account by email
  and verifies the password with the `IPasswordHasher`.
- Failed authentication returns `401` with the standard error body — it does
  not reveal whether the email exists (no account enumeration) unless a
  Story's approved design explicitly allows it.

## SC-4 Authorization

- Deny by default: a global fallback policy
  (`options.FallbackPolicy = new AuthorizationPolicyBuilder()
  .RequireAuthenticatedUser().Build()`) requires authentication on every
  endpoint unless the approved API design marks it `[AllowAnonymous]` (e.g.
  registration).
- Role checks with `[Authorize(Roles = "...")]` on the Controller action, or
  an authorization policy — stated per endpoint in the API design.
- Ownership checks (a customer may act only on their own resource) are
  enforced in the Service layer, not just by role.

## SC-5 CSRF

- Enabled for browser/session endpoints (the MVP default) via ASP.NET Core
  Antiforgery (`AddAntiforgery()`, validated on state-changing requests).
- Disabling CSRF for a stateless API endpoint requires an approved
  architecture decision recorded for that Story.

## SC-6 Database admin/diagnostic UI

- No database browser or admin UI is exposed by the application in **any**
  profile, including test and local. `dotnet ef` tooling is a
  developer-machine command, never an HTTP endpoint.

## SC-7 Secrets & repository hygiene

- No credentials, tokens, private keys, or `.env` files committed.
- Local secrets use `dotnet user-secrets`; deployed config secrets come from
  environment variables / externalized config.
- Generated SQLite database files are git-ignored (see
  `persistence-conventions.md` PC-1).

## SC-8 Schema safety

- Schema changes happen only through a committed EF Core Migration applied
  via `Database.Migrate()` (see the policy block and
  `persistence-conventions.md` PC-2). `EnsureCreated()` / `EnsureDeleted()`
  against the file database, and raw schema-mutating SQL, are **forbidden**
  — they bypass migrations and can destroy or silently diverge the schema.

## SC-9 Error & log hygiene

- Error responses follow `api-conventions.md` AC-6 and never leak stack
  traces, SQL, entity/class names, filesystem paths, database
  connection strings, or secrets.
- Logs never contain passwords, password hashes, `Authorization`/`Cookie`
  headers, tokens, or full credential-bearing request bodies.
- Tool-usage telemetry (`docs/hooks/tool-usage.jsonl`) records metadata only
  (tool, timestamp, status, sizes), never full sensitive payloads.
