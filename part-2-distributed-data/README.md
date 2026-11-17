# Part 2: Distributed Data

- Part 1 looked at data systems where data is stored on a single machine. Part 2 asks: what happens when multiple machines are involved in the storage and retrieval of data?
- Reasons to distribute a database across multiple machines:
  - **Scalability**: if the data volume, read load or write load grows bigger than a single machine can handle, the load can be spread across multiple machines
  - **Fault tolerance / high availability**: if the application needs to keep working even when one machine (or several, or a whole datacenter) goes down, multiple machines provide redundancy so another can take over
  - **Latency**: if users are spread around the world, having servers in datacenters close to them means requests don't have to travel halfway around the world, reducing wait times

## Scaling to Higher Load

- The simplest approach is **vertical scaling (scaling up)**: buy a more powerful machine
- **Shared-memory architecture**: many CPUs, RAM chips and disks are joined together under one operating system and treated as a single machine
  - Cost grows faster than linearly: a machine twice the size costs significantly more than twice as much, and due to bottlenecks may not be able to handle twice the load
  - Offers limited fault tolerance: even with hot-swappable components, it is still confined to a single geographic location
- **Shared-disk architecture**: several machines with independent CPUs and RAM store data on an array of disks shared between them over a fast network
  - Contention and the overhead of locking limit its scalability

### Shared-Nothing Architectures

- In **shared-nothing architectures** (**horizontal scaling** or **scaling out**), each machine or virtual machine running the database is an independent **node** with its own CPUs, RAM and disks
- No special hardware is required; any coordination between nodes is done at the software level over a conventional network
- Shared-nothing systems have many advantages, but they add complexity for the application and can limit the expressiveness of the data models you can use

### Replication vs. Partitioning

- There are two common ways data is distributed across multiple nodes:
  - **Replication**: keeping a copy of the same data on several different nodes, providing redundancy and potentially improving performance
  - **Partitioning (sharding)**: splitting a big database into smaller subsets called **partitions**, so that different partitions can be assigned to different nodes
- These are separate mechanisms, but they often go hand in hand
