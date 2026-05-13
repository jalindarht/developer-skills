---
name: add-endpoint
description: Create a new API endpoint following Ardalis.ApiEndpoints pattern with request class, route constant, Swagger annotations, and Result-to-ActionResult conversion
user_invocable: true
---

# Add New API Endpoint

Create a new API endpoint following the Ardalis.ApiEndpoints pattern. This project does NOT use traditional MVC controllers.

## Step 1: Gather Requirements

Ask the user:
1. **Feature area** (e.g., Cart, Menu, OMS, Subscription, Payments)
2. **Endpoint name** (PascalCase verb+noun — e.g., `CreateCoupon`, `GetOrderDetails`)
3. **HTTP method** (GET, POST, PUT, DELETE)
4. **Route** (kebab-case — e.g., `/coupons/create`, `/orders/{orderId:int}`)
5. **Request properties** with types and validation
6. **Response type** (DTO class or primitive like `bool`)
7. **Which service method** does this call? (or should a new one be created?)
8. **Authorization required?** (default YES — `[Authorize]`)

## Step 2: Validate Architecture Compliance

Before creating, verify:
- [ ] Route follows kebab-case convention
- [ ] Feature folder exists in `CentralAPI/Endpoints/` (create if new feature)
- [ ] Service method exists or will be created (use `/add-service` skill)
- [ ] Route doesn't conflict with existing routes in `ApiEndPoints.cs`
- [ ] Response DTO exists in `Core/DTO/` (create if needed)

## Step 3: Create Files

### 3a. Route Constant
**File:** `HADV.OMS.CentralAPI/Constants/ApiEndPoints.cs`

Add to the appropriate nested class:
```csharp
public class FeatureName
{
    public const string EndpointName = "/feature/route/{paramId:int}";
}
```

### 3b. Request Class
**File:** `HADV.OMS.CentralAPI/Endpoints/[Feature]/[EndpointName].Request.cs`

```csharp
using System.ComponentModel.DataAnnotations;

namespace HADV.OMS.CentralAPI.Endpoints.[Feature];

public class [EndpointName]Request
{
    [Required]
    public int LocationId { get; set; }

    [Required, MaxLength(100)]
    public string Name { get; set; } = string.Empty;

    // Route parameters use [FromRoute]:
    // [FromRoute] public int Id { get; set; }
}
```

### 3c. Endpoint Class
**File:** `HADV.OMS.CentralAPI/Endpoints/[Feature]/[EndpointName].cs`

```csharp
using Ardalis.ApiEndpoints;
using Ardalis.Result.AspNetCore;
using HADV.OMS.CentralAPI.Constants;
using HADV.OMS.Core.Services;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Swashbuckle.AspNetCore.Annotations;

namespace HADV.OMS.CentralAPI.Endpoints.[Feature];

[Authorize]
public class EndpointName : EndpointBaseAsync
    .WithRequest<EndpointNameRequest>
    .WithActionResult<ResponseType>
{
    readonly IFeatureService _service;

    public EndpointName(IFeatureService service)
    {
        _service = service;
    }

    [HttpPost(ApiEndPoints.Feature.EndpointName)]
    [SwaggerOperation(
        Summary = "Brief description of what this endpoint does",
        Tags = new[] { "Feature" },
        OperationId = "Feature.EndpointName")]
    public override async Task<ActionResult<ResponseType>> HandleAsync(
        EndpointNameRequest request,
        CancellationToken cancellationToken = default)
    {
        var result = await _service.MethodAsync(
            /* map request to service params */,
            cancellationToken);

        return this.ToActionResult(result);
    }
}
```

### For endpoints WITHOUT request body (route params only):

```csharp
public class GetById : EndpointBaseAsync
    .WithRequest<int>  // Single route parameter
    .WithActionResult<ResponseDto>
{
    [HttpGet(ApiEndPoints.Feature.GetById)]
    [SwaggerOperation(...)]
    public override async Task<ActionResult<ResponseDto>> HandleAsync(
        [FromRoute] int id,
        CancellationToken cancellationToken = default)
    {
        // ...
    }
}
```

### For endpoints WITHOUT any request:

```csharp
public class GetAll : EndpointBaseAsync
    .WithoutRequest
    .WithActionResult<List<ResponseDto>>
{
    // ...
}
```

## Step 4: Post-Creation Checklist

- [ ] Route constant added to `ApiEndPoints.cs` in correct nested class
- [ ] Request file is separate: `[Endpoint].Request.cs`
- [ ] Endpoint file: `[Endpoint].cs`
- [ ] Inherits correct `EndpointBaseAsync.With*` chain
- [ ] `[SwaggerOperation]` with Summary, Tags, OperationId
- [ ] Uses `this.ToActionResult(result)` for response
- [ ] `[Authorize]` attribute present (unless public endpoint — confirm with user)
- [ ] No business logic in endpoint — delegates to service
- [ ] CancellationToken passed through to service

## Architecture Alerts

Flag to user if:
- Endpoint contains business logic (must be in service layer)
- Multiple service calls without a coordinating service (suggest creating one)
- Route doesn't follow kebab-case convention
- Missing `[Authorize]` on sensitive endpoints
- Response returns entity directly instead of DTO
