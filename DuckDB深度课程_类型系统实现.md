# DuckDB 深度课程：类型系统实现

> 本课程深入讲解DuckDB的类型系统设计，包括LogicalType与PhysicalType的关系、各种类型的内存布局、类型转换机制和特殊类型处理。

---

## 课程概览

### 学习目标

- 理解LogicalType和PhysicalType的设计理念
- 掌握各种类型的内存布局和存储优化
- 学习类型转换的实现机制
- 理解DECIMAL、VARCHAR、复合类型的特殊处理
- 掌握类型比较和排序规则
- 能够实现自定义类型

### 前置知识

- C++模板和union类型
- 内存管理和堆分配
- SQL类型系统
- 浮点数表示

---

## 第一部分：类型系统架构

### 1.1 LogicalType vs PhysicalType

```cpp
// src/include/duckdb/common/types.hpp

// ==================== LogicalType ====================
// 逻辑类型：SQL层面的类型表示

class LogicalType {
public:
    // 类型ID
    LogicalTypeId id_;

    // 物理类型（存储方式）
    PhysicalType physical_type_;

    // 类型信息（精度、小数位等）
    shared_ptr<ExtraTypeInfo> type_info_;

    LogicalType(LogicalTypeId id = LogicalTypeId::INVALID)
        : id_(id), physical_type_(PhysicalType::INVALID) {

        // 设置默认物理类型
        UpdatePhysicalType();
    }

    // ==================== 类型信息 ====================

    // 获取类型ID
    LogicalTypeId id() const { return id_; }

    // 获取物理类型
    PhysicalType InternalType() const { return physical_type_; }

    // 获取类型名称
    string ToString() const;

    // ==================== 类型属性 ====================

    // 是否为数值类型
    bool IsNumeric() const;

    // 是否为整数类型
    bool IsIntegral() const;

    // 是否为定长类型
    bool IsFixedSize() const;

    // 获取类型大小
    idx_t GetFixedSize() const;

    // ==================== 特殊类型属性 ====================

    // DECIMAL精度
    uint8_t GetWidth() const;
    uint8_t GetScale() const;

    // VARCHAR宽度
    uint32_t GetVARCHARSize() const;

    // ENUM基数
    idx_t GetEnumCount() const;

    // ==================== 复合类型 ====================

    // LIST子类型
    LogicalType &GetChildType() const;

    // STRUCT字段
    ChildTypeMapping &GetChildTypes() const;

private:
    void UpdatePhysicalType() {
        switch (id_) {
        case LogicalTypeId::BOOLEAN:
            physical_type_ = PhysicalType::BOOL;
            break;
        case LogicalTypeId::TINYINT:
            physical_type_ = PhysicalType::INT8;
            break;
        case LogicalTypeId::SMALLINT:
            physical_type_ = PhysicalType::INT16;
            break;
        case LogicalTypeId::INTEGER:
            physical_type_ = PhysicalType::INT32;
            break;
        case LogicalTypeId::BIGINT:
            physical_type_ = PhysicalType::INT64;
            break;
        case LogicalTypeId::HUGEINT:
            physical_type_ = PhysicalType::INT128;
            break;
        case LogicalTypeId::VARCHAR:
            physical_type_ = PhysicalType::VARCHAR;
            break;
        case LogicalTypeId::DECIMAL:
            // DECIMAL的物理类型取决于精度
            physical_type_ = GetDecimalPhysicalType();
            break;
        case LogicalTypeId::LIST:
            physical_type_ = PhysicalType::LIST;
            break;
        case LogicalTypeId::STRUCT:
            physical_type_ = PhysicalType::STRUCT;
            break;
        default:
            break;
        }
    }

    static PhysicalType GetDecimalPhysicalType() {
        // 根据精度选择物理类型
        if (width <= 4) return PhysicalType::INT16;
        if (width <= 9) return PhysicalType::INT32;
        if (width <= 18) return PhysicalType::INT64;
        return PhysicalType::INT128;
    }
};

// ==================== PhysicalType ====================
// 物理类型：内存存储方式

enum class PhysicalType : uint8_t {
    BOOL,       // 1 bit
    INT8,       // 1 byte
    INT16,      // 2 bytes
    INT32,      // 4 bytes
    INT64,      // 8 bytes
    UINT8,
    UINT16,
    UINT32,
    UINT64,
    INT128,     // 16 bytes
    FLOAT,      // 4 bytes
    DOUBLE,     // 8 bytes
    VARCHAR,    // 变长
    STRUCT,     // 复合类型
    LIST,       // 列表类型
    ARRAY       // 数组类型
};
```

### 1.2 类型层次结构

