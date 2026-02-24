# DuckDB 深度课程：Catalog元数据管理

> 本课程深入讲解DuckDB的元数据管理系统，包括Catalog的层次结构、各种元数据条目的实现、持久化机制和事务支持。

---

## 课程概览

### 学习目标

- 理解Catalog系统的架构设计
- 掌握Schema、Table、Index等元数据的管理
- 学习约束系统的实现
- 理解依赖管理和级联操作
- 掌握元数据的持久化机制
- 理解事务与元数据的关系

### 前置知识

- SQL数据库对象概念
- 事务处理基础
- MVCC基础
- 序列化/反序列化

---

## 第一部分：Catalog系统架构

### 1.1 Catalog层次结构

```
┌─────────────────────────────────────────────────────────┐
│              DuckDB Catalog 架构                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  DatabaseInstance (数据库实例)                         │
│  ├── Catalog (元数据目录)                            │
│  │   ├── SchemaCatalogEntry (模式)                   │
│  │   │   ├── TableCatalogEntry (表)                 │
│  │   │   │   ├── 列定义                           │
│  │   │   │   ├── 约束                             │
│  │   │   │   │   ├── 主键                          │
│  │   │   │   │   ├── 外键                          │
│  │   │   │   │   ├── 唯一约束                      │
│  │   │   │   │   └── 检查约束                      │
│  │   │   │   ├── 索引                             │
│  │   │   │   └── 统计信息                         │
│  │   │   ├── ViewCatalogEntry (视图)                 │
│  │   │   ├── IndexCatalogEntry (索引)               │
│  │   │   ├── SequenceCatalogEntry (序列)            │
│  │   │   ├── TableFunctionCatalogEntry (表函数)      │
│  │   │   └── ScalarFunctionCatalogEntry (标量函数)  │
│  │   │                                            │
│  │   ├── CatalogSet (集合管理)                       │
│  │   └── DependencyManager (依赖管理)                 │
│  │                                                │
│  └── CatalogTransaction (事务支持)                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 1.2 CatalogEntry基类

```cpp
// src/include/duckdb/catalog/catalog_entry.hpp

class CatalogEntry {
public:
    // 条目类型
    CatalogType type;

    // 条目名称
    string name;

    // 是否为内部条目（不能删除）
    bool internal;

    // 是否为临时条目
    bool temporary;

    // 创建时间戳（用于MVCC）
    transaction_t timestamp;

    // 父条目
    CatalogEntry *parent;

    // 子条目映射
    case_insensitive_map_t<unique_ptr<CatalogEntry>> children;

    // ==================== 构造函数 ====================

    CatalogEntry(CatalogType type, string name)
        : type(type), name(std::move(name)),
          internal(false), temporary(false),
          timestamp(0), parent(nullptr) {}

    virtual ~CatalogEntry() = default;

    // ==================== 虚函数 ====================

    // 转换为SQL
    virtual string ToSQL() const = 0;

    // 序列化
    virtual void Serialize(Serializer &serializer) const;

    // ==================== 类型转换 ====================

    // 转换为SchemaCatalogEntry
    virtual SchemaCatalogEntry &CastToSchema() {
        throw InternalException("Not a schema entry");
    }

    // 转换为TableCatalogEntry
    virtual TableCatalogEntry &CastToTable() {
        throw InternalException("Not a table entry");
    }

    // 转换为ViewCatalogEntry
    virtual ViewCatalogEntry &CastToView() {
        throw InternalException("Not a view entry");
    }
};
```

---

## 第二部分：Schema管理

### 2.1 SchemaCatalogEntry实现

```cpp
// src/catalog/schema_catalog_entry.cpp

class SchemaCatalogEntry : public CatalogEntry {
public:
    SchemaCatalogEntry(Catalog *catalog,
                       string name,
                       transaction_t timestamp)
        : CatalogEntry(CatalogType::SCHEMA_ENTRY,
                       std::move(name)),
          catalog(catalog) {

        this->timestamp = timestamp;

        // 初始化集合
        tables = make_unique<CatalogSet<TableCatalogEntry>>(catalog);
        views = make_unique<CatalogSet<ViewCatalogEntry>>(catalog);
        indexes = make_unique<CatalogSet<IndexCatalogEntry>>(catalog);
        sequences = make_unique<CatalogSet<SequenceCatalogEntry>>(catalog);
        table_functions = make_unique<CatalogSet<TableFunctionCatalogEntry>>(catalog);
        scalar_functions = make_unique<CatalogSet<ScalarFunctionCatalogEntry>>(catalog);
    }

