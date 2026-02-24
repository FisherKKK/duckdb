# DuckDB 高级课程：扩展系统开发

> 本课程深入讲解DuckDB扩展系统的开发、编译、分发和最佳实践。

---

## 第一部分：扩展系统架构

### 1.1 扩展系统概述

DuckDB扩展是独立于主代码库的库，提供额外的功能。扩展可以：
- 添加新的标量函数和聚合函数
- 实现新的表函数（如文件扫描器）
- 添加新的复制格式
- 集成外部数据源
- 实现自定义类型

#### 扩展类型分类

```
DuckDB扩展
├── In-tree（树内扩展）
│   ├── 位于主仓库 extension/ 目录
│   ├── 与DuckDB深度集成
│   ├── 由DuckDB团队维护
│   └── 示例：parquet, json, icu
│
└── Out-of-tree（树外扩展）
    ├── 独立仓库
    ├── 保持主仓库精简
    ├── 许可证灵活性
    │
    ├── DuckDB托管OOTE
    │   ├── 通过DuckDB CI分发
    │   ├── 使用DuckDB签名密钥
    │   └── 示例：sqlite_scanner, postgres_scanner
    │
    └── 外部OOTE
        ├── 独立CI/CD
        ├── 维护者负责分发
        └── 社区贡献
```

### 1.2 扩展加载机制

```
┌─────────────────────────────────────────┐
│   扩展加载流程                           │
├─────────────────────────────────────────┤
│                                         │
│  1. 扩展发现                             │
│     ├── 搜索 ~/.duckdb/extensions/      │
│     ├── 搜索 build/extension/           │
│     └── 从GitHub自动下载                │
│                                         │
│  2. 签名验证（可选）                     │
│     ├── 验证扩展签名                    │
│     ├── 检查版本兼容性                  │
│     └── 安全检查                        │
│                                         │
│  3. 动态链接                             │
│     ├── dlopen (.so/.dylib/.dll)       │
│     ├── 符号解析                        │
│     └── 依赖解析                        │
│                                         │
│  4. 初始化                               │
│     ├── 调用 DUCKDB_EXTENSION_ENTRY     │
│     ├── 注册函数/类型                   │
│     └── 执行初始化逻辑                  │
│                                         │
└─────────────────────────────────────────┘
```

### 1.3 扩展入口点

```cpp
// src/include/duckdb_extension.h

#define DUCKDB_EXTENSION_NAME my_extension
#define DUCKDB_EXTENSION_ENTRYPOINT \
    duckdb_extension_init(duckdb_connection connection, \
                          duckdb_extension_info info, \
                          duckdb_extension_access *access)

// 实现入口点
extern "C" {
duckdb_extension_init(duckdb_connection connection,
                      duckdb_extension_info info,
                      duckdb_extension_access *access) {
    // 获取数据库实例
    auto db = (duckdb::DatabaseInstance*)connection;

    // 创建扩展对象
    duckdb::MyExtension extension;

    // 加载扩展
    extension.Load(db);
}
}
```

---

## 第二部分：创建自定义扩展

### 2.1 项目结构

```
my_extension/
├── CMakeLists.txt          # CMake构建配置
├── vcpkg.json              # 依赖管理（可选）
├── src/
│   ├── my_extension.cpp    # 扩展实现
│   └── include/
│       └── my_extension.hpp
├── test/
│   └── sql/
│       └── my_extension.test
└── README.md
```

### 2.2 最小化扩展示例

#### CMakeLists.txt

```cmake
# 扩展名称
set(DUCKDB_MY_EXTENSION_NAME "my_extension")

# 依赖DuckDB
find_package(duckdb REQUIRED)

# 添加源文件
add_library(${DUCKDB_MY_EXTENSION_NAME} STATIC
    src/my_extension.cpp
)

# 包含头文件
target_include_directories(${DUCKDB_MY_EXTENSION_NAME} PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/src/include
)

# 链接DuckDB
target_link_libraries(${DUCKDB_MY_EXTENSION_NAME} PUBLIC
    duckdb_static
)

# 创建扩展目标
duckdb_extension(${DUCKDB_MY_EXTENSION_NAME})
```

