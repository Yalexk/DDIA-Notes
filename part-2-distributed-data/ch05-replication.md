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

## Problems with Replication Lag

- Leader-based replication suits workloads that are **mostly reads** with only a small percentage of writes (common on the web). Read requests can be spread across followers, so you can scale up read capacity by adding followers. This is a **read-scaling architecture**
- This only works realistically with **asynchronous** replication. With synchronous replication, a single node failing (or a network fault) would block writes for the whole system
- The downside of asynchronous followers: a follower that has fallen behind may return **different results** from the leader (or from other followers), since not all writes have been applied yet
- This inconsistency is only temporary: if you stop writing to the leader and wait, followers will eventually catch up and become consistent with the leader. This is called **eventual consistency**
  - "Eventually" is deliberately vague: there is no limit on how far behind a replica can fall
- The delay between a write happening on the leader and being reflected on a follower is called **replication lag**
  - Normally this is a fraction of a second, but it can grow to seconds or even minutes if the system is operating near capacity or there is a network problem
  - When lag is large, the inconsistencies become a real problem for applications

### Reading Your Own Writes

- Users often submit data (e.g. a comment) and then view what they submitted. Writes go to the **leader**, but the read may be served by a **follower**
- With asynchronous replication, the new data may not have reached that follower yet, so to the user it looks like the data was **lost**
- **Read-after-write consistency** (or **read-your-writes consistency**) guarantees that a user will always see any updates they submitted themselves
  - It makes **no promises about other users**: their updates may not be visible yet
- Techniques for implementing it:
  - When reading something the user **may have modified**, read it from the leader, otherwise read from a follower (e.g. a user's own profile is always read from the leader, other users' profiles from a follower). This requires knowing what might have been modified without querying
  - If most things in the application are potentially editable by the user, this is not effective. Instead, track the **time of the last update**: for one minute after the last update, make all reads from the leader. You can also prevent queries on followers that are more than a minute behind
  - The client remembers the **timestamp of its most recent write**. The system ensures the replica serving reads reflects updates at least up to that timestamp. If it doesn't, the read can be handled by **another replica**, or **wait** until the replica has caught up
    - The timestamp could be a **logical timestamp** (e.g. a log sequence number) or the system clock (which requires clock synchronisation)
- Extra complexity arises when the same user accesses the service from **multiple devices**: **cross-device read-after-write consistency**
  - Approaches that remember the timestamp of the user's last update become harder, since one device doesn't know what updates happened on the other. This metadata needs to be **centralised**
  - If replicas are in different datacenters, there is no guarantee that connections from different devices (e.g. cellular vs. home broadband) are routed to the same datacenter. If an approach requires reading from the leader, requests from **all of the user's devices** must be routed to the datacenter containing the leader

### Monotonic Reads

- Another anomaly with asynchronous followers: a user can see things **moving backward in time**
- This can happen if a user makes several reads from **different replicas**. E.g. a user refreshes a page and the request goes to a random replica. The first replica has a recent comment, but the second replica has more lag, so the comment **disappears** on refresh
- **Monotonic reads** guarantee this doesn't happen: if a user makes several reads in sequence, they will not read older data after having previously read newer data
  - It is a weaker guarantee than strong consistency, but stronger than eventual consistency
- One way to achieve it is to make sure each user always reads from the **same replica** (different users can read from different replicas)
  - The replica can be chosen based on a **hash of the user ID**, rather than randomly
  - If that replica fails, the user's queries need to be rerouted to another replica

### Consistent Prefix Reads

- A third anomaly concerns **causality**, where writes happen in a certain order but are observed in a different one (e.g. an observer sees an answer to a question before the question itself)
- **Consistent prefix reads** guarantee that if a sequence of writes happens in a certain order, anyone reading those writes will see them appear in the **same order**
- This is a particular problem in **partitioned (sharded) databases**:
  - Within a single partition, replication applies writes in the same order, so the problem doesn't occur
  - Different partitions operate **independently**, so there is **no global ordering of writes**. A reader may see some parts of the database in an older state and others in a newer state
- One solution is to make sure any writes that are **causally related** to each other are written to the **same partition**. In some applications this cannot be done efficiently
  - There are also algorithms that explicitly keep track of causal dependencies (see later in this chapter)

### Solutions for Replication Lag

- When working with an eventually consistent system, think about how the application behaves if replication lag increases to **several minutes or even hours**
- If this results in a bad experience for users, it's important to design the system to provide a **stronger guarantee**, such as read-after-write
- Pretending replication is synchronous when it is actually asynchronous is a source of problems
- Applications can provide stronger guarantees than the underlying database (e.g. performing certain reads on the leader), but handling this in application code is **complex** and easy to get wrong
- It would be better if application developers didn't have to worry about these subtle replication issues. This is why **transactions** exist: they are a way for a database to provide stronger guarantees so that the application can be simpler (see Chapter 7)

