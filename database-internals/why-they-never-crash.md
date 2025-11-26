# Why Databases Never Loose Your Data: A Not So Deep Dive Into Fsync, WAL amd Storage Guarantees

You are software engineer with some good years of experience, and your interviewer asks you, _"Your app just wrote a user's balance update, and the power goes off, why doesn't the database lose the update or get corrupt?"_. How would you have answered?

In today's piece, we'll explore the reasoning behind the question and uncover how our database works internally to keep data secure. We will be exploring topics like **WAL**, **Redo LOG**, **Undo Log**, **MVCC(Multi Version Concurrency Control)**, and **Fsync**.

A lot of people think that as soon as you issue a **DML(Data Manipulation Command) command** like **INSERT**, **UPDATE**, or **DELETE**, they believe _the data is saved or their commit has been persisted._ Well, this depends on your database configuration. By default, most databases ensure durability by flushing to disk, but the data isn't guaranteed to survive a crash unless **Fsync** happens (or is configured to happen).

## What The **** Is Fsync?

Well, you see our OS hardly loves writing to disk, because this operation is expensive. It's a lot of I/O, and it's true cause writing to the disk is slower compared to writing to the RAM. To optimize performance, our OS uses a **page cache** (also called buffer cache) - it keeps recently written data in RAM and batches those writes before flushing them to disk. This allows the OS to:

- Coalesce multiple small writes into larger, more efficient disk operations
- Reorder writes for better disk performance
- Reduce the number of expensive disk I/O operations

The OS flushes dirty pages (modified data in the page cache) to disk based on various policies such as;

- when memory pressure occurs.
- when dirty page thresholds are reached.

or periodically via background writeback. However, this means data might sit in RAM for seconds or even minutes before being written to disk.

So you ask, well if the power goes off, or someone yanks off the battery from my PC, the changes are gone! Lols, Our database designers thought about this problem, and the solution? **Fsync!!!!**. **Fsync** is a system call that forces the OS to immediately flush those buffers (dirty pages in the page cache) to disk and wait for the write to complete before returning. 

