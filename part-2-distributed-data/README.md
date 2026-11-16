# Part 2: Distributed Data

- Part 1 looked at data systems where data is stored on a single machine. Part 2 asks: what happens when multiple machines are involved in the storage and retrieval of data?
- Reasons to distribute a database across multiple machines:
  - **Scalability**: if the data volume, read load or write load grows bigger than a single machine can handle, the load can be spread across multiple machines
  - **Fault tolerance / high availability**: if the application needs to keep working even when one machine (or several, or a whole datacenter) goes down, multiple machines provide redundancy so another can take over
  - **Latency**: if users are spread around the world, having servers in datacenters close to them means requests don't have to travel halfway around the world, reducing wait times
