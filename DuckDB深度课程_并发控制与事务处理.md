# DuckDB 深度课程：并发控制与事务处理

> 本课程深入讲解DuckDB的并发控制机制、事务处理、MVCC实现和性能优化。

---

## 课程概览

### 学习目标

- 理解事务的ACID特性
- 掌握MVCC（多版本并发控制）实现
- 学习锁机制和冲突检测
- 了解WAL和Checkpoint机制
- 掌握并发性能优化技巧

### 前置知识

- 熟悉C++基础
- 了解多线程编程
- 理解基本的数据结构（哈希表、链表等）

---

## 第一部分：事务基础

### 1.1 ACID特性

```
A - Atomicity (原子性)
    ↓
    事务中的所有操作要么全部成功，要么全部失败

C - Consistency (一致性)
    ↓
    事务执行前后，数据库从一种一致状态转换到另一种一致状态

I - Isolation (隔离性)
    ↓
    并发事务之间互不干扰

D - Durability (持久性)
    ↓
    一旦事务提交，其结果永久保存
```

### 1.2 事务隔离级别

```sql
-- READ UNCOMMITTED (读未提交)
-- 可以读到未提交的数据，可能导致脏读
SET TRANSACTION_ISOLATION LEVEL READ UNCOMMITTED;

-- READ COMMITTED (读已提交)
-- 只能读到已提交的数据，避免脏读
SET TRANSACTION_ISOLATION LEVEL READ COMMITTED;

-- REPEATABLE READ (可重复读)
-- 同一事务中多次读取同一数据结果一致
SET TRANSACTION_ISOLATION LEVEL REPEATABLE READ;

-- SERIALIZABLE (串行化)
-- 最高隔离级别，完全隔离
SET TRANSACTION_ISOLATION LEVEL SERIALIZABLE;
```

### 1.3 并发问题

```cpp
// 并发问题示例

// 脏读 (Dirty Read)
// 事务A读到了事务B未提交的修改
Transaction A: SELECT balance FROM accounts WHERE id = 1;  // 余额100
Transaction B: UPDATE accounts SET balance = 200 WHERE id = 1;  // 未提交
Transaction A: SELECT balance FROM accounts WHERE id = 1;  // 余额200 (脏读)
Transaction B: ROLLBACK;

// 不可重复读 (Non-repeatable Read)
// 同一事务中两次读取同一数据结果不同
Transaction A: SELECT balance FROM accounts WHERE id = 1;  // 余额100
Transaction B: UPDATE accounts SET balance = 200 WHERE id = 1; COMMIT;
Transaction A: SELECT balance FROM accounts WHERE id = 1;  // 余额200

// 幻读 (Phantom Read)
// 同一事务中两次查询返回的行数不同
Transaction A: SELECT * FROM accounts WHERE balance > 100;  // 返回5行
Transaction B: INSERT INTO accounts VALUES (6, 'Alice', 150); COMMIT;
Transaction A: SELECT * FROM accounts WHERE balance > 100;  // 返回6行
```

---

## 第二部分：MVCC实现

### 2.1 MVCC基本概念

MVCC（Multi-Version Concurrency Control）通过维护数据的多个版本来实现并发控制。

```
传统锁机制 vs MVCC：

锁机制：
┌─────────┐     Lock     ┌─────────┐
│ Reader  │ ───────────→ │  Data   │
└─────────┘              └─────────┘
                          ↑
                          │ Lock
                      ┌───┴────┐
                      │ Writer │
                      └────────┘
问题：读写互斥，并发度低

MVCC：
┌─────────┐   Read    ┌──────────────────┐
│ Reader  │ ─────────→│ Version 1 (old) │  读取旧版本
└─────────┘           └──────────────────┘
                              ↓
                       ┌──────────────────┐
                       │ Version 2 (new) │  Writer写入新版本
                       └──────────────────┘
                              ↑
                       ┌──────┴──────┐
                       │   Writer    │
                       └─────────────┘
优势：读写不冲突，高并发
```

### 2.2 DuckDB的MVCC架构

