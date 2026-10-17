```javascript
class TinyPromise<T> {
  private state: "pending" | "fulfilled" | "rejected" = "pending";

  private value: T | undefined;

  private reason: unknown;

  private successHandlers: Array<(value: T) => void> = [];

  private failureHandlers: Array<(reason: unknown) => void> = [];

  constructor(
    executor: (
      resolve: (value: T) => void,
      reject: (reason: unknown) => void
    ) => void
  ) {
    const resolve = (value: T) => {
      if (this.state !== "pending") return;

      this.state = "fulfilled";
      this.value = value;

      for (const handler of this.successHandlers) {
        queueMicrotask(() => {
          handler(value);
        });
      }
    };

    const reject = (reason: unknown) => {
      if (this.state !== "pending") return;

      this.state = "rejected";
      this.reason = reason;

      for (const handler of this.failureHandlers) {
        queueMicrotask(() => {
          handler(reason);
        });
      }
    };

    executor(resolve, reject);
  }

  then(onFulfilled: (value: T) => void) {
    if (this.state === "pending") {
      this.successHandlers.push(onFulfilled);
    }

    if (this.state === "fulfilled") {
      queueMicrotask(() => {
        onFulfilled(this.value!);
      });
    }
  }

  catch(onRejected: (reason: unknown) => void) {
    if (this.state === "pending") {
      this.failureHandlers.push(onRejected);
    }

    if (this.state === "rejected") {
      queueMicrotask(() => {
        onRejected(this.reason);
      });
    }
  }
}
```



```
Promise.resolve(getUser()).then(user => {
  console.log(user);

  return 123;
});
```

Assume:

```
getUser()
```

starts some network operation and returns:

```
Promise<User>
```

For example, conceptually:

```
function getUser() {
  // Starts network I/O
  // immediately returns a pending Promise
}
```

The important actors are:

```
┌──────────────────── JavaScript Engine ────────────────────┐
│                                                          │
│  Call Stack                                              │
│                                                          │
│  Promise objects / internal reaction records             │
│                                                          │
│  Microtask Queue                                         │
│  ┌───────────────────────────────────────────────────┐   │
│  │ Promise reaction jobs                             │   │
│  └───────────────────────────────────────────────────┘   │
│                                                          │
└──────────────────────────────────────────────────────────┘

┌────────────────── Browser / Host ─────────────────────────┐
│                                                          │
│  Network system                                          │
│  Timers                                                  │
│  DOM events                                              │
│  etc.                                                    │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

One correction before we go deeper: people often draw a single "Web API queue". That's useful for beginners but isn't literally how browsers are implemented.

For our mental model, think:

```
Browser subsystems
    ↓
network/timers/events complete
    ↓
they eventually cause JavaScript jobs/tasks/microtasks
to become runnable
```

Now let's execute your code.

---

# 1. JavaScript first evaluates `getUser()`

JavaScript does **not** start with `Promise.resolve`.

Function arguments must be evaluated first.

So:

```
Promise.resolve(getUser())
                ^^^^^^^^^
                FIRST
```

Call stack:

```
┌──────────── CALL STACK ────────────┐
│ getUser()                          │
│ global/script                     │
└────────────────────────────────────┘
```

Let's say `getUser()` starts a network request:

```
JS
│
│ getUser()
▼
Browser networking system
│
│ GET /api/user
▼
network request in flight
```

At this point the browser doesn't have the user yet.

So `getUser()` creates/returns a Promise.

Let's call it:

```
P1
```

Internally, conceptually:

```
P1
┌────────────────────────────────┐
│ [[PromiseState]] = "pending"   │
│ [[PromiseResult]] = undefined  │
│                                │
│ fulfill reactions = []         │
│ reject reactions  = []         │
└────────────────────────────────┘
```

Meanwhile:

```
Browser Network System
┌──────────────────────────────┐
│ GET /api/user               │
│ status: in flight           │
└──────────────────────────────┘
```

So `getUser()` immediately returns:

```
P1
```

without waiting for the network.

---

# 2. Now `Promise.resolve(P1)` runs

We have effectively reached:

```
Promise.resolve(P1)
```

This part is subtle.

If `P1` is already a normal native Promise created by the same Promise constructor, then:

```
Promise.resolve(P1) === P1
```

is normally:

```
true
```

So `Promise.resolve()` doesn't create another unnecessary Promise here.

Conceptually:

```
Promise.resolve(P1)

        │
        │ "This is already one of my Promises."
        ▼

       return P1
