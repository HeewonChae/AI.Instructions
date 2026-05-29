# Performance & Memory — Advanced Patterns

## Span<T> / Memory<T>

```csharp
// Span<T>: stack-based, async not supported — synchronous parsing only
void ParsePacketHeader(ReadOnlySpan<byte> raw)
{
    ushort opCode = BinaryPrimitives.ReadUInt16BigEndian(raw[..2]);
    int bodyLength = BinaryPrimitives.ReadInt32BigEndian(raw[2..6]);
}

// Memory<T>: heap-based, async supported
async Task ProcessAsync(Memory<byte> buffer, CancellationToken ct)
{
    await _stream.ReadAsync(buffer, ct);
}
```

### Selection Criteria
| | `Span<T>` | `Memory<T>` |
|---|---|---|
| Async support | ❌ | ✅ |
| Stack allocation integration | ✅ | ❌ |
| Usage context | Synchronous parsing/transform | Asynchronous I/O |

---

## ArrayPool<T>

```csharp
// Pattern: Rent → try/finally → Return (always)
byte[] rentedBuffer = ArrayPool<byte>.Shared.Rent(minimumLength: 8192);
try
{
    int bytesRead = await stream.ReadAsync(rentedBuffer.AsMemory(0, 8192), ct);
    ProcessPayload(rentedBuffer.AsSpan(0, bytesRead));
}
finally
{
    // clearArray: set to true if buffer contains sensitive data
    ArrayPool<byte>.Shared.Return(rentedBuffer, clearArray: false);
}
```

> **Note**: `Rent(n)` may return a buffer larger than n — always track the actual used size separately.

---

## stackalloc

```csharp
// Use only for short-lived (single stack frame) buffers under 1KB
Span<byte> packetHeader = stackalloc byte[16];
BinaryPrimitives.WriteUInt16BigEndian(packetHeader, opCode);

// When size is determined at runtime, a safety guard is required
int size = ComputeRequiredSize();
Span<byte> buf = size <= 256
    ? stackalloc byte[256]
    : new byte[size];  // heap fallback when threshold is exceeded
```

---

## ObjectPool<T>

```csharp
// Use Microsoft.Extensions.ObjectPool
public sealed class PacketContextPoolPolicy : IPooledObjectPolicy<PacketContext>
{
    public PacketContext Create() => new PacketContext();

    public bool Return(PacketContext obj)
    {
        obj.Reset();  // full state reset is mandatory
        return true;
    }
}

// DI registration
services.AddSingleton<ObjectPool<PacketContext>>(sp =>
{
    var provider = new DefaultObjectPoolProvider();
    return provider.Create(new PacketContextPoolPolicy());
});

// Usage
PacketContext ctx = _pool.Get();
try { await HandleAsync(ctx, ct); }
finally { _pool.Return(ctx); }
```

> **Reset() rule**: All fields must be restored to their initial values before returning the object. Failure to do so causes data contamination from previous sessions.

---

## Preventing Value Type Copies

```csharp
// Use `in` for large structs passed as parameters (read-only)
void ProcessSnapshot(in WorldSnapshot snapshot) { }

// Use `ref` when mutation is required
void ApplyDamage(ref PlayerStats stats, int damage)
{
    stats.Health = Math.Max(0, stats.Health - damage);
}
```

---

## Hot Path Allocation Checklist

- [ ] Use `ArrayPool<byte>` instead of `new byte[]`
- [ ] Use `for` loops instead of LINQ chains (LINQ causes allocations)
- [ ] Use `StringBuilder` or `string.Create()` for string construction
- [ ] Consider `IReadOnlyList<T>` or `Span<T>` instead of `IEnumerable<T>` returns
- [ ] Remove `object` parameters that cause boxing
- [ ] Use `LoggerMessage.Define()` to prevent ILogger formatting allocations
