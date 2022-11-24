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