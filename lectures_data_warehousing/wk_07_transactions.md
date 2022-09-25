# DSSA Data Gathering & Warehousing
---

**Instructor**: Carl Chatterton 
**Term**: Fall 2022 
**Module**: 1 
**Week**: 7

---
![img](/assets/img/fault-tolerance.jpg)
---

### __Transactions__

Implementing fault-tolerant mechanisms require a lot of work.

A __transaction__ is a mechanism for grouping multiple operations on multiple objects into one unit of execution.


Many things can go wrong like:
* Database software or hardware may fail
* Application may crash
* Network Interruptions can happen unexpectedly
* Several clients may write to a database at the same time, causing overwrites
* Clients may read data that is stale or partially updated
* Race conditions between clients can cause bugs

__The slippery concept of a transaction__
Transactions have been the mechanism of choice for simplifying these issues for some time since the 1970s. The implementation details have somewhat changed, but the general idea remains the same. Conceptually, all the reads and writes in a transaction are executed as one operation: either the entire transaction succeeds (commit) or it fails (abort, rollback).

The application is free to ignore certain potential error scenarios and concurrency issues (safety guarantees).

In the late 2000s, NoSQL Databases offered a choice of data models which included replication and partitioning by default, which abandoned transactions entirely or redefine the work to describe a weaker set of guarantees compared to its predecessor. 

### __The meaning of ACID__


#### __Atomicity__

__Atomicity__: refers to something that cannot be broken down into smaller parts.

Atomicity is not about concurrency. It is what happens if a client wants to make several writes, but a fault occurs after some of the writes have been processed. Abortability would have been a better term than atomicity.

#### __Consistency__

__Consistency__  refers to an application-specific notion of the database being in a “good state”.

Invariants on your data must always be true. The idea of consistency depends on the application's notion of invariants. Atomicity, isolation, and durability are properties of the database, whereas consistency (in an ACID sense) is a property of the application.

#### __Isolation__

__Isolation__ refers to the executing transactions that are isolated from each other. 

Each transaction can pretend that it is the only transaction running on the entire database, and the result is the same as if they had run serially (one after the other).

#### __Durability__

__Durability__ refers to providing a safe place where data can be stored without losing it. 

Once a transaction has committed successfully, any data it has written will not be forgotten, even if there is a hardware fault or the database crashes. In a single-node database this means the data has been written to nonvolatile storage. In a replicated database it means the data has been successfully copied to some number of nodes.
Atomicity can be implemented using a log for crash recovery, and isolation can be implemented using a lock on each object, allowing only one thread to access an object at any one time.

---

### __Handling errors and aborts__
A key feature of a transaction is that it can be aborted and safely retried if an error occurred. However, not all databases follow this philosophy. 
  * In datastores with leaderless replication, it is the application's responsibility to recover from errors.

Errors will inevitably happen, and many developers tend to only think about the happy path rather than the intricacies of error handling. 
> For example, many ORM frameworks (like Django in python) do not retry aborted transactions. The error instead results in an exception bubbling up the stack. 

__Handling Errors is not perfect and brings some additional issues:__
* If the transaction succeed, but the network failed to acknowledge the successful commit to the client. (Client thinks it failed). Then retrying the transaction causes it to be performed twice.
* If the error is due to overload, retrying the transaction will make the problem worse not better. Limiting the number of retries using _exponential backoff_ instead.
* If the transaction has side-effects outside of the database, those side-effects may happen even if the transaction aborted. 
* If the client-side process fails while retrying, the data will be lost.

__The whole point of aborts is to enable safe retries.__

---

### __Weak isolation levels__

If two transactions do not touch the same data, they can safely be run in parallel. However, concurrency issues (race conditions) come into play when one transaction reads data that is concurrently modified by another transaction, or when two transactions try to simultaneously modify the same data.

Databases have long tried to hide concurrency issues by providing transaction isolation.

In practice, is not that simple. Serializable isolation has a performance cost. It's common for systems to use weaker levels of isolation, which protect against some concurrency issues, but not all.

Types of Weak isolation levels used in practice:

#### __Read committed__
It makes two guarantees:
* When reading from the database, you will only see data that has been committed __(no dirty reads)__. 
  
* When writing to the database, transaction only become visible to others when that transaction commits __(no dirty writes)__.

__No Dirty Reads:__

