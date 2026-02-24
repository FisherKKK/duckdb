# DuckDB 大师课程：从零实现数据库系统

> 本课程将从零开始，教你如何设计和实现一个完整的数据库管理系统。我们将参考DuckDB的架构，构建一个功能完备的Mini-DB。

---

## 课程概览

### 学习目标

完成本课程后，你将能够：
- 理解数据库系统的完整架构
- 实现一个可用的存储引擎
- 构建查询执行引擎
- 实现事务管理系统
- 掌握数据库核心算法

### 课程结构

```
第一阶段：存储引擎 (Week 1-2)
├── 页面管理器
├── 缓冲池管理器
├── B+树索引
└── 列式存储

第二阶段：执行引擎 (Week 3-4)
├── SQL解析器
├── 查询计划器
├── 执行算子
└── 向量化执行

第三阶段：事务系统 (Week 5-6)
├── 并发控制
├── WAL日志
├── MVCC实现
└── 恢复机制

第四阶段：优化器 (Week 7-8)
├── 逻辑优化
├── 物理优化
├── 统计信息
└── 代价估算

第五阶段：高级特性 (Week 9-10)
├── 压缩算法
├── 并行执行
├── 分布式扩展
└── 性能调优
```

---

## 第一阶段：存储引擎实现

### 模块1：页面管理器

页面是磁盘I/O的基本单位，通常为4KB-16KB。

#### 1.1 页面结构设计

```cpp
// mini_db/storage/page.hpp

#pragma once
#include "common/types.hpp"
#include <cstring>
#include <memory>

namespace mini_db {

// 页面大小：4KB
constexpr uint64_t PAGE_SIZE = 4096;

// 页面类型
enum class PageType : uint8_t {
    INVALID_PAGE = 0,   // 无效页
    DATA_PAGE = 1,      // 数据页
    INDEX_PAGE = 2,     // 索引页
    META_PAGE = 3,      // 元数据页
    FREE_PAGE = 4       // 空闲页
};

// 页面头部（48字节）
struct alignas(64) PageHeader {
    uint64_t page_id;        // 页面ID (8 bytes)
    PageType page_type;      // 页面类型 (1 byte)
    uint32_t checksum;       // 校验和 (4 bytes)
    uint64_t lsn;            // 日志序列号 (8 bytes)
    uint32_t prev_page;      // 前一页 (4 bytes)
    uint32_t next_page;      // 下一页 (4 bytes)
    uint16_t free_space;     // 空闲空间偏移 (2 bytes)
    uint16_t tuple_count;    // 元组数量 (2 bytes)
    uint8_t padding[15];     // 对齐填充 (15 bytes)

    PageHeader() : page_id(0), page_type(PageType::INVALID_PAGE),
                   checksum(0), lsn(0), prev_page(0), next_page(0),
                   free_space(0), tuple_count(0) {
        std::memset(padding, 0, sizeof(padding));
    }
};

// 数据页结构
class DataPage {
private:
    char data_[PAGE_SIZE];   // 原始数据

public:
    DataPage() {
        std::memset(data_, 0, PAGE_SIZE);
    }

    // 获取页面头部
    PageHeader* GetHeader() {
        return reinterpret_cast<PageHeader*>(data_);
    }

    const PageHeader* GetHeader() const {
        return reinterpret_cast<const PageHeader*>(data_);
    }

    // 获取数据区域（页面头部之后）
    char* GetData() {
        return data_ + sizeof(PageHeader);
    }

    const char* GetData() const {
        return data_ + sizeof(PageHeader);
    }

    // 获取可用数据大小
    uint32_t GetAvailableSize() const {
        return PAGE_SIZE - sizeof(PageHeader);
    }

    // 插入元组
    bool InsertTuple(const char* tuple_data, uint32_t tuple_size) {
        auto header = GetHeader();

        // 检查空间是否足够
        if (header->free_space + tuple_size > GetAvailableSize()) {
            return false;
        }

        // 计算插入位置
        char* insert_pos = GetData() + header->free_space;
        std::memcpy(insert_pos, tuple_data, tuple_size);

        // 更新头部
        header->free_space += tuple_size;
        header->tuple_count++;

        return true;
    }

    // 计算校验和（用于崩溃恢复）
    uint32_t ComputeChecksum() const {
        // 简单的CRC32或XOR校验
        uint32_t checksum = 0;
        const uint32_t* data = reinterpret_cast<const uint32_t*>(data_);

        for (size_t i = 0; i < PAGE_SIZE / sizeof(uint32_t); i++) {
            checksum ^= data[i];
        }

        return checksum;
    }

    // 验证页面完整性
    bool Validate() const {
        auto header = GetHeader();
        return header->checksum == ComputeChecksum();
    }
};

} // namespace mini_db
```

#### 1.2 页面管理器实现

