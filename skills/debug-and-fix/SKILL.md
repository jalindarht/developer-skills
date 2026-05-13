---
name: debug-and-fix
description: Debug and fix issues in one flow — diagnoses errors by tracing through architecture layers (endpoint→service→spec→DB→external APIs), presents a fix plan for approval, then applies the fix following all project conventions
user_invocable: true
---

# Debug & Fix

One-command workflow: investigate the issue, find root cause, present a fix plan, and apply after approval.

---

## PHASE 1: GATHER CONTEXT

Ask the user for whatever they have:
- Error message, exception, HTTP status code, or unexpected behavior
- Where it occurs (endpoint route, service, background job, entity)
- Error codes (format: `"EnumName.Value"` e.g., `"ServerErrors.RecordNotFound"`)
- Stack trace or logs (if available)
- Steps to reproduce (if known)

---

## PHASE 2: DIAGNOSE — Trace Through Architecture Layers

Systematically trace top-down. Read the actual source files at each layer. Do NOT guess.

### Layer 1: API Endpoint (`CentralAPI/Endpoints/[Feature]/`)
- Read the endpoint's `HandleAsync` method
- Check request binding (`[FromRoute]`, `[FromBody]`, `[FromQuery]`)
- Check route constant in `Constants/ApiEndPoints.cs`
- Verify `this.ToActionResult(result)` is used
- Check `[Authorize]` attribute

### Layer 2: Request Validation
- Check `[Required]`, `[MaxLength]`, `[Range]` on request properties
- Data annotations return 400 before the handler runs
- Guard clause failures throw `ArgumentException`

### Layer 3: Service Layer (`Core/Services/[Feature]/`)
- Read the service method being called
- Check all `Result<T>` return paths — every branch must return a Result
- Check try/catch blocks — watch for **silent catches** (`catch { }`) that swallow errors
- Verify Guard clauses match expected input
- Check CancellationToken is passed through
- Verify DI registration in `CoreExtensions.cs` or `InfrastructureExtensions.cs`

### Layer 4: Specification / Repository (`Core/Entities/Specifications/`)
- Read the specification class
- Check `.Where()` filters — too aggressive?
- Check `.Include()` — missing includes cause null navigation properties
- Global `HasQueryFilter(p => !p.IsDeleted)` is ALWAYS active (need `IgnoreQueryFilters()` for soft-deleted)
- Only aggregate roots (`IAggregateRoot`) can be queried via `IRepository<T>`

### Layer 5: Database / EF Core (`Infrastructure/Data/`)
- Read entity configuration in `Configurations/[Entity]Config.cs`
- Verify schema from `Constants.TableSchemas.*`
- Check `base.Configure(builder)` called LAST
- Check relationships, default values, JSON columns

### Layer 6: External Services
- **Payment (CardPointe):** `Infrastructure/Fiserv/CardPointeGatewayClient.cs` — errors in `ApiResponse` with `RequestJson`/`ResponseJson`
- **Email (SendGrid):** `Core/Services/Email/EmailService.cs` — inconsistent error handling (some fire-and-forget)
- **Auth0:** middleware + `Program.cs` config
- **Azure Blob:** `Core/Services/AzureBlobStorage/BlobStorageService.cs`

### Quick Reference: Error Code Lookup

| Error Code | Meaning | Typical Cause |
|-----------|---------|---------------|
| `ServerErrors.Exception` | Unhandled exception | Generic catch block |
| `ServerErrors.RecordNotFound` | Entity missing | Wrong ID, soft-deleted, bad spec filter |
| `PaymentErrors.ExternalApiError` | Payment gateway error | CardPointe API rejection |
| `PaymentErrors.ExternalApiHttpError` | HTTP error to gateway | Network/gateway down |
| `PaymentErrors.InvalidStatus` | Wrong payment state | Cancel on completed payment |
| `PaymentErrors.ReferenceIdMissing` | No external ref ID | Not yet authorized |

### Quick Reference: Result → HTTP Status

| Result | HTTP | Source |
|--------|------|--------|
| `Result.Success()` | 200 | Service success |
| `Result.NotFound()` | 404 | Entity not found |
| `Result.Error()` | 422/400 | Business logic failure |
| `Result.Invalid()` | 400 | Input validation |
| `Result.Forbidden()` | 403 | Auth failure |
| `Result.Unauthorized()` | 401 | No/bad token |

### Quick Reference: Common Exceptions

| Exception | Likely Cause |
|-----------|-------------|
| `NullReferenceException` | Missing `.Include()` in spec, null DTO mapping |
| `ArgumentException` | Guard clause failed |
| `InvalidOperationException` | DI resolution fail, empty sequence |
| `DbUpdateException` | DB constraint violation |
| `NpgsqlException` | PostgreSQL error, schema mismatch |
| `HttpRequestException` | External API unreachable |
| `JsonException` | JSON column/API parsing failure |

### Common Issue Patterns

**"Record Not Found" but it exists:**
→ Check `IsDeleted = true` (soft delete filter), `IsActive` flag, specification `.Where()` conditions

**Null Reference Exception:**
→ Missing `.Include()` in specification, entity loaded without eager loading, DTO mapping on null nav property

**DI / Service Resolution Error:**
→ Not registered in CoreExtensions/InfrastructureExtensions, interface mismatch, Scoped in Singleton, background service missing `IServiceScopeFactory`