## Multi-Leader Replication

- Leader-based replication has one major downside: there is only **one leader**, and all writes must go through it. If you can't connect to the leader (e.g. a network interruption), you can't write to the database
- **Multi-leader replication** (also known as **master–master** or **active/active**) is a natural extension: allow **more than one node** to accept writes
  - Each node that processes a write forwards the data change to all the other nodes
  - Each leader simultaneously acts as a **follower** to the other leaders

### Multi-Datacenter Operation

- It rarely makes sense to use a multi-leader setup within a **single datacenter**, because the benefits rarely outweigh the added complexity. There are some situations where it is reasonable, the first being multiple datacenters
- With a normal leader-based setup across multiple datacenters, the leader must be in **one** of the datacenters, and all writes go through it
- In a multi-leader configuration, you can have a **leader in each datacenter**. Within each datacenter, regular leader–follower replication is used. Between datacenters, each datacenter's leader replicates its changes to the leaders in other datacenters
- Comparing single-leader and multi-leader in a multi-datacenter deployment:
  - **Performance**:
    - Single-leader: every write must go over the internet to the datacenter with the leader, adding significant latency
    - Multi-leader: every write is processed in the **local datacenter** and replicated **asynchronously** to the others. The inter-datacenter network delay is hidden from users, so perceived performance is better
  - **Tolerance of datacenter outages**:
    - Single-leader: if the datacenter with the leader fails, failover can promote a follower in another datacenter
    - Multi-leader: each datacenter can continue operating **independently** of the others, and replication catches up when the failed datacenter comes back online
  - **Tolerance of network problems**:
    - Traffic between datacenters usually goes over the public internet, which is typically less reliable than the local network within a datacenter
    - A single-leader configuration is very sensitive to problems on this link, as writes are made synchronously over it. A multi-leader configuration with asynchronous replication can usually tolerate network problems better: a temporary interruption doesn't prevent writes from being processed
- **Downside**: the same data may be **concurrently modified in two different datacenters**, and those write conflicts must be resolved
- Multi-leader replication is often implemented with external tools (e.g. Tungsten Replicator for MySQL, BDR for PostgreSQL, GoldenGate for Oracle) and has subtle configuration pitfalls and surprising interactions with other database features:
  - **Autoincrementing keys**, **triggers** and **integrity constraints** can be problematic
  - For this reason, multi-leader replication is often considered **dangerous territory** that should be avoided if possible

### Clients with Offline Operation

- Another situation for multi-leader replication is an application that needs to keep working while **disconnected from the internet**
- E.g. a calendar app: you need to be able to view and make meetings at any time, even offline. Changes made offline are synced with a server and your other devices when the device is next online
- Every device has a local database that acts as a **leader** (it accepts writes), and there is an asynchronous multi-leader replication process (sync) between the replicas of the calendar on all devices
- This is essentially multi-leader replication **taken to the extreme**: each device is a "datacenter", and the network connection between them is extremely unreliable
  - Replication lag may be **hours or even days**, depending on when the device next has internet access
- Tools such as CouchDB are designed for this mode of operation

### Collaborative Editing

- **Real-time collaborative editing** applications (e.g. Google Docs, Etherpad) allow several people to edit a document **simultaneously**
- This is not usually thought of as a database replication problem, but has a lot in common with offline editing
- When a user edits a document, the changes are applied to their **local replica** (in their browser or client app) instantly, and **asynchronously replicated** to the server and any other users editing the same document
- To guarantee there are **no editing conflicts**, the application must obtain a **lock** on the document before a user can edit it
  - If another user wants to edit the same document, they must wait until the first user has committed their changes and released the lock
  - This is equivalent to **single-leader replication with transactions on the leader**
- For **faster collaboration**, the unit of change should be very small (e.g. a single keystroke) to avoid locking. This allows multiple users to edit simultaneously, but brings all the challenges of **multi-leader replication**, including **conflict resolution**

### Handling Write Conflicts

- The main issue with multi-leader replication is that **write conflicts** can occur, e.g. when two users simultaneously modify the same record, and the writes are sent to two different leaders

#### Synchronous vs. Asynchronous Conflict Detection

- With multi-leader replication, both writes succeed locally, so the conflict is only detected **asynchronously at a later point in time**
- The system could be made synchronous to avoid conflicts (waiting for a write to be replicated to all replicas before telling the user it succeeded), but that **defeats the main advantage** of multi-leader replication (each replica accepting writes independently)