```
┌─────────────────────────────────────────────────────────┐
│              DuckDB 类型层次结构                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  LogicalType (SQL语义)                                │
│  ├── 布尔类型                                          │
│  │   └── BOOLEAN                                       │
│  ├── 数值类型                                          │
│  │   ├── 整数：TINYINT, SMALLINT, INTEGER, BIGINT, HUGEINT │
│  │   └── 浮点：FLOAT, DOUBLE                            │
│  ├── 精确数值                                          │
│  │   └── DECIMAL(precision, scale)                       │
│  ├── 字符串类型                                        │
│  │   ├── VARCHAR                                        │
│  │   ├── BLOB                                           │
│  │   └── ENUM                                           │
│  ├── 日期时间类型                                        │
│  │   ├── DATE                                           │
│  │   ├── TIME                                           │
│  │   ├── TIMESTAMP                                      │
│  │   └── INTERVAL                                       │
│  ├── 复合类型                                          │
│  │   ├── LIST(child_type)                               │
│  │   ├── STRUCT(field_types)                             │
│  │   └── MAP                                            │
│  └── 特殊类型                                          │
│      ├── UUID                                           │
│      ├── JSON                                           │
│      └── VARIANT (动态类型)                             │
│                                                         │
│  PhysicalType (存储方式)                                 │
│  ├── 固定大小                                          │
│  │   ├── BOOL (1 bit)                                   │
│  │   ├── INT8-INT64, INT128 (1-16 bytes)               │
│  │   ├── FLOAT (4 bytes), DOUBLE (8 bytes)              │
│  │   └── DATE (4 bytes), TIMESTAMP (8 bytes)            │
│  └── 变长类型                                          │
│      ├── VARCHAR (string_t结构)                          │
│      ├── STRUCT (NestedValueInfo)                         │
│      └── LIST (NestedValueInfo)                          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 第二部分：基本类型实现

### 2.1 整数类型

```cpp
// 整数类型的内存布局

// ==================== 值存储 ====================
union Value {
    // 有符号整数
    int8_t tinyint;      // TINYINT: 1 byte, -128 to 127
    int16_t smallint;    // SMALLINT: 2 bytes, -32,768 to 32,767
    int32_t integer;      // INTEGER: 4 bytes, -2^31 to 2^31-1
    int64_t bigint;       // BIGINT: 8 bytes, -2^63 to 2^63-1
    hugeint_t hugeint;   // HUGEINT: 16 bytes, ±2^127

    // 无符号整数
    uint8_t utinyint;
    uint16_t usmallint;
    uint32_t uinteger;
    uint64_t ubigint;
    // ...

    value_;
};

// ==================== 获取值 ====================
template <class T>
static inline T GetValue(Value &value) {
    // 类型安全访问
    return *reinterpret_cast<T *>(&value.value_);
}

// 使用示例
Value int_val(LogicalType::INTEGER);
int_val.value_.integer = 42;

int32_t value = GetValue<int32_t>(int_val);
```

#### 整数类型范围

| 类型 | 存储大小 | 范围 | PhysicalType |
|------|---------|------|-------------|
| TINYINT | 1 byte | -128 to 127 | INT8 |
| SMALLINT | 2 bytes | -32,768 to 32,767 | INT16 |
| INTEGER | 4 bytes | -2,147,483,648 to 2,147,483,647 | INT32 |
| BIGINT | 8 bytes | ±9,223,372,036,854,775,808 | INT64 |
| HUGEINT | 16 bytes | ±170,141,183,460,469,231,731,687,303,715,884,105,728 | INT128 |

### 2.2 浮点类型

```cpp
// 浮点类型的内存布局

union Value {
    float float_;      // FLOAT: IEEE 754 单精度 (4 bytes)
    double double_;    // DOUBLE: IEEE 754 双精度 (8 bytes)
};

// ==================== 浮点特殊值 ====================

// NaN (Not a Number)
Value CreateNaN() {
    Value result(LogicalType::DOUBLE);
    result.value_.double_ = std::numeric_limits<double>::quiet_NaN();
    return result;
}

// Infinity
Value CreateInfinity(bool is_negative = false) {
    Value result(LogicalType::DOUBLE);
    if (is_negative) {
        result.value_.double_ = -std::numeric_limits<double>::infinity();
    } else {
        result.value_.double_ = std::numeric_limits<double>::infinity();
    }
    return result;
}

// ==================== 浮点比较 ====================

// 近似相等（处理精度问题）
bool ApproxEqual(double a, double b, double epsilon = 1e-9) {
    return std::abs(a - b) < epsilon;
}

