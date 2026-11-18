# Chapter 5: Replication

- **Replication** means keeping a copy of the same data on multiple machines connected via a network
- Reasons to replicate data:
  - **Latency**: keep data geographically close to users
  - **Availability**: allow the system to keep working even if some of its parts fail
  - **Read throughput**: scale out the number of machines that can serve read queries
- This chapter assumes the dataset is small enough that each machine can hold a copy of the entire dataset
- Copying the data once is easy; the difficulty lies in handling **changes** to replicated data
- There are several approaches to replication, each with its own trade-offs

## Leaders and Followers

- Each node that stores a copy of the database is called a **replica**
- Every write to the database needs to be processed by every replica, otherwise the replicas would no longer contain the same data
- **Leader-based replication** (also known as **active/passive** or **master–slave** replication) is the most common solution:
  - One of the replicas is designated the **leader**. Clients must send all writes to the leader, which first writes the new data to its local storage
  - The other replicas are **followers**. Whenever the leader writes new data, it sends the change to all followers as part of a **replication log** or **change stream**
  - Each follower applies the changes to its local copy **in the same order** as they were processed on the leader
  - Reads can be served by the leader or any follower, but writes are only accepted by the leader

### Synchronous vs. Asynchronous Replication

- Replication can happen **synchronously** (the leader waits for the follower to confirm it received the write before reporting success to the client) or **asynchronously** (the leader sends the change but doesn't wait for a response)
- **Synchronous replication**:
  - **Advantage**: the follower is guaranteed to have an up-to-date copy of the data consistent with the leader, so if the leader fails, the data is still available on the follower
  - **Disadvantage**: if the synchronous follower doesn't respond (because it crashed, or there is a network fault), the write cannot be processed. The leader must block all writes until the synchronous replica is available again
- It is therefore impractical for all followers to be synchronous, as any one node outage would halt the whole system
- In practice, enabling synchronous replication usually means **one** follower is synchronous and the others are asynchronous. This is called **semi-synchronous**
  - If the synchronous follower becomes unavailable or slow, one of the asynchronous followers is made synchronous
  - This guarantees an up-to-date copy of the data on at least two nodes: the leader and one synchronous follower
- **Asynchronous replication**:
  - Replication is normally fast, but there is no guarantee of how long it might take. Followers may fall behind the leader by minutes or more, e.g. if a follower is recovering from a failure, the system is operating near maximum capacity, or there are network problems
  - **Disadvantage**: if the leader fails and is not recoverable, any writes that have not yet been replicated to followers are lost. A write is not guaranteed to be durable, even if it has been confirmed to the client
  - **Advantage**: the leader can continue processing writes even if all of its followers have fallen behind
- Often, leader-based replication is configured to be completely asynchronous

## Setting Up New Followers

- Followers need to be set up regularly, e.g. to replace a failed node, or to increase the number of replicas
- Simply copying the data files from one node to another is typically not sufficient: clients are constantly writing to the database, so different parts of the copy would reflect different points in time, giving an inconsistent result
- Locking the database (making it unavailable for writes) would fix this, but goes against the goal of high availability
- Setting up a follower can usually be done without downtime:
  1. Take a **consistent snapshot** of the leader's database at some point in time (ideally without locking the whole database). Most databases have this feature, as it is also needed for backups
  2. Copy the snapshot to the new follower node
  3. The follower connects to the leader and requests all changes made **since the snapshot** was taken. This requires the snapshot to be associated with an exact position in the leader's replication log (e.g. PostgreSQL's **log sequence number**, MySQL's **binlog coordinates**)
  4. When the follower has processed the backlog of changes, it has **caught up** and can continue to process changes from the leader as they happen
- The exact steps vary significantly between databases. Some automate the whole process, others require a manual multi-step workflow

## Handling Node Outages

- Any node can go down unexpectedly (faults) or due to planned maintenance (e.g. rebooting for a security patch)
- Being able to reboot individual nodes without downtime is a big advantage for operations and maintenance
- The goal is to keep the system running as a whole despite individual node failures, and to minimise the impact of a node outage

### Follower Failure: Catch-up Recovery

- Each follower keeps a log of the data changes it has received from the leader on its local disk
- If a follower crashes and is restarted, or the network between it and the leader is interrupted, it knows the **last transaction it processed** before the fault
- It connects to the leader and requests all changes that occurred after that point. Once it has applied them, it has caught up and continues receiving the change stream as normal

### Leader Failure: Failover

- Handling a failed leader is trickier. One of the followers must be **promoted** to be the new leader, clients must be reconfigured to send writes to the new leader, and the other followers must start consuming changes from the new leader
- This process is called a **failover**, and can happen **manually** (an administrator is notified and takes the steps) or **automatically**
- An automatic failover consists of the following steps:
  1. **Determining that the leader has failed**: there is no foolproof way of detecting what has gone wrong (crash, power outage, network issue, etc.), so most systems use a **timeout**. Nodes frequently bounce messages between each other, and if a node doesn't respond for some period (e.g. 30 seconds), it is assumed to be dead
  2. **Choosing a new leader**: done through an **election** (the remaining replicas choose), or by a previously elected **controller node** appointing the new leader. The best candidate is usually the replica with the **most up-to-date data**, to minimise data loss
  3. **Reconfiguring the system to use the new leader**: clients send their writes to the new leader, and followers consume changes from it. If the old leader comes back, it may still believe it is the leader, so the system needs to make sure it **becomes a follower** and recognises the new leader
- Things that can go wrong in a failover:
  - With **asynchronous replication**, the new leader may not have received all the writes from the old leader before it failed. If the old leader rejoins, what happens to those writes? The common solution is to **discard** them, which may violate clients' durability expectations
  - Discarding writes is especially dangerous if other systems are coordinated with the database's contents (e.g. a database using auto-incrementing primary keys that were reused by the new leader, while an external store like Redis still referenced the old keys, leading to inconsistency)
  - **Split brain**: two nodes both believe they are the leader. If both accept writes and there is no process for resolving conflicts, data is lost or corrupted. Some systems have a mechanism to shut down one node if two leaders are detected, but a poorly designed one can end up shutting down both
  - **Choosing the timeout**: the longer the timeout, the longer the time to recovery. If it is too short, there will be **unnecessary failovers**: a temporary load spike or network glitch can make a healthy leader's response time exceed the timeout. An unnecessary failover makes things worse when the system is already struggling with high load or network problems
- There are no easy solutions to these problems, so some operations teams prefer to perform failovers **manually**, even if the software supports automatic failover

## Implementation of Replication Logs

- There are several different methods for how the leader's changes are sent to followers

### Statement-Based Replication

- The simplest approach: the leader logs every write **statement** (e.g. `INSERT`, `UPDATE`, `DELETE`) and sends it to its followers. Each follower parses and executes the statement as if it had been received from a client
- This can break in several ways:
  - Statements calling a **non-deterministic function** (e.g. `NOW()`, `RAND()`) will likely generate a different value on each replica
  - Statements using an **auto-incrementing column**, or depending on existing data (e.g. `UPDATE ... WHERE <condition>`), must be executed in **exactly the same order** on each replica, which limits concurrently executing transactions
  - Statements with **side effects** (triggers, stored procedures, user-defined functions) may result in different side effects on each replica, unless the side effects are deterministic
- These can be worked around, e.g. the leader can replace non-deterministic function calls with a fixed return value before logging the statement
- However, there are so many edge cases that other replication methods are generally preferred

### Write-Ahead Log (WAL) Shipping

- Storage engines already keep an **append-only log** of all writes (the WAL in B-trees, or the log segments themselves in log-structured engines)
- The leader can send this exact log across the network to its followers, who process it to build a copy of the same data structure as on the leader
- Used in PostgreSQL and Oracle, among others
- **Disadvantage**: the log describes data at a very **low level**: which bytes were changed in which disk blocks. This makes replication tightly coupled to the **storage engine**
  - If the storage format changes between database versions, it is typically not possible to run different versions on the leader and followers
  - This has a big operational impact: it rules out **zero-downtime upgrades** (upgrade followers first, then fail over to an upgraded follower). If the replication protocol doesn't allow a version mismatch, upgrades require downtime

### Logical (Row-Based) Log Replication

- Uses **different log formats** for replication and for the storage engine, so replication is decoupled from the storage engine's internals. This replication log is called a **logical log**, as opposed to the storage engine's *physical* data representation
- A logical log for a relational database is usually a sequence of records describing writes to tables at the **granularity of a row**:
  - **Inserted row**: contains the new values of all columns
  - **Deleted row**: contains enough information to uniquely identify the row, typically the primary key. If there is no primary key, the old values of all columns are logged
  - **Updated row**: contains enough information to identify the row, plus the new values of the columns
- A transaction that modifies several rows generates several log records, followed by a record indicating the transaction was **committed**. MySQL's binlog (with row-based replication) works this way
- Advantages:
  - Decoupled from the storage engine, so it is easier to keep **backward compatible**, allowing leader and followers to run **different software versions or even different storage engines**
  - Easier for **external applications** to parse, e.g. to send data to a data warehouse, build custom indexes and caches. This technique is called **change data capture** (covered in Chapter 11)

### Trigger-Based Replication

- The previous approaches are implemented by the database system, without involving application code
- Sometimes more flexibility is needed, e.g. replicating only a **subset** of the data, replicating from one kind of database to another, or applying **conflict resolution logic**. In these cases, replication may need to move up to the **application layer**
- **Triggers** let you register custom application code that is automatically executed when a data change (write transaction) occurs in the database
  - The trigger can log the change into a separate **table**, from which an external process can read it and apply any custom logic (e.g. replicating it elsewhere)
  - Examples: Databus for Oracle, Bucardo for PostgreSQL
- **Disadvantage**: greater overheads than other replication methods, and more prone to bugs and limitations than the database's built-in replication
- Despite this, its **flexibility** makes it useful