#### my_extension.hpp

```cpp
#pragma once

#include "duckdb.hpp"

namespace duckdb {

class MyExtension : public Extension {
public:
    // 加载扩展
    void Load(ExtensionLoader &db) override;

    // 扩展名称
    std::string Name() override {
        return "my_extension";
    }

    // 扩展版本
    std::string Version() const override {
        return "1.0.0";
    }
};

} // namespace duckdb
```

#### my_extension.cpp

```cpp
#include "my_extension.hpp"
#include "duckdb/function/scalar_function.hpp"
#include "duckdb/common/vector_operations/vector_operations.hpp"

namespace duckdb {

// 标量函数实现
static void AddOneFunction(DataChunk &args,
                           ExpressionState &state,
                           Vector &result) {
    auto &input = args.data[0];
    auto input_data = FlatVector::GetData<int32_t>(input);
    auto result_data = FlatVector::GetData<int32_t>(result);

    // 向量化操作
    for (idx_t i = 0; i < args.size(); i++) {
        result_data[i] = input_data[i] + 1;
    }

    // 处理NULL值
    if (input.validity.AllValid()) {
        result.validity.SetAllValid(args.size());
    } else {
        result.validity.Copy(input.validity, args.size());
    }
}

// 注册函数
void MyExtension::Load(ExtensionLoader &db) {
    // 注册标量函数 "add_one"
    auto add_one = ScalarFunction("add_one",
                                  {LogicalType::INTEGER},
                                  LogicalType::INTEGER,
                                  AddOneFunction);

    db.RegisterFunction(add_one);

    // 也可以注册其他类型
    // - 聚合函数: db.RegisterAggregateFunction(...)
    // - 表函数: db.RegisterTableFunction(...)
    // - 复制函数: db.RegisterCopyFunction(...)
    // - pragma: db.RegisterPragma(...)
}

} // namespace duckdb

// 扩展入口点
extern "C" {
DUCKDB_EXTENSION_API void my_extension_init(duckdb::DatabaseInstance *db) {
    duckdb::MyExtension extension;
    extension.Load(db);
}
}
```

### 2.3 使用扩展模板

DuckDB提供了官方的扩展模板，可以快速开始：

```bash
# 克隆模板
git clone https://github.com/duckdb/extension-template.git my_extension
cd my_extension

# 自定义扩展名称
# 1. 替换 CMakeLists.txt 中的 EXTENSION_NAME
# 2. 替换源文件中的类名和函数名
# 3. 添加自定义功能

# 构建扩展
mkdir build && cd build
cmake ..
make
```

---

## 第三部分：高级扩展功能

### 3.1 标量函数开发

#### 带状态的标量函数