// Value类中的浮点比较
bool Value::operator==(const Value &rhs) const {
    if (type_.id() == LogicalTypeId::FLOAT ||
        type_.id() == LogicalTypeId::DOUBLE) {
        // 浮点数使用近似相等
        return ApproxEqual(value_.double_, rhs.value_.double_);
    }
    // 其他类型精确比较
    return value_ == rhs.value_;
}
```

### 2.3 BOOLEAN类型

```cpp
// BOOLEAN类型实现

class ValidityMask {
public:
    // 使用位图存储有效性
    // 每个bit表示一行是否有效

    validity_t *validity_mask;

    // 检查行是否有效
    inline bool RowIsValid(idx_t row_idx) const {
        if (!validity_mask) {
            return true;  // 无掩码 = 全部有效
        }

        idx_t entry_idx = row_idx / BITS_PER_VALUE;
        idx_t idx_in_entry = row_idx % BITS_PER_VALUE;

        auto entry = validity_mask[entry_idx];
        return entry & (validity_t(1) << idx_in_entry);
    }

    // 设置行无效（NULL）
    inline void SetInvalid(idx_t row_idx) {
        D_ASSERT(validity_mask);
        idx_t entry_idx = row_idx / BITS_PER_VALUE;
        idx_t idx_in_entry = row_idx % BITS_PER_VALUE;

        auto &entry = validity_mask[entry_idx];
        entry &= ~(validity_t(1) << idx_in_entry);
    }

private:
    static constexpr const idx_t BITS_PER_VALUE = 64;
};

// BOOLEAN值的存储
union Value {
    bool boolean;  // 1 bit (存储在value_中)
};

// 获取BOOLEAN值
bool Value::GetBoolean() const {
    D_ASSERT(type_.id() == LogicalTypeId::BOOLEAN);
    return value_.boolean;
}
```

---

## 第三部分：VARCHAR类型

### 3.1 string_t结构

```cpp
// src/include/duckdb/common/types string.hpp

// ==================== string_t 结构 ====================
// 优化小字符串的存储

class string_t {
public:
    // 小字符串内联阈值
    static constexpr idx_t INLINE_LENGTH = 12;

    // 前缀长度
    static constexpr idx_t PREFIX_LENGTH = 4;

    union {
        // 小字符串：<= 12 字节
        struct {
            uint32_t length;     // 字符串长度
            char inlined[12];     // 内联存储
        } inlined;

        // 大字符串：> 12 字节
        struct {
            uint32_t length;     // 字符串长度
            char prefix[4];      // 前缀（用于快速比较）
            char *ptr;          // 指向堆内存的指针
        } pointer;
    } value;

    // ==================== 构造函数 ====================

    // 从const char*构造
    static inline string_t Dupe(const char *data, uint32_t len) {
        string_t result;
        result.SetLength(len);

        if (len <= INLINE_LENGTH) {
            // 小字符串：内联存储
            memcpy(result.value.inlined.inlined, data, len);
        } else {
            // 大字符串：堆分配
            auto ptr = new char[len];
            memcpy(ptr, data, len);
            result.value.pointer.ptr = ptr;

            // 复制前缀
            memcpy(result.value.pointer.prefix,
                   data,
                   PREFIX_LENGTH);
        }

        return result;
    }

    // ==================== 访问方法 ====================

    // 获取字符串指针
    const char *GetData() const {
        if (GetLength() <= INLINE_LENGTH) {
            return value.inlined.inlined;
        } else {
            return value.pointer.ptr;
        }
    }

    // 获取长度
    uint32_t GetLength() const {
        return value.inlined.length;
    }

    // 设置长度
    void SetLength(uint32_t len) {
        value.inlined.length = len;
    }

    // ==================== 比较优化 ====================

    // 快速比较：先比较长度和前缀
    static bool Equals(const string_t &a, const string_t &b) {
        if (a.GetLength() != b.GetLength()) {
            return false;
        }

        if (a.GetLength() <= INLINE_LENGTH) {
            // 小字符串：直接比较
            return memcmp(a.value.inlined.inlined,
                        b.value.inlined.inlined,
                        a.GetLength()) == 0;
        } else {
            // 大字符串：先比较前缀
            if (memcmp(a.value.pointer.prefix,
                        b.value.pointer.prefix,
                        PREFIX_LENGTH) != 0) {
                return false;
            }
            // 前缀相同，比较完整字符串
            return memcmp(a.value.pointer.ptr,
                        b.value.pointer.ptr,
                        a.GetLength()) == 0;
        }
    }

    // ==================== 哈希 ====================

