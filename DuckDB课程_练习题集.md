,# DuckDB 30天学习课程 - 练习题与答案

本文档提供每周的练习题和详细答案，帮助巩固所学知识。

---

## 第一周练习题

### Week 1 Day 3: Vector和DataChunk

**练习1：Vector切片操作**

实现一个函数，给定一个Vector和一个索引范围，返回切片后的Vector。

```cpp
// 题目
Vector SliceVector(Vector &input, idx_t start, idx_t end);

// 示例
Vector input = ...; // 包含 [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
Vector result = SliceVector(input, 3, 7);
// result 应包含 [3, 4, 5, 6]
```

<details>
<summary>点击查看答案</summary>

```cpp
Vector SliceVector(Vector &input, idx_t start, idx_t end) {
    D_ASSERT(start < end && end <= input.GetVectorSize());

    idx_t count = end - start;
    Vector result(input.GetType(), count);

    // 创建SelectionVector
    SelectionVector sel(count);
    for (idx_t i = 0; i < count; i++) {
        sel.set_index(i, start + i);
    }

    // 使用Slice方法
    result.Slice(input, sel, count);

    return result;
}

// 测试
void TestSliceVector() {
    Vector input(LogicalType::INTEGER, 10);
    auto data = FlatVector::GetData<int32_t>(input);

    for (idx_t i = 0; i < 10; i++) {
        data[i] = i;
    }

    Vector result = SliceVector(input, 3, 7);
    auto result_data = FlatVector::GetData<int32_t>(result);

    for (idx_t i = 0; i < 4; i++) {
        assert(result_data[i] == 3 + i);
    }

    std::cout << "SliceVector测试通过!" << std::endl;
}
```

</details>

---

**练习2：DataChunk合并**

实现一个函数，将两个具有相同schema的DataChunk合并为一个。

```cpp
// 题目
DataChunk MergeDataChunks(DataChunk &chunk1, DataChunk &chunk2);
```

<details>
<summary>点击查看答案</summary>

```cpp
DataChunk MergeDataChunks(DataChunk &chunk1, DataChunk &chunk2) {
    D_ASSERT(chunk1.ColumnCount() == chunk2.ColumnCount());

    idx_t total_rows = chunk1.size() + chunk2.size();
    D_ASSERT(total_rows <= STANDARD_VECTOR_SIZE);

    DataChunk result;
    result.Initialize(Allocator::DefaultAllocator(), chunk1.GetTypes());

    // 拷贝chunk1的数据
    for (idx_t col = 0; col < chunk1.ColumnCount(); col++) {
        for (idx_t row = 0; row < chunk1.size(); row++) {
            result.SetValue(col, row, chunk1.GetValue(col, row));
        }
    }

    // 拷贝chunk2的数据
    idx_t offset = chunk1.size();
    for (idx_t col = 0; col < chunk2.ColumnCount(); col++) {
        for (idx_t row = 0; row < chunk2.size(); row++) {
            result.SetValue(col, offset + row, chunk2.GetValue(col, row));
        }
    }

    result.SetCardinality(total_rows);
    return result;
}

// 测试
void TestMergeDataChunks() {
    vector<LogicalType> types = {LogicalType::INTEGER, LogicalType::DOUBLE};

    DataChunk chunk1, chunk2;
    chunk1.Initialize(Allocator::DefaultAllocator(), types);
    chunk2.Initialize(Allocator::DefaultAllocator(), types);

    // 填充chunk1
    auto c1_col0 = FlatVector::GetData<int32_t>(chunk1.data[0]);
    auto c1_col1 = FlatVector::GetData<double>(chunk1.data[1]);
    for (idx_t i = 0; i < 5; i++) {
        c1_col0[i] = i;
        c1_col1[i] = i * 1.5;
    }
    chunk1.SetCardinality(5);

    // 填充chunk2
    auto c2_col0 = FlatVector::GetData<int32_t>(chunk2.data[0]);
    auto c2_col1 = FlatVector::GetData<double>(chunk2.data[1]);
    for (idx_t i = 0; i < 3; i++) {
        c2_col0[i] = 10 + i;
        c2_col1[i] = (10 + i) * 1.5;
    }
    chunk2.SetCardinality(3);

    // 合并
    DataChunk result = MergeDataChunks(chunk1, chunk2);

    assert(result.size() == 8);
    std::cout << "MergeDataChunks测试通过! 合并后行数: " << result.size() << std::endl;
}
```

</details>

---

### Week 1 Day 6: 表达式系统

**练习3：实现简单表达式求值器**

实现一个简单的表达式求值器，支持 `a + b` 和 `a * b`。

