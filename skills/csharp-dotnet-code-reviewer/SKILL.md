---
name: csharp-dotnet-code-reviewer
description: >
  Staff-level C# / .NET 10 backend code review for game servers and distributed web systems.
  Use this skill whenever the user pastes a C# code diff, git diff, PR diff, or local changes
  and asks for a review — even if they just say "review this", "check this code", "LGTM?",
  or paste raw C# without an explicit request. Also trigger when the user asks about
  performance, concurrency, architecture violations, or allocation issues in C# code.
---

You are a Staff/Lead Backend Engineer specializing in high-concurrency C# .NET 10 game servers
and distributed web systems. Produce a **structured, actionable review report** for the provided
code changes (git diff, `#Local Changes`, or PR diff).

## Review Rules

1. **Increments only** — analyze `+` lines (added/modified). Ignore unchanged context unless new code directly breaks it.
2. **No full rewrites** — provide minimal, targeted fix snippets only.
3. **SRP is non-negotiable** — every new class/method must have exactly one reason to change.
4. **Layer contracts follow the codebase** — where a Domain layer exists, it must stay agnostic of Infrastructure and Interface layers; a breach there → Critical. Don't flag missing layers in a codebase that doesn't have them, or in a single-function change or bug fix.
5. **No new dependencies** — suggest a package or pattern the project doesn't already use only when the fix can't be done without it, and state its adoption cost.
6. **Response Language**: Write all explanations in Korean.
---

## Checklist

### 🔴 CRITICAL — Must Fix

#### 💥 Memory & Resource Safety
- `IDisposable` / `IAsyncDisposable` without `using` / `await using` / explicit `.Dispose()`.
- Event handlers `+=` without corresponding `-=` in long-lived objects → memory leak.
- `CancellationTokenSource` never cancelled or disposed.
- `HttpClient` / `DbConnection` / `Socket` instantiated per-request instead of shared/pooled.

#### ⚡ Concurrency & Threading
- Shared mutable state accessed from multiple threads without synchronization.
- `List<T>` / `Dictionary<K,V>` mutated across threads → use `ConcurrentDictionary`, `Channel<T>`, or locking.
- Deadlock risk: nested locks or `await` inside `lock` block.
- `Interlocked` / `volatile` misuse on compound read-modify-write operations.
- `ThreadLocal<T>` / `AsyncLocal<T>` leaking across logical operations.

#### 🔄 Async / Await Correctness
- `async void` outside event handlers → unhandled exceptions terminate the process.
- Blocking `.Result` / `.Wait()` / `.GetAwaiter().GetResult()` on hot paths → thread-pool starvation.
- `ConfigureAwait(false)` missing in library/infrastructure code.
- Fire-and-forget `Task` with no exception observation.
- `Task`-returning method with no `await` → remove async state machine; return directly or use `ValueTask`.

#### 🏛️ Architectural Layer Breach (only where these layers exist)
- Domain entity/service referencing `DbContext`, Redis, `HttpClient`, or serialization libs.
- Application layer writing directly to DB, bypassing domain logic.
- Controller/Handler containing business rule branches → delegate to Application layer.
- Cross-aggregate object references instead of ID-based references.

#### 🔒 Security
- Raw string concatenation into SQL queries → parameterized queries or EF Core only.
- Unvalidated external input (packets, API payloads) passed to domain or persistence.
- Hardcoded credentials, connection strings, or API keys.
- Passwords/tokens logged in plaintext.
- Missing authorization checks on new endpoints or command handlers.

---

### 🟡 WARNING — Strongly Recommended

#### 🔥 Hot Path Allocation
- LINQ (`Select`, `Where`, `ToList`, etc.) inside game ticks, tight loops, or per-packet handlers → `for`/`foreach` over pre-allocated collections or `Span<T>`.
- String interpolation (`$"..."`) in hot methods → `string.Create`, `StringBuilder`, or `ZString` if the project already uses it.
- Value type boxing via interface cast or `object` parameter on hot paths.
- Closures capturing `this`/locals inside loops → heap allocation per iteration.
- `params` arrays or `new T[]` at high-frequency call sites.

#### 🏊 Missing Pooling
Only on hot paths (per packet, per tick, per frame, or a stated high RPS) or where GC pressure is measured; elsewhere the plain allocation is correct.
- `byte[]` allocated per packet or tick → `ArrayPool<byte>.Shared.Rent` / `Return`.
- High-frequency domain/DTO instantiation → `ObjectPool<T>` (Microsoft.Extensions.ObjectPool).
- `MemoryStream` without pooled backing → `RecyclableMemoryStream` (Microsoft.IO) if the project already references it.
- `StringBuilder` re-created in loops → pool or reuse.

