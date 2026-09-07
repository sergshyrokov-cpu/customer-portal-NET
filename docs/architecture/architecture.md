# Architecture

Explicit architecture decisions for the Customer Portal training project. These
are project decisions, not general framework advice. Skills
(`impact-analyzer`, `implementation-planner`, `dotnet-implementor`,
`design-reviewer`, `implementation-verifier`, `security-reviewer`,
`reconciliation-reviewer`) treat this file as authoritative.

## AD-1 Solution & project layout

- Single ASP.NET Core Web API project (`CustomerPortal`), one solution file
  `CustomerPortal.sln`. No multi-project split in this project.
- Production code under `src/CustomerPortal/`, root namespace
  `CustomerPortal`.
- Test code lives in `tests/CustomerPortal.Tests/`, namespace
  `CustomerPortal.Tests`, mirroring the production namespace tree.
- No new solution projects without an approved decision.

## AD-2 Layered architecture

```
Controller → Service → Repository → Database
```

| Layer | Namespace/folder | Responsibility | Must not |
|---|---|---|---|
| Controller | `Controllers` | HTTP mapping, request/response DTO binding, delegate to a Service, map Service outcomes to HTTP status | contain business rules; call a Repository; return an entity |
| Service | `Services` | all business logic, orchestration, transaction boundaries, mapping between entities and DTOs | depend on `HttpContext` / ASP.NET Core MVC types; call another Controller |
| Repository | `Repositories` | EF Core `DbContext` access, persistence queries only | contain business logic; call a Service |
| Entity | `Models/Entities` | persisted domain state | be serialized as an API request/response |

Allowed dependency directions: `Controller → Service → Repository`. Everything
else in that set is forbidden (`Controller → Repository`,
`Controller → Models/Entities` as an API type, `Repository → Service`,
`Repository → Controller`, `Service → Controller`).

## AD-3 Transaction boundary policy

- Transactions begin and end in the **Service** layer.
- Write operations: the Service calls `DbContext.SaveChangesAsync()` after all
  repository calls for that use case complete. A Service method that must make
  several writes atomically wraps them in an explicit
  `IDbContextTransaction` (`Database.BeginTransactionAsync()` /
  `CommitAsync()` / `RollbackAsync()`).
- Repositories never call `SaveChangesAsync()` — they stage changes on the
  tracked entity/`DbSet` and return; the owning Service commits.
- Read-only query methods use `AsNoTracking()` by default; wrap multiple
  repository calls in an explicit transaction only when a consistent snapshot
  across calls is required.
- Controllers and Repositories must not open a `DbContext` transaction or call
  `SaveChangesAsync()`.

## AD-4 DTO / entity boundary

- Every API request body binds to a class in `Models/Requests`.
- Every API response body is a class in `Models/Dtos`.
- Entities (`Models/Entities`) never appear in a Controller signature, a
  request body, or a response body.
- Mapping entity ↔ DTO/request happens in the Service layer (a dedicated
  mapper class or extension method is allowed; a mapping library such as
  AutoMapper is not added without an approved decision).
- A response DTO includes only fields the API contract lists. Credential
  fields (password, password hash) are never present on a response DTO, even
  as `null`.

## AD-5 Validation boundary

- Request-shape validation (required, length, format, allowed values): Data
  Annotation attributes (`[Required]`, `[StringLength]`, `[EmailAddress]`,
  etc.) on the `Models/Requests` class. `[ApiController]` on every controller
  makes ASP.NET Core validate `ModelState` and return `400` automatically —
  Controllers never check `ModelState.IsValid` by hand.
- Business-rule validation (uniqueness, cross-field rules, state checks): in
  the Service layer, before persistence.
- Custom validation attributes (subclassing `ValidationAttribute`, or
  `IValidatableObject` implementations) live in the `Validation` namespace.
- Validation is server-side and independent of any client.

## AD-6 Exception handling architecture

- One `IExceptionHandler` implementation in the `Exceptions` namespace,
  registered via `AddExceptionHandler<T>()` / `UseExceptionHandler()`, is the
  single place that maps exceptions to HTTP responses.
- Domain/application exceptions are declared in the `Exceptions` namespace
  (e.g. `DuplicateEmailException`, `ResourceNotFoundException`). Services
  throw these; they carry no HTTP concepts.
- The handler maps: model-validation failure → 400 (handled by
  `[ApiController]` before a Controller runs); domain "not found" → 404;
  domain "conflict/duplicate" → 409; authn failure → 401; authz failure →
  403; anything unmapped → 500.
- Every error response body uses the structure defined in
  `api-conventions.md` (§ Errors). Stack traces, SQL, entity/class names,
  database paths, and secrets are never in a response body.

## AD-7 Configuration boundaries

- Framework/infrastructure startup wiring lives in `Program.cs` and
  `IServiceCollection` extension methods under `Configuration`
  (e.g. `AddPersistence()`, `AddApiVersioning()`).
- Security configuration lives in `Security` (see `security-conventions.md`).
- No business logic in a `Configuration` extension method.
- Application settings come from `appsettings.json` / `appsettings.
  {Environment}.json` / environment variables, never hard-coded; secrets
  never committed (see `security-conventions.md`).

## AD-8 Reuse over duplication

Before creating a component, check for an existing one that can be extended
within these rules. New namespaces/folders beyond the map in
`package-map.md` require an approved decision (an Open Decision resolved by a
human, not a silent addition).
