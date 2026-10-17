---
type: concept
---

# TanStack Query Under the Hood

## A mechanism-first mental model of caching, synchronization, observers, state machines, retries, invalidation, and React integration

---

# 0. What are we actually trying to understand?

When you write:

```tsx
const {
  data,
  isLoading,
  isFetching,
  error,
} = useQuery({
  queryKey: ["todos"],
  queryFn: fetchTodos,
});
```

it looks almost suspiciously simple.

Somehow TanStack Query:

- calls your API
    
- stores the result
    
- remembers it after a component unmounts
    
- gives that cached data to another component
    
- knows when that data is old
    
- refetches it
    
- retries failures
    
- deduplicates simultaneous requests
    
- garbage-collects unused data
    
- reacts to browser focus
    
- reacts to network reconnection
    
- cancels requests
    
- avoids unnecessary React renders
    
- lets mutations update/invalidate queries
    
- handles background fetching without removing old data
    
- supports SSR/hydration
    
- supports persistence
    
- supports offline mutations
    

None of this is React magic.

Underneath, TanStack Query is mostly built from very ordinary programming concepts:

```text
Map
Set
Promise
Observer pattern
Pub/Sub
State machines
setTimeout
setInterval
AbortController
Proxy
JSON.stringify
timestamps
React.useSyncExternalStore
```

The impressive part is how those simple mechanisms are composed.

The most important mental model is:

```text
             TanStack Query Core
                    │
        ┌───────────┼────────────┐
        │           │            │
      Query       Cache       Observers
        │           │            │
        └───────────┼────────────┘
                    │
             React adapter
                    │
            useSyncExternalStore
                    │
                 React
```

React is not the database.

React is not the cache.

React is not responsible for fetching.

React is essentially a **subscriber** to an external state-management system.

That is the foundation of everything that follows.

---

# 1. TanStack Query is not really a fetching library

A common mental model is:

```text
TanStack Query = easier fetch()
```

That is incomplete.

A much better model is:

```text
TanStack Query
=
asynchronous server-state synchronization engine
```

The HTTP request is almost the least interesting part.

TanStack Query does not even care whether you use:

```text
fetch
axios
GraphQL
Supabase
Firebase
IndexedDB
WebSocket-backed promise
custom RPC
```

Your `queryFn` just has to return a Promise.

TanStack Query manages everything **around that Promise**.

```text
            Your server
                │
                ▼
             Promise
                │
                ▼
        ┌──────────────────┐
        │ TanStack Query   │
        │                  │
        │ cache            │
        │ status           │
        │ retries          │
        │ timestamps       │
        │ observers        │
        │ cancellation     │
        │ invalidation     │
        │ garbage collect. │
        └────────┬─────────┘
                 │
                 ▼
               React
```

That distinction matters.

TanStack Query is mostly concerned with this question:

> "What is the best currently known local representation of some remotely owned data, and when should I attempt to synchronize it again?"

That is server-state management.

---

# 2. Server state is fundamentally different from normal React state

Imagine:

```tsx
const [theme, setTheme] = useState("dark");
```

Your application owns `theme`.

But imagine:

```tsx
useQuery({
  queryKey: ["users", userId],
  queryFn: () => fetchUser(userId),
});
```

Your frontend does **not** really own that user.

The backend/database owns it.

Your frontend merely has a snapshot.

```text
DATABASE

User #42
name = "Alice"
      │
      │ HTTP response
      ▼
FRONTEND CACHE

User #42
name = "Alice"
```

Five seconds later somebody could change the database:

```text
DATABASE

User #42
name = "Alice Smith"
```

Your frontend still has:

```text
User #42
name = "Alice"
```

Therefore server state has concepts normal local state does not naturally have:

```text
freshness
staleness
revalidation
refetching
deduplication
cache lifetime
network failures
retries
invalidation
synchronization
```

That is the problem TanStack Query is solving.

---

# 3. The architecture

At the center of TanStack Query are several objects.

You should know these names extremely well:

```text
QueryClient
    │
    ├── QueryCache
    │       │
    │       ├── Query A
    │       ├── Query B
    │       └── Query C
    │
    └── MutationCache
            │
            ├── Mutation A
            └── Mutation B
```

Then components connect to queries using observers:

```text
Component
   │
   ▼
QueryObserver
   │
   ▼
Query
   │
   ▼
QueryCache
```

Multiple components can observe the same Query:

```text
Component A ──► QueryObserver A ─┐
                                │
Component B ──► QueryObserver B ─┼──► Query ["todos"]
                                │
Component C ──► QueryObserver C ─┘
```

This is one of the deepest ideas in TanStack Query.

The cached query exists independently of any specific component.

---

# 4. `QueryClient`: the coordinator

You usually create:

```ts
const queryClient = new QueryClient();
```

You can think of `QueryClient` as the public control panel for the whole system.

Internally it owns things such as:

```text
QueryClient
│
├── QueryCache
├── MutationCache
├── query defaults
├── mutation defaults
├── focus subscriptions
└── online subscriptions
```

The source constructs a `QueryCache` and `MutationCache` when you don't provide custom ones. When mounted, the client also subscribes to focus and connectivity managers, which is how window focus and network reconnection eventually propagate into cache/query behavior.

When you call:

```ts
queryClient.invalidateQueries(...)
queryClient.setQueryData(...)
queryClient.cancelQueries(...)
queryClient.getQueryData(...)
queryClient.prefetchQuery(...)
```

you are mostly telling the coordinator:

> Find the relevant Query objects and perform this operation on them.

So:

```text
queryClient.invalidateQueries({
  queryKey: ["todos"]
})
```

conceptually becomes:

```text
QueryClient
    │
    ▼
QueryCache.findAll(["todos"])
    │
    ├── Query ["todos"]
    ├── Query ["todos", { page: 1 }]
    └── Query ["todos", { page: 2 }]
            │
            ▼
        mark invalid
            │
            ▼
      optionally refetch
```

The actual implementation searches matching cached queries, marks each invalidated, and normally refetches matching active queries.

---

# 5. `QueryCache`: where queries actually live

This is the first big under-the-hood revelation.

Conceptually, the query cache is basically:

```ts
Map<QueryHash, Query>
```

The real `QueryCache` uses a query store backed by a JavaScript `Map`. When it builds a query, it computes a hash, checks whether a Query with that hash already exists, and creates one only when necessary.

A simplified version looks like:

```ts
class QueryCache {
  queries = new Map<string, Query>();

  get(hash: string) {
    return this.queries.get(hash);
  }

  add(query: Query) {
    this.queries.set(query.hash, query);
  }

  remove(query: Query) {
    this.queries.delete(query.hash);
  }
}
```

Suppose your application has:

```tsx
useQuery({
  queryKey: ["todos"],
});

useQuery({
  queryKey: ["users", 42],
});

useQuery({
  queryKey: ["posts", { page: 1 }],
});
```

The cache may conceptually contain:

```text
QueryCache
│
├── '"todos" hash'
│      └── Query(["todos"])
│
├── '"users",42 hash'
│      └── Query(["users", 42])
│
└── '"posts",{page:1} hash'
       └── Query(["posts", { page: 1 }])
```

Important:

## The cache does not merely contain data.

It contains **Query objects**.

Each Query contains:

```text
data
error
status
fetch status
timestamps
failure count
invalidation state
query key
query hash
options
observers
current retry/fetch machinery
```

So this idea:

```ts
cache["todos"] = todos;
```

is too simplistic.

It is closer to:

```ts
cache["todos"] = {
  queryKey: ["todos"],

  state: {
    data: [...],
    dataUpdatedAt: 1788561200000,
    error: null,
    status: "success",
    fetchStatus: "idle",
    isInvalidated: false,
    fetchFailureCount: 0,
    ...
  },

  observers: [...],

  retryer: ...
};
```

That is why TanStack Query can do so much more than simply save API responses.

---

# 6. Query keys become cache identities

Consider:

```ts
["todos"]
```

and:

```ts
["todos", { status: "completed", page: 1 }]
```

TanStack Query needs a usable cache identity.