```cpp
// 题目
class SimpleExpressionEvaluator {
public:
    Value Evaluate(const string &expr,
                   const unordered_map<string, Value> &variables);
};

// 示例
unordered_map<string, Value> vars = {
    {"a", Value::INTEGER(10)},
    {"b", Value::INTEGER(5)}
};
auto result = evaluator.Evaluate("a + b", vars);
// result 应该是 15
```

<details>
<summary>点击查看答案</summary>

```cpp
class SimpleExpressionEvaluator {
public:
    Value Evaluate(const string &expr,
                   const unordered_map<string, Value> &variables) {
        // 简化版：只支持 "a op b" 形式

        // 分割表达式
        size_t op_pos = string::npos;
        char op = 0;

        if ((op_pos = expr.find('+')) != string::npos) {
            op = '+';
        } else if ((op_pos = expr.find('*')) != string::npos) {
            op = '*';
        } else {
            // 单个变量
            return variables.at(Trim(expr));
        }

        // 提取操作数
        string left_str = Trim(expr.substr(0, op_pos));
        string right_str = Trim(expr.substr(op_pos + 1));

        Value left = variables.at(left_str);
        Value right = variables.at(right_str);

        // 计算
        if (op == '+') {
            return Value::INTEGER(left.GetValue<int32_t>() + right.GetValue<int32_t>());
        } else if (op == '*') {
            return Value::INTEGER(left.GetValue<int32_t>() * right.GetValue<int32_t>());
        }

        throw Exception("Unsupported operation");
    }

private:
    string Trim(const string &s) {
        size_t start = s.find_first_not_of(" \t");
        size_t end = s.find_last_not_of(" \t");
        return (start == string::npos) ? "" : s.substr(start, end - start + 1);
    }
};

// 测试
void TestSimpleExpressionEvaluator() {
    SimpleExpressionEvaluator evaluator;

    unordered_map<string, Value> vars = {
        {"a", Value::INTEGER(10)},
        {"b", Value::INTEGER(5)},
        {"c", Value::INTEGER(2)}
    };

    auto result1 = evaluator.Evaluate("a + b", vars);
    assert(result1.GetValue<int32_t>() == 15);

    auto result2 = evaluator.Evaluate("a * c", vars);
    assert(result2.GetValue<int32_t>() == 20);

    auto result3 = evaluator.Evaluate("a", vars);
    assert(result3.GetValue<int32_t>() == 10);

    std::cout << "SimpleExpressionEvaluator测试通过!" << std::endl;
}
```

</details>

---

## 第二周练习题

### Week 2 Day 11: TableScan和Filter

**练习4：实现带过滤的表扫描**

实现一个简单的表扫描算子，支持谓词过滤。

```cpp
// 题目
class SimpleFilteredScan {
    DataChunk data;
    std::function<bool(idx_t)> predicate;
    idx_t current_pos = 0;

public:
    SimpleFilteredScan(DataChunk &data, std::function<bool(idx_t)> pred);
    bool GetNext(DataChunk &result);
};
```

<details>
<summary>点击查看答案</summary>

```cpp
class SimpleFilteredScan {
    DataChunk data;
    std::function<bool(idx_t)> predicate;
    idx_t current_pos = 0;

public:
    SimpleFilteredScan(DataChunk &input_data, std::function<bool(idx_t)> pred)
        : predicate(pred), current_pos(0) {
        // 拷贝输入数据
        data.Initialize(Allocator::DefaultAllocator(), input_data.GetTypes());
        input_data.Copy(data);
    }

    bool GetNext(DataChunk &result) {
        if (current_pos >= data.size()) {
            return false;
        }

        result.Initialize(Allocator::DefaultAllocator(), data.GetTypes());

        SelectionVector sel(STANDARD_VECTOR_SIZE);
        idx_t approved_count = 0;

        // 扫描并应用谓词
        idx_t scan_count = 0;
        while (current_pos < data.size() && scan_count < STANDARD_VECTOR_SIZE) {
            if (predicate(current_pos)) {
                sel.set_index(approved_count++, current_pos);
            }
            current_pos++;
            scan_count++;
        }

        if (approved_count == 0) {
            return GetNext(result);  // 递归，继续扫描
        }

        // 切片数据
        for (idx_t col = 0; col < data.ColumnCount(); col++) {
            result.data[col].Slice(data.data[col], sel, approved_count);
        }
        result.SetCardinality(approved_count);

        return true;
    }
};

// 测试
void TestSimpleFilteredScan() {
    // 创建测试数据
    DataChunk input;
    vector<LogicalType> types = {LogicalType::INTEGER, LogicalType::DOUBLE};
    input.Initialize(Allocator::DefaultAllocator(), types);

    auto col0 = FlatVector::GetData<int32_t>(input.data[0]);
    auto col1 = FlatVector::GetData<double>(input.data[1]);

    for (idx_t i = 0; i < 100; i++) {
        col0[i] = i;
        col1[i] = i * 2.0;
    }
    input.SetCardinality(100);

    // 创建扫描器，过滤 col0 > 50
    SimpleFilteredScan scanner(input, [&](idx_t row) {
        auto data = FlatVector::GetData<int32_t>(input.data[0]);
        return data[row] > 50;
    });

    // 扫描结果
    DataChunk result;
    idx_t total_rows = 0;

    while (scanner.GetNext(result)) {
        total_rows += result.size();
    }

    assert(total_rows == 49);  // 51-99 共49行
    std::cout << "SimpleFilteredScan测试通过! 过滤后行数: " << total_rows << std::endl;
}
```