```cpp
// 函数状态（用于缓存/优化）
struct MyFunctionData : public FunctionData {
    int multiplier;

public:
    MyFunctionData(int multiplier) : multiplier(multiplier) {}

    unique_ptr<FunctionData> Copy() const override {
        return make_uniq<MyFunctionData>(multiplier);
    }

    bool Equals(const FunctionData &other) const override {
        auto &other_data = other.Cast<MyFunctionData>();
        return multiplier == other_data.multiplier;
    }
};

// Bind函数：在查询规划时调用
static unique_ptr<FunctionData> MyBindFunction(
    ClientContext &context,
    ScalarFunction &bound_function,
    vector<unique_ptr<Expression>> &arguments) {

    // 从常量参数提取值
    if (arguments[1]->IsFoldable()) {
        Value multiplier_val = ExpressionExecutor::EvaluateScalar(
            context, *arguments[1]);
        int multiplier = multiplier_val.GetValue<int>();
        return make_uniq<MyFunctionData>(multiplier);
    }

    // 默认值
    return make_uniq<MyFunctionData>(1);
}

// 主函数实现
static void MyFunction(DataChunk &args,
                       ExpressionState &state,
                       Vector &result) {
    auto &func_data = (MyFunctionData &)*state.expr.function_data;

    auto &input = args.data[0];
    auto input_data = FlatVector::GetData<int32_t>(input);
    auto result_data = FlatVector::GetData<int32_t>(result);

    // 使用缓存的乘数
    for (idx_t i = 0; i < args.size(); i++) {
        result_data[i] = input_data[i] * func_data.multiplier;
    }

    result.validity.Copy(input.validity, args.size());
}

// 注册带bind函数的标量函数
void RegisterMyFunction(ExtensionLoader &db) {
    ScalarFunctionSet func_set("my_multiply");

    ScalarFunction func(
        {LogicalType::INTEGER, LogicalType::INTEGER},  // 两个参数
        LogicalType::INTEGER,
        MyFunction,
        MyBindFunction  // bind函数
    );

    func_set.AddFunction(func);
    db.RegisterFunction(func_set);
}
```

#### 可变参数函数

```cpp
// 处理可变参数
static unique_ptr<FunctionData> VarArgsBind(
    ClientContext &context,
    ScalarFunction &bound_function,
    vector<unique_ptr<Expression>> &arguments) {

    // 验证所有参数类型相同
    if (!arguments.empty()) {
        auto first_type = arguments[0]->return_type;
        for (size_t i = 1; i < arguments.size(); i++) {
            if (arguments[i]->return_type != first_type) {
                throw BinderException(
                    "All arguments must have the same type");
            }
        }
    }

    return nullptr;
}

static void VarArgsFunction(DataChunk &args,
                            ExpressionState &state,
                            Vector &result) {
    // 处理可变数量参数
    // args.ColumnCount() 是实际参数数量
    auto &result_data = FlatVector::GetData<int32_t>(result);

    // 示例：计算所有参数的和
    for (idx_t i = 0; i < args.size(); i++) {
        int32_t sum = 0;
        for (idx_t col = 0; col < args.ColumnCount(); col++) {
            auto &input_vec = args.data[col];
            auto input_data = FlatVector::GetData<int32_t>(input_vec);
            sum += input_data[i];
        }
        result_data[i] = sum;
    }
}

// 注册可变参数函数
void RegisterVarArgsFunction(ExtensionLoader &db) {
    ScalarFunctionSet func_set("sum_all");

    // 使用 LogicalType::ANY 表示接受任何类型
    ScalarFunction func(
        {LogicalType::ANY},  // 可变参数：一个或多个
        LogicalType::INTEGER,
        VarArgsFunction,
        VarArgsBind
    );

    func.varargs = LogicalType::ANY;  // 标记为可变参数
    func_set.AddFunction(func);
    db.RegisterFunction(func_set);
}
```

### 3.2 聚合函数开发