JavaScript cannot directly use two separately created arrays as equivalent Map keys:

```ts
["todos"] === ["todos"]
// false
```

So TanStack Query produces a stable hash.

The v5 core uses deterministic JSON serialization where plain-object keys are sorted before serialization. That means object property order does not change the resulting identity, while array position still matters.

Conceptually:

```ts
function hashKey(key) {
  return JSON.stringify(key, (_, value) => {
    if (isPlainObject(value)) {
      return Object.keys(value)
        .sort()
        .reduce((obj, key) => {
          obj[key] = value[key];
          return obj;
        }, {});
    }

    return value;
  });
}
```

Therefore:

```ts
["todos", { page: 1, status: "done" }]
```

and:

```ts
["todos", { status: "done", page: 1 }]
```

can identify the same query.

But:

```ts
["todos", "done", 1]
```

and:

```ts
["todos", 1, "done"]
```

do not.

Array order is meaningful.

---

# 7. A `Query` is the real heart of TanStack Query

A Query represents:

> One independently cacheable piece of asynchronous server state.

For example:

```ts
["user", 123]
```

has one Query object.

Conceptually:

```ts
class Query {
  queryKey;
  queryHash;

  state;

  observers = [];

  retryer;

  fetch() {}
  invalidate() {}
  setData() {}
  addObserver() {}
  removeObserver() {}
}
```

The real Query state includes fields such as:

```ts
{
  data,
  dataUpdateCount,
  dataUpdatedAt,

  error,
  errorUpdateCount,
  errorUpdatedAt,

  fetchFailureCount,
  fetchFailureReason,

  isInvalidated,

  status,
  fetchStatus
}
```

The Query implementation updates these through reducer-like actions including:

```text
fetch
success
error
failed
invalidate
pause
continue
setState
```

In other words, it behaves very much like a small state machine.

---

# 8. The most important state-machine concept

TanStack Query separates:

```text
status
```

from:

```text
fetchStatus
```

This is extremely important.

## `status`

This answers approximately:

> What do we currently know about the result?

Typical values:

```text
pending
success
error
```

## `fetchStatus`

This answers:

> What is the network/fetch machinery currently doing?

Typical values:

```text
idle
fetching
paused
```

These are deliberately independent.

For example:

```text
status      = success
fetchStatus = fetching
```

means:

> We already have valid cached data, but we're currently refreshing it in the background.

That is why this is possible:

```tsx
data !== undefined
isSuccess === true
isFetching === true
```

Visualize it:

```text
Initial load

data: undefined
status: pending
fetchStatus: fetching

         │
         │ request succeeds
         ▼

data: [A, B, C]
status: success
fetchStatus: idle

         │
         │ background refetch
         ▼

data: [A, B, C]
status: success
fetchStatus: fetching

         │
         │ new response
         ▼

data: [A, B, C, D]
status: success
fetchStatus: idle
```

TanStack does **not** throw away good cached data just because another request started.

This is what enables stale-while-revalidate behavior.

---

# 9. The observer pattern: how React gets updates

Consider:

```tsx
function TodoList() {
  const query = useQuery({
    queryKey: ["todos"],
    queryFn: fetchTodos,
  });

  return ...
}
```

What actually connects `TodoList` to the cached Query?

A:

```text
QueryObserver
```

TanStack Query has a generic `Subscribable` base class whose core mechanism is simply a `Set` of listeners with `subscribe()` returning an unsubscribe function.

At the conceptual level:

```ts
class Subscribable {
  listeners = new Set();

  subscribe(listener) {
    this.listeners.add(listener);

    return () => {
      this.listeners.delete(listener);
    };
  }
}
```

Then:

```text
Query
  │
  ▼
QueryObserver
  │
  ▼
listener
  │
  ▼
React
```

If three components use the same query:

```tsx
<Navbar />
<Dashboard />
<Profile />
```

and all do:

```ts
useQuery({
  queryKey: ["user", userId],
});
```

you don't necessarily get three independent Query objects.

You get something like:

```text
                 Query ["user", 42]
                /        |         \
               /         |          \
              ▼          ▼           ▼
       Observer A   Observer B   Observer C
           │            │           │
           ▼            ▼           ▼
        Navbar      Dashboard     Profile
```

One piece of cache state.

Multiple subscribers.

---

# 10. `useQuery` itself is surprisingly thin

This is useful to see.

Conceptually, the React implementation is basically:

```ts
function useQuery(options) {
  return useBaseQuery(
    options,
    QueryObserver
  );
}
```

That is very close to the real architecture: the React `useQuery` adapter delegates to a shared base-query implementation with `QueryObserver`.

Most of the interesting logic isn't in:

```text
useQuery.ts
```

It is in:

```text
query-core/
```

especially:

```text
QueryClient
QueryCache
Query
QueryObserver
Mutation
MutationCache
Retryer
NotifyManager
FocusManager
OnlineManager
TimeoutManager
```

That is why TanStack Query can support multiple frameworks.

```text
                @tanstack/query-core
                        │
          ┌─────────────┼─────────────┐
          │             │             │
          ▼             ▼             ▼
        React          Vue          Solid
```

The cache engine does not fundamentally depend on React.

---

# 11. How React subscribes: `useSyncExternalStore`

React has a standard API for connecting React components to state that lives **outside React**:

```ts
useSyncExternalStore()
```

TanStack Query uses exactly this kind of integration.

The React adapter:

1. creates a `QueryObserver`
    
2. asks it for the current result
    
3. subscribes React to the observer
    
4. React calls the provided snapshot function
    
5. when the observer announces a change, React checks the snapshot and rerenders as necessary
    

The current React adapter uses `React.useSyncExternalStore` and wraps observer notifications through TanStack's notification manager.

Conceptually:

```ts
const observer = new QueryObserver(
  queryClient,
  options
);

const result = useSyncExternalStore(
  observer.subscribe,
  observer.getCurrentResult,
  observer.getCurrentResult
);
```

The real implementation has much more logic, but that is the underlying shape.

This means:

```text
React state
```

does not hold the query cache.

Instead:

```text
External TanStack Query state
        │
        │ subscribe
        ▼
     React
```

This is a pattern worth learning independently of TanStack Query.

---

# 12. Full lifecycle: first `useQuery`

Let's follow one request all the way through.

You write:

```tsx
const query = useQuery({
  queryKey: ["users", 42],
  queryFn: () => fetch("/api/users/42").then(r => r.json()),
});
```

## Step 1 — React calls `useQuery`

```text
useQuery(...)
```

delegates toward:

```text
useBaseQuery(...)
```

---

## Step 2 — options are normalized

TanStack merges:

```text
global defaults
+
query defaults
+
local options
```

into a final configuration.

---

## Step 3 — query key is hashed

```ts
["users", 42]
```

becomes some deterministic hash.

Conceptually:

```text
'["users",42]'
```

---

## Step 4 — QueryCache is checked

```text
QueryCache
    │
    │ get(hash)
    ▼
undefined
```

There is no existing query.

So the cache creates one.

```text
QueryCache

'["users",42]'
      │
      ▼
Query
```

---

## Step 5 — QueryObserver is created

```text
React component
      │
      ▼
QueryObserver
      │
      ▼
Query ["users",42]
```

The observer attaches itself to the Query.

The actual `QueryObserver` adds itself on first subscription and decides whether a fetch should occur on mount.

---

## Step 6 — Query determines whether fetching is necessary

Maybe:

```text
data doesn't exist
```

therefore:

```text
fetch required
```

The query state transitions toward:

```text
status = pending
fetchStatus = fetching
```

---

## Step 7 — your `queryFn` executes

TanStack calls something conceptually like:

```ts
queryFn({
  queryKey,
  signal,
  client,
  meta
});
```

Your function returns:

```text
Promise<User>
```

---

## Step 8 — Query holds the in-flight Promise machinery

The Query keeps retry/fetch machinery around the running operation.

This detail becomes extremely important for request deduplication.

---

## Step 9 — Promise resolves

Suppose:

```json
{
  "id": 42,
  "name": "Alice"
}
```

comes back.

TanStack dispatches a successful state transition.

Conceptually:

```ts
state = {
  ...state,

  data: user,
  dataUpdatedAt: Date.now(),

  error: null,

  status: "success",
  fetchStatus: "idle",

  isInvalidated: false
};
```

---

## Step 10 — observers are notified

```text
Query updated
    │
    ▼
QueryObserver
    │
    ▼
notify listener
    │
    ▼
React's external-store subscription
    │
    ▼
component renders
```

Result:

```tsx
query.data
```

now contains Alice.

---

# 13. Request deduplication

This is another major mechanism.

Imagine:

```tsx
<ComponentA />
<ComponentB />
<ComponentC />
```

All mount almost simultaneously.

All use:

```ts
queryKey: ["todos"]
```

Without deduplication:

```text
A ─────► GET /todos
B ─────► GET /todos
C ─────► GET /todos
```

Three requests.

TanStack Query instead identifies all three as the same Query.

```text
A ─┐
B ─┼──► Query ["todos"]
C ─┘
           │
           ▼
      one fetch Promise
           │
           ▼
      GET /todos
```

The key implementation idea is straightforward.

When a Query is already fetching, subsequent fetch attempts can return the existing in-flight retryer's Promise rather than start another request. The Query source explicitly contains this path.

Simplified:

```ts
class Query {
  currentPromise;

  fetch() {
    if (this.currentPromise) {
      return this.currentPromise;
    }

    this.currentPromise = queryFn();

    return this.currentPromise;
  }
}
```

That is promise sharing.

You don't need magical HTTP interception.

You need:

```text
stable query identity
+
central Query object
+
stored in-flight Promise
```

---

# 14. Freshness: `staleTime`

Suppose the server responds at:

```text
12:00:00
```

TanStack stores approximately:

```ts
dataUpdatedAt = timestamp;
```

Then you configure:

```ts
staleTime: 60_000
```

Meaning:

```text
fresh for 60 seconds
```

The idea is essentially:

```ts
isStale =
  Date.now() >= dataUpdatedAt + staleTime;
```

More precisely, the Query considers missing data stale, invalidated data stale, and otherwise uses `dataUpdatedAt` plus the resolved stale time to determine freshness.

Timeline:

```text
12:00:00        fetched
    │
    ├──────────── fresh
    │
    │
12:00:59        fresh
    │
12:01:00        stale
    │
    ▼
```

Extremely important:

# Stale does NOT mean deleted.

And:

# Stale does NOT mean unusable.

Stale means roughly:

> "TanStack is allowed to consider synchronization useful again."

The old value stays in the cache.

---

# 15. Default `staleTime = 0`

By default, cached query data is considered stale immediately after it is fetched.

That sounds strange until you understand the philosophy.

It does **not** mean:

```text
fetch continuously forever
```

It means:

```text
if an important synchronization trigger occurs,
this data is eligible for refetch.
```

Important triggers include things like:

```text
new observer mounts
window refocuses
network reconnects
manual invalidation/refetch
configured polling
```

The current defaults also keep inactive queries around for five minutes by default before garbage collection.

---

# 16. Stale-while-revalidate

This is arguably the most important UX principle in TanStack Query.

Suppose the cache contains:

```text
Todos = [A, B, C]
```

but the data is stale.

A component mounts.

A naive fetching implementation might do:

```text
remove existing data
show spinner
fetch
show result
```

TanStack can instead do:

```text
show cached [A,B,C]
        │
        ├──────── background request
        │
        ▼
server returns [A,B,C,D]
        │
        ▼
replace/update cache
        │
        ▼
UI becomes [A,B,C,D]
```

So the user gets immediate UI.

This is why:

```ts
isFetching
```

and:

```ts
isPending
```

represent different ideas.

You may already have data while fetching.

---

# 17. Active vs inactive queries

Suppose:

```tsx
function TodosPage() {
  useQuery({
    queryKey: ["todos"],
    queryFn: fetchTodos,
  });
}
```

While the page is mounted:

```text
Query ["todos"]
observers = 1

ACTIVE
```

Navigate away.

The observer disconnects:

```text
Query ["todos"]
observers = 0

INACTIVE
```

TanStack doesn't immediately delete it.

That would defeat caching.

Instead:

```text
active
  │
  │ component unmount
  ▼
inactive
  │
  │ gcTime passes
  ▼
garbage collected
```

---

# 18. `gcTime`: cache lifetime after inactivity

Suppose:

```ts
gcTime: 5 * 60 * 1000
```

Then:

```text
0:00  component unmounts

Query becomes inactive

0:01
cache still exists

0:03
cache still exists

0:04:59
cache still exists

0:05:00
eligible for removal
```

Internally this is literally timer-based.

The removable abstraction schedules a timeout through TanStack's timeout manager; when the timer expires, `optionalRemove()` determines whether the object can actually be removed. The default client-side query GC time is five minutes.

Simplified:

```ts
function scheduleGc() {
  gcTimer = setTimeout(() => {
    if (observers.length === 0) {
      queryCache.remove(this);
    }
  }, gcTime);
}
```

When another observer connects:

```text
cancel GC timer
```

because the Query became active again.

---

# 19. `staleTime` vs `gcTime`

These solve completely different problems.

## staleTime

Question:

> How long should cached data be considered fresh?

## gcTime

Question:

> How long should an unused query remain in memory?

Example:

```ts
staleTime: 60_000
gcTime: 300_000
```

Timeline:

```text
fetch
│
├──────────── 60 sec ──────────────► stale
│
│
│ cached and usable
│
├───────────────────────────────
│
component unmounts
│
└──────────── 5 minutes ───────────► removed
```

You can have:

```text
stale but cached
```

which is extremely normal.

---

# 20. Staleness is mostly timestamp arithmetic

A useful insight:

TanStack Query does not need some constantly running background daemon asking:

```text
is it stale yet?
is it stale yet?
is it stale yet?
```

Freshness can mostly be derived from:

```text
dataUpdatedAt
+
staleTime
+
isInvalidated
+
current time
```

When observers need their displayed stale result to flip at the appropriate moment, `QueryObserver` schedules a stale timeout for the remaining duration.

This is efficient.

Instead of constantly updating state:

```text
fresh → stale
```

the system can calculate or schedule the transition when necessary.

---

# 21. Query invalidation

Imagine:

```text
GET /todos
```

returns:

```text
[A, B, C]
```

Then:

```text
POST /todos
```

creates:

```text
D
```

Your cache still contains:

```text
[A, B, C]
```

TanStack Query cannot magically inspect your database.

Therefore you tell it:

```ts
queryClient.invalidateQueries({
  queryKey: ["todos"],
});
```

Internally, invalidation means roughly:

```text
find matching Queries
        │
        ▼
isInvalidated = true
        │
        ▼
consider stale
        │
        ▼
refetch matching active queries
```

The source makes invalidation explicit Query state rather than deleting the cached value.

This distinction matters.

Invalidation means:

```text
"I no longer trust the freshness of this snapshot."
```

It does not mean:

```text
"destroy it immediately."
```

---

# 22. Why invalidation is often better than manual cache synchronization

Suppose:

```ts
createTodo()
```

succeeds.

You could manually reconstruct every affected cached query:

```text
["todos"]
["todos", {status:"open"}]
["dashboard"]
["todo-count"]
["recent-todos"]
```

But this becomes difficult.

Instead:

```text
mutation
   │
   ▼
server becomes source of truth
   │
   ▼
invalidate related queries
   │
   ▼
refetch
```

That is a synchronization strategy.

TanStack Query is largely built around:

```text
cache snapshot
+
invalidation
+
revalidation
```

rather than pretending the frontend owns remote truth.

---

# 23. Automatic refetch on focus

Suppose:

```text
9:00
open app

9:05
switch to another browser tab

9:20
return to app
```

During those fifteen minutes, server data may have changed.

TanStack Query tracks focus/visibility state through a focus manager. The QueryClient subscribes to focus changes; when focus returns, the QueryCache forwards that signal to its queries, and relevant observers decide whether they should refetch.

Conceptually:

```text
browser visibility event
        │
        ▼
FocusManager
        │
        ▼
QueryClient
        │
        ▼
QueryCache.onFocus()
        │
        ▼
Query.onFocus()
        │
        ▼
Observer says:
"stale + refetchOnWindowFocus?"
        │
        ▼
fetch
```

This is not React doing anything.

It is browser events feeding an external state system.

---

# 24. Network reconnection

A similar system exists for connectivity.

Imagine:

```text
online
  │
  ▼
offline
  │
request cannot continue normally
  │
  ▼
paused
  │
network restored
  ▼
online
  │
  ▼
resume/refetch where appropriate
```

`QueryClient.mount()` subscribes to an online manager. When connectivity returns it can resume paused mutations and notify the query cache that the system is online again.

This explains:

```text
fetchStatus = paused
```

It isn't the same thing as:

```text
status = error
```

Again:

```text
result state
```

and:

```text
transport state
```

are separate concepts.

---

# 25. Retry system

Suppose your API returns:

```text
500
```

or the request rejects.

Instead of immediately entering permanent error state, queries can retry.

Conceptually:

```text
attempt 1
   │
   └── fail
        │
        ▼
      delay
        │
attempt 2
   │
   └── fail
        │
        ▼
      delay
        │
attempt 3
   │
   └── success
```

TanStack Query has an internal Retryer abstraction responsible for things such as:

```text
running promise
retry count
retry delay
pause
continue
cancel
network-mode behavior
```

The Query constructs a retryer around the query function rather than directly treating your Promise as the entire lifecycle.

A simplified model:

```ts
async function retry(fn, maxRetries) {
  let failures = 0;

  while (true) {
    try {
      return await fn();
    } catch (error) {
      failures++;

      if (failures > maxRetries) {
        throw error;
      }

      await delay(calculateDelay(failures));
    }
  }
}
```

The real implementation also considers:

```text
focus
online state
network mode
cancellation
pause/resume
```

---

# 26. Retry state is observable state

While retries happen, the Query can track:

```text
fetchFailureCount
fetchFailureReason
```

This is useful because:

```text
failure during retry
```

is different from:

```text
final query error
```

You can think of it like:

```text
attempt failed
     │
     ├── retry possible
     │      │
     │      ▼
     │   continue fetching
     │
     └── retries exhausted
            │
            ▼
        status = error
```

Again, state machine design.

---

# 27. Cancellation uses `AbortController`

When TanStack starts a query, it creates an:

```ts
AbortController
```

and exposes:

```ts
signal
```

to your query function.

Example:

```ts
queryFn: async ({ signal }) => {
  const response = await fetch("/api/todos", {
    signal,
  });

  return response.json();
};
```

Then:

```text
TanStack Query
    │
    ├── AbortController
    │       │
    │       └── AbortSignal
    │              │
    │              ▼
    │           fetch()
    │
    └── cancel()
            │
            ▼
     controller.abort()
```

The Query source creates an `AbortController` around fetch execution. One subtle optimization is that TanStack even tracks whether the supplied signal was actually consumed.

The documented behavior is nuanced: if an unused query's Promise is not genuinely cancelled, it can still resolve and populate the cache; if you consume the AbortSignal and cancellation occurs, the Query can revert state to the state before that fetch.

That is much smarter than simply:

```text
component unmounted → throw result away
```

---

# 28. Query state updates resemble Redux internally

A simplified Query might look like:

```ts
dispatch({
  type: "fetch"
});
```

Then:

```ts
switch (action.type) {
  case "fetch":
    return {
      ...state,
      fetchStatus: "fetching",
    };

  case "success":
    return {
      ...state,
      data: action.data,
      status: "success",
      fetchStatus: "idle",
    };

  case "error":
    return {
      ...state,
      error: action.error,
      status: "error",
      fetchStatus: "idle",
    };

  case "invalidate":
    return {
      ...state,
      isInvalidated: true,
    };
}
```

That is very close conceptually to the Query's reducer-style dispatch machinery.

This gives TanStack Query predictable transitions.

Instead of many unrelated booleans getting modified randomly:

```text
isLoading = ...
isFetching = ...
isError = ...
```

most public booleans can be derived from a smaller internal state machine.

---

# 29. Notifications are batched

Imagine one successful fetch modifies:

```text
data
status
fetchStatus
dataUpdatedAt
failureCount
```

If TanStack notified React separately after every tiny internal change:

```text
notification
render

notification
render

notification
render
```

performance would be terrible.

TanStack therefore has a:

```text
notifyManager
```

which queues and batches notifications.

The notification manager maintains transaction depth and a queue, then schedules grouped notification delivery rather than necessarily executing every callback immediately.

Conceptually:

```ts
batch(() => {
  updateQuery();
  notifyObserverA();
  notifyObserverB();
  notifyCache();
});
```

becomes:

```text
many internal changes
      │
      ▼
 notification queue
      │
      ▼
   one scheduled flush
      │
      ▼
 notify subscribers
```

This is another classic systems pattern:

```text
buffer events
→ batch
→ flush
```

---

# 30. Avoiding unnecessary renders: tracked properties

Suppose:

```tsx
const query = useQuery(...);

return <div>{query.data?.name}</div>;
```

You only read:

```text
data
```

Maybe later:

```text
isFetching changes
```

Should your component necessarily rerender?

TanStack Query performs access tracking.

One mechanism used by `QueryObserver` is JavaScript:

```ts
Proxy
```

The observer can wrap its result and record which properties are accessed. The source implementation literally creates a `Proxy` whose `get` handler records the property.

Simplified:

```ts
const trackedProps = new Set();

const result = new Proxy(realResult, {
  get(target, property) {
    trackedProps.add(property);

    return target[property];
  }
});
```

If your component accesses:

```ts
query.data
query.error
```

TanStack learns:

```text
trackedProps = {
  data,
  error
}
```

Then if only some unrelated result property changes, the observer can avoid notifying React unnecessarily.

That is a fairly sophisticated optimization built from one ordinary JavaScript feature:

```text
Proxy
```

---

# 31. Why object rest destructuring can matter

Consider:

```ts
const { data } = useQuery(...);
```

You accessed:

```text
data
```

Now consider patterns that touch every property.

If your JavaScript operation enumerates all properties, the Proxy observes those property reads.

That can defeat fine-grained tracking because now TanStack thinks:

```text
you depend on almost everything
```

The deeper principle is:

> Access tracking only works if the library can accurately observe what the consumer actually accesses.

This is why understanding the Proxy mechanism is more useful than memorizing lint rules.

---

# 32. Structural sharing

Suppose the old response is:

```ts
const oldData = {
  user: {
    id: 1,
    name: "Alice"
  },
  preferences: {
    theme: "dark"
  }
};
```

A refetch produces:

```ts
const newData = {
  user: {
    id: 1,
    name: "Alice Smith"
  },
  preferences: {
    theme: "dark"
  }
};
```

Normally every nested object from JSON parsing is newly allocated:

```ts
oldData.preferences !== newData.preferences
```

even though the contents are identical.

TanStack Query performs structural sharing.

The idea is:

```text
oldData
│
├── user A
└── preferences A

new network response
│
├── user B         changed
└── preferences B  equal contents
```

becomes:

```text
result
│
├── user B
└── preferences A
```

The unchanged old reference gets reused.

The docs describe this behavior as preserving unchanged references, and the core includes a `replaceEqualDeep` utility that recursively reuses equal branches of JSON-compatible structures.

Why?

Because this:

```ts
oldPreferences === newPreferences
```

can remain:

```text
true
```

which helps:

```text
useMemo
memo
useEffect dependencies
selectors
reference comparisons
React rendering
```

---

# 33. Structural sharing simplified

Imagine:

```ts
function structuralShare(oldValue, newValue) {
  if (deepEqual(oldValue, newValue)) {
    return oldValue;
  }

  if (
    isObject(oldValue) &&
    isObject(newValue)
  ) {
    const result = {};

    for (const key of Object.keys(newValue)) {
      result[key] = structuralShare(
        oldValue[key],
        newValue[key]
      );
    }

    return result;
  }

  return newValue;
}
```

