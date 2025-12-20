---
title:  "#7 Finding and Fixing Performance Pathologies in Persistent Memory Software Stacks"
paper_link: https://dl.acm.org/doi/10.1145/3297858.3304077
paper_year: ASPLOS 2019
---

# Background
## File Systems for NVM
The paper distinguishes between native NVM fs, fs designed specifically for NVMs, and adapted fs, block based fs extended to 
implement NVM features. The Linux kernel uses xfs-DAX and ext4-DAX, which are adapted fs. Adapted fs offers limited NVM
optimizations because they still ne to be optimized for disk operations (yes this is kind of vague).

## SQLite (Mine) [2] [3]
SQLite uses either rollback journals (undo) or write-ahead logs (WAL) to implement atomic transactions. For rollback journal,
in DELETE mode, the journal file (one per database) is created and deleted at the start and end of each transaction. In PERSIST mode, 
instead of deleting the log file, it is marked as invalid. Rollback journal logs a copy of the original unchanged database and write
changes directly in place. WAL is the opposite, where the original content is untouched while modifications are appended into the log
file. Multiple transactions can be appended to a single WAL log file (log does not need to reset at commit). Eventually we would like
to flush the changes in the log to the database. This is called checkpointing. SQLite by default checkpoints when the WAL log file 
reaches 1000 pages.

WAL is faster than rollback since its writes are all sequential (to the log) and only require writing the content once (to the WAL log.
I guess the second write to the actual database is delayed. Rollback requires writing twice, once log and once database). WAL allows
user to delay syncing with the disk by sacrificing durability. On the other hand, WAL's read performance degrades as the log grows,
since we need to check all entries to find the lastest value.

## In-Memory Database (Mine) [4]
An in-memory database stores its data, well, in memory. In contrast to dbs that store data on disk, in-memory databases have lower
response time. To persist data, in-memory dbs store each operation in a log residing on disk. Redis is an in-memory database.

# Details 
## Optimally Adapting Applications to NVM 
only focuses on legacy apps built for blk based storage. (so not dram use case)
first hand exp from porting 5 dbs and key-stores to better utilize nvm. identify how to do this for best perf

### SQLite
SQLite offers four logging modes: DELETE, TRUNCATE, PERSIST, and WAL. WAL uses redo logging and the rest undo logging.
Figure 1 shows the SET throughput of each modes under different filesystems. xfs-DAX and ext4-DAX are adapted NVM filesystems
while NOVA is a native NVM fs. In the WAL column, we observe that even with WAL, the two adapted fs has similar performance 
as the three undo logging modes, while NOVA has significantly better performance. This is because xfs-DAX and ext4-DAX keep their
allocator state (I think this is basically which blocks are used by which files) in NVM, so WAL allocations are expensive (since
they need to go to the NVM). This is because block-based fs keep allocator state in the disk to avoid expensive media scan after
a crash (why do the DAX extensions still do this with NVM? See question 1). To combat this, the authors use `fallocate` to preallocate
the WAL log file, which closes the gap between the two adapted fs and NOVA. To get even better performance, the paper proposes
*FiLe Emulation with DAX (FLEX), which allows the fs to access the WAL log in DAX mode (instead of through the fs).*
The key takeaway here is that *the FLEX technique enables adapted NVM fs to achieve similar high performance as a native NVM fs.*

{:refdef: style="text-align: center;"}
![](/assets/images/posts/autopersist/example_change_state.jpg){: width="650" } 
{: refdef}
{:refdef: style="text-align: center;"}
*Figure 2: SQLite SET throughput with different journaling modes [1].*
{: refdef}

### Kyoto Cabinet and LMDB
We won't go into too much details here. Both of these database/libraries already uses mmap to access files on traditional disk 
storage medias. This makes them a natural match for porting to DAX systems, but we can still optimize them more. 
The key takeaway here is that rather than using `msync` (traditional media), we should use CLWB and SFENCE on NVM platforms for 
better performance. This is because cache line flush is on cache line granularity while msync operate on pages.

