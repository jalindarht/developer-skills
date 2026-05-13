---
name: add-ef-configuration
description: Create a new EF Core entity configuration using BaseEntityConfiguration base class with proper schema, table naming, relationships, and soft delete filters
user_invocable: true
---

# Add New EF Core Entity Configuration

Create a new entity type configuration following the project's BaseEntityConfiguration pattern.

## Step 1: Gather Requirements

Ask the user:
1. **Entity class** being configured
2. **Does it extend BaseEntity?** (determines base class usage)
3. **Database schema** — from `TableSchemas`: store, sale, menu, management, salesforce
4. **Custom table name** (or default pluralized + snake_cased)
5. **Default values** for any properties
6. **Relationships** — foreign keys, cascade behavior
7. **Indexes** needed?
8. **JSON columns** — any `[Column(TypeName = "json")]` properties?

## Step 2: Create File

**Location:** `HADV.OMS.Infrastructure/Data/Configurations/[EntityName]Config.cs`

### For entities extending BaseEntity:

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
        SqlSchema = Constants.TableSchemas.Sale; // Set schema
        TableName = "EntityNames"; // Pluralized, auto-snake_cased by base

        // Default values
        builder.Property(b => b.Status).HasDefaultValue(1);

        // String constraints (if not using data annotations)
        builder.Property(b => b.Code).HasMaxLength(50);

        // Relationships
        builder.HasOne(b => b.Location)
            .WithMany()
            .HasForeignKey(b => b.LocationId)
            .OnDelete(DeleteBehavior.Restrict);

        // Indexes
        builder.HasIndex(b => b.Code).IsUnique();

        // IMPORTANT: Call base.Configure LAST
        // This applies: table name, schema, IsActive default, IsDeleted default,
        // Id column order, CreatedBy/ModifiedBy max length, CreatedOn required,
        // and the global soft-delete query filter
        base.Configure(builder);
    }
}
```

### For entities NOT extending BaseEntity:

```csharp
using HADV.OMS.Core.Entities;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace HADV.OMS.Infrastructure.Data.Configurations;

internal class EntityNameConfig : IEntityTypeConfiguration<EntityName>
{
    public void Configure(EntityTypeBuilder<EntityName> builder)
    {
        builder.ToTable("entity_names", Constants.TableSchemas.Sale);

        // Manual soft-delete filter if entity has IsDeleted
        builder.HasQueryFilter(b => !b.IsDeleted);

        // All other configuration...
    }
}
```

## Step 3: Register DbSet

**File:** `HADV.OMS.Infrastructure/Data/AppDbContext.cs`

```csharp
public DbSet<EntityName> EntityNames { get; set; }
```

The configuration is auto-discovered via `ApplyConfigurationsFromAssembly()` in `OnModelCreating`.

## Checklist

- [ ] Class is `internal`
- [ ] Correct base class (`BaseEntityConfiguration<T>` or `IEntityTypeConfiguration<T>`)
- [ ] Schema from `Constants.TableSchemas.*`
- [ ] `base.Configure(builder)` called LAST (for BaseEntityConfiguration inheritors)
- [ ] Relationships use `DeleteBehavior.Restrict` (not Cascade — project convention)
- [ ] DbSet added to AppDbContext
- [ ] Table name is pluralized PascalCase (base will snake_case it)

## Architecture Alerts

Flag to user if:
- Using `DeleteBehavior.Cascade` (project prefers Restrict + soft delete)
- Missing `base.Configure()` call (will miss audit fields and soft delete filter)
- Schema doesn't match the entity's domain
- Table name not following pluralization convention