The real implementation is optimized, but conceptually this is the trick.

It doesn't merely answer:

```text
equal / not equal
```

It reconstructs the new value while preserving unchanged references.

---

# 34. `select` happens at the observer level

Consider:

```tsx
useQuery({
  queryKey: ["users"],
  queryFn: fetchUsers,
  select: users => users.length,
});
```

The server response/cache might hold:

```ts
[
  { id: 1 },
  { id: 2 },
  { id: 3 }
]
```

while your component sees:

```ts
3
```

An important architectural distinction is:

```text
Query cache data
```

versus:

```text
Observer result
```

Think:

```text
Query
cached raw-ish data
       │
       ▼
QueryObserver
       │
       ├── select()
       ├── placeholder processing
       ├── status derivation
       ├── tracked props
       └── result shaping
       │
       ▼
Component
```

That means two components can observe the same Query differently.

```tsx
Component A:
select: users => users.length

Component B:
select: users => users.filter(...)

Component C:
no select
```

All share one Query.

```text
                    Query ["users"]
                    cached users
                    /    |    \
                   /     |     \
                  ▼      ▼      ▼
              Observer Observer Observer
                 │       │       │
               count   filtered full
```

This is a powerful separation.

---

# 35. Query state and observer state are not identical

This is another advanced mental model.

The Query owns core server-state information:

```text
cached data
network state
error
timestamps
invalidation
```

But an observer has its own concerns:

```text
select
placeholderData
staleTime options
enabled
refetch interval
tracked properties
derived booleans
```

Therefore:

```text
Query state
≠
exact object returned from useQuery()
```

Instead:

```text
Query state
      │
      ▼
QueryObserver
      │
      ▼
derived QueryObserverResult
      │
      ▼
useQuery()
```

---

# 36. Different components can use different `staleTime`

Suppose both observe:

```text
["users"]
```

but:

```tsx
Component A:
staleTime = 5 seconds

Component B:
staleTime = 5 minutes
```

There is one underlying Query data object.

But freshness can be evaluated in the context of observer options.

This explains why the architecture needs:

```text
Query
+
QueryObserver
```

instead of storing every behavior directly on the Query.

---

# 37. Refetch intervals use ordinary timers

When you configure polling:

```ts
refetchInterval: 5000
```

there isn't some mysterious scheduler.

QueryObserver manages timers through TanStack's timeout manager.

Conceptually:

```ts
setInterval(() => {
  query.fetch();
}, 5000);
```

The timeout manager centralizes timers used by features including stale timing, garbage collection, retries, throttling, and intervals.

The abstraction exists partly so timer behavior can be customized and managed consistently.

---

# 38. `setQueryData`

Suppose you call:

```ts
queryClient.setQueryData(
  ["todo", 123],
  newTodo
);
```

No HTTP request is needed.

The flow is approximately:

```text
QueryClient
    │
    ▼
find/build Query
    │
    ▼
Query.setData()
    │
    ▼
structural sharing
    │
    ▼
dispatch success-like state update
    │
    ▼
notify observers
    │
    ▼
React rerenders
```

So TanStack Query can be viewed partly as an external reactive store.

But its specialty is remotely owned asynchronous state.

---

# 39. Mutations use a parallel architecture

Queries are primarily about:

```text
read / synchronize
```

Mutations are about:

```text
write / side effect
```

Their architecture mirrors queries:

```text
Query side

QueryClient
   │
QueryCache
   │
Query
   │
QueryObserver


Mutation side

QueryClient
   │
MutationCache
   │
Mutation
   │
MutationObserver
```

A Mutation has its own state including things such as:

```text
status
data
error
variables
submittedAt
failureCount
isPaused
context
```

and also uses the retryer machinery.

---

# 40. Mutation state machine

Conceptually:

```text
idle
  │
mutate()
  ▼
pending
  │
  ├──────── success ───────► success
  │
  └──────── failure ───────► retry/error
```

A simplified state:

```ts
{
  status: "pending",
  variables: {
    title: "Learn Query"
  },
  error: null,
  data: undefined
}
```

Then:

```text
server succeeds
```

becomes:

```ts
{
  status: "success",
  variables: {...},
  data: createdTodo
}
```

---

# 41. Why mutations aren't cached like queries

A query represents an identity such as:

```text
["todo", 42]
```

It makes sense to say:

```text
What is the cached value of Todo #42?
```

A mutation is different.

```text
POST /todos
```

is an event:

```text
Create Todo A
Create Todo B
Create Todo C
```

Those executions are separate operations.

So mutation state is centered around mutation executions rather than treating them as one canonical server-state snapshot.

---

# 42. Mutation concurrency and scopes

By default, mutation executions can run in parallel.

```text
Mutation A ───► server
Mutation B ───► server
Mutation C ───► server
```

TanStack also supports mutation scopes that can serialize mutations sharing a scope id:

```text
Mutation A ───► server ───► complete
                                │
Mutation B ───── paused ────────┘
                  │
                  ▼
                server
```

The documented implementation queues same-scope mutations by pausing later ones until their turn.

This becomes useful when write ordering matters.

---

# 43. Optimistic updates

Suppose the user creates:

```text
Todo D
```

Without optimism:

```text
click
 │
 ▼
spinner
 │
 ▼
server
 │
 ▼
response
 │
 ▼
update UI
```

With optimistic updating:

```text
click
 │
 ├────────► update cache immediately
 │                 │
 │                 ▼
 │           UI shows Todo D
 │
 └────────► server request
                   │
             ┌─────┴─────┐
             │           │
          success      failure
             │           │
           keep       rollback
```

The important mechanisms are not specific to React.

They are:

```text
snapshot previous cache
modify cache
run mutation
if failure → restore previous snapshot
```

Conceptually:

```ts
const previous = queryClient.getQueryData(["todos"]);

queryClient.setQueryData(
  ["todos"],
  old => [...old, optimisticTodo]
);

try {
  await createTodo();
} catch {
  queryClient.setQueryData(
    ["todos"],
    previous
  );
}
```

That is transaction-like behavior at the client cache level.

Not a real database transaction, of course—but the mental structure is similar:

```text
before state
tentative state
commit / compensate
```

---

# 44. Important warning about optimistic updates

An optimistic UI does not guarantee that the server accepted the operation.

Your frontend might temporarily believe:

```text
balance = $500
```

while the server rejects the mutation.

Therefore optimistic updates require a recovery strategy.

Usually:

```text
snapshot
optimistic write
server request
rollback on error
invalidate/refetch after settlement
```

The final refetch lets the real server state win.

That matters a lot for financial applications.

---

# 45. Infinite queries are still Queries

Consider:

```ts
useInfiniteQuery(...)
```

It might look like a completely separate mechanism.

It isn't.

At a high level it still uses the same Query infrastructure but adds specialized fetching behavior.

The Query's fetch path can install infinite-query behavior when the query is marked as infinite.

Instead of cached data being:

```ts
Todo[]
```

it looks more like:

```ts
{
  pages: [
    page1,
    page2,
    page3
  ],

  pageParams: [
    initialParam,
    param2,
    param3
  ]
}
```

Conceptually:

```text
Query
│
├── page 1
├── page 2
└── page 3
```

Still:

```text
one query identity
one Query object
observer(s)
state machine
cache
```

but with special pagination behavior.

---

# 46. Placeholder data is not the same as cache data

Suppose:

```ts
placeholderData: fakeTodos
```

The observer may temporarily display:

```text
fakeTodos
```

while the real Query does not yet have successful fetched data.

Architecturally this makes sense because placeholder handling belongs to:

```text
observer result construction
```

rather than necessarily becoming canonical query-cache data.

That distinction is important:

```text
What component currently sees
```

does not always equal:

```text
what canonical Query data exists in the cache
```

---

# 47. Initial data is conceptually different

Initial data can actually initialize query state.

So these concepts differ:

```text
placeholderData
```

→ presentation-level temporary stand-in.

```text
initialData
```

→ actual initial cached data state.

This distinction becomes obvious once you understand:

```text
Observer
vs
Query
```

---

# 48. Prefetching

Suppose the user is hovering over:

```text
/products/42
```

You predict they may navigate there.

Prefetching means:

```text
create/find Query
      │
      ▼
execute queryFn
      │
      ▼
store data in QueryCache
```

without requiring a mounted component.

Then later:

```tsx
useQuery({
  queryKey: ["product", 42],
});
```

runs.

The cache already has it.

```text
prefetch
   │
   ▼
QueryCache
   │
   ▼
later component mount
   │
   ▼
instant cached value
```

This is another proof that the cache does not belong to components.

---

# 49. Cache sharing across components

Suppose:

```tsx
function Sidebar() {
  useQuery({
    queryKey: ["me"],
  });
}

function AccountPage() {
  useQuery({
    queryKey: ["me"],
  });
}
```

Both ultimately discover:

```text
same query hash
```

and therefore:

```text
same Query
```

So TanStack Query is not:

```text
hook state
hook state
hook state
```

It is:

```text
                 QueryCache
                     │
                     ▼
                Query ["me"]
                 /        \
                ▼          ▼
          Observer A   Observer B
              │            │
              ▼            ▼
          Sidebar      AccountPage
```

This is closer to an external normalized-by-key store than independent hooks.

---

# 50. What happens when a second component mounts while data is fresh?

Suppose:

```text
12:00
Component A fetches users

staleTime = 5 min
```

At:

```text
12:01
```

Component B mounts with the same Query.

Flow:

```text
Component B
    │
    ▼
QueryObserver B
    │
    ▼
QueryCache lookup
    │
    ▼
existing Query found
    │
    ▼
data exists
    │
    ▼
still fresh
    │
    ▼
return cached data
```

No new request needed.

---

# 51. What if the second component mounts when the cache is stale?

Suppose:

```text
12:10
```

Now data is stale.

Then:

```text
Component B
    │
    ▼
existing stale Query
    │
    ├────────► return cached data
    │
    └────────► background fetch
```

That is the characteristic TanStack Query experience.

Immediate data plus background synchronization.

---

# 52. Persistence is separate from the normal cache

Very important misconception:

```text
TanStack Query cache = localStorage
```

No.

Normally, QueryCache is in-memory JavaScript state.

Refresh the page:

```text
JavaScript runtime dies
      │
      ▼
memory disappears
      │
      ▼
cache disappears
```

Persistence is an additional layer.

Conceptually:

```text
QueryCache
    │
    ▼
dehydrate/serialize
    │
    ▼
localStorage / AsyncStorage / other storage
```

Then later:

```text
storage
   │
   ▼
deserialize
   │
   ▼
hydrate
   │
   ▼
QueryCache
```

So:

```text
caching
```

and:

```text
persistent caching
```

are separate concepts.

---

# 53. SSR dehydration/hydration

SSR introduces two JavaScript environments:

```text
SERVER
and
BROWSER
```

Server:

```text
QueryClient
    │
    ▼
prefetch
    │
    ▼
QueryCache
    │
    ▼
dehydrate
    │
    ▼
serializable state
```

HTML/data travels to browser.

Browser:

```text
serialized state
      │
      ▼
hydrate
      │
      ▼
browser QueryClient
      │
      ▼
browser QueryCache
```

This avoids throwing away server-fetched server state.

Freshness still uses update timestamps; the docs explicitly note that staleness after hydration is based on when the server query was fetched.

Think of hydration as:

> Reconstructing cache objects from serialized cache state.

---

# 54. Why server and client QueryClients must be treated carefully

On the browser, one QueryClient generally needs to survive normal React rerenders.

Bad mental model:

```tsx
function App() {
  const client = new QueryClient();
}
```

if this recreates the cache constantly.

That would mean:

```text
render
→ new QueryClient
→ new QueryCache

render
→ another QueryClient
→ another QueryCache
```

You would continuously discard state.

The core principle:

```text
cache identity depends on QueryClient lifetime
```

The QueryClient is effectively the owner of that cache universe.

---

# 55. Window-focus refetch does not directly call every API

It works through layers.

```text
visibility changes
      │
      ▼
FocusManager
      │
      ▼
QueryClient
      │
      ▼
QueryCache
      │
      ▼
Queries
      │
      ▼
Observers inspect their options
      │
      ▼
eligible query refetches
```

This separation is good architecture.

The focus manager does not need to understand:

```text
todos
users
products
React components
```

It only emits:

```text
focused = true
```

Then higher-level objects decide what that means.

---

# 56. OnlineManager follows the same philosophy

```text
browser/network event
      │
      ▼
OnlineManager
      │
      ▼
QueryClient
      │
      ▼
Queries / Mutations
      │
      ▼
resume/revalidate
```

This is an example of **decoupled event-driven architecture**.

Each subsystem has one responsibility.

---

# 57. Why QueryObserver exists instead of Components subscribing directly to Query

At first you might ask:

> Why not just make React subscribe directly to Query?

Because each consumer can have different options.

Example:

```text
Component A

select: user => user.name
staleTime: 10 sec
enabled: true
```

```text
Component B

select: user => user.avatar
staleTime: 5 min
enabled: true
```

Same Query:

```text
["user",42]
```

Different views of it.

So:

```text
          Query
         /     \
        /       \
Observer A     Observer B
   │              │
select name    select avatar
   │              │
Component A    Component B
```

That is why the observer layer is essential.

---

# 58. The actual data flow you should memorize

Memorize this:

```text
useQuery()
   │
   ▼
useBaseQuery()
   │
   ▼
QueryObserver
   │
   ▼
QueryCache
   │
   ▼
Query
   │
   ▼
queryFn()
   │
   ▼
Promise
   │
   ▼
Retryer
   │
   ▼
Query state update
   │
   ▼
notifyManager
   │
   ▼
QueryObserver
   │
   ▼
useSyncExternalStore
   │
   ▼
React render
```

That is TanStack Query in one diagram.

---

# 59. The caching flow you should memorize

```text
queryKey
   │
   ▼
stable hash
   │
   ▼
QueryCache Map
   │
   ├── Query exists?
   │       │
   │       ├── yes
   │       │    │
   │       │    ├── fresh → use cache
   │       │    │
   │       │    └── stale → use cache + maybe refetch
   │       │
   │       └── no
   │            │
   │            ▼
   │        create Query
   │            │
   │            ▼
   │          fetch
   │
   ▼
cached Query
```

---

# 60. The lifetime flow you should memorize

```text
Query created
    │
    ▼
observer attached
    │
    ▼
ACTIVE
    │
    │ observer removed
    ▼
INACTIVE
    │
    │ gcTime
    ▼
garbage collected
```

If another observer appears before GC:

```text
INACTIVE
    │
    │ observer added
    ▼
ACTIVE

GC timer cancelled
```

---

# 61. The freshness flow you should memorize

```text
response arrives
     │
     ▼
dataUpdatedAt = now
     │
     ▼
FRESH
     │
     │ staleTime expires
     ▼
STALE
     │
     │ sync trigger
     ▼
background refetch
     │
     ▼
new data
     │
     ▼
FRESH
```

---

# 62. The mutation flow you should memorize

```text
mutate()
   │
   ▼
Mutation created/executed
   │
   ▼
status = pending
   │
   ▼
mutationFn()
   │
   ▼
Promise
   │
   ├──────── success
   │             │
   │             ▼
   │        status = success
   │
   └──────── error
                 │
                 ├── retry?
                 │
                 └── status = error
```

Then commonly:

```text
mutation success
     │
     ▼
invalidate Query
     │
     ▼
refetch
```

---

# 63. A miniature TanStack Query implementation

If you really want to understand the library, imagine writing this yourself.

This is intentionally simplified.