```cpp
// 聚合状态
struct ProductState {
    double product;
    idx_t count;
};

// 聚合函数
struct ProductAggregateFunction {
    // 创建状态
    static unique_ptr<FunctionData> Bind(ClientContext &context,
                                         AggregateFunction &function,
                                         vector<unique_ptr<Expression>> &arguments) {
        return nullptr;
    }

    // 初始化状态
    static void Initialize(DataChunk &args,
                           AggregateFunctionInput &bind_data,
                           Vector &state_vector,
                           idx_t count) {
        auto states = (ProductState**)FlatVector::GetData<data_ptr_t>(state_vector);

        for (idx_t i = 0; i < count; i++) {
            states[i] = new ProductState{1.0, 0};
        }
    }

    // 更新状态（逐批）
    static void Update(DataChunk &args,
                       AggregateFunctionInput &bind_data,
                       Vector &state_vector,
                       Vector &input_vector) {

        auto states = AggregateExecutor::GetStateData<ProductState>(state_vector);
        auto input_data = FlatVector::GetData<double>(input_vector);

        UnifiedVectorFormat input_format;
        input_vector.ToUnifiedFormat(args.size(), input_format);

        for (idx_t i = 0; i < args.size(); i++) {
            auto idx = input_format.sel->get_index(i);

            if (!input_format.validity.RowIsValid(idx)) {
                continue;  // 跳过NULL
            }

            states[i]->product *= input_data[idx];
            states[i]->count++;
        }
    }

    // 合并状态（并行执行）
    static void Combine(Vector &state_vector,
                        Vector &combined_state) {

        auto states = AggregateExecutor::GetStateData<ProductState>(state_vector);
        auto combined = AggregateExecutor::GetStateData<ProductState>(combined_state)[0];

        for (idx_t i = 0; i < state_vector.size(); i++) {
            combined->product *= states[i]->product;
            combined->count += states[i]->count;
        }
    }

    // 最终化（计算结果）
    static void Finalize(Vector &state_vector,
                         AggregateFunctionInput &bind_data,
                         Vector &result) {

        auto states = AggregateExecutor::GetStateData<ProductState>(state_vector);
        auto result_data = FlatVector::GetData<double>(result);

        for (idx_t i = 0; i < state_vector.size(); i++) {
            if (states[i]->count == 0) {
                // 没有非NULL值，返回NULL
                result.validity.SetInvalid(i);
            } else {
                result_data[i] = states[i]->product;
            }

            delete states[i];
        }
    }

    // 注册聚合函数
    static void Register(ExtensionLoader &db) {
        AggregateFunctionSet aggregates("product");

        AggregateFunction product_aggr(
            {LogicalType::DOUBLE},  // 输入类型
            LogicalType::DOUBLE,    // 返回类型
            nullptr,                // 简单聚合不需要state_size
            nullptr,
            Initialize,
            Update,
            Combine,
            Finalize
        );

        aggregates.AddFunction(product_aggr);
        db.RegisterFunction(aggregates);
    }
};
```

### 3.3 表函数开发

表函数返回一个完整的表（多行多列），常用于数据导入。

```cpp
// 表函数：生成斐波那契数列
struct FibonacciFunctionData : public TableFunctionData {
    idx_t count;
    idx_t current;
    idx_t next;
};

// Bind函数
static unique_ptr<TableFunctionData> FibonacciBind(
    ClientContext &context,
    TableFunctionBindInput &input,
    vector<LogicalType> &return_types,
    vector<string> &names) {

    auto result = make_uniq<FibonacciFunctionData>();

    // 参数：生成数量
    if (input.inputs.size() != 1) {
        throw BinderException("fibonacci requires exactly one argument");
    }

    Value count_val = input.inputs[0];
    if (!count_val.type().IsIntegral()) {
        throw BinderException("fibonacci argument must be an integer");
    }

    result->count = count_val.GetValue<idx_t>();

    // 定义返回类型
    return_types = {LogicalType::BIGINT, LogicalType::BIGINT};
    names = {"index", "value"};

    return result;
}

// Init函数
static unique_ptr<GlobalTableFunctionState> FibonacciInit(
    ClientContext &context,
    TableFunctionInitInput &input) {

    auto &bind_data = (FibonacciFunctionData &)*input.bind_data;
    bind_data.current = 0;
    bind_data.next = 1;

    return make_uniq<GlobalTableFunctionState>();
}

// 执行函数
static void FibonacciExecute(ClientContext &context,
                             TableFunctionInput &input,
                             DataChunk &output) {

    auto &bind_data = (FibonacciFunctionData &)*input.bind_data;

    idx_t count = 0;
    const idx_t batch_size = STANDARD_VECTOR_SIZE;

    while (count < batch_size && bind_data.current < bind_data.count) {
        // 填充输出
        auto index_data = FlatVector::GetData<int64_t>(output.data[0]);
        auto value_data = FlatVector::GetData<int64_t>(output.data[1]);

        index_data[count] = bind_data.current;
        value_data[count] = bind_data.next;

        // 计算下一个斐波那契数
        idx_t new_next = bind_data.current + bind_data.next;
        bind_data.current = bind_data.next;
        bind_data.next = new_next;

        count++;
    }

    output.SetCardinality(count);
}

// 注册表函数
void RegisterFibonacciFunction(ExtensionLoader &db) {
    TableFunction fib_func("fibonacci",
                           {LogicalType::BIGINT},  // 参数：count
                           FibonacciExecute,
                           FibonacciBind,
                           FibonacciInit);

    db.RegisterFunction(fib_func);
}
```