```cpp
// mini_db/storage/page_manager.hpp

#pragma once
#include "page.hpp"
#include <string>
#include <vector>
#include <mutex>
#include <fstream>

namespace mini_db {

// 页面管理器：管理磁盘上的页面
class PageManager {
private:
    std::string db_file_path_;      // 数据库文件路径
    std::fstream db_file_;          // 文件流
    uint64_t num_pages_;            // 总页数
    uint64_t next_page_id_;         // 下一个可分配页面ID
    std::mutex mutex_;              // 线程安全

    // 空闲页面链表
    std::vector<uint64_t> free_pages_;

public:
    PageManager(const std::string& db_path)
        : db_file_path_(db_path), num_pages_(0), next_page_id_(0) {

        // 以读写模式打开文件
        db_file_.open(db_path,
                      std::ios::in | std::ios::out |
                      std::ios::binary);

        // 如果文件不存在，创建新文件
        if (!db_file_.is_open()) {
            db_file_.open(db_path,
                          std::ios::out | std::ios::binary);
            db_file_.close();
            db_file_.open(db_path,
                          std::ios::in | std::ios::out |
                          std::ios::binary);
        }

        // 读取文件信息
        LoadMetadata();
    }

    ~PageManager() {
        if (db_file_.is_open()) {
            db_file_.close();
        }
    }

    // 分配新页面
    uint64_t AllocatePage() {
        std::lock_guard<std::mutex> lock(mutex_);

        uint64_t page_id;

        // 优先使用空闲页面
        if (!free_pages_.empty()) {
            page_id = free_pages_.back();
            free_pages_.pop_back();
        } else {
            // 分配新页面
            page_id = next_page_id_++;
            num_pages_++;
        }

        return page_id;
    }

    // 释放页面
    void FreePage(uint64_t page_id) {
        std::lock_guard<std::mutex> lock(mutex_);
        free_pages_.push_back(page_id);
    }

    // 读取页面
    bool ReadPage(uint64_t page_id, DataPage& page) {
        std::lock_guard<std::mutex> lock(mutex_);

        if (page_id >= num_pages_) {
            return false;
        }

        // 定位到页面
        uint64_t offset = page_id * PAGE_SIZE;
        db_file_.seekg(offset);

        if (!db_file_.good()) {
            return false;
        }

        // 读取页面数据
        db_file_.read(page.GetData(), PAGE_SIZE);

        // 验证页面
        if (!page.Validate()) {
            return false;
        }

        return true;
    }

    // 写入页面
    bool WritePage(uint64_t page_id, const DataPage& page) {
        std::lock_guard<std::mutex> lock(mutex_);

        if (page_id >= num_pages_) {
            return false;
        }

        // 计算校验和
        const_cast<DataPage&>(page).GetHeader()->checksum =
            page.ComputeChecksum();

        // 定位到页面
        uint64_t offset = page_id * PAGE_SIZE;
        db_file_.seekp(offset);

        if (!db_file_.good()) {
            return false;
        }

        // 写入页面数据
        db_file_.write(page.GetData(), PAGE_SIZE);

        // 确保写入磁盘
        db_file_.flush();

        return true;
    }

    // 获取页面数量
    uint64_t GetNumPages() const {
        return num_pages_;
    }

private:
    // 加载元数据
    void LoadMetadata() {
        // 获取文件大小
        db_file_.seekg(0, std::ios::end);
        uint64_t file_size = db_file_.tellg();

        // 计算页数
        num_pages_ = file_size / PAGE_SIZE;
        next_page_id_ = num_pages_;
    }
};

} // namespace mini_db
```

### 模块2：缓冲池管理器

缓冲池是数据库的核心，管理内存中的页面缓存。

```cpp
// mini_db/storage/buffer_pool.hpp

#pragma once
#include "page.hpp"
#include "page_manager.hpp"
#include <unordered_map>
#include <list>
#include <memory>
#include <mutex>

namespace mini_db {

// 缓冲页面（带额外信息）
struct BufferedPage {
    DataPage page;              // 实际页面数据
    bool is_dirty;             // 是否被修改
    int pin_count;             // 引用计数

    BufferedPage() : is_dirty(false), pin_count(0) {}
};

// LRU替换策略的缓冲池
class BufferPool {
private:
    // 页面框架：页面ID -> 缓存页面的映射
    struct Frame {
        uint64_t page_id;
        std::shared_ptr<BufferedPage> buffered_page;

        Frame() : page_id(0), buffered_page(nullptr) {}
        Frame(uint64_t pid, std::shared_ptr<BufferedPage> bp)
            : page_id(pid), buffered_page(bp) {}
    };

    PageManager& page_manager_;             // 页面管理器
    size_t pool_size_;                      // 缓冲池大小（页数）
    std::unordered_map<uint64_t, Frame> page_table_;  // 页表
    std::list<uint64_t> lru_list_;          // LRU链表
    std::mutex mutex_;                      // 线程安全

    // 统计信息
    uint64_t hit_count_;
    uint64_t miss_count_;

public:
    BufferPool(PageManager& pm, size_t pool_size)
        : page_manager_(pm), pool_size_(pool_size),
          hit_count_(0), miss_count_(0) {}

    // 获取页面（从缓存或磁盘）
    std::shared_ptr<BufferedPage> FetchPage(uint64_t page_id) {
        std::lock_guard<std::mutex> lock(mutex_);

        // 1. 检查缓存
        auto it = page_table_.find(page_id);
        if (it != page_table_.end()) {
            // 命中缓存
            hit_count_++;

            // 更新LRU
            lru_list_.remove(page_id);
            lru_list_.push_front(page_id);

            return it->second.buffered_page;
        }

        // 2. 缓存未命中
        miss_count_++;

        // 3. 如果缓冲池已满，淘汰页面
        if (page_table_.size() >= pool_size_) {
            EvictPage();
        }

        // 4. 从磁盘读取页面
        auto buffered_page = std::make_shared<BufferedPage>();
        if (!page_manager_.ReadPage(page_id, buffered_page->page)) {
            return nullptr;
        }

        // 5. 加入缓存
        page_table_[page_id] = Frame(page_id, buffered_page);
        lru_list_.push_front(page_id);

        return buffered_page;
    }

    // 解除页面（减少引用计数）
    void UnpinPage(uint64_t page_id, bool is_dirty) {
        std::lock_guard<std::mutex> lock(mutex_);

        auto it = page_table_.find(page_id);
        if (it != page_table_.end()) {
            auto& buffered_page = it->second.buffered_page;
            buffered_page->pin_count--;
            if (is_dirty) {
                buffered_page->is_dirty = true;
            }
        }
    }

    // 刷所有脏页到磁盘
    void FlushAllPages() {
        std::lock_guard<std::mutex> lock(mutex_);

        for (auto& entry : page_table_) {
            auto& buffered_page = entry.second.buffered_page;
            if (buffered_page->is_dirty) {
                page_manager_.WritePage(entry.first,
                                        buffered_page->page);
                buffered_page->is_dirty = false;
            }
        }
    }

    // 获取统计信息
    double GetHitRatio() const {
        uint64_t total = hit_count_ + miss_count_;
        return total > 0 ? static_cast<double>(hit_count_) / total : 0.0;
    }

private:
    // 淘汰页面（LRU策略）
    void EvictPage() {
        if (lru_list_.empty()) {
            return;
        }

        // 找到最久未使用的页面
        uint64_t victim_id = lru_list_.back();

        auto it = page_table_.find(victim_id);
        if (it != page_table_.end()) {
            auto& buffered_page = it->second.buffered_page;

            // 如果被引用，跳过
            if (buffered_page->pin_count > 0) {
                // 查找下一个可用页面
                lru_list_.pop_back();
                return;
            }

            // 如果是脏页，写回磁盘
            if (buffered_page->is_dirty) {
                page_manager_.WritePage(victim_id,
                                        buffered_page->page);
            }

            // 从缓存移除
            page_table_.erase(it);
        }

        lru_list_.pop_back();
    }
};

// RAII页面访问器（自动管理引用计数）
class PageGuard {
private:
    BufferPool& buffer_pool_;
    uint64_t page_id_;
    std::shared_ptr<BufferedPage> buffered_page_;
    bool is_dirty_;

public:
    PageGuard(BufferPool& bp, uint64_t page_id)
        : buffer_pool_(bp), page_id_(page_id),
          buffered_page_(bp.FetchPage(page_id)),
          is_dirty_(false) {

        if (buffered_page_) {
            buffered_page_->pin_count++;
        }
    }

    ~PageGuard() {
        if (buffered_page_) {
            buffer_pool_.UnpinPage(page_id_, is_dirty_);
        }
    }

    // 获取页面
    DataPage* GetPage() {
        return buffered_page_ ? &buffered_page_->page : nullptr;
    }

    // 标记为脏页
    void MarkDirty() {
        is_dirty_ = true;
    }

    // 禁止拷贝
    PageGuard(const PageGuard&) = delete;
    PageGuard& operator=(const PageGuard&) = delete;
};

} // namespace mini_db
```