```cpp
// src/transaction/transaction.hpp

namespace duckdb {

// 事务状态
enum class TransactionStatus : uint8_t {
    TRANSACTION_NOT_STARTED,
    TRANSACTION_ACTIVE,
    TRANSACTION_COMMITTED,
    TRANSACTION_ABORTED
};

// 事务类
class Transaction {
public:
    Transaction(TransactionManager &manager, ClientContext &context);
    virtual ~Transaction();

    // 事务ID
    transaction_t transaction_id;
    transaction_t start_time;       // 开始时间戳
    transaction_t commit_id;        // 提交时间戳

    // 状态
    TransactionStatus status;

    // Undo buffer（用于回滚）
    UndoBuffer undo_buffer;

    // 当前语句信息
    unique_ptr<TransactionContext> current_query;

    // Catalog信息
    unordered_set<catalog_t> catalogs_modified;
};

// 版本信息
struct UndoBuffer {
    vector<UndoEntry> entries;

    // 添加undo entry
    void PushEntry(UndoEntry entry);

    // 回滚
    void Rollback();
};

} // namespace duckdb
```

### 2.3 版本链实现

```cpp
// 版本链节点
struct VersionNode {
    UndoFlags type;                    // 操作类型
    transaction_t transaction_id;      // 创建此版本的事务
    idx_t column_idx;                  // 列索引
    Value value;                       // 值
    unique_ptr<VersionNode> next;      // 指向下一个版本

    VersionNode(UndoFlags type_p, transaction_t txn_id,
                idx_t col_idx_p, Value val_p)
        : type(type_p), transaction_id(txn_id),
          column_idx(col_idx_p), value(std::move(val_p)) {
    }
};

// 版本信息
struct VersionInfo {
    unique_ptr<VersionNode> root;      // 版本链根节点
    idx_t tuple_version;               // 元组版本号

    // 插入新版本
    void InsertVersion(transaction_t txn_id,
                        idx_t column_idx,
                        Value value) {
        auto new_node = make_unique<VersionNode>(
            UndoFlags::UPDATE_TUPLE,
            txn_id,
            column_idx,
            std::move(value)
        );

        // 添加到链头
        new_node->next = std::move(root);
        root = std::move(new_node);
        tuple_version++;
    }

    // 获取可见版本
    bool GetVersion(transaction_t txn_id,
                     transaction_t start_time,
                     idx_t column_idx,
                     Value &result) {
        auto node = root.get();

        while (node) {
            if (node->transaction_id < start_time) {
                // 此版本在事务开始前已提交，可见
                if (node->column_idx == column_idx) {
                    result = node->value;
                    return true;
                }
            }
            node = node->next.get();
        }

        return false;
    }
};
```

### 2.4 可见性判断

```cpp
// 可见性判断逻辑
class MVCC {
public:
    // 判断版本是否对事务可见
    bool IsVersionVisible(transaction_t version_txn_id,
                         transaction_t txn_id,
                         transaction_t start_time) {
        // 规则1: 版本由当前事务创建，可见
        if (version_txn_id == txn_id) {
            return true;
        }

        // 规则2: 版本在事务开始前已提交，可见
        if (version_txn_id < start_time) {
            // 需要检查事务是否真的提交了
            return IsCommitted(version_txn_id);
        }

        // 规则3: 版本在事务开始后才创建，不可见
        return false;
    }

    // 检查事务是否已提交
    bool IsCommitted(transaction_t txn_id) {
        // 查询事务状态表
        auto it = transaction_table.find(txn_id);
        if (it != transaction_table.end()) {
            return it->second.status == TransactionStatus::TRANSACTION_COMMITTED;
        }
        return false;
    }

private:
    struct TransactionInfo {
        TransactionStatus status;
        transaction_t commit_time;
    };

    unordered_map<transaction_t, TransactionInfo> transaction_table;
};
```

---

## 第三部分：并发控制机制

### 3.1 锁机制

