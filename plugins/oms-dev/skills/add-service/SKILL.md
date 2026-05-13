---
name: add-service
description: Create a new service with interface and implementation following Result pattern, Guard clauses, repository/specification usage, and DI registration
user_invocable: true
---

# Add New Service

Create a new service following the project's interface + implementation pattern with Ardalis.Result returns.

## Step 1: Gather Requirements

Ask the user:
1. **Service name** (e.g., `CouponService` — I prefix auto-added: `ICouponService`)
2. **Feature area** (determines folder: `Core/Services/[Feature]/`)
3. **Methods** with signatures (params, return type)
4. **Dependencies** — which repositories, other services does it need?
5. **Layer** — Core (business logic) or Infrastructure (external integrations/data access)?

## Step 2: Validate Architecture Compliance

Before creating, verify:
- [ ] Service doesn't duplicate existing functionality (check `CoreExtensions.cs` registrations)
- [ ] Repository dependencies use `IRepository<T>` where T is an aggregate root
- [ ] If infrastructure-dependent (DB direct access, external APIs): place in Infrastructure layer
- [ ] If pure business logic: place in Core layer

## Step 3: Create Files

### 3a. Service Interface + Implementation (Core Layer)

**File:** `HADV.OMS.Core/Services/[Feature]/[ServiceName].cs`

Both interface and implementation go in the SAME file (project convention):

```csharp
using Ardalis.GuardClauses;
using Ardalis.Result;
using HADV.OMS.Core.Shared;

namespace HADV.OMS.Core.Services.[Feature];

public interface IServiceName
{
    Task<Result<ResponseDto>> GetByIdAsync(int id, CancellationToken cancellationToken = default);
    Task<Result<bool>> CreateAsync(CreateDto dto, CancellationToken cancellationToken = default);
    Task<Result<bool>> UpdateAsync(int id, UpdateDto dto, CancellationToken cancellationToken = default);
    Task<Result<bool>> DeleteAsync(int id, CancellationToken cancellationToken = default);
}

internal class ServiceName : IServiceName
{
    readonly IRepository<Entities.AggregateRoot> _entityRepo;
    readonly IOtherService _otherService;

    public ServiceName(
        IRepository<Entities.AggregateRoot> entityRepo,
        IOtherService otherService)
    {
        _entityRepo = entityRepo;
        _otherService = otherService;
    }

    public async Task<Result<ResponseDto>> GetByIdAsync(int id, CancellationToken cancellationToken = default)
    {
        Guard.Against.NegativeOrZero(id);

        var entity = await _entityRepo.FirstOrDefaultAsync(
            new Entities.Specifications.AggregateRoot.GetById(id),
            cancellationToken);

        if (entity is null)
            return Result.NotFound();

        return Result.Success(entity);
    }

    public async Task<Result<bool>> CreateAsync(CreateDto dto, CancellationToken cancellationToken = default)
    {
        Guard.Against.Null(dto);

        var entity = new Entities.EntityName
        {
            // Map from DTO to entity
            PropertyName = dto.PropertyName,
            CreatedBy = "system", // or from session
            CreatedOn = DateTimeOffset.UtcNow
        };

        await _entityRepo.AddAsync(entity, cancellationToken);
        await _entityRepo.SaveChangesAsync(cancellationToken);

        return Result.Success(true);
    }

    public async Task<Result<bool>> UpdateAsync(int id, UpdateDto dto, CancellationToken cancellationToken = default)
    {
        Guard.Against.NegativeOrZero(id);

        var entity = await _entityRepo.GetByIdAsync(id, cancellationToken);
        if (entity is null)
            return Result.NotFound();

        // Update properties
        entity.PropertyName = dto.PropertyName;
        entity.ModifiedBy = "system";
        entity.ModifiedOn = DateTimeOffset.UtcNow;

        await _entityRepo.UpdateAsync(entity, cancellationToken);
        await _entityRepo.SaveChangesAsync(cancellationToken);

        return Result.Success(true);
    }

    public async Task<Result<bool>> DeleteAsync(int id, CancellationToken cancellationToken = default)
    {
        Guard.Against.NegativeOrZero(id);

        var entity = await _entityRepo.GetByIdAsync(id, cancellationToken);
        if (entity is null)
            return Result.NotFound();

        // Soft delete (project convention)
        entity.IsDeleted = true;
        entity.ModifiedOn = DateTimeOffset.UtcNow;

        await _entityRepo.UpdateAsync(entity, cancellationToken);
        await _entityRepo.SaveChangesAsync(cancellationToken);

        return Result.Success(true);
    }
}
```

### 3b. Infrastructure Service (if needed)

**File:** `HADV.OMS.Infrastructure/Services/[ServiceName].cs`

```csharp
namespace HADV.OMS.Infrastructure.Services;

internal class ServiceName : IServiceName
{
    readonly AppDbContext _dbContext;

    public ServiceName(AppDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    // Implementation using direct DbContext or external APIs
}
```

### 3c. Register in DI Container

**Core services** → `HADV.OMS.Core/CoreExtensions.cs`:
```csharp
services.AddTransient<IServiceName, ServiceName>();
```

**Infrastructure services** → `HADV.OMS.Infrastructure/InfrastructureExtensions.cs`:
```csharp
services.AddTransient<IServiceName, ServiceName>();
```

## Step 4: Post-Creation Checklist

- [ ] Interface is `public`, implementation is `internal`
- [ ] All methods return `Result<T>` or `Result`
- [ ] Guard clauses used for input validation
- [ ] CancellationToken parameter on all async methods
- [ ] Registered as `AddTransient` in correct extensions file
- [ ] Dependencies are injected via constructor (readonly fields)
- [ ] No HTTP/API concerns in service (no HttpContext, ActionResult, etc.)
- [ ] Specifications used for complex queries (not inline LINQ on repo)
- [ ] Soft delete used instead of hard delete

## Architecture Alerts

Flag to user if:
- Service has HTTP/request dependencies (should be in endpoint layer)
- Service directly uses DbContext when it could use IRepository (prefer repository)
- Service has too many dependencies (> 5 — suggest splitting)
- Service returns entities instead of DTOs to callers
- Missing DI registration
- Using Scoped lifetime (should be Transient unless stateful like SessionService)