```ts
type Listener = () => void;

type QueryState<T> = {
  data?: T;
  error?: unknown;
  status: "pending" | "success" | "error";
  fetchStatus: "idle" | "fetching";
  updatedAt: number;
};

class Query<T> {
  state: QueryState<T> = {
    status: "pending",
    fetchStatus: "idle",
    updatedAt: 0,
  };

  listeners = new Set<Listener>();

  promise?: Promise<T>;

  gcTimer?: ReturnType<typeof setTimeout>;

  constructor(
    public key: unknown[],
    public queryFn: () => Promise<T>,
  ) {}

  subscribe(listener: Listener) {
    this.listeners.add(listener);

    if (this.gcTimer) {
      clearTimeout(this.gcTimer);
    }

    return () => {
      this.listeners.delete(listener);

      if (this.listeners.size === 0) {
        this.scheduleGc();
      }
    };
  }

  notify() {
    for (const listener of this.listeners) {
      listener();
    }
  }

  isStale(staleTime: number) {
    return (
      Date.now() >
      this.state.updatedAt + staleTime
    );
  }

  async fetch() {
    // Promise deduplication
    if (this.promise) {
      return this.promise;
    }

    this.state = {
      ...this.state,
      fetchStatus: "fetching",
    };

    this.notify();

    this.promise = this.queryFn();

    try {
      const data = await this.promise;

      this.state = {
        data,
        error: undefined,
        status: "success",
        fetchStatus: "idle",
        updatedAt: Date.now(),
      };

      this.notify();

      return data;
    } catch (error) {
      this.state = {
        ...this.state,
        error,
        status: "error",
        fetchStatus: "idle",
      };

      this.notify();

      throw error;
    } finally {
      this.promise = undefined;
    }
  }

  scheduleGc() {
    this.gcTimer = setTimeout(() => {
      // cache would remove this query
    }, 5 * 60 * 1000);
  }
}
```

Then build a cache:

```ts
class QueryCache {
  queries = new Map<string, Query<any>>();

  get(key: unknown[]) {
    return this.queries.get(
      JSON.stringify(key)
    );
  }

  build<T>(
    key: unknown[],
    queryFn: () => Promise<T>
  ) {
    const hash = JSON.stringify(key);

    let query = this.queries.get(hash);

    if (!query) {
      query = new Query(
        key,
        queryFn
      );

      this.queries.set(
        hash,
        query
      );
    }

    return query;
  }
}
```

Then:

```ts
class QueryClient {
  cache = new QueryCache();
}
```

And finally React integration:

```tsx
function useMyQuery({
  queryKey,
  queryFn,
  staleTime = 0,
}) {
  const client = useQueryClient();

  const query = client.cache.build(
    queryKey,
    queryFn
  );

  const state = useSyncExternalStore(
    callback => query.subscribe(callback),

    () => query.state,

    () => query.state
  );

  useEffect(() => {
    if (
      state.data === undefined ||
      query.isStale(staleTime)
    ) {
      query.fetch();
    }
  }, [query]);

  return state;
}
```

Congratulations.

You now have the skeleton of React Query.

The production implementation adds:

```text
better hashing
observers
selections
tracked properties
structural sharing
invalidation
query matching
focus events
online events
retrying
network modes
cancellation
garbage collection
refetch intervals
infinite queries
mutations
SSR
hydration
persistence
optimistic results
Suspense
error boundaries
batched notifications
lots of edge cases
```

But the core mental model is already there.

---

# 64. The technologies behind TanStack Query

You specifically asked what technologies/mechanisms make TanStack Query work.

Here they are.

## 1. JavaScript `Map`

Used to associate query hashes with Query objects.

```text
hash → Query
```

---

## 2. JavaScript `Set`

Useful for subscriber/listener collections.

```text
Observer listeners
Cache listeners
Manager listeners
```

---

## 3. Promises

Every async operation revolves around Promise lifecycle.

```text
pending
fulfilled
rejected
```

TanStack adds state, retry, sharing, and synchronization around those Promises.

---

## 4. Promise sharing

One active Promise can service multiple consumers.

```text
multiple observers
      │
      ▼
one Query
      │
      ▼
one running Promise
```

---

## 5. Observer / Pub-Sub pattern

```text
Query changes
   │
   ▼
Observers notified
   │
   ▼
Subscribers notified
```

---

## 6. State machines / reducers

Query and Mutation state transitions are modeled with explicit actions.

```text
fetch
success
error
pause
continue
invalidate
```

---

## 7. Timestamps

Used heavily for freshness.

```text
dataUpdatedAt
errorUpdatedAt
submittedAt
```

---

## 8. `setTimeout`

Used for things such as:

```text
stale transitions
garbage collection
retry delays
```

---

## 9. `setInterval`

Used for polling/refetch intervals.

---

## 10. `AbortController` / `AbortSignal`

Used for cooperative request cancellation.

---

## 11. JavaScript `Proxy`

Used to track which returned query-result properties consumers access.

---

## 12. Stable serialization / hashing

Used to map complex query keys onto stable cache identities.

---

## 13. Structural sharing

Used to preserve references for unchanged portions of new data.

---

## 14. Browser visibility/focus APIs

Feed the focus-management system.

---

## 15. Browser network signals

Feed the online-management system.

---

## 16. `React.useSyncExternalStore`

Connects TanStack's external cache/observer state into React's rendering system.

---

## 17. Event batching

`notifyManager` groups notifications to reduce redundant work.

---

## 18. Serialization / hydration

Used for SSR and cache persistence.

---

# 65. TanStack Query is basically several computer-science patterns assembled together

You can map it like this:

|TanStack feature|Fundamental concept|
|---|---|
|QueryCache|Hash map|
|queryKey|Cache identity|
|Query|State machine|
|QueryObserver|Observer pattern|
|React updates|External-store subscription|
|deduplication|Shared Promise|
|staleTime|Timestamp arithmetic|
|gcTime|Timed garbage collection|
|retries|Retry state machine + timers|
|cancellation|AbortController|
|tracked properties|Proxy|
|structural sharing|Persistent-data-style reference reuse|
|focus refetch|Event-driven architecture|
|reconnect|Pub/Sub + environment event|
|invalidation|Cache coherency signal|
|optimistic update|speculative state + compensation|
|SSR hydration|serialization/deserialization|
|polling|interval scheduling|
|batching|event queue|

This is why learning TanStack Query deeply teaches you more than TanStack Query.

It teaches:

```text
caching
state machines
event-driven design
observer architecture
async coordination
cache invalidation
concurrency
reactivity
lifecycle management
```

---

# 66. TanStack Query and Redux are architecturally more similar than they first appear

Not identical.

But compare.

Redux:

```text
Store
  │
  ▼
state
  │
  ▼
subscribe()
  │
  ▼
React
```

TanStack Query:

```text
QueryCache
  │
  ▼
Queries
  │
  ▼
Observers
  │
  ▼
subscribe()
  │
  ▼
React
```

The difference is TanStack Query is specialized around:

```text
remote asynchronous state
```

and therefore understands:

```text
Promise
freshness
retry
invalidation
network status
refetching
cache lifetime
```

Redux does not inherently understand those concepts.

---

# 67. TanStack Query is not a database

The cache may make it feel like one.

But it lacks database guarantees such as:

```text
durability
transactions
cross-client consistency
authoritative ownership
constraints
ACID guarantees
```

Your server/database remains the authority.

Think:

```text
PostgreSQL
     │
     │ authoritative state
     ▼
Backend API
     │
     │ snapshot
     ▼
TanStack Query Cache
     │
     │ synchronized approximation
     ▼
React UI
```

This distinction is crucial in production.

---

# 68. TanStack Query does not make your backend consistent

Suppose two users submit conflicting writes.

TanStack Query cannot solve:

```text
database race conditions
transaction isolation
lost updates
authorization
idempotency
server concurrency
```

Those remain backend concerns.

TanStack Query solves:

```text
client-side server-state synchronization
```

Don't move server correctness responsibilities into the client cache.

---

# 69. Cache coherence is the real hard problem

Fetching is easy:

```ts
await fetch(...)
```

The difficult question is:

> How do I know whether my current value still represents the server?

TanStack Query gives you multiple strategies:

```text
time-based freshness
manual invalidation
focus revalidation
reconnect revalidation
polling
mutation-triggered invalidation
explicit cache writes
```

All of those are solutions to:

```text
cache coherence
```

This is the deeper topic behind TanStack Query.

---

# 70. Why staleTime should reflect your domain

Consider stock prices.

```text
staleTime = 30 minutes
```

