You now understand **transactions** (the rules of safety). But rules are just words on paper unless someone enforces them. In a database, the enforcer is the **Lock Manager**.

If you've ever wondered why your simple `UPDATE` query is "hanging" and the spinner is just spinning, welcome to module 9. Your query isn't slow; it's waiting in line.

## 9.1 The Traffic Cop: Shared Locks vs. Exclusive Locks
Imagine a single-stall bathroom with a lock on the door.

- If someone is inside, you wait.
- This is **Safety** (isolation).
- But it is also a **Bottleneck** (Performance).

In the database engine, we cannot simply lock the entire bathroom (the whole database) every time someone wants to use it. That would be a throughput of 1 user at a time. We need a system that allows thousands of users to read and write simultaneously without stepping on each other's toes.

To do this, the engine uses two primary types of locks. These aren't abstract concepts—they are physical flags set in a hash table in the server's RAM.

### 1. The Shared Lock (S-Lock) — "I am Reading"
When you run a `SELECT` statement (in many isolation levels), the database requests a **Shared Lock** on the rows or pages you are reading.

- **The Meaning**: "I am looking at this data. Please don't change it while I'm looking."
- **The Vibe**: Friendly.
- **Concurrency**: "High.
    - If User A has an S-Lock on Row 5, User B can also get an S-Lock on Row 5.
    - Infinite users can read the same row at the same time. They "share" access.

### 2. The Exclusive Lock (X-Lock) — "I am Writing"
When you run `INSERT`, `UPDATE`, or `DELETE`, the database requests an **Exclusive Lock**.

- **The Meaning**: "I am modifying this data. Everyone else stay away."
- **The Vibe**: Hostile.
- **Concurrency**: Zero.
    - If User A has an X-Lock on Row 5, User B **cannot** get any lock (S or X) on that row.
    - User B must wait. This is called **Blocking**.

### The Matrix of Pain (Compatibility)
The Lock Managers uses a simple compatibility matrix to decide who waits and who goes.

| Current Lock | Requesting: Shared (S) | Requesting: Exclusive (X) |
|:---|:---|:---|
| None | ✅ Granted | ✅ Granted |
| Shared (S) | ✅ Granted (Read + Read) | 🛑 BLOCKED (Read + Write) |
| Exclusive (X) | 🛑 BLOCKED (Write + Read) | 🛑 BLOCKED (Write + Write) |

### The Physics of "Blocking"
When we say a query is "Blocked," what is physically happening inside the CPU?

1. **Request**: Your thread asks the Lock Manager, "Give me an X-Lock on Page 105."
2. **Check**: The Lock Manager hashes "Page 105" and looks in its memory map. It sees that another process (Process ID 502) already holds a lock there.
3. **Sleep**: The Lock Manager tells the Operating System, "Put this thread to sleep. Do not give it any CPU cycles."
4. **Queue**: Your thread is added to a linked list of "Waiters" attached to that lock structure.
5. **Wake Up**: When Process 502 finishes (`COMMIT` or `ROLLBACK`), the Lock Manager wakes up the first thread in the waiters list.

!!! warning "The Application Perspective"

    To your Python script or Java application, a block looks exactly like a slow network or a heavy computation. The code just stops on `cursor.execute()`. You must check the database's "Active Locks" view (e.g., `pg_locks`) to tell the difference.

### Granularity: What exactly are we locking?
The "Traffic Cop" can block different sizes of things. This is **Lock Granularity**.

1. **Row Lock (Fine-Grained)**: Locking just one row.
    - *Pro*: Max Concurrency (I edit Row 1, you edit Row 2).
    - *Con*: Expensive. Managing 1,000,000 individual locks eats up RAM.
2. **Page Lock**: Locking the 8 KB storage block.
    - *Pro*: Fewer locks to manage.
    - *Con*: False contention. I want Row 1, you want Row 50, but they live on the same page, so we block each other.
3. **Table Lock (Coarse-Grained)**: Locking the whole table.
    - *Pro*: Very fast management (just one lock).
    - *Con*: Zero concurrency.

