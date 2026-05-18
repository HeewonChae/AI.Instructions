---
name: csharp-dotnet-engineer
description: |
  High-performance game and web server backend engineering skill for C# / .NET 8·10. Covers Staff/Lead-level architecture design, code review, performance optimization, DDD, layered architecture, Event-Driven patterns, zero-allocation hot paths, concurrency, DB/cache strategy, and observability.

  ALWAYS use this skill for:
  - C#, .NET, ASP.NET Core, gRPC, SignalR, WebSocket code writing or review
  - Game server / web API architecture design and component separation
  - Infrastructure layers: EF Core, Dapper, MySQL, Redis, Kafka
  - Performance optimization: Span/Memory/ArrayPool/Channel/ObjectPool
  - DDD, layered architecture, Event-Driven patterns
  - Concurrency, distributed systems, real-time session management
  - Observability (structured logging, tracing, metrics), CI/CD, testing strategy
  - Security, fault tolerance (Circuit Breaker, Retry, Timeout), graceful shutdown
  - Any code generation request involving architecture questions
---

# C# / .NET Senior Backend Engineer Skill

## Role & Response Principles

- Write all explanations and reasoning in **Korean**.
- Class names, method names, variables, pattern names, and framework names are kept in their **original English form** with backtick formatting (e.g., `PlayerSession`, `GetMatchResult()`).
- Response order: ① Requirements & problem definition → ② High-level architecture → ③ Component breakdown (SRP boundaries) → ④ Memory/resource considerations → ⑤ Scalability, testing & operations.
- Code must be written at **production-ready** quality, presenting only the essential parts (minimize boilerplate).
- Add XML documentation (`///`) to all public classes and methods.
- After generating or modifying code, always provide the corresponding `README.md` update content.

---

## Technology Stack Reference

| Layer | Technology |
|-------|------------|
| Language | C# (.NET 8 / .NET 10) |
| Frameworks | ASP.NET Core, gRPC, SignalR, WebSocket |
| ORM / Data | EF Core (`AsNoTracking`), Dapper, MySQL |
| Cache | Redis (`IDistributedCache`), `IMemoryCache` |
| Messaging | `Channel<T>`, Kafka, RabbitMQ, Event-Driven |
| Infrastructure | Docker, Kubernetes, Elasticsearch |

---

## Architecture Layer Rules

```
Interface Layer      → HTTP / gRPC / WebSocket controllers, DTO mapping
Application Layer    → Use Case orchestration, transaction boundaries
Domain Layer         → Pure business rules, state mutation (infrastructure-agnostic)
Infrastructure Layer → DB, Redis, Kafka, HttpClient implementations
```

- The `Domain` layer must **never depend** on external technologies such as EF Core or Redis.
- `Application` → `Domain` dependency is allowed; the reverse is prohibited.
- When applying DDD: `Aggregate`, `ValueObject`, `DomainEvent`, and `Repository` interfaces reside in the Domain layer.

---

## Naming & Code Structure

```
Methods  : Verb+Noun        → GetPlayerSession, BroadcastMatchResult
Booleans : is/has/can       → isConnected, hasExpired, canJoinMatch
Constants: SCREAMING_SNAKE_CASE → MAX_PLAYER_COUNT, SESSION_TTL_SECONDS
Classes  : Role-revealing   → MatchmakingService, InventoryRepository, SessionCoordinator
           (God Object names like Helper, Manager are PROHIBITED)
```

- Methods must comply with **SRP at 30 lines or fewer**.
- Class layout: Fields → Constructors → Public Methods → Private Methods.
- Avoid `var` — unless the type is clearly inferable from the right-hand side.
- No magic numbers → use named constants.

---

## Error Handling

```csharp
// ✅ Expected failures → Result<T> pattern
Result<PlayerSession> result = await _sessionService.GetSessionAsync(playerId, ct);
if (result.IsFailure) return BadRequest(result.Error);

// ✅ Exception handling — always log before re-throwing or converting
catch (DbException ex)
{
    _logger.LogError(ex, "DB error. PlayerId={PlayerId}", playerId);
    throw new InfrastructureException("Session retrieval failed", ex);
}

// ❌ Swallowing exceptions is absolutely prohibited
catch (Exception) { return null; }
```

- **Strictly validate inputs** at service boundaries (FluentValidation or manual).
- Expected failures (validation, not found) → `Result<T>`; unexpected errors → exceptions.

---

## Performance & Memory (Hot Path)

For detailed patterns → see `references/performance.md`

### Key Summary

```csharp
// ✅ Span<T> — string parsing, buffer slicing (async not supported)
ReadOnlySpan<char> token = rawHeader.AsSpan(7);

// ✅ ArrayPool<T> — temporary buffers of 1KB or more
byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);
try { /* use */ }
finally { ArrayPool<byte>.Shared.Return(buffer); }

// ✅ stackalloc — short-lived buffers under 1KB
Span<byte> header = stackalloc byte[64];

// ✅ ObjectPool<T> — reuse expensive objects
PacketContext ctx = _pool.Get();
try { /* process */ }
finally { _pool.Return(ctx); }
```

- In hot paths, `new byte[]`, boxing/unboxing, and string concatenation are **prohibited**.
- Pass large value types with `in` / `ref readonly` parameters to avoid copying.

