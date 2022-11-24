# Map Reduce

## Some History

- Most computations are conceptually straightforward.
- As input data is often large, computation has to be distributed across various machines in order to finish in a reasonable amount of time.
- This gives rise to issues such as parallelisation, distribution of data, fault-tolerance and inter-machine computation.
- Map Reduce was introduced to abstract away all these issues so that programmers not familiar with dealing such issues can still use this platform. 
- The `map` and `reduce` functions are based on the `map` and `reduce` primitives already built in to various programming languages such as Lisp and Python.
- Most computations consist of two steps:
    - `Map`: Applying a function to each input, resulting in an intermediate output.
    - `Reduce`: Concatenating all the intermediate values that have the same key.

## Introduction

- Programming model for processing large datasets.
- Users specify a `map` function which produces intermediate key-value pairs. 
- Users specify a `reduce` function which takes intermediate key-value pairs as input and produces output key-value pairs.
- Map Reduce is highly scalable, can work on scale on commodity hardware.
- Run-time system takes care of partitioning the input data, fault-tolerance, scheduling the jobs, and inter-machine communication.

## Programming Model

- `Map`: Takes a set of input key-value pairs and produces intermediate key-value pairs.
    - The Map Reduce library then groups together all intermediate key-value pairs according to their keys and gives it to the `Reduce` function.
- `Reduce`: Takes in intermediate key-value pairs and merges them by key to form a smaller output. 
    - Output is typically either zero or one input values.
    - Intermediate key-value pairs are provided to reducers by an iterator, enabling the handling of a large number of records that would otherwise be too large to fit in memory.

## Map Reduce Computation Examples

### Distributed Grep

- `Map` functions emits a line if it matches the required pattern.
- `Reduce` function just copies this line and emits it as is without any modifications.

### Reverse Web-Link Graph

- `Map` function outputs `(target, source)` pairs for each link to a target URL found in a `source` page.
- `Reduce` function concatenates the output of the mappers by `target` and outputs `(target, list(source))`.

### Inverted Index

- `Map` function parses each document and outputs `(word, docID)`.
- `Reduce` function concatenates by word and outputs `(word, list(docID))`.

### Distributed Sort

- `Map` function extracts key from record and emits `(key, record)`.
- `Reduce` function just prints `(key, record)` pairs.
    - This is done as sorting depends on partitioning and ordering, which is done by the Map Reduce run-time on intermediate key-value pairs.

## Implementation

- Many different implementations are possible depending on the available hardware.
- Google's implementation:
    -  Commodity PCs with dual-processor x86 machines running Linux with 2-4GB of memory per machine.
    - Connected to each other by commodity networking hardware, maxing out at 100Mb/s or 1Gb/s.
    - Cluster consists of anywhere from hundreds to thousands of machines, making machine failure common.
    - Storage provided by standard IDE disks (similar to commodity Hard Drives) attached directly to the physical machines themselves.
        - A in-house distributed file system is running on these disks to provide replication and reliability to unreliable (commodity) hardware.
    - Users submit a job to a scheduling system. Each job has a set of tasks and is mapped by the scheduler to available machines.

### Execution Overview

- Map Reduce library splits the input data into `M` partitions (from 16MB to 64MB per partition) and starts many copies of the program on a cluster of machines.
- One of the copies is the **Master**, which is responsible for scheduling jobs.
    - There are `M` map tasks and `R` reduce tasks. The master takes these tasks and schedules them to idle workers.
- A worker who is assigned a map task reads the contents of the input key-value pair split it receives.
    - It performs the `map` function and produces intermediate key-value pairs.
    - These intermediate pairs are then buffered into memory.
- Periodically, the intermediate key-value pairs are saved to disk in `R` partitions. 
    - The locations of these pairs on disk are passed to the master, who is responsible for forwarding these locations when allocating reduce jobs to idle workers.
- When a worker who is assigned a reduce job is notified by the master about these locations, it uses remote procedure calls to fetch this data.
    - It then sorts by key and groups all occurences of the same key.
    - If this sorting cannot be done in memory, an external sort is used.
- Reduce worker iterates through every intermediate record and passes the value to the `reduce` function.
    - The output is then appended to a consolidated output file that is unique to this specific partition.
- The master wakes up the user program when all the map and reduce jobs are done.
<br>
- Post the job, one will find `R` different output files, one per reduce partiton.
    - Typically these do not need to be combined as they are often given as input to another Map-Reduce job.

