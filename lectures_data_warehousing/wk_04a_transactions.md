# DSSA Data Gathering & Warehousing
---

**Instructor**: Carl Chatterton 
**Term**: Fall 2022 
**Module**: 1 
**Week**: 4

---
![img](/assets/img/fault-tolerance.jpg)
---

### Transactions
Implementing fault-tolerant mechanisms require a lot of work.

__The slippery concept of a transaction__
Transactions have been the mechanism of choice for simplifying these issues. Conceptually, all the reads and writes in a transaction are executed as one operation: either the entire transaction succeeds (commit) or it fails (abort, rollback).

The application is free to ignore certain potential error scenarios and concurrency issues (safety guarantees).

__The meaning of ACID__


__Atomicity__ refers to something that cannot be broken down into smaller parts.

Atomicity is not about concurrency. It is what happens if a client wants to make several writes, but a fault occurs after some of the writes have been processed. Abortability would have been a better term than atomicity.

__Consistency__  refers to an application-specific notion of the
database being in a “good state”.

Invariants on your data must always be true. The idea of consistency depends on the application's notion of invariants. Atomicity, isolation, and durability are properties of the database, whereas consistency (in an ACID sense) is a property of the application.

__Isolation__ refers to the executing transactions that are isolated from each other. 

Each transaction can pretend that it is the only transaction running on the entire database, and the result is the same as if they had run serially (one after the other).

__Durability__.
Once a transaction has committed successfully, any data it has written will not be forgotten, even if there is a hardware fault or the database crashes. In a single-node database this means the data has been written to nonvolatile storage. In a replicated database it means the data has been successfully copied to some number of nodes.
Atomicity can be implemented using a log for crash recovery, and isolation can be implemented using a lock on each object, allowing only one thread to access an object at any one time.

A __transaction__ is a mechanism for grouping multiple operations on multiple objects into one unit of execution.

__Handling errors and aborts__
A key feature of a transaction is that it can be aborted and safely retried if an error occurred.

In datastores with leaderless replication, it is the application's responsibility to recover from errors.

__The whole point of aborts is to enable safe retries.__


__Weak isolation levels__
Concurrency issues (race conditions) come into play when one transaction reads data that is concurrently modified by another transaction, or when two transactions try to simultaneously modify the same data.

Databases have long tried to hide concurrency issues by providing transaction isolation.

In practice, is not that simple. Serializable isolation has a performance cost. It's common for systems to use weaker levels of isolation, which protect against some concurrency issues, but not all.

Weak isolation levels used in practice:

Read committed
It makes two guarantees:

When reading from the database, you will only see data that has been committed (no dirty reads). Writes by a transaction only become visible to others when that transaction commits.
When writing to the database, you will only overwrite data that has been committed (no dirty writes). Dirty writes are prevented usually by delaying the second write until the first write's transaction has committed or aborted.
Most databases prevent dirty writes by using row-level locks that hold the lock until the transaction is committed or aborted. Only one transaction can hold the lock for any given object.

On dirty reads, requiring read locks does not work well in practice as one long-running write transaction can force many read-only transactions to wait. For every object that is written, the database remembers both the old committed value and the new value set by the transaction that currently holds the write lock. While the transaction is ongoing, any other transactions that read the object are simply given the old value.

Snapshot isolation and repeatable read
There are still plenty of ways in which you can have concurrency bugs when using this isolation level.

Nonrepeatable read or read skew, when you read at the same time you committed a change you may see temporal and inconsistent results.

There are some situations that cannot tolerate such temporal inconsistencies:

Backups. During the time that the backup process is running, writes will continue to be made to the database. If you need to restore from such a backup, inconsistencies can become permanent.
Analytic queries and integrity checks. You may get nonsensical results if they observe parts of the database at different points in time.
Snapshot isolation is the most common solution. Each transaction reads from a consistent snapshot of the database.

The implementation of snapshots typically use write locks to prevent dirty writes.

The database must potentially keep several different committed versions of an object (multi-version concurrency control or MVCC).

