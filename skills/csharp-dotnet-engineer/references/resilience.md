# Resilience & Stability — Advanced Patterns

## Retry + Exponential Backoff (Polly)

```csharp
// NuGet: Microsoft.Extensions.Http.Resilience (.NET 8+)
services.AddHttpClient<IMatchmakingClient, MatchmakingClient>()
    .AddStandardResilienceHandler(options =>
    {
        options.Retry.MaxRetryAttempts = 3;
        options.Retry.Delay = TimeSpan.FromMilliseconds(200);
        options.Retry.BackoffType = DelayBackoffType.Exponential;
        options.CircuitBreaker.SamplingDuration = TimeSpan.FromSeconds(30);
        options.CircuitBreaker.FailureRatio = 0.5;
        options.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(10);
    });
```

---

## Circuit Breaker State Flow

```
Closed ──[failure threshold exceeded]──► Open ──[recovery wait]──► Half-Open
  ▲                                                                     │
  └──────────────────────[probe success]────────────────────────────────┘
```

- **Closed**: Requests pass through normally.
- **Open**: Immediately returns failure (Fail Fast). Protects DB and external APIs.
- **Half-Open**: Limited traffic to detect recovery.

```csharp
// Direct Polly usage (complex custom policy)
ResiliencePipeline pipeline = new ResiliencePipelineBuilder()
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        FailureRatio = 0.5,
        SamplingDuration = TimeSpan.FromSeconds(30),
        MinimumThroughput = 10,
        BreakDuration = TimeSpan.FromSeconds(15),
    })
    .AddRetry(new RetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromMilliseconds(100),
        BackoffType = DelayBackoffType.Exponential,
        ShouldHandle = new PredicateBuilder().Handle<HttpRequestException>(),
    })
    .AddTimeout(TimeSpan.FromSeconds(5))
    .Build();
```

---

## Graceful Shutdown

```csharp
// Program.cs — register lifetime events
IHostApplicationLifetime lifetime = app.Lifetime;

lifetime.ApplicationStopping.Register(() =>
{
    _logger.LogInformation("Shutdown signal received — starting session save");
    // Synchronous hook: keep short (< 2 seconds)
});

// Watch CancellationToken in BackgroundService
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    try
    {
        await RunGameLoopAsync(stoppingToken);
    }
    finally
    {
        // Cleanup logic that must run on shutdown
        await FlushPendingSessionsAsync(CancellationToken.None); // use a fresh token
        _logger.LogInformation("Game loop shutdown complete");
    }
}
```

### Kubernetes Shutdown Timeline

```
SIGTERM received
 → PreStop hook (if configured)
 → terminationGracePeriodSeconds timer starts (default: 30s)
 → App cleanup logic executes
 → SIGKILL (if period is exceeded)
```

> `terminationGracePeriodSeconds` must be set higher than the app's maximum cleanup time.

---

## Health Check

```csharp
services.AddHealthChecks()
    .AddDbContextCheck<GameDbContext>("db", tags: ["live", "ready"])
    .AddRedis(redisConnectionString, "redis", tags: ["ready"]);

// Liveness: /health/live — determines whether to restart (DB check not required)
// Readiness: /health/ready — determines whether to receive traffic

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live"),
    ResultStatusCodes = { [HealthStatus.Unhealthy] = 503 }
});
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResultStatusCodes = { [HealthStatus.Unhealthy] = 503 }
});
```

---

## Timeout Strategy Layers

```
Client request Timeout    (e.g., 5s)
 └─ Internal service call Timeout (e.g., 2s)
     └─ DB query Timeout          (e.g., 1s, CommandTimeout)
```

- Each layer's Timeout must be **shorter than the layer above** to be meaningful.
- Leverage `CancellationTokenSource.CreateLinkedTokenSource(requestCt, timeoutCt)`.

```csharp
using CancellationTokenSource cts =
    CancellationTokenSource.CreateLinkedTokenSource(requestCancellationToken);
cts.CancelAfter(TimeSpan.FromSeconds(2));

try
{
    PlayerSession session = await _repo.GetAsync(id, cts.Token);
}
catch (OperationCanceledException) when (!requestCancellationToken.IsCancellationRequested)
{
    // Internal timeout, not client cancellation — return 503
    throw new ServiceUnavailableException("Session retrieval timed out");
}
```