```

So this:

```
Promise.resolve(getUser())
```

has effectively become:

```
P1
```

Therefore your original code is currently equivalent to:

```
P1.then(user => {
  console.log(user);
  return 123;
});
```

---

# 3. `.then()` runs while P1 is pending

Now JavaScript executes:

```
P1.then(handler)
```

where:

```
handler = user => {
  console.log(user);
  return 123;
}
```

Here's where the Promise machinery becomes important.

`.then()` does two major things.

First, it records what should happen when `P1` finishes.

Second, **it creates another Promise**.

Call that Promise:

```
P2
```

So:

```
const P2 = P1.then(handler);
```

Even though you didn't assign it to a variable.

The structure is:

```
P1
│
│ .then(handler)
│
└──────────────────► P2
```

At this moment:

```
P1
┌─────────────────────────────────────┐
│ state: pending                      │
│ result: undefined                   │
│                                     │
│ fulfill reactions:                  │
│                                     │
│   ┌─────────────────────────────┐   │
│   │ handler(user)               │   │
│   │ child promise: P2           │   │
│   └─────────────────────────────┘   │
└─────────────────────────────────────┘


P2
┌───────────────────────────────┐
│ state: pending                │
│ result: undefined             │
└───────────────────────────────┘
```

This part is crucial:

> `.then()` does not just store your callback. It associates that callback with a new Promise whose fate depends on what the callback eventually does.

---

# 4. What are the queues right now?

Suppose the current script has reached the end.

We have:

```
CALL STACK
┌────────────────────────────┐
│ empty                      │
└────────────────────────────┘


MICROTASK QUEUE
┌────────────────────────────┐
│ empty                      │
└────────────────────────────┘


TASK / EVENT SOURCES
┌────────────────────────────┐
│ maybe other timers/events  │
└────────────────────────────┘


NETWORK SYSTEM
┌────────────────────────────┐
│ GET /api/user              │
│                            │
│ still downloading...       │
└────────────────────────────┘
```

And memory contains:

```
P1 = pending
P2 = pending
```

Nothing is continuously polling `P1`.

There is no:

```
while (P1.pending) {}
```

The Promise just sits in memory.

---

# 5. The network finishes

Eventually the browser receives something like:

```
{
  "name": "Takaki"
}
```

The browser's networking/fetch machinery processes the completion.

Conceptually, something eventually causes `P1` to be resolved with:

```
{ name: "Takaki" }
```

Before:

```
P1
state  = pending
result = undefined
```

After settlement:

```
P1
┌────────────────────────────────────────┐
│ state: fulfilled                       │
│                                        │
│ result:                                │
│ {                                      │
│   name: "Takaki"                       │
│ }                                      │
└────────────────────────────────────────┘
```

But remember:

```
P1.fulfillReactions
```

contains our `.then()` handler.

So the Promise machinery sees:

```
"Oh. Someone is waiting for this result."
```

---

# 6. It does NOT call your handler immediately

This is extremely important.

It does not simply do:

```
handler(user);
```

right inside the resolution algorithm.

Instead it creates a **Promise Reaction Job**.

Conceptually:

```
PromiseReactionJob {
    handler: user => {
        console.log(user);
        return 123;
    },

    argument: {
        name: "Takaki"
    },

    childPromise: P2
}
```

That job goes into the microtask queue.

Now the system looks like:

```
P1
┌───────────────────────────────┐
│ fulfilled                     │
│ { name: "Takaki" }            │
└───────────────────────────────┘


P2
┌───────────────────────────────┐
│ pending                       │
└───────────────────────────────┘
```

And:

```
MICROTASK QUEUE
┌─────────────────────────────────────────┐
│ PromiseReactionJob                      │
│                                         │
│ run handler with:                       │
│ { name: "Takaki" }                      │
│                                         │
│ result controls P2                      │
└─────────────────────────────────────────┘
```

Notice:

```
Network operation
        ↓
P1 settlement
        ↓
Promise reaction job
        ↓
microtask queue
```

---

# 7. The event loop reaches the microtask checkpoint

Suppose the currently running JavaScript task has finished.

Call stack:

```
CALL STACK
┌──────────────────┐
│ empty            │
└──────────────────┘
```

The event loop checks the microtask queue.

It finds:

```
PromiseReactionJob
```

So it dequeues it.

Before:

```
MICROTASK QUEUE

┌────────────────────────────┐
│ PromiseReactionJob         │
└────────────────────────────┘
```

After dequeue:

```
MICROTASK QUEUE