### Redis and RockDB
These two applications keep data in DRAM and flush to disk only when nessesary. Redis uses an Append Only File (AOF) to log all
write operations. The user may change the AOF flush frequency to trade performance for recoverability. Naively porting 
Redis to NVM is low, since the AOF log append operations are expensive (see SQLite section).
The proposed solution is to make Redis' internal hash table (I am guessing this is how Redis implements key-value store) persistent.
This eliminates the need for the AOF log, because the authors adopts an atomic scheme to update key-value pairs (I think
this might mean no transaction support? See question 2). Hence, by eliminating the AOF, performance is significantly improved.
However, the authors note that making the hash table persistent requries significant programming effort.

Below is a summary of best practices to exploit NVM performance benefits (when porting application from disk to NVM)
- Use FiLe Emulation with DAX (FLEX) to bypass the kernel when writing to log files
- Use cache flushes (fine grained) instead of msync (page granularity)
- Think twice before converting a data structure from volatile to persistent. This may require large amount of effort 
while giving little performance improvements. It would be wise to estimate the maximum performance impact of the change first.
- Preallocate file spaces on NVM to avoid allocation overheads.
- Reduce metadata overheads. Traditional fs uses coarse-grained logging to ensure metadata update consistency. Even if the 
metadata only changes one byte, the fs will still write an entire 4KB page.

### More Notes on FLEX
The authors note that applying FLEX to legacy programs could be simple: 
- FLEX should replace open() with open() + mmap().
- A FLEX write needs to check if the write will increase the file size. If so, must fallocate() + mmap() or mremap().
- FLEX read can be implemented with memcpy() (I am not sure why memcpy. Can we not access the mapped file like an array? Perhaps
this is for reading larger blocks)

## File System Scalability
Coming soon...

# Thoughts and Comments
- Note that this work focuses on porting disk, not DRAM, operations onto NVM. 
- This paper's writing style feels clear and logical. It consistently prepends each list item with first, second, third etc, making
them easy to follow.
- Although the paper provides numerous insights when porting applications to NVM, the amount of required engineering effort to 
achieve them manually seems daunting. The number of lines changed is in the order of 100-1000s, which might be not much for 
big companies, but this is assuming the programmers know exactly what they need to change. More automation would be very desirable.
- The idea of FLEX sounds like just applying DAX... is this novel?

# Questions
1. In the SQLite evaluation, the authors mention that WAL is slow under xfs-DAX and ext4-DAX because these two fs keep their 
allocator state on NVM (while NOVA in DRAM). The reason behind this is that these two adapted fs do this for block-based fs to
avoid expensive media scan after a crash. Why do the NVM extensions still do this then?

2. The authors claim that by making Redis' internal top level hash table persistent, we can eliminate the need for AOF log.
What does this mean for Redis transactions support? Do transactions use a different log to achieve atomicity?

3. The paper's strategy to optimize Redis on NVM is to persist the internal hash table thus eliminating the AOF log. But doesn't this
defeat the purpose of in-memory database, since the database is not in memory anymore (I am assuming the internal hash table 
is a significant portion of the db)? We eliminated AOF from the NVM, but we persisted another (potentially larger) structure. 
How does this improve performance?

# Sources
[1] Jian Xu, Juno Kim, Amirsaman Memaripour, and Steven Swanson. 2019. Finding and Fixing Performance Pathologies in Persistent Memory Software Stacks. In Proceedings of the Twenty-Fourth International Conference on Architectural Support for Programming Languages and Operating Systems (ASPLOS '19). Association for Computing Machinery, New York, NY, USA, 427-439. DOI:https://doi.org/10.1145/3297858.3304077

[2] https://www.sqlite.org/tempfiles.html

[3] https://www.sqlite.org/wal.html 

[4] https://aws.amazon.com/nosql/in-memory/