Imagine one transaction has written some data to the database, but it has not yet committed or aborted. Can another transaction see that uncommitted data? If yes, that data is called a dirty read.

Why is this a problem?:
* If the transaction updates serval objects in the database, the user may only see some of the updated state. For example, A user sees a new email in their inbox but the inbox counter is not updated. _Seeing the database in a partially updated state causes confusion to users and may cause other transactions to take incorrect decisions_.
 
![img](/assets/img/dirtyreads.png)

__No Dirty Writes:__

Image two transactions concurrently try to update the same object in a database. We don't know in which order the writes will happen, but we assume that the later write overwrites the earlier write. What happens if the earlier write is part of a transaction that has not yet committed to the database so the later write overwrites an uncommitted value?


If transactions update multiple objects in the database, dirty writes can lead to mismatches. Example below shows a mismatch between the Listings tbl and Invoices tbl in a database.

![img](/assets/img/dirtywrites.png)


__implementing read committed__

To prevent dirty writes:
* Dirty writes are prevented usually by delaying the second write until the first write's transaction has committed or aborted.
* Most databases prevent dirty writes by using row-level locks that hold the lock until the transaction is committed or aborted. 
* __Only one transaction can hold the lock for any given object.__

To prevent dirty reads:
* For every object that is written, the database remembers both the old committed value and the new value set by the transaction that currently holds the write lock. 

---

#### __Snapshot isolation and repeatable read__

Read committed is imperfect, and there are still plenty of ways in which you can have concurrency bugs when using this isolation level.

__Non-repeatable read or read skew__ occur when you read at the same time you committed a change and you may see inconsistent temporal changes or results.

There are some situations that cannot tolerate such temporal inconsistencies:

__Backups__: During the time that the backup process is running, writes will continue to be made to the database. If you need to restore from such a backup, inconsistencies can become permanent.
__Analytic queries and integrity checks__: You may get nonsensical results if they observe parts of the database at different points in time.


Scenario: Alice has $1000 of savings at a bank, split evenly across two accounts with $500 each.
* Alice transfers $100 from Account 1 to Account 2
* If she looks at the balances in the same moment as that transaction being processed, she may see inconsistent balance before the incoming transfer has arrived.
  * Incoming Transfer (still shows balance of $500)
  * Outgoing Transfer (shows balance of $400)
  * Alice's total Account balance appears to be $900 instead of $1000
  
To avoid this we must use snapshot isolation.

![img](/assets/img/readskew.png)


__Implementing snapshot isolation__ means that each transaction will read from a consistent snapshot of the database.
* The implementation of snapshots typically utilizes write locks to prevent dirty writes.
* The database must potentially keep several different committed versions of an object (multi-version concurrency control or MVCC).
* Read committed uses a separate snapshot for each query, while snapshot isolation uses the same snapshot for an entire transaction.

__wrapping up__
* Snapshot isolation is useful, especially for read-only transactions.
* Snapshot isolation may go by different names (called serializable in Oracle and repeatable read in PostgreSQL and MySQL)

---

#### __Preventing lost updates__
This might happen if the <u>application code</u> reads some value from the database, modifies it, and writes it back. If two transactions do this concurrently, one of the modifications can be lost (later write clobbers the earlier write). Object-relational-mapping frameworks make it easy to accidentally write code which performs unsafe read-modify-write cycles instead of using atomic operations provided by the database. _Its not a problem if you know what your doing but a place to find subtle gus that are difficult to test for._

```python
# example of read-modify-write using python
q = Query.builder(conn='<my connection string>')

# Implements a Builder Pattern for creating a sql query 
counter = q \
    .read(tbl='counters') \
    .where(key='foo') \
    .execute()

# we use the application code to increment the counter    
counter += 1 # <- This is where the problem happens
# The database does not know we have changed the value in the application code
# There is no lock set for this operation. 

# We use the cursor object to update the database
update_query = q \
    .update(tbl='counters') \
    .set(value=counter) \
    .where(key='foo') \
    .execute()

```

__Atomic write operations__
A solution for this is to avoid implementing read-modify-write cycles in application code and provide atomic operations such as:

```sql
-- Here the DB can utilize locking on the object when it is read, so no other transaction can read it until the update has been applied.
UPDATE counters SET value = value + 1 WHERE key = 'foo';
```


__Explicit locking__

