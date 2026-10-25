# ACID

**Atomicity, Consistency, Isolation and Durability in Relational Database Systems**

This document is gathered from my note-taking of the
**Udemy - Fundamentals of Database Engineering**  : <http://Husseinnasser.com>

---


## Agenda

- Introduction
- What is a Transaction?
- Atomicity
- Isolation
- Consistency
- Durability

---

# Introduction

## ACID

ACID stands for **A**tomicity, **C**onsistency, **I**solation and **D**urability.

### Lecture notes

- These are four critical properties of relational database systems.
- Any engineer working with Postgres, MySQL, SQL Server, Oracle, and even NoSQL or graph databases should understand these four building blocks. They are the first principles that database systems are built on.
- The course covers each property in detail and explains why each one is critical to building and using relational databases successfully.
- **Order of the course:**
  1. What is a transaction? (needed before the four properties)
  2. Atomicity: what it is and why it matters so much for guaranteeing consistency.
  3. Isolation: what happens when many transactions run concurrently, access the same data and change it at the same time? What can they see, what can't they, what shouldn't they?
  4. Consistency.
  5. Durability.
- **Why consistency comes third and not second:** The instructor deliberately flips the usual order of the acronym. Understanding atomicity and isolation first is critical to understanding consistency. If a transaction changes something you just wrote, you get a result you didn't expect, and that hurts consistency.
- **Durability is the one many NoSQL databases play with.** Some databases sacrifice durability for performance, and there is nothing wrong with that. What matters is understanding it: when you commit ("save"), the data should be persisted in a non-volatile place. Then you can still read it after a database crash or a power loss.
- A small quiz at the end of the section explains why each wrong answer is wrong.

---

# What is a Transaction?

## Transaction

- A collection of queries
- One unit of work
- E.g. Account deposit (SELECT, UPDATE, UPDATE)

### Lecture notes

- A transaction is a collection of SQL queries treated as **one unit of work**.
- **Why one unit of work?** SQL data is structured and spread over many tables. It is hard (sometimes impossible) to do everything the application logically needs in one query, so you run one or more queries. You `BEGIN`, run queries 1 through N, then end the transaction. Logically it is one thing that cannot be split.
- **Example, an account transfer:**
  1. SELECT the balance of the first account and check that there is enough money.
  2. UPDATE the first account to deduct $100.
  3. UPDATE the second account to add $100.
  That is read, update, update: three queries in one transaction.

## Transaction Lifespan

- Transaction BEGIN
- Transaction COMMIT
- Transaction ROLLBACK
- Transaction unexpected ending = ROLLBACK (e.g. crash)

### Lecture notes

- **BEGIN** tells the database you are about to start a new transaction with multiple queries in it.
- Nothing you write is durable until you say **COMMIT**. COMMIT asks the database to persist all the changes made during the lifespan of the transaction.
- **Think like a database builder.** Say you ran 1000 queries. When do you write to disk?
  - **Option A:** write each change to disk as it happens. **Commit is fast**, because the work is already done. **Rollback is slow**, because you must undo all the work already written.
  - **Option B:** keep the changes in memory and write everything at commit. **Commit is slower.** **Rollback is fast**, because you just throw away the memory.
  - There is no right answer, only trade-offs. Databases keep undo and redo space for these changes, and each one implements this differently.
  - The instructor encourages students to think about how they would design this *before* reading how real databases do it, to avoid being "polluted".
- **ROLLBACK:** "Forget that I made all these changes. Do not persist them." The user can ask for it explicitly, for example after executing 50,000 queries. Long transactions are not recommended in general.
- **Unexpected endings:** If the database crashes 20,000 queries into a transaction, nobody can issue a rollback. The database must know on restart that the transaction was in flight and roll it back. That is another code path the database developers have to write: "databases are very hard to build."
- **Every database is different** (Postgres, SQL Server, MySQL, Oracle). Each optimizes for what its engineers considered most important: commits, rollbacks or crash recovery.
  - According to the instructor, **Postgres has very fast commits**. It persists changes during the transaction, so it does a lot of I/O but commit is quick.
  - The instructor believes (and says "if I'm not mistaken") that **SQL Server commits are slower** for large transactions.
- **The scariest case: a crash during a commit.** Fast commits make that less likely. Slow commits on large transactions make it more likely. After such a crash you don't know whether the transaction was committed or not.

## Nature of Transactions

- Usually transactions are used to change and modify data
- However, it is perfectly normal to have a read-only transaction
- Example: you want to generate a report and you want a consistent snapshot based on the time of the transaction
- We will learn more about this in the Isolation section

### Lecture notes

- Why use a transaction just to read? Why not let each query be its own transaction? Because you want **consistency**.
- A read-only transaction gives you a snapshot as of the start of the transaction. Everything you read is based on that initial time. If a concurrent transaction changes something, you don't care, because you are isolated from it.
- You can also tell the database that the transaction is read-only so it can optimize itself.

## Transaction (example)

Send $100 from Account 1 to Account 2.

Table `ACCOUNT` (balances after the transaction commits):

| ACCOUNT_ID | BALANCE |
| --- | --- |
| 1 | $1000 → $900 |
| 2 | $500 → $600 |

```
BEGIN TX1
  |
  |  SELECT BALANCE FROM ACCOUNT WHERE ID = 1
  |
  |  BALANCE > 100   (check passes)
  |
  |  UPDATE ACCOUNT SET BALANCE = BALANCE - 100 WHERE ID = 1    (1000 -> 900)
  |
  |  UPDATE ACCOUNT SET BALANCE = BALANCE + 100 WHERE ID = 2    (500 -> 600)
  |
  v
COMMIT TX1
```


### Lecture notes

- Before the slide, the instructor suggests pausing and designing this yourself.
- The check `BALANCE > 100` can live in the application, or as a **constraint at the database level** ("the balance should never go negative"). Negative data would indicate inconsistent data, unless your application gives negative balances a meaning.
- When the transaction commits, both updates are physically written to disk (exactly where is a separate discussion).

### Lecture notes: summary

- A transaction is a collection of queries treated as a single unit of work.
- It can change data or be read-only.
- **A transaction is always running.** If you don't start one, the database starts one for you implicitly and commits it immediately. A plain `UPDATE` or `INSERT` is therefore a one-statement transaction. Some transactions are user-defined, and some are implicit.

### [EXTRA] The transfer in real SQL

```sql
BEGIN;

SELECT balance FROM account WHERE id = 1;          -- app checks balance > 100

UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;

COMMIT;     -- or ROLLBACK;
```

A database-level safety net for the "never go negative" rule:

```sql
ALTER TABLE account ADD CONSTRAINT balance_not_negative CHECK (balance >= 0);
```

With this constraint, a debit that would make the balance negative fails. Under atomicity, the whole transaction then rolls back.

Marking a transaction as read-only (PostgreSQL syntax):

```sql
BEGIN TRANSACTION READ ONLY;
```

*end of [EXTRA]*

## Summary: What is a Transaction?

---

# Atomicity

## Atomicity

- All queries in a transaction must succeed.
- If one query fails, all prior successful queries in the transaction should roll back.
- If the database went down prior to a commit of a transaction, all the successful queries in the transactions should roll back.

### Lecture notes

