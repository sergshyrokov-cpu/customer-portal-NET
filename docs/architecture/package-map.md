# Package Map

Root namespace: `CustomerPortal`. Test namespaces mirror this tree under
`tests/CustomerPortal.Tests/`. Adding a namespace/folder not listed here
requires an approved decision.

| Namespace/folder | Contains | Depends on (allowed) | Notes |
|---|---|---|---|
| `Controllers` | ASP.NET Core MVC controller classes | `Services`, `Models.Dtos`, `Models.Requests` | No business logic. No repository access. No entity in a signature. |
| `Services` | business logic, orchestration, entity↔DTO mapping, transaction boundaries | `Repositories`, `Models.Entities`, `Models.Dtos`, `Models.Requests`, `Exceptions`, `Validation`, `Security` (read-only helpers) | Owns `SaveChangesAsync()`. No MVC / `HttpContext` types. |
| `Repositories` | EF Core `DbContext`-backed repository classes/interfaces | `Models.Entities` | Queries and staged writes only. No `SaveChangesAsync()`. No business logic. |
| `Models.Entities` | EF Core entity classes — persisted domain state | (none — leaf) | Never used as an API request/response type. |
| `Models.Dtos` | API **response** DTOs | (none — leaf) | No credential fields, ever. |
| `Models.Requests` | API **request** DTOs with Data Annotation attributes | `Validation` | Validated automatically by `[ApiController]`. |
| `Validation` | custom `ValidationAttribute` / `IValidatableObject` implementations | `Models.Entities` (read-only, when a validator must query) | |
| `Security` | cookie authentication setup, sign-in service, password hasher, authorization policies | `Repositories`, `Models.Entities`, `Configuration` | See `security-conventions.md`. |
| `Configuration` | `IServiceCollection` / `WebApplicationBuilder` extension methods (persistence, Swagger, etc.) | framework only | No business logic. |
| `Exceptions` | domain exception classes + one `IExceptionHandler` | `Models.Dtos` (error body) | Single place that maps exceptions → HTTP. |

## Dependency direction rules

- `Controllers` may depend on `Services`, `Models.Dtos`, `Models.Requests` —
  nothing else in the app.
- `Services` may depend on everything except `Controllers`.
- `Repositories` may depend only on `Models.Entities`.
- `Models.Entities`, `Models.Dtos` are leaves (no intra-app dependencies).
- No cycles. `Repositories → Services`, `Repositories → Controllers`,
  `Services → Controllers`, `Controllers → Repositories` are all forbidden and
  are architecture violations (Major or Critical finding depending on
  impact).

## Test namespace rule

For a production class `CustomerPortal.<Namespace>.<Name>`, its tests live in
`CustomerPortal.Tests.<Namespace>` under `tests/CustomerPortal.Tests/`.
Integration tests that span layers may sit in a `...<Feature>` namespace but
still under the test root namespace.