### 模块3：B+树索引

B+树是数据库最常用的索引结构。

```cpp
// mini_db/storage/btree.hpp

#pragma once
#include "page.hpp"
#include "buffer_pool.hpp"
#include <vector>
#include <optional>

namespace mini_db {

// B+树配置
constexpr uint32_t BTREE_ORDER = 128;  // 每个节点最多128个key

// B+树节点
class BTreeNode {
public:
    bool is_leaf;                      // 是否叶子节点
    uint32_t num_keys;                 // 当前key数量
    uint64_t keys[BTREE_ORDER - 1];    // key数组
    union {
        uint64_t children[BTREE_ORDER];      // 内部节点：子节点指针
        uint64_t next_leaf;                  // 叶子节点：下一叶子指针
    };

    BTreeNode() : is_leaf(true), num_keys(0) {
        std::memset(keys, 0, sizeof(keys));
        std::memset(children, 0, sizeof(children));
    }
};

// B+树索引
class BTreeIndex {
private:
    BufferPool& buffer_pool_;
    uint64_t root_page_id_;           // 根节点页面ID
    uint64_t metadata_page_id_;       // 元数据页面ID

public:
    BTreeIndex(BufferPool& bp) : buffer_pool_(bp),
                                 root_page_id_(0),
                                 metadata_page_id_(0) {
        // 创建新的B+树
        CreateNewTree();
    }

    // 插入key-value
    bool Insert(uint64_t key, uint64_t value) {
        // 1. 查找插入位置
        auto result = SearchToLeaf(root_page_id_, key);

        // 2. 插入到叶子节点
        if (result.leaf_node->num_keys < BTREE_ORDER - 1) {
            InsertIntoLeaf(result.leaf_node, key, value);
            return true;
        }

        // 3. 叶子节点已满，需要分裂
        return SplitAndInsert(key, value);
    }

    // 查找key
    std::optional<uint64_t> Search(uint64_t key) {
        return SearchFromNode(root_page_id_, key);
    }

    // 删除key
    bool Delete(uint64_t key) {
        return DeleteFromNode(root_page_id_, key, nullptr);
    }

    // 范围查询 [start_key, end_key]
    std::vector<uint64_t> RangeSearch(uint64_t start_key,
                                       uint64_t end_key) {
        std::vector<uint64_t> results;

        // 找到起始位置
        auto leaf_guard = buffer_pool_.FetchPage(
            FindLeafPage(root_page_id_, start_key));
        // ... 实现范围查询

        return results;
    }

private:
    // 创建新树
    void CreateNewTree() {
        // 分配元数据页
        metadata_page_id_ = /* PageManager::AllocatePage() */;

        // 分配根节点页
        root_page_id_ = /* PageManager::AllocatePage() */;

        // 初始化根节点
        PageGuard root_guard(buffer_pool_, root_page_id_);
        auto root_page = root_guard.GetPage();
        auto root_node = reinterpret_cast<BTreeNode*>(
            root_page->GetData());

        root_node->is_leaf = true;
        root_node->num_keys = 0;

        root_guard.MarkDirty();
    }

    // 从根节点向下搜索
    std::optional<uint64_t> SearchFromNode(uint64_t page_id,
                                            uint64_t key) {
        PageGuard node_guard(buffer_pool_, page_id);
        auto node_page = node_guard.GetPage();
        auto node = reinterpret_cast<BTreeNode*>(
            node_page->GetData());

        // 叶子节点：线性查找
        if (node->is_leaf) {
            for (uint32_t i = 0; i < node->num_keys; i++) {
                if (node->keys[i] == key) {
                    return node->keys[i];  // 简化：key即value
                }
            }
            return std::nullopt;
        }

        // 内部节点：找到合适的子节点
        uint32_t i = 0;
        while (i < node->num_keys && key >= node->keys[i]) {
            i++;
        }

        return SearchFromNode(node->children[i], key);
    }

    // 找到包含key的叶子节点
    uint64_t FindLeafPage(uint64_t page_id, uint64_t key) {
        PageGuard node_guard(buffer_pool_, page_id);
        auto node_page = node_guard.GetPage();
        auto node = reinterpret_cast<BTreeNode*>(
            node_page->GetData());

        if (node->is_leaf) {
            return page_id;
        }

        uint32_t i = 0;
        while (i < node->num_keys && key >= node->keys[i]) {
            i++;
        }

        return FindLeafPage(node->children[i], key);
    }

    // 搜索到叶子节点
    struct SearchResult {
        BTreeNode* leaf_node;
        uint64_t leaf_page_id;
        uint32_t index;
    };

    SearchResult SearchToLeaf(uint64_t page_id, uint64_t key) {
        SearchResult result;
        // ... 实现搜索逻辑
        return result;
    }

    // 插入到叶子节点
    void InsertIntoLeaf(BTreeNode* leaf,
                        uint64_t key,
                        uint64_t value) {
        uint32_t i = leaf->num_keys;

        // 找到插入位置
        while (i > 0 && key < leaf->keys[i - 1]) {
            leaf->keys[i] = leaf->keys[i - 1];
            i--;
        }

        leaf->keys[i] = key;
        leaf->num_keys++;
    }

    // 分裂节点
    bool SplitAndInsert(uint64_t key, uint64_t value) {
        // ... 实现节点分裂逻辑
        return true;
    }

    // 删除节点
    bool DeleteFromNode(uint64_t page_id, uint64_t key,
                       BTreeNode* parent) {
        // ... 实现删除逻辑
        return true;
    }
};

} // namespace mini_db
```