Read committed uses a separate snapshot for each query, while snapshot isolation uses the same snapshot for an entire transaction.

How do indexes work in a multi-version database? One option is to have the index simply point to all versions of an object and require an index query to filter out any object versions that are not visible to the current transaction.

Snapshot isolation is called serializable in Oracle, and repeatable read in PostgreSQL and MySQL.

Preventing lost updates
This might happen if an application reads some value from the database, modifies it, and writes it back. If two transactions do this concurrently, one of the modifications can be lost (later write clobbers the earlier write).

Atomic write operations
A solution for this it to avoid the need to implement read-modify-write cycles and provide atomic operations such us

UPDATE counters SET value = value + 1 WHERE key = 'foo';
MongoDB provides atomic operations for making local modifications, and Redis provides atomic operations for modifying data structures.

Explicit locking
The application explicitly lock objects that are going to be updated.

Automatically detecting lost updates
Allow them to execute in parallel, if the transaction manager detects a lost update, abort the transaction and force it to retry its read-modify-write cycle.

MySQL/InnoDB's repeatable read does not detect lost updates.

Compare-and-set
If the current value does not match with what you previously read, the update has no effect.

UPDATE wiki_pages SET content = 'new content'
  WHERE id = 1234 AND content = 'old content';
Conflict resolution and replication
With multi-leader or leaderless replication, compare-and-set do not apply.

A common approach in replicated databases is to allow concurrent writes to create several conflicting versions of a value (also know as siblings), and to use application code or special data structures to resolve and merge these versions after the fact.

Write skew and phantoms
Imagine Alice and Bob are two on-call doctors for a particular shift. Imagine both the request to leave because they are feeling unwell. Unfortunately they happen to click the button to go off call at approximately the same time.

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
Since database is using snapshot isolation, both checks return 2. Both transactions commit, and now no doctor is on call. The requirement of having at least one doctor has been violated.

Write skew can occur if two transactions read the same objects, and then update some of those objects. You get a dirty write or lost update anomaly.

Ways to prevent write skew are a bit more restricted:

Atomic operations don't help as things involve more objects.
Automatically prevent write skew requires true serializable isolation.
The second-best option in this case is probably to explicitly lock the rows that the transaction depends on.
BEGIN TRANSACTION;

SELECT * FROM doctors
WHERE on_call = true
AND shift_id = 1234 FOR UPDATE;

UPDATE doctors
SET on_call = false
WHERE name = 'Alice'
AND shift_id = 1234;

COMMIT;
Serializability
This is the strongest isolation level. It guarantees that even though transactions may execute in parallel, the end result is the same as if they had executed one at a time, serially, without concurrency. Basically, the database prevents all possible race conditions.

There are three techniques for achieving this:

Executing transactions in serial order
Two-phase locking
Serializable snapshot isolation.
Actual serial execution
The simplest way of removing concurrency problems is to remove concurrency entirely and execute only one transaction at a time, in serial order, on a single thread. This approach is implemented by VoltDB/H-Store, Redis and Datomic.

Encapsulating transactions in stored procedures
With interactive style of transaction, a lot of time is spent in network communication between the application and the database.

For this reason, systems with single-threaded serial transaction processing don't allow interactive multi-statement transactions. The application must submit the entire transaction code to the database ahead of time, as a stored procedure, so all the data required by the transaction is in memory and the procedure can execute very fast.

There are a few pros and cons for stored procedures:

Each database vendor has its own language for stored procedures. They usually look quite ugly and archaic from today's point of view, and they lack the ecosystem of libraries.
It's harder to debug, more awkward to keep in version control and deploy, trickier to test, and difficult to integrate with monitoring.
Modern implementations of stored procedures include general-purpose programming languages instead: VoltDB uses Java or Groovy, Datomic uses Java or Clojure, and Redis uses Lua.

Partitioning
Executing all transactions serially limits the transaction throughput to the speed of a single CPU.

