# Global Instructions

## Critical Constraints

- **Response Language**: You MUST explain all concepts, architectures, and logic in **Korean**.
- **Code Identifiers**: Never translate programming languages, framework names, class names, method names, variables, or design patterns. Keep them in English and wrap them in backticks (e.g., `PlayerSession`, `GetMatchResult()`).
- **Target Audience**: You are assisting a Senior Backend Engineer specializing in high-performance Game and Web Servers. Code must be production-ready, highly optimized, and robust.


## Tech Stack

- **Language**: C\# (.NET 8 / .NET 10)
- **Frameworks**: ASP.NET Core, gRPC, SignalR, WebSocket
- **Data \& ORM**: Entity Framework Core, Dapper, MySQL, Redis
- **Messaging**: Internal Message Queues, Event-driven architecture
- **Infrastructure**: Docker, Kubernetes, Elasticsearch


## Architecture Patterns

- **Layered \& DDD**: Strict separation of concerns (Application, Domain, Infrastructure, Interface). Use Domain-Driven Design for complex game states.
- **Event-Driven**: Utilize events for game session and state management to decouple logic.
- **Dependency Injection**: Leverage the built-in .NET DI container.


## Code Guidelines

### Documentation and Maintenance

- **README Updates**: After generating or modifying code, immediately provide the updated content for the project's `README.md`. Clearly document new features, architecture changes, and execution steps.


### Naming Conventions

- Use descriptive and declarative names that instantly reveal the responsibility and data flow.
- **Methods**: Verb + Noun (e.g., `GetPlayerSession`, `BroadcastMatchResult`).
- **Booleans**: Prefix with `is`, `has`, `can` (e.g., `isConnected`).
- **Constants**: Use SCREAMING_SNAKE_CASE (e.g., `MAX_PLAYER_COUNT`).
- Avoid arbitrary abbreviations unless widely accepted (e.g., `id`, `dto`).


### Structure and Reusability

- Keep methods small and strictly adhere to the Single Responsibility Principle (SRP) (guideline: < 30 lines).
- Extract repeated logic into dedicated services, utilities, or extension methods.
- Adhere to the Open-Closed Principle (OCP) to ensure code is open for extension but closed for modification.


### Readability

- Add XML documentation (`///`) to all public classes and methods.
- Use inline comments only to explain the "Why" (business logic context), never the "What".
- Use `#region` only in exceptionally large files to group related fields and methods.
- Class layout: Fields → Constructors → Public Methods → Private Methods.
- Avoid using `var` when the right-hand side does not explicitly reveal the type.
- Avoid overly clever, unreadable one-liners. Prioritize clarity.


### Error Handling

- Use structured exception handling with specific exception types.
- Never swallow exceptions. Always log before returning an error response or rethrowing.
- For expected failures (e.g., validation, missing items), use the `Result<T>` pattern instead of throwing exceptions.
- Validate all inputs strictly at the service boundaries.


## Performance and Memory

### Zero-Allocation Hot Paths

- Be extremely sensitive to memory allocation in hot paths (packet processing, matchmaking, combat loops).
- Prevent struct copying by utilizing `in`, `ref`, and `ref readonly` parameters for large value types.


### Memory Management

```
- **`Span<T>` / `Memory<T>`**: Use for string parsing, buffer slicing, and byte manipulation to eliminate heap allocation. Remember `Span<T>` cannot be used in async state machines; use `Memory<T>` instead.
```

- **`stackalloc`**: Use for tiny, short-lived buffers (< 1KB). Always wrap the result in `Span<T>`.
- **`ArrayPool<T>`**: Rent arrays for temporary buffers larger than 1KB. Always return them using a `try/finally` block.
- **`ObjectPool<T>`**: Pool expensive or frequently created objects (e.g., packet contexts, custom buffers). Always `Reset()` state upon return.


### Async and I/O

- All I/O operations must be `async`/`await`.
- Never use `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()`.
- Always propagate `CancellationToken` to the lowest asynchronous level.
- Use `Task.Delay(ms, cancellationToken)` instead of `Thread.Sleep()`.
- Use `Task.WhenAll()` for independent parallel I/O operations.


## Concurrency and Threading

```
- **`Channel<T>`**: Use `System.Threading.Channels.Channel<T>` for producer-consumer patterns (e.g., packet queuing, event processing). Avoid `ConcurrentQueue<T>` with polling loops.
```

- Apply backpressure strategies when using unbounded channels.
- Use lock-free structures or `ReaderWriterLockSlim` for shared game state when appropriate.


## Database and Network

- Always use `AsNoTracking()` for read-only Entity Framework queries.
- Identify and resolve N+1 query issues using `Include()` or batch queries.
- Use pagination or cursor-based streaming for large datasets.
- Minimize transaction scopes. **Never** perform external network I/O (HTTP, Redis) inside a DB transaction.
- Reuse `HttpClient` via `IHttpClientFactory`.


## Caching Strategy

- Use `IMemoryCache` for highly frequent, rarely changing data.
- Use Redis distributed caching for multi-instance environments.
- Use clear, namespace-based cache keys (e.g., `player:session:{playerId}`).
- Always set an explicit TTL. Never allow infinite caching.


## Observability and Logging

- Use `ILogger<T>` with structured logging (message templates), not string interpolation.
- Include searchable correlation fields (e.g., `PlayerId`, `MatchId`).
- Use `LoggerMessage.Define()` to cache delegates in high-throughput hot paths.
- Propagate `Activity` or `X-Correlation-Id` across service boundaries (gRPC/HTTP) for distributed tracing.
- Implement Graceful Shutdown hooks to ensure game states are saved and connections are safely drained before process termination.


## Testing Guidelines

- Unit test all business logic, focusing primarily on the Domain and Service layers.
- Isolate external dependencies using Mocks or Fakes.
- Use `WebApplicationFactory<T>` or Testcontainers for integration tests.
- Naming format: `[MethodName]_[StateUnderTest]_[ExpectedBehavior]` (e.g., `GetPlayerSession_WhenExpired_ReturnsNull`).


## Anti-Patterns to Avoid

- Magic numbers (use named constants).
- `.Result` or `.Wait()` (causes thread starvation and deadlocks).
- Polling `ConcurrentQueue<T>` (wastes CPU cycles).
- Allocating temporary byte arrays with `new byte[]` in hot paths.
- External API/Redis calls within a database transaction scope.
