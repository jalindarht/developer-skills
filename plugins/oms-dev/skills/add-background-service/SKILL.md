---
name: add-background-service
description: Create a new hosted BackgroundService with proper DI scope creation, conditional registration in Program.cs (worker vs API mode)
user_invocable: true
---

# Add New Background Service

Create a new hosted background service following project conventions.

## Step 1: Gather Requirements

Ask the user:
1. **Service name** (e.g., `CleanupExpiredCouponsBackgroundService`)
2. **Execution mode** — Worker mode (`--worker`) or API mode (default)?
3. **Schedule** — interval-based (e.g., every 5 minutes) or one-time on startup?
4. **Which services** does it depend on?
5. **Error handling** — retry strategy?

## Step 2: Create File

**Location:** `HADV.OMS.CentralAPI/CronJob/[ServiceName].cs`

```csharp
namespace HADV.OMS.CentralAPI.CronJob;

public class FeatureNameBackgroundService : BackgroundService
{
    readonly IServiceScopeFactory _scopeFactory;
    readonly ILogger<FeatureNameBackgroundService> _logger;

    public FeatureNameBackgroundService(
        IServiceScopeFactory scopeFactory,
        ILogger<FeatureNameBackgroundService> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope = _scopeFactory.CreateScope();
                var service = scope.ServiceProvider.GetRequiredService<IFeatureService>();

                // Execute business logic
                await service.ProcessAsync(stoppingToken);

                _logger.LogInformation("FeatureName background job completed at {Time}", DateTimeOffset.UtcNow);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in FeatureName background service");
            }

            // Wait before next execution
            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}
```

## Step 3: Register in Program.cs

**File:** `HADV.OMS.CentralAPI/Program.cs`

For **worker mode** services (add inside the `if (isWorker)` block):
```csharp
builder.Services.AddHostedService<FeatureNameBackgroundService>();
```

For **API mode** services (add inside the `else` block):
```csharp
builder.Services.AddHostedService<FeatureNameBackgroundService>();
```

## Checklist

- [ ] Uses `IServiceScopeFactory` to create DI scopes (NEVER inject scoped services directly)
- [ ] Creates a new scope for each execution cycle
- [ ] Disposes scope after use (`using var scope`)
- [ ] Handles exceptions gracefully (logs and continues)
- [ ] Respects `CancellationToken` for graceful shutdown
- [ ] Registered in correct mode (worker vs API) in Program.cs
- [ ] Delay between executions to avoid tight loops

## Architecture Alerts

Flag to user if:
- Injecting scoped services directly into constructor (must use IServiceScopeFactory)
- Missing try/catch (unhandled exceptions will crash the background service)
- No delay between iterations (will consume 100% CPU)
- Registered in wrong mode (worker background jobs shouldn't run in API mode unless needed)
