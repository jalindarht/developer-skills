---
name: add-specification
description: Create a new Ardalis.Specification query class with proper filtering, includes, ordering, and optional DTO projection
user_invocable: true
---

# Add New Specification

Create a new query specification using the Ardalis.Specification pattern.

## Step 1: Gather Requirements

Ask the user:
1. **Aggregate root entity** being queried (must implement `IAggregateRoot`)
2. **Specification name** (descriptive — e.g., `GetByLocationId`, `GetActiveProducts`)
3. **Filter criteria** (Where clauses)
4. **Includes** (navigation properties to load eagerly)
5. **Ordering** (OrderBy/OrderByDescending)
6. **Projection** — should it project to a DTO? (use `Specification<TEntity, TDto>`)
7. **Pagination** — Skip/Take needed?

## Step 2: Create File

**Location:** `HADV.OMS.Core/Entities/Specifications/[AggregateRootName]/[SpecName].cs`

### Simple Query Specification (returns entity):

```csharp
using Ardalis.Specification;

namespace HADV.OMS.Core.Entities.Specifications.[AggregateRoot];

internal class GetByCondition : Specification<Entities.[AggregateRoot]>
{
    public GetByCondition(int paramId)
    {
        Query
            .Include(e => e.NavigationProperty)
            .Where(e => e.ForeignKeyId == paramId && e.IsActive);
    }
}
```

### Projection Specification (returns DTO):

```csharp
using Ardalis.Specification;

namespace HADV.OMS.Core.Entities.Specifications.[AggregateRoot];

internal class GetDetailById : Specification<Entities.[AggregateRoot], DTO.[DtoName]>
{
    public GetDetailById(int id)
    {
        Query
            .Where(e => e.Id == id)
            .Select(e => new DTO.[DtoName]
            {
                Id = e.Id,
                Name = e.Name,
                // Map all needed properties
                RelatedItems = e.Children
                    .Where(c => !c.IsDeleted)
                    .OrderBy(c => c.DisplayOrder)
                    .Select(c => new DTO.ChildDto
                    {
                        Id = c.Id,
                        Name = c.Name
                    }).ToList()
            });
    }
}
```

### SingleResultSpecification (when expecting exactly one result):

```csharp
internal class GetByUserId : SingleResultSpecification<Entities.Cart>
{
    public GetByUserId(string userId)
    {
        Query
            .Include(c => c.Items)
            .Where(c => c.UserId == userId);
    }
}
```

### With Pagination:

```csharp
internal class GetPaged : Specification<Entities.[AggregateRoot]>
{
    public GetPaged(int skip, int take)
    {
        Query
            .Where(e => e.IsActive)
            .OrderByDescending(e => e.CreatedOn)
            .Skip(skip)
            .Take(take);
    }
}
```

## Step 3: Usage in Service

```csharp
// Single result
var entity = await _repo.FirstOrDefaultAsync(
    new Specs.AggregateRoot.GetByCondition(id), cancellationToken);

// List result
var entities = await _repo.ListAsync(
    new Specs.AggregateRoot.GetPaged(skip, take), cancellationToken);

// Count
var count = await _repo.CountAsync(
    new Specs.AggregateRoot.GetByCondition(id), cancellationToken);
```

## Post-Creation Checklist

- [ ] Class is `internal`
- [ ] Located in `Specifications/[AggregateRootName]/` folder
- [ ] Entity type is an aggregate root (implements `IAggregateRoot`)
- [ ] Uses `Query` builder in constructor
- [ ] Projection selects only needed fields (avoids over-fetching)
- [ ] Nested collections filter `!IsDeleted` if accessing entities with soft delete
- [ ] No business logic in specification (query only)

## Architecture Alerts

Flag to user if:
- Querying a non-aggregate-root entity directly (must go through aggregate root)
- Specification contains business logic beyond filtering
- Over-fetching: includes navigation properties that aren't needed
- Missing IsDeleted filter on nested child collections in projections