---

## 第二阶段：执行引擎实现

### 模块4：SQL解析器

使用递归下降解析器实现SQL解析。

```cpp
// mini_db/parser/sql_parser.hpp

#pragma once
#include "common/types.hpp"
#include <string>
#include <memory>
#include <vector>

namespace mini_db {

// SQL语句类型
enum class StatementType {
    SELECT,
    INSERT,
    UPDATE,
    DELETE,
    CREATE_TABLE,
    DROP_TABLE,
    UNKNOWN
};

// 抽象语法树节点
class ASTNode {
public:
    virtual ~ASTNode() = default;
    virtual std::string ToString() const = 0;
};

// 表达式节点
class Expression : public ASTNode {
public:
    virtual ~Expression() = default;
};

// 列引用表达式
class ColumnRef : public Expression {
private:
    std::string table_name_;
    std::string column_name_;

public:
    ColumnRef(const std::string& table, const std::string& column)
        : table_name_(table), column_name_(column) {}

    std::string ToString() const override {
        if (table_name_.empty()) {
            return column_name_;
        }
        return table_name_ + "." + column_name_;
    }
};

// 常量表达式
class Constant : public Expression {
private:
    Value value_;

public:
    explicit Constant(const Value& val) : value_(val) {}

    std::string ToString() const override {
        return value_.ToString();
    }
};

// 比较表达式
class ComparisonExpr : public Expression {
public:
    enum class Op {
        EQ, NE, LT, LE, GT, GE
    };

private:
    std::unique_ptr<Expression> left_;
    std::unique_ptr<Expression> right_;
    Op op_;

public:
    ComparisonExpr(std::unique_ptr<Expression> left,
                   std::unique_ptr<Expression> right,
                   Op op)
        : left_(std::move(left)),
          right_(std::move(right)),
          op_(op) {}

    std::string ToString() const override {
        std::string op_str;
        switch (op_) {
            case Op::EQ: op_str = "="; break;
            case Op::NE: op_str = "!="; break;
            case Op::LT: op_str = "<"; break;
            case Op::LE: op_str = "<="; break;
            case Op::GT: op_str = ">"; break;
            case Op::GE: op_str = ">="; break;
        }
        return left_->ToString() + " " + op_str + " " + right_->ToString();
    }
};

// SELECT语句
class SelectStatement : public ASTNode {
private:
    std::vector<std::unique_ptr<Expression>> select_list_;  // SELECT列表
    std::string from_table_;                                // FROM表
    std::unique_ptr<Expression> where_clause_;              // WHERE条件

public:
    SelectStatement(
        std::vector<std::unique_ptr<Expression>> select_list,
        const std::string& from_table,
        std::unique_ptr<Expression> where_clause)
        : select_list_(std::move(select_list)),
          from_table_(from_table),
          where_clause_(std::move(where_clause)) {}

    std::string ToString() const override {
        std::string result = "SELECT ";

        // SELECT列表
        for (size_t i = 0; i < select_list_.size(); i++) {
            if (i > 0) result += ", ";
            result += select_list_[i]->ToString();
        }

        result += " FROM " + from_table_;

        // WHERE子句
        if (where_clause_) {
            result += " WHERE " + where_clause_->ToString();
        }

        return result;
    }
};

// 词法分析器
class Lexer {
private:
    std::string input_;
    size_t pos_;

public:
    explicit Lexer(const std::string& input)
        : input_(input), pos_(0) {}

    enum class TokenType {
        KEYWORD,
        IDENTIFIER,
        NUMBER,
        STRING,
        OPERATOR,
        EOF,
        UNKNOWN
    };

    struct Token {
        TokenType type;
        std::string value;

        Token(TokenType t, const std::string& v)
            : type(t), value(v) {}
    };

    Token NextToken() {
        SkipWhitespace();

        if (pos_ >= input_.length()) {
            return Token(TokenType::EOF, "");
        }

        char c = input_[pos_];

        // 数字
        if (std::isdigit(c)) {
            return ReadNumber();
        }

        // 字符串
        if (c == '\'') {
            return ReadString();
        }

        // 运算符
        if (IsOperator(c)) {
            return ReadOperator();
        }

        // 标识符或关键字
        if (std::isalpha(c)) {
            return ReadIdentifier();
        }

        pos_++;
        return Token(TokenType::UNKNOWN, std::string(1, c));
    }

private:
    void SkipWhitespace() {
        while (pos_ < input_.length() &&
               std::isspace(input_[pos_])) {
            pos_++;
        }
    }

    Token ReadNumber() {
        size_t start = pos_;
        while (pos_ < input_.length() &&
               std::isdigit(input_[pos_])) {
            pos_++;
        }
        return Token(TokenType::NUMBER,
                     input_.substr(start, pos_ - start));
    }

    Token ReadString() {
        pos_++;  // 跳过开引号
        size_t start = pos_;
        while (pos_ < input_.length() && input_[pos_] != '\'') {
            if (input_[pos_] == '\\' && pos_ + 1 < input_.length()) {
                pos_++;  // 跳过转义字符
            }
            pos_++;
        }
        auto value = input_.substr(start, pos_ - start);
        pos_++;  // 跳过闭引号
        return Token(TokenType::STRING, value);
    }

    Token ReadOperator() {
        size_t start = pos_;
        while (pos_ < input_.length() && IsOperator(input_[pos_])) {
            pos_++;
        }
        return Token(TokenType::OPERATOR,
                     input_.substr(start, pos_ - start));
    }

    Token ReadIdentifier() {
        size_t start = pos_;
        while (pos_ < input_.length() &&
               (std::isalnum(input_[pos_]) || input_[pos_] == '_')) {
            pos_++;
        }
        auto value = input_.substr(start, pos_ - start);

        // 检查是否是关键字
        if (IsKeyword(value)) {
            return Token(TokenType::KEYWORD, value);
        }

        return Token(TokenType::IDENTIFIER, value);
    }

    bool IsOperator(char c) {
        return c == '=' || c == '!' || c == '<' ||
               c == '>' || c == '+' || c == '-';
    }

    bool IsKeyword(const std::string& str) {
        return str == "SELECT" || str == "FROM" ||
               str == "WHERE" || str == "INSERT" ||
               str == "INTO" || str == "VALUES";
    }
};

// SQL解析器
class SQLParser {
private:
    Lexer lexer_;
    Lexer::Token current_token_;

public:
    explicit SQLParser(const std::string& sql)
        : lexer_(sql) {
        current_token_ = lexer_.NextToken();
    }

    std::unique_ptr<ASTNode> Parse() {
        if (current_token_.type == Lexer::TokenType::KEYWORD) {
            if (current_token_.value == "SELECT") {
                return ParseSelect();
            }
            // 其他语句类型...
        }

        return nullptr;
    }

private:
    void Advance() {
        current_token_ = lexer_.NextToken();
    }

    void Expect(const std::string& expected) {
        if (current_token_.value != expected) {
            throw std::runtime_error("Expected: " + expected);
        }
        Advance();
    }

    std::unique_ptr<SelectStatement> ParseSelect() {
        Expect("SELECT");

        // 解析SELECT列表
        std::vector<std::unique_ptr<Expression>> select_list;
        select_list.push_back(ParseExpression());

        while (current_token_.value == ",") {
            Advance();
            select_list.push_back(ParseExpression());
        }

        // 解析FROM
        Expect("FROM");
        std::string from_table = current_token_.value;
        Advance();

        // 解析WHERE（可选）
        std::unique_ptr<Expression> where_clause;
        if (current_token_.value == "WHERE") {
            Advance();
            where_clause = ParseExpression();
        }

        return std::make_unique<SelectStatement>(
            std::move(select_list),
            from_table,
            std::move(where_clause)
        );
    }

    std::unique_ptr<Expression> ParseExpression() {
        return ParseComparison();
    }

    std::unique_ptr<Expression> ParseComparison() {
        auto left = ParseTerm();

        if (current_token_.type == Lexer::TokenType::OPERATOR) {
            auto op_str = current_token_.value;
            Advance();
            auto right = ParseTerm();

            ComparisonExpr::Op op;
            if (op_str == "=") op = ComparisonExpr::Op::EQ;
            else if (op_str == "!=") op = ComparisonExpr::Op::NE;
            else if (op_str == "<") op = ComparisonExpr::Op::LT;
            else if (op_str == "<=") op = ComparisonExpr::Op::LE;
            else if (op_str == ">") op = ComparisonExpr::Op::GT;
            else if (op_str == ">=") op = ComparisonExpr::Op::GE;

            return std::make_unique<ComparisonExpr>(
                std::move(left), std::move(right), op);
        }

        return left;
    }

    std::unique_ptr<Expression> ParseTerm() {
        // 简化实现：只处理列引用和常量
        if (current_token_.type == Lexer::TokenType::IDENTIFIER) {
            auto name = current_token_.value;
            Advance();

            // 检查是否是 table.column 形式
            if (current_token_.value == ".") {
                Advance();
                auto column = current_token_.value;
                Advance();
                return std::make_unique<ColumnRef>(name, column);
            }

            return std::make_unique<ColumnRef>("", name);
        }

        if (current_token_.type == Lexer::TokenType::NUMBER) {
            auto value = std::stoi(current_token_.value);
            Advance();
            return std::make_unique<Constant>(Value::INTEGER(value));
        }

        return nullptr;
    }
};

} // namespace mini_db
```