```cpp
// src/transaction/lock_manager.hpp

// 锁类型
enum class LockType {
    LOCK_SHARED,      // 共享锁（读锁）
    LOCK_EXCLUSIVE    // 排他锁（写锁）
};

// 锁请求
struct LockRequest {
    transaction_t transaction_id;
    LockType type;
    unique_ptr<LockRequest> next;     // 等待队列
};

// 锁管理器
class LockManager {
private:
    // 对象 -> 锁列表的映射
    unordered_map<void*, unique_ptr<LockRequest>> locks;

    // 死锁检测
    bool DetectDeadlock(transaction_t txn_id);

public:
    // 获取锁
    bool AcquireLock(void *object, transaction_t txn_id, LockType type) {
        // 检查是否已有锁
        auto it = locks.find(object);

        if (it == locks.end()) {
            // 没有锁，直接获取
            locks[object] = make_unique<LockRequest>(txn_id, type);
            return true;
        }

        auto *current_lock = it->second.get();

        // 检查锁兼容性
        if (current_lock->transaction_id == txn_id) {
            // 同一事务，锁升级
            if (current_lock->type == LockType::LOCK_SHARED &&
                type == LockType::LOCK_EXCLUSIVE) {
                current_lock->type = LockType::LOCK_EXCLUSIVE;
                return true;
            }
            return true;  // 已持有兼容的锁
        }

        // 检查锁兼容性矩阵
        if (!IsLockCompatible(current_lock->type, type)) {
            // 锁冲突，加入等待队列
            // ... 检查死锁
            return false;
        }

        // 兼容，可以获取
        return true;
    }

    // 释放锁
    void ReleaseLock(void *object, transaction_t txn_id) {
        auto it = locks.find(object);
        if (it != locks.end() &&
            it->second->transaction_id == txn_id) {
            locks.erase(it);
        }
    }

private:
    // 锁兼容性检查
    bool IsLockCompatible(LockType held, LockType requested) {
        if (held == LockType::LOCK_SHARED &&
            requested == LockType::LOCK_SHARED) {
            return true;  // S-S兼容
        }
        return false;    // 其他情况不兼容
    }
};
```

### 3.2 写偏（Write Skew）问题

```cpp
// 写偏问题示例
// 银行场景：保证两个账户余额之和 >= 1000

// 初始状态：Alice = 500, Bob = 500

Transaction A:
    // 检查 Alice + Bob >= 1000
    SELECT sum(balance) FROM accounts
    WHERE name IN ('Alice', 'Bob');  // 返回 1000

Transaction B:
    // 同时检查
    SELECT sum(balance) FROM accounts
    WHERE name IN ('Alice', 'Bob');  // 返回 1000

Transaction A:
    // 转账 200
    UPDATE accounts SET balance = balance - 200 WHERE name = 'Alice';
    COMMIT;  // Alice = 300

Transaction B:
    // 转账 200
    UPDATE accounts SET balance = balance - 200 WHERE name = 'Bob';
    COMMIT;  // Bob = 300

// 最终状态：Alice = 300, Bob = 300
// 违反约束：300 + 300 = 600 < 1000

// 解决方案：使用SERIALIZABLE隔离级别或谓词锁
```

### 3.3 乐观并发控制（OCC）

```cpp
// 乐观并发控制实现

class OptimisticConcurrencyControl {
public:
    struct Version {
        Value value;
        transaction_t creator_txn;
        unique_ptr<Version> next;
    };

    // 读取阶段
    Value Read(DataChunk &chunk, idx_t col_idx, idx_t row_idx,
               transaction_t txn_id) {
        auto &version = GetVersion(chunk, col_idx, row_idx);
        // 记录读取的版本
        read_set[txn_id].insert(&version);
        return version.value;
    }

    // 写入阶段（暂存）
    void Write(DataChunk &chunk, idx_t col_idx, idx_t row_idx,
               Value value, transaction_t txn_id) {
        // 暂存写操作
        write_set[txn_id].push_back({&chunk, col_idx, row_idx, value});
    }

    // 验证阶段（提交时）
    bool Validate(transaction_t txn_id) {
        // 检查读集中的版本是否被修改
        for (auto *version : read_set[txn_id]) {
            if (version->creator_txn > txn_id) {
                // 有新版本，冲突！
                return false;
            }
        }
        return true;
    }

    // 写入阶段
    void Commit(transaction_t txn_id) {
        if (Validate(txn_id)) {
            // 应用写操作
            for (auto &write : write_set[txn_id]) {
                WriteInternal(write.chunk, write.col_idx,
                            write.row_idx, write.value, txn_id);
            }
            // 清理
            read_set.erase(txn_id);
            write_set.erase(txn_id);
        } else {
            // 回滚
            Abort(txn_id);
        }
    }

private:
    unordered_map<transaction_t, unordered_set<Version*>> read_set;
    unordered_map<transaction_t, vector<WriteOp>> write_set;
};
```

---

## 第四部分：WAL与恢复

### 4.1 WAL（Write-Ahead Log）概述

WAL是确保事务持久性和原子性的关键机制。