    // ==================== 扫描接口 ====================

    // 扫描所有条目
    void Scan(ClientContext &context,
             CatalogType type,
             const std::function<void(CatalogEntry &)> &callback) override {

        switch (type) {
        case CatalogType::TABLE_ENTRY:
            tables->Scan(context, callback);
            break;
        case CatalogType::VIEW_ENTRY:
            views->Scan(context, callback);
            break;
        case CatalogType::INDEX_ENTRY:
            indexes->Scan(context, callback);
            break;
        // ... 其他类型
        default:
            break;
        }
    }

    // ==================== 创建接口 ====================

    // 创建表
    optional_ptr<CatalogEntry> CreateTable(
        CatalogTransaction transaction,
        BoundCreateTableInfo &info) override {

        // 检查表是否已存在
        if (tables->GetEntry(info.table)) {
            throw CatalogException("Table '%s' already exists",
                                  info.table);
        }

        // 创建表条目
        auto table_entry = make_unique<TableCatalogEntry>(
            catalog, info.table, *this
        );

        // 添加到集合
        tables->CreateEntry(transaction, info.table,
                         std::move(table_entry));

        return tables->GetEntry(info.table);
    }

    // 创建视图
    optional_ptr<CatalogEntry> CreateView(
        CatalogTransaction transaction,
        CreateViewInfo &info) override {

        if (views->GetEntry(info.view_name)) {
            throw CatalogException("View '%s' already exists",
                                  info.view_name);
        }

        auto view_entry = make_unique<ViewCatalogEntry>(
            catalog, info.view_name, *this, info
        );

        views->CreateEntry(transaction, info.view_name,
                         std::move(view_entry));

        return views->GetEntry(info.view_name);
    }

private:
    Catalog *catalog;

    // 各种对象的集合
    unique_ptr<CatalogSet<TableCatalogEntry>> tables;
    unique_ptr<CatalogSet<ViewCatalogEntry>> views;
    unique_ptr<CatalogSet<IndexCatalogEntry>> indexes;
    unique_ptr<CatalogSet<SequenceCatalogEntry>> sequences;
    unique_ptr<CatalogSet<TableFunctionCatalogEntry>> table_functions;
    unique_ptr<CatalogSet<ScalarFunctionCatalogEntry>> scalar_functions;
};
```

### 2.2 内置Schema

```cpp
// DuckDB的内置Schema

// ==================== main Schema ====================
// 包含用户创建的所有表和视图
// 示例：CREATE TABLE users (...) → main.users

// ==================== system Schema ====================
// 系统元数据表
// 示例：system.information_schema_tables

// ==================== temp Schema ====================
// 临时表（会话结束后自动删除）
// 示例：CREATE TEMP TABLE temp_users (...)

// ==================== pg_catalog Schema ====================
// PostgreSQL兼容性
// 示例：pg_catalog.pg_namespace

// Schema查找顺序
string Catalog::GetDefaultSchema(ClientContext &context) {
    // 1. 首先检查搜索路径
    auto &search_path = context.registered_state.search_path;

    if (!search_path.empty()) {
        return search_path[0];
    }

    // 2. 返回默认Schema（main）
    return DEFAULT_SCHEMA;  // "main"
}
```

---

## 第三部分：表管理

### 3.1 TableCatalogEntry实现

```cpp
// src/catalog/table_catalog_entry.cpp

class TableCatalogEntry : public StandardEntry {
public:
    TableCatalogEntry(Catalog *catalog,
                     SchemaCatalogEntry &schema,
                     string name,
                     ColumnList &columns,
                     vector<unique_ptr<Constraint>> &constraints)
        : StandardEntry(CatalogType::TABLE_ENTRY,
                       catalog, name),
          schema(schema),
          columns(std::move(columns)),
          constraints(std::move(constraints)) {

        // 提取约束信息
        for (auto &constraint : constraints) {
            switch (constraint->type) {
            case ConstraintType::PRIMARY_KEY:
                primary_key = (PrimaryKeyConstraint *)constraint.get();
                break;
            // ... 其他约束类型
            }
        }
    }