**500 Internal Server Error:**
→ `ExceptionHandler` middleware is minimal (logs + rethrows), unhandled exception in service, DB connection failure, EF migration mismatch

**Payment Failures:**
→ Check `PaymentActivity` records for API JSON, verify `PaymentGatewayConfig` in appsettings, check payment status flow

**Background Job Not Running:**
→ Check worker vs API mode registration in `Program.cs`, `IServiceScopeFactory` scope, unhandled exceptions killing loop

---

## PHASE 3: PRESENT FIX PLAN

**MANDATORY: Present this plan and WAIT for user approval before making any code changes.**

Format the plan as:

```
============================================
DEBUG & FIX PLAN
============================================

ISSUE:
  [What the user reported]

ROOT CAUSE:
  [What's actually wrong — file:line reference]

DATA FLOW:
  [endpoint] → [service method] → [spec/repo] → [failure point]

AFFECTED FILES:
  1. [file path] — [what changes]
  2. [file path] — [what changes]

FIX APPROACH:
  [Step-by-step description of the code changes]

ARCHITECTURE COMPLIANCE:
  ✓ Layer boundaries respected
  ✓ Result pattern used correctly
  ✓ Error handling follows conventions
  ✓ No new anti-patterns introduced

RISK ASSESSMENT:
  [What else could be affected, breaking changes]

RELATED ISSUES FOUND (optional):
  - [Other problems noticed during investigation]
============================================

Proceed with this fix? (yes/no)
```

**DO NOT write any code until the user approves.**

---

## PHASE 4: APPLY FIX (After Approval Only)

Apply the fix following these architecture rules:

### Service Layer Fixes
```csharp
// Error returns — use Result pattern, NEVER throw
if (entity is null)
    return Result.NotFound();

return Result.Error(ServerErrors.RecordNotFound.ToErrorCode(), "Entity not found");

// Catch blocks — NEVER swallow silently
catch (Exception ex)
{
    _logger.LogError(ex, "Failed to process {Operation} for {EntityId}", operation, id);
    return Result.Error(ServerErrors.Exception.ToErrorCode(), "Operation failed");
}

// Guard clauses for input validation
Guard.Against.NegativeOrZero(id);
Guard.Against.Null(request, nameof(request));
```

### Specification Fixes
```csharp
// Add missing Include
Query
    .Include(e => e.NavigationProperty)
    .ThenInclude(n => n.Child)
    .Where(e => e.Id == id);

// Filter soft-deleted children in projections
.Select(e => new Dto
{
    Children = e.Children
        .Where(c => !c.IsDeleted && c.IsActive)
        .OrderBy(c => c.DisplayOrder)
        .Select(c => new ChildDto { ... })
        .ToList()
})
```

### Endpoint Fixes
```csharp
// ALWAYS use Result conversion
return this.ToActionResult(result);

// NEVER bypass with manual status codes
// return Ok(result.Value);      ← WRONG
// return BadRequest("error");   ← WRONG
```

### Entity / Configuration Fixes
```csharp
// Relationships use Restrict (project convention)
builder.HasOne(b => b.Parent)
    .WithMany(p => p.Children)
    .HasForeignKey(b => b.ParentId)
    .OnDelete(DeleteBehavior.Restrict);

// base.Configure() called LAST
base.Configure(builder);
```

### DI Registration Fixes
```csharp
// Core → CoreExtensions.cs
services.AddTransient<IService, Service>();

// Infrastructure → InfrastructureExtensions.cs
services.AddTransient<IInfraService, InfraService>();

// Lifetimes: Transient (business), Scoped (session/repo), Singleton (HttpClient/Blob)
```

### Background Service Fixes
```csharp
// MUST use IServiceScopeFactory
using var scope = _scopeFactory.CreateScope();
var service = scope.ServiceProvider.GetRequiredService<IService>();
```

### Null-Safe DTO Mapping
```csharp
dto.RelatedName = entity.Related?.Name ?? string.Empty;
dto.Items = entity.Items?.Select(i => new ItemDto { ... }).ToList();
```

---

## PHASE 5: POST-FIX VALIDATION

After applying the fix, verify:

- [ ] **Architecture compliance** — layer boundaries intact
- [ ] **No new anti-patterns** — no silent catches, no business logic in endpoints, no raw entities in responses
- [ ] **Consistent** — follows same patterns as neighboring code
- [ ] **Error handling** — all paths return Result with meaningful codes
- [ ] **Logging** — errors logged with structured parameters (`{ParamName}` format)
- [ ] **Side effects** — callers of modified methods still work correctly
- [ ] **DI** — any new/changed services registered correctly

Report the validation results to the user.

---

## PHASE 6: REPORT RELATED ISSUES (Optional)

If during investigation you found other issues, report them:

> "While fixing [issue], I also found these related problems:"
> 1. [file:line] — [description of issue]
> 2. [file:line] — [description of issue]
>
> "Want me to fix any of these?"

### Known Codebase Issues to Watch For
- `ExceptionHandler` middleware is incomplete — logs to Console and rethrows (TODO in code)
- Some services have silent `catch { }` blocks (e.g., `ClearOrdersAsync`)
- Email sending error handling is inconsistent
- No structured logging (Serilog) configured
- No health check endpoints