┌────────────────────────────┐
│ empty                      │
└────────────────────────────┘
```

The job now invokes your handler.

---

# 8. Your `.then()` callback enters the call stack

Now:

```
user => {
  console.log(user);

  return 123;
}
```

starts executing.

Call stack:

```
┌────────────── CALL STACK ───────────────┐
│ Promise reaction handler               │
│                                        │
│ user = { name: "Takaki" }              │
└─────────────────────────────────────────┘
```

The parameter:

```
user
```

came from:

```
P1.[[PromiseResult]]
```

So:

```
console.log(user);
```

prints:

```
{ name: "Takaki" }
```

---

# 9. Then the callback returns `123`

Now we reach the most important Promise-chaining mechanism:

```
return 123;
```

This is **not** returning from your outer program.

It's returning from the `.then()` handler.

Remember that `.then()` created:

```
P2
```

The Promise machinery watches what the handler does.

The handler returned:

```
123
```

Therefore:

```
P2 fulfills with 123
```

Before:

```
P2
┌─────────────────────────┐
│ state: pending          │
│ result: undefined       │
└─────────────────────────┘
```

After:

```
P2
┌─────────────────────────┐
│ state: fulfilled        │
│ result: 123             │
└─────────────────────────┘
```

So the relationship was:

```
P1 fulfills with User
        │
        ▼
handler(User)
        │
        │ returns 123
        ▼
P2 fulfills with 123
```

---

# 10. Full visualization

Let's put the whole thing together.

Your code:

```
Promise.resolve(getUser()).then(user => {
  console.log(user);

  return 123;
});
```

### Phase 1 — synchronous execution

```
                          JAVASCRIPT

                    ┌─────────────────┐
                    │    Call Stack   │
                    │                 │
                    │    getUser()    │
                    └────────┬────────┘
                             │
                             │ starts operation
                             ▼

                         BROWSER

                    ┌─────────────────┐
                    │ Network System  │
                    │                 │
                    │ GET /api/user   │
                    └─────────────────┘
```

`getUser()` returns:

```
P1
┌────────────────────────┐
│ pending                │
│ result = undefined     │
└────────────────────────┘
```

---

### Phase 2 — `Promise.resolve`

```
Promise.resolve(P1)
        │
        │ P1 already a Promise
        ▼
      returns P1
```

No extra waiting.

---

### Phase 3 — `.then()`

```
                    P1
          ┌───────────────────────┐
          │ state: pending        │
          │                       │
          │ fulfill reaction:     │
          │ handler → P2          │
          └───────────────────────┘
                     │
                     │
                     ▼
                    P2
          ┌───────────────────────┐
          │ state: pending        │
          └───────────────────────┘
```

Network still:

```
Browser Network System

GET /api/user
████████████░░░░░░
```

Microtask queue:

```
[]
```

---

### Phase 4 — network finishes

```
Browser
   │
   │ user data becomes available
   ▼
settle P1
```

Now:

```
P1
┌─────────────────────────────┐
│ fulfilled                   │
│                             │
│ { name: "Takaki" }          │
└─────────────────────────────┘
```

The reaction is scheduled:

```
MICROTASK QUEUE
┌──────────────────────────────────┐
│ PromiseReactionJob               │
│                                  │
│ handler({ name: "Takaki" })      │
│ child = P2                       │
└──────────────────────────────────┘
```

---

### Phase 5 — microtask executes

```
Microtask Queue
      │
      ▼

Call Stack

┌────────────────────────────────┐
│ handler(user)                  │
│                                │
│ user = {name:"Takaki"}         │
└────────────────────────────────┘
```

Execute:

```
console.log(user);
```

Then:

```
return 123;
```

---

### Phase 6 — return value resolves P2

```
handler
  │
  │ return 123
  ▼

P2
┌──────────────────────┐
│ fulfilled            │
│ result = 123         │
└──────────────────────┘
```

Done.

---

# 11. The hidden P2 is extremely important

Your code doesn't save the result:

```
Promise.resolve(getUser()).then(...)
```

but `.then()` still returns something.

You could write:

```
const p2 = Promise.resolve(getUser()).then(user => {
  console.log(user);

  return 123;
});
```

Later:

```
p2.then(value => {
  console.log(value);
});
```

would print:

```
123
```

Now there would be another Promise:

```
P1
 │
 │ user
 ▼
handler1
 │
 │ return 123
 ▼
P2
 │
 │ 123
 ▼
handler2
 │
 ▼
P3
```

Promises form these dependency chains continuously.

---

# 12. Now let's add another `.then()` to expose the queues

Imagine:

```
Promise.resolve(getUser())
  .then(user => {
    console.log("A", user);

    return 123;
  })
  .then(number => {
    console.log("B", number);

    return 456;
  });