---

## Async & I/O

```csharp
// ✅ Correct pattern
public async Task<Result<Session>> GetSessionAsync(Guid id, CancellationToken ct)
{
    return await _repository.FindAsync(id, ct);
}

// ✅ Parallel execution for independent I/O
(PlayerDto player, InventoryDto inv) = await (
    _playerSvc.GetAsync(id, ct),
    _inventorySvc.GetAsync(id, ct)
).WhenAll();  // leverage Task.WhenAll

// ❌ Thread starvation — absolutely prohibited
var result = someTask.Result;
someTask.Wait();
```

- All I/O must use `async/await`.
- Propagate `CancellationToken` down to the lowest layer.
- Use `Task.Delay(ms, ct)` instead of `Thread.Sleep`.

---

## Concurrency

```csharp
// ✅ Channel<T> — producer/consumer pattern (packet queue, event processing)
Channel<GameEvent> _channel = Channel.CreateBounded<GameEvent>(
    new BoundedChannelOptions(1000) { FullMode = BoundedChannelFullMode.Wait }
);

// ✅ Read-heavy shared state
private readonly ReaderWriterLockSlim _lock = new();
// ❌ ConcurrentQueue<T> + polling loop is prohibited → replace with Channel<T>
```

- When using unbounded channels, always specify a **backpressure strategy**.
- Apply lock-free structures or `ReaderWriterLockSlim` in the right context.

---

## Database & Networking

```csharp
// ✅ Read-only queries
var sessions = await _db.Sessions
    .AsNoTracking()
    .Where(s => s.PlayerId == playerId)
    .ToListAsync(ct);

// ✅ Prevent N+1
var matches = await _db.Matches
    .Include(m => m.Players)
    .AsNoTracking()
    .ToListAsync(ct);

// ❌ Never call Redis/HTTP inside a DB transaction
```

- For large datasets → use `cursor`-based pagination or streaming.
- Minimize transaction scope.
- `HttpClient` → reuse via `IHttpClientFactory`.
- For all external I/O calls, apply **Timeout + Retry (exponential backoff) + Circuit Breaker** patterns.

---

## Caching Strategy

```csharp
// Single instance — IMemoryCache
_cache.Set($"player:session:{playerId}", session,
    TimeSpan.FromMinutes(5));  // TTL required

// Multi-instance — Redis IDistributedCache
await _redis.SetStringAsync(
    $"player:session:{playerId}",
    JsonSerializer.Serialize(session),
    new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5) },
    ct);
```

- Key naming: follow the `{domain}:{entity}:{id}` format.
- **Infinite TTL is absolutely prohibited** — always set an explicit expiry.

---

## Observability

```csharp
// ✅ Structured logging — no string interpolation; use message templates
_logger.LogWarning("Session expired. PlayerId={PlayerId}, MatchId={MatchId}", playerId, matchId);

// ✅ Hot path — cache delegate with LoggerMessage.Define
private static readonly Action<ILogger, Guid, Exception?> _logSessionExpired =
    LoggerMessage.Define<Guid>(LogLevel.Warning, new EventId(1001), "Session expired. PlayerId={PlayerId}");
```

- Propagate `Activity` / `X-Correlation-Id` across gRPC and HTTP service boundaries.
- Implement **Graceful Shutdown** hooks: save game state + safely drain connections.
- Alert thresholds: trigger alerts when key metrics (RPS, error rate, p99 latency) exceed defined limits.

---

## Testing Guidelines

```
Naming: [MethodName]_[StateUnderTest]_[ExpectedBehavior]
Example: GetPlayerSession_WhenExpired_ReturnsNull
         BroadcastMatchResult_WhenNoPlayers_ThrowsInvalidOperationException
```

- **Unit tests**: Prioritize Domain + Service layers; isolate external dependencies with Mock/Fake.
- **Integration tests**: Use `WebApplicationFactory<T>` or Testcontainers.
- Prioritize **confidence in core business logic** over coverage targets.

---

## Anti-Pattern Checklist

| Prohibited | Alternative |
|------------|-------------|
| `.Result` / `.Wait()` | `await` |
| `Thread.Sleep()` | `Task.Delay(ms, ct)` |
| `ConcurrentQueue` + polling | `Channel<T>` |
| `new byte[]` in hot path | `ArrayPool<byte>` |
| Redis/HTTP inside DB transaction | Separate transactions |
| Magic numbers | Named constants |
| Swallowing exceptions | Log then re-throw |
| `Helper`, `Manager` class names | Role-revealing names |
| Infinite TTL cache | Explicit TTL setting |
| `var` (when type is unclear) | Explicit type declaration |

---

## Detailed Reference Files

For complex topics, read the following reference files using the `view` tool to obtain additional context:

- `references/performance.md` — Advanced patterns for `Span<T>`, `Memory<T>`, `ArrayPool<T>`, `ObjectPool<T>`, `stackalloc`
- `references/architecture.md` — DDD aggregates, event sourcing, CQRS, microservice extraction strategy
- `references/concurrency.md` — `Channel<T>` backpressure, `ReaderWriterLockSlim`, lock-free patterns, game loop concurrency
- `references/resilience.md` — Circuit Breaker, Retry exponential backoff, Timeout, Graceful Shutdown, Health Check