    // ==================== 列操作 ====================

    // 检查列是否存在
    bool ColumnExists(const string &name) const {
        return GetColumn(name) != nullptr;
    }

    // 获取列定义
    const ColumnDefinition &GetColumn(const string &name) const {
        for (auto &col : columns.GetColumns()) {
            if (col.Name() == name) {
                return col;
            }
        }
        throw InternalException("Column '%s' not found", name);
    }

    // 获取列索引
    LogicalIndex GetColumnIndex(string &name, bool if_exists = false) const {
        auto &col = GetColumn(name);
        return columns.GetLogicalIndex(col);
    }

    // ==================== 约束操作 ====================

    // 获取主键约束
    optional_ptr<Constraint> GetPrimaryKey() const {
        if (primary_key) {
            return primary_key->shared_from_this();
        }
        return nullptr;
    }

    // 检查是否有主键
    bool HasPrimaryKey() const {
        return primary_key != nullptr;
    }

    // 获取所有约束
    const vector<unique_ptr<Constraint>> &GetConstraints() const {
        return constraints;
    }

private:
    SchemaCatalogEntry &schema;

    // 列定义列表
    ColumnList columns;

    // 约束列表
    vector<unique_ptr<Constraint>> constraints;

    // 主键约束引用
    PrimaryKeyConstraint *primary_key = nullptr;
};
```

### 3.2 列定义

```cpp
// src/include/duckdb/catalog/column_definition.hpp

class ColumnDefinition {
public:
    string name;
    LogicalType type;

    // 列属性
    bool nullable;
    unique_ptr<Value> default_value;
    unique_ptr<Constraint> generated_constraint;

    // 存储信息
    string oid;
    idx_t storage_oid;
    bool compression_pushdown;

    // 统计信息
    unique_ptr<ColumnStatistics> stats;

    ColumnDefinition(string name, LogicalType type)
        : name(std::move(name)), type(std::move(type)),
          nullable(true), storage_oid(0),
          compression_pushdown(false) {}

    // ==================== 序列化 ====================

    void Serialize(Serializer &serializer) const {
        FieldWriter writer(serializer);
        writer.WriteString(name);
        writer.WriteSerializable(type);
        writer.WriteField<bool>(nullable);
        writer.WriteOptional(default_value);
        writer.WriteField<string>(oid);
        writer.Finalize();
    }

    static ColumnDefinition Deserialize(Deserializer &deserializer) {
        FieldReader reader(deserializer);
        auto name = reader.ReadRequired<string>();
        auto type = reader.ReadRequired<LogicalType>();
        auto nullable = reader.ReadRequired<bool>();
        auto default_value = reader.ReadOptional<Value>();
        auto oid = reader.ReadRequired<string>();
        reader.Finalize();

        ColumnDefinition col(name, type);
        col.nullable = nullable;
        col.default_value = std::move(default_value);
        col.oid = std::move(oid);
        return col;
    }
};
```

### 3.3 约束系统

```cpp
// 约束类型枚举

enum class ConstraintType : uint8_t {
    INVALID = 0,           // 无效约束
    NOT_NULL = 1,          // NOT NULL约束
    CHECK = 2,             // CHECK约束
    UNIQUE = 3,            // UNIQUE约束
    PRIMARY_KEY = 4,       // PRIMARY KEY约束
    FOREIGN_KEY = 5,        // FOREIGN KEY约束
    GENERATED = 6           // 生成列约束
};

// 约束基类

class Constraint {
public:
    ConstraintType type;
    string expression;

    Constraint(ConstraintType type)
        : type(type) {}

    virtual ~Constraint() = default;

    // 转换为具体类型
    virtual NotNullConstraint &CastToNotNull() {
        throw InternalException("Not a NOT NULL constraint");
    }

    virtual UniqueConstraint &CastToUnique() {
        throw InternalException("Not a UNIQUE constraint");
    }