- Atomicity is one of the four ACID properties. It applies to any database system (relational, NoSQL, graph, time-series), not only relational ones.
- **The name:** an atom is something that cannot be split. A transaction is one unit of work and must not be split either.
- **What counts as a failed query?** A failed constraint (for example the balance would go negative), a duplicate primary key entry, or invalid SQL syntax. Even if 100 queries in the transaction succeeded, **one failure rolls back the whole transaction.**
- **Crash before commit:** If the database goes down mid-transaction, you can't issue a rollback because the database is gone. No query failed, but the transaction was never committed. On **restart** the database must detect this and roll the transaction back.
- **Why this matters when choosing a database:** The instructor says people often ask "which database should I choose?". The answer depends on understanding how databases work internally and matching that to your use case. For example:
  - Some databases are **"optimistic" about commits.** They write to disk while the queries run, assuming you will commit. At commit they only have to write one flag saying "this transaction is committed", which is very fast. In return, a rollback has to undo work on disk.
  - Other databases keep changes in memory while the queries run, so queries are very fast. At commit they must flush everything to disk, so commit is slow, but rollback is instant.
  - There is no right or wrong, only trade-offs.

## Atomicity (example: crash mid-transaction)

Send $100 from Account 1 to Account 2.

| ACCOUNT_ID | BALANCE |
| --- | --- |
| 1 | $1000 → $900 |
| 2 | $500 |

```
BEGIN TX1
  |
  |  SELECT BALANCE FROM ACCOUNT WHERE ID = 1
  |
  |  BALANCE > 100
  |
  |  UPDATE ACCOUNT SET BALANCE = BALANCE - 100 WHERE ID = 1    (1000 -> 900)
  |
  X  *** CRASH ***  (the machine goes down, blue screen)
```

a Windows blue-screen ("A problem has been detected and Windows has been shut down to prevent damage to your computer... THREAD_STUCK_IN_DEVICE_DRIVER") to illustrate the crash.

### Lecture notes

- The first change was made, then the database crashed before the second change.
- Imagine this is a **badly implemented database**. After a restart, account 1 shows $900 and account 2 still shows $500.

## Atomicity (after the restart)

| ACCOUNT_ID | BALANCE |
| --- | --- |
| 1 | $900 |
| 2 | $500 |

- After we restarted the machine the first account has been debited but the other account has not been credited.
- This is really bad as we just lost data, and the information is inconsistent.
- An atomic transaction is a transaction that will roll back all queries if one or more queries failed.
- The database should clean this up after restart.

### Lecture notes

- We just lost $100 "in thin air", and not only data but real money. **A lack of atomicity leads to inconsistency**, which is covered in the Consistency section.
- Strictly speaking nothing "failed" here, because the second query never even executed. The database never got a chance to issue a rollback. So **the database itself is responsible** for cleaning up the garbage on restart. It has to know a transaction was running and crashed, and it has to change the $900 back to $1000.
- **Real-world experience (from the instructor):** The instructor started working professionally in 2005 and has seen rollbacks that took **over an hour**, especially on SQL Server, for long transactions.
  - These weren't rollbacks requested by a user. The database refused to finish starting up until it had cleaned up the half-written data.
  - In recent releases, databases let the rollback continue in the background while other tables or databases come online. The database is still in a rollback state, so **CPU and memory are hammered** while it runs.
  - How long it takes depends on how much was written and how long the transaction was. That is one more reason long transactions are generally a bad idea.

### Lecture notes: summary

- Atomicity means a transaction is one unit of work that cannot be split.
- With 100 queries, either all 100 succeed or **everything that succeeded is rolled back** if any one fails.
- It also covers **crashes**, which are not an explicit failure by the user. The changes must still be rolled back.

### [EXTRA] How databases roll back after a crash (undo / redo)

Most databases keep two kinds of log records:

- **Redo information:** how to re-apply a change. It is used to rebuild committed work that did not reach the data files before a crash.
- **Undo information:** how to reverse a change. It is used to erase changes of transactions that never committed.

A typical restart (crash recovery) goes like this:

```
Database restarts after a crash
        |
        v
 1. REDO phase : replay the log forward so the data files are
                 back to exactly the state at the moment of the crash
        |
        v
 2. UNDO phase : find transactions that were in flight (no COMMIT record)
                 and reverse their changes
        |
        v
 Database is consistent again and accepts connections
```

This is why the example above is repaired automatically: the debit of account 1 has no matching commit, so it gets undone.

*end of [EXTRA]*

### [EXTRA] A failed statement inside a transaction: database differences

- **PostgreSQL:** after any error inside a transaction, the transaction is marked as aborted. Every further command is rejected until you `ROLLBACK`. This is a strict form of "all or nothing".
- **MySQL (InnoDB):** by default only the statement that failed is rolled back. The transaction stays open, and the application decides whether to continue or `ROLLBACK`.

In both cases, nothing is persisted unless the transaction commits. The practical lesson is to always handle errors by explicitly rolling back.

*end of [EXTRA]*

### [EXTRA] Two meanings of "optimistic"

In the lectures the word *optimistic* appears in two different contexts. Don't mix them up:

1. **Optimistic writing (this section):** the database writes changes to disk while the transaction runs, *assuming it will commit*. This is about **when data is written** (durability and rollback speed).
2. **Optimistic concurrency control (Isolation section):** the database does not lock rows. It detects conflicts at the end and fails one of the transactions. This is about **how concurrent transactions are kept isolated**.

*end of [EXTRA]*

## Summary: Atomicity

---

# Isolation

## Isolation

- Can my in-flight transaction see changes made by other transactions?
- Read phenomena
- Isolation levels

### Lecture notes

- Isolation is a critical and **underrated** property. Few people understand the ramifications of transaction isolation.
- Real databases have many TCP connections (local or remote), and each connection executes its own transactions. So **concurrency** happens: multiple transactions fighting to read and write the same pieces of data.
- The key question: *can my in-flight transaction see changes made by other transactions?* That includes transactions that are also in flight, and also transactions that have already finished. If I'm reading, reading, reading and another transaction commits a change, should I see it? There is no right or wrong answer. It depends on what you want.
- The result is a set of **read phenomena**. They are usually undesirable and are some of the **nastiest things to debug**. The goal is to recognize them when you run into one.
- **Isolation levels** were introduced to solve these undesirable read phenomena.

## Isolation - Read phenomena

- Dirty reads
- Non-repeatable reads
- Phantom reads
- Lost updates

### Lecture notes: definitions

- **Dirty read:** you read something that another transaction has written but not yet committed. That change could be rolled back, committed, or lost if the database crashes. Anything "dirty" means it is not fully flushed or committed and might change.
- **Non-repeatable read:** within one transaction you read a value, then read it again, and it has changed. You may think "why would I ever read twice?", but it happens all the time. It is usually not the *same* query twice. It is *different* queries that depend on the *same* entry. For example, you read a row and then run `SELECT SUM(...)`, which also reads that row behind the scenes.
- **Phantom read:** a read of something that did not exist when you first read. Typical for **range queries** ("all activities between this date and that date"). Another transaction inserts a brand-new row that satisfies your range, so the next range query returns a different result. It isn't a non-repeatable read, because you never read that row before. You can't catch it, like a phantom.
- **Lost update:** you write something, but before you commit, you read it again (or build on it), and another transaction has overwritten it. You can read your own writes inside your own transaction, but another transaction may change the same data, and your update is lost.

## Dirty Reads

Table `SALES` (initial state):

| PID | QNT | PRICE |
| --- | --- | --- |
| Product 1 | 10 | $5 |
| Product 2 | 20 | $4 |

