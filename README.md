# Engineering Journey

My ongoing learning notes on how software works, from first principles.

I am building an understanding of the mechanisms behind everyday tools: how
databases keep changes consistent, how Linux manages memory, how packets travel,
and how JavaScript and UI libraries coordinate work.

These are working study notes, questions, examples, and explanations collected
along the way, including material developed through AI-assisted study. They may
be incomplete or contain mistakes; this repository documents learning in progress,
not mastery or a list of completed projects.

## Explore the notes

The original note names and the `takaki learns/` folder are preserved.
Start with any topic below; shorter notes sit alongside longer explorations.

- **[Database correctness](takaki%20learns/Atomicity%20from%20scratch.md)** — Transactions, WAL, MVCC, locks, isolation, and recovery.
- **[Applying ACID](takaki%20learns/ACID%20for%20my%20ExpenseTracker.md)** — Questions about idempotency, atomicity, and concurrency.
- **[Linux memory](takaki%20learns/How%20Memory%20works%20in%20linux.md)** — Virtual memory, pages, cache, swap, and memory accounting.
- **[Networking](takaki%20learns/NETWORKING%20101.md)** — Routing, DNS, TCP, NAT, TLS, and the path of an HTTP request.
- **[JavaScript promises](takaki%20learns/Inside%20Promise.md)** — Promise state, reaction jobs, microtasks, and chaining.
- **[React](takaki%20learns/How%20React%20works.md)** — JSX, rendering, Fiber, state, and component identity.
- **[TanStack Query](takaki%20learns/How%20Tanstack%20Query%20works.md)** — Caching, observers, invalidation, retries, and server state.
- **[Cursor pagination](takaki%20learns/CURSOR%20PAGINATION.md)** — A short outline of cursor-based fetching.
- **[Cookies](takaki%20learns/Cookies.md)** — A brief note on browser and server responsibilities.
- **[Calendar logic](takaki%20learns/BASIC%20CALENDER%20LOGIC.md)** — Working notes on building a calendar grid.


## More study notes

- **[Row-level security for secret items](takaki%20learns/RLS%20Secret%20Item%20Security.md)** — Notes on keeping privileged Plaid item access behind a server-side authorization boundary.
- **[Database schema design](takaki%20learns/How%20to%20design%20DB%20schema.md)** — A reference link for schema-design principles.

## Related documents

- **[Budget fetching with URL query parameters](Documents/2026-09-08%20%28Tue%29.md)** — Notes on selecting a budget month in the BankOS project.
- **[Exploring a hacking lab](Documents/EXPLORING%20HACKING%20LAB.md)** — A short lab outline.
- **[Linux monitoring tool](Documents/LINUX%20MONITORING%20TOOL.md)** — Linux system and network observations, including the machine-specific sample output preserved from the original note.
- **[Reverse Engineer Anything](Documents/REA%20-%20Reverse%20Engineer%20Anything.md)** — A learning note on using analysis tools to investigate software behavior.

## Reading and contributing to the journey

Browse the Markdown files directly on GitHub, or open a local copy as an Obsidian
vault. Existing note contents are preserved, including examples and source links.
Topics will grow as I study and revise my understanding.

The public collection contains selected learning notes. Personal vault settings
and unrelated personal documents are kept locally. Updates to approved notes are
synchronized from Obsidian through Git.

[Synchronization test](SYNC-TEST.md) records the non-sensitive setup check.