```
WAL原则：
1. 日志必须先于数据写入磁盘
2. 提交前日志必须刷盘
3. 崩溃恢复时重放日志

写入流程：
┌─────────┐
│ 事务    │
└────┬────┘
     │
     ↓
┌─────────┐     写入     ┌──────────┐
│ 内存    │ ──────────→ │ WAL日志  │ ──→ 刷盘（fsync）
│ 数据    │             │ (顺序IO) │
└─────────┘             └──────────┘
     │                        │
     │ 提交时                │ 重放日志
     ↓                       ↓
┌─────────┐              ┌──────┐
│ 数据文件 │ ←─────────── │ 恢复 │
│ (随机IO)│              └──────┘
└─────────┘
```

### 4.2 WAL实现

```cpp
// src/storage/write_ahead_log.hpp

class WriteAheadLog {
public:
    // 日志记录类型
    enum class WALType : uint8_t {
        WAL_BEGIN_TXN,
        WAL_COMMIT_TXN,
        WAL_ROLLBACK_TXN,
        WAL_INSERT,
        WAL_UPDATE,
        WAL_DELETE,
        WAL_CHECKPOINT
    };

    // 日志记录头
    struct WALRecordHeader {
        WALType type;
        transaction_t txn_id;
        uint32_t record_size;
        uint64_t timestamp;
    };

    // 写入事务开始
    void WriteBeginTransaction(transaction_t txn_id) {
        WALRecordHeader header;
        header.type = WALType::WAL_BEGIN_TXN;
        header.txn_id = txn_id;
        header.record_size = sizeof(WALRecordHeader);
        header.timestamp = GetCurrentTimestamp();

        WriteLog(&header, sizeof(header));
        Flush();  // 确保日志落盘
    }

    // 写入更新操作
    void WriteUpdate(transaction_t txn_id,
                     row_t row_id,
                     idx_t column_idx,
                     Value old_value,
                     Value new_value) {
        // 序列化记录
        auto record = SerializeUpdate(txn_id, row_id,
                                     column_idx,
                                     old_value,
                                     new_value);

        // 写入日志
        WriteLog(record.data(), record.size());
    }

    // 写入提交
    void WriteCommit(transaction_t txn_id) {
        WALRecordHeader header;
        header.type = WALType::WAL_COMMIT_TXN;
        header.txn_id = txn_id;
        header.record_size = sizeof(WALRecordHeader);
        header.timestamp = GetCurrentTimestamp();

        WriteLog(&header, sizeof(header));
        Flush();  // 提交必须刷盘
    }

    // 检查点
    void WriteCheckpoint(CheckpointData &data) {
        // 记录检查点位置
        checkpoint_id = next_log_id;

        WALRecordHeader header;
        header.type = WALType::WAL_CHECKPOINT;
        header.txn_id = 0;  // 全局操作
        header.record_size = sizeof(WALRecordHeader) + data.size;

        WriteLog(&header, sizeof(header));
        WriteLog(data.data, data.size);
        Flush();
    }

private:
    void WriteLog(const void *data, size_t size) {
        log_file.write((const char*)data, size);
        current_log_id++;
        log_size += size;
    }

    void Flush() {
        log_file.flush();
#ifdef __linux__
        // 确保数据写入磁盘
        fsync(fileno(log_file));
#endif
    }

    ofstream log_file;
    uint64_t current_log_id = 0;
    uint64_t log_size = 0;
    uint64_t checkpoint_id = 0;
};
```

### 4.3 崩溃恢复