    virtual ForeignKeyConstraint &CastToForeignKey() {
        throw InternalException("Not a FOREIGN KEY constraint");
    }
};

// ==================== NOT NULL约束 ====================

class NotNullConstraint : public Constraint {
public:
    NotNullConstraint()
        : Constraint(ConstraintType::NOT_NULL) {}
};

// ==================== UNIQUE约束 ====================

class UniqueConstraint : public Constraint {
public:
    // 约束名称
    string name;

    // 包含的列索引
    vector<idx_t> columns;

    // 约束类型（PRIMARY KEY 或 UNIQUE）
    bool is_primary_key;

    UniqueConstraint(vector<idx_t> columns, bool is_primary_key)
        : Constraint(is_primary_key ? ConstraintType::PRIMARY_KEY
                                    : ConstraintType::UNIQUE),
          columns(std::move(columns)),
          is_primary_key(is_primary_key) {}
};

// ==================== FOREIGN KEY约束 ====================

class ForeignKeyConstraint : public Constraint {
public:
    // 约束名称
    string name;

    // 信息Schema名称
    string info_schema;

    // 外键列索引
    vector<idx_t> fk_keys;

    // 引用表名
    string pk_table;

    // 引用Schema名
    string pk_schema;

    // 主键列索引
    vector<idx_t> pk_keys;

    // 删除行为
    OnDeleteRule on_delete;

    // 更新行为
    OnUpdateRule on_update;

    ForeignKeyConstraint()
        : Constraint(ConstraintType::FOREIGN_KEY),
          on_delete(OnDeleteRule::NO_ACTION),
          on_update(OnUpdateRule::NO_ACTION) {}
};

// 删除/更新行为规则
enum class OnDeleteRule : uint8_t {
    NO_ACTION = 0,
    RESTRICT = 1,
    CASCADE = 2,
    SET_NULL = 3,
    SET_DEFAULT = 4
};
```

---

## 第四部分：索引管理

### 4.1 IndexCatalogEntry实现

```cpp
// src/catalog/index_catalog_entry.cpp

class IndexCatalogEntry : public StandardEntry {
public:
    IndexCatalogEntry(Catalog *catalog,
                      TableCatalogEntry &table,
                      string name,
                      IndexType index_type,
                      vector<column_t> column_ids,
                      IndexConstraintType constraint_type)
        : StandardEntry(CatalogType::INDEX_ENTRY,
                       catalog, name),
          table(table),
          index_type(index_type),
          column_ids(std::move(column_ids)),
          constraint_type(constraint_type) {}

    // ==================== 索引属性 ====================

    // 是否为唯一索引
    bool IsUnique() const {
        return constraint_type == IndexConstraintType::UNIQUE ||
               constraint_type == IndexConstraintType::PRIMARY_KEY;
    }

    // 是否为主键索引
    bool IsPrimary() const {
        return constraint_type == IndexConstraintType::PRIMARY_KEY;
    }

    // 获取索引类型
    string GetIndexTypeString() const {
        switch (index_type) {
        case IndexType::ART_INDEX:
            return "ART";
        case IndexType::BPTREE_INDEX:
            return "BPTREE";
        case IndexType::HASH_INDEX:
            return "HASH";
        default:
            return "INVALID";
        }
    }

    // 获取索引列
    const vector<column_t> &GetColumnIds() const {
        return column_ids;
    }

private:
    TableCatalogEntry &table;

    // 索引类型
    IndexType index_type;

    // 约束类型
    IndexConstraintType constraint_type;

    // 索引列
    vector<column_t> column_ids;
};
```

### 4.2 索引类型

```cpp
// 索引类型枚举

enum class IndexType : uint8_t {
    INVALID = 0,
    ART_INDEX,        // Adaptive Radix Tree（DuckDB默认）
    BPTREE_INDEX,     // B+树索引
    HASH_INDEX,       // 哈希索引
    CSI_INDEX         // 列存储索引（实验性）
};

