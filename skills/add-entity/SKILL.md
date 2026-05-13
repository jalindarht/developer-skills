---
name: add-entity
description: Create a new domain entity following BaseEntity/IAggregateRoot patterns, with EF Core configuration, DbSet registration, and proper schema assignment
user_invocable: true
---

# Add New Domain Entity

Create a new entity following the project's established architecture. Before generating any code, confirm the following with the user:

## Step 1: Gather Requirements

Ask the user:
1. **Entity name** (PascalCase, singular — e.g., `Coupon`)
2. **Properties** with types (e.g., `string Code`, `decimal Amount`, `bool IsPercentage`)
3. **Is it an aggregate root?** (Can it be queried directly via repository?)
4. **Should it extend BaseEntity?** (includes Id, IsActive, IsDeleted, audit fields) — default YES
5. **Is it a lookup entity?** (extends LookupBaseEntity — adds Name, Description)
6. **Database schema** — one of: `store`, `sale`, `menu`, `management`, `salesforce` (or new schema if justified — confirm with user)
7. **Table name** (default: pluralized entity name, will be auto-snake_cased)
8. **Relationships** — foreign keys, navigation properties
9. **JSON columns** — any complex types stored as JSON?

## Step 2: Validate Architecture Compliance

Before creating, verify:
- [ ] Entity name doesn't conflict with existing entities in `Core/Entities/`
- [ ] If aggregate root: must implement `IAggregateRoot` marker interface
- [ ] If NOT aggregate root: should NOT be queried directly — accessed through parent aggregate
- [ ] Schema choice aligns with domain boundaries (check `Infrastructure/Data/Constants/TableSchemas.cs`)
- [ ] If a new schema is needed, add it to `TableSchemas.cs` constants

## Step 3: Create Files

### 3a. Entity Class
**Location:** `HADV.OMS.Core/Entities/[Feature]/[EntityName].cs` or `HADV.OMS.Core/Entities/[EntityName].cs`

```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;
using HADV.OMS.Core.Shared;

namespace HADV.OMS.Core.Entities;

// Add IAggregateRoot ONLY if this entity is an aggregate root
public class EntityName : BaseEntity, IAggregateRoot
{
    [MaxLength(100)]
    public string PropertyName { get; set; } = string.Empty;

    // For JSON columns:
    [Column(TypeName = "json")]
    public List<ComplexType>? JsonProperty { get; set; }

    // Navigation properties at the bottom:
    public RelatedEntity? Related { get; set; }
}
```

### 3b. EF Core Configuration
**Location:** `HADV.OMS.Infrastructure/Data/Configurations/[EntityName]Config.cs`

For entities extending BaseEntity:
```csharp
using HADV.OMS.Core.Entities;
using HADV.OMS.Infrastructure.Data.Configurations.Base;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace HADV.OMS.Infrastructure.Data.Configurations;

internal class EntityNameConfig : BaseEntityConfiguration<EntityName>
{
    public override void Configure(EntityTypeBuilder<EntityName> builder)
    {
        SqlSchema = Constants.TableSchemas.Sale; // Use appropriate schema
        TableName = "EntityNames"; // Pluralized, will be auto-snake_cased

        // Property configurations
        builder.Property(b => b.PropertyName).HasDefaultValue("default");

        // Relationships
        builder.HasOne(b => b.Related)
            .WithMany()
            .HasForeignKey(b => b.RelatedId)
            .OnDelete(DeleteBehavior.Restrict);

        // MUST call base.Configure LAST — applies audit fields, soft delete filter
        base.Configure(builder);
    }
}
```

For entities NOT extending BaseEntity (like Cart):
```csharp
internal class EntityNameConfig : IEntityTypeConfiguration<EntityName>
{
    public void Configure(EntityTypeBuilder<EntityName> builder)
    {
        builder.ToTable("entity_names", Constants.TableSchemas.Sale);
        // Add HasQueryFilter if soft delete is needed manually
    }
}
```

### 3c. Register DbSet
**File:** `HADV.OMS.Infrastructure/Data/AppDbContext.cs`

Add: `public DbSet<EntityName> EntityNames { get; set; }`

## Step 4: Post-Creation Checklist

- [ ] Entity follows BaseEntity audit pattern (unless exempted)
- [ ] EF configuration uses correct schema from `TableSchemas`
- [ ] `base.Configure(builder)` called LAST in configuration
- [ ] DbSet added to AppDbContext
- [ ] If aggregate root: IAggregateRoot implemented
- [ ] If new schema needed: added to `TableSchemas.cs`
- [ ] String properties have `[MaxLength]` annotations
- [ ] Navigation properties are nullable
- [ ] No business logic in entity class (entities are anemic/POCO)

## Architecture Alerts

Flag to user if:
- Entity has business methods (violates anemic domain model pattern used here)
- Entity doesn't fit any existing schema (might indicate a new bounded context)
- Entity has circular references
- Entity has too many properties (suggest splitting into value objects or related entities)