```cpp
// src/storage/recovery_manager.hpp

class RecoveryManager {
public:
    // 从WAL恢复
    void RecoverFromWAL(const string &wal_path,
                       const string &db_path) {
        // 1. 打开WAL文件
        ifstream wal_file(wal_path, ios::binary);

        // 2. 扫描WAL
        vector<WALRecord> records;
        while (!wal_file.eof()) {
            auto record = ReadWALRecord(wal_file);
            if (record) {
                records.push_back(*record);
            }
        }

        // 3. 找到检查点位置
        uint64_t checkpoint_id = FindCheckpoint(records);

        // 4. 重做已提交事务
        RedoTransactions(records, checkpoint_id);

        // 5. 撤销未提交事务
        UndoTransactions(records, checkpoint_id);

        // 6. 清理WAL
        CleanWAL(wal_path);
    }

private:
    void RedoTransactions(const vector<WALRecord> &records,
                         uint64_t checkpoint_id) {
        for (const auto &record : records) {
            // 只重做检查点之后的事务
            if (record.log_id <= checkpoint_id) {
                continue;
            }

            if (record.type == WALType::WAL_COMMIT_TXN) {
                // 找到已提交事务的所有操作
                auto txn_records = GetTransactionRecords(records, record.txn_id);

                // 重做操作
                for (const auto &txn_rec : txn_records) {
                    RedoOperation(txn_rec);
                }
            }
        }
    }

    void UndoTransactions(const vector<WALRecord> &records,
                         uint64_t checkpoint_id) {
        // 找出未提交的事务
        unordered_set<transaction_t> committed_txns;

        for (const auto &record : records) {
            if (record.log_id > checkpoint_id) {
                if (record.type == WALType::WAL_COMMIT_TXN) {
                    committed_txns.insert(record.txn_id);
                }
            }
        }

        // 撤销未提交事务
        for (auto it = records.rbegin(); it != records.rend(); ++it) {
            const auto &record = *it;

            if (record.log_id <= checkpoint_id) {
                break;
            }

            if (record.type == WALType::WAL_BEGIN_TXN) {
                if (committed_txns.find(record.txn_id) ==
                    committed_txns.end()) {
                    // 未提交事务，撤销其操作
                    auto txn_records = GetTransactionRecords(records, record.txn_id);
                    for (auto txn_rec = txn_records.rbegin();
                         txn_rec != txn_records.rend(); ++txn_rec) {
                        UndoOperation(*txn_rec);
                    }
                }
            }
        }
    }

    void RedoOperation(const WALRecord &record) {
        switch (record.type) {
        case WALType::WAL_INSERT:
            InsertRow(record.row_id, record.values);
            break;
        case WALType::WAL_UPDATE:
            UpdateRow(record.row_id, record.column_idx,
                     record.new_value);
            break;
        case WALType::WAL_DELETE:
            DeleteRow(record.row_id);
            break;
        default:
            break;
        }
    }

    void UndoOperation(const WALRecord &record) {
        switch (record.type) {
        case WALType::WAL_UPDATE:
            // 恢复旧值
            UpdateRow(record.row_id, record.column_idx,
                     record.old_value);
            break;
        case WALType::WAL_INSERT:
            // 删除插入的行
            DeleteRow(record.row_id);
            break;
        case WALType::WAL_DELETE:
            // 恢复删除的行
            InsertRow(record.row_id, record.values);
            break;
        default:
            break;
        }
    }
};
```

### 4.4 Checkpoint机制

```cpp
// src/storage/checkpoint.hpp

class CheckpointWriter {
public:
    // 创建检查点
    void CreateCheckpoint(const string &db_path) {
        // 1. 停止所有写操作
        PauseWAL();

        // 2. 将内存中的脏页写入数据文件
        FlushDirtyPages();

        // 3. 写入检查点记录
        WriteCheckpointMetadata();

        // 4. 截断WAL
        TruncateWAL();

        // 5. 恢复写操作
        ResumeWAL();
    }

    // 增量检查点（减少停顿时间）
    void CreateIncrementalCheckpoint(const string &db_path) {
        // 只写入修改过的页
        auto dirty_pages = GetDirtyPages();

        // 分批写入，避免长时间停顿
        for (size_t i = 0; i < dirty_pages.size(); i += BATCH_SIZE) {
            auto end_idx = min(i + BATCH_SIZE, dirty_pages.size());

            for (size_t j = i; j < end_idx; j++) {
                WritePage(dirty_pages[j]);
            }

            // 允许其他操作
            Yield();
        }

        // 写入检查点记录
        WriteCheckpointMetadata();
    }

private:
    void WriteCheckpointMetadata() {
        CheckpointMetadata metadata;
        metadata.checkpoint_id = next_checkpoint_id++;
        metadata.timestamp = GetCurrentTimestamp();
        metadata.lsn = current_lsn;

        // 写入元数据
        checkpoint_file.write((const char*)&metadata,
                             sizeof(metadata));
        checkpoint_file.flush();
    }

    void TruncateWAL() {
        // 备份当前WAL
        string wal_backup = wal_path + ".bak";
        rename(wal_path.c_str(), wal_backup.c_str());

        // 创建新的WAL
        ofstream new_wal(wal_path, ios::binary);

        // 清空旧数据
        wal_size = 0;
        current_lsn = 0;
    }
};
```

---

## 第五部分：性能优化

### 5.1 并发性能优化

