# DSSA Data Gathering & Warehousing
---

**Instructor**: Carl Chatterton 
**Term**: Fall 2022 
**Module**: 1 
**Week**: 4

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

__Nonrepeatable read or read skew__ occur when you read at the same time you committed a change and you may see inconsistent temporal changes or results.

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

### __Preventing lost updates__
This might happen if the <u>application code</u> reads some value from the database, modifies it, and writes it back. If two transactions do this concurrently, one of the modifications can be lost (later write clobbers the earlier write). Object-relational-mapping frameworks make it easy to accidentally write code which performs unsafe read-modify-write cycles instead of using atomic operations provided by the database. _Its not a problem if you know what your doing but a place to find subtle gus that are difficult to test for._

```python
# example of read-modify-write using python
db = Database.builder(conn='<my connection string>')

# Implements a Builder Pattern for creating a sql query 
counter = db \
    .read(tbl='counters') \
    .where(key='foo') \
    .execute()

# we use the application code to increment the counter    
counter += 1 # <- This is where the problem happens
# The database does not know we have changed the value in the application code
# There is no lock set for this operation. 

# We use the cursor object to update the database
update_query = db \
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