#### Conflict Avoidance

- The simplest strategy is to **avoid conflicts**: ensure all writes for a particular record go through the **same leader**
- This breaks down if you want to **change the designated leader** for a record, e.g. because that datacenter has failed or the user has moved. Concurrent writes on different leaders are possible again

#### Converging Toward a Consistent State

- A single-leader database applies writes in a **defined order**; multi-leader configurations have **no defined ordering** of writes
- The database must resolve conflicts **convergently**: all replicas must arrive at the **same final value** once all changes have been replicated
- Ways of achieving convergent conflict resolution:
  - Give each write a **unique ID** (e.g. timestamp, random number, UUID), pick the write with the highest ID as the winner and discard the others (**last write wins**, LWW). **Prone to data loss**
  - Give each **replica** a unique ID, and let writes originating from a higher-numbered replica take precedence. Also **prone to data loss**
  - Somehow **merge** the values together (e.g. order alphabetically and concatenate)
  - **Record** the conflict in an explicit data structure that preserves all information, and write application code that resolves it later (perhaps by prompting the user)

#### Custom Conflict Resolution Logic

- The most appropriate resolution strategy depends on the application, so many multi-leader systems let you write **conflict resolution code**, executed either on write or on read
- **On write**: as soon as the database detects a conflict in the log of replicated changes, it calls a **conflict handler**
  - Typically runs in a background process and must execute quickly, so it can't prompt a user
- **On read**: when a conflict is detected, **all conflicting writes are stored**. The next time the data is read, the multiple versions are returned to the application, which may resolve the conflict automatically or prompt the user, and then writes the result back

### Multi-Leader Replication Topologies

- A **replication topology** describes the communication paths along which writes are propagated from one node to another
- With more than two leaders, there are several possible paths a write could take
- **All-to-all**: every leader sends its writes to every other leader. The most general topology
- **Circular**: each node receives writes from **one** node and forwards them (plus its own) to **one** other node
- **Star**: one designated root node forwards writes to all of the other nodes. Can be generalised to a tree
- In circular and star topologies a write may pass through several nodes before reaching all replicas, so nodes must forward data from other nodes
  - To prevent infinite replication loops, each write is tagged with the identifiers of the nodes it has passed through, and a node ignores writes already tagged with its own identifier
- **Fault tolerance**:
  - In circular and star topologies, if one node fails it can **interrupt the flow of replication messages** between other nodes, until the topology is reconfigured
  - All-to-all topologies are generally more fault tolerant because messages can travel along **multiple paths**, avoiding a single point of failure
- All-to-all topologies have their own issues: some network links may be **faster than others**, so some replication messages can **overtake** others
  - This causes **causality** problems: e.g. an update to a row may arrive at a replica before the insert that created it
  - Simply attaching timestamps to writes is not enough, as clocks cannot be trusted to be synchronised. **Version vectors** can be used to order events correctly (covered later)

## Leaderless Replication

- Some storage systems abandon the concept of a leader and allow **any replica to directly accept writes** from clients
- In some leaderless implementations, the client **sends its writes directly to several replicas**. In others, a **coordinator node** does this on behalf of the client
  - Unlike a leader, the coordinator does **not enforce a particular ordering** of writes
- This approach was used in Amazon's in-house **Dynamo** system, so leaderless stores modelled on it (Riak, Cassandra, Voldemort) are often called **Dynamo-style**. Some memcached-style caches also use it

### Writing to the Database When a Node Is Down

- Consider a cluster of three replicas where one replica is unavailable (down for a reboot, a crash, a network fault, etc.)
- In a **leader-based** configuration, if the unavailable node is the **leader**, writes cannot continue until a **failover** has promoted one of the followers. An unavailable *follower* doesn't require a failover, as the leader still accepts writes
- In a **leaderless** configuration, **failover does not exist**:
  - The client sends the write to **all replicas in parallel**, and the write is considered successful once a sufficient number of replicas acknowledge it. The missing acknowledgement from the unavailable node is simply **ignored**
  - When that node comes back online and clients start reading from it, it returns **stale** values for any writes it missed
  - To handle this, the client doesn't read from just one replica: reads are also sent to **several replicas in parallel**, and the client may get different responses back
  - Every value is tagged with a **version number**, which lets the client work out which value is newer and which responses are stale

### Read Repair and Anti-Entropy

- After an unavailable node comes back online, how does it catch up on the writes it missed? Two mechanisms are often used together:
- **Read repair**: when a client reads from several nodes in parallel, it can detect any **stale responses** and write the newer value back to the stale replica
  - Works well for values that are **frequently read**
  - A value that is rarely read may stay stale for a long time, which **reduces durability**: it may be missing from all but one or two replicas