```cpp
// 优化1：延迟锁获取
class OptimisticScan {
public:
    // 扫描时不加锁
    DataChunk ScanWithoutLock(Table &table) {
        DataChunk chunk;
        chunk.Initialize(table.types);

        // 直接读取数据
        for (idx_t i = 0; i < STANDARD_VECTOR_SIZE; i++) {
            chunk.SetValue(i, table.GetValue(current_row + i));
        }

        current_row += STANDARD_VECTOR_SIZE;
        return chunk;
    }

    // 使用时验证版本
    bool ValidateVersion(DataChunk &chunk, transaction_t txn_id) {
        for (idx_t i = 0; i < chunk.size(); i++) {
            auto version = chunk.GetVersion(i);
            if (version && version->creator_txn > txn_id) {
                return false;  // 版本已变化
            }
        }
        return true;
    }
};

// 优化2：锁粒度优化
class GranularLocking {
private:
    // 行级锁
    LockManager row_locks;

    // 页级锁
    LockManager page_locks;

    // 表级锁
    LockManager table_locks;

public:
    // 根据操作范围选择锁粒度
    void AcquireLock(Table &table,
                    idx_t start_row,
                    idx_t end_row,
                    transaction_t txn_id) {
        idx_t row_count = end_row - start_row;

        if (row_count < 100) {
            // 少量行，使用行级锁
            for (idx_t i = start_row; i < end_row; i++) {
                row_locks.AcquireLock(&table.GetRow(i), txn_id,
                                      LockType::LOCK_EXCLUSIVE);
            }
        } else if (row_count < 10000) {
            // 中等数量，使用页级锁
            auto start_page = start_row / ROWS_PER_PAGE;
            auto end_page = end_row / ROWS_PER_PAGE;

            for (idx_t page = start_page; page <= end_page; page++) {
                page_locks.AcquireLock(&table.GetPage(page), txn_id,
                                       LockType::LOCK_EXCLUSIVE);
            }
        } else {
            // 大量行，使用表级锁
            table_locks.AcquireLock(&table, txn_id,
                                    LockType::LOCK_EXCLUSIVE);
        }
    }
};

// 优化3：读写分离
class ReadWriteSeparation {
private:
    // 主库（处理写操作）
    unique_ptr<Database> primary;

    // 多个只读副本（处理读操作）
    vector<unique_ptr<Database>> replicas;

public:
    // 读操作路由到副本
    Result ExecuteRead(const string &query) {
        // 选择负载最少的副本
        auto replica = SelectReplica();
        return replica->Execute(query);
    }

    // 写操作路由到主库
    Result ExecuteWrite(const string &query) {
        return primary->Execute(query);
    }

private:
    Database* SelectReplica() {
        Database* min_load = replicas[0].get();
        idx_t min_connections = replicas[0]->GetConnectionCount();

        for (size_t i = 1; i < replicas.size(); i++) {
            auto conn_count = replicas[i]->GetConnectionCount();
            if (conn_count < min_connections) {
                min_load = replicas[i].get();
                min_connections = conn_count;
            }
        }

        return min_load;
    }
};
```

### 5.2 事务吞吐量优化

```cpp
// 批量提交
class BatchCommit {
private:
    vector<Transaction> pending_txns;
    size_t batch_size = 100;

public:
    // 积累事务到批量提交
    void AddTransaction(Transaction &&txn) {
        pending_txns.push_back(std::move(txn));

        if (pending_txns.size() >= batch_size) {
            CommitBatch();
        }
    }

    void CommitBatch() {
        // 1. 写入所有WAL
        for (const auto &txn : pending_txns) {
            WriteWAL(txn);
        }

        // 2. 一次性刷盘
        FlushWAL();

        // 3. 应用更改
        for (const auto &txn : pending_txns) {
            ApplyTransaction(txn);
        }

        // 4. 清空批量
        pending_txns.clear();
    }
};

// 组提交（Group Commit）
class GroupCommit {
private:
    vector<Transaction> commit_group;
    mutex group_mutex;
    condition_variable cv;

public:
    void Commit(Transaction &&txn) {
        unique_lock<mutex> lock(group_mutex);

        // 加入提交组
        commit_group.push_back(std::move(txn));

        // 等待组内其他事务（最多1ms）
        cv.wait_for(lock, chrono::milliseconds(1));

        // 如果是第一个事务，负责提交整个组
        if (commit_group.size() == 1) {
            cv.notify_all();  // 通知其他等待的事务
        }

        if (commit_group[0].transaction_id == txn.transaction_id) {
            // 执行组提交
            CommitGroup();
        }
    }

private:
    void CommitGroup() {
        // 写入所有WAL
        for (const auto &txn : commit_group) {
            WriteWAL(txn);
        }

        // 一次刷盘
        FlushWAL();

        // 标记所有事务为已提交
        for (const auto &txn : commit_group) {
            MarkCommitted(txn.transaction_id);
        }

        commit_group.clear();
    }
};
```