**Fsync** doesn't bypass the OS. It's part of the OS API, but it ensures data reaches the physical disk (or at least the disk's write cache if it has battery backup) before the call returns. For MySQL and PostgreSQL, this behaviour or option is customizable. You could tune **Fsync** to flush on every COMMIT for PostgreSQL (via `synchronous_commit = on`), and you could set this to be deferred or batched in MySQL via the **innodb_flush_log_at_trx_commit = 2**.  

**Fsync** behavior is controlled separately by durability settings like `synchronous_commit` in PostgreSQL and `innodb_flush_log_at_trx_commit` in MySQL. These settings control how often happens and you could even turn them off tbh but you risk loosing committed transactions or corrupting the entire database.

Hold up Stephen, so what does the **Fsync** flush?

## WAL && Redo Log

One thing that gets me a lot in computer science is how we often times can't agree on the same term to label something with. 

The WAL(In PostgreSQL) and the Redo Log(In MySQL) actually mean the same thing technically. The WAL stands for Write Ahead Log, and the Redo Log is just well, it's just the Redo Log.

So we now understand that **Fsync** forces the OS to flush buffered data to disk, but what data does the database actually ask the OS to flush?

When we issue a **DML(Data Manipulation Command) command** like **INSERT**, **UPDATE**, or **DELETE**, our database writes those changes to a journal (The WAL or the REDO Log). The WAL and Redo Log contain a sequential record of all the changes (operations) we've made - think of it as a chronological log of "what happened" that can be replayed to reconstruct the database state. They also have size constraints as well. In PostgreSQL, individual WAL segment files are 16 MB by default, but the `max_wal_size` parameter (which controls when checkpoints trigger) defaults to 1GB and can be set much higher. In MySQL, the default `innodb_log_file_size` is typically 48MB per file (with 2 files = 96MB total), and this can be tuned from a minimum of about 4MB per file to a maximum of 256GB per file (512GB total with 2 files).

The size configuration affects checkpoint frequency and recovery time. Smaller WAL sizes mean checkpoints happen more frequently, which can reduce recovery time after a crash but may increase write overhead. Larger sizes allow more WAL to accumulate before checkpointing, reducing checkpoint overhead but potentially increasing recovery time. The relationship between size and flush frequency is more nuanced and we won't be covering that in this writeup.

In the case of a crash, the DBMS performs crash recovery: it replays all the changes recorded in the WAL files (both committed and uncommitted transactions are in the WAL). The system then uses transaction commit records to determine which transactions were successfully committed. For uncommitted transactions, it uses undo logs (in MySQL) or ignores uncommitted tuple versions (in PostgreSQL) to roll them back, restoring the database to a consistent state with only committed changes. For some databases with a lot of data, this process can take a lot of time, and rightfully so, cause if anything goes wrong here, the whole thing could get corrupt.

## The Undo Log

In MySQL we have a special log called the Undo Log, it's a nice lazy name tbh, and I am glad it's not something titled the **MVCC** in PostgreSQL. In terms of functionality they are similar but the **MVCC** in PostgreSQL does a whole lot more.

The Undo Log in MySQL stores the previous state of modified data, allowing us to rollback those changes if necessary. Yes, our special Rollback in MySQL for instance only works due to this guy cause it keeps track of those states. However, the undo log serves more than just rollback - it's also used for MVCC (Multi-Version Concurrency Control), allowing other transactions to read consistent snapshots of data even while modifications are in progress. The undo log is stored in undo tablespaces and is itself protected by the redo log.

When you issue a rollback, MySQL uses the undo log to restore the previous state, and this restoration operation is logged in the redo log. Once a transaction is committed and no longer needed for MVCC or consistent reads, the undo log entries can be purged. However, long-running transactions may keep undo log entries active even after other transactions have committed.

MySQL is pretty much straightforward so what in God's name is the **MVCC**?

## The MVCC (Multi Version Concurrency Control)

In PostgreSQL, we actually don't have an explicit **Undo Log** like the case of MySQL, what we have is the MVCC, and it does a whole lot more than just helping us rollback to a previous state. The MVCC in PostgreSQL lets multiple transactions work in the database without interfering with each other, it does by keeping multiple versions of rows so that each transaction sees a consistent snapshot of the data even while others are making changes. 

In PostgreSQL, when Emeka for instance updates or deletes a row, PostgreSQL creates new **tuple versions** (physical rows, not virtual) with transaction IDs. 

- For updates, it creates a new tuple version with the updated data, and the old version remains in the table. 
- For deletes, it marks the tuple as deleted. 

When Emeka decides to rollback, PostgreSQL simply doesn't commit the transaction. The new tuple versions remain in the table but are invisible to other transactions because they're associated with an uncommitted transaction ID. The old versions remain visible to other transactions. A background process called VACUUM comes to clean up tuple versions that are no longer needed by any transaction (More on this in a later article). So this way, the old versions of the row remain intact and the database can just pretend it didn't make any changes.

## Conclusion

Our OS writes in pages (typically 4KB), and depending on our drive type, these pages are stored differently. For SSDs, OS pages (4KB) are stored inside erase blocks (typically 256KB to 4MB in size). In the case of HDDs, those writes happen in sectors (typically 512 bytes or 4KB).

When we do a write, our DBMS writes pages that need to fit within these physical storage units. The interesting thing about this is that there is always a price to pay. HDDs are slower because they have mechanical parts. They need to physically move the read/write head to the correct track and wait for the platter to rotate to the right sector before reading or writing data. SSDs have limited write endurance (a "shelf life" measured in program/erase cycles), and updates are particularly expensive because SSDs can't overwrite data in place. They must perform a read-modify-write cycle: read the entire erase block, modify it, erase the block, and write it back. This is why append-only workloads are ideal for SSDs.

Each SSD cell has a maximum number of program/erase cycles (typically thousands to hundreds of thousands depending on the NAND type). When individual blocks wear out, the SSD controller marks them as bad and uses spare blocks (SSDs are manufactured with extra capacity for this purpose). The controller also uses wear leveling algorithms to distribute writes evenly across all blocks, preventing premature wear of specific blocks. The drive continues to function until it runs out of spare blocks.

So in the case of PostgreSQL, it follows the write-ahead logging principle: it writes changes to the WAL first, then applies those changes to the shared buffers (in-memory pages). The WAL is flushed to disk using Fsync (based on `synchronous_commit` settings), while the pages in shared buffers are written to disk later during checkpoint or eviction.

So the process for PostgreSQL is 

```
 $ WAL -> DISK (Immediately with Fsync)
 $ Pages -> DISK (Much later during checkpoint or eviction)
```

In MySQL, the process is a bit different cause it manages the Redo log, and the undo log.

So the process is:

```
 $ Writes changes to the buffer pool & the redo log buffer simultaneously
 $ Flushes the redo log to disk (based on innodb_flush_log_at_trx_commit setting)
 $ Writes undo log entries to undo tablespace (also protected by redo log)
 $ Changes in buffer pool are later flushed to disk during checkpoint or eviction
```

Note: The redo log must be flushed to disk before a transaction is considered durable. The order matters - redo log flush happens before the transaction commit is acknowledged to the client (depending on durability settings).

It's actually a nice process tbh, and the key thing for me is how we can adjust the values for these log files so that they either get flushed immediately or not, and this could give drastic performance gains. But as always, there is always a price to pay with our SSD and HDDs. See you in the next writup. Till then, Sayonara!