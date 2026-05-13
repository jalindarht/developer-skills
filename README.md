# oms-dev — Claude Code Plugin

Claude Code skills for **HADV.OMS.Services** — a .NET 8 Clean Architecture backend using Ardalis libraries (ApiEndpoints, Result, GuardClauses, Specification) with Entity Framework Core and PostgreSQL.

## Installation

```
/plugin install github.com/stratis-jalindar/developer-skills
```

Once installed, all skills are available under the `oms-dev` namespace.

## Skills

### Review & Analysis

| Skill | Invoke | Description |
|-------|--------|-------------|
| Code Review | `/oms-dev:code-review` | 8-pass review covering architecture, logic, security, error handling, performance, duplication, naming, and completeness |
| Architecture Review | `/oms-dev:architecture-review` | Validates layer boundaries, naming conventions, DI registration, and pattern compliance |
| .NET Performance | `/oms-dev:analyzing-dotnet-performance` | Scans for ~50 anti-patterns across async, memory, strings, collections, LINQ, regex, and I/O |
| Debug & Fix | `/oms-dev:debug-and-fix` | Traces errors top-down through all architecture layers and applies fixes after approval |

### Feature Scaffolding

| Skill | Invoke | Description |
|-------|--------|-------------|
| Add Feature | `/oms-dev:add-feature` | Full end-to-end feature scaffold — entity → DTO → spec → service → EF config → endpoints |
| Add Entity | `/oms-dev:add-entity` | Domain entity with BaseEntity/IAggregateRoot, EF config, and DbSet registration |
| Add DTO | `/oms-dev:add-dto` | Response DTO in `Core/DTO/` following POCO conventions |
| Add Service | `/oms-dev:add-service` | Service interface + implementation with Result pattern, Guard clauses, and DI registration |
| Add Specification | `/oms-dev:add-specification` | Ardalis.Specification query class with filtering, includes, ordering, or DTO projection |
| Add Endpoint | `/oms-dev:add-endpoint` | Ardalis.ApiEndpoints endpoint with request class, route constant, and Swagger annotations |
| Add EF Configuration | `/oms-dev:add-ef-configuration` | EF Core entity configuration using BaseEntityConfiguration with soft-delete and schema setup |
| Add Enum | `/oms-dev:add-enum` | Domain enum or error code enum with explicit int values and correct folder placement |
| Add Background Service | `/oms-dev:add-background-service` | Hosted BackgroundService with IServiceScopeFactory and worker/API mode registration |

## Project Architecture

**HADV.OMS.Services** is a .NET 8 Clean Architecture project with three layers:

```
HADV.OMS.Core           — Entities, DTOs, Services (interfaces), Specifications, Error Codes
HADV.OMS.Infrastructure — EF Core configs, Repositories, External API clients
HADV.OMS.CentralAPI     — Endpoints (Ardalis.ApiEndpoints), Route constants, Program.cs
```

Key patterns enforced by these skills:
- No MVC controllers — `Ardalis.ApiEndpoints` only
- Services return `Result<T>` / `Result` — never throw (except Guard failures)
- Queries use `Ardalis.Specification` — no inline LINQ in services
- `Guard.Against.*` for input validation at service boundaries
- Soft delete via `IsDeleted` global query filter
- `DeleteBehavior.Restrict` on all relationships
- `base.Configure(builder)` called last in EF configurations
- `IServiceScopeFactory` in all background services

## Local Development

Test the plugin locally without installing:

```
claude --plugin-dir "path/to/developer-skills"
```