### 模块5：查询执行引擎

实现向量化的查询执行引擎。

```cpp
// mini_db/execution/execution_engine.hpp

#pragma once
#include "common/types.hpp"
#include "storage/buffer_pool.hpp"
#include <vector>
#include <memory>

namespace mini_db {

// 批大小（向量化）
constexpr uint32_t BATCH_SIZE = 1024;

// 向量（列数据）
class Vector {
private:
    LogicalType type_;
    std::vector<Value> values_;
    std::vector<bool> null_mask_;

public:
    Vector(LogicalType type, uint32_t capacity)
        : type_(type), values_(capacity),
          null_mask_(capacity, false) {}

    void Append(const Value& value) {
        values_.push_back(value);
        null_mask_.push_back(value.IsNull());
    }

    Value GetValue(uint32_t index) const {
        if (null_mask_[index]) {
            return Value(type_);
        }
        return values_[index];
    }

    uint32_t GetSize() const {
        return values_.size();
    }

    void Reset() {
        values_.clear();
        null_mask_.clear();
    }
};

// DataChunk（批数据）
class DataChunk {
private:
    std::vector<std::unique_ptr<Vector>> vectors_;
    uint32_t num_tuples_;

public:
    DataChunk(uint32_t num_columns)
        : vectors_(num_columns), num_tuples_(0) {}

    void Initialize(std::vector<LogicalType> types) {
        for (size_t i = 0; i < types.size(); i++) {
            vectors_[i] = std::make_unique<Vector>(
                types[i], BATCH_SIZE);
        }
    }

    Vector* GetVector(uint32_t col_idx) {
        return vectors_[col_idx].get();
    }

    void SetCardinality(uint32_t count) {
        num_tuples_ = count;
    }

    uint32_t GetCardinality() const {
        return num_tuples_;
    }

    void Reset() {
        for (auto& vec : vectors_) {
            vec->Reset();
        }
        num_tuples_ = 0;
    }
};

// 物理算子基类
class PhysicalOperator {
public:
    virtual ~PhysicalOperator() = default;

    // 初始化算子
    virtual void Open() = 0;

    // 获取下一个数据块
    virtual bool Next(DataChunk& chunk) = 0;

    // 关闭算子
    virtual void Close() = 0;
};

// 表扫描算子
class TableScanOperator : public PhysicalOperator {
private:
    BufferPool& buffer_pool_;
    std::string table_name_;
    uint64_t current_page_id_;
    bool is_open_;

public:
    TableScanOperator(BufferPool& bp, const std::string& table)
        : buffer_pool_(bp), table_name_(table),
          current_page_id_(0), is_open_(false) {}

    void Open() override {
        is_open_ = true;
        current_page_id_ = 1;  // 跳过元数据页
    }

    bool Next(DataChunk& chunk) override {
        if (!is_open_) {
            return false;
        }

        // 从缓冲池获取页面
        PageGuard page_guard(buffer_pool_, current_page_id_);
        auto page = page_guard.GetPage();

        if (!page) {
            return false;
        }

        // 解析页面数据到向量
        // ... 实现数据解析

        current_page_id_++;
        return true;
    }

    void Close() override {
        is_open_ = false;
    }
};

// 过滤算子
class FilterOperator : public PhysicalOperator {
private:
    std::unique_ptr<PhysicalOperator> child_;
    std::function<bool(const DataChunk&, uint32_t)> filter_fn_;

public:
    FilterOperator(std::unique_ptr<PhysicalOperator> child,
                   std::function<bool(const DataChunk&, uint32_t)> fn)
        : child_(std::move(child)), filter_fn_(fn) {}

    void Open() override {
        child_->Open();
    }

    bool Next(DataChunk& chunk) override {
        // 从子节点获取数据
        while (child_->Next(chunk)) {
            // 应用过滤
            uint32_t output_count = 0;

            for (uint32_t i = 0; i < chunk.GetCardinality(); i++) {
                if (filter_fn_(chunk, i)) {
                    // 保留满足条件的行
                    // ... 实现选择逻辑
                    output_count++;
                }
            }

            if (output_count > 0) {
                chunk.SetCardinality(output_count);
                return true;
            }
        }

        return false;
    }

    void Close() override {
        child_->Close();
    }
};

// 投影算子
class ProjectionOperator : public PhysicalOperator {
private:
    std::unique_ptr<PhysicalOperator> child_;
    std::vector<uint32_t> projection_list_;

public:
    ProjectionOperator(std::unique_ptr<PhysicalOperator> child,
                       std::vector<uint32_t> proj_list)
        : child_(std::move(child)),
          projection_list_(std::move(proj_list)) {}

    void Open() override {
        child_->Open();
    }

    bool Next(DataChunk& chunk) override {
        // 从子节点获取数据
        if (!child_->Next(chunk)) {
            return false;
        }

        // 应用投影（简化实现）
        DataChunk projected(projection_list_.size());
        projected.Initialize({LogicalType::INTEGER});

        for (size_t i = 0; i < projection_list_.size(); i++) {
            auto col_idx = projection_list_[i];
            // 复制列数据
        }

        return true;
    }

    void Close() override {
        child_->Close();
    }
};

// 执行引擎
class ExecutionEngine {
private:
    BufferPool& buffer_pool_;

public:
    ExecutionEngine(BufferPool& bp) : buffer_pool_(bp) {}

    // 执行查询计划
    std::unique_ptr<DataChunk> Execute(
        std::unique_ptr<PhysicalOperator> plan) {

        auto result = std::make_unique<DataChunk>(1);
        result->Initialize({LogicalType::INTEGER});

        plan->Open();

        while (plan->Next(*result)) {
            // 处理结果
        }

        plan->Close();

        return result;
    }
};

} // namespace mini_db
```