In order to scale to multiple CPU cores you can potentially partition your data and each partition can have its own transaction processing thread. You can give each CPU core its own partition.

For any transaction that needs to access multiple partitions, the database must coordinate the transaction across all the partitions. They will be vastly slower than single-partition transactions.

Two-phase locking (2PL)
Two-phase locking (2PL) sounds similar to two-phase commit (2PC) but be aware that they are completely different things.

Several transactions are allowed to concurrently read the same object as long as nobody is writing it. When somebody wants to write (modify or delete) an object, exclusive access is required.

Writers don't just block other writers; they also block readers and vice versa. It protects against all the race conditions discussed earlier.

Blocking readers and writers is implemented by a having lock on each object in the database. The lock is used as follows:

if a transaction want sot read an object, it must first acquire a lock in shared mode.
If a transaction wants to write to an object, it must first acquire the lock in exclusive mode.
If a transaction first reads and then writes an object, it may upgrade its shared lock to an exclusive lock.
After a transaction has acquired the lock, it must continue to hold the lock until the end of the transaction (commit or abort). First phase is when the locks are acquired, second phase is when all the locks are released.
It can happen that transaction A is stuck waiting for transaction B to release its lock, and vice versa (deadlock).

The performance for transaction throughput and response time of queries are significantly worse under two-phase locking than under weak isolation.

A transaction may have to wait for several others to complete before it can do anything.

Databases running 2PL can have unstable latencies, and they can be very slow at high percentiles. One slow transaction, or one transaction that accesses a lot of data and acquires many locks can cause the rest of the system to halt.

Predicate locks
With phantoms, one transaction may change the results of another transaction's search query.

In order to prevent phantoms, we need a predicate lock. Rather than a lock belonging to a particular object, it belongs to all objects that match some search condition.

Predicate locks applies even to objects that do not yet exist in the database, but which might be added in the future (phantoms).

Index-range locks
Predicate locks do not perform well. Checking for matching locks becomes time-consuming and for that reason most databases implement index-range locking.

It's safe to simplify a predicate by making it match a greater set of objects.

These locks are not as precise as predicate locks would be, but since they have much lower overheads, they are a good compromise.

Serializable snapshot isolation (SSI)
It provides full serializability and has a small performance penalty compared to snapshot isolation. SSI is fairly new and might become the new default in the future.

Pesimistic versus optimistic concurrency control
Two-phase locking is called pessimistic concurrency control because if anything might possibly go wrong, it's better to wait.

Serial execution is also pessimistic as is equivalent to each transaction having an exclusive lock on the entire database.

Serializable snapshot isolation is optimistic concurrency control technique. Instead of blocking if something potentially dangerous happens, transactions continue anyway, in the hope that everything will turn out all right. The database is responsible for checking whether anything bad happened. If so, the transaction is aborted and has to be retried.

If there is enough spare capacity, and if contention between transactions is not too high, optimistic concurrency control techniques tend to perform better than pessimistic ones.

SSI is based on snapshot isolation, reads within a transaction are made from a consistent snapshot of the database. On top of snapshot isolation, SSI adds an algorithm for detecting serialization conflicts among writes and determining which transactions to abort.

The database knows which transactions may have acted on an outdated premise and need to be aborted by:

Detecting reads of a stale MVCC object version. The database needs to track when a transaction ignores another transaction's writes due to MVCC visibility rules. When a transaction wants to commit, the database checks whether any of the ignored writes have now been committed. If so, the transaction must be aborted.
Detecting writes that affect prior reads. As with two-phase locking, SSI uses index-range locks except that it does not block other transactions. When a transaction writes to the database, it must look in the indexes for any other transactions that have recently read the affected data. It simply notifies the transactions that the data they read may no longer be up to date.
Performance of serializable snapshot isolation
Compared to two-phase locking, the big advantage of SSI is that one transaction doesn't need to block waiting for locks held by another transaction. Writers don't block readers, and vice versa.

Compared to serial execution, SSI is not limited to the throughput of a single CPU core. Transactions can read and write data in multiple partitions while ensuring serializable isolation.

