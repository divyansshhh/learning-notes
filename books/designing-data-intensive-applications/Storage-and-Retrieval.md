# Storage and Retrieval

## Hash Indexes

The idea is simple - use a key-value which maintains a log of all the keys written to the database. These keys are then efficiently fetched using a in-memory hash map which contains information about a key's location and it's offset from the start of a database. This allows instant reads to fetch a key. The caveat here is that the indexing hash map must fit into the RAM. This can be paired with a technique called **Compation**. 

### Compation

The on-disk log (aka the database) may contain duplicate keys which would be a waste of space so this so called segment of the database is periodically compacted to create a single smaller database with only one instance of the keys to save space. Compaction can be done on a separate thread allowing reads to happen with the current log and replacing it with the compacted log once the writes are completed. These on-disk logs are called segments. 

Multiple logs (called segments) can also be maintained and a key can be searched in the order of the latest segment to the oldest. These can again be merged and compacted using a background thread and replaced once the compaction is completed. 

### Real Implementation Considerations 

1. **File Format** - Though a CSV format is easy to understand, typically the segments should use a binary format since it's faster and simpler.
2. **Deleting Records** - To delete a key and associated value, a special deletion record must be appended to the segment. This is sometimes called a tombstone.
3. **Crash Recovery** - When restarted, the in-memory hash map is lost and must be restored. This is easy to do by parsing each segment and noting the offset to the latest occurance of a particular key. 
4. **Partially Written Records** - Bitcask (a famous hash index database) uses checksums to ensure the integrity of it's database. 
5. **Concurrency Control** - As writes are done in a purely sequential order, most implementions use a single writer thread. Data file segments are append only and otherwise immutable so they can be read concurrently by multiple threads.


### Advantages 
1. Appending and segment merging are sequential operations which are generally faster than random writes on disk. 
2. Concurrency and crash recovery are simpler if the segments are append only or immutable. You don't have to worry about where a crash happened while a value was being overwritten.
3. Merging old segments avoids the problem of data getting fragmented over time. 

### Limitations 
1. The hash table must fit in memory. It doesn't work for a very large number of keys. You could save the hash map on disk but this doesn't perform well.
2. Range queries on this structure are not efficient.


## SSTables and LSM-Trees

SSTable stands for _Sorted String Table_. This builds on the idea of Hash Indexes by storing keys in a sorted order. In this format, write can occur anywhere. This makes merging multiple segments easy because a simple merge sort like algorithm can be used. You also don't need to store all the keys in an index, you can maintain a sparse index with pointers to only certain keys in the table. To locate any key, find the keys in the index which would contain it and do a linear search in between. A binary search cannot be applied since the records may be of variable length. 

### How do writes work?

A self-balancing tree datastructure like AVL Trees or Red black trees can be useful here. With these data structures, values can be inserted in any order and retrieved in a sorted order. This is sometimes called a _memtable_. When the memtable becomes bigger than a certain size, write it to disk. This can be done efficiently because the tree maintains the sorted order. 

LSM-Tree stands for - _Log Structured Merge Tree_. 


### Performance Optimizations 
1. Checking if a key exists can be slow since a linear search needs to be done before it can be determined if a value exists. This can be avoided by using a _Bloom Filter_ which is an in-memory datastructure which can approximate the contents of a set. 
2. **Size Tiered vs Level Tiered Compation** - In size tiered compaction newer and smaller SSTables are merged into older and larger SSTables. In Level Tiered compaction, the key range is split up into smaller SSTables and older data is moved into separate "levels". This allows compaction to proceed more incrementally and use less disk space. 

The key idea behind a LSMTree is - keep a cascade of SSTables that are merged in the background. This is simple and effective. Since data is sorted, range queries are also quick. 


## B-Tree Indexes

The log structured indexes discussed till now are gaining popularity but are not the most used. The most common datastructure for indexes is - B-Tree. Similar to SSTables, B-Tree keeps the key-value pairs in a sorted order by the key which allows for efficient key-value lookups and range queries. 

Unlike SSTables, which breakdown the database into variable size segments, B-Trees break the database down into fixed size blocks or _pages_ (typically around 4KB in size) and read or write one page at a time. Each page can be referred to using an address or location which allows one page to refer to another (similar to a pointer) but on disk instead of in-memory. 

### Structure of B-Tree 