</details>

---

### Week 2 Day 13: Aggregation

**练习5：实现简单的SUM聚合**

实现一个简单的SUM聚合函数。

```cpp
// 题目
class SimpleSumAggregator {
public:
    void Update(Vector &input, idx_t count);
    Value Finalize();
};
```

<details>
<summary>点击查看答案</summary>

```cpp
class SimpleSumAggregator {
    int64_t sum = 0;
    bool has_value = false;

public:
    void Update(Vector &input, idx_t count) {
        auto data = FlatVector::GetData<int32_t>(input);
        auto &validity = FlatVector::Validity(input);

        for (idx_t i = 0; i < count; i++) {
            if (validity.RowIsValid(i)) {
                sum += data[i];
                has_value = true;
            }
        }
    }

    Value Finalize() {
        if (!has_value) {
            return Value(LogicalType::BIGINT);  // NULL
        }
        return Value::BIGINT(sum);
    }

    void Reset() {
        sum = 0;
        has_value = false;
    }
};

// 测试
void TestSimpleSumAggregator() {
    SimpleSumAggregator aggregator;

    Vector input(LogicalType::INTEGER, 10);
    auto data = FlatVector::GetData<int32_t>(input);

    for (idx_t i = 0; i < 10; i++) {
        data[i] = i + 1;  // 1, 2, 3, ..., 10
    }

    aggregator.Update(input, 10);

    Value result = aggregator.Finalize();

    assert(result.GetValue<int64_t>() == 55);  // 1+2+...+10 = 55
    std::cout << "SimpleSumAggregator测试通过! SUM = " << result.GetValue<int64_t>() << std::endl;
}
```

</details>

---

## 第三周练习题

### Week 3 Day 16: Filter Pushdown

**练习6：实现简单的Filter Pushdown**

给定一个查询计划树，将Filter下推到TableScan。

```cpp
// 题目
// 输入: Filter -> Projection -> TableScan
// 输出: Projection -> TableScan (with filter)

struct QueryPlan {
    enum Type { SCAN, FILTER, PROJECTION };
    Type type;
    unique_ptr<QueryPlan> child;
    // ...
};

QueryPlan OptimizeFilterPushdown(QueryPlan plan);
```

<details>
<summary>点击查看答案</summary>

```cpp
struct QueryPlan {
    enum Type { SCAN, FILTER, PROJECTION };
    Type type;
    string filter_expr;  // 简化：用字符串表示
    unique_ptr<QueryPlan> child;

    QueryPlan(Type t) : type(t) {}
};

QueryPlan OptimizeFilterPushdown(unique_ptr<QueryPlan> plan) {
    if (!plan) return nullptr;

    // 情况1: Filter -> Projection -> Scan
    if (plan->type == QueryPlan::FILTER &&
        plan->child && plan->child->type == QueryPlan::PROJECTION &&
        plan->child->child && plan->child->child->type == QueryPlan::SCAN) {

        // 提取Filter表达式
        string filter = plan->filter_expr;

        // 重组：Projection -> Scan (with filter)
        auto new_plan = std::move(plan->child);  // Projection
        new_plan->child->filter_expr = filter;    // 将filter下推到Scan

        return new_plan;
    }

    // 递归优化子树
    if (plan->child) {
        plan->child = OptimizeFilterPushdown(std::move(plan->child));
    }

    return plan;
}

// 测试
void TestFilterPushdown() {
    // 构造计划: Filter -> Projection -> Scan
    auto scan = make_unique<QueryPlan>(QueryPlan::SCAN);

    auto proj = make_unique<QueryPlan>(QueryPlan::PROJECTION);
    proj->child = std::move(scan);

    auto filter = make_unique<QueryPlan>(QueryPlan::FILTER);
    filter->filter_expr = "age > 25";
    filter->child = std::move(proj);

    // 优化
    auto optimized = OptimizeFilterPushdown(std::move(filter));

    // 验证
    assert(optimized->type == QueryPlan::PROJECTION);
    assert(optimized->child->type == QueryPlan::SCAN);
    assert(optimized->child->filter_expr == "age > 25");

    std::cout << "Filter Pushdown优化测试通过!" << std::endl;
}
```