### 3.4 复制函数开发

复制函数用于导入/导出数据。

```cpp
// CSV导入示例
static unique_ptr<FunctionData> CSVReadBind(
    ClientContext &context,
    CopyFunctionBindInput &input,
    vector<LogicalType> &return_types,
    vector<string> &names) {

    // 从options读取参数
    auto file_path = input.info.file_path;

    // 自动检测列
    // ...

    return nullptr;
}

// 全局初始化
static unique_ptr<GlobalTableFunctionState> CSVReadInitGlobal(
    ClientContext &context,
    TableFunctionInitInput &input) {

    // 打开文件
    // ...

    return make_uniq<GlobalTableFunctionState>();
}

// 执行读取
static void CSVReadFunction(ClientContext &context,
                            TableFunctionInput &input,
                            DataChunk &output) {

    // 读取一批数据
    // 解析CSV
    // 填充output
}

// 注册复制函数
void RegisterCSVFunction(ExtensionLoader &db) {
    CopyFunction csv_func("csv");

    // 读取选项
    csv_func.read_parameter_defaults["delimiter"] = Value(",");
    csv_func.read_parameter_defaults["header"] = Value::BOOLEAN(true);
    csv_func.read_parameter_defaults["quote"] = Value("\"");

    // 绑定函数
    csv_func.bind = CSVReadBind;
    csv_func.init_global = CSVReadInitGlobal;
    csv_func.function = CSVReadFunction;

    db.RegisterFunction(csv_func);
}
```

---

## 第四部分：构建与分发

### 4.1 本地构建

```bash
# 方法1：使用DUCKDB_EXTENSIONS变量
DUCKDB_EXTENSIONS='my_extension' make

# 方法2：使用BUILD_EXTENSIONS变量
cmake -B build -DBUILD_EXTENSIONS=my_extension
cmake --build build

# 方法3：使用扩展配置文件
cat >> extension/extension_config_local.cmake <<EOF
duckdb_extension_load(my_extension)
EOF
make
```

### 4.2 跨平台构建

#### Linux

```bash
# 构建Linux扩展
mkdir build && cd build
cmake .. \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_EXTENSIONS=my_extension \
    -DCMAKE_INSTALL_PREFIX=./install

cmake --build .

# 生成扩展文件
# build/extension/my_extension/my_extension.duckdb_extension
```

#### macOS

```bash
# 通用二进制（x86_64 + arm64）
cmake .. \
    -DOSX_BUILD_UNIVERSAL=ON \
    -DBUILD_EXTENSIONS=my_extension

cmake --build .

# 特定架构
cmake .. \
    -DOSX_BUILD_ARCH=arm64 \
    -DBUILD_EXTENSIONS=my_extension
```

#### Windows

```bash
# 使用Visual Studio
cmake .. -G "Visual Studio 17 2022" \
         -DBUILD_EXTENSIONS=my_extension

cmake --build . --config Release

# 输出：
# build/release/my_extension.duckdb_extension
```

### 4.3 版本控制与签名

```cmake
# 设置扩展版本
set(EXTENSION_VERSION "1.0.0")

# 设置扩展名称
set(DUCKDB_EXTENSION_NAME "my_extension")

# 生成版本信息
configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR/src/version.hpp.in
    ${CMAKE_CURRENT_BINARY_DIR}/version.hpp
)
```