The rate of aborts significantly affects the overall performance of SSI. SSI requires that read-write transactions be fairly short (long-running read-only transactions may be okay).

The trouble with distributed systems
Faults and partial failures
A program on a single computer either works or it doesn't. There is no reason why software should be flaky (non deterministic).

In a distributed systems we have no choice but to confront the messy reality of the physical world. There will be parts that are broken in an unpredictable way, while others work. Partial failures are nondeterministic. Things will unpredicably fail.

We need to accept the possibility of partial failure and build fault-tolerant mechanism into the software. We need to build a reliable system from unreliable components.

Unreliable networks
Focusing on shared-nothing systems the network is the only way machines communicate.

The internet and most internal networks are asynchronous packet networks. A message is sent and the network gives no guarantees as to when it will arrive, or whether it will arrive at all. Things that could go wrong:

Request lost
Request waiting in a queue to be delivered later
Remote node may have failed
Remote node may have temporarily stoped responding
Response has been lost on the network
The response has been delayed and will be delivered later
If you send a request to another node and don't receive a response, it is impossible to tell why.

The usual way of handling this issue is a timeout: after some time you give up waiting and assume that the response is not going to arrive.

Nobody is immune to network problems. You do need to know how your software reacts to network problems to ensure that the system can recover from them. It may make sense to deliberately trigger network problems and test the system's response.

If you want to be sure that a request was successful, you need a positive response from the application itself.

If something has gone wrong, you have to assume that you will get no response at all.

Timeouts and unbounded delays
A long timeout means a long wait until a node is declared dead. A short timeout detects faults faster, but carries a higher risk of incorrectly declaring a node dead (when it could be a slowdown).

Premature declaring a node is problematic, if the node is actually alive the action may end up being performed twice.

When a node is declared dead, its responsibilities need to be transferred to other nodes, which places additional load on other nodes and the network.

Network congestion and queueing
Different nodes try to send packets simultaneously to the same destination, the network switch must queue them and feed them to the destination one by one. The switch will discard packets when filled up.
If CPU cores are busy, the request is queued by the operative system, until applications are ready to handle it.
In virtual environments, the operative system is often paused while another virtual machine uses a CPU core. The VM queues the incoming data.
TCP performs flow control, in which a node limits its own rate of sending in order to avoid overloading a network link or the receiving node. This means additional queuing at the sender.
You can choose timeouts experimentally by measuring the distribution of network round-trip times over an extended period.

Systems can continually measure response times and their variability (jitter), and automatically adjust timeouts according to the observed response time distribution.

Synchronous vs ashynchronous networks
A telephone network estabilishes a circuit, we say is synchronous even as the data passes through several routers as it does not suffer from queing. The maximum end-to-end latency of the network is fixed (bounded delay).

A circuit is a fixed amount of reserved bandwidth which nobody else can use while the circuit is established, whereas packets of a TCP connection opportunistically use whatever network bandwidth is available.

Using circuits for bursty data transfers wastes network capacity and makes transfer unnecessary slow. By contrast, TCP dinamycally adapts the rate of data transfer to the available network capacity.

We have to assume that network congestion, queueing, and unbounded delays will happen. Consequently, there's no "correct" value for timeouts, they need to be determined experimentally.

Unreliable clocks
The time when a message is received is always later than the time when it is sent, we don't know how much later due to network delays. This makes difficult to determine the order of which things happened when multiple machines are involved.

Each machine on the network has its own clock, slightly faster or slower than the other machines. It is possible to synchronise clocks with Network Time Protocol (NTP).

