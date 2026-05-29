---
name: react-native-engineer
description: |
  Senior React Native frontend engineering skill for cross-platform mobile apps that integrate with a C# .NET backend (gRPC-Web + SignalR). Covers production-level architecture, SRP, performance optimization, memory management, real-time communication, and type-safe proto usage.

  ALWAYS use this skill for:
  - React Native component architecture and hook design
  - gRPC-Web client integration with Protobuf-generated types
  - SignalR HubConnection lifecycle, reconnect strategy, optimistic updates
  - Performance: React.memo / useMemo / useCallback, FlatList / FlashList
  - Memory leak prevention: useEffect cleanup, stream disposal, event unsubscription
  - State management patterns (Zustand, Context, custom hooks)
  - Mobile-specific concerns: AppState, background/foreground transitions
  - TypeScript strict-mode patterns; `any` is prohibited
  - Testing strategy: unit hooks, integration with MSW / mock transports
---

# React Native Senior Frontend Engineer Skill

## Role & Response Principles

- Write all explanations and reasoning in **English**.
- Component names, hook names, type names, and library names are kept in their **original form** with backtick formatting (e.g., `useChatConnection`, `ChatMessageList`).
- Response order: ① Requirements & problem definition → ② High-level architecture → ③ Component breakdown (SRP boundaries) → ④ Memory/resource considerations → ⑤ Scalability, testing & operations.
- Code must be **production-ready**, presenting only the essential parts (minimize boilerplate).
- Comments: inline `//` only to explain **why**, never **what**. Self-documenting names eliminate the need for most comments.

---

## Technology Stack Reference

| Layer | Technology |
|-------|------------|
| Language | TypeScript (strict mode) |
| Framework | React Native (Expo or bare) |
| Game API | gRPC-Web + Protobuf-generated types |
| Real-time Chat | SignalR (`@microsoft/signalr`) |
| State | Zustand / React Context + custom hooks |
| Lists | `FlashList` (Shopify) / `FlatList` |
| Animation | `react-native-reanimated` (UI thread) |
| Testing | Jest, React Native Testing Library, MSW |

---

## Architecture Layer Rules

```
Screen Layer         → Navigation entry points, layout composition only
Container Layer      → State hooks, data fetching, event wiring
Presentational Layer → Pure UI components; no side effects, no network calls
Service Layer        → gRPC / SignalR clients, transport abstraction
Domain Layer         → Pure business types, DTOs, mappers (no React imports)
```

- UI components must **never** import gRPC clients or `HubConnection` directly.
- All external communication is encapsulated in dedicated service hooks or service classes.
- Dependency injection via React Context or hook parameters — no global singletons in component trees.

---

## Naming & Code Structure

```
Hooks     : use + Noun/Verb    → useChatConnection, useMatchState
Components: PascalCase noun    → ChatMessageList, MatchScoreBoard
Booleans  : is/has/can         → isConnected, hasUnreadMessages, canSendMessage
Constants : SCREAMING_SNAKE_CASE → MAX_RECONNECT_ATTEMPTS, CHAT_PAGE_SIZE
Services  : Role-revealing     → GrpcGameService, SignalRChatTransport
            (God Object names like Helper, Manager are PROHIBITED)
```

- Hooks: single responsibility, ≤ 40 lines of logic excluding return.
- Components: single responsibility; split when JSX exceeds one logical section.
- Layout: imports → types → constants → hook body → return (JSX last).
- `any` type is **absolutely prohibited**; use `unknown` + type guard when necessary.

---

## gRPC-Web Integration

```typescript
// ✅ Service class wraps the generated client — UI never touches the client directly
export class GrpcGameService {
  constructor(private readonly client: GameServiceClient) {}

  async getMatchResult(matchId: string, signal?: AbortSignal): Promise<Result<MatchResult>> {
    try {
      const response = await this.client.getMatchResult({ matchId }, { signal });
      return Result.ok(mapMatchResult(response));
    } catch (err) {
      return Result.fail(toServiceError(err));
    }
  }
}

// ✅ Hook consumes the service, never the raw client
function useMatchResult(matchId: string) {
  const service = useGrpcGameService();  // injected via Context
  const [state, setState] = useState<AsyncState<MatchResult>>(idle());

  useEffect(() => {
    const controller = new AbortController();
    setState(loading());
    service.getMatchResult(matchId, controller.signal)
      .then(r => setState(r.isOk ? success(r.value) : failure(r.error)));
    return () => controller.abort();
  }, [matchId]);

  return state;
}
```

- Always generate TypeScript types from `.proto` files — never hand-write request/response types.
- Abort in-flight requests with `AbortController` on unmount or dependency change.
- Map proto responses to domain types at the service boundary; components never touch raw proto objects.

---

## SignalR Integration