```cpp
// version.hpp.in
namespace my_extension {
    constexpr const char *VERSION = "@EXTENSION_VERSION@";
    constexpr const char *EXTENSION_NAME = "@DUCKDB_EXTENSION_NAME@";
}
```

### 4.4 扩展签名

```bash
# 使用DuckDB的签名工具

# 1. 生成签名密钥（仅首次）
python3 scripts/extension_signing.py generate_key

# 2. 签名扩展
python3 scripts/extension_signing.py sign \
    --extension build/extension/my_extension.duckdb_extension \
    --output build/extension/my_extension.duckdb_extension.signed

# 3. 验证签名
python3 scripts/extension_signing.py verify \
    --extension build/extension/my_extension.duckdb_extension.signed
```

### 4.5 发布扩展

#### GitHub Releases

```bash
# 1. 创建Git tag
git tag v1.0.0
git push origin v1.0.0

# 2. 构建所有平台的扩展
# Linux
BUILD_EXTENSIONS=my_extension make
# macOS
OSX_BUILD_ARCH=arm64 BUILD_EXTENSIONS=my_extension make
OSX_BUILD_ARCH=x86_64 BUILD_EXTENSIONS=my_extension make
# Windows
cmake -G "Visual Studio 17 2022" -DBUILD_EXTENSIONS=my_extension

# 3. 上传到GitHub Releases
gh release create v1.0.0 \
    build/linux_amd64/my_extension.duckdb_extension \
    build/osx_arm64/my_extension.duckdb_extension \
    build/osx_amd64/my_extension.duckdb_extension \
    build/windows_amd64/my_extension.duckdb_extension
```

#### DuckDB扩展仓库

对于DuckDB托管的扩展，需要：