// ART (Adaptive Radix Tree) 索引
// - DUCKDB的默认索引类型
// - 高度优化的基数树实现
// - 支持前缀压缩和路径压缩
// - 对等值查询和范围查询都高效
```

---

## 第五部分：依赖管理

### 5.1 依赖关系

```cpp
// src/catalog/dependency_manager.hpp

// 依赖者（Dependent）
enum class DependencyDependent : uint8_t {
    TABLE_ENTRY,        // 表
    VIEW_ENTRY,         // 视图
    TYPE_ENTRY,        // 类型
    SEQUENCE_ENTRY      // 序列
};

// 被依赖对象（Subject）
enum class DependencySubject : uint8_t {
    TABLE_ENTRY,        // 表
    COLUMN_ENTRY,       // 列
    TYPE_ENTRY,         // 类型
    SEQUENCE_ENTRY      // 序列
    COLLATION_ENTRY     // 排序规则
};

// 依赖信息
struct DependencyInfo {
    DependencyDependent dependent;
    DependencySubject subject;

    // 依赖对象标识
    string schema;
    string name;

    // 依赖类型
    DependencyType type;
};

// 依赖类型
enum class DependencyType {
    REGULAR,           // 常规依赖（DROP时需要检查）
    OWNED,             // 所有权依赖（级联删除）
    AUTOMATIC_DEPENDENCY // 自动依赖（如视图对表的依赖）
};
```

### 5.2 DependencyManager实现

```cpp
// src/catalog/dependency_manager.cpp

class DependencyManager {
public:
    // ==================== 注册依赖 ====================

    // 添加表依赖
    void AddObject(TableCatalogEntry &object,
                 CatalogEntry &dependent) {

        DependencyInfo info;
        info.subject = DependencySubject::TABLE_ENTRY;
        info.schema = object.schema.name;
        info.name = object.name;
        info.dependent = DependencyDependent::TABLE_ENTRY;
        info.type = DependencyType::REGULAR;

        dependencies.push_back(std::move(info));
    }

    // 添加列依赖
    void AddColumn(TableCatalogEntry &table,
                   string &column,
                   CatalogEntry &dependent) {

        DependencyInfo info;
        info.subject = DependencySubject::COLUMN_ENTRY;
        info.schema = table.schema.name;
        info.name = table.name + "." + column;
        info.dependent = DependencyDependent::TABLE_ENTRY;
        info.type = DependencyType::REGULAR;

        dependencies.push_back(std::move(info));
    }

    // ==================== 依赖检查 ====================

    // 检查表是否被依赖
    bool IsDependent(TableCatalogEntry &object) {
        for (auto &dep : dependencies) {
            if (dep.subject == DependencySubject::TABLE_ENTRY &&
                dep.schema == object.schema.name &&
                dep.name == object.name) {
                return true;
            }
        }
        return false;
    }

    // ==================== 级联操作 ====================

    // 获取导出顺序（按依赖关系排序）
    vector<DependencyInfo> GetExportOrder(bool cascade = false) {
        // 拓扑排序：确保依赖对象在依赖者之前导出
        // 如果 cascade = true，包含所有权依赖
        vector<DependencyInfo> result;

        TopologicalSort(dependencies, result, cascade);

        return result;
    }

private:
    // 所有依赖关系
    vector<DependencyInfo> dependencies;

    // 拓扑排序
    void TopologicalSort(vector<DependencyInfo> &deps,
                        vector<DependencyInfo> &result,
                        bool include_owned) {
        // Kahn算法实现
        // 1. 计算入度
        map<string, int> in_degree;

        for (auto &dep : deps) {
            string key = dep.schema + "." + dep.name;
            in_degree[key] = 0;
        }

        for (auto &dep : deps) {
            if (!include_owned &&
                dep.type == DependencyType::OWNED) {
                continue;
            }

            string dependent = GetDependentKey(dep);
            in_degree[dependent]++;
        }

        // 2. 处理入度为0的节点
        queue<string> queue;
        for (auto &entry : in_degree) {
            if (entry.second == 0) {
                queue.push(entry.first);
            }
        }

        // 3. 拓扑排序
        while (!queue.empty()) {
            auto key = queue.front();
            queue.pop();

            // 添加到结果
            auto dep = FindDependencyByKey(key);
            if (dep) {
                result.push_back(*dep);
            }

            // 减少依赖节点入度
            for (auto &other_dep : deps) {
                string dependent = GetDependentKey(*other_dep);
                if (dependent == key) {
                    in_degree[other_dep.schema + "." + other_dep.name]--;
                    if (in_degree[other_dep.schema + "." + other_dep.name] == 0) {
                        queue.push(other_dep.schema + "." + other_dep.name);
                    }
                }
            }
        }
    }
};
```

---

## 第六部分：持久化机制

### 6.1 序列化机制

```cpp
// 元数据序列化

