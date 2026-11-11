# Chapter 3: Storage and Retrieval

## Data Structures for Databases

### Hash Indexes

- Key-value stores, similar to a Python dictionary
- Example: append-only design. When writing to the database, append the key-value pair to a log file
- An in-memory hash map maps each key to its byte offset in the file

#### Compaction

- Break the log into segments of a certain size, then throw away duplicate keys, keeping only the most recent update
- Merged segments are written to a new file, so existing segments are never modified
- Each segment has its own in-memory hash table, mapping keys to file offsets
- To find the value for a key, check the most recent segment's hash map, then the next most recent, and so on until the key is found

#### Considerations

- **File format**: CSV isn't ideal. A binary format is faster and simpler: encode the length of a string in bytes, followed by the raw string
- **Deleting**: append a special deletion record (a **tombstone**) so the merge process discards previous values for that key
- **Crash recovery**: in-memory hash maps are lost on a crash. They can be rebuilt by reading the segments, or faster by storing snapshots of each segment's hash map on disk
- **Partially written records**: use checksums to detect and ignore records corrupted by a crash mid-write
- **Concurrency control**: have only one writer thread

#### Why not just update files in place?

Pros of append-only:

- Appending and segment merging are sequential writes, which are much faster than random writes
- Concurrency and crash recovery are much simpler if segment files are append-only or immutable
- Merging old segments avoids fragmentation over time

Limitations:

- The hash table must fit in memory
- Range queries are inefficient: each key in the range has to be looked up individually

### SSTables and LSM-Trees

- **Sorted String Table (SSTable)**: like the hash-indexed log segments, but each segment is sorted by key, and each key appears only once per segment file
- Merging segments is simple and efficient (like mergesort). If the same key appears in several segments, keep the newest value and discard the older ones
- To search, you don't need an index of all keys: jump to a nearby key and scan forward. An in-memory index is still required, but it can be sparse
- Records within a range can be grouped into blocks and compressed before writing (saves disk space and I/O, at some CPU cost)

#### Building the storage engine

- Maintaining a sorted structure in memory is easy using a balanced tree, e.g. an AVL tree or red-black tree
- When a write comes in, add it to the in-memory tree (the **memtable**)
- When the memtable gets too big, write it to disk as a new SSTable segment
- To serve a read, check the memtable first, then the most recent on-disk segment, then older ones
- Occasionally run merging/compaction in the background to combine segments and discard overwritten or deleted entries
- Keep a separate log on disk that every write is appended to as it goes into the memtable, so recent writes aren't lost if the database crashes. The log is discarded once its memtable has been written out as a segment

This whole design (memtable + SSTables on disk + background compaction) is a **Log-Structured Merge-Tree (LSM-tree)**. The AVL/red-black tree is just the memtable, not the LSM-tree itself.

### B-Trees

- The most widely used indexing structure
- Like SSTables, B-trees keep key-value pairs sorted by key
- B-trees break the database into fixed-size **blocks/pages** (instead of variable-size segments), traditionally 4 KB, and read or write one page at a time
- Each page can be identified by an address, so pages can refer to each other, forming a tree
  - One page is the **root**, containing keys and references to child pages, each covering a range of keys (e.g. 200–300)
  - Those pages are broken into sub-ranges, until you reach a **leaf page** containing individual keys and their values
- **Updates**: search for the leaf page containing the key, change the value, and write the page back
- **Inserts**: find the page whose range covers the key and add it there. If the page is full, split it into two half-full pages and update the parent to point to both. The parent only splits if it then overflows in turn
- B-trees are always balanced, with a depth of O(log n)
- Writes overwrite pages in place on disk (unlike LSM-trees, which only append)
- A **write-ahead log (WAL)**, or redo log, records every modification before it's applied to the pages, so the tree can be restored after a crash
- **Latches** (lightweight locks) provide concurrency control

#### LSM-trees vs B-trees

- B-tree indexes write data at least twice (WAL + page); LSM-trees also rewrite data multiple times through background merging and compaction
- Rule of thumb: LSM-trees are typically faster for writes, whereas B-trees are thought to be faster for reads

### Other Indexing Structures

- **Secondary indexes** make joins efficient
- **Heap files**: the index stores a reference to where the row lives in a heap file, rather than the row itself
  - This avoids duplicating data when there are several secondary indexes, since each one just references a location in the heap file
  - A value can be updated in place without changing the key, as long as the new value isn't larger than the old one
- **Clustered indexes** store the row data directly within the index
- **Multi-column indexes**
  - **Concatenated indexes** combine several fields into one key by appending one column to another
  - **Multi-dimensional indexes** query several columns at once, e.g. geospatial ranges on a map view
- These techniques assume exact data. When that isn't the case, **fuzzy search** techniques are required

### Keeping Everything in Memory

- Many of the data structures so far are designed around the limitations of disks
- As RAM becomes cheaper, whole datasets can be kept in memory
- In-memory key-value stores like Memcached are used for caching, where it's acceptable for data to be lost if a machine restarts
- In-memory databases have a performance advantage because they avoid the overhead of encoding in-memory data structures into a form that can be written to disk

## Transaction Processing vs Analytics

- Basic access patterns look up a small number of records by key, based on user input. This is **online transaction processing (OLTP)**
- Analytic queries instead scan over a large number of records, usually computing aggregates. This is **online analytic processing (OLAP)**
- Over time it became common to use separate databases for the two, storing data for analytic queries in **data warehouses**

### Data Warehousing

- OLTP systems are usually expected to be highly available with low latency, so expensive scans are avoided because they hurt performance
- For data warehouses, data is extracted from OLTP systems and transformed into an analysis-friendly schema. This process is called **Extract-Transform-Load (ETL)**
- Data warehouses most commonly use a relational model
- Both commonly use SQL, but the internals can look very different, since the warehouse is optimised for analytic queries

### Stars and Snowflakes: Schemas for Analytics

- The **star schema** is a common way to structure a data warehouse
  - A central **fact table** where each row represents an event
  - Fact rows contain foreign key references to **dimension tables**, which represent the who, what, where, when, how and why of the event
  - Also called **dimensional modelling**. The name "star" comes from the diagram: the fact table in the middle, with dimension tables around it like rays
- **Snowflake schemas** are a variation where dimensions are further broken down into sub-dimension tables
- In typical data warehouses, tables are often very wide, with 100+ columns

## Column-Oriented Storage

> TODO
