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

- Post the job, one will find `R` different output files, one per reduce partiton.
    - Typically these do not need to be combined as they are often given as input to another Map-Reduce job.

### Master Data Structures

- For each map task and reduce task, master stores the state: `idle`, `in-progress`, or `completed` along with the identity of the worker for each of the `non-idle` tasks.
- For each **completed** map task, master stores the locations and sizes of the `R` intermediate files produced by the map task.
    - These locations are then given to the reduce workers on allocation of a reduce job.
    - This information is pushed **incrementally** to workers that have `in-progress` reduce tasks.

### Fault-Tolerance

#### Worker Failure

- Master pings every worker periodically.
- If it does not receive a response in a certain amount of time, the master considers the worker to have failed.
- Any map tasks `completed` by the worker are reset to `idle` state and are re-scheduled on other workers.
    - This is done as their output is stored on local disks, which becomes inaccessible during worker failure.
- Similarly, any map or reduce tasks that are being executed on the failed worker are reset to the `idle` state and re-scheduled on the other workers.
- Completed reducer tasks do not need to be re-executed as they are stored on a **global distributed file system**.
- If a map task was first executed on Worker A and then again on Worker B (due to the failure of A), all the reduce workers that have not yet read the data from A read the data from B.

#### Master Failure

- Master writes periodic checkpoints of the master data structures.
- In case of master failure, a new copy of the master is created from the most recent checkpoint.

#### Semantics in Presence of Failures

- If map and reduce functions are deterministic, the Map Reduce framework produces the same output as the sequential executions of these functions on the entire input data.
- Commits of map and reduce task outputs are **atomic**. This is necessary to achieve this feature.
- If a master gets the location of the `R` output files of a map task that has already been completed earlier, it simply ignores this message.
    - In case it is the first time it is receiving this message, it updates its records with the location of these output files.
- When a reduce task is completed, the worker automatically renames the name of its temporary output file to the permanent output file.
    - This renaming of files is also **atomic** to ensure that the file system contains data produced by just one execution of the reduce task.

### Locality

- Input data is stored locally on local disks that are managed by GFS.
- GFS divides data into 64MB blocks and stores 3 copies on different machines.
- Master takes into account the location of the input data and attempts to schedule a map task on a machine that has a replica of the corresponding data.
    - If that is not possible, it schedules the task on a machine that is close to the one that has the corresponding inut split.
    - **Eg**: A worker that is running on the same network switch

### Task Granularity

- `M` and `R` should be much larger than the number of worker machines.
- This enables each worker to perform multiple tasks, improving dynamic load balancing and recovery when a worker fails.
    - Map tasks executed by a failed worker can be spread across multiple workers for re-execution.
- Master makes `O(M + R)` scheduling decisions and has `O(M * R)` state in memory.

### Backup Tasks

- **Straggler**: A machine that takes an unusally long time to execute the last few map tasks or last few reduce tasks, resulting in lengthening the total time for a MapReduce computation.
- This can arise due to the machine having a bad disk, master scheduling too many tasks on that machine leading to CPU and memory contention, etc.
- The master schedules a **backup** execution when the Map-Reduce job is close to completion. 
    - The task is marked as `completed` when either the primary or the backup finish execution.

## Refinements

### Partitioning Function

- Users specify `R`, the number of reduce tasks and output files.
- Data gets partitioned across these `R` tasks by using a partitioning function such as $ hash(key)\ mod\ R $.
    - This basic partition function results in fairly-balanced partitions.
- The partition function can also use elements of the key to partition data.
    - For example, one can want all URLs of a particular host to land in a particular machine.

### Ordering Guarantees

- Within a given partition, the intermediate key-value pairs are sorted in the **increasing** order of the keys.

### Combiner Function

- A **Combiner** function does partial merging of the intermediate key-value pairs before this data is sent over the network.
    - Typically the same code is used for both combiner and reducer functions.
    - Only difference with how Map Reduce handles combiner and reducer is:
        - Output of reducer is sent to an output file.
        - Output of combiner is sent to an intermediate file that is sent to a reduce task.
- The combiner function is executed on the same worker as the map task.
- This is typically useful when the output of a map function has lot of repetition in the keys. 
    - All of these repeated keys will have to be sent over the network.

### Skipping Bad Records

- Bugs in user code may prevent MapReduce jobs from executing.
- Map-Reduce gives the option of detecting when these deterministic errors occur and simply skip these records and move forward.
- Each worker has a signal handler that catches segmentation faults and bus errors.
- Before starting `map`/`reduce` function, the Map Reduce library stores the sequence number of the argument in a global variable.
- When the worker encounters an error, it sends a _last gasp_ UDP packet to the master containing the sequence number.
- If the master gets multiple such packets with the same sequence number, it asks the worker to skip that record and continue with execution.

### Status Information

- Master runs an internal HTTP server and exports status pages.
- These pages contain the progress of computation and links to standard output and standard error files.
- The page also indicates which workers have failed and what map/reduce tasks they were executing at the time.

### Counter

- A counter facility is present in the Map-Reduce library to count a number of events.
- Each individual worker has its own counter, the values of which are periodically sent to the master.
- The master aggregates all these values and returns it to the user program once the Map-Reduce job has been completed.
- These counter values are also displayed on the master status page so that the user can track the computation live.