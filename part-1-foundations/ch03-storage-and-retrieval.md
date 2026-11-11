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

- With trillions of rows, storing and querying data efficiently becomes challenging
- Although fact tables have 100+ columns, a typical analytic query only needs a few of them
- A row-oriented store has to load every row in full (all columns) from disk, then parse it and filter out the columns it doesn't need
- **Column-oriented storage** solves this by storing all the values from each column together, e.g. each column in a separate file. A query only reads the columns it uses
- Each column file stores its rows in the same order, so the kth entry in every column belongs to the kth row

### Column Compression

- Columns are often very repetitive, so they compress well, which reduces disk throughput even further
- **Bitmap encoding** works well when the number of distinct values in a column is small compared with the number of rows
  - e.g. a retailer may have billions of sales transactions but only 100,000 distinct products
  - Each distinct value gets its own bitmap, with one bit per row: 1 if the row has that value, 0 if not
- If n (the number of distinct values) is large, most bits in each bitmap are 0, giving a **sparse** bitmap. **Run-length encoding** then stores each bitmap compactly as lengths of runs of 0s and 1s

### Memory Bandwidth and Vectorized Processing

- For a query that scans millions of rows, the bottlenecks are:
  - getting data from disk into memory
  - getting data from main memory into the CPU cache
  - using CPU cycles efficiently
- Column storage reduces the amount of data that has to be loaded
- **Vectorized processing**: the query engine takes a chunk of compressed column data that fits in the CPU's L1 cache and processes it in a tight loop, with no function call per row
- Some operators, such as bitwise AND/OR on bitmaps, work directly on the compressed data without decompressing it first

### Sort Order in Column Storage

- By default, row order doesn't matter: rows can simply be stored in insertion order
- If we do sort, we must sort whole rows, not each column independently, otherwise the kth entries would no longer belong to the same row
- The database administrator chooses the sort columns based on common queries, e.g. sort by date first, then by product
- Sorting helps compression: the first sort column ends up with long runs of the same value, which run-length encoding shrinks to almost nothing. The effect is weaker for the second and third sort keys, whose values are more jumbled
- Different queries benefit from different sort orders, so the same data can be stored sorted in several different ways (introduced by **C-Store**, adopted by Vertica)
  - The data needs replicating across machines for fault tolerance anyway, so each replica can use a different sort order

### Writing to Column-Oriented Storage

- Column storage, sorting and compression make large reads fast, but writes become harder
- Compressed columns can't be updated in place the way B-tree pages can
- Rows are identified by their position, so inserting a row into the middle of a sorted table would mean rewriting every column file
- Solution: **LSM-trees**
  - Writes go to a sorted in-memory store first
  - Once enough rows have accumulated, they are merged with the column files on disk and written out to new files in bulk (Vertica does this)
  - Queries must combine data from memory and disk, but the query engine hides this from the user

### Aggregation: Data Cubes and Materialized Views

- Not all data warehouses use column stores, but they are becoming popular because they make analytic queries much faster
- Warehouse queries often use aggregate functions such as `SUM`, `AVG` and `COUNT`, so it can help to cache frequently used aggregates
- A **materialized view** is one such cache: an actual copy of the query results, written to disk
  - A virtual view, by contrast, is just a shortcut for writing a query
  - A materialized view must be updated whenever the underlying data changes
  - This makes writes more expensive, which is why materialized views are rarely used in OLTP databases but make sense in read-heavy data warehouses
- A **data cube** (OLAP cube) is a special kind of materialized view: a grid of aggregates grouped by different dimensions
  - Certain queries become very fast, because the answer has already been computed
  - Downside: they are inflexible. You can only query by the dimensions the cube was built on. If price isn't a dimension, you can't ask for "sales where price > 100"
  - So most warehouses keep the raw data and use cubes only to speed up common queries