---

## 第三阶段：事务系统实现

### 模块6：并发控制（MVCC）

```cpp
// mini_db/transaction/mvcc.hpp

#pragma once
#include "common/types.hpp"
#include <atomic>
#include <unordered_map>
#include <mutex>

namespace mini_db {

// 事务时间戳
using Timestamp = uint64_t;

// 事务状态
enum class TransactionState {
    ACTIVE,
    COMMITTED,
    ABORTED
};

// 事务记录
class Transaction {
private:
    uint64_t transaction_id_;
    Timestamp start_ts_;
    TransactionState state_;

public:
    Transaction(uint64_t id, Timestamp ts)
        : transaction_id_(id), start_ts_(ts),
          state_(TransactionState::ACTIVE) {}

    uint64_t GetId() const { return transaction_id_; }
    Timestamp GetStartTimestamp() const { return start_ts_; }
    TransactionState GetState() const { return state_; }
    void SetState(TransactionState s) { state_ = s; }
};

// 元组版本（用于MVCC）
struct TupleVersion {
    Value value;
    Timestamp begin_ts;
    Timestamp end_ts;
    Transaction* creator;

    TupleVersion(const Value& val, Timestamp ts, Transaction* txn)
        : value(val), begin_ts(ts), end_ts(UINT64_MAX),
          creator(txn) {}
};

// MVCC管理器
class MVCCManager {
private:
    std::atomic<uint64_t> next_txn_id_;
    std::atomic<Timestamp> global_timestamp_;
    std::mutex mutex_;

    // 事务表
    std::unordered_map<uint64_t, Transaction*> transactions_;

    // 元组版本链（简化：key -> versions）
    std::unordered_map<uint64_t, std::vector<TupleVersion*>> tuple_versions_;

public:
    MVCCManager() : next_txn_id_(1), global_timestamp_(1) {}

    // 开始事务
    Transaction* BeginTransaction() {
        std::lock_guard<std::mutex> lock(mutex_);

        uint64_t txn_id = next_txn_id_++;
        Timestamp ts = global_timestamp_++;

        auto txn = new Transaction(txn_id, ts);
        transactions_[txn_id] = txn;

        return txn;
    }

    // 读取元组（MVCC可见性检查）
    bool Read(Transaction* txn, uint64_t key, Value& value) {
        std::lock_guard<std::mutex> lock(mutex_);

        auto it = tuple_versions_.find(key);
        if (it == tuple_versions_.end()) {
            return false;  // 元组不存在
        }

        auto& versions = it->second;

        // 找到对当前事务可见的版本
        for (auto version : versions) {
            if (IsVisible(txn, version)) {
                value = version->value;
                return true;
            }
        }

        return false;
    }

    // 写入元组
    bool Write(Transaction* txn, uint64_t key, const Value& value) {
        std::lock_guard<std::mutex> lock(mutex_);

        // 创建新版本
        auto version = new TupleVersion(value,
                                        txn->GetStartTimestamp(),
                                        txn);

        tuple_versions_[key].push_back(version);

        return true;
    }

    // 提交事务
    bool CommitTransaction(Transaction* txn) {
        std::lock_guard<std::mutex> lock(mutex_);

        Timestamp commit_ts = global_timestamp_++;

        // 更新所有写入版本的end_ts
        for (auto& entry : tuple_versions_) {
            for (auto version : entry.second) {
                if (version->creator == txn) {
                    version->end_ts = commit_ts;
                }
            }
        }

        txn->SetState(TransactionState::COMMITTED);
        return true;
    }

    // 回滚事务
    void AbortTransaction(Transaction* txn) {
        std::lock_guard<std::mutex> lock(mutex_);

        // 删除事务创建的所有版本
        for (auto& entry : tuple_versions_) {
            auto& versions = entry.second;
            versions.erase(
                std::remove_if(versions.begin(), versions.end(),
                    [txn](TupleVersion* v) {
                        return v->creator == txn;
                    }),
                versions.end()
            );
        }

        txn->SetState(TransactionState::ABORTED);
    }

private:
    // 检查版本是否对事务可见
    bool IsVisible(Transaction* txn, TupleVersion* version) {
        // 规则1: 版本的开始时间 <= 事务的开始时间
        if (version->begin_ts > txn->GetStartTimestamp()) {
            return false;
        }

        // 规则2: 版本的结束时间 > 事务的开始时间
        if (version->end_ts <= txn->GetStartTimestamp()) {
            return false;
        }

        // 规则3: 版本的创建者不是当前事务（避免读自己的未提交写入）
        if (version->creator == txn &&
            version->begin_ts == txn->GetStartTimestamp()) {
            return false;
        }

        return true;
    }
};

} // namespace mini_db
```