    uint64_t Hash() const {
        // 使用前缀计算快速哈希
        uint64_t hash = 0;
        const char *data = GetData();
        uint32_t len = GetLength();

        for (uint32_t i = 0; i < len && i < PREFIX_LENGTH; i++) {
            hash = hash * 31 + data[i];
        }

        return hash;
    }
};
```

### 3.2 VARCHAR内存布局

```
┌─────────────────────────────────────────────────────────┐
│           VARCHAR 内存布局                             │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  小字符串（<= 12 字节）：                             │
│  ┌──────────────────────────────────┐                   │
│  │ length (4 bytes)              │                   │
│  │   ┌────────────────────┐       │                   │
│  │   │ inlined[12]       │       │                   │
│  │   │ "hello world!"    │       │                   │
│  │   └────────────────────┘       │                   │
│  │                                 │                   │
│  │ 总计：16 bytes                    │                   │
│  └──────────────────────────────────┘                   │
│                                                         │
│  大字符串（> 12 字节）：                              │
│  ┌──────────────────────────────────┐                   │
│  │ length (4 bytes)              │                   │
│  │   ┌────────────┐               │                   │
│  │   │ prefix[4]  │               │                   │
│  │   │ "hell"     │               │                   │
│  │   └────────────┘               │                   │
│  │   ┌────────────┐               │                   │
│  │   │ ptr (8)    │               │                   │
│  │   │ ────────→  Heap        │                   │
│  │   │            "hello world..." │                   │
│  │   └────────────┘               │                   │
│  │                                 │                   │
│  │ 总计：16 bytes + 堆分配          │                   │
│  └──────────────────────────────────┘                   │
│                                                         │
│  优势：                                                 │
│  • 小字符串无堆分配                                     │
│  • 前缀加速比较                                         │
│  • 缓存友好（16字节对齐）                            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 第四部分：DECIMAL类型

### 4.1 DECIMAL类型定义

```cpp
// DECIMAL类型信息

struct DecimalTypeInfo : public ExtraTypeInfo {
    uint8_t width;   // 总位数 (1-38)
    uint8_t scale;   // 小数位数 (0-width)

    DecimalTypeInfo(uint8_t width, uint8_t scale)
        : width(width), scale(scale) {
        D_ASSERT(width >= 1 && width <= 38);
        D_ASSERT(scale <= width);
    }
};

// DECIMAL值存储
union Value {
    // 小DECIMAL：直接存储
    int16_t smallint;     // width <= 4
    int32_t integer;       // width <= 9
    int64_t bigint;        // width <= 18

    // 大DECIMAL：HUGEINT存储
    hugeint_t hugeint;      // width <= 38
};
```

### 4.2 DECIMAL存储策略

```
┌─────────────────────────────────────────────────────────┐
│          DECIMAL 存储策略                           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  精度与存储类型映射：                                   │
│                                                         │
│  width <= 4     → INT16   (2 bytes)                 │
│    范围：-999.9999 to 999.9999                        │
│                                                         │
│  width <= 9     → INT32   (4 bytes)                 │
│    范围：-99,999,999.9 to 99,999,999.9               │
│                                                         │
│  width <= 18    → INT64   (8 bytes)                 │
│    范围：-9,999,999,999,999,999.99 to ...             │
│                                                         │
│  width <= 38    → INT128  (16 bytes)                │
│    范围：最大精度                                     │
│                                                         │
│  示例：DECIMAL(10, 2)                                 │
│  ┌──────────────────────────────────┐                   │
│  │ width = 10, scale = 2          │                   │
│  │                                 │                   │
│  │ 存储方式：INT32                  │                   │
│  │ 实际存储值 = value * 10^2        │                   │
│  │                                 │                   │
│  │ 输入：12345.67                  │                   │
│  │ 存储：1234567                   │                   │
│  │ 输出：12345.67                  │                   │
│  └──────────────────────────────────┘                   │
│                                                         │
│  DECIMAL运算：                                         │
│  • 加减法：对齐小数位后运算                           │
│  • 乘法：精度相加，小数位相加                         │
│  • 除法：需要特殊处理舍入                               │
│  • 舍入模式：Half-even (银行家舍入）                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 4.3 DECIMAL运算实现

```cpp
// DECIMAL运算

class DecimalOperations {
public:
    // 加法
    static Value Add(const Value &left, const Value &right,
                   uint8_t width, uint8_t scale) {
        // 1. 对齐小数位
        hugeint_t l = GetAsHugeint(left);
        hugeint_t r = GetAsHugeint(right);
        hugeint_t result = l + r;

        // 2. 检查溢出
        if (!CheckOverflow(result, width)) {
            throw OutOfRangeException("DECIMAL overflow in addition");
        }

        // 3. 返回结果
        return CreateDecimal(result, width, scale);
    }