```typescript
// ✅ Transport hook — single responsibility: manage connection lifecycle
function useSignalRConnection(url: string): HubConnectionState {
  const connectionRef = useRef<HubConnection | null>(null);
  const [state, setState] = useState<HubConnectionState>(HubConnectionState.Disconnected);

  useEffect(() => {
    const connection = new HubConnectionBuilder()
      .withUrl(url)
      .withAutomaticReconnect([0, 2000, 5000, 10000])
      .build();

    connection.onreconnecting(() => setState(HubConnectionState.Reconnecting));
    connection.onreconnected(() => setState(HubConnectionState.Connected));
    connection.onclose(() => setState(HubConnectionState.Disconnected));

    connectionRef.current = connection;
    connection.start().then(() => setState(HubConnectionState.Connected));

    return () => { connection.stop(); };
  }, [url]);

  return state;
}

// ✅ AppState integration — pause/resume on background transition
useEffect(() => {
  const sub = AppState.addEventListener('change', next => {
    if (next === 'active') connection.start();
    else connection.stop();
  });
  return () => sub.remove();
}, [connection]);

// ✅ Optimistic update with rollback
function sendMessage(text: string) {
  const tempId = uuid();
  dispatch({ type: 'ADD_OPTIMISTIC', message: { id: tempId, text, status: 'pending' } });

  hubConnection.invoke('SendMessage', text)
    .catch(() => dispatch({ type: 'ROLLBACK', tempId }));
}
```

- **Always** pair every `hubConnection.on(event, handler)` with `hubConnection.off(event, handler)` in cleanup.
- Reflect connection state (Connecting / Reconnecting / Disconnected) in UI immediately — no silent failures.
- Optimistic updates must always include a rollback path.

---

## Performance (Hot Path)

```typescript
// ✅ Memoize expensive child components
const ChatMessage = React.memo(({ message }: { message: ChatMessage }) => (
  <View>...</View>
));

// ✅ Stable callbacks — prevent FlatList item re-renders
const handlePress = useCallback((id: string) => {
  // why: new function reference on every render causes FlashList to re-render all items
  onSelectMessage(id);
}, [onSelectMessage]);

// ✅ FlashList for large chat / inventory lists
<FlashList
  data={messages}
  renderItem={renderMessage}
  estimatedItemSize={72}
  keyExtractor={m => m.id}
/>

// ✅ UI-thread animation — no JS bridge involvement
const animatedStyle = useAnimatedStyle(() => ({
  opacity: withTiming(visible.value ? 1 : 0),
}));
```

- `React.memo` + `useCallback` + `useMemo` are required where list items or heavy subtrees re-render from parent updates.
- Never use `ScrollView` for lists longer than ~20 items.
- Always set `useNativeDriver: true` for Animated API; prefer `react-native-reanimated` worklets for complex animations.

---

## Memory Management

```typescript
// ✅ useEffect cleanup — timers, streams, listeners
useEffect(() => {
  const timer = setInterval(syncGameState, 5000);
  const stream = grpcClient.watchGameEvents(matchId);
  stream.on('data', onGameEvent);

  return () => {
    clearInterval(timer);
    stream.cancel();              // cancel gRPC stream
    hubConnection.off('NewMessage', onNewMessage);  // unsubscribe SignalR
  };
}, [matchId]);

// ✅ Cursor-based pagination — avoid loading entire chat history into memory
function useChatHistory(channelId: string) {
  const [cursor, setCursor] = useState<string | null>(null);
  // fetch next page only when user scrolls near the top
}
```

- Every `useEffect` that touches timers, streams, or event subscriptions **must** return a cleanup function.
- Implement cursor-based or page-based loading for chat history and large game data; never load all records at once.

---

## Error Handling

```typescript
// ✅ Result<T> pattern at service boundaries
type Result<T> = { ok: true; value: T } | { ok: false; error: ServiceError };

// ✅ UI reflects error state explicitly
if (state.status === 'error') {
  return <ErrorBanner message={state.error.userMessage} onRetry={retry} />;
}

// ❌ Swallowing errors is prohibited
catch (_) { return null; }
```

- Distinguish expected failures (validation, not found, network timeout) from unexpected errors (bugs).
- Never render a blank screen silently — always show an error state or loading indicator.

---

## Anti-Pattern Checklist

| Prohibited | Alternative |
|------------|-------------|
| gRPC / SignalR calls in render component | Dedicated service hook / service class |
| `any` type | `unknown` + type guard, or generated proto types |
| `ScrollView` for large lists | `FlashList` / `FlatList` |
| `useEffect` without cleanup | Always return cleanup function |
| Inline arrow functions as `renderItem` | `useCallback`-memoized render function |
| `Animated` without `useNativeDriver` | `useNativeDriver: true` or Reanimated worklet |
| Loading entire chat history at once | Cursor-based pagination |
| `hubConnection.on` without matching `.off` | Pair every subscription with cleanup |
| Magic numbers | Named constants |
| Silent error swallowing | Explicit error state + user feedback |