```

Internally:

```
P1
│
│ user
▼
handler A
│
│ 123
▼
P2
│
│ 123
▼
handler B
│
│ 456
▼
P3
```

Initially:

```
P1 pending
P2 pending
P3 pending
```

When `P1` fulfills:

```
MICROTASK QUEUE
┌───────────────────────┐
│ handler A             │
└───────────────────────┘
```

Run A:

```
return 123;
```

That fulfills P2:

```
P2 → fulfilled(123)
```

But P2 already has handler B registered.

Therefore fulfilling P2 schedules **another microtask**:

```
MICROTASK QUEUE
┌───────────────────────┐
│ handler B             │
└───────────────────────┘
```

And because the event loop drains the microtask queue before moving to the next regular task, B can run immediately after A.

This gives you:

```
microtask A
    │
    │ resolves P2
    ▼
enqueue microtask B
    │
    ▼
microtask B
```

That's why long Promise chains can execute many callbacks during one microtask checkpoint.

---

# 13. What happens if `return 123` becomes `return anotherPromise`

Suppose:

```
Promise.resolve(getUser()).then(user => {
  return getPosts(user.id);
});
```

Now the handler returns:

```
P_posts
```

Instead of:

```
123
```

Promise machinery does **not** do:

```
P2 = fulfilled(P_posts)
```

Instead:

```
P2 adopts P_posts
```

Visualization:

```
P1
 │
 │ User
 ▼
handler
 │
 │ returns P_posts
 ▼
┌─────────────────────────────┐
│ P2 waits for P_posts        │
└─────────────────────────────┘

P_posts pending
      │
      │ later...
      ▼
P_posts fulfilled([posts])
      │
      ▼
P2 fulfilled([posts])
```

That automatic adoption is what lets this:

```
getUser()
  .then(getPosts)
  .then(renderPosts);
```

remain flat instead of becoming nested Promise objects.

---

# 14. Where exactly are things physically?

A useful way of organizing your mental model is:

```
┌──────────────────── JS HEAP ─────────────────────┐
│                                                  │
│ P1 Promise object                               │
│ P2 Promise object                               │
│ handler function object                         │
│ user object                                     │
│ reaction records                                │
│                                                  │
└──────────────────────────────────────────────────┘


┌────────────────── CALL STACK ────────────────────┐
│                                                  │
│ functions actively executing RIGHT NOW          │
│                                                  │
└──────────────────────────────────────────────────┘


┌──────────────── MICROTASK QUEUE ─────────────────┐
│                                                  │
│ Promise continuation waiting to execute          │
│                                                  │
└──────────────────────────────────────────────────┘


┌────────────────── TASK QUEUES ───────────────────┐
│                                                  │
│ timers                                           │
│ user interaction                                │
│ various browser tasks                           │
│                                                  │
└──────────────────────────────────────────────────┘


┌──────────────── BROWSER SYSTEMS ─────────────────┐
│                                                  │
│ network stack                                    │
│ timers                                           │
│ rendering                                        │
│ IndexedDB                                        │
│ etc.                                             │
│                                                  │
└──────────────────────────────────────────────────┘
```

The Promise itself is essentially data in memory.

The microtask queue contains **work scheduled to run later**.

The network system handles the actual HTTP work.

The call stack contains only JavaScript executing **right now**.

---

# 15. The most precise way to read your line

When you see:

```
Promise.resolve(getUser()).then(user => {
  console.log(user);
  return 123;
});
```

read it as:

```
1. Execute getUser() NOW.

2. getUser starts an operation and returns some value,
   probably P1.

3. Normalize that value into a Promise using
   Promise.resolve().

4. Register:

       "When P1 fulfills, run this function."

5. Create another Promise P2 representing:

       "the eventual result of running that function."

6. Return P2 immediately.

7. Later, P1 fulfills with User.

8. Queue the registered continuation as a microtask.

9. When JavaScript reaches the microtask checkpoint,
   run:

       handler(User)

10. Handler returns 123.

11. Resolve P2 with 123.
```

Or visually:

```
                              NOW
                               │
                               ▼

getUser()
   │
   ├────────────► Browser network request
   │
   ▼
 P1 pending
   │
   │ Promise.resolve(P1)
   ▼
 P1
   │
   │ .then(handler)
   ├───────────────────────────► P2 pending
   │
   │
   │              ...time...
   │
   │
network finishes
   │
   ▼
P1 fulfilled(User)
   │
   │ schedule reaction
   ▼
┌───────────────────┐
│ Microtask Queue   │
│ handler(User)     │
└─────────┬─────────┘
          │
          ▼
      Call Stack
          │
          │ console.log(User)
          │
          │ return 123
          ▼
    P2 fulfilled(123)
```

The strongest mental model is:

> **A Promise holds state and relationships. The queue holds executable work. The browser handles the actual external operation.**

Those three things are separate.

`P1` does not "sit in the microtask queue." The **Promise object lives in memory**. Only the reaction job created because of its settlement gets placed into the microtask queue.