    // 乘法
    static Value Multiply(const Value &left, const Value &right,
                        uint8_t width, uint8_t scale) {
        hugeint_t l = GetAsHugeint(left);
        hugeint_t r = GetAsHugeint(right);

        // 使用128位乘法
        uhugeint_t result = l * r;

        // 缩放调整
        result /= 10^scale;

        return CreateDecimal(result, width, scale);
    }

    // 除法
    static Value Divide(const Value &left, const Value &right,
                       uint8_t width, uint8_t scale) {
        hugeint_t l = GetAsHugeint(left);
        hugeint_t r = GetAsHugeint(right);

        if (r == 0) {
            throw Exception("Division by zero");
        }

        // 扩展精度以避免舍入误差
        hugeint_t scaled_l = l * 10^scale;
        hugeint_t result = scaled_l / r;

        // 舍入处理
        RoundHalfEven(result, scale);

        return CreateDecimal(result, width, scale);
    }

private:
    // 舍入到最近的偶数（银行家舍入）
    static void RoundHalfEven(hugeint_t &value, uint8_t scale) {
        hugeint_t remainder = value % 10;
        value /= 10;

        if (remainder > 5) {
            value++;
        } else if (remainder == 5) {
            // 舍入到偶数
            if (value % 2 != 0) {
                value++;
            }
        }
    }
};
```

---

## 第五部分：复合类型

### 5.1 LIST类型

```cpp
// LIST类型实现

struct list_entry_t {
    uint64_t offset;  // 在ListData中的偏移量
    uint64_t length;  // 子元素数量
};

class LogicalType {
public:
    // 创建LIST类型
    static LogicalType LIST(LogicalType &child) {
        LogicalType type(LogicalTypeId::LIST);
        type.type_info_ = make_shared<ListTypeInfo>(child);
        return type;
    }

    // 获取子类型
    LogicalType &GetChildType() const {
        D_ASSERT(id_ == LogicalTypeId::LIST);
        return ((ListTypeInfo *)type_info_.get())->child_type;
    }
};

struct ListTypeInfo : public ExtraTypeInfo {
    LogicalType child_type;

    ListTypeInfo(LogicalType child_type)
        : child_type(std::move(child_type)) {}
};

// LIST值的存储
struct NestedValueInfo : public ExtraValueInfo {
    vector<Value> values;  // 子元素值
};

// Value类中的LIST访问
class Value {
public:
    // 获取LIST元素
    Value GetListElement(idx_t index) {
        D_ASSERT(type_.id() == LogicalTypeId::LIST);

        auto &nested_info = (NestedValueInfo &)*value_info_;
        D_ASSERT(index < nested_info.values.size());

        return nested_info.values[index];
    }

    // 获取LIST大小
    idx_t GetListSize() {
        D_ASSERT(type_.id() == LogicalTypeId::LIST);
        auto &nested_info = (NestedValueInfo &)*value_info_;
        return nested_info.values.size();
    }
};
```

### 5.2 STRUCT类型

```cpp
// STRUCT类型实现

class LogicalType {
public:
    // 创建STRUCT类型
    static LogicalType STRUCT(
        case_insensitive_map_t<string, LogicalType> children) {

        LogicalType type(LogicalTypeId::STRUCT);
        type.type_info_ = make_shared<StructTypeInfo>(
            std::move(children)
        );
        return type;
    }

    // 获取字段类型
    LogicalType &GetChildType(const string &field_name) {
        D_ASSERT(id_ == LogicalTypeId::STRUCT);
        auto &struct_info = (StructTypeInfo *)type_info_.get();
        return struct_info->child_types[field_name];
    }
};

struct StructTypeInfo : public ExtraTypeInfo {
    // 字段名 → 类型映射
    case_insensitive_map_t<string, LogicalType> child_types;

    StructTypeInfo(case_insensitive_map_t<string, LogicalType> children)
        : child_types(std::move(children)) {}
};

// STRUCT值的存储
struct NestedValueInfo : public ExtraValueInfo {
    vector<Value> values;  // 字段值（按声明顺序）
};

// Value类中的STRUCT访问
class Value {
public:
    // 获取STRUCT字段
    Value GetStructField(const string &field_name) {
        D_ASSERT(type_.id() == LogicalTypeId::STRUCT);

        auto &struct_info = (StructTypeInfo &)*type_.type_info_;
        auto &nested_info = (NestedValueInfo &)*value_info_;

        // 查找字段索引
        idx_t field_idx = struct_info.GetChildIndex(field_name);

        return nested_info.values[field_idx];
    }