Explicit locking refers to the application code explicitly locking objects that are going to be updated. This way the application can perform a read-modify-write cycle and force any other concurrent transactions to wait until the first read-modify-write cycle is completed.

`For Update` clause is used to indicate that the database should take a lock on all rows returned in the query

```SQL
BEGIN TRANSACTION;

SELECT * FROM figures
  WHERE name = 'robot' AND game_id = 222
  FOR UPDATE;

-- Check whether move is valid, then update the position
-- of the piece that was returned by the previous SELECT.
UPDATE figures SET position = 'c4' WHERE id = 1234;

COMMIT;
```


__Automatically detecting lost updates__

Atomic operations and locks are ways to prevent lost updates by forcing the database to perform read-modify-write cycles sequentially. 

An alternative approach would be to allow them to execute in parallel, if the transaction manager detects a lost update, abort the transaction and force it to retry its read-modify-write cycle.

The advantage to this approach is that the database can perform efficient checks for lost updates in conjunction with snapshot isolation. 


__Compare-and-set__

In databases that don't provide transactions you sometimes find an atomic compare-and-set operation useful to avoid lost updates. 

Compare-and-set works by allowing an update to happen only if the value has not changed since you last read it, otherwise the update has no effect.


__Conflict resolution and replication__
In replicated databases, preventing lost updates becomes more challenging because copies of the data exist on multiple nodes and the data can be modified concurrently. For this reason locks and compare-and-set do not apply.

Instead, a common approach in replicated databases is to allow concurrent writes to create several conflicting versions of a value (also know as siblings), and to use application code or special data structures to resolve and merge these versions after the fact.


__Write skew and phantoms__

So far we have covered two types of race conditions that occur when different transactions try to concurrently write to the same object in a database. 

Imagine Alice and Bob are two on-call doctors for a particular shift. Imagine both doctors request to leave because they are feeling unwell. Unfortunately they happen to click the button to go off call at approximately the same time.


```
ALICE                                   BOB

┌─ BEGIN TRANSACTION                    ┌─ BEGIN TRANSACTION
│                                       │
├─ currently_on_call = (                ├─ currently_on_call = (
│   select count(*) from doctors        │    select count(*) from doctors
│   where on_call = true                │    where on_call = true
│   and shift_id = 1234                 │    and shift_id = 1234
│  )                                    │  )
│  // now currently_on_call = 2         │  // now currently_on_call = 2
│                                       │
├─ if (currently_on_call  2) {          │
│    update doctors                     │
│    set on_call = false                │
│    where name = 'Alice'               │
│    and shift_id = 1234                ├─ if (currently_on_call >= 2) {
│  }                                    │    update doctors
│                                       │    set on_call = false
└─ COMMIT TRANSACTION                   │    where name = 'Bob'  
                                        │    and shift_id = 1234
                                        │  }
                                        │
                                        └─ COMMIT TRANSACTION
```
Since database is using snapshot isolation, both checks return 2. Both transactions commit, and now no doctor is on call. The requirement of having at least one doctor has been violated.

Write skew can occur if two transactions read the same objects, and then update some of those objects. You get a dirty write or lost update anomaly.

Ways to prevent write skew are a bit more restricted:

Atomic operations don't help as things involve more objects.
Automatically prevent write skew requires true serializable isolation.
The second-best option in this case is probably to explicitly lock the rows that the transaction depends on.

```sql
BEGIN TRANSACTION;

SELECT * FROM doctors
WHERE on_call = true
AND shift_id = 1234 FOR UPDATE;

UPDATE doctors
SET on_call = false
WHERE name = 'Alice'
AND shift_id = 1234;

COMMIT;
```

### __Serializability__

This is the strongest isolation level. It guarantees that even though transactions may execute in parallel, the end result is the same as if they had executed one at a time, serially, without concurrency. Basically, the database prevents all possible race conditions.

There are three techniques for achieving this:
* Executing transactions in serial order
* Two-phase locking
* Serializable snapshot isolation.


#### __Actual serial execution__

The simplest way of removing concurrency problems is to remove concurrency entirely and execute only one transaction at a time, in serial order, on a single thread. This approach is implemented by VoltDB/H-Store and Redis.

__Encapsulating transactions in stored procedures__:
With interactive style of transaction, a lot of time is spent in network communication between the application and the database.

For this reason, systems with single-threaded serial transaction processing don't allow interactive multi-statement transactions. The application must submit the entire transaction code to the database ahead of time, as a stored procedure, so all the data required by the transaction is in memory and the procedure can execute very fast.