### 模块7：WAL（Write-Ahead Log）

```cpp
// mini_db/transaction/wal.hpp

#pragma once
#include "common/types.hpp"
#include <fstream>
#include <mutex>

namespace mini_db {

// WAL记录类型
enum class WalRecordType : uint8_t {
    BEGIN_TXN = 1,
    COMMIT_TXN = 2,
    ABORT_TXN = 3,
    INSERT = 4,
    UPDATE = 5,
    DELETE = 6,
    CHECKPOINT = 7
};

// WAL记录头部
struct WalRecordHeader {
    WalRecordType type;
    uint64_t txn_id;
    uint32_t record_size;

    WalRecordHeader(WalRecordType t, uint64_t tid, uint32_t size)
        : type(t), txn_id(tid), record_size(size) {}
};

// WAL管理器
class WriteAheadLog {
private:
    std::string wal_path_;
    std::ofstream wal_file_;
    std::mutex mutex_;
    uint64_t lsn_;  // Log Sequence Number

public:
    WriteAheadLog(const std::string& db_path)
        : lsn_(0) {
        wal_path_ = db_path + ".wal";
        wal_file_.open(wal_path_,
                       std::ios::out | std::ios::binary | std::ios::app);
    }

    ~WriteAheadLog() {
        if (wal_file_.is_open()) {
            wal_file_.close();
        }
    }

    // 写入事务开始
    uint64_t LogBeginTxn(uint64_t txn_id) {
        std::lock_guard<std::mutex> lock(mutex_);

        WalRecordHeader header(WalRecordType::BEGIN_TXN,
                               txn_id,
                               sizeof(WalRecordHeader));
        wal_file_.write(reinterpret_cast<const char*>(&header),
                       sizeof(header));
        wal_file_.flush();

        return lsn_++;
    }

    // 写入事务提交
    uint64_t LogCommitTxn(uint64_t txn_id) {
        std::lock_guard<std::mutex> lock(mutex_);

        WalRecordHeader header(WalRecordType::COMMIT_TXN,
                               txn_id,
                               sizeof(WalRecordHeader));
        wal_file_.write(reinterpret_cast<const char*>(&header),
                       sizeof(header));
        wal_file_.flush();

        return lsn_++;
    }

    // 写入插入操作
    uint64_t LogInsert(uint64_t txn_id, uint64_t key,
                       const Value& value) {
        std::lock_guard<std::mutex> lock(mutex_);

        // 序列化value
        std::vector<char> value_data;
        // ... 实现序列化

        WalRecordHeader header(WalRecordType::INSERT,
                               txn_id,
                               sizeof(WalRecordHeader) +
                               sizeof(key) + value_data.size());

        wal_file_.write(reinterpret_cast<const char*>(&header),
                       sizeof(header));
        wal_file_.write(reinterpret_cast<const char*>(&key),
                       sizeof(key));
        wal_file_.write(value_data.data(), value_data.size());
        wal_file_.flush();

        return lsn_++;
    }

    // 截断WAL（checkpoint后）
    void Truncate() {
        wal_file_.close();
        std::ofstream(wal_path_, std::ios::out | std::ios::binary |
                      std::ios::trunc);
        lsn_ = 0;
    }
};

} // namespace mini_db
```

