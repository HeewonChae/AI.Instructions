# Concurrency — Advanced Patterns

## Channel<T> — Producer/Consumer

```csharp
// ✅ Bounded channel + backpressure strategy
public sealed class PacketProcessor : BackgroundService
{
    private readonly Channel<RawPacket> _channel;

    public PacketProcessor()
    {
        _channel = Channel.CreateBounded<RawPacket>(new BoundedChannelOptions(4096)
        {
            FullMode = BoundedChannelFullMode.Wait,  // backpressure: producer waits
            SingleReader = true,
            SingleWriter = false,
        });
    }

    // Producer (called from receive loop)
    public ValueTask EnqueueAsync(RawPacket packet, CancellationToken ct)
        => _channel.Writer.WriteAsync(packet, ct);

    // Consumer (BackgroundService)
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await foreach (RawPacket packet in _channel.Reader.ReadAllAsync(ct))
        {
            await ProcessPacketAsync(packet, ct);
        }
    }
}
```

### FullMode Selection Criteria

| Strategy | Description | Best For |
|----------|-------------|----------|
| `Wait` | Suspend producer temporarily | No data loss allowed (payments, critical events) |
| `DropWrite` | Drop the new item | Freshness first (sensor data, game ticks) |
| `DropOldest` | Drop the oldest item | Real-time streams (chat broadcast) |

---

## ReaderWriterLockSlim — Many Readers / Few Writers

```csharp
public sealed class SessionStore
{
    private readonly Dictionary<Guid, PlayerSession> _sessions = new();
    private readonly ReaderWriterLockSlim _lock = new(LockRecursionPolicy.NoRecursion);

    public PlayerSession? GetSession(Guid playerId)
    {
        _lock.EnterReadLock();
        try { return _sessions.GetValueOrDefault(playerId); }
        finally { _lock.ExitReadLock(); }
    }

    public void UpsertSession(PlayerSession session)
    {
        _lock.EnterWriteLock();
        try { _sessions[session.PlayerId] = session; }
        finally { _lock.ExitWriteLock(); }
    }
}
```

> **Warning**: `ReaderWriterLockSlim` cannot be mixed with `async` methods — synchronous paths only.

---

## Game Loop Concurrency Pattern

```csharp
// Game tick loop — fixed interval, CancellationToken propagation
public sealed class GameLoopService : BackgroundService
{
    private const int TICK_INTERVAL_MS = 50; // 20 TPS

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using PeriodicTimer timer = new(TimeSpan.FromMilliseconds(TICK_INTERVAL_MS));

        while (await timer.WaitForNextTickAsync(ct))
        {
            await TickAsync(ct);
        }
    }

    private async Task TickAsync(CancellationToken ct)
    {
        // Process independent sessions in parallel
        IReadOnlyList<GameSession> activeSessions = _sessionStore.GetAllActive();
        await Task.WhenAll(activeSessions.Select(s => s.UpdateAsync(ct)));
    }
}
```

- `PeriodicTimer` — .NET 6+; safer than `Timer` callbacks (`async` supported).
- If sessions are independent → `Task.WhenAll`; if order matters → sequential processing.

---

## Lock-Free Patterns

```csharp
// Interlocked — simple counters and flags
private int _connectedCount = 0;

public void OnConnected() => Interlocked.Increment(ref _connectedCount);
public void OnDisconnected() => Interlocked.Decrement(ref _connectedCount);

// ConcurrentDictionary — concurrent key-value access
private readonly ConcurrentDictionary<Guid, PlayerSession> _sessions = new();

_sessions.AddOrUpdate(
    playerId,
    addValueFactory: id => CreateSession(id),
    updateValueFactory: (id, existing) => RefreshSession(existing));
```

---

## Anti-Pattern: ConcurrentQueue + Polling

```csharp
// ❌ Wastes CPU — absolutely prohibited
while (!_queue.IsEmpty)
{
    if (_queue.TryDequeue(out var item))
        Process(item);
    else
        Thread.Sleep(1); // or SpinWait
}

// ✅ Alternative: Channel<T>.Reader.ReadAllAsync()
```