class CatalogMetadata {
public:
    // 序列化所有Schema
    static void Serialize(Serializer &serializer,
                         Catalog &catalog) {

        // 1. 序列化Schema列表
        auto schemas = catalog.GetSchemas();

        serializer.WriteProperty(101, "schemas", schemas.size());

        idx_t schema_idx = 0;
        schemas->Scan(*context, [&](CatalogEntry &entry) {
            auto &schema = entry.CastToSchema();

            FieldWriter writer(serializer);
            writer.WriteString(schema.name);
            writer.WriteField<transaction_t>(schema.timestamp);

            // 序列化Schema下的所有对象
            SerializeSchemaObjects(serializer, schema);

            writer.Finalize();
            schema_idx++;
        });
    }

private:
    static void SerializeSchemaObjects(Serializer &serializer,
                                     SchemaCatalogEntry &schema) {
        // 序列化表
        serializer.WriteProperty(102, "tables",
                                schema.tables->GetEntryCount());

        schema.tables->Scan(*context, [&](CatalogEntry &entry) {
            auto &table = entry.CastToTable();
            table.Serialize(serializer);
        });

        // 序列化视图
        serializer.WriteProperty(103, "views",
                                schema.views->GetEntryCount());

        schema.views->Scan(*context, [&](CatalogEntry &entry) {
            auto &view = entry.CastToView();
            view.Serialize(serializer);
        });

        // 序列化索引
        serializer.WriteProperty(104, "indexes",
                                schema.indexes->GetEntryCount());

        schema.indexes->Scan(*context, [&](CatalogEntry &entry) {
            auto &index = entry.CastToIndex();
            index.Serialize(serializer);
        });
    }
};
```

### 6.2 反序列化机制

```cpp
// 元数据反序列化

class CatalogMetadata {
public:
    // 反序列化所有Schema
    static void Deserialize(Deserializer &deserializer,
                           Catalog &catalog) {

        auto schema_count = deserializer.ReadProperty<idx_t>(101);

        for (idx_t i = 0; i < schema_count; i++) {
            FieldReader reader(deserializer);

            auto schema_name = reader.ReadRequired<string>();
            auto timestamp = reader.ReadRequired<transaction_t>();

            // 创建Schema
            auto &schema = catalog.CreateSchema(
                context, schema_name, timestamp
            );

            // 反序列化Schema对象
            DeserializeSchemaObjects(deserializer, schema);

            reader.Finalize();
        }
    }

private:
    static void DeserializeSchemaObjects(Deserializer &deserializer,
                                       SchemaCatalogEntry &schema) {
        // 反序列化表
        auto table_count = deserializer.ReadProperty<idx_t>(102);

        for (idx_t i = 0; i < table_count; i++) {
            auto table_info = TableCatalogEntry::Deserialize(deserializer);

            schema.CreateTable(transaction, *table_info);
        }

        // 反序列化视图
        auto view_count = deserializer.ReadProperty<idx_t>(103);

        for (idx_t i = 0; i < view_count; i++) {
            auto view_info = ViewCatalogEntry::Deserialize(deserializer);

            schema.CreateView(transaction, *view_info);
        }

        // 反序列化索引
        auto index_count = deserializer.ReadProperty<idx_t>(104);

        for (idx_t i = 0; i < index_count; i++) {
            auto index_info = IndexCatalogEntry::Deserialize(deserializer);

            schema.CreateIndex(transaction, *index_info);
        }
    }
};
```

---

## 第七部分：事务与元数据

### 7.1 Catalog事务

```cpp
// src/catalog/catalog_transaction.hpp