</details>

---

### Week 3 Day 17: Join Order

**练习7：3表Join顺序选择**

给定3个表和它们的大小，选择最优的Join顺序。

```cpp
// 题目
struct Table {
    string name;
    idx_t row_count;
};

vector<Table> OptimizeJoinOrder(vector<Table> tables);
// 返回优化后的Join顺序（最小的表先Join）
```

<details>
<summary>点击查看答案</summary>

```cpp
struct Table {
    string name;
    idx_t row_count;
};

vector<Table> OptimizeJoinOrder(vector<Table> tables) {
    // 简单策略：按大小排序，小表优先
    std::sort(tables.begin(), tables.end(),
              [](const Table &a, const Table &b) {
                  return a.row_count < b.row_count;
              });
    return tables;
}

// 更复杂的成本估计
double EstimateJoinCost(const vector<Table> &order) {
    if (order.size() < 2) return 0;

    double cost = 0;
    idx_t current_size = order[0].row_count;

    for (size_t i = 1; i < order.size(); i++) {
        // 成本 = 左表大小 + 右表大小 + 输出大小
        // 简化：假设输出大小 = 左表大小 * 右表大小 / 较大表大小
        idx_t right_size = order[i].row_count;
        idx_t output_size = (current_size * right_size) /
                           std::max(current_size, right_size);

        cost += current_size + right_size + output_size;
        current_size = output_size;
    }

    return cost;
}

// 测试
void TestJoinOrderOptimization() {
    vector<Table> tables = {
        {"large_table", 1000000},
        {"small_table", 100},
        {"medium_table", 10000}
    };

    auto optimized = OptimizeJoinOrder(tables);

    assert(optimized[0].name == "small_table");
    assert(optimized[1].name == "medium_table");
    assert(optimized[2].name == "large_table");

    double cost = EstimateJoinCost(optimized);

    std::cout << "Join Order优化测试通过!" << std::endl;
    std::cout << "优化后顺序: ";
    for (const auto &t : optimized) {
        std::cout << t.name << " ";
    }
    std::cout << "\n估计成本: " << cost << std::endl;
}
```

</details>

---

## 第四周练习题

### Week 4 Day 24: 压缩算法

**练习8：实现RLE压缩**

实现Run-Length Encoding压缩和解压。

```cpp
// 题目
class RLECompressor {
public:
    vector<pair<int32_t, idx_t>> Compress(const vector<int32_t> &data);
    vector<int32_t> Decompress(const vector<pair<int32_t, idx_t>> &compressed);
};
```

<details>
<summary>点击查看答案</summary>

```cpp
class RLECompressor {
public:
    // 压缩：返回 (值, 重复次数) 对
    vector<pair<int32_t, idx_t>> Compress(const vector<int32_t> &data) {
        if (data.empty()) return {};

        vector<pair<int32_t, idx_t>> result;

        int32_t current_value = data[0];
        idx_t run_length = 1;

        for (size_t i = 1; i < data.size(); i++) {
            if (data[i] == current_value) {
                run_length++;
            } else {
                result.push_back({current_value, run_length});
                current_value = data[i];
                run_length = 1;
            }
        }

        // 最后一个run
        result.push_back({current_value, run_length});

        return result;
    }

    // 解压
    vector<int32_t> Decompress(const vector<pair<int32_t, idx_t>> &compressed) {
        vector<int32_t> result;

        for (const auto &run : compressed) {
            for (idx_t i = 0; i < run.second; i++) {
                result.push_back(run.first);
            }
        }

        return result;
    }

    // 计算压缩率
    double CompressionRatio(const vector<int32_t> &original,
                           const vector<pair<int32_t, idx_t>> &compressed) {
        size_t original_size = original.size() * sizeof(int32_t);
        size_t compressed_size = compressed.size() * sizeof(pair<int32_t, idx_t>);
        return (double)original_size / compressed_size;
    }
};

// 测试
void TestRLECompressor() {
    RLECompressor compressor;

    // 测试数据：有大量重复
    vector<int32_t> data = {
        1, 1, 1, 1, 1,
        2, 2, 2,
        3, 3, 3, 3, 3, 3, 3,
        1, 1
    };

    auto compressed = compressor.Compress(data);

    std::cout << "原始数据大小: " << data.size() << std::endl;
    std::cout << "压缩后runs数量: " << compressed.size() << std::endl;

    assert(compressed.size() == 4);
    assert(compressed[0] == make_pair(1, (idx_t)5));
    assert(compressed[1] == make_pair(2, (idx_t)3));
    assert(compressed[2] == make_pair(3, (idx_t)7));
    assert(compressed[3] == make_pair(1, (idx_t)2));

    // 解压并验证
    auto decompressed = compressor.Decompress(compressed);
    assert(decompressed == data);

    double ratio = compressor.CompressionRatio(data, compressed);
    std::cout << "压缩比: " << ratio << "x" << std::endl;
    std::cout << "RLE压缩测试通过!" << std::endl;
}
```

