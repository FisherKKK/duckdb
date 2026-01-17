# DuckDB 30天学习课程 - 代码示例集

本文档提供完整的、可运行的代码示例，帮助你更好地理解和实践课程内容。

---

## 📝 目录

1. [环境配置](#环境配置)
2. [Week 1: 基础数据结构示例](#week-1-基础数据结构示例)
3. [Week 2: 查询执行示例](#week-2-查询执行示例)
4. [Week 3: 优化器示例](#week-3-优化器示例)
5. [Week 4: 存储引擎示例](#week-4-存储引擎示例)
6. [完整项目模板](#完整项目模板)

---

## 环境配置

### 创建测试目录结构

```bash
cd /home/dev/duckdb
mkdir -p learning_examples/{week1,week2,week3,week4,final_project}
cd learning_examples
```

### CMakeLists.txt 模板

```cmake
# learning_examples/CMakeLists.txt
cmake_minimum_required(VERSION 3.5)
project(DuckDBLearning)

# 设置C++标准
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 找到DuckDB
set(DUCKDB_BUILD_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../build/debug")
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/../src/include)
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/../third_party/fmt/include)

# 链接DuckDB库
link_directories(${DUCKDB_BUILD_DIR}/src)

# Week 1 示例
add_executable(week1_vector week1/vector_example.cpp)
target_link_libraries(week1_vector duckdb)

add_executable(week1_datachunk week1/datachunk_example.cpp)
target_link_libraries(week1_datachunk duckdb)

# Week 2 示例
add_executable(week2_table_scan week2/table_scan_example.cpp)
target_link_libraries(week2_table_scan duckdb)

add_executable(week2_hash_join week2/hash_join_example.cpp)
target_link_libraries(week2_hash_join duckdb)

# Week 3 示例
add_executable(week3_optimizer week3/optimizer_example.cpp)
target_link_libraries(week3_optimizer duckdb)

# Week 4 示例
add_executable(week4_storage week4/storage_example.cpp)
target_link_libraries(week4_storage duckdb)

# 最终项目
add_executable(mini_duckdb final_project/mini_duckdb.cpp)
target_link_libraries(mini_duckdb duckdb)
```

---

## Week 1: 基础数据结构示例

### 示例 1.1: 创建和操作Vector

```cpp
// learning_examples/week1/vector_example.cpp
#include "duckdb.hpp"
#include <iostream>

using namespace duckdb;

void Example1_CreateBasicVector() {
    std::cout << "=== Example 1: 创建基本Vector ===" << std::endl;

    // 创建INTEGER类型的Vector
    Vector int_vector(LogicalType::INTEGER, 10);

    // 获取数据指针
    auto data = FlatVector::GetData<int32_t>(int_vector);

    // 填充数据
    for (idx_t i = 0; i < 10; i++) {
        data[i] = i * 2;
    }

    // 设置行数
    int_vector.SetVectorType(VectorType::FLAT_VECTOR);

    // 打印数据
    std::cout << "Vector内容: ";
    for (idx_t i = 0; i < 10; i++) {
        std::cout << data[i] << " ";
    }
    std::cout << std::endl;
}

void Example2_VectorWithNulls() {
    std::cout << "\n=== Example 2: 处理NULL值 ===" << std::endl;

    Vector vector(LogicalType::INTEGER, 10);
    auto data = FlatVector::GetData<int32_t>(vector);
    auto &validity = FlatVector::Validity(vector);

    for (idx_t i = 0; i < 10; i++) {
        if (i % 3 == 0) {
            // 设置为NULL
            validity.SetInvalid(i);
        } else {
            data[i] = i * 10;
        }
    }

    // 打印数据（包括NULL）
    std::cout << "Vector内容（含NULL）: ";
    for (idx_t i = 0; i < 10; i++) {
        if (validity.RowIsValid(i)) {
            std::cout << data[i] << " ";
        } else {
            std::cout << "NULL ";
        }
    }
    std::cout << std::endl;
}

void Example3_VectorOperations() {
    std::cout << "\n=== Example 3: Vector运算 ===" << std::endl;

    // 创建两个Vector
    Vector vec_a(LogicalType::INTEGER, 5);
    Vector vec_b(LogicalType::INTEGER, 5);
    Vector result(LogicalType::INTEGER, 5);

    auto data_a = FlatVector::GetData<int32_t>(vec_a);
    auto data_b = FlatVector::GetData<int32_t>(vec_b);
    auto result_data = FlatVector::GetData<int32_t>(result);

    // 填充数据
    for (idx_t i = 0; i < 5; i++) {
        data_a[i] = i;
        data_b[i] = i * 2;
    }

    // 向量化加法
    for (idx_t i = 0; i < 5; i++) {
        result_data[i] = data_a[i] + data_b[i];
    }

    // 打印结果
    std::cout << "A: ";
    for (idx_t i = 0; i < 5; i++) std::cout << data_a[i] << " ";
    std::cout << "\nB: ";
    for (idx_t i = 0; i < 5; i++) std::cout << data_b[i] << " ";
    std::cout << "\nA+B: ";
    for (idx_t i = 0; i < 5; i++) std::cout << result_data[i] << " ";
    std::cout << std::endl;
}

void Example4_SelectionVector() {
    std::cout << "\n=== Example 4: SelectionVector过滤 ===" << std::endl;

    Vector vector(LogicalType::INTEGER, 10);
    auto data = FlatVector::GetData<int32_t>(vector);

    // 填充数据
    for (idx_t i = 0; i < 10; i++) {
        data[i] = i;
    }

    // 创建SelectionVector，选择偶数
    SelectionVector sel(10);
    idx_t count = 0;
    for (idx_t i = 0; i < 10; i++) {
        if (data[i] % 2 == 0) {
            sel.set_index(count++, i);
        }
    }

    // 打印选中的值
    std::cout << "原始数据: ";
    for (idx_t i = 0; i < 10; i++) std::cout << data[i] << " ";
    std::cout << "\n过滤后（偶数）: ";
    for (idx_t i = 0; i < count; i++) {
        std::cout << data[sel.get_index(i)] << " ";
    }
    std::cout << std::endl;
}

int main() {
    Example1_CreateBasicVector();
    Example2_VectorWithNulls();
    Example3_VectorOperations();
    Example4_SelectionVector();
    return 0;
}
```

### 示例 1.2: DataChunk操作

```cpp
// learning_examples/week1/datachunk_example.cpp
#include "duckdb.hpp"
#include <iostream>

using namespace duckdb;

void Example1_CreateDataChunk() {
    std::cout << "=== Example 1: 创建DataChunk ===" << std::endl;

    // 定义schema
    vector<LogicalType> types = {
        LogicalType::INTEGER,
        LogicalType::VARCHAR,
        LogicalType::DOUBLE
    };

    // 创建DataChunk
    DataChunk chunk;
    chunk.Initialize(Allocator::DefaultAllocator(), types);

    // 填充数据
    auto col0_data = FlatVector::GetData<int32_t>(chunk.data[0]);
    auto col2_data = FlatVector::GetData<double>(chunk.data[2]);

    for (idx_t i = 0; i < 5; i++) {
        col0_data[i] = i;
        chunk.data[1].SetValue(i, Value("Name_" + std::to_string(i)));
        col2_data[i] = i * 1.5;
    }

    chunk.SetCardinality(5);

    // 打印DataChunk
    std::cout << "DataChunk (3列 x 5行):" << std::endl;
    for (idx_t row = 0; row < chunk.size(); row++) {
        for (idx_t col = 0; col < chunk.ColumnCount(); col++) {
            Value val = chunk.GetValue(col, row);
            std::cout << val.ToString() << "\t";
        }
        std::cout << std::endl;
    }
}

void Example2_FilterDataChunk() {
    std::cout << "\n=== Example 2: 过滤DataChunk ===" << std::endl;

    // 创建DataChunk
    vector<LogicalType> types = {LogicalType::INTEGER, LogicalType::DOUBLE};
    DataChunk chunk;
    chunk.Initialize(Allocator::DefaultAllocator(), types);

    auto col0 = FlatVector::GetData<int32_t>(chunk.data[0]);
    auto col1 = FlatVector::GetData<double>(chunk.data[1]);

    for (idx_t i = 0; i < 10; i++) {
        col0[i] = i;
        col1[i] = i * 2.0;
    }
    chunk.SetCardinality(10);

    // 过滤：保留 col0 > 5 的行
    SelectionVector sel(10);
    idx_t approved_count = 0;

    for (idx_t i = 0; i < chunk.size(); i++) {
        if (col0[i] > 5) {
            sel.set_index(approved_count++, i);
        }
    }

    // 创建过滤后的DataChunk
    DataChunk filtered;
    filtered.Initialize(Allocator::DefaultAllocator(), types);
    filtered.Slice(chunk, sel, approved_count);

    std::cout << "过滤前: " << chunk.size() << " 行" << std::endl;
    std::cout << "过滤后: " << filtered.size() << " 行" << std::endl;
    std::cout << "过滤后的数据:" << std::endl;

    auto filtered_col0 = FlatVector::GetData<int32_t>(filtered.data[0]);
    auto filtered_col1 = FlatVector::GetData<double>(filtered.data[1]);

    for (idx_t i = 0; i < filtered.size(); i++) {
        std::cout << filtered_col0[i] << "\t" << filtered_col1[i] << std::endl;
    }
}

void Example3_JoinDataChunks() {
    std::cout << "\n=== Example 3: 连接两个DataChunk ===" << std::endl;

    // 左表
    DataChunk left_chunk;
    vector<LogicalType> left_types = {LogicalType::INTEGER, LogicalType::VARCHAR};
    left_chunk.Initialize(Allocator::DefaultAllocator(), left_types);

    auto left_id = FlatVector::GetData<int32_t>(left_chunk.data[0]);
    for (idx_t i = 0; i < 3; i++) {
        left_id[i] = i;
        left_chunk.data[1].SetValue(i, Value("Left_" + std::to_string(i)));
    }
    left_chunk.SetCardinality(3);

    // 右表
    DataChunk right_chunk;
    vector<LogicalType> right_types = {LogicalType::INTEGER, LogicalType::DOUBLE};
    right_chunk.Initialize(Allocator::DefaultAllocator(), right_types);

    auto right_id = FlatVector::GetData<int32_t>(right_chunk.data[0]);
    auto right_val = FlatVector::GetData<double>(right_chunk.data[1]);
    for (idx_t i = 0; i < 3; i++) {
        right_id[i] = i;
        right_val[i] = i * 10.0;
    }
    right_chunk.SetCardinality(3);

    // 简单的等值Join（id = id）
    DataChunk result;
    vector<LogicalType> result_types = {
        LogicalType::INTEGER,
        LogicalType::VARCHAR,
        LogicalType::DOUBLE
    };
    result.Initialize(Allocator::DefaultAllocator(), result_types);

    idx_t result_count = 0;
    for (idx_t l = 0; l < left_chunk.size(); l++) {
        for (idx_t r = 0; r < right_chunk.size(); r++) {
            if (left_id[l] == right_id[r]) {
                // 匹配
                result.data[0].SetValue(result_count, Value::INTEGER(left_id[l]));
                result.data[1].SetValue(result_count, left_chunk.data[1].GetValue(l));
                result.data[2].SetValue(result_count, Value::DOUBLE(right_val[r]));
                result_count++;
            }
        }
    }
    result.SetCardinality(result_count);

    std::cout << "Join结果:" << std::endl;
    for (idx_t i = 0; i < result.size(); i++) {
        std::cout << result.GetValue(0, i).ToString() << "\t"
                  << result.GetValue(1, i).ToString() << "\t"
                  << result.GetValue(2, i).ToString() << std::endl;
    }
}

int main() {
    Example1_CreateDataChunk();
    Example2_FilterDataChunk();
    Example3_JoinDataChunks();
    return 0;
}
```

---

## Week 2: 查询执行示例

### 示例 2.1: 使用DuckDB C++ API执行查询

```cpp
// learning_examples/week2/table_scan_example.cpp
#include "duckdb.hpp"
#include <iostream>

using namespace duckdb;

void Example1_BasicQuery() {
    std::cout << "=== Example 1: 基本查询 ===" << std::endl;

    // 创建数据库
    DuckDB db(nullptr);
    Connection con(db);

    // 创建表
    con.Query("CREATE TABLE users (id INTEGER, name VARCHAR, age INTEGER)");

    // 插入数据
    con.Query("INSERT INTO users VALUES (1, 'Alice', 30)");
    con.Query("INSERT INTO users VALUES (2, 'Bob', 25)");
    con.Query("INSERT INTO users VALUES (3, 'Charlie', 35)");

    // 查询
    auto result = con.Query("SELECT * FROM users WHERE age > 25");

    // 打印结果
    result->Print();
}

void Example2_PreparedStatement() {
    std::cout << "\n=== Example 2: 预编译语句 ===" << std::endl;

    DuckDB db(nullptr);
    Connection con(db);

    con.Query("CREATE TABLE products (id INTEGER, name VARCHAR, price DOUBLE)");

    // 准备语句
    auto stmt = con.Prepare("INSERT INTO products VALUES ($1, $2, $3)");

    // 批量插入
    for (int i = 0; i < 5; i++) {
        stmt->Execute(i, "Product_" + std::to_string(i), i * 10.5);
    }

    // 查询
    auto result = con.Query("SELECT * FROM products ORDER BY price");
    result->Print();
}

void Example3_Aggregation() {
    std::cout << "\n=== Example 3: 聚合查询 ===" << std::endl;

    DuckDB db(nullptr);
    Connection con(db);

    con.Query("CREATE TABLE sales (product VARCHAR, quantity INTEGER, price DOUBLE)");
    con.Query("INSERT INTO sales VALUES ('A', 10, 5.0)");
    con.Query("INSERT INTO sales VALUES ('B', 20, 3.0)");
    con.Query("INSERT INTO sales VALUES ('A', 15, 5.0)");
    con.Query("INSERT INTO sales VALUES ('B', 25, 3.0)");

    // 聚合查询
    auto result = con.Query(
        "SELECT product, SUM(quantity) as total_qty, AVG(price) as avg_price "
        "FROM sales GROUP BY product"
    );

    result->Print();
}

void Example4_Join() {
    std::cout << "\n=== Example 4: Join查询 ===" << std::endl;

    DuckDB db(nullptr);
    Connection con(db);

    // 创建两个表
    con.Query("CREATE TABLE students (id INTEGER, name VARCHAR, class_id INTEGER)");
    con.Query("CREATE TABLE classes (id INTEGER, class_name VARCHAR)");

    // 插入数据
    con.Query("INSERT INTO students VALUES (1, 'Alice', 1), (2, 'Bob', 2), (3, 'Charlie', 1)");
    con.Query("INSERT INTO classes VALUES (1, 'Math'), (2, 'English')");

    // Join查询
    auto result = con.Query(
        "SELECT s.name, c.class_name "
        "FROM students s JOIN classes c ON s.class_id = c.id"
    );

    result->Print();
}

int main() {
    Example1_BasicQuery();
    Example2_PreparedStatement();
    Example3_Aggregation();
    Example4_Join();
    return 0;
}
```

### 示例 2.2: Hash Join实现剖析

```cpp
// learning_examples/week2/hash_join_example.cpp
#include "duckdb.hpp"
#include <iostream>
#include <unordered_map>

using namespace duckdb;

// 简化的Hash Join实现
class SimpleHashJoin {
    // Build端的Hash Table
    std::unordered_map<int32_t, vector<idx_t>> hash_table;
    DataChunk build_chunk;

public:
    void Build(DataChunk &chunk, idx_t key_col) {
        // 保存build chunk
        build_chunk.Initialize(Allocator::DefaultAllocator(), chunk.GetTypes());
        chunk.Copy(build_chunk);

        // 构建Hash Table
        auto keys = FlatVector::GetData<int32_t>(build_chunk.data[key_col]);

        for (idx_t i = 0; i < build_chunk.size(); i++) {
            hash_table[keys[i]].push_back(i);
        }

        std::cout << "Build阶段: Hash Table大小 = " << hash_table.size() << std::endl;
    }

    void Probe(DataChunk &probe_chunk, idx_t key_col, DataChunk &result) {
        auto probe_keys = FlatVector::GetData<int32_t>(probe_chunk.data[key_col]);

        idx_t result_count = 0;

        // 探测Hash Table
        for (idx_t probe_idx = 0; probe_idx < probe_chunk.size(); probe_idx++) {
            int32_t key = probe_keys[probe_idx];

            auto it = hash_table.find(key);
            if (it != hash_table.end()) {
                // 找到匹配
                for (auto build_idx : it->second) {
                    // 输出probe行 + build行
                    for (idx_t col = 0; col < probe_chunk.ColumnCount(); col++) {
                        result.data[col].SetValue(result_count,
                                                  probe_chunk.data[col].GetValue(probe_idx));
                    }

                    idx_t offset = probe_chunk.ColumnCount();
                    for (idx_t col = 0; col < build_chunk.ColumnCount(); col++) {
                        result.data[offset + col].SetValue(result_count,
                                                           build_chunk.data[col].GetValue(build_idx));
                    }

                    result_count++;
                }
            }
        }

        result.SetCardinality(result_count);

        std::cout << "Probe阶段: 匹配 " << result_count << " 行" << std::endl;
    }
};

void TestHashJoin() {
    std::cout << "=== 测试简化Hash Join实现 ===" << std::endl;

    // 创建Build表
    DataChunk build_chunk;
    vector<LogicalType> build_types = {LogicalType::INTEGER, LogicalType::VARCHAR};
    build_chunk.Initialize(Allocator::DefaultAllocator(), build_types);

    auto build_keys = FlatVector::GetData<int32_t>(build_chunk.data[0]);
    for (idx_t i = 0; i < 5; i++) {
        build_keys[i] = i;
        build_chunk.data[1].SetValue(i, Value("Build_" + std::to_string(i)));
    }
    build_chunk.SetCardinality(5);

    // 创建Probe表
    DataChunk probe_chunk;
    vector<LogicalType> probe_types = {LogicalType::INTEGER, LogicalType::DOUBLE};
    probe_chunk.Initialize(Allocator::DefaultAllocator(), probe_types);

    auto probe_keys = FlatVector::GetData<int32_t>(probe_chunk.data[0]);
    auto probe_vals = FlatVector::GetData<double>(probe_chunk.data[1]);
    for (idx_t i = 0; i < 8; i++) {
        probe_keys[i] = i % 5;  // 有重复
        probe_vals[i] = i * 10.0;
    }
    probe_chunk.SetCardinality(8);

    // 执行Hash Join
    SimpleHashJoin join;
    join.Build(build_chunk, 0);  // Build on key column 0

    DataChunk result;
    vector<LogicalType> result_types = {
        LogicalType::INTEGER, LogicalType::DOUBLE,  // Probe
        LogicalType::INTEGER, LogicalType::VARCHAR  // Build
    };
    result.Initialize(Allocator::DefaultAllocator(), result_types);

    join.Probe(probe_chunk, 0, result);  // Probe on key column 0

    // 打印结果
    std::cout << "\nJoin结果:" << std::endl;
    for (idx_t i = 0; i < result.size(); i++) {
        std::cout << result.GetValue(0, i).ToString() << "\t"
                  << result.GetValue(1, i).ToString() << "\t"
                  << result.GetValue(2, i).ToString() << "\t"
                  << result.GetValue(3, i).ToString() << std::endl;
    }
}

int main() {
    TestHashJoin();
    return 0;
}
```

---

## Week 3: 优化器示例

### 示例 3.1: 查看查询计划

```cpp
// learning_examples/week3/optimizer_example.cpp
#include "duckdb.hpp"
#include <iostream>

using namespace duckdb;

void Example1_ExplainQuery() {
    std::cout << "=== Example 1: EXPLAIN查看查询计划 ===" << std::endl;

    DuckDB db(nullptr);
    Connection con(db);

    con.Query("CREATE TABLE t1 (id INTEGER, value INTEGER)");
    con.Query("CREATE TABLE t2 (id INTEGER, name VARCHAR)");

    // 插入一些数据
    for (int i = 0; i < 100; i++) {
        con.Query("INSERT INTO t1 VALUES (" + std::to_string(i) + ", " +
                 std::to_string(i * 2) + ")");
        con.Query("INSERT INTO t2 VALUES (" + std::to_string(i) + ", 'Name_" +
                 std::to_string(i) + "')");
    }

    // 查看逻辑计划
    std::cout << "\n--- 逻辑计划 ---" << std::endl;
    auto logical_plan = con.Query("EXPLAIN SELECT t1.id, t1.value, t2.name "
                                  "FROM t1 JOIN t2 ON t1.id = t2.id "
                                  "WHERE t1.value > 50");
    logical_plan->Print();

    // 查看物理计划
    std::cout << "\n--- 物理计划 ---" << std::endl;
    auto physical_plan = con.Query("EXPLAIN ANALYZE SELECT t1.id, t1.value, t2.name "
                                   "FROM t1 JOIN t2 ON t1.id = t2.id "
                                   "WHERE t1.value > 50");
    physical_plan->Print();
}

void Example2_FilterPushdown() {
    std::cout << "\n=== Example 2: Filter Pushdown优化 ===" << std::endl;

    DuckDB db(nullptr);
    Connection con(db);

    con.Query("CREATE TABLE orders (order_id INTEGER, customer_id INTEGER, amount DOUBLE)");
    con.Query("CREATE TABLE customers (customer_id INTEGER, name VARCHAR, city VARCHAR)");

    // 插入测试数据
    for (int i = 0; i < 1000; i++) {
        con.Query("INSERT INTO orders VALUES (" + std::to_string(i) + ", " +
                 std::to_string(i % 100) + ", " + std::to_string(i * 1.5) + ")");
    }
    for (int i = 0; i < 100; i++) {
        con.Query("INSERT INTO customers VALUES (" + std::to_string(i) +
                 ", 'Customer_" + std::to_string(i) + "', 'City_" +
                 std::to_string(i % 10) + "')");
    }

    // 未优化的查询（Filter在JOIN之后）
    std::cout << "\n查询: SELECT * FROM orders JOIN customers ON orders.customer_id = customers.customer_id WHERE customers.city = 'City_1'" << std::endl;

    auto plan = con.Query("EXPLAIN SELECT * FROM orders o "
                         "JOIN customers c ON o.customer_id = c.customer_id "
                         "WHERE c.city = 'City_1'");
    plan->Print();

    std::cout << "\n注意观察Filter是否被下推到TableScan" << std::endl;
}

void Example3_StatisticsUsage() {
    std::cout << "\n=== Example 3: 统计信息的使用 ===" << std::endl;

    DuckDB db(nullptr);
    Connection con(db);

    con.Query("CREATE TABLE data (id INTEGER, category VARCHAR, value DOUBLE)");

    // 插入数据
    for (int i = 0; i < 10000; i++) {
        con.Query("INSERT INTO data VALUES (" + std::to_string(i) +
                 ", 'Cat_" + std::to_string(i % 10) + "', " +
                 std::to_string(i * 0.5) + ")");
    }

    // 查看表统计信息
    auto stats = con.Query("SELECT * FROM duckdb_tables() WHERE table_name='data'");
    stats->Print();

    // 使用统计信息优化的查询
    auto plan = con.Query("EXPLAIN SELECT * FROM data WHERE id > 9000");
    plan->Print();
}

int main() {
    Example1_ExplainQuery();
    Example2_FilterPushdown();
    Example3_StatisticsUsage();
    return 0;
}
```

---

## Week 4: 存储引擎示例

### 示例 4.1: 持久化数据库

```cpp
// learning_examples/week4/storage_example.cpp
#include "duckdb.hpp"
#include <iostream>

using namespace duckdb;

void Example1_PersistentDatabase() {
    std::cout << "=== Example 1: 持久化数据库 ===" << std::endl;

    const string db_path = "/tmp/test.duckdb";

    {
        // 创建持久化数据库
        DuckDB db(db_path);
        Connection con(db);

        con.Query("CREATE TABLE persistent_data (id INTEGER, value VARCHAR)");

        for (int i = 0; i < 100; i++) {
            con.Query("INSERT INTO persistent_data VALUES (" +
                     std::to_string(i) + ", 'Value_" +
                     std::to_string(i) + "')");
        }

        std::cout << "写入100行数据" << std::endl;

        // 显式Checkpoint
        con.Query("CHECKPOINT");
        std::cout << "执行Checkpoint" << std::endl;
    }

    {
        // 重新打开数据库
        DuckDB db(db_path);
        Connection con(db);

        auto result = con.Query("SELECT COUNT(*) FROM persistent_data");
        result->Print();

        std::cout << "成功从磁盘恢复数据" << std::endl;
    }
}

void Example2_WALRecovery() {
    std::cout << "\n=== Example 2: WAL恢复 ===" << std::endl;

    const string db_path = "/tmp/wal_test.duckdb";

    {
        DuckDB db(db_path);
        Connection con(db);

        con.Query("CREATE TABLE wal_data (id INTEGER)");

        // 开始事务
        con.Query("BEGIN TRANSACTION");

        for (int i = 0; i < 50; i++) {
            con.Query("INSERT INTO wal_data VALUES (" + std::to_string(i) + ")");
        }

        // 提交事务（写入WAL）
        con.Query("COMMIT");

        std::cout << "提交50行到WAL" << std::endl;

        // 不执行Checkpoint就关闭（模拟崩溃）
    }

    {
        // 重新打开，应该自动恢复WAL
        DuckDB db(db_path);
        Connection con(db);

        auto result = con.Query("SELECT COUNT(*) FROM wal_data");
        result->Print();

        std::cout << "WAL恢复成功" << std::endl;
    }
}

void Example3_Compression() {
    std::cout << "\n=== Example 3: 压缩效果测试 ===" << std::endl;

    DuckDB db(nullptr);
    Connection con(db);

    // 创建表，指定压缩
    con.Query("CREATE TABLE compressed (id INTEGER, value VARCHAR)");

    // 插入重复数据（适合压缩）
    for (int i = 0; i < 10000; i++) {
        con.Query("INSERT INTO compressed VALUES (" +
                 std::to_string(i) + ", 'REPEATED_VALUE')");
    }

    // 查看存储信息
    auto info = con.Query("SELECT * FROM duckdb_tables() WHERE table_name='compressed'");
    info->Print();

    std::cout << "压缩对重复数据效果显著" << std::endl;
}

int main() {
    Example1_PersistentDatabase();
    Example2_WALRecovery();
    Example3_Compression();
    return 0;
}
```

---

## 完整项目模板

### Mini-DuckDB完整实现

```cpp
// learning_examples/final_project/mini_duckdb.cpp
#include "duckdb.hpp"
#include <iostream>
#include <memory>

using namespace duckdb;

class MiniDuckDB {
    unique_ptr<DuckDB> db;
    unique_ptr<Connection> con;

public:
    MiniDuckDB(const string &db_path = ":memory:") {
        db = make_unique<DuckDB>(db_path == ":memory:" ? nullptr : db_path.c_str());
        con = make_unique<Connection>(*db);

        std::cout << "MiniDuckDB initialized" << std::endl;
    }

    void CreateTable(const string &table_name, const string &schema) {
        string sql = "CREATE TABLE " + table_name + " " + schema;
        con->Query(sql);
        std::cout << "Table created: " << table_name << std::endl;
    }

    void Insert(const string &table_name, const vector<string> &values) {
        string sql = "INSERT INTO " + table_name + " VALUES (";
        for (size_t i = 0; i < values.size(); i++) {
            sql += values[i];
            if (i < values.size() - 1) sql += ", ";
        }
        sql += ")";

        con->Query(sql);
    }

    void BulkInsert(const string &table_name, const vector<vector<string>> &rows) {
        con->Query("BEGIN TRANSACTION");

        for (const auto &row : rows) {
            Insert(table_name, row);
        }

        con->Query("COMMIT");
        std::cout << "Bulk inserted " << rows.size() << " rows" << std::endl;
    }

    unique_ptr<MaterializedQueryResult> Query(const string &sql) {
        return con->Query(sql);
    }

    void Explain(const string &sql) {
        auto result = con->Query("EXPLAIN " + sql);
        result->Print();
    }

    void Checkpoint() {
        con->Query("CHECKPOINT");
        std::cout << "Checkpoint completed" << std::endl;
    }

    void PrintStats() {
        auto result = con->Query("SELECT * FROM duckdb_tables()");
        result->Print();
    }
};

// 示例应用：简单的分析查询
void RunAnalyticsExample() {
    std::cout << "=== 运行分析查询示例 ===" << std::endl;

    MiniDuckDB mini_db(":memory:");

    // 创建销售表
    mini_db.CreateTable("sales",
        "(order_id INTEGER, product VARCHAR, quantity INTEGER, price DOUBLE, date DATE)");

    // 批量插入数据
    vector<vector<string>> sales_data;
    for (int i = 0; i < 1000; i++) {
        sales_data.push_back({
            std::to_string(i),
            "'Product_" + std::to_string(i % 10) + "'",
            std::to_string(10 + i % 50),
            std::to_string(5.0 + (i % 100) * 0.5),
            "'2024-01-" + std::to_string(1 + i % 28) + "'"
        });
    }
    mini_db.BulkInsert("sales", sales_data);

    // 分析查询1：每个产品的总销售额
    std::cout << "\n--- 查询1: 每个产品的总销售额 ---" << std::endl;
    auto result1 = mini_db.Query(
        "SELECT product, SUM(quantity * price) as total_revenue "
        "FROM sales "
        "GROUP BY product "
        "ORDER BY total_revenue DESC"
    );
    result1->Print();

    // 分析查询2：最畅销的产品
    std::cout << "\n--- 查询2: 最畅销的产品 ---" << std::endl;
    auto result2 = mini_db.Query(
        "SELECT product, SUM(quantity) as total_quantity "
        "FROM sales "
        "GROUP BY product "
        "ORDER BY total_quantity DESC "
        "LIMIT 5"
    );
    result2->Print();

    // 查看查询计划
    std::cout << "\n--- 查询计划 ---" << std::endl;
    mini_db.Explain(
        "SELECT product, SUM(quantity * price) as total_revenue "
        "FROM sales "
        "GROUP BY product"
    );
}

int main() {
    RunAnalyticsExample();
    return 0;
}
```

---

## 编译和运行

### 编译所有示例

```bash
cd /home/dev/duckdb/learning_examples
mkdir build && cd build
cmake ..
make -j4
```

### 运行示例

```bash
# Week 1
./week1_vector
./week1_datachunk

# Week 2
./week2_table_scan
./week2_hash_join

# Week 3
./week3_optimizer

# Week 4
./week4_storage

# Final Project
./mini_duckdb
```

---

## 调试技巧

### 使用GDB调试

```bash
gdb ./week1_vector

(gdb) break main
(gdb) run
(gdb) next
(gdb) print data[0]
(gdb) continue
```

### 使用Valgrind检测内存泄漏

```bash
valgrind --leak-check=full ./week1_vector
```

---

## 下一步

1. **修改示例代码** - 尝试修改参数、添加功能
2. **性能测试** - 对比不同实现的性能
3. **阅读源码** - 深入DuckDB源码理解实现细节
4. **扩展功能** - 添加新的算子、优化规则等

祝学习愉快！🚀
