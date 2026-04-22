# Architecture — Advanced Patterns

## DDD Core Components

```
Domain/
├── Aggregates/
│   ├── Match/
│   │   ├── Match.cs             ← Aggregate Root
│   │   ├── MatchPlayer.cs       ← Entity (internal)
│   │   └── MatchStatus.cs       ← Value Object
│   └── Player/
│       ├── Player.cs
│       └── PlayerStats.cs
├── Events/
│   ├── MatchStartedEvent.cs     ← Domain Event
│   └── PlayerKilledEvent.cs
├── Repositories/
│   └── IMatchRepository.cs      ← Interface only (implementation lives in Infrastructure)
└── Services/
    └── MatchmakingService.cs    ← Domain Service (coordinates multiple aggregates)
```

### Aggregate Root Example

```csharp
/// <summary>Match aggregate root. External code must not manipulate MatchPlayer directly.</summary>
public sealed class Match
{
    private readonly List<MatchPlayer> _players = new();
    private readonly List<IDomainEvent> _domainEvents = new();

    public MatchId Id { get; private init; }
    public MatchStatus Status { get; private set; }
    public IReadOnlyList<MatchPlayer> Players => _players.AsReadOnly();
    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    public Result StartMatch()
    {
        if (_players.Count < MIN_PLAYER_COUNT)
            return Result.Failure("Insufficient player count");

        Status = MatchStatus.InProgress;
        _domainEvents.Add(new MatchStartedEvent(Id, DateTime.UtcNow));
        return Result.Success();
    }

    public void ClearDomainEvents() => _domainEvents.Clear();
}
```

---

## CQRS Separation Strategy

```
Application/
├── Commands/
│   ├── StartMatchCommand.cs
│   └── StartMatchCommandHandler.cs   ← Write (EF Core + transaction)
└── Queries/
    ├── GetMatchResultQuery.cs
    └── GetMatchResultQueryHandler.cs  ← Read (Dapper + AsNoTracking, cache)
```

- Command: state mutation, domain event publishing, transaction management.
- Query: read-only, uses a separate Read Model or optimized Dapper queries.

---

## Event-Driven Session/State Management

```csharp
// Domain event publishing (Application Layer)
public async Task HandleStartMatchAsync(StartMatchCommand cmd, CancellationToken ct)
{
    Match match = await _matchRepo.GetAsync(cmd.MatchId, ct);
    Result result = match.StartMatch();

    if (result.IsFailure) return;

    await _matchRepo.SaveAsync(match, ct);

    // Domain events → dispatch to message bus or Channel<T>
    foreach (IDomainEvent evt in match.DomainEvents)
        await _eventBus.PublishAsync(evt, ct);

    match.ClearDomainEvents();
}
```

---

## Microservice Extraction Criteria

Consider service extraction when you see these signals:
1. **Independent deployment needed** — two domains with different release cadences coexist in one service.
2. **Data isolation** — two aggregates owned by different teams.
3. **Different scaling units** — matchmaking is CPU-intensive, inventory is DB-intensive.

**Extraction priority decision flow**:
```
Monolith module → review traffic, team, and data boundaries
→ Boundaries are clear    → microservice
→ Boundaries are unclear  → keep as modular monolith (avoid complexity cost)
```

---

## Layer Dependency Rules Summary

```
Interface → Application → Domain      (allowed)
Infrastructure → Domain               (allowed — implements interfaces)
Domain → Infrastructure               (❌ absolutely prohibited)
Domain → Application                  (❌ absolutely prohibited)
Interface → Domain (direct)           (❌ prohibited in principle — DTO conversion layer required)
```