    // 获取STRUCT字段值
    Value &GetStructFieldRef(const string &field_name) {
        D_ASSERT(type_.id() == LogicalTypeId::STRUCT);

        auto &struct_info = (StructTypeInfo &)*type_.type_info_;
        auto &nested_info = (NestedValueInfo &)*value_info_;

        idx_t field_idx = struct_info.GetChildIndex(field_name);

        return nested_info.values[field_idx];
    }
};
```

### 5.3 ARRAY类型（固定大小数组）

```cpp
// ARRAY类型实现

class LogicalType {
public:
    // 创建ARRAY类型
    static LogicalType ARRAY(LogicalType &child, idx_t size) {
        LogicalType type(LogicalTypeId::ARRAY);
        type.type_info_ = make_shared<ArrayTypeInfo>(child, size);
        return type;
    }

    // 获取数组大小
    idx_t GetArraySize() const {
        D_ASSERT(id_ == LogicalTypeId::ARRAY);
        return ((ArrayTypeInfo *)type_info_.get())->array_size;
    }
};

struct ArrayTypeInfo : public ExtraTypeInfo {
    LogicalType child_type;
    idx_t array_size;

    ArrayTypeInfo(LogicalType child_type, idx_t array_size)
        : child_type(std::move(child_type)),
          array_size(array_size) {}
};
```

---

## 第六部分：类型转换

### 6.1 类型转换机制

```cpp
// 类型转换接口

class Value {
public:
    // 显式转换
    Value CastAs(ClientContext &context,
                const LogicalType &target_type,
                bool strict = false) const {

        // 1. 相同类型：直接返回
        if (type_ == target_type) {
            return *this;
        }

        // 2. NULL转换
        if (IsNull()) {
            return Value(target_type);
        }

        // 3. 调用特定转换函数
        switch (target_type.id()) {
        case LogicalTypeId::VARCHAR:
            return CastAsString();

        case LogicalTypeId::INTEGER:
            return CastAsInteger();

        case LogicalTypeId::BIGINT:
            return CastAsBigInt();

        case LogicalTypeId::DOUBLE:
            return CastAsDouble();

        case LogicalTypeId::DECIMAL:
            return CastAsDecimal(context, target_type);

        case LogicalTypeId::DATE:
            return CastAsDate();

        case LogicalTypeId::TIMESTAMP:
            return CastAsTimestamp();

        case LogicalTypeId::BOOLEAN:
            return CastAsBoolean();

        default:
            throw NotImplementedException(
                "Type cast not implemented"
            );
        }
    }

    // 安全转换（带错误处理）
    bool TryCastAs(ClientContext &context,
                   const LogicalType &target_type,
                   Value &new_value,
                   string *error_message = nullptr,
                   bool strict = false) const {

        try {
            new_value = CastAs(context, target_type, strict);
            return true;
        } catch (Exception &ex) {
            if (error_message) {
                *error_message = ex.what();
            }
            return false;
        }
    }

private:
    // ==================== 具体转换实现 ====================

    // 转换为字符串
    Value CastAsString() const {
        string str;

        switch (type_.id()) {
        case LogicalTypeId::INTEGER:
            str = to_string(value_.integer);
            break;

        case LogicalTypeId::BIGINT:
            str = to_string(value_.bigint);
            break;

        case LogicalTypeId::DOUBLE:
            str = to_string(value_.double_);
            break;

        case LogicalTypeId::BOOLEAN:
            str = value_.boolean ? "true" : "false";
            break;

        case LogicalTypeId::DATE:
            str = DateToString(value_.date);
            break;

        default:
            throw NotImplementedException("Cast to VARCHAR");
        }

        return Value::VARCHAR(str);
    }

    // 转换为整数
    Value CastAsInteger() const {
        int32_t result = 0;

        switch (type_.id()) {
        case LogicalTypeId::VARCHAR:
            result = StringToInteger<int32_t>(GetString());
            break;

        case LogicalTypeId::BIGINT:
            // 范围检查
            if (value_.bigint < INT_MIN || value_.bigint > INT_MAX) {
                throw OutOfRangeException("BIGINT out of INTEGER range");
            }
            result = (int32_t)value_.bigint;
            break;

        case LogicalTypeId::DOUBLE:
            result = (int32_t)value_.double_;
            break;

        case LogicalTypeId::BOOLEAN:
            result = value_.boolean ? 1 : 0;
            break;

        default:
            throw NotImplementedException("Cast to INTEGER");
        }

        return Value::INTEGER(result);
    }