</details>

---

## 综合练习

### 最终项目：实现TPC-H Query 1

实现TPC-H基准测试的Query 1（简化版）。

```sql
SELECT
    l_returnflag,
    l_linestatus,
    SUM(l_quantity) as sum_qty,
    AVG(l_extendedprice) as avg_price,
    COUNT(*) as count_order
FROM
    lineitem
WHERE
    l_shipdate <= '1998-12-01'
GROUP BY
    l_returnflag,
    l_linestatus
ORDER BY
    l_returnflag,
    l_linestatus;
```

<details>
<summary>点击查看实现思路</summary>

```cpp
void ExecuteTPCH_Q1() {
    DuckDB db(nullptr);
    Connection con(db);

    // 1. 创建表
    con.Query(
        "CREATE TABLE lineitem ("
        "l_orderkey INTEGER, "
        "l_partkey INTEGER, "
        "l_suppkey INTEGER, "
        "l_linenumber INTEGER, "
        "l_quantity DECIMAL(15,2), "
        "l_extendedprice DECIMAL(15,2), "
        "l_discount DECIMAL(15,2), "
        "l_tax DECIMAL(15,2), "
        "l_returnflag CHAR(1), "
        "l_linestatus CHAR(1), "
        "l_shipdate DATE, "
        "l_commitdate DATE, "
        "l_receiptdate DATE, "
        "l_shipinstruct CHAR(25), "
        "l_shipmode CHAR(10), "
        "l_comment VARCHAR(44))"
    );

    // 2. 加载数据（示例）
    con.Query("INSERT INTO lineitem VALUES "
              "(1, 1, 1, 1, 10.00, 100.00, 0.05, 0.02, 'N', 'O', '1998-01-15', '1998-01-20', '1998-01-25', 'DELIVER', 'TRUCK', 'test'), "
              "(2, 2, 2, 1, 20.00, 200.00, 0.03, 0.01, 'R', 'F', '1998-11-15', '1998-11-20', '1998-11-25', 'DELIVER', 'SHIP', 'test')");

    // 3. 执行Query 1
    auto result = con.Query(
        "SELECT "
        "l_returnflag, "
        "l_linestatus, "
        "SUM(l_quantity) as sum_qty, "
        "AVG(l_extendedprice) as avg_price, "
        "COUNT(*) as count_order "
        "FROM lineitem "
        "WHERE l_shipdate <= DATE '1998-12-01' "
        "GROUP BY l_returnflag, l_linestatus "
        "ORDER BY l_returnflag, l_linestatus"
    );

    // 4. 打印结果
    result->Print();

    // 5. 查看执行计划
    auto plan = con.Query("EXPLAIN ANALYZE " + /* Query 1 SQL */);
    plan->Print();
}
```

**关键点：**
1. Filter Pushdown: `l_shipdate <= '1998-12-01'` 应该下推到TableScan
2. Hash Aggregation: GROUP BY使用Hash聚合
3. 排序: 最后结果需要排序

**性能优化建议：**
- 在 `l_shipdate` 上创建索引
- 使用列式存储减少I/O
- 并行执行聚合

</details>

---

## 学习检验清单

完成以下练习后，你应该能够：

- ✅ 熟练操作Vector和DataChunk
- ✅ 实现基本的查询算子
- ✅ 应用查询优化技术
- ✅ 实现压缩算法
- ✅ 理解TPC-H查询的执行过程

## 下一步

1. **扩展练习** - 尝试实现更多TPC-H查询
2. **性能对比** - 对比优化前后的性能
3. **贡献代码** - 向DuckDB提交PR

祝学习愉快！🎉
