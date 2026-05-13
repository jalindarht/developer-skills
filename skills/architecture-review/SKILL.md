---
name: architecture-review
description: Review code changes or files for architecture compliance — validates patterns, naming conventions, layer boundaries, DI registration, and best practices
user_invocable: true
---

# Architecture Review

Review code for compliance with the project's established architecture and best practices. Run this before committing or when reviewing existing code.

## What to Review

When invoked, analyze the specified files or recent changes and check against ALL of the following rules:

### Layer Boundaries
- [ ] **Entities** are in `Core/Entities/` — NO business logic in entities (anemic domain model)
- [ ] **DTOs** are in `Core/DTO/` — simple POCOs, no base class
- [ ] **Service interfaces** are `public` in `Core/Services/`
- [ ] **Service implementations** are `internal` in `Core/Services/` or `Infrastructure/Services/`
- [ ] **Endpoints** are in `CentralAPI/Endpoints/[Feature]/` — NO business logic, delegates to services
- [ ] **EF Configurations** are in `Infrastructure/Data/Configurations/`
- [ ] **Specifications** are in `Core/Entities/Specifications/[AggregateRoot]/`
- [ ] Core layer has NO dependency on Infrastructure or CentralAPI
- [ ] Infrastructure depends ONLY on Core
- [ ] CentralAPI depends on Core and Infrastructure

### Naming Conventions
- [ ] File-scoped namespaces (`namespace X;` not `namespace X { }`)
- [ ] PascalCase for classes, methods, properties
- [ ] camelCase for local variables and parameters
- [ ] `_camelCase` for private readonly fields
- [ ] `I` prefix for interfaces (`IServiceName`)
- [ ] Route constants: kebab-case (`/product-management/category-definition/create`)
- [ ] Table names: PascalCase plural in config (`"EntityNames"` → auto snake_cased)
- [ ] Endpoint request files: `[EndpointName].Request.cs`

### Pattern Compliance
- [ ] Services return `Result<T>` or `Result` (Ardalis.Result)
- [ ] Services use `Guard.Against.*` for validation (Ardalis.GuardClauses)
- [ ] Specifications used for complex queries (not inline LINQ)
- [ ] Only aggregate roots (implementing `IAggregateRoot`) queried via `IRepository<T>`
- [ ] Endpoints use `this.ToActionResult(result)` for response conversion
- [ ] Endpoints have `[SwaggerOperation]` with Summary, Tags, OperationId
- [ ] `[Authorize]` on all non-public endpoints
- [ ] BaseEntity used for all entities requiring audit/soft-delete
- [ ] `base.Configure(builder)` called LAST in BaseEntityConfiguration subclasses

### DI Registration
- [ ] New services registered in `CoreExtensions.cs` or `InfrastructureExtensions.cs`
- [ ] Business services registered as `Transient`
- [ ] Repository registered as `Scoped` (already done generically)
- [ ] SessionService registered as `Scoped`
- [ ] No missing registrations

### Database
- [ ] EF configurations use `Constants.TableSchemas.*` for schema
- [ ] Relationships use `DeleteBehavior.Restrict` (not Cascade)
- [ ] Soft delete pattern: `IsDeleted` flag with global query filter
- [ ] New DbSets added to `AppDbContext`
- [ ] String properties have `[MaxLength]` or `.HasMaxLength()`
- [ ] JSON columns use `[Column(TypeName = "json")]`

### Best Practices
- [ ] Async methods have `CancellationToken` parameter
- [ ] No `async void` methods
- [ ] No catching generic `Exception` without logging/re-throwing
- [ ] No hardcoded connection strings or secrets
- [ ] No entity objects returned from endpoints (use DTOs)
- [ ] Explicit types preferred over `var`
- [ ] Background services use `IServiceScopeFactory` for scoped dependencies

## Output Format

For each issue found, report:
1. **File and line** where the issue exists
2. **Rule violated** (which convention/pattern)
3. **Current code** (the problematic snippet)
4. **Recommended fix** (what it should be)
5. **Severity**: Error (breaks architecture), Warning (deviates from convention), Info (improvement suggestion)

Ask the user if they want the issues auto-fixed or just reported.

## When to Ask for Confirmation

If during review you discover the application is NOT following a standard best practice that SHOULD be adopted (e.g., missing validation, inconsistent patterns, security issues), explicitly flag it and ask:

> "This code/feature is not following [standard/best practice]. Should I update it to follow [recommended approach]? Here's what would change: [brief description]"

Never silently "fix" architectural deviations — always confirm with the user first.