!!! tip "Lock Escalation"

    If you try to update 1,000,000 rows, the database might realize, "I don't have enough RAM to store 1,000,000 row locks." It will spontaneously perform **Lock Escalation**, swapping your million row locks for a single Table Lock. Suddenly, the whole table freezes for everyone else. This is a common cause of production outages during "backfill" scripts.

## 9.2 Deadlocks
In the previous section, we talked about **Blocking**—a situation where Transaction A waits for Transaction B to finish. Blocking is annoying, but it is temporary. Eventually, B finishes, and traffic moves.

A **Deadlock** is different. It is permanent. It is a fatal embrace where two transactions strangle each other, and neither can let go.

Without intervention, these two transactions would sit there, consuming RAM and holding connections open, until the server was rebooted or the end of time—whichever comes first.

### The Mechanics of the Standoff
A deadlock is a **Circular Dependency**.

- **Transaction A** holds Key 1 and wants Key 2.
- **Transaction B** holds Key 2 and wants Key 1.

Since locks are physical flags in memory, Transaction A cannot proceed until B releases Key 2. But B cannot finish (and release Key 2) until A releases Key 1.

They are stuck.

### The Anatomy of a Crash
Let's look at a classic textbook deadlock. This often happens in financial systems where money moves between accounts in different directions simultaneously.

**The Setup**:

- **Transaction A**: Transfer $100 from Alice to Bob.
- **Transaction B**: Transfer $100 from Bob to Alice.

**The Execution (Time-ordered)**:

| Time | Transaction A (PID 101) | Transaction B (PID 102) | State of the Engine |
|:---|:---|:---|:---|
| 0:01 | `UPDATE accounts SET bal = ... WHERE id = 'Alice';` | (Idle) | A holds X-Lock on Alice. |
| 0:02 | (Processing...) | `UPDATE accounts SET bal = ... WHERE id = 'Bob';` |B holds X-Lock on Bob. |
| 0:03 | `UPDATE accounts SET bal = ... WHERE id = 'Bob';` | (Processing...) | A requests lock on Bob. BLOCKED by B. |
| 0:04 | (WAITING...) | `UPDATE accounts SET bal = ... WHERE id = 'Alice';` | B requests lock on Alice. BLOCKED by A. |

At 0:04, the circle is complete.

- A is waiting for B.
- B is waiting for A.
- The CPU usage drops to near zero for these threads, but they are technically "running" forever.

### The Grim Reaper: The Deadlock Detector
Databases don't just let these threads rot. They have a background process usually called the **Deadlock Detector**.

This process wakes up periodically (e.g., every 1 second) and inspects the **Wait-For Graph** in memory. It looks for cycles.

If it finds a circle (A -> B -> A), it knows physics has broken down. It must intervene.

1. **Selection**: It picks a **Victim**. Usually, this is the transaction that has done the *least* amount of work (wrote the fewest log records) because it is the cheapest to rollback.
2. **Execution**: It ruthlessly kills the victim's connection.
    - **Postgres**: `ERROR: deadlock detected`.
    - **SQL Server**: `Transaction (Process ID 55) was deadlocked... and has been chosen as the deadlock victim`.
3. **Resolution**: The victim is rolled back (releasing all its locks). The survivor (the other transaction) is instantly unblocked and proceeds to finish its work.

### How to Avoid Deadlocks
You cannot "tune" deadlocks away with hardware. They are logic errors, not resource errors.

**1. Canonical Ordering (The Golden Rule)**: If you always lock resources in the same order, deadlocks are impossible.
    - *Bad*: Transaction A updates Alice then Bob. Transaction B updates Bob then Alice.
    - *Good*: Both transactions order the IDs and update them alphabetically (Alice first, then Bob). Transaction B will simply block at step 1, waiting for A to finish. No circle is formed.

**2. Keep Transactions Short**: The longer your transaction holds locks, the wider your window for collision. Do your math in the application layer, not inside the `BEGIN...COMMIT` block.

