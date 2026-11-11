# Chapter 1: Reliable, Scalable and Maintainable Applications

## Reliability

- The system should work as intended even through adversity
- **Faults**: a component deviating from its spec. We want to design fault-tolerant systems
- **Hardware faults** can be minimised by adding redundancy: if a single machine fails, there are hundreds of others
- **Software faults** are often caused when the software makes an assumption about its environment, and they can be systematic

## Scalability

- As the service grows, there should be ways to deal with the growth
- Scalability is the system's ability to cope with increased load
- Load is described by **load parameters**, e.g. number of requests, number of people in a chat
- Performance can be measured by throughput, response time and other metrics
  - It can be measured in averages, but the percentiles are more important
- When scaling up, you can go **vertical** (a bigger machine) or **horizontal** (distributing load)
  - Distributing load across multiple machines is also known as a **shared-nothing architecture**
- Architectures for handling load differ based on the data, so it's good to build around assumptions about which operations will be common

## Maintainability

- As the system is used over time, many different people should be able to work on it and use it productively
- We should avoid creating legacy systems, to minimise pain during maintenance
- Three design principles:
  - **Operability**: easy to keep the system running smoothly
  - **Simplicity**: easy for new engineers to understand the system, by removing complexity
  - **Evolvability** (extensibility): easy for future engineers to make changes