probably makes little sense.

Consider:

```text
country list
```

Thirty minutes may be unnecessarily short.

Therefore `staleTime` isn't merely:

```text
performance tuning
```

It is a business/domain decision:

> How long can this information be considered sufficiently current?

Examples:

```text
stock price
seconds

chat messages
seconds

bank balance
perhaps short

user profile
minutes

product catalog
minutes

country list
hours/days/Infinity

static config
very long / static
```

This is how a senior engineer should think about it.

---

# 71. Query keys are part of your application's data model

Bad mindset:

```ts
["whatever"]
```

Better mindset:

```text
query keys describe remote-state identity
```

For example:

```ts
["transactions"]
```

```ts
["transactions", {
  accountId,
  month,
  category
}]
```

```ts
["transaction", transactionId]
```

You are effectively designing:

```text
cache address space
```

This matters because it controls:

```text
deduplication
sharing
invalidation
refetching
cache boundaries
```

Poor keys create incorrect cache behavior.

---

# 72. Query keys act like dependencies

Consider:

```ts
queryFn: () =>
  fetchTransactions(accountId)
```

If:

```text
accountId
```

changes the requested data, it belongs in the query identity.

```ts
queryKey: [
  "transactions",
  accountId
]
```

Why?

Because these are different server-state resources:

```text
transactions for account 1
transactions for account 2
```

Without the variable in the key, TanStack has no principled way to distinguish those snapshots.

---

# 73. Why "server state" should usually not be copied into `useState`

Bad:

```tsx
const { data } = useQuery(...);

const [users, setUsers] =
  useState(data);
```

Now you created:

```text
TanStack Query cache
        │
        ▼
copy
        │
        ▼
React state
```

You have two sources of truth.

```text
Query cache changed

users state did not
```

Now you need synchronization logic.

Usually you want:

```text
server state
        │
        ▼
Query cache
        │
        ▼
derived UI
```

not duplicated ownership.

---

# 74. A useful production architecture

Think in layers.

```text
DATABASE
   │
   ▼
BACKEND
   │
   ▼
API DTO
   │
   ▼
queryFn
   │
   ▼
TanStack Query
   │
   ▼
Query cache
   │
   ▼
select / domain transformation
   │
   ▼
React UI
```

For your engineering mental model:

```text
server owns truth

TanStack owns cached server-state synchronization

React owns presentation/client interaction state
```

For example:

```text
Server state:
transactions
accounts
categories
user profile

TanStack Query

Client state:
modal open
selected tab
input text
dragging state

React/Zustand/etc.
```

Different jobs.

---

# 75. The five objects you absolutely must remember

If everything in this document disappears from your memory except five things, remember:

## `QueryClient`

Coordinator.

```text
"Manage my query universe."
```

---

## `QueryCache`

Storage/index of Query objects.

```text
queryHash → Query
```

---

## `Query`

Canonical cached asynchronous resource and its state machine.

```text
["transactions", accountId]
```

---

## `QueryObserver`

Consumer-specific view/subscription around a Query.

```text
Query → React component
```

---

## `Mutation`

State machine representing a write operation.

---

# 76. The one diagram I want burned into your brain

```text
                         SERVER
                           │
                           │
                      queryFn Promise
                           │
                           ▼
                    ┌─────────────┐
                    │   Query     │
                    │             │
                    │ data        │
                    │ error       │
                    │ timestamps  │
                    │ status      │
                    │ fetchStatus │
                    │ retryer     │
                    └──────┬──────┘
                           │
                    stored inside
                           │
                           ▼
                    ┌─────────────┐
                    │ QueryCache  │
                    │    Map      │
                    └──────┬──────┘
                           │
                     owned by
                           │
                           ▼
                    ┌─────────────┐
                    │ QueryClient │
                    └─────────────┘


      React Component A               React Component B
             │                               │
             ▼                               ▼
      QueryObserver A                 QueryObserver B
             │                               │
             └──────────────┬────────────────┘
                            │
                            ▼
                         Query
                            │
                            ▼
                    notifyManager
                            │
                            ▼
                 useSyncExternalStore
                            │
                            ▼
                          React
```

That is the architecture.

---

# 77. The deepest mental model

TanStack Query can be reduced to this:

```text
1. Give remote data a stable identity.

2. Represent that identity with a long-lived Query object.

3. Store Query objects in a cache.

4. Let consumers observe Query objects.

5. Wrap asynchronous requests in a state machine.

6. Store timestamps describing when successful data arrived.

7. Determine when cached snapshots are no longer sufficiently fresh.

8. Revalidate snapshots when useful events occur.

9. Share in-flight work between consumers.

10. Notify consumers efficiently when meaningful state changes.

11. Remove unused state after an inactivity period.

12. Keep the server as the ultimate source of truth.
```

Everything else is refinement.

---

# 78. Final comparison: what you write vs what actually happens

You write:

```tsx
const {
  data,
  isPending,
  isFetching,
} = useQuery({
  queryKey: ["transactions"],
  queryFn: fetchTransactions,
});
```

Your mental model should now expand that into:

```text
useQuery()
   │
   ▼
React Query adapter
   │
   ▼
create/reuse QueryObserver
   │
   ▼
normalize options
   │
   ▼
hash ["transactions"]
   │
   ▼
QueryCache.get(hash)
   │
   ├── not found
   │      │
   │      ▼
   │   create Query
   │
   ▼
Observer subscribes
   │
   ▼
Query determines whether fetch required
   │
   ▼
create AbortController
   │
   ▼
create Retryer
   │
   ▼
execute queryFn
   │
   ▼
store in-flight Promise
   │
   ▼
request succeeds
   │
   ▼
structural sharing
   │
   ▼
dispatch success
   │
   ├── data
   ├── updatedAt
   ├── status = success
   ├── fetchStatus = idle
   └── invalidation reset
   │
   ▼
notifyManager batches notifications
   │
   ▼
QueryObserver builds new result
   │
   ├── apply select
   ├── derive flags
   ├── track properties
   └── evaluate staleness
   │
   ▼
useSyncExternalStore notified
   │
   ▼
React rerenders
```

Later:

```text
component unmounts
   │
   ▼
observer removed
   │
   ▼
0 observers
   │
   ▼
Query becomes inactive
   │
   ▼
GC timer starts
   │
   ├── another observer appears
   │      │
   │      └── cancel GC
   │
   └── gcTime expires
          │
          ▼
      Query removed
```

Meanwhile freshness follows:

```text
dataUpdatedAt
     │
     ▼
fresh
     │
staleTime
     ▼
stale
     │
     ├── mount
     ├── focus
     ├── reconnect
     ├── invalidate
     └── refetch
            │
            ▼
     background fetch
            │
            ▼
      fresh data again
```

If you understand these three flows—

```text
request flow
cache lifecycle
observer/render flow
```

—you understand the architecture of TanStack Query at a level far beyond simply knowing how to call `useQuery`.

---

# 79. What to study next to truly master it

TanStack Query becomes much easier once you independently understand these underlying topics:

```text
1. Observer / Pub-Sub pattern
2. JavaScript Map and Set
3. Promises and Promise sharing
4. State machines
5. AbortController
6. Event loop + timers
7. useSyncExternalStore
8. Proxy
9. Structural sharing / referential equality
10. Cache invalidation
11. Cache coherence
12. optimistic concurrency
13. stale-while-revalidate
14. serialization / hydration
15. garbage collection and object lifetimes
```

Those aren't merely "TanStack Query concepts."

They are general software-engineering concepts.

And once you recognize them, TanStack Query stops looking magical.

You start seeing:

```text
"Oh.

It's a Map of stateful Query objects.

Queries expose observable state.

Observers translate Query state into consumer-specific results.

React subscribes through useSyncExternalStore.

Promises are shared for deduplication.

Timestamps determine freshness.

Timers handle stale boundaries, polling, retries, and GC.

AbortController handles cancellation.

Events handle focus/connectivity.

Proxy tracks consumed properties.

Structural sharing preserves references.

Invalidation tells the cache its knowledge may be outdated."
```

That is the under-the-hood model.