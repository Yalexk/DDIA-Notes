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
