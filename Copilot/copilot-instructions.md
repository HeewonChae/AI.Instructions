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

---

# AI Guidelines for Code Generation

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.