One page is designated as the root of the B-Tree, whenever you want to look up a key in the index, you start here. The page contains several keys and references to the child pages. Each child is responsible for a continuous range of keys, and the keys between the references indicate where the boundaries between those ranges lie. 

![B-Tree-Example](https://www.cs.cornell.edu/courses/cs3110/2009sp/recitations/images/B-trees.gif)

Eventually we get down to a page containing individual keys (_a leaf page_), which either contains the key inline or contains references to the pages where the values can be found. 

The number of references to the child pages in one page of a B-Tree is called the **branching factor**. Typically this is several hundred. 

### Updating values in a B-Tree 

If you want to update the value for an existing key in B-Tree, you search for the leaf page containing the key, change the value in that page and write the page back to disk (any references to that page remain valid). 
If you want to add a new key, you need to find the page whose range encompasses the new key and add it to that page. If there isn't enough space in the page to accomodate the new key, it is split into two half-full pages and the parent page needs to be updated to account for the update subdivision in the key-ranges. 

This algorithm ensures that the tree remains balanced. 

Most databases can fit into a B-Tree that is three or four levels deep so that you don't have to follow many page references to find the page you are looking for. 
Interestingly - A four level tree of 4KB pages with a branching factor of 500 can store upto 250 TB.

B-Trees have existed for a long time and several optimizations have been found - like pages can store references to siblings to avoid going to the parent to find a sibling. 


## Other Indexing Structures
So far, we have studied key-value indexes which are like _primary keys_ index in a relational database model. 

It is also very common to have _secondary indexes_. For example creating a secondary index on the _user\_id_ column so that you can easily get a row corresponding to a user. Secondary indexes need not be unique, ie. several rows might come under the same index entry. This can be solved in two ways - 
1. Making each value in the index a list of matching row identifiers (eg. list of user_id's)
2. Making each entry unique by appending a row-identifier to it. 

Either way, both B-Trees and log-structured indexes can be used as secondary indexes. 

### Storing values within the index 
The key in the index is what queries search for but the value can be one of two things: the actual row or a reference to the row stored elsewhere. In the latter case, the place where rows are stored is called a _heap file_.

In some situations, the extra hop from the index to the heap file can be too much of a performance penalty for reads, so it's desirable to store the indexed row directly within the index. This is known as a **clustered index**. For eg. in MySQL's InnoDB storage engine, the primary key of a table is always a clustered index and the secondary indexes refer to the primary key (rather than a heap location).

A compromise between a clustered index and a nonclustered index is knows as a **covering index** or **index with included columns**, which stores some of a table's columns within the index. This allows some queries to be answered by using the index alone. 


## Multi Column indexes 

Most common type of multi-column index is called a **concatenated index**, which simply combines several fields into one key by appending one column to another. Consider the example of a phone book which provides _lastname, firstname_ mapping to a _phone number_. A concatenated index with lastname appended to the firstname can be used to find the phone number of everyone with a certain lastname or a particular lastname firstname combination but it fails to answer queries about people with the same first name. 

Consider the case where you have to search for restaurants in a rectangular area. All of the indexing techniques discussed till now can only filter along one axis - either longitude or latitude. This makes it so that the application has to linearly search through the other axis which is clearly inefficient. Most commonly geospatial indexing is done with the use of specialized R-Trees. Eg. PostGIS (an extension to PostgreSQL object relational database system) implements geospatial indexes as R-Trees using PostgreSQL's Generalized Search Tree. 


### Transaction Processing vs Analytics 
Transaction Processing is typically user facing database in which queries interact only with a few records at a time. 
Analytics requirements are different where entire columns need to be scanned to aggregate data to answer queries by analysists. 
This interesting difference in the access patterns makes it so that once the database grows to a certain size, SQL can be used as the query language but the internal implementations of data storage (known as data warehousing in this case) are very different. 
Transaction processing tries to keep entire rows together for easy access to data but analytics processing tries to keep entire columns together so that reads are quicker. 
Another key difference is that analytics queries are typically read-only. 

Transaction processing is typically called - OLTP - _online transaction processing_ while analytics processing is typically called - OLAP - _online analytic processing_. 

This is where ETL pipelines come in, they take the user facing databases, periodically extract, transform and load them into a dataware house where the data is store in a manner that can be easily queired for analytic results (column first manner). 

