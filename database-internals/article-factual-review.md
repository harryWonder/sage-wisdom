# Factual Review: "Why They Never Crash"

## ✅ **What You Got Right**

1. **Fsync concept** (Lines 11-13): Correct that fsync forces OS to flush page cache to disk. The MySQL `innodb_flush_log_at_trx_commit = 2` setting description is accurate.

2. **WAL vs Redo Log equivalence** (Line 21): Correct that they serve the same fundamental purpose (write-ahead logging).

3. **Crash recovery** (Line 31): Accurate description of replaying committed transactions and rolling back uncommitted ones.

4. **Undo Log basics** (Line 37): Correct that it stores previous state for rollback, though the description is incomplete (see corrections below).

5. **MVCC purpose** (Line 45): Correct that MVCC allows concurrent transactions without interference.

6. **VACUUM process** (Line 47): Correct that VACUUM cleans up old tuple versions.

7. **SSD/HDD write characteristics** (Lines 51-53): Generally correct about erase blocks, sectors, and wear characteristics.

## ❌ **What Needs Correction**

### **Critical Issues:**

1. **Line 7 - Commit semantics**: The statement "data isn't fully committed or saved unless Fsync happens" is misleading. With `synchronous_commit = off` in PostgreSQL or `innodb_flush_log_at_trx_commit = 2` in MySQL, transactions can be considered committed before fsync, trading durability for performance.

2. **Line 25 - WAL size claims**: 
   - **PostgreSQL**: Default is NOT 16MB. The default `max_wal_size` is 1GB (since PostgreSQL 9.5+). Individual WAL segment files are 16MB by default, but `max_wal_size` controls when checkpoint triggers. The maximum is much higher than 1GB (practically unlimited, though 1GB is a common default).
   - **MySQL**: Default `innodb_log_file_size` is typically 48MB per file (with 2 files = 96MB total), not 100MB. Maximum is 512GB total (256GB per file × 2 files), minimum is typically around 4MB per file.

3. **Line 27 - Size tuning logic**: The explanation is backwards/confusing. Smaller WAL sizes don't necessarily mean "more flushes" - they mean checkpoints happen more frequently, which can actually reduce the amount of WAL that needs to be replayed. The relationship between size and flush frequency is more nuanced.

4. **Line 37 - Undo Log scope**: Incorrect to say it "keeps track of Uncommitted transactions" only. MySQL's undo log is also used for:
   - MVCC (read consistency for other transactions)
   - Consistent reads across transaction isolation levels
   - Long-running transactions may keep undo log entries even after commit

5. **Line 39 - Undo log to Redo log flow**: The description is misleading. When you rollback, the undo log is used to restore the previous state, and this restoration is logged in the redo log. But it's not accurate to say "changes from the Undo log trickle down to the WAL or Redo Log" - the undo log itself is part of the data that gets protected by the redo log.

6. **Line 45-47 - MVCC implementation**:
   - PostgreSQL doesn't create "virtual rows" - it creates new **tuple versions** (physical rows) with transaction IDs (xmin/xmax). These are real rows, not virtual.
   - On rollback, PostgreSQL doesn't "mark rows as invalid" - it simply doesn't commit the transaction, so the new tuple versions remain invisible to other transactions. The old versions remain visible. VACUUM later removes tuple versions that are no longer needed.

7. **Line 55 - PostgreSQL write process**: Not quite accurate. PostgreSQL:
   - Writes changes to WAL first (write-ahead logging principle)
   - Then applies changes to shared buffers (in-memory pages)
   - Flushes WAL to disk (with fsync based on `synchronous_commit`)
   - Pages are written to disk later during checkpoint
   - It's not truly "simultaneous" - WAL comes first, then buffer updates

8. **Line 69-72 - MySQL process**: The description is somewhat accurate but oversimplified:
   - Changes are written to redo log buffer and buffer pool
   - Redo log is flushed to disk (based on `innodb_flush_log_at_trx_commit`)
   - Undo log is written to undo tablespace (also protected by redo log)
   - Buffer pool pages are flushed later (checkpoint or eviction)
   - The order matters: redo log must be flushed before transaction is considered durable

### **Minor Issues:**

1. **Line 3**: Typo - "aks" → "asks", "loose" → "lose"
2. **Line 5**: Typo - "n today's" → "In today's"
3. **Line 51**: Typo - "earse blocks" → "erase blocks"
4. **Line 70**: Typo - "FLushes" → "Flushes"
5. **Line 57**: Typo - "proces" → "process"

## 📝 **Additional Clarifications Needed**

1. **PostgreSQL `synchronous_commit`**: You mention it's customizable but don't explain the different modes (`on`, `off`, `local`, `remote_write`, `remote_apply`).

2. **Checkpoint process**: Mentioned but not explained. Checkpoints are crucial - they determine when dirty pages are flushed and when WAL can be recycled.

3. **Undo log in MySQL**: Should clarify it's stored in undo tablespaces (separate from redo log) and is also protected by the redo log.

4. **Transaction durability levels**: The article implies all databases guarantee durability by default, but this depends on configuration. Many production systems use relaxed durability for performance.

## 🎯 **Overall Assessment**

The article covers the right concepts and demonstrates good understanding of database internals. The main issues are:
- Some factual inaccuracies about default sizes and limits
- Oversimplifications that miss important nuances
- A few technical misunderstandings about how MVCC and undo logs work
- Some confusing explanations that could mislead readers

The core message is sound: databases use WAL/redo logs with fsync to ensure durability, and MVCC/undo logs handle concurrency and rollbacks.

