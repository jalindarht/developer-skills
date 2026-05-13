---
name: add-enum
description: Create a new domain enum following project naming conventions, explicit int values, and proper folder organization by feature
user_invocable: true
---

# Add New Enum

Create a new domain enum following project conventions.

## Step 1: Gather Requirements

Ask the user:
1. **Enum name** (PascalCase, singular — e.g., `CouponType`, `NotificationStatus`)
2. **Feature area** (determines subfolder — e.g., Payments, Subscription, OMS)
3. **Values** with explicit integer assignments
4. **Is it an error code enum?** (goes in `Core/ErrorCodes/` instead)

## Step 2: Create File

### Domain Enum
**Location:** `HADV.OMS.Core/Enums/[Feature]/[EnumName].cs` or `HADV.OMS.Core/Enums/[EnumName].cs`

```csharp
namespace HADV.OMS.Core.Enums;

public enum EnumName
{
    Value1 = 1,
    Value2 = 2,
    Value3 = 3
}
```

### Error Code Enum
**Location:** `HADV.OMS.Core/ErrorCodes/[FeatureErrors].cs`

```csharp
namespace HADV.OMS.Core.ErrorCodes;

public enum FeatureErrors
{
    InvalidOperation,
    DuplicateEntry,
    InsufficientFunds
}
```

Usage in services: `Result.Error(FeatureErrors.InvalidOperation.ToErrorCode(), "message")`

## Conventions

- Always use explicit integer values starting from 1 (not 0) for domain enums
- Error code enums can use default ordinal values (0, 1, 2...)
- Use `ToErrorCode()` extension to convert error enums to string format: `"EnumName.Value"`
- One enum per file
- PascalCase enum name and values

## Architecture Alerts

Flag to user if:
- Enum has more than ~15 values (consider if it should be a lookup table instead)
- Enum is being used for complex state machine (suggest separate state management)
- Duplicate values across different enums in the same feature area