---

## 第六部分：实践项目

### 项目1：实现简单MVCC

```cpp
// simple_mvcc.hpp

#include <unordered_map>
#include <memory>
#include <mutex>
#include <vector>

class SimpleMVCC {
private:
    // 版本信息
    struct Version {
        std::string value;
        uint64_t creator_txn;
        std::shared_ptr<Version> next;

        Version(std::string val, uint64_t txn)
            : value(std::move(val)), creator_txn(txn) {
        }
    };

    // 元组
    struct Tuple {
        std::shared_ptr<Version> version;
        std::mutex mutex;  // 保护版本链
    };

    // 事务状态
    enum class TxnStatus {
        ACTIVE,
        COMMITTED,
        ABORTED
    };

    struct TransactionInfo {
        TxnStatus status;
        uint64_t start_time;
    };

    // 存储
    std::unordered_map<std::string, Tuple> data;

    // 事务管理
    std::unordered_map<uint64_t, TransactionInfo> transactions;
    uint64_t next_txn_id = 1;
    std::mutex txn_mutex;

public:
    // 开始事务
    uint64_t BeginTransaction() {
        std::lock_guard<std::mutex> lock(txn_mutex);

        auto txn_id = next_txn_id++;
        transactions[txn_id] = {
            TxnStatus::ACTIVE,
            GetCurrentTimestamp()
        };

        return txn_id;
    }

    // 读取数据
    bool Read(uint64_t txn_id,
              const std::string &key,
              std::string &value) {
        auto it = data.find(key);
        if (it == data.end()) {
            return false;  // 键不存在
        }

        auto &tuple = it->second;
        auto version = tuple.version.get();

        // 遍历版本链
        while (version) {
            if (IsVisible(version->creator_txn, txn_id)) {
                value = version->value;
                return true;
            }
            version = version->next.get();
        }

        return false;  // 无可见版本
    }

    // 写入数据
    void Write(uint64_t txn_id,
               const std::string &key,
               const std::string &value) {
        std::lock_guard<std::mutex> lock(data[key].mutex);

        auto new_version = std::make_shared<Version>(value, txn_id);
        new_version->next = data[key].version;
        data[key].version = new_version;
    }

    // 提交事务
    bool Commit(uint64_t txn_id) {
        std::lock_guard<std::mutex> lock(txn_mutex);

        auto it = transactions.find(txn_id);
        if (it == transactions.end() ||
            it->second.status != TxnStatus::ACTIVE) {
            return false;  // 事务不存在或已结束
        }

        it->second.status = TxnStatus::COMMITTED;
        return true;
    }

    // 回滚事务
    void Rollback(uint64_t txn_id) {
        std::lock_guard<std::mutex> lock(txn_mutex);

        auto it = transactions.find(txn_id);
        if (it != transactions.end()) {
            it->second.status = TxnStatus::ABORTED;

            // 清理该事务创建的版本
            CleanupVersions(txn_id);
        }
    }

private:
    // 判断版本是否可见
    bool IsVisible(uint64_t version_txn, uint64_t reader_txn) {
        if (version_txn == reader_txn) {
            return true;  // 自己创建的版本
        }

        auto txn_it = transactions.find(version_txn);
        if (txn_it == transactions.end()) {
            return false;  // 事务不存在
        }

        // 只有已提交的版本才可见
        return txn_it->second.status == TxnStatus::COMMITTED;
    }

    void CleanupVersions(uint64_t txn_id) {
        // 遍历所有元组，清理指定事务创建的版本
        for (auto &entry : data) {
            auto &tuple = entry.second;
            std::lock_guard<std::mutex> lock(tuple.mutex);

            std::shared_ptr<Version> prev;
            auto current = tuple.version;

            while (current) {
                if (current->creator_txn == txn_id) {
                    // 找到目标版本，移除
                    if (prev) {
                        prev->next = current->next;
                    } else {
                        tuple.version = current->next;
                    }
                    break;
                }
                prev = current;
                current = current->next;
            }
        }
    }

    uint64_t GetCurrentTimestamp() {
        return std::chrono::duration_cast<std::chrono::milliseconds>(
            std::chrono::system_clock::now().time_since_epoch()
        ).count();
    }
};
```