(`QNT` of Product 1 goes from 10 to 15 during TX2's uncommitted update.)

```
        TX1                                      TX2
         |                                        |
   BEGIN TX1                                BEGIN TX2
         |                                        |
 SELECT PID, QNT*PRICE FROM SALES                 |
   -> Product 1, 50                               |
   -> Product 2, 80                               |
         |                         UPDATE SALES SET QNT = QNT+5
         |                         WHERE PID = 1      (10 -> 15, NOT committed)
         |                                        |
 SELECT SUM(QNT*PRICE) FROM SALES                 |
   -> we get $155, should be $130                 |
   -> we read a "dirty" value that has            |
      not been committed                          |
         |                                        |
   COMMIT TX1                               ROLLBACK TX2
```

### Lecture notes

- Product 1: 10 × $5 = $50. Product 2: 20 × $4 = $80. The true total is **$130**.
- TX1 starts a sales report. In parallel, TX2 records 5 extra sales of Product 1 (quantity 10 → 15) but has **not committed**.
- TX1's `SUM` now returns **$155** (15 × 5 + 80), even though TX2 later **rolls back** (the sales never happened). So TX1 read a value that never officially existed.
- **Consistency of the report is broken.** The report lists Product 1 as 50 and Product 2 as 80, then prints a total of 155. The manager would say "that's just wrong".
- Side remark: the query has no `WHERE` clause, so it is unbounded. Fine with two rows, but not with billions of sales. Real queries should be bounded.
- The instructor once built an application that relied on dirty reads. It was a very bad idea, and he regretted it. Nobody is really fine with dirty reads. They are the worst phenomenon.

## Non-repeatable read

Table `SALES` (initial state):

| PID | QNT | PRICE |
| --- | --- | --- |
| Product 1 | 10 | $5 |
| Product 2 | 20 | $4 |

(`QNT` of Product 1 becomes 15 once TX2 commits.)

```
        TX1                                      TX2
         |                                        |
   BEGIN TX1                                BEGIN TX2
         |                                        |
 SELECT PID, QNT*PRICE FROM SALES                 |
   -> Product 1, 50                               |
   -> Product 2, 80                               |
         |                         UPDATE SALES SET QNT = QNT+5
         |                         WHERE PID = 1      (10 -> 15)
         |                                        |
         |                                  COMMIT TX2
         |                                        |
 SELECT SUM(QNT*PRICE) FROM SALES
   -> we get $155, should be $130
   -> we did read a committed value, but it
      gave us inconsistent results
         |
   COMMIT TX1
```

### Lecture notes

- Same setup as before, but this time TX2 **commits before** TX1 runs its second query. This is a valid committed write and a valid read, not a dirty read.
- TX1 read Product 1 as 50 at first, then read it again indirectly through the aggregation and got 75. The result is inconsistent within a single transaction.
- **You might not want this** (although not always, see isolation levels below). **Fixing it is expensive**, because the database must keep the original version of the row visible to your transaction. Many databases do this through **MVCC** (multi-version concurrency control), and implementations differ:
  - **Postgres:** every update creates a **new version of the row** and never overwrites the old one. A transaction keeps a version pointer, so TX1 keeps reading the original version.
  - **MySQL (and Oracle, and maybe SQL Server, per the instructor):** the row is changed **in place** (to 15), and the previous values are written to an **undo log** on disk (the "undo stack"). To serve an old read, the database must open the undo log and reconstruct the old value. For **long-running transactions** this can be expensive, because the latest value can no longer be trusted.
- Whether it matters depends on what you are building. Understand the problem, then decide.

## Phantom read

Table `SALES` (initial state):

| PID | QNT | PRICE |
| --- | --- | --- |
| Product 1 | 10 | $5 |
| Product 2 | 20 | $4 |

Row that appears after TX2 commits:

| PID | QNT | PRICE |
| --- | --- | --- |
| Product 3 | 10 | $1 |

```
        TX1                                      TX2
         |                                        |
   BEGIN TX1                                      |
         |                                        |
 SELECT PID, QNT*PRICE FROM SALES            BEGIN TX2
   -> Product 1, 50                               |
   -> Product 2, 80                  INSERT INTO SALES
         |                           VALUES ('Product 3', 10, 1)
         |                                        |
         |                                  COMMIT TX2
         |                                        |
 SELECT SUM(QNT*PRICE) FROM SALES
   -> we get $140, should be $130
   -> we read a committed value that showed
      up in our range query
         |
   COMMIT TX1
```

### Lecture notes

- Phantom reads come from **range queries**, which are any queries that can return one or more rows. The instructor can hardly think of a query with a `WHERE` clause where phantoms could not happen.
- TX2 inserts a **brand-new** product (10 units at $1 = $10) and commits. TX1's unbounded `SUM` includes it and returns **$140** instead of $130. It is not dirty, because the value is committed. It is not a non-repeatable read either, because TX1 never read Product 3 before.
- **Why the distinction matters (implementation):** To guarantee repeatable reads, a transaction obtains a lock (a type of shared lock) on every row it reads, so those rows cannot change. But a row that **doesn't exist yet** can't be locked, so you can't prevent it from appearing. That is why phantoms are a different problem from non-repeatable reads.

## Lost updates

Table `SALES` (initial state):

| PID | QNT | PRICE |
| --- | --- | --- |
| Product 1 | 10 | $5 |
| Product 2 | 20 | $4 |

(TX1 sets `QNT` of Product 1 to 20, then TX2 overwrites it with 15.)

```
        TX1                                      TX2
         |                                        |
   BEGIN TX1                                BEGIN TX2
         |                                        |
 UPDATE SALES SET QNT = QNT+10                    |
 WHERE PID = 1          (10 -> 20)                |
         |                         UPDATE SALES SET QNT = QNT+5
         |                         WHERE PID = 1      (10 -> 15)
         |                                        |
         |                                  COMMIT TX2
         |                                        |
 SELECT SUM(QNT*PRICE) FROM SALES
   -> we get $155, should be $180
   -> our update was overwritten by another
      transaction and as a result "lost"
         |
   COMMIT TX1
```

### Lecture notes

- Both transactions start at the same time and **both read quantity = 10**. TX1 adds 10 (→ 20). TX2 adds 5 to what *it* read (10 → 15). TX2 is faster and commits first, so the stored value becomes **15**. TX1's update has been overwritten.
- TX1 assumed its sale was recorded, runs the `SUM`, and gets **$155 instead of $180**. This is one of the nastiest phenomena.
- **The fix is locking:** if a transaction changes a row, no other transaction should change the same row until it finishes. Row-level locks are very popular here (see the implementation section).

### [EXTRA] Why the slide's example needs a read-modify-write to lose an update

In the diagram above both transactions use a *single* statement, `UPDATE ... SET QNT = QNT + n`. In PostgreSQL and MySQL under Read Committed, that statement is actually **safe**. The second update waits for the row lock held by the first, then re-reads the latest committed value and adds to it, giving 10 + 10 + 5 = 25.

A lost update typically happens when the application does the arithmetic itself, in separate statements:

```
TX1: SELECT qnt FROM sales WHERE pid = 1;          -- reads 10
TX2: SELECT qnt FROM sales WHERE pid = 1;          -- reads 10
TX1: UPDATE sales SET qnt = 20 WHERE pid = 1;      -- app computed 10 + 10
TX2: UPDATE sales SET qnt = 15 WHERE pid = 1;      -- app computed 10 + 5  -> overwrites 20
```

Takeaways:

- Prefer atomic updates (`qnt = qnt + 5`) whenever possible.
- When you have to read first, use `SELECT ... FOR UPDATE` (pessimistic) or a version check (optimistic). Both are explained in the **[EXPANDED]** section below.

*end of [EXTRA]*

## Isolation - Isolation Levels for in-flight transactions

- **Read uncommitted** - No isolation, any change from the outside is visible to the transaction, committed or not.
- **Read committed** - Each query in a transaction only sees committed changes by other transactions.
- **Repeatable read** - The transaction will make sure that when a query reads a row, that row will remain unchanged while it is running.
- **Snapshot** - Each query in a transaction only sees changes that have been committed up to the start of the transaction. It is like a snapshot version of the database at that moment.
- **Serializable** - Transactions are run as if they were serialized one after the other.

Each DBMS implements isolation levels differently.

### Lecture notes

- You set the level in the transaction using the SQL command `SET TRANSACTION ISOLATION LEVEL ...`.
- **Read uncommitted:** no isolation at all. You can read uncommitted values, so you get dirty reads. In theory it is fast because the database doesn't have to maintain anything, but in practice you can't be sure, because it depends on the implementation. The instructor believes few databases truly support it.
- **Read committed:** one of the **most popular** levels, and the default in many databases. Many engines are optimized for read-committed workloads. Each *query* sees only committed changes, **but** if another transaction commits while yours is still running, your next query will see that change. With a long-running transaction, anything committed in the meantime is picked up, which can be bad.
- **Repeatable read:** invented to fix non-repeatable reads. If you read a row 1000 times in the same transaction, it will not change. The instructor rewrote the definition in practical terms: *when a query reads a row, that row remains unchanged while the transaction is running.* Only rows you actually read are protected. The database doesn't copy a version of rows you haven't read. This protection has a **cost**.
- **Snapshot:** like taking a snapshot of the whole database at the start of the transaction. It gets rid of **every** read phenomenon, but it is also expensive. ("Version of the database" doesn't mean a real backup. Each database implements it in an optimized way.)
  - Repeatable read does **not** stop phantom reads, because the new row was never read, so there was nothing to lock or version.
  - **Postgres' repeatable read is a snapshot isolation:** every row is versioned, and the transaction is timestamped with its version. It filters everything by that version, so inserted rows don't show up.
- **Serializable:** the slowest in theory. Transactions behave as if they ran one after another with no concurrency. Which one goes first is a question the database must answer (related to linearizability). Implementations differ.
- Don't over-stress about isolation levels. Most of the time you won't run into the complex situations, but with many users changing and reading at the same time, you need to know they exist.
- **Snapshot vs serializable:** serializable effectively slows things down by serializing transactions. Snapshot instead takes a version of the database at the start of the transaction.

## Isolation Levels vs read phenomena

| Isolation level | Dirty reads | Lost updates | Non-repeatable reads | Phantoms |
| --- | --- | --- | --- | --- |
| Read Uncommitted | may occur | may occur | may occur | may occur |
| Read Committed | don't occur | may occur | may occur | may occur |
| Repeatable Read | don't occur | don't occur | don't occur | may occur |
| Serializable | don't occur | don't occur | don't occur | don't occur |

Source: <https://en.wikipedia.org/wiki/Isolation_(database_systems)>


### Lecture notes

- The instructor likes this table from Wikipedia. It does not include snapshot isolation.
- Read uncommitted: every phenomenon can occur. Read committed: everything except dirty reads. Repeatable read: no dirty reads, no lost updates, no non-repeatable reads, but phantoms still occur. **Serializable and snapshot: none of them occur.**

### [EXTRA] The same table with Snapshot added, plus one more phenomenon

| Isolation level | Dirty reads | Lost updates | Non-repeatable reads | Phantoms | Write skew |
| --- | --- | --- | --- | --- | --- |
| Read Uncommitted | may occur | may occur | may occur | may occur | may occur |
| Read Committed | don't occur | may occur | may occur | may occur | may occur |
| Repeatable Read | don't occur | don't occur | don't occur | may occur | may occur |
| Snapshot | don't occur | don't occur | don't occur | don't occur | **may occur** |
| Serializable | don't occur | don't occur | don't occur | don't occur | don't occur |

**Write skew** is the reason snapshot and serializable are not the same thing. Two transactions each read the same data, make a decision based on it, and then update *different* rows. Each is valid on its own, but together they break a rule.

> Example: a hospital requires at least one doctor on call. Alice and Bob are both on call. Each checks "are there at least 2 doctors on call?", sees yes, and sets themselves off-call. Both commit. Now zero doctors are on call. Neither wrote to the other's row, so row-level conflict detection did not notice. Only true serializable isolation prevents this.

Notes on the table: the exact behavior depends on the database. For example, in the SQL standard, Repeatable Read is allowed to have phantoms, but PostgreSQL's version is stricter (it is snapshot isolation).

*end of [EXTRA]*

## Database Implementation of Isolation

- Each DBMS implements isolation level differently
- **Pessimistic** - Row level locks, table locks, page locks to avoid lost updates
- **Optimistic** - No locks, just track if things changed and fail the transaction if so
- Repeatable read "locks" the rows it reads but it could be expensive if you read a lot of rows, postgres implements RR as snapshot. That is why you don't get phantom reads with postgres in repeatable read
- Serializable are usually implemented with optimistic concurrency control, you can implement it pessimistically with SELECT FOR UPDATE

### Lecture notes

**Pessimistic concurrency control**
- It means using **locks**. (The course has a whole section on locks.)
- Types: **row-level locks, table locks, page locks**.
  - A **table lock** means that any change you make locks the whole table. This is very expensive.
  - The instructor has often run into **lock escalation**: a row lock is escalated to a table lock, and every other transaction trying to change other values in that table sits pending, because the table is locked.
  - A **page lock** locks the logical page that contains the changed rows. This matters especially with clustered tables.
- The philosophy: "I'm changing this, lock it, don't let anybody touch it." If you lock one row, only that row is blocked for others. In the Postgres/MySQL demos in the course, one transaction updates a row successfully, and a second transaction trying to update the same row **sits there waiting**, because the row is locked.

**Optimistic concurrency control**
- Usually **doesn't use locks**. No transaction waits for another.
- The benefit is also that **lock management is expensive**. The database must track in memory which row is locked and which is not, and this gets very costly (the instructor mentions row locking across 700,000 rows).
- Instead: "let things play by ear." When transactions start stepping on each other, the database **fails the transaction** and the user (application) should **retry**. The error is usually called a **serialization error**: something changed since the beginning of the transaction, so retry.
- **NoSQL databases** tend to prefer this approach to locking because locking is expensive.

**Level-specific notes**
- **Repeatable read** "locks" the rows it reads, which can be expensive if you read a lot of rows. **Postgres implements repeatable read as a snapshot**, so in Postgres the two levels are identical. That's why you don't get phantom reads in Postgres' repeatable read, while in other databases you do. (The instructor notes this is a quiz question.)
- **Serializable** is usually implemented with **optimistic** concurrency control, because truly running everything one after another would make the database very slow. You can also implement it pessimistically with `SELECT ... FOR UPDATE`: select the row and lock it at the same time.

---

### [EXPANDED] How pessimistic and optimistic isolation really work

*This section is the deeper explanation you asked for. It goes beyond the instructor's lecture, so keep or trim it as you wish.*

#### 1. The core idea in one picture

Concurrent transactions will sometimes want the same data. A database has two basic strategies:

```
                 Two transactions want the same row
                                |
          +---------------------+----------------------+
          |                                            |
  PESSIMISTIC: "Conflicts will happen,         OPTIMISTIC: "Conflicts are rare,
  so prevent them."                            so just detect them."
          |                                            |
  Take a lock BEFORE touching data.            Don't lock. Work on a private view.
  The 2nd transaction WAITS.                   Check for conflicts at write/commit.
          |                                            |
  Cost: waiting, deadlocks,                    Cost: aborted transactions that
  lock bookkeeping.                            must be RETRIED.
```

Neither is "better". Which one is cheaper depends on how often transactions really collide.

#### 2. Pessimistic: locks

**What a lock is.** A lock is a marker the database keeps ("this row is in use by transaction X"). Before a transaction reads or writes a row, it must get a compatible lock. If it can't, it waits.

**The two basic lock modes**

| Mode | Used for | Meaning |
| --- | --- | --- |
| **Shared (S)** | reading | Many transactions can hold it at once. It blocks writers. |
| **Exclusive (X)** | writing | Only one transaction can hold it. It blocks readers that want a lock and other writers. |

**Compatibility matrix** (can the request be granted while another transaction holds the lock?):

| Requested ↓ / Held → | Shared (S) | Exclusive (X) |
| --- | --- | --- |
| **Shared (S)** | ✅ granted | ❌ wait |
| **Exclusive (X)** | ❌ wait | ❌ wait |

**Lock granularity.** Locks can be taken at different sizes:

```
  Table lock     [ ############################################ ]   cheap to track,
                                                                      blocks everyone
  Page lock      [ ##### ][ ##### ][ ##### ][ ##### ]                 middle ground
  Row lock       [#][#][#][#][#][#][#][#][#][#][#][#]                 precise, but many
                                                                      locks to track
```

- **Fine locks (rows)** allow more concurrency, but the database must track many locks in memory.
- **Coarse locks (tables)** are cheap to track, but block many transactions.
- **Lock escalation:** when one transaction accumulates too many fine locks, some databases automatically swap them for one coarse lock to save memory (SQL Server does this, by default at roughly 5,000 locks). That's exactly the "everything got stuck on the table" situation the instructor describes.

**How long are locks held?** This is what separates the isolation levels in a lock-based database:

| Isolation level | Read locks | Write locks |
| --- | --- | --- |
| Read Uncommitted | none | held until commit (some engines) |
| Read Committed | shared lock taken and **released right after the read** | exclusive lock held **until commit/rollback** |
| Repeatable Read | shared locks on rows read, **held until commit** | exclusive locks held until commit |
| Serializable | like Repeatable Read **plus range locks** (a lock on the *gap* where rows could appear) | exclusive locks held until commit |

How this fixes each phenomenon:

- **Dirty read** is prevented because a reader can't get a shared lock on a row that has an exclusive lock held by an uncommitted writer.
- **Non-repeatable read** is prevented because a shared lock held until commit stops anyone from changing the row you read.
- **Phantom read** is prevented only by **range locks** (also called gap or next-key locks). They lock the *space* between index entries so no new row can be inserted into your range. This answers the lecturer's question of how you lock something that doesn't exist yet.
- **Lost update** is prevented because the second writer must wait for the first writer's exclusive lock.

Holding exclusive locks until the end of the transaction is called **strict two-phase locking (strict 2PL)**: locks are only acquired while the transaction is running and are all released together at commit or rollback.

**A pessimistic timeline (lost update avoided with a lock):**

```
        TX1                                      TX2
         |                                        |
   BEGIN                                    BEGIN
         |                                        |
 SELECT qnt FROM sales                            |
 WHERE pid = 1 FOR UPDATE   -> 10                 |
 [takes EXCLUSIVE lock on row]                    |
         |                         SELECT qnt FROM sales
         |                         WHERE pid = 1 FOR UPDATE
         |                         ...BLOCKED, waiting for TX1...
 UPDATE sales SET qnt = 20                        |
 WHERE pid = 1                                    :
         |                                        :
   COMMIT  [lock released]  ------------------>  unblocked, now reads qnt = 20
                                                  |
                                   UPDATE sales SET qnt = 25 WHERE pid = 1
                                                  |
                                            COMMIT
```

Final value: **25**. Both sales are kept. TX2 paid with waiting time instead of a wrong result.

**The price of pessimism: waiting and deadlocks.** If two transactions each hold a lock the other needs, neither can continue:

```
        TX1                                      TX2
         |                                        |
 UPDATE row A  (holds lock on A)          UPDATE row B  (holds lock on B)
         |                                        |
 UPDATE row B  -> waits for TX2           UPDATE row A  -> waits for TX1
         |                                        |
         +----------  DEADLOCK: circular wait  ----+
```

Databases detect this (they look for cycles in a "wait-for" graph) and **abort one transaction** (the "victim") with a deadlock error. The application must retry it. Ways to reduce deadlocks:

- Keep transactions short.
- Always lock rows in the same order (for example, lower account ID first).
- Don't hold locks while waiting for a human (user "think time").

**Useful pessimistic commands**

```sql
SELECT ... FOR UPDATE;               -- lock the rows for writing (exclusive)
SELECT ... FOR SHARE;                -- lock the rows for reading (shared)
SELECT ... FOR UPDATE NOWAIT;        -- fail immediately instead of waiting
SELECT ... FOR UPDATE SKIP LOCKED;   -- skip rows others have locked (job queues)
```

#### 3. Optimistic: detect instead of lock

**The idea.** Let every transaction run without blocking. A transaction works on its own consistent view of the data. When it writes (or commits), the database **validates** that nobody else changed what it relied on. If someone did, the transaction is **aborted** and must be retried.

Classic three phases:

```
  1. READ / WORK      read data, compute changes on a private copy or snapshot
        |
        v
  2. VALIDATE         did anyone change the data I read or want to write?
        |
        +-- no  --> 3. WRITE: apply the changes and commit
        |
        +-- yes --> ABORT with a serialization / conflict error -> RETRY
```

**Optimistic locking in application code (version column).** You can do this yourself in any database, with no special isolation level:

```sql
-- table has an extra column: version INT
SELECT qnt, version FROM sales WHERE pid = 1;      -- returns qnt=10, version=1

UPDATE sales
SET    qnt = 15, version = version + 1
WHERE  pid = 1 AND version = 1;                    -- only succeeds if nobody changed it
-- check "rows affected": 1 = success, 0 = conflict -> re-read and retry
```

**An optimistic timeline:**

```
        TX1                                      TX2
         |                                        |
 SELECT qnt, version -> 10, v1            SELECT qnt, version -> 10, v1
         |                                        |
 UPDATE ... SET qnt=20, version=2                 |
 WHERE pid=1 AND version=1  -> 1 row OK           |
         |                                        |
   COMMIT                                         |
         |                         UPDATE ... SET qnt=15, version=2
         |                         WHERE pid=1 AND version=1  -> 0 rows!
         |                         CONFLICT DETECTED  -> retry
         |                                        |
         |                         SELECT -> 20, v2
         |                         UPDATE ... SET qnt=25, version=3
         |                         WHERE pid=1 AND version=2  -> 1 row OK
         |                                  COMMIT
```

Final value: **25**, with no waiting. TX2 paid by redoing its work.

**The retry loop is mandatory.** With optimistic control, a conflict is a normal event, not a bug. The application should catch the error and retry the whole transaction:

```
repeat up to N times:
    begin transaction
    do the work
    try commit
    if success:        stop
    if serialization / conflict error:  rollback and continue the loop
give up and report an error
```

In PostgreSQL, the error code to watch for is `40001` (`serialization_failure`). In SQL Server, snapshot update conflicts raise error 3960.

#### 4. MVCC: how most databases deliver "snapshot" views

**MVCC (multi-version concurrency control)** is the technique behind snapshot isolation and a big part of how modern databases avoid read locks. The motto is: **readers don't block writers, and writers don't block readers.** Each transaction reads from a consistent version of the data as of a certain point in time. The two common designs are the ones the instructor describes:

```
PostgreSQL: keep OLD and NEW versions side by side in the table

  row version 1:  Product 1, qnt=10   (created by tx 100, replaced by tx 205)
  row version 2:  Product 1, qnt=15   (created by tx 205)

  A transaction that started before tx 205 committed  -> sees version 1 (qnt=10)
  A transaction that starts after tx 205 committed    -> sees version 2 (qnt=15)
  Old versions nobody needs anymore are removed later by VACUUM.


MySQL InnoDB / Oracle: update IN PLACE, keep the old values in an UNDO log

  table:     Product 1, qnt=15                      (always the latest value)
  undo log:  "Product 1 used to be qnt=10"          (how to reverse the change)

  An old snapshot reads the table row, then follows the undo log back
  to rebuild qnt=10.  Long transactions = long undo chains = slower reads.
```

Trade-offs:

- **Postgres style (versions in the table):** old reads are cheap, rollbacks are cheap, but the table accumulates dead rows that must be cleaned up (vacuuming).
- **Undo-log style:** the table stays compact, but old reads have to reconstruct values, and a very long transaction can force the database to keep a lot of undo data (Oracle can fail such reads with a "snapshot too old" error).

**MVCC alone is not full serializability.** Snapshot isolation can still allow **write skew** (see the earlier **[EXTRA]** table). To get true serializable behavior on top of snapshots, PostgreSQL adds **Serializable Snapshot Isolation (SSI)**. It tracks read/write dependencies between concurrent transactions *without blocking them*, and aborts one transaction if it finds a dangerous pattern. This is the lecturer's "serializable is usually implemented optimistically" in practice: it runs fast most of the time, and the price is occasional serialization failures that you must retry.

#### 5. Pessimistic vs optimistic: side by side

| | **Pessimistic (locks)** | **Optimistic (detect + retry)** |
| --- | --- | --- |
| Assumption | Conflicts are likely | Conflicts are rare |
| What happens on conflict | The second transaction **waits** | The second transaction **fails and retries** |
| Typical tools | Row/page/table locks, `SELECT ... FOR UPDATE` | MVCC snapshots, version columns, SSI |
| Overhead | Lock bookkeeping in memory, lock escalation | Keeping versions / undo data, validation at commit |
| Risks | Deadlocks, long waits, blocked readers (in lock-based engines) | Wasted work, retry storms under heavy contention |
| Application must | Keep transactions short, order locks, handle deadlock errors | **Always** handle serialization errors with retries |
| Works best when | Heavy write contention on the same rows (e.g. a hot inventory row, bank account) | Read-heavy workloads, spread-out writes, short transactions |
| Typical examples | SQL Server's default Read Committed, MySQL Serializable, explicit `FOR UPDATE` | PostgreSQL Repeatable Read/Serializable, SQL Server snapshot, many NoSQL stores |

Most real databases are **hybrids**: MVCC for reads (no read locks) plus row locks for writes. Concretely, readers never wait, but two writers to the *same row* still queue.

#### 6. How the popular databases actually behave

*Check your version's documentation, since defaults and options change between releases.*

| Database | Default level | How isolation is typically implemented |
| --- | --- | --- |
| **PostgreSQL** | Read Committed | MVCC with row versions. Read Uncommitted behaves as Read Committed. Repeatable Read = snapshot isolation (no phantoms). Serializable = SSI (optimistic, may raise `40001`). Writers lock rows they update. |
| **MySQL (InnoDB)** | Repeatable Read | MVCC with undo log for normal `SELECT`s. Row locks and next-key (gap) locks for writes and locking reads (`FOR UPDATE`), which also guard against phantoms. Serializable turns plain reads into shared-lock reads (pessimistic). Read Uncommitted is truly supported. |
| **Oracle** | Read Committed | MVCC with undo. Offers Read Committed, Serializable (snapshot-based, errors if it can't serialize) and Read-only. No Read Uncommitted. |
| **SQL Server** | Read Committed | By default **lock-based**: shared locks released after the read. Optional row versioning in `tempdb` via *Read Committed Snapshot* and *Snapshot* isolation (optimistic). Serializable uses range locks. |

#### 7. How to choose in practice

1. **Start with your database's default** (usually Read Committed) and understand what it does *not* protect you from.
2. For **read-then-write** logic (check balance, then debit), don't rely on luck. Use `SELECT ... FOR UPDATE`, an atomic `UPDATE`, or a version check.
3. For **reports** that must be internally consistent, use a read-only transaction at Repeatable Read or Snapshot.
4. For rules that span *multiple rows* (the "at least one doctor on call" kind), use Serializable, or lock explicitly, and be ready to retry.
5. Keep transactions **short**. Both strategies suffer when transactions are long: pessimistic ones hold locks longer, and optimistic ones have more chance to conflict.
6. In both worlds, **write the retry logic** for deadlocks and serialization failures.

*end of [EXPANDED]*

## Summary: Isolation

### Lecture notes: summary

- Isolation is the result of having each transaction run in isolation from other concurrent transactions.
- The consequence is the read phenomena, and the isolation levels were invented to fix them.

### [EXTRA] Setting the isolation level in SQL

```sql
-- PostgreSQL / standard syntax
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- ... queries ...
COMMIT;

-- or set it for the current transaction after BEGIN
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- MySQL: set for the next transaction (or the session)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

*end of [EXTRA]*

---

# Consistency

## Consistency

- Consistency in data
- Consistency in reads

### Lecture notes

- Consistency is one of the properties that gets **traded off** between database platforms (NoSQL vs relational vs graph). Some databases sacrifice consistency for **speed, performance and scalability**.
- There are two kinds:
  1. **Consistency in the data:** the state of the data currently persisted on disk. Is what's on disk consistent with your data model?
  2. **Consistency in reads:** your data may be consistent on disk, but reading it becomes inconsistent, because you have multiple instances (shards, partitions, replicas) that are slightly out of sync. This applies to the **system as a whole**. As a user, you don't care which instance you hit. Your read had better be consistent.

## Consistency in Data

- Defined by the user
- Referential integrity (foreign keys)
- Atomicity
- Isolation

### Lecture notes

- Consistency in data is **defined by the user**, meaning the DBA or whoever builds the data model.
- It mostly comes down to enforcing **referential integrity** (foreign keys). People coming from NoSQL may not use that term, but the concept applies anyway: if one document refers to another document, there must be integrity in that reference (for example, users and followers lists in MongoDB).
- **Atomicity affects consistency:** if the database crashed after debiting one account and the account stays half-debited, you lost $100. That is an inconsistency in the data, and the database is effectively *corrupt*.
- **Isolation affects consistency:** as shown in the isolation lecture, two reads in the same transaction can return different results depending on the isolation level. The data may be correct but keeps changing, so your *view* of it is wrong.

## Consistency in Data (example)

**Pictures**

| ID (PK) | BLOB | LIKES |
| --- | --- | --- |
| 1 | xx | 2 |
| 2 | xx | 1 |

**Picture_Likes**

| USER (PK) | PICTURE_ID (PK)(FK) |
| --- | --- |
| Jon | 1 |
| Edmond | 1 |
| Jon | 2 |

### Lecture notes

- This is an Instagram-style data model. The `Pictures` table stores an ID, the picture as a BLOB (binary large object) and the number of likes. The `Picture_Likes` table records who liked which picture (Jon liked picture 1 and 2, Edmond liked picture 1).
- **The rules defined by the DBA:**
  - The `LIKES` number must equal the number of rows in `Picture_Likes` for that picture. `SELECT COUNT(*) FROM Picture_Likes WHERE PICTURE_ID = 1` must return 2.
  - Every `PICTURE_ID` in the likes table must refer to a picture that exists.
- (The instructor notes that many people would tolerate an out-of-sync like counter, because nobody checks a celebrity's 1.8-million-like picture, but it illustrates the point.)

## Spot inconsistency in this data

**Pictures**

| ID (PK) | BLOB | LIKES |
| --- | --- | --- |
| 1 | xx | 5 |
| 2 | xx | 1 |

**Picture_Likes**

| USER (PK) | PICTURE_ID (PK)(FK) |
| --- | --- |
| Jon | 1 |
| Edmond | 1 |
| Jon | 2 |
| Edmond | 4 |

### Lecture notes: the answers

1. **Picture 1 says 5 likes, but `Picture_Likes` has only 2 rows** for it. The counter is out of sync, which is an inconsistent referential integrity.
2. **Edmond liked picture 4, which doesn't exist** in `Pictures`. Someone deleted the picture without a cascading delete of its likes, leaving **orphaned** rows.

You can have the database enforce these constraints or enforce them at the application level. It's up to you. This is what "consistency" means in **ACID** (the A-C-I-D consistency).

### [EXTRA] Enforcing these rules in the database

```sql
CREATE TABLE pictures (
    id     INT PRIMARY KEY,
    blob   BYTEA,
    likes  INT NOT NULL DEFAULT 0 CHECK (likes >= 0)
);

CREATE TABLE picture_likes (
    user_name   TEXT,
    picture_id  INT REFERENCES pictures(id) ON DELETE CASCADE,   -- no orphans
    PRIMARY KEY (user_name, picture_id)                          -- one like per user per picture
);
```

- The foreign key with `ON DELETE CASCADE` prevents the "Edmond liked picture 4" orphan: deleting a picture also deletes its likes, and inserting a like for a missing picture fails.
- The counter (`likes`) is **derived data**. The database can't automatically know it should equal the row count, so you keep it correct with a transaction (atomicity + isolation do the rest):

```sql
BEGIN;
INSERT INTO picture_likes (user_name, picture_id) VALUES ('Edmond', 2);
UPDATE pictures SET likes = likes + 1 WHERE id = 2;
COMMIT;      -- both happen or neither happens
```

Alternatively, don't store the counter at all and compute `COUNT(*)` when needed, or keep it up to date with a trigger.

*end of [EXTRA]*

## Consistency in reads

```
                                  .-------.
   Update X  ------------------->|         |
                                 |   DB    |     X
   Read X    ------------------->|         |
                                  '-------'
```

(Red arrow: "Update X". Blue arrow: "Read X". Both go to the database, which holds X.)

## Consistency in reads

- If a transaction committed a change, will a new transaction immediately see the change?
- Affects the system as a whole
- Relational and NoSQL databases suffer from this
- Eventual consistency

### Lecture notes

- Picture a system with a reverse proxy in front that forwards requests to the back end. You update value X, the database persists it and commits. The **next read after the commit must return X.** That sounds obvious, but not always.
- The key question: *if a transaction committed a change, will a new transaction immediately see it?* If you commit and then read immediately and don't see the change, the system is inconsistent, because you read something that isn't the value you were supposed to get.
- **Typical cause: replication.** With a primary and several replica (worker) databases, writes go to the primary, which syncs the changes to the replicas. If you read from a replica before the sync arrives, you get the **old value**. (The course has a whole section on replication.)
- **Relational and NoSQL databases both suffer from this** kind of inconsistency. NoSQL suffers from the first kind (data) too. If you don't maintain referential integrity between, say, MongoDB documents, you get exactly the same problem.
- **Eventual consistency** is a "marketing term": "I'm not consistent now, but I'll eventually be consistent."
  - It applies only to **reads**. If you keep reading, you will get old values for a while, but eventually the correct one.
  - It does **not** fix corrupt *data*. If picture 1 says 5 likes but only 2 records exist, no eventual consistency is coming, unless you run some repair or batch job.
  - The instructor says he is torn about the term and will share his thoughts in another lecture. There are many variants (strong eventual consistency, weak eventual consistency), and a whole course could be made on it.
- **Synchronous vs asynchronous replication:** synchronous replication is slower but gives **strong consistency**; asynchronous replication is faster but only gives **eventual consistency**. It comes down to what you want.

### Lecture notes: summary

- **Consistency in data:** enforced by referential integrity, about what is persisted. There is no "eventual" recovery from corrupt data.
- **Consistency in reads:** after an update commits, a new read might still return the old version. Most systems heal from this through eventual consistency as replication completes.
- Both relational and NoSQL databases deal with eventual consistency.

### [EXTRA] Replication lag in a picture

```
   client                primary DB                 replica DB
     |                       |                          |
     |-- UPDATE X = 2 ------>|                          |
     |<-- COMMIT OK ---------|                          |
     |                       |-- replicate X = 2 ------>|   (takes some time)
     |                       |                          |
     |-- SELECT X ---------------------------------------->|
     |<-- X = 1  (STALE!) --------------------------------|
     |                       |                          |
     |                       |        ... replication finishes ...
     |-- SELECT X ---------------------------------------->|
     |<-- X = 2  (eventually consistent) -----------------|
```

Common ways to deal with stale reads:

- **Read your own writes:** after a write, send that user's next reads to the primary (or wait until the replica has caught up).
- **Synchronous replication:** the primary waits for the replica before saying "commit OK" (stronger consistency, slower writes).
- **Accept staleness** where it's harmless (like counters or feeds).

*end of [EXTRA]*

### [EXTRA] Don't confuse the two "C"s

- **C in ACID:** the data satisfies the rules (constraints, referential integrity) after each transaction.
- **C in the CAP theorem:** all nodes of a distributed system return the same, latest data (this is closer to the lecture's "consistency in reads").

Same word, different meanings.

*end of [EXTRA]*

## Summary: Consistency

---

# Durability

## Durability

- Changes made by committed transactions must be persisted in a durable non-volatile storage.
- Durability techniques:
  - WAL - Write ahead log
  - Asynchronous snapshot
  - AOF

### Lecture notes

- Durability is persisting the writes that clients make to **non-volatile storage** (SSD or hard drive). After a commit, even if power is lost, or the client crashes, or the database crashes, or the whole computer crashes, you should still see the change when you come back.
- It sounds obvious, but **a lot of databases play with durability**. Persisting to disk is inherently slow, so some databases keep writes in memory and take snapshots to disk occasionally in the background. That makes writes faster, and many offer it as an option.
- **Relational databases must be durable.** And "changes made by a *committed* transaction": if it isn't committed, all bets are off.
- **Techniques:**
  - **WAL (write-ahead log):** tables and indexes (B-trees, etc.) are what clients read, but they are large, so writing them to disk on every commit takes time. Instead the database writes only the **deltas** (the changes) to a log and **flushes that log to disk immediately**. That guarantees persistence. The table and index files are updated later. After a crash the database replays the WAL to rebuild the state. The database still writes the actual table changes too, but it doesn't rely on that being fast.
  - **Asynchronous snapshots** (or synchronous): everything is kept in memory, and in the background the database snapshots everything to disk at once. Redis uses this.
  - **AOF (append-only file):** very similar to the WAL. It keeps track of the changes and appends them to a file. Redis uses this too. It is a lightweight, fast way to store changes and rebuild the table after a crash.

## Durability - WAL

- Writing a lot of data to disk is expensive (indexes, data files, columns, rows, etc..)
- That is why DBMSs persist a compressed version of the changes known as WAL (write-ahead-log segments)

### Lecture notes

- Think about everything a change could touch: indexes, data files, columns, rows, system files and system tables. The **write amplification** is large, so writing all of it on each commit would be slow.
- So databases persist a **compact, compressed** version of the changes, the **WAL segments**. The WAL has to be persistent because it is what tells the database what to change. It says "we changed this value to that", and it is compressed into tiny segments.

## Durability - OS Cache

- A write request in OS usually goes to the OS cache
- When the writes go the OS cache, an OS crash, machine restart could lead to loss of data
- Fsync OS command forces writes to always go to disk
- fsync can be expensive and slows down commits

### Lecture notes

- **The problem:** when a database asks the OS (Windows or Linux) to write to disk, the OS usually does something "shady": it writes to its **own memory cache** (RAM) and returns "success". The reason is performance: the OS batches many writes and flushes them in one go, which reduces I/O.
- **What can go wrong:** the database says `COMMIT`, asks the OS to write the WAL segment, and the OS says "done" (it only wrote to the cache). The database tells the client "transaction committed". Then the machine crashes. After the restart the data is gone, because it was in RAM. **The database lied**, and in that situation it is not durable.
- **The fix: `fsync`.** This command tells the OS "I don't trust you, flush this directly to disk." Databases call it on the WAL at commit.
- **The cost:** `fsync` is slow, "very, very slow". It bypasses the OS cache, so commits slow down. So databases have to use it smartly.
- **Why is bypassing the cache not dangerous for normal apps?** Saving a Word file or a Notepad document through the cache is fine, because crashes are rare. Databases are different. They must guarantee durability, so they accept the slowdown. "There is always a trade-off."

### Lecture notes: summary

- Durability is the ability to persist changes when the transaction commits. When the database says "committed", you could cut the power at that exact moment and your data must still be there. Where it is stored (SSD, disk, "in the air") doesn't matter. What matters is that you get your data back.
- Many databases trade durability for speed, because a physical medium is slower than memory.
- **Redis** gives an option that, as far as the instructor knows, no other database offers: be **strongly durable**, or be **"eventually durable"** (a made-up term). With the latter, changes are persisted a few seconds later, so a crash or power loss could lose those last few seconds. Some people accept that trade for fast writes, for example for application logs or IoT data, where losing a few records doesn't matter.
- Crashes are rare, so most people never see the loss. But, following Murphy's law, "if it can happen, it eventually will".
- **Next in the course:** an "ACID examples in practice" lecture, running these properties against a real database, plus eventual consistency and the quiz.

### [EXTRA] The commit path with a WAL

```
 client: COMMIT
    |
    v
 [1] change is applied in memory (the database's buffer cache)
 [2] a WAL record describing the change is appended to the log buffer
 [3] the WAL is written AND fsync'ed to disk        <-- the commit becomes DURABLE here
 [4] database answers "COMMIT OK" to the client
    .
    .   later, in the background (checkpoint)
 [5] modified data pages / indexes are written to the data files

 Crash after [3] but before [5]?
    -> on restart the database REPLAYS the WAL and rebuilds the missing changes.
 Crash before [3]?
    -> the transaction was never acknowledged, so it is simply rolled back.
```

And why `fsync` matters:

```
 database --write()--> [ OS page cache (RAM) ] --fsync()--> [ DISK ]
                              ^                                ^
                  "success" returned here                durable only here
                  (lost if the machine crashes)
```

*end of [EXTRA]*

### [EXTRA] Group commit

Since `fsync` is expensive, databases often use **group commit**. When many transactions commit at about the same moment, they share a single `fsync` of the WAL. Each transaction is still durable when it gets its "COMMIT OK", but the cost of the disk flush is split across many commits.

*end of [EXTRA]*

### [EXTRA] Durability dials in real databases

These settings let you trade durability for speed (the lecture's "eventually durable"):

| Database | Setting | Effect |
| --- | --- | --- |
| **PostgreSQL** | `synchronous_commit = on` (default) | commit waits for the WAL flush: fully durable |
| | `synchronous_commit = off` | commit returns before the flush. A crash can lose the last fraction of a second of commits, but the database stays consistent (no corruption) |
| | `fsync = off` | skips flushing entirely. Fast, but **can corrupt the database** after a crash. Only for disposable data |
| **MySQL (InnoDB)** | `innodb_flush_log_at_trx_commit = 1` (default) | flush the log to disk at every commit: fully durable |
| | `= 2` | write to the OS cache at every commit, flush about once per second |
| | `= 0` | write and flush about once per second |
| **Redis** | `appendfsync always` | fsync the AOF on every write: strongest, slowest |
| | `appendfsync everysec` (default) | fsync once per second: up to about one second of writes can be lost |
| | `appendfsync no` | leave flushing to the OS |
| | RDB snapshots | periodic point-in-time snapshots, losing everything since the last snapshot |

*end of [EXTRA]*

## Summary: Durability

---

## Summary

- What is a Transaction?
- Atomicity
- Isolation
- Consistency
- Durability

### [EXTRA] ACID at a glance

| Property | In one sentence | What breaks without it | Typical mechanism |
| --- | --- | --- | --- |
| **Atomicity** | All queries of a transaction succeed, or none of them take effect | Half-finished work, e.g. money debited but never credited | Undo / rollback, crash recovery |
| **Consistency** | Data always satisfies the rules (and reads reflect committed data) | Orphaned rows, wrong counters, stale reads | Constraints, foreign keys, atomicity + isolation, replication settings |
| **Isolation** | Concurrent transactions don't see each other's in-flight mess | Dirty / non-repeatable / phantom reads, lost updates | Locks (pessimistic), MVCC and conflict detection (optimistic) |
| **Durability** | Once committed, data survives crashes and power loss | Acknowledged commits that vanish | WAL + `fsync`, AOF, snapshots, replication |

*end of [EXTRA]*

### [EXTRA] Self-check quiz

The instructor mentions a quiz at the end of the section, but its questions are not in the slides or transcripts. Here is a short practice quiz based on the material.

1. **A transaction reads a row twice and gets different values because another transaction committed in between. Which phenomenon is this?**
   *Answer: Non-repeatable read.*
2. **Which isolation level in PostgreSQL does not allow phantom reads, even though the SQL standard says Repeatable Read may?**
   *Answer: Repeatable Read. Postgres implements it as snapshot isolation.*
3. **Why can't a lock on rows already read prevent phantom reads?**
   *Answer: The new row doesn't exist yet, so there is nothing to lock. You need range (gap) locks or snapshot/versioning.*
4. **What must an application do when an optimistic transaction fails with a serialization error?**
   *Answer: Roll back and retry the whole transaction.*
5. **Why can an acknowledged commit be lost if the database doesn't call `fsync`?**
   *Answer: The data may only be in the OS cache (RAM), which is lost on a crash.*
6. **Picture 1 has `LIKES = 5` but only two rows in `Picture_Likes`. Is this an ACID consistency problem, or an eventual consistency problem?**
   *Answer: ACID consistency (data). Eventual consistency only applies to reads and will not repair corrupt data.*
7. **Two transactions both read 10, add 10 and 5 in the application, and write back. One update is lost. Name two ways to prevent it.**
   *Answer: Use `SELECT ... FOR UPDATE` (pessimistic), or a version check with retry (optimistic). An atomic `UPDATE qnt = qnt + n` also works.*

*end of [EXTRA]*