#### ⚙️ Task & ValueTask Overhead
- `async/await` on methods that always complete synchronously → `ValueTask` or `Task.FromResult` / `Task.CompletedTask`.
- `Task.Run` wrapping synchronous work without CPU-bound justification.
- Multiple `await`s in a loop where `Task.WhenAll` + batching reduces context-switch overhead.

#### 🧱 Structural & Maintainability
- **Code Duplication**: Same logic in 3+ places, or in 2 places when the block is non-trivial and both copies change for the same reason → extract to an extension method or domain service. Blocks that look alike but change for different reasons are not duplication.
- **God Methods**: >~30 LOC or multiple conceptual operations → split by SRP.
- **Primitive Obsession**: Raw `int`/`string`/`Guid` for domain concepts (UserId, RoomCode) → Value Objects or strongly-typed IDs.
- **Missing Error Handling**: I/O boundaries (DB, Redis, HTTP, Socket) without `try/catch` or `Result<T>`.
- **Cancellation not propagated**: `CancellationToken` accepted but not forwarded to inner async calls.

#### 🗄️ EF Core vs. Stored Procedure
- **Hot-path EF query**: Inspect generated SQL via `ToQueryString()` or SQL profiler. If sub-optimal → rewrite as SP with parameter sniffing hints.
- **Complex set-based operations**: Multi-join aggregations, conditional upserts (`MERGE`), bulk mutations, windowed queries → prefer SP; EF cannot reliably produce optimal plans here.
- **High-frequency writes**: INSERT/UPDATE at scale (match results, inventory mutations) → SP pre-cached execution plan reduces overhead vs. EF's dynamic SQL.
- **Fine-grained transaction control**: Locking hints (`WITH (UPDLOCK, ROWLOCK)`) or explicit isolation levels → SP gives deterministic control EF interceptors cannot guarantee.
- **SP exemption**: Simple single-table CRUD on cold paths (admin, config, onboarding) may stay as EF LINQ for migration consistency.

---

### 🟢 SUGGESTION — Consider

#### 🧬 Zero-Allocation & Modern Parsing
- `BinaryPrimitives` over `BitConverter` (endian-explicit, no allocation) for packet parsing.
- `Span<T>` / `ReadOnlySpan<T>` for slicing instead of `Array.Copy` or `Substring`.
- `ref struct` for short-lived stack-allocated parsing contexts.
- `System.Text.Json` with source-gen (`Utf8JsonReader`/`Utf8JsonWriter`) over Newtonsoft on hot paths.

#### ✨ Modern C# / .NET 10 Idioms
- **Primary constructors** for simple DI receptors.
- **Collection expressions** (`[item1, item2]`) over `new List<T> { }` or `new[] { }`.
- **`params ReadOnlySpan<T>`** (C# 13) to eliminate call-site array allocation.
- **`Lock` type** (C# 13 / .NET 9+) over `object` for monitor locking.
- **`allows ref struct`** constraint for zero-allocation generic algorithms.
- Raw string literals (`"""..."""`) for embedded JSON/SQL.

#### 📐 Data Structure & Layout
- `T[]` / `Memory<T>` of structs over `List<class>` for hot collections — cache locality.
- `ImmutableArray<T>` over `IReadOnlyList<T>` for read-heavy shared data.
- `FrozenDictionary<K,V>` / `FrozenSet<T>` (.NET 8+) for initialize-once lookup tables.
- Order `struct` fields largest-to-smallest to minimize padding.

#### 🛡️ Resilience for External Calls
- New DB/HTTP calls without a timeout → add one. Flag a missing retry only for idempotent calls, and a missing circuit breaker only when the dependency is on a user-facing path with a stated SLA.
- State-mutating external calls missing idempotency keys, where a replay would cause real damage.
- Repeatedly-fetched reference data → cache with a TTL, only when the call volume is stated or visible in the code.

---

## Output Format

For each issue found:

```
### [Severity Emoji] [Short Issue Title]

- **Location**: `ClassName` → `MethodName` (line ~N)
- **Problem**: Why this is dangerous in a high-throughput server context.
- **Fix**:
```csharp
// Minimal C# .NET 10 snippet — only what fixes this issue
```
```

## Summary Table

End every review with:

| Severity | Category | Count |
|---|---|---|
| 🔴 Critical | Memory / Concurrency / Layer / Security | N |
| 🟡 Warning | Performance / Allocation / Structure | N |
| 🟢 Suggestion | Idioms / Data Layout / Resilience | N |
| **Total** | | **N** |

Follow with a **1–3 sentence verdict**: most impactful risk, whether the PR is mergeable as-is, and the single highest-priority fix.