class CatalogTransaction {
public:
    // ==================== 事务状态 ====================

    // 关联的数据库实例
    optional_ptr<DatabaseInstance> db;

    // 关联的上下文
    optional_ptr<ClientContext> context;

    // 关联的事务
    optional_ptr<Transaction> transaction;

    // 事务ID
    transaction_t transaction_id;

    // 开始时间
    transaction_t start_time;

    // ==================== 事务操作 ====================

    // 创建表
    TableCatalogEntry *CreateTable(BoundCreateTableInfo &info) {
        // 1. 获取Schema
        auto &schema = catalog->GetSchema(info.schema);

        // 2. 创建表
        auto entry = schema.CreateTable(*this, info);

        // 3. 记录元数据变更
        if (db) {
            db->GetDatabaseManager().MarkMetadataModified();
        }

        return entry;
    }

    // 删除表
    void DropTable(SchemaCatalogEntry &schema,
                  string &table_name,
                  bool cascade = false) {

        // 1. 检查依赖
        if (dependency_manager->IsDependent(table)) {
            if (cascade) {
                // 级联删除依赖对象
                CascadeDrop(table);
            } else {
                throw CatalogException("Cannot drop table '%s': "
                                      "other objects depend on it",
                                      table_name);
            }
        }

        // 2. 删除表
        schema.DropTable(*this, table_name);

        // 3. 记录元数据变更
        if (db) {
            db->GetDatabaseManager().MarkMetadataModified();
        }
    }

private:
    Catalog *catalog;
    DependencyManager *dependency_manager;
};
```

### 7.2 MVCC支持

```cpp
// 多版本并发控制

class CatalogEntry {
public:
    // ==================== 版本管理 ====================

    // 创建时间戳
    transaction_t timestamp;

    // 子条目（使用指针管理版本链）
    unique_ptr<CatalogEntry> child;

    // 父条目
    CatalogEntry *parent;

    // ==================== 可见性检查 ====================

    // 检查条目是否对事务可见
    bool isVisible(CatalogTransaction &transaction,
                 transaction_t query_timestamp) const {

        // 1. 检查时间戳
        if (timestamp > query_timestamp) {
            // 条目在查询之后创建
            return false;
        }

        // 2. 检查删除
        if (deleted) {
            if (deletion_time > query_timestamp) {
                // 删除在查询之后
                return true;  // 查询时刻条目仍存在
            } else {
                return false;  // 条目已被删除
            }
        }

        return true;
    }

    // ==================== 删除管理 ====================

    bool deleted;
    transaction_t deletion_time;

    // 标记删除
    void MarkDeleted(transaction_t delete_time) {
        deleted = true;
        deletion_time = delete_time;
    }
};
```

---

## 学习总结

### Catalog系统关键要点

1. **层次结构**：Database → Schema → Table → Column
2. **类型安全**：强类型的元数据条目
3. **事务支持**：完整的MVCC支持
4. **依赖管理**：自动处理对象依赖
5. **持久化**：序列化/反序列化支持

### 元数据操作对照

| 操作 | CatalogEntry | 事务支持 | 级联 |
|------|--------------|----------|------|
| CREATE | CreateXXX | ✅ | ❌ |
| ALTER | AlterEntry | ✅ | ❌ |
| DROP | DropEntry | ✅ | ✅ (cascade) |
| SELECT | GetEntry/Scan | ✅ | ❌ |

### 推荐阅读

**论文：**
- "A Study of the MVCC Mechanism in Database Systems"
- "Metadata Management in Distributed Databases"
- "Efficient Locking for Concurrent Operations on B-Trees"

**代码位置：**
- `src/catalog/catalog.cpp` - Catalog主实现
- `src/catalog/schema_catalog_entry.cpp` - Schema实现
- `src/catalog/table_catalog_entry.cpp` - Table实现
- `src/catalog/dependency_manager.cpp` - 依赖管理

**相关课程：**
- [DuckDB深度课程_Planner与Binder](./DuckDB深度课程_Planner与Binder.md)
- [DuckDB深度课程_并发控制与事务处理](./DuckDB深度课程_并发控制与事务处理.md)

---

**最后更新：2026-01-23**
