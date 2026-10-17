---
type: concept
---
1. # ACID
2. # Transactions
3. # Create from scratch

Implementing Atomicity (All-or-Nothing)

Atomicity ensures that all operations in a transaction complete or none do. [[1](https://motherduck.com/learn/acid-transactions-sql/), [2](https://www.youtube.com/watch?v=49FAQkrU7NQ&t=12)]

- **Write-Ahead Logging (WAL):** Record every change in a sequential log on disk before modifying the actual database pages. If a failure occurs before completion, the system uses the log to undo changes (rollback). [[1](https://www.youtube.com/watch?v=GAe5oB742dw), [2](https://dev.to/jaiminbariya/implementation-of-acid-transaction-in-database-44nc), [3](https://medium.com/@rohanpudasaini581/understanding-database-atomicity-the-foundation-of-reliable-transactions-0a11de0de9b4), [4](https://dev.to/ujjwall-r/ensuring-atomicity-in-modern-databases-2e97)]

- **Shadow Paging:** Maintain two tables or page directories during a transaction: a current (old) copy and a shadow (new) copy. Modifications are written only to the shadow copy; the database pointer is swapped to the shadow copy atomically only upon a successful commit. If it fails, the shadow copy is discarded, leaving the original data untouched. [[1](https://studyglance.in/dbms/display.php?tno=39&topic=Implementation-of-Atomicity-and-Durability--in-DBMS), [2](https://www.scribd.com/document/324652023/BScCSIT-Transaction-DBMS), [3](https://dev.to/jaiminbariya/implementation-of-acid-transaction-in-database-44nc)]


Absolutely. The most useful way to understand a **database transaction** is to stop thinking of it as “a block of SQL between `BEGIN` and `COMMIT`” and instead think of it as:

> **A temporary, isolated version of database history that becomes real only if the database can safely commit it.**

Since you work with PostgreSQL through Prisma/Supabase, I’ll use PostgreSQL as the main model. The concepts carry over to MySQL, SQL Server, Oracle, etc., though implementation details differ.

---

# 1. Start with the problem transactions solve

Imagine a banking transfer.

You want to move `$100` from Alice to Bob:

```sql
UPDATE accounts
SET balance = balance - 100
WHERE id = 'alice';

UPDATE accounts
SET balance = balance + 100
WHERE id = 'bob';
```

At first glance, this looks fine.

But what happens if:

```text
1. Alice loses $100
2. Server crashes
3. Bob never receives $100
```

Now your database contains an impossible state.

```text
Alice: $900
Bob:   $500
```

instead of:

```text
Alice: $900
Bob:   $600
```

The database needs a mechanism that says:

```text
Either:

Alice -100
AND
Bob +100

both happen

OR

neither happens.
```

That's a transaction.

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE id = 'alice';

UPDATE accounts
SET balance = balance + 100
WHERE id = 'bob';

COMMIT;
```

If something fails:

```sql
ROLLBACK;
```

But that syntax is the easy part.

The interesting question is:

> How can PostgreSQL undo/redo things, hide partially completed work, handle crashes, and allow hundreds of transactions simultaneously?

That's where WAL, MVCC, locks, snapshots, transaction IDs, and isolation come in.

---

# 2. A transaction is really a state machine

Internally, PostgreSQL effectively tracks something like:

```text
TRANSACTION STARTED
        |
        v
   ACTIVE
        |
   +----+----+
   |         |
   v         v
COMMITTED  ABORTED
```

You can imagine transaction `T42`:

```text
T42:
    UPDATE Alice
    UPDATE Bob
```

While it is running:

```text
T42 = ACTIVE
```

After:

```sql
COMMIT;
```

Postgres records:

```text
T42 = COMMITTED
```

If it fails:

```text
T42 = ABORTED
```

And this status becomes extremely important because rows themselves contain information about **which transaction created or modified them**.

---

# 3. PostgreSQL does not usually "edit a row"

This surprises many developers.

Suppose the table contains:

```text
id      balance
alice   1000
```

You run:

```sql
UPDATE accounts
SET balance = 900
WHERE id = 'alice';
```

You might imagine PostgreSQL doing this:

```text
1000 -> overwrite memory -> 900
```

But PostgreSQL mostly uses **MVCC**:

> Multi-Version Concurrency Control

Instead of modifying the existing row in place, PostgreSQL creates another **row version**.

Conceptually:

```text
Old row version:

alice | 1000
created_by = transaction 10
deleted_by = transaction 42


New row version:

alice | 900
created_by = transaction 42
deleted_by = null
```

You now temporarily have:

```text
Version A:
alice = 1000

Version B:
alice = 900
```

Which one you see depends on your transaction's snapshot.

This is one of the most important concepts in relational databases.

---

# 4. Why keep multiple versions?

Imagine two users.

Transaction A starts:

```sql
BEGIN;

SELECT balance
FROM accounts
WHERE id = 'alice';
```

Returns:

```text
1000
```

Meanwhile Transaction B:

```sql
BEGIN;

UPDATE accounts
SET balance = 900
WHERE id = 'alice';

COMMIT;
```

Depending on the isolation level and snapshot timing, Transaction A may continue seeing:

```text
1000
```

while new transactions see:

```text
900
```

That sounds strange, but it's incredibly useful.

Without MVCC, readers and writers would constantly block each other.

You'd get:

```text
Transaction A reads row
      ↓
locks row
      ↓
Transaction B waits
      ↓
Transaction C waits
      ↓
Transaction D waits
```

Instead PostgreSQL can say:

```text
Transaction A:
"You started when version 1000 was valid."

Transaction B:
"You created a newer version: 900."

New transaction:
"You can see 900."
```

So readers often don't block writers, and writers often don't block readers.

---

# 5. A row has hidden transaction metadata

A PostgreSQL row internally has metadata including fields conceptually like:

```text
xmin
xmax
```

Very roughly:

```text
xmin = transaction that created this row version
xmax = transaction that invalidated/deleted this row version
```

Suppose:

```text
Transaction 10 inserts Alice $1000
```

Row:

```text
balance = 1000
xmin = 10
xmax = 0
```

Later transaction 42 updates Alice.

Old version:

```text
balance = 1000
xmin = 10
xmax = 42
```

New version:

```text
balance = 900
xmin = 42
xmax = 0
```

And PostgreSQL asks:

> Based on my current transaction snapshot, is transaction 42 visible to me?

If yes:

```text
show 900
```

If no:

```text
show 1000
```

This is what MVCC visibility rules are doing.

---

# 6. `COMMIT` is not just "finish"

When you run:

```sql
COMMIT;
```

the database must make a very strong promise:

> If I tell you this transaction committed, I will not forget it even if the machine crashes immediately afterward.

That is the **Durability** part of ACID.

This is where the WAL comes in.

---

# 7. WAL: Write-Ahead Log

PostgreSQL uses a **Write-Ahead Log**.

The fundamental rule is:

> Write the change to the log before relying on the changed database page.

Hence:

```text
Write
Ahead
Log
```

Imagine a database page currently contains:

```text
Alice: 1000
Bob:   500
```

Transaction:

```text
Alice -> 900
Bob   -> 600
```

PostgreSQL does not need to immediately flush the actual table pages to disk.

Instead, it records enough information in WAL:

```text
WAL record:
transaction 42 changed Alice

WAL record:
transaction 42 changed Bob

WAL record:
transaction 42 committed
```

Then WAL is flushed to persistent storage.

Only after that can PostgreSQL safely say:

```text
COMMIT successful
```

---

# 8. Why WAL exists

Writing every modified database page synchronously would be expensive.

Suppose one SQL statement changes:

```text
50 rows
```

spread across:

```text
15 database pages
4 indexes
```

Flushing everything immediately would generate lots of random disk writes.

Instead Postgres can write sequential WAL records:

```text
WAL
↓
append
↓
append
↓
append
↓
fsync
```

Sequential writes are much cheaper.

The dirty database pages can be written later.

---

# 9. Crash recovery

Suppose this happens:

```text
1. Transaction modifies Alice
2. Transaction modifies Bob
3. WAL gets flushed
4. COMMIT succeeds
5. Server crashes
6. Actual table pages weren't fully written
```

After restart, PostgreSQL reads WAL.

It sees:

```text
T42 committed
```

and can replay the changes.

```text
Alice -> 900
Bob -> 600
```

This is **REDO** style recovery.

That's why WAL is fundamental to database durability.

---

# 10. What if PostgreSQL crashes before COMMIT?

Suppose WAL contains:

```text
T42 changed Alice
T42 changed Bob
```

but no valid commit record.

PostgreSQL sees:

```text
T42 never committed
```

Therefore its row versions are considered invisible.

This is important:

PostgreSQL doesn't necessarily need to physically erase every piece of aborted data immediately.

It can simply say:

```text
transaction 42 = aborted

therefore rows created by transaction 42 are invisible
```

Later cleanup can reclaim them.

That cleanup process is part of what `VACUUM` does.

---

# 11. So what does `ROLLBACK` actually do?

Developers often imagine:

```text
UPDATE:
100 -> 90

ROLLBACK:
90 -> 100
```

Not exactly.

With MVCC, rollback can often work more like:

```text
Transaction 42 created a new row version.

Mark transaction 42 as aborted.

That row version is now invisible.
```

Conceptually:

```text
old version:
balance = 100

new version:
balance = 90
xmin = transaction 42
```

If transaction 42 aborts:

```text
transaction 42 = ABORTED
```

Other transactions ignore that new tuple.

So rollback can be largely about **visibility**, not reversing each mutation instruction one by one.

---

# 12. ACID is easier to understand now

Transactions are often explained as ACID.

Let's connect ACID to the mechanisms.

## Atomicity

> All operations happen or none do.

Implemented using things like:

```text
transaction status
MVCC versions
WAL
rollback
```

Example:

```text
Alice -100
Bob +100
```

cannot leave half the transaction visible.

---

## Consistency

Consistency is slightly misunderstood.

It means the transaction moves the database from one valid state to another, assuming your constraints/business rules are correctly encoded.

For example:

```sql
balance DECIMAL CHECK (balance >= 0)
```

or:

```sql
UNIQUE(email)
```

or:

```sql
FOREIGN KEY (user_id)
REFERENCES users(id)
```

A transaction cannot commit if it violates these constraints.

But application-level consistency may still require your code.

For example:

```text
Total debits == total credits
```

may need application logic or database constraints.

---

## Isolation

Transactions should behave as though other concurrent transactions aren't interfering beyond what the chosen isolation level allows.

Implemented through:

```text
MVCC
snapshots
locks
serialization checks
```

We'll go much deeper into this shortly.

---

## Durability

Once committed:

```text
data survives crash
```

Implemented primarily through:

```text
WAL
fsync
storage guarantees
```

---

# 13. The hardest part: concurrency

Single-transaction examples don't demonstrate why transactions are truly necessary.

Imagine Alice has:

```text
balance = $100
```

Two API requests arrive simultaneously.

Request A:

```text
withdraw $80
```

Request B:

```text
withdraw $80
```

Naive application logic:

```ts
const account = await db.account.findUnique(...);

if (account.balance >= 80) {
  await db.account.update({
    data: {
      balance: account.balance - 80
    }
  });
}
```

Now both requests might execute:

```text
Request A reads 100
Request B reads 100

A checks 100 >= 80 => yes
B checks 100 >= 80 => yes

A writes 20
B writes 20
```

Final balance:

```text
20
```

But two withdrawals occurred:

```text
-$160
```

The system lost one update.

This is called a:

> lost update

---

# 14. Transactions alone don't automatically fix concurrency

This is very important.

You might write:

```sql
BEGIN;

SELECT balance
FROM accounts
WHERE id = 'alice';

UPDATE accounts
SET balance = 20
WHERE id = 'alice';

COMMIT;
```

But two concurrent transactions can still both read `100`.

So:

> **Using a transaction is not the same as correctly handling concurrency.**

You need appropriate locking, atomic operations, or isolation.

---

# 15. Solution 1: atomic UPDATE

Instead of:

```text
read
calculate
write
```

let the database perform the state change atomically.

```sql
UPDATE accounts
SET balance = balance - 80
WHERE id = 'alice'
  AND balance >= 80;
```

Then inspect how many rows were updated.

If:

```text
rowCount = 1
```

withdrawal succeeded.

If:

```text
rowCount = 0
```

insufficient funds.

This is excellent because the database controls the modification.

---

# 16. Solution 2: row-level locking

You can explicitly lock the row:

```sql
BEGIN;

SELECT balance
FROM accounts
WHERE id = 'alice'
FOR UPDATE;
```

`FOR UPDATE` tells PostgreSQL:

> I intend to modify this row. Other transactions trying to update/lock it must wait.

Timeline:

```text
Transaction A
--------------------------------
SELECT ... FOR UPDATE
locks Alice

Transaction B
--------------------------------
SELECT ... FOR UPDATE
WAITING...
```

Then A:

```sql
UPDATE accounts
SET balance = balance - 80
WHERE id = 'alice';

COMMIT;
```

Lock released.

B wakes up.

Then B reads the new balance:

```text
20
```

Now:

```text
20 >= 80
```

false.

So B rejects the withdrawal.

---

# 17. What is a row lock under the hood?

Think of PostgreSQL maintaining information that effectively says:

```text
Transaction 101 has modification rights on Alice's row version.
```

If another transaction attempts a conflicting operation:

```text
Transaction 102
```

Postgres checks:

```text
Does T102's requested lock conflict with T101?
```

If yes:

```text
T102 -> WAIT
```

The operating process sleeps instead of spinning.

When T101 commits/rolls back:

```text
lock released
```

T102 wakes up.

---

# 18. PostgreSQL has different lock types

There's more than one lock.

At a high level:

```text
Table locks
Row locks
Predicate-style serialization locks
Advisory locks
```

Row locking examples:

```sql
FOR UPDATE
FOR NO KEY UPDATE
FOR SHARE
FOR KEY SHARE
```

The stronger the lock, the more operations it blocks.

You don't need to memorize them immediately, but it's worth understanding that locking isn't just:

```text
locked / unlocked
```

It's a compatibility matrix.

For example:

```text
Read lock
Write lock
Schema modification lock
Foreign key related locks
```

may have different conflict rules.

---

# 19. What happens to locks when transaction ends?

Locks belong to the transaction.

```sql
BEGIN;

SELECT *
FROM accounts
WHERE id = 'alice'
FOR UPDATE;
```

The lock remains held.

Then:

```sql
COMMIT;
```

or:

```sql
ROLLBACK;
```

PostgreSQL releases the lock.

This is why long-running transactions are dangerous.

Example:

```text
BEGIN
lock 500 rows

do network request
wait 15 seconds

write database

COMMIT
```

For those 15 seconds, other requests may be blocked.

A good transaction usually should be:

```text
short
database-focused
deterministic
```

Not:

```text
BEGIN

call Stripe

call OpenAI

send email

sleep

UPDATE

COMMIT
```

That's a production anti-pattern.

---

# 20. Real production example: order checkout

Suppose you build an ecommerce application.

User buys:

```text
Product: PS5
Stock: 1
```

Two people click Buy simultaneously.

Naive code:

```ts
const product = await db.product.findUnique({
  where: { id: productId }
});

if (product.stock > 0) {
  await db.product.update({
    where: { id: productId },
    data: {
      stock: product.stock - 1
    }
  });

  await db.order.create(...);
}
```

Two requests:

```text
A reads stock = 1
B reads stock = 1

A: available
B: available

A creates order
B creates order
```

Congratulations, you sold one PS5 twice.

---

# 21. Correct transaction

In SQL:

```sql
BEGIN;

SELECT stock
FROM products
WHERE id = 'ps5'
FOR UPDATE;

-- application checks stock

UPDATE products
SET stock = stock - 1
WHERE id = 'ps5';

INSERT INTO orders (...);

COMMIT;
```

Now only one transaction can safely operate on that inventory row at a time.

---

# 22. Prisma example

With Prisma:

```ts
await prisma.$transaction(async (tx) => {
  const product = await tx.product.findUnique({
    where: { id: productId },
  });

  if (!product) {
    throw new Error("Product not found");
  }

  if (product.stock <= 0) {
    throw new Error("Out of stock");
  }

  await tx.product.update({
    where: { id: productId },
    data: {
      stock: {
        decrement: 1,
      },
    },
  });

  await tx.order.create({
    data: {
      userId,
      productId,
      quantity: 1,
    },
  });
});
```

But notice something.

Even here, this:

```text
find stock
check stock
decrement
```

can have race-condition concerns depending on isolation and update strategy.

A stronger pattern is:

```ts
const result = await tx.product.updateMany({
  where: {
    id: productId,
    stock: {
      gt: 0,
    },
  },
  data: {
    stock: {
      decrement: 1,
    },
  },
});

if (result.count !== 1) {
  throw new Error("Out of stock");
}
```

Now the database itself performs:

```text
check + modify
```

atomically.

Then:

```ts
await tx.order.create(...);
```

If order creation fails:

```text
whole transaction rolls back
```

That is a much better design.

---

# 23. Your Expense Tracker example

Imagine you're syncing transactions from Plaid.

Suppose one sync operation must:

```text
1. Insert new transactions
2. Update modified transactions
3. Mark removed transactions
4. Update Plaid cursor
```

This is a perfect transaction use case.

Without a transaction:

```text
insert transactions ✓
update transactions ✓
mark removals ✓
update cursor ✗ crash
```

Next sync starts with the old cursor.

You may fetch/reprocess data unexpectedly.

Worse:

```text
update cursor ✓
insert transactions ✗
```

Now you've advanced your cursor while failing to persist the corresponding financial data.

You may permanently skip those records.

That's bad.

A strong boundary is:

```text
persist all sync results
+
advance cursor

as one atomic transaction
```

Conceptually:

```ts
await db.transaction(async (tx) => {
  await persistAddedTransactions(tx, added);

  await persistModifiedTransactions(tx, modified);

  await persistRemovedTransactions(tx, removed);

  await tx.plaidItem.update({
    where: {
      id: plaidItemId,
    },
    data: {
      transactionsCursor: nextCursor,
    },
  });
});
```

If anything fails:

```text
ROLLBACK everything
```

Your database stays logically aligned with the cursor.

That's the kind of scenario where transactions are not just convenient—they protect a domain invariant.

---

# 24. The best way to decide whether something belongs in one transaction

Ask this:

> If operation #2 fails after operation #1 succeeds, would the database be logically invalid?

If yes, they probably belong in the same transaction.

Examples:

```text
create user
+
create required profile
```

Maybe.

```text
create order
+
create order items
+
decrement inventory
```

Yes.

```text
delete trip
+
delete required dependent records
```

Possibly, depending on cascading constraints.

```text
Plaid sync writes
+
cursor advancement
```

Definitely.

```text
create expense
+
create expense splits
```

Very likely.

```text
transfer money from account A
+
transfer money to account B
```

Absolutely.

---

# 25. Example: expense splitting

Say you create:

```text
Expense:
Dinner $120
```

Participants:

```text
Takaki $40
Angela $40
Bob    $40
```

You might have:

```text
expenses
expense_splits
```

Your code:

```ts
const expense = await prisma.expense.create(...);

await prisma.expenseSplit.createMany(...);
```

What if:

```text
expense succeeds
split creation fails
```

Now:

```text
expense exists
splits missing
```

If your application requires every group expense to have splits, that's an invalid state.

Use:

```ts
await prisma.$transaction(async (tx) => {
  const expense = await tx.expense.create({
    data: {
      tripId,
      amount: 120,
      title: "Dinner",
    },
  });

  await tx.expenseSplit.createMany({
    data: [
      {
        expenseId: expense.id,
        memberId: aliceId,
        amount: 40,
      },
      {
        expenseId: expense.id,
        memberId: bobId,
        amount: 40,
      },
      {
        expenseId: expense.id,
        memberId: charlieId,
        amount: 40,
      },
    ],
  });
});
```

If `createMany` throws:

```text
expense insertion becomes invisible
```

No orphan expense.

---

# 26. Now isolation levels

This is where transactions become much more subtle.

Standard isolation levels:

```text
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

PostgreSQL effectively maps `READ UNCOMMITTED` to `READ COMMITTED`, so the practical PostgreSQL levels are:

```text
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

---

# 27. READ COMMITTED

PostgreSQL default.

Each SQL statement sees a snapshot of committed data as of the start of that statement.

Example:

Transaction A:

```sql
BEGIN;

SELECT balance FROM accounts;
```

gets:

```text
100
```

Transaction B:

```sql
UPDATE accounts
SET balance = 200;

COMMIT;
```

Transaction A executes again:

```sql
SELECT balance FROM accounts;
```

and may now get:

```text
200
```

Same transaction.

Different result.

Because each statement gets a newer snapshot.

This is called:

> non-repeatable read

---

# 28. REPEATABLE READ

Now PostgreSQL gives the transaction a stable snapshot.

Transaction A:

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;

SELECT balance FROM accounts;
```

returns:

```text
100
```

B changes it:

```text
200
```

and commits.

A runs:

```sql
SELECT balance FROM accounts;
```

Still sees:

```text
100
```

Why?

Because A is effectively saying:

```text
Show me the database as it appeared when my snapshot was established.
```

MVCC makes that possible.

---

# 29. Snapshot mental model

Imagine database history:

```text
Time ----->

T1 committed
T2 committed
T3 running
T4 committed
T5 running
```

Your transaction snapshot might record something conceptually like:

```text
visible committed transactions:
T1
T2
T4

not visible:
T3
T5
future transactions
```

Rows created by invisible transactions are ignored.

Rows invalidated only by invisible transactions may still be visible.

So a snapshot is essentially:

> Which transaction history am I allowed to observe?

---

# 30. SERIALIZABLE

This is strongest.

The goal is:

> Make concurrent transactions behave as if they had executed one after another.

Even though they may physically execute concurrently.

Example:

```text
Transaction A
Transaction B
```

The result must be explainable as either:

```text
A then B
```

or:

```text
B then A
```

If PostgreSQL detects a concurrency pattern that cannot safely be serialized, one transaction may fail with:

```text
serialization failure
```

Then your application is expected to retry.

---

# 31. Why serializable doesn't mean "everything gets locked"

PostgreSQL's Serializable mode uses a technique called:

> Serializable Snapshot Isolation

It tracks dependency patterns between transactions.

Conceptually it asks:

```text
T1 read something
T2 changed something

T2 read something
T1 changed something

Could these create an impossible serialization?
```

If yes, one transaction gets aborted.

That's more sophisticated than simply putting a giant lock around the database.

---

# 32. Classic write-skew example

Suppose two doctors are on call:

```text
Alice = on call
Bob   = on call
```

Rule:

```text
at least one doctor must always remain on call
```

Alice's transaction:

```text
reads Bob = on call
sets Alice = off
```

Bob's transaction:

```text
reads Alice = on call
sets Bob = off
```

If both happen concurrently:

```text
Alice sees Bob on call
Bob sees Alice on call
```

Both think:

```text
safe to go off-call
```

Final result:

```text
Alice off
Bob off
```

Invariant broken.

This is not a basic lost-update problem because they update different rows.

Serializable isolation can detect the dangerous dependency and abort one transaction.

---

# 33. Deadlocks

Transactions and locking create another problem.

Suppose:

Transaction A:

```text
lock Alice
then wants Bob
```

Transaction B:

```text
lock Bob
then wants Alice
```

Timeline:

```text
A locks Alice
B locks Bob

A waits for Bob
B waits for Alice
```

Diagram:

```text
Transaction A
    |
    v
 Alice locked
    |
 wants Bob
    |
    v
 waiting on B


Transaction B
    |
    v
 Bob locked
    |
 wants Alice
    |
    v
 waiting on A
```

Nobody can continue.

That's a:

> deadlock

PostgreSQL detects this wait cycle and aborts one transaction.

You may see:

```text
ERROR: deadlock detected
```

---

# 34. How PostgreSQL detects deadlocks

Internally there's effectively a **wait-for graph**.

```text
T1 -> T2
means:
T1 is waiting for T2
```

Example:

```text
T1 -> T2
T2 -> T1
```

Cycle detected.

Postgres chooses one transaction as victim:

```text
abort T2
```

Then:

```text
T1 continues
```

This is another reason application code should be able to handle transaction failure.

---

# 35. Preventing deadlocks

One common technique:

> Always acquire locks in a consistent order.

Bad:

```text
Request A:
lock account A
lock account B

Request B:
lock account B
lock account A
```

Better:

```text
always lock smaller account ID first
```

Then every transaction locks:

```text
A
then B
```

Never:

```text
B
then A
```

Greatly reducing deadlock opportunities.

---

# 36. Savepoints

Transactions can also have partial rollback points.

```sql
BEGIN;

INSERT INTO users (...);

SAVEPOINT user_created;

INSERT INTO optional_settings (...);
```

Something goes wrong:

```sql
ROLLBACK TO SAVEPOINT user_created;
```

The earlier work remains part of the transaction.

Then:

```sql
COMMIT;
```

This is like nested rollback checkpoints.

PostgreSQL doesn't really implement arbitrary nested top-level transactions the way developers sometimes imagine.

Instead libraries often emulate nested transactions using:

```text
SAVEPOINT
```

---

# 37. What a transaction looks like inside an API request

Imagine:

```text
POST /api/orders
```

Your Node process receives the request.

Conceptually:

```text
HTTP Request
      |
      v
Node / Express
      |
      v
Prisma
      |
      v
PostgreSQL connection
      |
      v
BEGIN
      |
      v
SQL #1
      |
      v
SQL #2
      |
      v
SQL #3
      |
      v
COMMIT
```

The database connection is important.

All statements in one transaction need to execute within the same transaction context.

That's why ORMs manage a dedicated connection for interactive transactions.

---

# 38. Why transaction connection ownership matters

Imagine your app has a connection pool:

```text
Connection 1
Connection 2
Connection 3
Connection 4
```

You cannot safely do:

```text
BEGIN on connection 1

UPDATE on connection 2

INSERT on connection 3

COMMIT on connection 4
```

Those are unrelated database sessions.

Instead:

```text
Connection 2:

BEGIN
UPDATE
INSERT
DELETE
COMMIT
```

All on the same connection.

Prisma's transaction client:

```ts
await prisma.$transaction(async (tx) => {
```

gives you a `tx` object tied to that transactional context.

This is why you should use:

```ts
tx.user.create()
tx.order.create()
```

inside the callback.

Not accidentally:

```ts
prisma.user.create()
```

because that may run outside the transaction.

---

# 39. Connection pools make long transactions expensive

Suppose your server has:

```text
10 PostgreSQL connections
```

If 10 requests each open a transaction and then do:

```ts
await fetch("https://slow-api.com");
```

for 10 seconds, all connections can remain occupied.

Now request 11 arrives.

It waits for a database connection.

So this:

```ts
await prisma.$transaction(async (tx) => {
  await tx.order.create(...);

  const response = await fetch("https://stripe.com/...");

  await tx.order.update(...);
});
```

can be dangerous.

Much better architecture often looks like:

```text
external network work
↓
database transaction
↓
commit quickly
```

Or use workflows such as:

```text
pending state
outbox
webhook confirmation
idempotency
```

depending on the operation.

---

# 40. Database transactions cannot atomically control the entire world

This is one of the most important backend lessons.

Suppose:

```text
1. charge Stripe
2. create order in Postgres
```

You might want:

```text
BEGIN

charge Stripe
create order

COMMIT
```

But PostgreSQL cannot rollback Stripe.

If this happens:

```text
Stripe charge succeeds
database insert fails
```

Postgres can rollback:

```text
INSERT
```

but cannot tell history:

```text
Stripe charge never happened
```

Now you have a distributed consistency problem.

Transactions only guarantee atomicity inside the participating transactional system.

---

# 41. This leads to the outbox pattern

Suppose you want:

```text
create order
+
send email
```

You shouldn't do:

```ts
await prisma.$transaction(async (tx) => {
  await tx.order.create(...);

  await resend.sendEmail(...);
});
```

because email isn't transactional.

Instead:

```text
Transaction:

create order
create outbox event
COMMIT
```

Example:

```ts
await prisma.$transaction(async (tx) => {
  const order = await tx.order.create({
    data: orderData,
  });

  await tx.outbox.create({
    data: {
      type: "ORDER_CREATED",
      payload: {
        orderId: order.id,
      },
    },
  });
});
```

Now the database atomically contains:

```text
order exists
AND
event exists
```

A worker later reads:

```text
ORDER_CREATED
```

and sends the email.

This is one way production systems bridge database transactions and external side effects.

---

# 42. Transaction boundaries are domain boundaries

This is a senior-engineering way of looking at transactions.

Don't ask:

> Which queries should I wrap together?

Ask:

> What fact must become true atomically?

Example:

```text
"An order exists and its inventory is reserved."
```

Then the transaction boundary should protect that fact.

Another:

```text
"An expense exists and its splits add up correctly."
```

Another:

```text
"Plaid transactions are persisted and the cursor represents exactly that persisted state."
```

That's much better than using transactions mechanically.

---

# 43. Transactions + constraints are extremely powerful together

Suppose a transaction has:

```sql
INSERT INTO users(email)
VALUES ('a@example.com');
```

and another transaction concurrently inserts the same email.

You shouldn't rely on:

```ts
const existing = await findUser(email);

if (!existing) {
  await createUser(email);
}
```

because:

```text
A checks -> no user
B checks -> no user

A inserts
B inserts
```

Race condition.

Instead use:

```sql
UNIQUE(email)
```

Then Postgres guarantees:

```text
only one succeeds
```

The other transaction fails.

This pattern is huge:

> Let the database enforce invariants whenever possible.

---

# 44. Why "check then insert" is often unsafe

Naive:

```ts
const existing = await prisma.tripMember.findFirst({
  where: {
    tripId,
    userId,
  },
});

if (!existing) {
  await prisma.tripMember.create(...);
}
```

Two simultaneous join requests:

```text
A: no membership
B: no membership

A creates
B creates
```

Unless the DB has:

```sql
UNIQUE(trip_id, user_id)
```

you now get duplicate membership.

Correct design:

```text
DB unique constraint
+
handle unique violation
```

The transaction/constraint layer is the final authority.

---

# 45. What happens physically during a transaction?

Let's walk one end-to-end.

You execute:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;

COMMIT;
```

Very simplified internals:

### Step 1: BEGIN

Postgres establishes transactional state.

Conceptually:

```text
Transaction ID = X
```

It may establish or later acquire an MVCC snapshot depending on operation/isolation.

---

### Step 2: Parse SQL

```text
SQL text
↓
parser
↓
parse tree
```

---

### Step 3: Planner

Postgres determines:

```text
Index scan?
Sequential scan?
Which indexes?
Which join strategy?
```

---

### Step 4: Find visible row

Using MVCC rules:

```text
Which row version is visible to transaction X?
```

---

### Step 5: Acquire required lock

For update:

```text
row modification lock
```

Conflicting writers may wait.

---

### Step 6: Create new tuple version

Old:

```text
balance = 1000
```

New:

```text
balance = 900
```

Old tuple records that it has been superseded by transaction X.

---

### Step 7: Modify buffers

The relevant database page in shared memory becomes:

```text
dirty
```

Meaning:

```text
memory version differs from disk version
```

---

### Step 8: Generate WAL records

Postgres records enough information to recover the change.

```text
WAL record:
update tuple ...
```

---

### Step 9: COMMIT

Postgres creates commit information and ensures required WAL reaches durable storage.

```text
WAL flush
```

---

### Step 10: transaction becomes visible

Other transactions that get sufficiently new snapshots can now see:

```text
balance = 900
```

---

### Step 11: page may be written later

The dirty page itself may be flushed later by background/checkpoint processes.

That's okay because WAL can reconstruct committed state.

---

# 46. Visualized

```text
Application
    |
    | BEGIN
    v
PostgreSQL Transaction X
    |
    | UPDATE account
    v
Find row version
    |
    v
Acquire row lock
    |
    v
Create new row version
    |
    +------------------+
    |                  |
    v                  v
Shared Buffer        WAL Buffer
modified page        update record
    |                  |
    |                  v
    |               WAL disk
    |                  |
    |               fsync
    |                  |
    +---------> COMMIT successful
                       |
                       v
             transaction visible
```

Later:

```text
Shared buffer
     |
     v
database files on disk
```

---

# 47. Why `VACUUM` exists

Because PostgreSQL creates row versions, old versions accumulate.

Example:

```text
balance=1000
balance=900
balance=850
balance=700
```

Eventually no active transaction needs the older versions.

Postgres can reclaim them.

That's one of `VACUUM`'s jobs.

Without cleanup, tables would endlessly accumulate dead tuples.

This is also why **long-running transactions are bad**.

Imagine transaction T1 started hours ago.

Postgres may have to preserve old row versions because:

```text
T1's snapshot might still need them.
```

So long transactions can cause table bloat and vacuum problems.

---

# 48. Why transactions can hurt performance

Transactions have costs:

```text
locks
WAL writes
MVCC versions
connection occupancy
deadlock risk
contention
vacuum pressure
```

So the goal is not:

> Use giant transactions everywhere.

It is:

> Use the smallest transaction that fully protects the required invariant.

That distinction matters a lot.

---

# 49. Good transaction

```text
BEGIN

read/lock required rows
update 3 rows
insert 2 rows

COMMIT
```

Maybe:

```text
5–30 ms
```

Great.

---

# 50. Bad transaction

```text
BEGIN

SELECT stuff

call Stripe
wait 1.5s

call OpenAI
wait 4s

send email
wait 500ms

user-generated processing
wait 10s

UPDATE

COMMIT
```

Now you have:

```text
long locks
occupied connection
deadlock opportunities
slow requests
higher contention
```

Very bad architecture.

---

# 51. Real example from a group-trip application

Imagine deleting a trip.

You have:

```text
Trip
TripMembers
Expenses
ExpenseSplits
Bookings
ItineraryItems
Messages
```

You want:

```text
delete trip
```

If foreign keys use cascades:

```sql
FOREIGN KEY (...)
REFERENCES trips(id)
ON DELETE CASCADE
```

the database may handle dependent deletion atomically as part of one statement/transaction.

Conceptually:

```text
DELETE trip
    |
    +-> delete members
    |
    +-> delete expenses
    |
    +-> delete expense splits
    |
    +-> delete bookings
```

If any constraint fails:

```text
entire statement/transaction fails
```

This is far safer than manually deleting five tables without a transaction.

---

# 52. Example where transaction is unnecessary

Suppose you're just doing:

```sql
SELECT *
FROM users
WHERE id = 1;
```

You don't generally need to explicitly write:

```sql
BEGIN;
SELECT ...
COMMIT;
```

PostgreSQL already executes every SQL statement inside a transaction.

This is important:

> Even a single SQL statement is transactional.

If you run:

```sql
UPDATE users
SET status = 'active'
WHERE ...
```

Postgres treats that statement atomically.

Explicit transactions are needed when you want **multiple statements** to share one atomic unit or snapshot/lock scope.

---

# 53. PostgreSQL autocommit

Most application drivers operate in:

```text
autocommit
```

Meaning:

```sql
UPDATE ...
```

is effectively:

```sql
BEGIN;

UPDATE ...;

COMMIT;
```

automatically.

So these:

```ts
await db.user.create(...)
await db.profile.create(...)
```

are typically two separate transactions.

Conceptually:

```text
Transaction 1:
user create
COMMIT

Transaction 2:
profile create
COMMIT
```

If the second fails, the first stays committed.

If you need them atomic:

```ts
await prisma.$transaction(async (tx) => {
  await tx.user.create(...);
  await tx.profile.create(...);
});
```

Now:

```text
Transaction 1:

user create
profile create

COMMIT
```

---

# 54. ORM transaction doesn't replace database thinking

This:

```ts
await prisma.$transaction(...)
```

is not magic.

Prisma is essentially coordinating database operations roughly like:

```text
acquire connection

BEGIN

execute query
execute query
execute query

COMMIT

release connection
```

If callback throws:

```text
ROLLBACK
release connection
```

The real guarantees come from PostgreSQL.

Prisma is the API layer exposing them.

---

# 55. Error propagation example

```ts
try {
  await prisma.$transaction(async (tx) => {
    await tx.account.update(...);

    throw new Error("Something failed");

    await tx.account.update(...);
  });
} catch (error) {
  console.log("Transaction failed");
}
```

Sequence:

```text
BEGIN

UPDATE succeeds

JavaScript throws

Prisma detects rejected callback

ROLLBACK

connection released

error propagated to caller
```

The first update does not remain committed.

---

# 56. Transaction example with business logic

Let's implement money transfer properly.

```ts
async function transferMoney(
  fromAccountId: string,
  toAccountId: string,
  amount: number
) {
  return prisma.$transaction(async (tx) => {
    const debit = await tx.account.updateMany({
      where: {
        id: fromAccountId,
        balance: {
          gte: amount,
        },
      },
      data: {
        balance: {
          decrement: amount,
        },
      },
    });

    if (debit.count !== 1) {
      throw new Error("Insufficient balance");
    }

    await tx.account.update({
      where: {
        id: toAccountId,
      },
      data: {
        balance: {
          increment: amount,
        },
      },
    });

    await tx.transfer.create({
      data: {
        fromAccountId,
        toAccountId,
        amount,
      },
    });
  });
}
```

Now one logical event:

```text
money moved
+
ledger record created
```

is atomic.

If:

```text
credit fails
```

then:

```text
debit rollback
```

If:

```text
transfer history insert fails
```

then both balance modifications rollback.

---

# 57. Another important concept: retry

Production transaction code should understand that some failures are temporary.

Examples:

```text
serialization failure
deadlock victim
lock timeout
```

These do not necessarily mean:

```text
business logic is invalid
```

They may mean:

```text
concurrent transactions collided
```

So applications sometimes retry the entire transaction.

Pseudo-code:

```ts
for (let attempt = 0; attempt < 3; attempt++) {
  try {
    return await runTransaction();
  } catch (error) {
    if (!isRetryableTransactionError(error)) {
      throw error;
    }
  }
}
```

Important:

> Retry the entire transaction, not just the failed final query.

Because your reads and assumptions may now be stale.

---

# 58. One subtle rule: never trust stale state outside a transaction

Bad:

```ts
const account = await prisma.account.findUnique(...);

if (account.balance >= 100) {
  await prisma.$transaction(async (tx) => {
    await tx.account.update(...);
  });
}
```

The balance check happened before the transaction.

Between:

```text
SELECT
```

and:

```text
BEGIN transaction
```

another request may modify the account.

That's a classic TOCTOU problem:

```text
Time Of Check
vs
Time Of Use
```

The check and the mutation should be protected together or encoded into one atomic SQL operation.

---

# 59. Another practical rule

When deciding whether to use a transaction, don't start from:

```text
How many SQL queries do I have?
```

Instead start from:

```text
What invariant am I protecting?
```

Examples:

```text
Order and inventory must agree.

Expense and splits must agree.

Plaid cursor and persisted transactions must agree.

Transfer debit and credit must agree.

Trip membership and role state must agree.
```

Once you can state the invariant clearly, the transaction boundary becomes much easier to design.

---

# 60. The full mental model

When your code says:

```ts
await prisma.$transaction(async (tx) => {
  ...
});
```

I want you to mentally picture this:

```text
                APPLICATION
                     |
                     v
              acquire connection
                     |
                     v
                   BEGIN
                     |
                     v
          +----------------------+
          | transaction context  |
          | transaction ID       |
          | snapshot             |
          | locks                |
          +----------------------+
                     |
                     v
                SQL queries
                     |
         +-----------+------------+
         |                        |
         v                        v
       MVCC                     WAL
   row versions             recovery log
         |                        |
         v                        v
   visibility rules         durability
         |                        |
         +-----------+------------+
                     |
                     v
                  COMMIT
                     |
          WAL durable enough
                     |
                     v
          transaction visible
                     |
                     v
              locks released
                     |
                     v
           connection returned
```

If anything goes wrong:

```text
ROLLBACK
    |
transaction marked aborted
    |
its row versions are invisible
    |
locks released
    |
connection returned
```

That's a database transaction under the hood.

---

# 61. A senior-level checklist

Before adding a transaction, ask:

1. **What invariant am I protecting?**
    
2. **Can a single atomic SQL statement do this instead?**
    
3. **Could two requests execute this concurrently?**
    
4. **Do I need row locking?**
    
5. **Which isolation level do I actually need?**
    
6. **Are unique/foreign/check constraints protecting this too?**
    
7. **Am I doing network calls inside the transaction?**
    
8. **Could lock order cause deadlocks?**
    
9. **Should temporary transaction failures be retried?**
    
10. **Is the transaction as short as possible?**
    

If you can reason through those ten questions, you're moving well beyond “I know how to use `$transaction()`” and toward actually understanding database correctness.

The biggest mental shift is this:

> A transaction is not primarily a way to group queries.  
> It is a mechanism for controlling **visibility, concurrency, atomicity, and durability of state transitions**.

And underneath that mechanism, PostgreSQL is coordinating **transaction IDs, MVCC row versions, snapshots, locks, WAL records, buffers, commit records, and crash recovery** to make the illusion work.