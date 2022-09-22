# DSSA Data Gathering & Warehousing
---

**Instructor**: Carl Chatterton 
**Term**: Fall 2022 
**Module**: 1 
**Week**: 7

---

![img](/assets/img/k8s-meme.jpg)

---

The trouble with distributed systems
Faults and partial failures
A program on a single computer either works or it doesn't. There is no reason why software should be flaky (non deterministic).

In a distributed systems we have no choice but to confront the messy reality of the physical world. There will be parts that are broken in an unpredictable way, while others work. Partial failures are nondeterministic. Things will unpredictably fail.

We need to accept the possibility of partial failure and build fault-tolerant mechanism into the software. We need to build a reliable system from unreliable components.

Unreliable networks
Focusing on shared-nothing systems the network is the only way machines communicate.

The internet and most internal networks are asynchronous packet networks. A message is sent and the network gives no guarantees as to when it will arrive, or whether it will arrive at all. Things that could go wrong:

Request lost
Request waiting in a queue to be delivered later
Remote node may have failed
Remote node may have temporarily stopped responding
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

Synchronous vs asynchronous networks
A telephone network establishes a circuit, we say is synchronous even as the data passes through several routers as it does not suffer from queuing. The maximum end-to-end latency of the network is fixed (bounded delay).

A circuit is a fixed amount of reserved bandwidth which nobody else can use while the circuit is established, whereas packets of a TCP connection opportunistically use whatever network bandwidth is available.

Using circuits for bursty data transfers wastes network capacity and makes transfer unnecessary slow. By contrast, TCP dynamically adapts the rate of data transfer to the available network capacity.

We have to assume that network congestion, queueing, and unbounded delays will happen. Consequently, there's no "correct" value for timeouts, they need to be determined experimentally.

Unreliable clocks
The time when a message is received is always later than the time when it is sent, we don't know how much later due to network delays. This makes difficult to determine the order of which things happened when multiple machines are involved.

Each machine on the network has its own clock, slightly faster or slower than the other machines. It is possible to sycnchronize clocks with Network Time Protocol (NTP).

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