__There are a few pros and cons for stored procedures__:

  * (Con) Each database vendor has its own language for stored procedures. They usually look quite ugly and archaic from today's point of view, and they lack the ecosystem of libraries.
  * (Con) It's harder to debug, more awkward to keep in version control, trickier to test, and difficult to integrate with monitoring.
  * (Pro) Modern implementations of stored procedures include general-purpose programming languages instead: VoltDB uses Java or Groovy, Datomic uses Java, and Redis uses Lua.

__Partitioning__:
Executing all transactions serially limits the transaction throughput to the speed of a single CPU.

In order to scale to multiple CPU cores you can potentially partition your data and each partition can have its own transaction processing thread.

For any transaction that needs to access multiple partitions, the database must coordinate the transaction across all the partitions. They will be vastly slower than single-partition transactions.


#### __Two-Phase locking__

Two-phase locking (2PL) sounds similar to two-phase commit (2PC) but be aware that they are completely different things.

In 2PL, several transactions are allowed to concurrently read the same object as long as nobody is writing it. When somebody wants to write (modify or delete) an object, exclusive access is required.

Writers don't just block other writers; they also block readers and vice versa. It protects against all the race conditions discussed earlier.

Blocking readers and writers is implemented by a having lock on each object in the database. The lock is used as follows:
* if a transaction wants to read an object, it must first acquire a lock in shared mode.
* If a transaction wants to write to an object, it must first acquire the lock in exclusive mode.
* If a transaction first reads and then writes an object, it may upgrade its shared lock to an exclusive lock.
* After a transaction has acquired the lock, it must continue to hold the lock until the end of the transaction (commit or abort).

__Remember Two-Phase Locking like this:__
<u>__First phase</u> is when the locks are acquired, <u> second phase</u> is when all the locks are released.__

__Downsides to 2PL__:
A situation can occure where transaction A is stuck waiting for transaction B to release its lock, and vice versa (deadlock).

The performance for transaction throughput and response time of queries are significantly worse under two-phase locking than under weak isolation.

A transaction may have to wait for several others to complete before it can do anything.

Databases running 2PL can have unstable latencies, and they can be very slow at high percentiles. One slow transaction, or one transaction that accesses a lot of data and acquires many locks can cause the rest of the system to halt.

__Predicate locks__
With phantoms, one transaction may change the results of another transaction's search query.

In order to prevent phantoms, we need a predicate lock. Rather than a lock belonging to a particular object, it belongs to all objects that match some search condition.

Predicate locks applies even to objects that do not yet exist in the database, but which might be added in the future (phantoms).

__Index-range locks__
Predicate locks do not perform well. Checking for matching locks becomes time-consuming and for that reason most databases implement index-range locking.

It's safe to simplify a predicate by making it match a greater set of objects.

These locks are not as precise as predicate locks would be, but since they have much lower overheads, they are a good compromise.


#### __Serializable snapshot isolation (SSI)__

It provides full serializability and has a small performance penalty compared to snapshot isolation. SSI is fairly new and might become the new default in the future.

__Pessimistic versus optimistic concurrency control:__
* __Two-phase locking__ is called pessimistic concurrency control because if anything might possibly go wrong, it's better to wait.
* __Serial execution__ is also pessimistic as is equivalent to each transaction having an exclusive lock on the entire database.
* __Serializable snapshot isolation__ is optimistic concurrency control technique. Instead of blocking if something potentially dangerous happens, transactions continue anyway, in the hope that everything will turn out all right. The database is responsible for checking whether anything bad happened. If so, the transaction is aborted and has to be retried.

If there is enough spare capacity, and if contention between transactions is not too high, optimistic concurrency control techniques tend to perform better than pessimistic ones.

SSI is based on snapshot isolation: the reads within a transaction are made from a consistent snapshot of the database. On top of snapshot isolation, SSI adds an algorithm for detecting serialization conflicts among writes and determining which transactions to abort.

The database knows which transactions may have acted on an outdated premise and need to be aborted by:

Compared to serial execution, SSI is not limited to the throughput of a single CPU core. Transactions can read and write data in multiple partitions while ensuring serializable isolation.

The rate of aborts significantly affects the overall performance of SSI. SSI requires that read-write transactions be fairly short (long-running read-only transactions may be okay).
