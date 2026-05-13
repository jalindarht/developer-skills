---
name: add-dto
description: Create a new Data Transfer Object (DTO) in Core/DTO following POCO conventions with proper nullable types, string initialization, and collection defaults
user_invocable: true
---

# Add New DTO

Create a new Data Transfer Object following project conventions.

## Step 1: Gather Requirements

Ask the user:
1. **DTO name** (PascalCase — e.g., `CouponDetail`, `OrderSummary`)
2. **Properties** with types
3. **Related DTOs** — does it reference other DTOs as nested objects?
4. **Is it a request DTO or response DTO?**
   - Request DTOs go in endpoint's `.Request.cs` file with validation attributes
   - Response DTOs go in `Core/DTO/`

## Step 2: Create File

**Location:** `HADV.OMS.Core/DTO/[DtoName].cs`

```csharp
namespace HADV.OMS.Core.DTO;

public class DtoName
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string? OptionalField { get; set; }
    public decimal Price { get; set; }
    public bool IsEnabled { get; set; }
    public int? NullableInt { get; set; }
    public List<ChildDto>? Children { get; set; }
    public List<int> SelectedIds { get; set; } = new();
}
```

## Conventions

- Simple POCOs — no base class, no inheritance
- Initialize `string` properties with `string.Empty`
- Use nullable (`?`) for optional properties
- Initialize required collections with `new()`, optional ones as nullable
- No business logic or methods
- No data annotations (those go on Request classes and entities)
- One DTO per file (or related small DTOs can share a file)

## Architecture Alerts

Flag to user if:
- DTO inherits from BaseEntity or any entity class (DTOs must be independent)
- DTO contains business methods
- DTO mirrors entity exactly (confirm if projection/mapping will be needed)
- DTO exposes sensitive fields (passwords, tokens, internal IDs that shouldn't be public)
