# Persistence Conventions

Explicit persistence decisions for this project. `db-designer` and
`design-reviewer` enforce these; `dotnet-implementor` implements to them;
`implementation-verifier` and `security-reviewer` check against them.

## PC-1 Database & runtime mode

- SQLite, **file-based**, for local training runs. Connection string form:
  `Data Source=./data/customer-portal.db` (project-relative `./data/`
  directory).
- Generated database files (`./data/*.db`, `*.db-shm`, `*.db-wal`) are
  **not** committed — they are covered by `.gitignore`.
- Automated tests use an **isolated in-memory** SQLite connection
  (`Data Source=:memory:`, opened explicitly and kept open for the lifetime
  of the test's `DbContext` — SQLite in-memory databases are destroyed when
  the connection closes), never the file database.
- No database browser/admin UI is exposed through the application in any
  profile (see `security-conventions.md`).

## PC-2 Schema initialization

- Schema is managed **exclusively through EF Core Migrations**
  (`dotnet ef migrations add <Name>`), generated from the model and committed
  under `src/CustomerPortal/Migrations/`. There is no hand-maintained SQL
  schema file.
- Local/dev applies pending migrations at startup via
  `db.Database.Migrate()`. Tests use an isolated in-memory `DbContext` and
  may call `Database.EnsureCreated()` there **only** — `EnsureCreated()` /
  `EnsureDeleted()` are **forbidden** against the file database (they bypass
  migrations and can destroy or silently diverge the schema; see
  `security-conventions.md`).
- Every entity change is accompanied by the matching EF Core migration in the
  same Story. `db-designer` specifies the entity model and the expected
  migration effect; `design-reviewer` checks they agree.
- No raw `ExecuteSqlRaw` schema changes outside a migration.

## PC-3 Identifiers

- Surrogate primary key named `Id`, type `long`, configured
  `ValueGeneratedOnAdd()` (EF Core's default SQLite integer autoincrement
  behavior via `rowid`).
- No business/natural key as a primary key. Natural keys (e.g. email) get a
  unique index instead.
- IDs are not exposed as sequential where enumeration is a concern the Story
  raises — otherwise a `long` id in the API is acceptable for this project.

## PC-4 Explicit column mapping (no EF Core convention defaults)

Every persistent property is configured explicitly via Fluent API in
`DbContext.OnModelCreating` (not left to EF Core convention):

- `.IsRequired()` / nullable reference type matching the column's
  nullability;
- `.HasMaxLength(...)` — required for every `string` property;
- `.HasIndex(...).IsUnique()` for uniqueness;
- `.ToColumn(...)` when the column name differs from the snake_case-mapped
  property name.

`db-designer` states the exact constraints; the entity configuration and the
resulting migration must both match them.

## PC-5 Naming

- Tables: `snake_case`, singular (`customer`, `customer_role`).
- Columns: `snake_case`.
- Constraints: `uq_<table>_<col>` (unique), `fk_<table>_<ref>` (foreign key),
  `ix_<table>_<col>` (index), `pk_<table>` (primary key).
- The `EFCore.NamingConventions` package (`UseSnakeCaseNamingConvention()` on
  `DbContextOptionsBuilder`) is a required dependency so entity property
  `EmailAddress` maps to column `email_address` without per-property
  overrides.

## PC-6 Audit timestamps

- Every entity has `created_at` and `updated_at` columns, stored as UTC
  `DateTime`/`DateTimeOffset` (see `business-rules.md` BR-007).
- Populated by a `SaveChanges` interceptor (`ISaveChangesInterceptor`
  registered on the `DbContext`) that stamps `created_at`/`updated_at` on
  entities in the `Added`/`Modified` `EntityState`, not by hand in Services.
- `created_at` is non-null and not updated after insert; `updated_at` is
  non-null.

## PC-7 Indexes

- Index every foreign key column.
- Index every column used as a lookup key by a repository query (e.g.
  `email` for "find by email"). A unique index already covers uniqueness and
  lookups on that column.
- `db-designer` lists the required indexes; the EF Core migration creates
  them.

## PC-8 Relationships

- Declare cardinality explicitly via Fluent API (`HasOne().WithMany(...)`,
  `HasMany().WithOne(...)`, etc.) alongside navigation properties.
- No lazy-loading proxies package is referenced in this project. Navigation
  properties are loaded explicitly per query via `.Include()` /
  `.ThenInclude()` in the Repository — never implicitly on first access.
- Cascade delete behavior is explicit and minimal —
  `.OnDelete(DeleteBehavior.Restrict)` unless a stated reason justifies
  `DeleteBehavior.Cascade`.

## PC-9 Sensitive data

- Passwords are stored only as a BCrypt hash in a column named
  `password_hash` (`TEXT`/`VARCHAR(60)`, non-null). Plaintext is never
  persisted or logged.
- Other sensitive columns are identified by `db-designer` with their
  handling rules.