    // 转换为DOUBLE
    Value CastAsDouble() const {
        double result = 0.0;

        switch (type_.id()) {
        case LogicalTypeId::VARCHAR:
            result = StringToDouble(GetString());
            break;

        case LogicalTypeId::INTEGER:
            result = (double)value_.integer;
            break;

        case LogicalTypeId::BIGINT:
            result = (double)value_.bigint;
            break;

        case LogicalTypeId::DECIMAL:
            result = DecimalToDouble(value_);
            break;

        case LogicalTypeId::BOOLEAN:
            result = value_.boolean ? 1.0 : 0.0;
            break;

        default:
            throw NotImplementedException("Cast to DOUBLE");
        }

        return Value::DOUBLE(result);
    }
};
```

### 6.2 隐式类型转换

```cpp
// 隐式类型转换规则

class ImplicitCast {
public:
    // 检查是否可以隐式转换
    static bool CanImplicitCast(LogicalType from, LogicalType to) {
        // 1. 相同类型
        if (from == to) {
            return true;
        }

        // 2. NULL可以转换为任何类型
        if (from.id() == LogicalTypeId::SQLNULL) {
            return true;
        }

        // 3. VARCHAR可以转换为数值类型
        if (from.id() == LogicalTypeId::VARCHAR) {
            return to.IsNumeric() || to.id() == LogicalTypeId::BOOLEAN;
        }

        // 4. 数值类型可以转换
        if (from.IsNumeric() && to.IsNumeric()) {
            return GetNumericPriority(from.id()) <=
                   GetNumericPriority(to.id());
        }

        // 5. 整数可以转换为DOUBLE
        if (from.IsIntegral() && to.id() == LogicalTypeId::DOUBLE) {
            return true;
        }

        return false;
    }

    // 执行隐式转换
    static Value PerformCast(Value value, LogicalType target_type) {
        // 检查是否需要转换
        if (!CanImplicitCast(value.type(), target_type)) {
            throw BinderException(
                "Cannot implicitly cast %s to %s",
                value.type().ToString(),
                target_type.ToString()
            );
        }

        // 执行转换
        return value.CastAs(context, target_type);
    }

private:
    // 数值类型优先级
    static int GetNumericPriority(LogicalTypeId type) {
        switch (type) {
        case LogicalTypeId::TINYINT:   return 1;
        case LogicalTypeId::SMALLINT:  return 2;
        case LogicalTypeId::INTEGER:   return 3;
        case LogicalTypeId::BIGINT:    return 4;
        case LogicalTypeId::HUGEINT:   return 5;
        case LogicalTypeId::FLOAT:     return 6;
        case LogicalTypeId::DOUBLE:    return 7;
        default: return 0;
        }
    }
};
```

---

## 第七部分：类型比较和排序

### 7.1 比较操作

```cpp
// Value类比较操作符

class Value {
public:
    // 相等比较
    bool operator==(const Value &rhs) const {
        // 1. 类型不同
        if (type_ != rhs.type_) {
            return false;
        }

        // 2. NULL比较（SQL三值逻辑）
        if (IsNull() || rhs.IsNull()) {
            return false;  // NULL != NULL
        }

        // 3. 浮点数近似比较
        if (type_.id() == LogicalTypeId::FLOAT ||
            type_.id() == LogicalTypeId::DOUBLE) {
            return ApproxEqual(*this, rhs);
        }

        // 4. 其他类型精确比较
        return Compare(rhs) == 0;
    }

    // 不等比较
    bool operator!=(const Value &rhs) const {
        return !(*this == rhs);
    }

    // 小于比较
    bool operator<(const Value &rhs) const {
        // NULL比较
        if (IsNull() || rhs.IsNull()) {
            return false;
        }

        return Compare(rhs) < 0;
    }

    // 大于比较
    bool operator>(const Value &rhs) const {
        return Compare(rhs) > 0;
    }

private:
    // 核心比较函数
    int Compare(const Value &rhs) const {
        D_ASSERT(type_ == rhs.type_);

        switch (type_.id()) {
        case LogicalTypeId::BOOLEAN:
            return value_.boolean - rhs.value_.boolean;

        case LogicalTypeId::TINYINT:
            return value_.tinyint - rhs.value_.tinyint;

        case LogicalTypeId::SMALLINT:
            return value_.smallint - rhs.value_.smallint;

        case LogicalTypeId::INTEGER:
            return value_.integer - rhs.value_.integer;

        case LogicalTypeId::BIGINT:
            if (value_.bigint < rhs.value_.bigint) return -1;
            if (value_.bigint > rhs.value_.bigint) return 1;
            return 0;

        case LogicalTypeId::DOUBLE:
            if (value_.double_ < rhs.value_.double_) return -1;
            if (value_.double_ > rhs.value_.double_) return 1;
            return 0;

        case LogicalTypeId::VARCHAR:
            return CompareString(*this, rhs);

        case LogicalTypeId::DATE:
            return value_.date - rhs.value_.date;

        default:
            throw NotImplementedException("Comparison for type");
        }
    }