- **Anti-entropy process**: a **background process** in the datastore that constantly looks for differences in the data between replicas and copies any missing data from one replica to another
  - Unlike the leader's replication log, it does **not** copy writes in any particular order, and there may be a **significant delay** before data is copied

### Quorums for Reading and Writing

- How many replicas must acknowledge a write for it to be considered successful, and how many must we query for a read?
- If there are **n** replicas, every write must be confirmed by at least **w** nodes, and we must query at least **r** nodes for every read. As long as **w + r > n**, we expect to get an up-to-date value when reading, because at least one of the nodes we read from must have seen the latest successful write (the read and write sets **overlap**). Reads and writes that obey these rules are called **quorum** reads and writes
- A common choice is to make n an **odd** number (typically 3 or 5) and set w = r = (n + 1) / 2, rounded up
- n is usually not the number of nodes in the whole cluster: the dataset is **partitioned** across nodes, and n is the number of replicas **of the value being read or written**. In Dynamo-style databases, n, w and r are **configurable**
- Writes and reads are usually sent to **all n replicas in parallel**; w and r determine how many responses we **wait for** before declaring success
- The quorum condition also sets the failure tolerance:
  - If fewer than w or r nodes are available, writes or reads return an **error**
  - With n = 3, w = 2, r = 2 we can tolerate **one** unavailable node; with n = 5, w = 3, r = 3 we can tolerate **two**
- Setting w and r to smaller numbers (so that w + r ≤ n) means reads are **more likely to return stale values**, but it gives **lower latency** and **higher availability**: the system can still serve reads and writes when more nodes are down

### Limitations of Quorum Consistency

- With w + r > n you generally expect every read to return the most recently written value, but there are **edge cases** where this doesn't hold:
  - With a **sloppy quorum** (discussed later), the w writes may end up on **different nodes** than the r reads, so there is no longer a guaranteed overlap
  - If two writes occur **concurrently**, it is not clear which one happened first. The only safe solution is to **merge** the concurrent writes; if a winner is picked based on a timestamp (**last write wins**), writes can be **lost** due to clock skew
  - If a write happens **concurrently with a read**, the write may be reflected on only **some** of the replicas. It is then undetermined whether the read returns the old or the new value
  - If a write succeeded on fewer than w replicas and was reported as failed, it is **not rolled back** on the replicas where it did succeed. Subsequent reads **may or may not** return the value from that write
  - If a node carrying a new value fails and is restored from a replica carrying an older value, the number of replicas storing the new value may **fall below w**, breaking the quorum condition
  - Even when everything is working correctly, there are edge cases where **unlucky timing** leads to stale reads
- Quorums therefore appear to guarantee that a read returns the latest written value, but in practice it is **not that simple**. Dynamo-style databases are generally optimised for eventually consistent use cases; stronger guarantees (such as reading your own writes, monotonic reads and consistent prefix reads) generally require **transactions** or **consensus**

#### Monitoring Staleness

- Even if an application can tolerate stale reads, it's important to **monitor** the health of replication, so that if it falls behind significantly the cause can be investigated
- In leader-based replication this is fairly easy: replication lag can be measured by comparing positions in the replication log
- In leaderless replication there is **no fixed order** in which writes are applied, which makes monitoring much more difficult

### Sloppy Quorums and Hinted Handoff

- Databases with appropriately configured quorums can tolerate individual nodes being **down or slow**, which makes them appealing for use cases that require **high availability and low latency** and can tolerate occasional stale reads
- However, quorums are not as fault tolerant as they could be: a **network interruption** can cut a client off from a large number of nodes, leaving fewer than w or r of the value's home nodes reachable
- When a quorum of w or r nodes can't be reached, database designers face a trade-off:
  - Return an **error** for all requests that can't reach a quorum
  - Or **accept writes anyway**, and write them to some nodes that are reachable but **aren't among the n nodes** on which the value usually lives
- The second option is a **sloppy quorum**: writes and reads still require w and r successful responses, but those may include nodes that are not among the value's designated n "home" nodes
- Once the network interruption is fixed, the **temporary nodes** send any writes they accepted on behalf of another node to the appropriate home nodes. This is called **hinted handoff**
- Sloppy quorums **increase write availability**, but a read of r nodes is no longer guaranteed to see the latest value, as it may have been written to nodes outside the n
  - A sloppy quorum is really only an assurance of **durability** (the data is stored on w nodes *somewhere*), not a true quorum

#### Multi-Datacenter Operation