Time-of-day clocks. Return the current date and time according to some calendar (wall-clock time). If the local clock is toof ar ahead of the NTP server, it may be forcibly reset and appear to jump back to a previous point in time. This makes it is unsuitable for measuring elapsed time.
Monotonic clocks. Peg: System.nanoTime(). They are guaranteed to always move forward. The difference between clock reads can tell you how much time elapsed beween two checks. The absolute value of the clock is meaningless. NTP allows the clock rate to be speeded up or slowed down by up to 0.05%, but NTP cannot cause the monotonic clock to jump forward or backward. In a distributed system, using a monotonic clock for measuring elapsed time (peg: timeouts), is usually fine.
If some piece of sofware is relying on an accurately synchronised clock, the result is more likely to be silent and subtle data loss than a dramatic crash.

You need to carefully monitor the clock offsets between all the machines.

Timestamps for ordering events
It is tempting, but dangerous to rely on clocks for ordering of events across multiple nodes. This usually imply that last write wins (LWW), often used in both multi-leader replication and leaderless databases like Cassandra and Riak, and data-loss may happen.

The definition of "recent" also depends on local time-of-day clock, which may well be incorrect.

Logical clocks, based on counters instead of oscillating quartz crystal, are safer alternative for ordering events. Logical clocks do not measure time of the day or elapsed time, only relative ordering of events. This contrasts with time-of-the-day and monotic clocks (also known as physical clocks).

Clock readings have a confidence interval
It doesn't make sense to think of a clock reading as a point in time, it is more like a range of times, within a confidence internval: for example, 95% confident that the time now is between 10.3 and 10.5.

The most common implementation of snapshot isolation requires a monotonically increasing transaction ID.

Spanner implements snapshot isolation across datacenters by using clock's confidence interval. If you have two confidence internvals where

A = [A earliest, A latest]
B = [B earliest, B latest]
And those two intervals do not overlap (A earliest < A latest < B earliest < B latest), then B definetively happened after A.

Spanner deliberately waits for the length of the confidence interval before commiting a read-write transaction, so their confidence intervals do not overlap.

Spanner needs to keep the clock uncertainty as small as possible, that's why Google deploys a GPS receiver or atomic clock in each datacenter.

Process pauses
How does a node know that it is still leader?

One option is for the leader to obtain a lease from other nodes (similar ot a lock with a timeout). It will be the leader until the lease expires; to remain leader, the node must periodically renew the lease. If the node fails, another node can takeover when it expires.

We have to be very careful making assumptions about the time that has passed for processing requests (and holding the lease), as there are many reasons a process would be paused:

Garbage collector (stop the world)
Virtual machine can be suspended
In laptops execution may be suspended
Operating system context-switches
Synchronous disk access
Swapping to disk (paging)
Unix process can be stopped (SIGSTOP)
You cannot assume anything about timing

Response time guarantees
There are systems that require software to respond before a specific deadline (real-time operating system, or RTOS).

Library functions must document their worst-case execution times; dynamic memory allocation may be restricted or disallowed and enormous amount of testing and measurement must be done.

Garbage collection could be treated like brief planned outages. If the runtime can warn the application that a node soon requires a GC pause, the application can stop sending new requests to that node and perform GC while no requests are in progress.

A variant of this idea is to use the garbage collector only for short-lived objects and to restart the process periodically.

Knowledge, truth and lies
A node cannot necessarily trust its own judgement of a situation. Many distributed systems rely on a quorum (voting among the nodes).

Commonly, the quorum is an absolute majority of more than half of the nodes.

Fencing tokens
Assume every time the lock server grant sa lock or a lease, it also returns a fencing token, which is a number that increases every time a lock is granted (incremented by the lock service). Then we can require every time a client sends a write request to the storage service, it must include its current fencing token.

The storage server remembers that it has already processed a write with a higher token number, so it rejects the request with the last token.

If ZooKeeper is used as lock service, the transaciton ID zcid or the node version cversion can be used as a fencing token.

Byzantine faults
Fencing tokens can detect and block a node that is inadvertently acting in error.

Distributed systems become much harder if there is a risk that nodes may "lie" (byzantine fault).

A system is Byzantine fault-tolerant if it continues to operate correctly even if some of the nodes are malfunctioning.

Aerospace environments
Multiple participating organisations, some participants may attempt ot cheat or defraud others