---

## 第四阶段：项目整合

### 完整的Mini-DB实现

```cpp
// mini_db/mini_db.hpp

#pragma once
#include "storage/page_manager.hpp"
#include "storage/buffer_pool.hpp"
#include "storage/btree.hpp"
#include "execution/execution_engine.hpp"
#include "transaction/mvcc.hpp"
#include "transaction/wal.hpp"
#include "parser/sql_parser.hpp"
#include <memory>
#include <string>

namespace mini_db {

class MiniDB {
private:
    std::string db_path_;
    std::unique_ptr<PageManager> page_manager_;
    std::unique_ptr<BufferPool> buffer_pool_;
    std::unique_ptr<MVCCManager> mvcc_manager_;
    std::unique_ptr<WriteAheadLog> wal_;
    std::unique_ptr<ExecutionEngine> execution_engine_;

public:
    explicit MiniDB(const std::string& db_path)
        : db_path_(db_path) {

        // 初始化组件
        page_manager_ = std::make_unique<PageManager>(db_path);
        buffer_pool_ = std::make_unique<BufferPool>(
            *page_manager_, 1024);  // 1024页缓冲池
        mvcc_manager_ = std::make_unique<MVCCManager>();
        wal_ = std::make_unique<WriteAheadLog>(db_path);
        execution_engine_ = std::make_unique<ExecutionEngine>(
            *buffer_pool_);
    }

    // 执行SQL语句
    std::unique_ptr<DataChunk> ExecuteSQL(const std::string& sql) {
        // 1. 解析SQL
        SQLParser parser(sql);
        auto ast = parser.Parse();

        if (!ast) {
            throw std::runtime_error("Failed to parse SQL");
        }

        // 2. 开始事务
        auto txn = mvcc_manager_->BeginTransaction();

        // 3. 记录WAL
        wal_->LogBeginTxn(txn->GetId());

        // 4. 生成执行计划
        auto plan = GeneratePlan(ast.get());

        // 5. 执行计划
        auto result = execution_engine_->Execute(std::move(plan));

        // 6. 提交事务
        mvcc_manager_->CommitTransaction(txn);
        wal_->LogCommitTxn(txn->GetId());

        return result;
    }

private:
    // 生成执行计划（简化版）
    std::unique_ptr<PhysicalOperator> GeneratePlan(ASTNode* ast) {
        // 这里应该实现完整的查询规划
        // 简化：返回表扫描
        return std::make_unique<TableScanOperator>(
            *buffer_pool_, "table_name");
    }
};

} // namespace mini_db
```

---

## 课程总结

### 已实现的功能

✅ **存储引擎**
- 页面管理器
- 缓冲池（LRU替换）
- B+树索引

✅ **执行引擎**
- SQL词法和语法解析
- 查询执行算子
- 向量化执行

✅ **事务系统**
- MVCC并发控制
- WAL预写日志

### 下一步学习

1. **优化器实现**
   - 逻辑优化规则
   - 物理计划选择
   - 统计信息收集

2. **高级特性**
   - 数据压缩
   - 并行查询执行
   - 分布式扩展

3. **性能调优**
   - 查询性能分析
   - 索引优化
   - 并发性能优化

### 参考资源

**书籍：**
- "Database System Concepts" - Silberschatz
- "Database Internals" - Alex Petrov
- "Designing Data-Intensive Applications" - Martin Kleppmann

**论文：**
- "The Design of the B-tree Lock Protocol"
- "ARIES: A Transaction Recovery Method"
- "Designing a Qualitatively New Database System"

**代码：**
- DuckDB源码
- PostgreSQL源码
- SQLite源码

---

**最后更新：2026-01-23**

本课程提供了从零实现数据库系统的完整路线图。建议按照模块顺序逐步实现，每完成一个模块后进行测试和优化。