**3. The "Select for Update" Trick**: If you know you are going to update a row later, don't just read it with `SELECT`. Use `SELECT ... FOR UPDATE`. This grabs the Exclusive Lock *immediately* at the start of the transaction, rather than upgrading from a Shared Lock later (which is a common cause of deadlocks).

### Debugging Mindset
If your application logs are full of "Deadlock Victim" errors, do not blame the database. The database saved  you from an eternal freeze.

You must find the two queries involved (the database error log will usually print them) and draw the circle. Once you see the circle, the fix is almost always **reordering your writes**.

## 9.3 MVCC (Multi-Version Concurrency Control)
We have arrived at the modern engine's secret weapon.

In the old days (and in strictly locked systems), a database was like a single physical ledger. If I was writing on Page 5 with a pen, you couldn't read Page 5. You had to wait until I put the pen down. This meant that a heavy reporting query (reading the whole book) would freeze all updates, and a heavy update would freeze all reports.

**MVCC** changes the laws of physics. It declares: **Readers do not block Writers, and Writers do not block Readers**.

How? By abandoning the idea that there is only one "true" version of a row at any given moment. Instead, the database maintains a history of changes.

### The Physics of "Snapshotting"
When you update a row in an MVCC database (like Postgres), you aren't actually *changing* the data in place.

- **You think you are doing**: `UPDATE users SET age = 30 WHERE id = 1`.
- **The database is actually doing**:
    1. Mark the old row (age = 29) as "expired" (effective immediately for new transactions).
    2. Insert a **brand new row** (age = 30) somewhere else on the page.

Now there are two physical rows for `ID 1` on the disk.

- **Transaction A (The Reader)** started 5 minutes ago. It holds a "Snapshot" of the past. When it looks at `ID 1`, the engine shows it the old row (age = 29) because that was the truth when Transaction A started.
- **Transaction B (The Writer)** just finished. When it looks at `ID 1`, it sees the new row (age = 30).

Both transactions are running happily at the same time, looking at the same ID, seeing different data, with **zero locking**.

### The Hidden Columns: `xmin` and `xmax`
How does the engine know which version to show you? It uses hidden metadata columns on every single row. You don't see them in a `SELECT *`, but they are there, taking up space on the disk.

In PostgreSQL, these are literally called `xmin` (Created By) and `xmax` (Expired By).

Let's trace a physical update:

1. **Row 1 (Original)**: `(id: 1, val: 'A', xmin: 100, xmax: null)`
    - *Translation*: Created by transaction 100. Still alive (xmax is `null`).
2. **Transaction 101 runs**: `UPDATE ... SET val = 'B';`
3. **The Disk State After Update**:
    - **Row 1 (Old)**: `(id: 1, val: 'A', xmin: 100, xmax: 101)`
        - *Translation*: Created by 100. Killed by 101.
    - **Row 2 (New)**: `(id: 1, val: 'B', xmin: 101, xmax: null)`
        - *Translation*: Created by 101. Alive.

**The Visibility Rule**: When your query runs, it checks your Transaction ID.