- Leaderless replication is also suitable for **multi-datacenter** operation, since it is designed to tolerate conflicting concurrent writes, network interruptions and latency spikes
- **Cassandra**: n includes nodes in **all** datacenters. Each write is sent to all replicas regardless of datacenter, but the client usually only waits for acknowledgement from a quorum of nodes within its **local** datacenter, so it is unaffected by delays on cross-datacenter links
- **Riak**: keeps all client communication local to one datacenter, so n describes the replicas **within one datacenter**. Cross-datacenter replication happens **asynchronously** in the background, similar to multi-leader replication

### Detecting Concurrent Writes

- Dynamo-style databases allow several clients to **concurrently write** to the same key, so conflicts can occur even with strict quorums
- Events may **arrive in a different order** at different nodes, due to variable network delays and partial failures
- If each node simply overwrote the value whenever it received a write, the nodes would become **permanently inconsistent**
- To become **eventually consistent**, replicas should **converge** toward the same value

#### Last Write Wins (Discarding Concurrent Writes)

- One approach is for each replica to store only the **"most recent"** value, allowing older values to be overwritten and discarded
- But when writes are concurrent their order is undefined, so "most recent" is **arbitrary**. We can force an order, e.g. by attaching a **timestamp** to each write and picking the **biggest** timestamp as the most recent. This is **last write wins (LWW)**
- LWW achieves eventual convergence, but at the cost of **durability**:
  - If there are several concurrent writes to the same key, they are all reported as **successful** to the client, but only one survives and the others are **silently discarded**
  - Because of clock skew, LWW may even drop writes that were **not** concurrent
- LWW is a poor choice if **losing data** is not acceptable
- The only safe way to use LWW is to ensure a key is **written only once** and thereafter treated as immutable, avoiding concurrent updates to it. E.g. use a **UUID** as the key, giving every write operation a unique key

#### The "Happens-Before" Relationship and Concurrency

- An operation A **happens before** B if B knows about A, depends on A, or builds upon A in some way
- Two operations are **concurrent** if neither happens before the other (i.e. neither **knows about** the other)
- Given two operations A and B, there are three possibilities: A happened before B, B happened before A, or A and B are concurrent
- Concurrency is **not about physical time overlapping**: two operations can be concurrent even if they happened far apart in time, as long as neither was aware of the other

#### Capturing the Happens-Before Relationship

- A server can determine whether two operations are concurrent by looking at **version numbers**:
  - The server maintains a version number for **every key**, incrementing it every time that key is written, and stores the new version number along with the value written
  - When a client **reads** a key, the server returns **all values that have not been overwritten**, as well as the **latest version number**. A client must read a key before writing
  - When a client **writes** a key, it must include the **version number from the prior read**, and it must **merge** together all values it received in that read
  - When the server receives a write with a particular version number, it can **overwrite** all values with that version number **or below** (since they have been merged into the new value), but it must **keep** all values with a **higher** version number (because those are concurrent with the incoming write)
- A write that includes **no version number** is concurrent with all other writes, so it won't overwrite anything; it is returned as one of the values on subsequent reads

#### Merging Concurrently Written Values

- This algorithm ensures no data is silently dropped, but the client has to do extra work: if several operations happen concurrently, clients must **clean up** afterwards by merging the concurrently written values. Riak calls these concurrent values **siblings**
- A simple approach is to take the **union** of the siblings (e.g. for a shopping cart). But this doesn't work for **removals**: an item removed in one sibling would reappear in the union
- To handle removals, the system leaves a marker with an appropriate version number to indicate the item was removed, called a **tombstone**
- Merging siblings in application code is **complex and error prone**, so there are data structures designed to merge automatically, such as **CRDTs** (conflict-free replicated datatypes), which Riak supports

#### Version Vectors

- The example above used a **single replica**. How does the algorithm change with **multiple replicas** and no leader?
- With multiple replicas accepting writes concurrently, we need a version number **per replica** as well as per key
- Each replica increments **its own** version number when processing a write, and also keeps track of the version numbers it has seen from **each of the other** replicas. This tells it which values to overwrite and which to keep as siblings
- The collection of version numbers from all the replicas is called a **version vector**. Riak uses a variant called a **dotted version vector**
- As before, version vectors are sent from the database to clients when values are **read**, and need to be sent back to the database when a value is subsequently **written**. Riak encodes the version vector as a string it calls the **causal context**
- The version vector lets the database **distinguish between overwrites and concurrent writes**, and makes it safe to read from one replica and subsequently write back to another
- Version vectors are sometimes called **vector clocks**, but the two are subtly different: version vectors are the right structure for comparing the state of replicas