### 项目2：测试并发场景

```cpp
// test_concurrency.cpp

#include "simple_mvcc.hpp"
#include <thread>
#include <vector>
#include <cassert>

void TestDirtyRead() {
    SimpleMVCC mvcc;

    // 初始化数据
    auto txn1 = mvcc.BeginTransaction();
    mvcc.Write(txn1, "account", "100");
    mvcc.Commit(txn1);

    // 事务A读取
    auto txn_a = mvcc.BeginTransaction();
    std::string value;
    mvcc.Read(txn_a, "account", value);
    assert(value == "100");

    // 事务B修改（未提交）
    auto txn_b = mvcc.BeginTransaction();
    mvcc.Write(txn_b, "account", "200");

    // 事务A再次读取（不应该看到未提交的修改）
    mvcc.Read(txn_a, "account", value);
    assert(value == "100");  // 仍然是100

    mvcc.Rollback(txn_b);
    mvcc.Commit(txn_a);

    printf("TestDirtyRead: PASSED\n");
}

void TestRepeatableRead() {
    SimpleMVCC mvcc;

    // 初始化数据
    auto txn1 = mvcc.BeginTransaction();
    mvcc.Write(txn1, "x", "10");
    mvcc.Commit(txn1);

    // 事务A
    auto txn_a = mvcc.BeginTransaction();

    // 第一次读取
    std::string value;
    mvcc.Read(txn_a, "x", value);
    assert(value == "10");

    // 事务B修改并提交
    auto txn_b = mvcc.BeginTransaction();
    mvcc.Write(txn_b, "x", "20");
    mvcc.Commit(txn_b);

    // 事务A第二次读取（应该看到旧值）
    mvcc.Read(txn_a, "x", value);
    assert(value == "10");  // 可重复读

    mvcc.Commit(txn_a);

    printf("TestRepeatableRead: PASSED\n");
}

void TestConcurrency() {
    SimpleMVCC mvcc;

    // 初始化
    auto init_txn = mvcc.BeginTransaction();
    for (int i = 0; i < 100; i++) {
        mvcc.Write(init_txn, "key" + std::to_string(i),
                   "value" + std::to_string(i));
    }
    mvcc.Commit(init_txn);

    // 并发写
    std::vector<std::thread> threads;
    for (int t = 0; t < 10; t++) {
        threads.emplace_back([&mvcc, t]() {
            for (int i = 0; i < 10; i++) {
                auto txn = mvcc.BeginTransaction();

                // 读取
                std::string value;
                mvcc.Read(txn, "key" + std::to_string(i), value);

                // 修改
                mvcc.Write(txn, "key" + std::to_string(i),
                          "new_value_" + std::to_string(t) + "_" +
                          std::to_string(i));

                mvcc.Commit(txn);
            }
        });
    }

    for (auto &t : threads) {
        t.join();
    }

    printf("TestConcurrency: PASSED\n");
}

int main() {
    TestDirtyRead();
    TestRepeatableRead();
    TestConcurrency();

    printf("All tests passed!\n");
    return 0;
}
```

---

## 学习总结

### 核心概念

| 概念 | 说明 |
|------|------|
| ACID | 原子性、一致性、隔离性、持久性 |
| MVCC | 多版本并发控制，读写不冲突 |
| WAL | 预写日志，确保持久性 |
| Checkpoint | 检查点，缩短恢复时间 |
| 锁粒度 | 行锁、页锁、表锁 |
| 隔离级别 | READ UNCOMMITTED → SERIALIZABLE |

### 常见问题

| 问题 | 解决方案 |
|------|----------|
| 脏读 | MVCC + 版本可见性判断 |
| 不可重复读 | MVCC + 事务快照 |
| 幻读 | 谓词锁或SERIALIZABLE |
| 死锁 | 超时机制或死锁检测 |
| 写偏 | SERIALIZABLE或约束检查 |

### 推荐资源

**论文：**
- "Concurrency Control and Recovery in Database Systems"
- "A Critique of ANSI SQL Isolation Levels"
- "Implementing a Lock-Free Concurrency Control for Serializable Isolation"

**代码：**
- DuckDB: `src/transaction/`
- PostgreSQL: `src/backend/access/transam/`
- MySQL InnoDB: `storage/innodb/trx/`

---

**最后更新：2026-01-23**