- If your ID is 100, you ignore Row 2 (it's from the future) and read Row 1 (it was alive for you).
- If your ID is 102, you ignore Row 1 (it was killed before you started) and read Row 2.

### The Two Implementations
Different engines implement this "Time Machine" differently:

1. **Postgres (The "Heap" Approach)**: It keeps old versions of rows right next to new versions in the main table files.
    - *Pro*: Rolling back is instant (just toggle a flag).
    - *Con*: The table gets bloated with "dead" rows. Scanning the table means stepping over tombstones.
2. **MySQL/Oracle (The "Undo Log" Approach)**: They update the data in place but copy the *old* data to a separate storage area called the "Undo Segment."
    - *Pro*: The main table stays compact.
    - *Con*: If a reader needs data from 3 hours ago, the engine has to painstakingly reconstruct that row by replaying history from the Undo Logs. If the Undo Logs get too full, you get the infamous `ORA-01555 Snapshot Too Old` error.

### The Dark Side of MVCC
MVCC sounds like magic—infinite concurrency! But in engineering, there is no free lunch.

If we keep creating new versions of rows without deleting the old ones, what happens?

- The table grows indefinitely.
- A table with 1 million active rows might physically contain 100 million rows on disk (99 million of them being "dead" history).
- Your queries slow down because the disk head has to read though gigabytes of "ghost" data just to find the "live" data.

This is **Bloat**. And it brings us to the most misunderstood and critical part of database maintenance. Which we will learn in the next module.

## Quiz

<quiz>
In a database lock manager, which combination of locks is compatible (i.e., will NOT cause blocking) on the same resource?
- [ ] Exclusive Lock (X) + Exclusive Lock (X).
- [ ] Exclusive Lock (X) + Shared Lock (S).
- [ ] Shared Lock (S) + Exclusive Lock (X).
- [x] Shared Lock (S) + Shared Lock (S).

</quiz>

<quiz>
Physically, what happens to your database connection when it is 'blocked' by another transaction?
- [x] Your thread is put to sleep by the OS and added to a linked-list of waiters for that lock.
- [ ] Your query is converted into a 'Dirty Read' to bypass the lock.
- [ ] The database rolls back the other transaction to let you through.
- [ ] The database sends a 'Retry Later' error to your client application immediately.

</quiz>

<quiz>
What is the primary difference between a 'Block' and a 'Deadlock'?
- [ ] A block is caused by readers; a deadlock is caused by writers.
- [x] A block resolves itself when the other transaction finishes; a deadlock is a permanent cycle that requires intervention.
- [ ] A deadlock is simply a block that has lasted longer than 60 seconds.
- [ ] Blocks happen in RAM; deadlocks happen on disk.

</quiz>

<quiz>
How does a database engine typically resolve a detected deadlock?
- [ ] It pauses both transactions and asks the administrator to choose one.
- [ ] It allows both transactions to write simultaneously, merging the results.
- [x] It selects a 'victim' transaction and kills (rolls back) it to break the circle.
- [ ] It escalates the locks to a Table Lock.

</quiz>

<quiz>
What is 'Lock Escalation'?
- [ ] When a Shared Lock is promoted to an Exclusive Lock.
- [ ] When a deadlock is detected and the error is escalated to the log.
- [x] When the database swaps many fine-grained locks (rows) for a single coarse-grained lock (table) to save memory.
- [ ] When a transaction increases its priority to bypass the queue.

</quiz>

<quiz>
What is the primary benefit of Multi-Version Concurrency Control (MVCC) over strict locking?
- [ ] It eliminates the need for a Transaction Log (WAL).
- [ ] It compresses data to save disk space.
- [ ] It guarantees that deadlocks never happen.
- [x] Readers do not block writers, and writers do not block readers.

</quiz>

<quiz>
In an MVCC system like PostgreSQL, what physically happens when you UPDATE a row?
- [x] The old row is marked as expired, and a new row is inserted.
- [ ] The change is only written to RAM and never to disk.
- [ ] The database creates a copy of the entire table.
- [ ] The data is overwritten in place on the hard drive.

</quiz>

<quiz>
What is the purpose of the hidden `xmin` and `xmax` columns in PostgreSQL tuples?
- [x] They track which Transaction ID created the row and which Transaction ID expired (deleted/updated) it.
- [ ] They store the minimum and maximum values for the index.
- [ ] They store the user's password hash.
- [ ] They represent the physical data address of the row.

</quiz>

<quiz>
What is the major downside or 'cost' of using MVCC?
- [ ] It requires expensive specialized hardware.
- [ ] It forces all transactions to run serially.
- [x] It accumulates 'dead tuples' (bloat) that must be cleaned up later.
- [ ] It makes reading data slower than writing data.

</quiz>

<quiz>
Which strategy is most effective for preventing deadlocks in your application code?
- [x] Always access resources (tables/rows) in the same canonical order.
- [ ] Disable the Deadlock Detector.
- [ ] Use random delays before executing queries.
- [ ] Increase the timeout setting on the database.

</quiz>

<!-- mkdocs-quiz results -->

## Lab
Please complete module 9's lab in the companion GitHub repository.