1. Fork [extension-template](https://github.com/duckdb/extension-template)
2. 实现扩展功能
3. 添加测试
4. 提交PR到DuckDB

扩展会自动包含在DuckDB CI中，并在每次发布时构建。

---

## 第五部分：测试与调试

### 5.1 单元测试

```cpp
// test/unit/test_my_extension.cpp

#include "duckdb/common/vector_operations/vector_operations.hpp"
#include "duckdb/main/cached_query.hpp"
#include "my_extension.hpp"

using namespace duckdb;

// 测试标量函数
TEST_CASE("Test add_one function") {
    DuckDB db(nullptr);
    Connection con(db);

    // 注册扩展
    MyExtension extension;
    extension.Load(*db.instance);

    // 执行查询
    auto result = con.Query("SELECT add_one(42)");
    REQUIRE(result->GetValue(0, 0).GetValue<int32_t>() == 43);
}

// 测试边界情况
TEST_CASE("Test add_one NULL") {
    DuckDB db(nullptr);
    Connection con(db);

    MyExtension extension;
    extension.Load(*db.instance);

    auto result = con.Query("SELECT add_one(NULL::INTEGER)");
    REQUIRE(result->GetValue(0, 0).IsNull() == true);
}
```

### 5.2 SQL逻辑测试

```sql
-- test/sql/my_extension/test_add_one.test

# Test basic functionality
query I
SELECT add_one(1);
----
2

# Test NULL handling
query I
SELECT add_one(NULL::INTEGER);
----
NULL

# Test vectorized execution
query I
SELECT add_one(value) FROM range(10) t(value);
----
1
2
3
4
5
6
7
8
9
10

statement ok
CREATE TABLE numbers AS SELECT range(1000) AS value;

# Test on table data
query I
SELECT COUNT(*) FROM numbers WHERE add_one(value) > 500;
----
499
```

### 5.3 调试技巧

```bash
# 1. 使用GDB调试扩展
gdb --args ./build/debug/duckdb

# 在gdb中
(gdb) break duckdb::MyFunction
(gdb) run
(gdb) call my_function

# 2. 打印调试信息
#define MY_DEBUG_LOG(msg) \
    std::cerr << "[MY_EXT] " << msg << std::endl;

// 在函数中使用
MY_DEBUG_LOG("Processing chunk of size " << args.size());

# 3. Valgrind检测内存错误
valgrind --leak-check=full \
         ./build/debug/duckdb < test.sql

# 4. 使用DuckDB的日志系统
PRAGMA enable_logging=true;
PRAGMA log_file_path='duckdb.log';
```

---

## 第六部分：最佳实践

### 6.1 性能优化

```cpp
// ❌ 差：逐行处理
static void BadFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    auto &input = args.data[0];

    for (idx_t i = 0; i < args.size(); i++) {
        // 每次只处理一行
        result.SetValue(i, input.GetValue(i));
    }
}

// ✅ 好：向量化处理
static void GoodFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    auto &input = args.data[0];
    auto input_data = FlatVector::GetData<int32_t>(input);
    auto result_data = FlatVector::GetData<int32_t>(result);

    // 批量处理
    for (idx_t i = 0; i < args.size(); i++) {
        result_data[i] = input_data[i] + 1;
    }

    // 批量处理NULL
    result.validity.Copy(input.validity, args.size());
}
```

### 6.2 错误处理

```cpp
// 使用DuckDB的异常系统
#include "duckdb/common/exception.hpp"

static void ValidateInput(Value &val) {
    if (val.IsNull()) {
        throw InvalidInputException("Input value cannot be NULL");
    }

    if (!val.type().IsIntegral()) {
        throw TypeMismatchException(val.type(),
                                    LogicalType::INTEGER,
                                    "Input must be an integer");
    }
}

// 在函数中使用
static void SafeFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    if (args.ColumnCount() != 1) {
        throw InvalidInputException("Function expects exactly 1 argument");
    }

    // ... 处理逻辑
}
```

### 6.3 文档编写

```cpp
// 为函数添加文档注释

/**
 * @brief Calculates the factorial of a number
 *
 * This function computes the factorial n! = n * (n-1) * ... * 1
 * For n = 0, returns 1 (by definition)
 *
 * @param n Non-negative integer
 * @return The factorial of n
 * @throws InvalidInputException if n is negative or too large
 *
 * Example:
 * > SELECT factorial(5);
 * 120
 */
static void FactorialFunction(...);

// 注册时添加描述
auto func = ScalarFunction("factorial", ...);
func.description = "Calculate the factorial of a number";
func.examples = {
    {"SELECT factorial(5)", "120"},
    {"SELECT factorial(0)", "1"}
};
```

### 6.4 扩展检查清单

- [ ] **功能完整**
  - 实现所有目标功能
  - 处理边界情况
  - 正确的NULL处理

- [ ] **性能优化**
  - 向量化实现
  - 避免不必要的拷贝
  - 使用高效的数据结构

- [ ] **错误处理**
  - 清晰的错误消息
  - 验证输入参数
  - 处理异常情况

- [ ] **测试覆盖**
  - 单元测试
  - SQL逻辑测试
  - 边界测试

- [ ] **文档**
  - README说明
  - API文档
  - 使用示例

- [ ] **跨平台**
  - Linux测试
  - macOS测试
  - Windows测试

- [ ] **版本兼容**
  - 测试多个DuckDB版本
  - 处理API变化

---

## 第七部分：实际案例

### 7.1 案例1：IP地址函数扩展

```cpp
// IP地址解析和处理

struct IPFunctionData {
    bool is_ipv6;
};

static void ParseIPFunction(DataChunk &args,
                            ExpressionState &state,
                            Vector &result) {
    auto &input = args.data[0];

    // 解析IP地址字符串
    UnaryExecutor::Execute<string_t, string_t>(
        input, result, args.size(),
        [&](string_t ip_str) {
            // 验证IP格式
            if (!IsValidIP(ip_str)) {
                throw InvalidInputException("Invalid IP address: %s",
                                            ip_str.GetString());
            }

            // 返回标准化格式
            return NormalizeIP(ip_str);
        });
}

// 注册IP相关函数
void RegisterIPFunctions(ExtensionLoader &db) {
    ScalarFunctionSet funcs("ip");

    // ip_parse: 解析IP地址
    funcs.AddFunction(ScalarFunction(
        {LogicalType::VARCHAR},
        LogicalType::VARCHAR,
        ParseIPFunction
    ));

    // ip_to_int: IP转整数
    funcs.AddFunction(ScalarFunction(
        {LogicalType::VARCHAR},
        LogicalType::BIGINT,
        IPToIntFunction
    ));

    // ip_is_ipv6: 检查是否IPv6
    funcs.AddFunction(ScalarFunction(
        {LogicalType::VARCHAR},
        LogicalType::BOOLEAN,
        IsIPv6Function
    ));

    db.RegisterFunction(funcs);
}
```

### 7.2 案例2：机器学习扩展

```cpp
// 简单的线性回归聚合函数

struct LinearRegressionState {
    double sum_x;
    double sum_y;
    double sum_xy;
    double sum_x2;
    idx_t count;
};

static void LRUpdate(DataChunk &args,
                     AggregateFunctionInput &bind_data,
                     Vector &state_vector,
                     Vector &x,
                     Vector &y) {

    auto states = AggregateExecutor::GetStateData<LinearRegressionState>(state_vector);
    auto x_data = FlatVector::GetData<double>(x);
    auto y_data = FlatVector::GetData<double>(y);

    for (idx_t i = 0; i < args.size(); i++) {
        if (x.validity.RowIsValid(i) && y.validity.RowIsValid(i)) {
            states[i]->sum_x += x_data[i];
            states[i]->sum_y += y_data[i];
            states[i]->sum_xy += x_data[i] * y_data[i];
            states[i]->sum_x2 += x_data[i] * x_data[i];
            states[i]->count++;
        }
    }
}

static void LRFinalize(Vector &state_vector,
                       AggregateFunctionInput &bind_data,
                       Vector &result) {

    auto states = AggregateExecutor::GetStateData<LinearRegressionState>(state_vector);

    // 返回STRUCT { slope DOUBLE, intercept DOUBLE }
    for (idx_t i = 0; i < state_vector.size(); i++) {
        auto &state = states[i];

        if (state->count < 2) {
            result.validity.SetInvalid(i);
            continue;
        }

        // 计算斜率和截距
        double n = state->count;
        double slope = (n * state->sum_xy - state->sum_x * state->sum_y) /
                       (n * state->sum_x2 - state->sum_x * state->sum_x);
        double intercept = (state->sum_y - slope * state->sum_x) / n;

        // 构造STRUCT结果
        // ...
    }
}

// 注册线性回归函数
void RegisterLinearRegression(ExtensionLoader &db) {
    // ...
}
```

---

## 学习资源

**官方资源：**
- [DuckDB Extension Template](https://github.com/duckdb/extension-template)
- [DuckDB Extensions Documentation](https://duckdb.org/docs/extensions/)
- [Extension README](./extension/README.md)

**示例扩展：**
- [sqlite_scanner](https://github.com/duckdb/sqlite_scanner)
- [postgres_scanner](https://github.com/duckdb/postgres_scanner)
- [spatial](https://github.com/duckdb/duckdb_spatial)

**相关文档：**
- [DuckDB 30天学习课程](./DuckDB_30天学习课程.md)
- [DuckDB高级课程_CMake编译与优化技巧](./DuckDB高级课程_CMake编译与优化技巧.md)
- [DuckDB学习速查表](./DuckDB学习速查表.md)

---

**下一步：**
1. 使用extension-template创建你的第一个扩展
2. 实现一个简单的标量函数
3. 添加单元测试和SQL测试
4. 尝试构建跨平台版本
5. 发布到GitHub供他人使用

---

最后更新：2026-01-23
