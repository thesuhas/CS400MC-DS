# Google File System

- The goals of GFS are:
    - Performance
    - Scalability
    - Reliability
    - Availability

## Design choices that have been re-examined

- Component failures are assumed to be the **norm** rather than the exception.
    - File system has anywhere from hundreds to thousands of storage machines built from **commodity** parts and at any given time, it is virtually guaranteed that are some are not functional and the data has been permanently lost.
    - Constant monitoring, error correction, fault-tolerance, and automatic recovery are integral.
- Files are huge often being Gigabytes in size.
    - When we have datasets in the Terabytes, it is inefficient to manage multiple kilobyte-sized files even if the file system can support it.
    - I/O operation and block sizes have been revised as a result.
- Most files are updated by appending new data rather than overwriting existing data.
    - Random writes are practically **non-existent**.
    - Files are only read sequentially.


## Design Overview

### Assumptions

- System is built from commodity hardware that can fail frequently.
- System stores a modest number of files. Expected to store a couple million files, each file being about 100MB or a bit larger in size.
- Workloads consist of two kinds of reads:
    - **Large Streaming Reads**: Individual operations read hundreds of KBs. Successive operation from the same client often read contiguous regions of a file.
    - **Small Random Reads**: Typically reads a few KBs at some arbitrary offset.
- Workloads have many large, sequential writes that append information to files.
    - Once written, files are seldom modified.
- System must efficiently implement well-defined semantics for multiple clients that concurrently append to the same file.
    - Must ensure that these operations are atomic with minimal synchronisation overhead.
- High sustained bandwith is more important than low latency.

### Interface

- GFS does provide a similar file system interface but does not implement standard API.
- Files are organised hierarchicaly in directories and identified by path-names.
- **Snapshot**: creates a copy of a file or directory tree for a low cost.
- **Record Append**: Allows multiple clients to append data to the same file concurrently while guaranteeing atomicity of each individual client's append.

### Architecture

- Has a single `master`, multiple `chunk servers` and multiple `clients`.
- `clients` and `chunk servers` can lie on the same machine provided it has enough resources.
- Files are divided into 64MB `chunks` at the time of `chunk` creation. Each `chunk` has a unique and immutable `chunk handle`.
- `chunk servers` store `chunks` on disk as linux files and read or write data based on `chunk handle` and byte range.
- Each `chunk` is replicated on three replicas.
- The `master` contains all the metadata. Includes the namespace, access control information, mapping from file to chunks, and the current locations of the chunks.
- The `master` also periodically communicates with each `chunk server` via heartbeats to collect their states and give instructions.
- All metadata operations happens with the `master` but all read/write communication goes to the `chunk` servers directly.
- `client` and `chunk sever` **do not** cache any data as more often than not, the data is too large to be cached.

### Single Master

- Having a single master can lead to bottlenecks if the reads and writes are routed through the master.
- To combat this, a client talks to the master and gets the information of the chunk servers it needs to combat for read/write operations.
    - It then temporarily caches these chunk servers and communicates with them directly for all future operations.

#### Steps involved in a simple read

- Applicaaion gives the client a file name and a byte offset.
- Client translates this to a file name and a chunk index.
- Client sends the master a request with the file name and chunk index.
- Master replies with chunk handle and locations of the replicas.
- Client temporarily caches this info.
- Client sends a request to the nearest replica with a chunk handle and the byte range within the chunk.
- Further reads of same chunk require no client-master interaction till the cached info expires or the file is reopened.
- Client asks master for multiple chunks in one request reducing the number of client-master interactions required.