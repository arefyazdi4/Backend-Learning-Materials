# ACID

**Atomicity, Consistency, Isolation and Durability in Relational Database Systems**

---

## Agenda

- What is a Transaction?
- Atomicity
- Isolation
- Consistency
- Durability

---

# What is a Transaction?

## Transaction

- A collection of queries
- One unit of work
- E.g. Account deposit (SELECT, UPDATE, UPDATE)

## Transaction Lifespan

- Transaction BEGIN
- Transaction COMMIT
- Transaction ROLLBACK
- Transaction unexpected ending = ROLLBACK (e.g. crash)

## Nature of Transactions

- Usually transactions are used to change and modify data
- However, it is perfectly normal to have a read-only transaction
- Example: you want to generate a report and you want a consistent snapshot based on the time of the transaction
- We will learn more about this in the Isolation section

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


## Summary: What is a Transaction?

---

# Atomicity

## Atomicity

- All queries in a transaction must succeed.
- If one query fails, all prior successful queries in the transaction should roll back.
- If the database went down prior to a commit of a transaction, all the successful queries in the transaction should roll back.

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

Windows blue-screen ("A problem has been detected and Windows has been shut down to prevent damage to your computer... THREAD_STUCK_IN_DEVICE_DRIVER") to illustrate the crash.

## Atomicity (after the restart)

| ACCOUNT_ID | BALANCE |
| --- | --- |
| 1 | $900 |
| 2 | $500 |

- After we restarted the machine, the first account has been debited, but the other account has not been credited.
- This is really bad as we just lost data, and the information is inconsistent.
- An atomic transaction is a transaction that will roll back all queries if one or more queries fail.
- The database should clean this up after restart.

## Summary: Atomicity

---

# Isolation

## Isolation

- Can my in-flight transaction see changes made by other transactions?
- Read phenomena
- Isolation levels

## Isolation - Read phenomena

- Dirty reads
- Non-repeatable reads
- Phantom reads
- Lost updates

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

## Isolation - Isolation Levels for in-flight transactions

- **Read uncommitted** - No isolation, any change from the outside is visible to the transaction, committed or not.
- **Read committed** - Each query in a transaction only sees committed changes by other transactions.
- **Repeatable read** - The transaction will make sure that when a query reads a row, that row will remain unchanged while it is running.
- **Snapshot** - Each query in a transaction only sees changes that have been committed up to the start of the transaction. It is like a snapshot version of the database at that moment.
- **Serializable** - Transactions are run as if they were serialized one after the other.

Each DBMS implements isolation levels differently.

## Isolation Levels vs read phenomena

| Isolation level | Dirty reads | Lost updates | Non-repeatable reads | Phantoms |
| --- | --- | --- | --- | --- |
| Read Uncommitted | may occur | may occur | may occur | may occur |
| Read Committed | don't occur | may occur | may occur | may occur |
| Repeatable Read | don't occur | don't occur | don't occur | may occur |
| Serializable | don't occur | don't occur | don't occur | don't occur |

Source: <https://en.wikipedia.org/wiki/Isolation_(database_systems)>


## Database Implementation of Isolation

- Each DBMS implements isolation levels differently.
- **Pessimistic** - Row-level locks, table locks, page locks to avoid lost updates.
- **Optimistic** - No locks, just track if things changed and fail the transaction if so.
- Repeatable read "locks" the rows it reads but it could be expensive if you read a lot of rows. Postgres implements RR as snapshot. That is why you don't get phantom reads with Postgres in repeatable read.
- Serializable is usually implemented with optimistic concurrency control; you can implement it pessimistically with `SELECT FOR UPDATE`.

## Summary: Isolation

---

# Consistency

## Consistency

- Consistency in data
- Consistency in reads

## Consistency in Data

- Defined by the user
- Referential integrity (foreign keys)
- Atomicity
- Isolation

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

- If a transaction commits a change, will a new transaction immediately see the change?
- Affects the system as a whole
- Relational and NoSQL databases suffer from this
- Eventual consistency

## Summary: Consistency

---

# Durability

## Durability

- Changes made by committed transactions must be persisted in a durable non-volatile storage.
- Durability techniques:
  - WAL - Write-ahead log
  - Asynchronous snapshot
  - AOF

## Durability - WAL

- Writing a lot of data to disk is expensive (indexes, data files, columns, rows, etc.).
- That is why DBMSs persist a compressed version of the changes known as WAL (write-ahead-log segments).

## Durability - OS Cache

- A write request in the OS usually goes to the OS cache.
- When the writes go to the OS cache, an OS crash or machine restart could lead to loss of data.
- The `fsync` OS command forces writes to always go to disk.
- `fsync` can be expensive and slows down commits.

## Summary: Durability

---

## Summary

- What is a Transaction?
- Atomicity
- Isolation
- Consistency
- Durability

Source: <http://Husseinnasser.com>