    // 字符串比较
    static int CompareString(const Value &a, const Value &b) {
        auto a_str = a.GetString();
        auto b_str = b.GetString();

        return memcmp(a_str.GetData(),
                    b_str.GetData(),
                    MinValue(a_str.GetLength(), b_str.GetLength()));
    }

    // 浮点数近似相等
    static bool ApproxEqual(const Value &a, const Value &b,
                           double epsilon = 1e-9) {
        if (a.type_.id() == LogicalTypeId::FLOAT) {
            return std::abs(a.value_.float_ - b.value_.float_) < epsilon;
        } else {
            return std::abs(a.value_.double_ - b.value_.double_) < epsilon;
        }
    }
};
```

### 7.2 排序规则

```
┌─────────────────────────────────────────────────────────┐
│              DuckDB 排序规则                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  类型排序优先级（从低到高）：                           │
│                                                         │
│  1. NULL（最小）                                       │
│  2. BOOLEAN                                           │
│  3. TINYINT                                           │
│  4. SMALLINT                                          │
│  5. INTEGER                                           │
│  6. BIGINT                                            │
│  7. HUGEINT                                           │
│  8. FLOAT                                             │
│  9. DOUBLE                                            │
│  10. VARCHAR                                          │
│  11. DATE                                             │
│  12. TIMESTAMP                                        │
│                                                         │
│  复合类型排序：                                         │
│  • STRUCT：逐字段比较                                   │
│  • LIST：逐元素比较                                   │
│  • MAP：先比较键，再比较值                             │
│                                                         │
│  字符串排序规则：                                       │
│  • UTF-8字节序比较                                      │
│  • 大小写敏感                                          │
│  • 使用COLLATE可定制                                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 7.3 ORDER BY实现

```cpp
// 排序比较器

class ValueComparator {
public:
    ValueComparator(OrderType type,
                   vector<BoundOrderByNode> &orders)
        : order_type(type), orders(orders) {}

    // 比较两个DataChunk的行
    int Compare(DataChunk &left, idx_t left_idx,
                DataChunk &right, idx_t right_idx) {

        for (auto &order : orders) {
            // 1. 比较当前列
            int cmp = CompareColumn(left, left_idx,
                                   right, right_idx,
                                   order.expression);

            // 2. 如果不相等，应用排序方向
            if (cmp != 0) {
                return order.type == OrderType::ASCENDING ?
                    cmp : -cmp;
            }
        }

        return 0;  // 所有列相等
    }

private:
    OrderType order_type;
    vector<BoundOrderByNode> &orders;

    int CompareColumn(DataChunk &left, idx_t left_idx,
                     DataChunk &right, idx_t right_idx,
                     Expression &expr) {

        // 获取列值
        Value left_val = expr.GetValue(left, left_idx);
        Value right_val = expr.GetValue(right, right_idx);

        // 比较值
        if (left_val < right_val) return -1;
        if (left_val > right_val) return 1;
        return 0;
    }
};
```

---

## 学习总结

### 类型系统关键要点

1. **双层设计**：LogicalType（语义）+ PhysicalType（存储）
2. **内存优化**：小字符串内联、前缀比较
3. **类型安全**：强类型检查和安全的类型转换
4. **精度保持**：DECIMAL使用任意精度算术
5. **灵活扩展**：支持复合类型和用户自定义类型

### 类型性能优化

| 类型 | 优化策略 | 性能提升 |
|------|---------|---------|
| VARCHAR | 小字符串内联 | 2-3x |
| VARCHAR | 前缀比较 | 1.5-2x |
| DECIMAL | 直接存储 | 避免转换 |
| LIST/STRUCT | 延迟计算 | 减少拷贝 |
| BOOLEAN | 位图存储 | 64x压缩 |

### 推荐阅读

**论文：**
- "A survey of type systems for database systems"
- "Effective Type Specialization for High-Performance Database Systems"
- "IEEE 754-2019 - Floating-Point Arithmetic"

**代码位置：**
- `src/include/duckdb/common/types/` - 类型定义
- `src/common/types/` - 类型实现
- `src/function/cast/` - 类型转换函数

**相关课程：**
- [DuckDB深度课程_向量化执行引擎](./DuckDB深度课程_向量化执行引擎.md)
- [DuckDB深度课程_函数系统详解](./DuckDB深度课程_函数系统详解.md)

---

**最后更新：2026-01-23**
