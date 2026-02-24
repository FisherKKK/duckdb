# 🎓 DuckDB 30天深度学习课程 - 完整索引

欢迎来到DuckDB 30天深度学习课程！本课程将带你从零开始，系统学习DuckDB的核心架构、性能优化和数据库底层知识。

---

## 🚀 快速开始指南

### 第一次学习？从这里开始！

1. **📖 阅读本文档** (DuckDB课程总索引.md)
   - 了解课程结构和学习路径

2. **🎨 查看架构图解** ([DuckDB架构图解.md](./DuckDB架构图解.md))
   - 建立DuckDB整体架构的视觉认知
   - 理解数据流动和组件关系

3. **📚 浏览学习资源** ([DuckDB学习资源汇总.md](./DuckDB学习资源汇总.md))
   - 了解所有可用的学习材料
   - 选择适合你的学习路径

4. **📘 开始Week 1** ([DuckDB_30天学习课程.md](./DuckDB_30天学习课程.md))
   - Day 1: 整体架构
   - 按顺序学习每一天的内容

5. **💻 编写代码时**
   - 参考 [DuckDB学习速查表.md](./DuckDB学习速查表.md) - API和代码模式
   - 参考 [DuckDB课程_代码示例集.md](./DuckDB课程_代码示例集.md) - 完整代码示例

6. **🐛 遇到问题时**
   - 查阅 [DuckDB调试指南.md](./DuckDB调试指南.md) - 调试技巧
   - 查询 [DuckDB术语表.md](./DuckDB术语表.md) - 术语定义

7. **✏️ 每周末**
   - 完成 [DuckDB课程_练习题集.md](./DuckDB课程_练习题集.md) 中的练习

### 推荐学习流程

```
📌 开始
    ↓
🎨 DuckDB架构图解 (建立整体认知)
    ↓
📘 Week 1 课程 + 💻 代码示例 + ✏️ 练习题
    ↓
📗 Week 2 课程 + 💻 代码示例 + ✏️ 练习题
    ↓
📙 Week 3 课程 + 💻 代码示例 + ✏️ 练习题
    ↓
📕 Week 4 课程 + 💻 代码示例 + ✏️ 练习题
    ↓
🎓 完成 Mini-DuckDB 项目
    ↓
🚀 继续深入或贡献开源
```

---

## 📚 课程概述

**学习目标：**
- 💡 掌握向量化执行引擎的设计与实现
- 💡 理解查询优化器的工作原理
- 💡 学习列式存储和压缩技术
- 💡 掌握MVCC事务管理
- 💡 能够设计并实现一个简化版的分析型数据库

**适合人群：**
- 有C++基础的开发者
- 对数据库内核感兴趣的学生
- 希望深入理解DuckDB的工程师
- 想学习数据库底层知识的研究者

**学习方式：**
- 每天1-2小时理论学习
- 1-2小时源码阅读
- 2-3小时实践编码
- 每周完成一个综合项目

---

## 📖 课程文件导航

### 主课程文件（按学习顺序）

#### 1️⃣ [DuckDB_30天学习课程.md](./DuckDB_30天学习课程.md)
**第一周：核心架构与数据表示**
- ✅ Day 1: DuckDB总体架构概览
- ✅ Day 2: 类型系统与向量化执行基础
- ✅ Day 3: Vector和DataChunk - 数据流动的载体
- ✅ Day 4: 内存管理与BufferManager
- ✅ Day 5: SQL解析器架构
- ✅ Day 6: 表达式系统
- ✅ Day 7: 第一周总结 - 实现简单内存表

**关键概念：** Vector、DataChunk、ValidityMask、Allocator、AST、Expression

---

#### 2️⃣ [DuckDB_30天学习课程_第2-4周.md](./DuckDB_30天学习课程_第2-4周.md)
**第二周：查询处理与算子实现**
- ✅ Day 8: Binder与符号解析
- ✅ Day 9: 逻辑计划 - LogicalOperator
- ✅ Day 10: 物理计划 - PhysicalOperator
- ✅ Day 11: 算子实现 - TableScan和Filter
- ✅ Day 12: 算子实现 - Hash Join
- ✅ Day 13: 算子实现 - Aggregation
- ✅ Day 14: 第二周总结 - 实现基本查询执行

**关键概念：** Binder、LogicalOperator、PhysicalOperator、Pipeline、Source/Sink

---

#### 3️⃣ [DuckDB_30天学习课程_第3周_优化器.md](./DuckDB_30天学习课程_第3周_优化器.md)
**第三周：优化器与性能（Part 1）**
- ✅ Day 15: 优化器架构与规则系统
- ✅ Day 16: Filter Pushdown优化
- ✅ Day 17: Join Order优化
- ✅ Day 18: 统计信息与基数估计
- ✅ Day 19: 表达式优化与常量折叠

**关键概念：** Filter Pushdown、Join Order、HyperLogLog、基数估计、CSE

---

#### 4️⃣ [DuckDB_30天学习课程_第3-4周.md](./DuckDB_30天学习课程_第3-4周.md)
**第三周：优化器与性能（Part 2）& 第四周开篇**
- ✅ Day 20: 向量化执行深度解析
- ✅ Day 21: 第三周总结 - 实现优化规则
- ✅ Day 22: 存储引擎架构

**关键概念：** SIMD、缓存优化、DataTable、RowGroup、ColumnData

---

#### 5️⃣ [DuckDB_30天学习课程_第4周完整版.md](./DuckDB_30天学习课程_第4周完整版.md)
**第四周：存储引擎（Part 1）**
- ✅ Day 23: RowGroup与列存储详解
- ✅ Day 24: 压缩算法
- ✅ Day 25: MVCC事务管理
- ✅ Day 26: WAL与持久化

**关键概念：** 列式存储、FOR编码、Dictionary编码、MVCC、WAL、Checkpoint

---

#### 6️⃣ [DuckDB_30天学习课程_总结.md](./DuckDB_30天学习课程_总结.md)
**第四周：存储引擎（Part 2）& 课程总结**
- ✅ Day 27: Buffer管理与缓存策略
- ✅ Day 28: 第四周总结 - 实现简单存储引擎
- ✅ Day 29: 扩展系统与函数注册
- ✅ Day 30: 总结与Mini-DuckDB项目

**关键概念：** LRU、Clock算法、Extension、Mini-DuckDB

---

### 辅助学习文档

#### 7️⃣ [DuckDB学习速查表.md](./DuckDB学习速查表.md) 🚀
**快速参考手册**
- 核心数据结构API速查
- 常用代码模式和示例
- 算子实现模板
- 向量化操作参考
- 性能优化Checklist

**适用场景:** 编写代码时随时参考

---

#### 8️⃣ [DuckDB调试指南.md](./DuckDB调试指南.md) 🐛
**完整调试手册**
- GDB/LLDB调试技巧
- 内存调试 (Valgrind, ASan)
- 性能分析 (perf, Instruments)
- 常见错误解决方案
- 单元测试调试

**适用场景:** 遇到Bug或性能问题时查阅

---

#### 9️⃣ [DuckDB术语表.md](./DuckDB术语表.md) 📖
**完整术语词典**
- 200+ 核心术语定义
- A-Z字母索引
- 术语间关系说明
- 代码示例

**适用场景:** 不理解某个术语时快速查询

---

#### 🔟 [DuckDB架构图解.md](./DuckDB架构图解.md) 🎨
**可视化架构指南**
- ASCII艺术架构图
- 数据流可视化
- Pipeline执行图解
- 存储层次结构图
- 组件交互图

**适用场景:** 建立整体架构认知，理解数据流动

---

#### 1️⃣1️⃣ [DuckDB课程_代码示例集.md](./DuckDB课程_代码示例集.md) 💻
**完整代码示例库**
- CMakeLists.txt构建模板
- Week 1-4 完整代码示例
- Mini-DuckDB完整实现
- 调试和编译说明

**适用场景:** 需要可运行的代码参考

---

#### 1️⃣2️⃣ [DuckDB课程_练习题集.md](./DuckDB课程_练习题集.md) ✏️
**进阶练习题库**
- 8个进阶练习题
- 详细解答（可折叠）
- TPC-H Query 1实现
- 学习检验清单

**适用场景:** 巩固每周所学知识

---

#### 1️⃣3️⃣ [DuckDB学习资源汇总.md](./DuckDB学习资源汇总.md) 📚
**完整资源导航**
- 所有课程文件索引
- 学习路径推荐
- DuckDB官方资源
- 学术论文清单
- 推荐书籍和在线课程
- 开发工具推荐

**适用场景:** 📌 开始学习前必读，寻找额外学习资源

---

#### 1️⃣4️⃣ [DuckDB高级课程_CMake编译与优化技巧.md](./DuckDB高级课程_CMake编译与优化技巧.md) 🚀
**高级进阶课程**
- 第一部分：CMake构建系统详解
  - 构建系统架构概览
  - 核心CMakeLists文件解析
  - Unity Build系统
  - Makefile接口与常用命令
  - 编译器标志详解
  - 跨平台编译配置
- 第二部分：代码架构实现细节
  - 模块化架构
  - 类型系统架构
  - 向量化执行架构
  - 执行算子架构
- 第三部分：编译优化技巧
  - 编译时优化
  - 模板元编程优化
  - 内存优化技巧
  - SIMD优化
  - 缓存优化
  - 查询优化技巧
- 第四部分：性能分析与调优
  - 性能分析工具（perf、Instruments、Valgrind）
  - 查询性能分析
  - 常见性能问题与解决
  - 高级性能调优
- 第五部分：最佳实践总结

**适用场景:** 深入理解构建系统、性能优化和底层实现细节

---

#### 1️⃣5️⃣ [DuckDB高级课程_扩展系统开发.md](./DuckDB高级课程_扩展系统开发.md) 🔌
**扩展开发实战课程**
- 第一部分：扩展系统架构
  - 扩展类型分类（In-tree / Out-of-tree）
  - 扩展加载机制
  - 扩展入口点详解
- 第二部分：创建自定义扩展
  - 项目结构与CMakeLists.txt
  - 最小化扩展示例
  - 使用扩展模板
- 第三部分：高级扩展功能
  - 标量函数开发
  - 聚合函数开发
  - 表函数开发
  - 复制函数开发
- 第四部分：构建与分发
  - 本地构建
  - 跨平台构建（Linux/macOS/Windows）
  - 版本控制与签名
  - 发布扩展
- 第五部分：测试与调试
  - 单元测试
  - SQL逻辑测试
  - 调试技巧
- 第六部分：最佳实践
  - 性能优化
  - 错误处理
  - 文档编写
- 第七部分：实际案例
  - IP地址函数扩展
  - 机器学习扩展

**适用场景:** 开发自定义扩展、贡献DuckDB生态系统

---

#### 1️⃣6️⃣ [DuckDB大师课程_从零实现数据库系统.md](./DuckDB大师课程_从零实现数据库系统.md) 🏆
**从零实现数据库系统（大师课程）**
- 第一阶段：存储引擎实现
  - 模块1：页面管理器（Page Manager）
  - 模块2：缓冲池管理器（Buffer Pool - LRU）
  - 模块3：B+树索引实现
- 第二阶段：执行引擎实现
  - 模块4：SQL解析器（递归下降）
  - 模块5：查询执行引擎（向量化）
- 第三阶段：事务系统实现
  - 模块6：并发控制（MVCC）
  - 模块7：WAL预写日志
- 第四阶段：项目整合
  - 完整的Mini-DB实现
  - 各模块集成测试

**课程特点：**
- ✅ 从零开始，完整实现
- ✅ 生产级代码质量
- ✅ 参考DuckDB架构
- ✅ 可直接运行的代码

**适用场景:** 深入学习数据库底层实现、准备面试、研究数据库内核

---

#### 1️⃣7️⃣ [DuckDB高级课程_查询优化器深度解析.md](./DuckDB高级课程_查询优化器深度解析.md) ⚡
**查询优化器深度解析专题课程**
- 第一部分：优化器架构
  - 优化器执行流程
  - 优化规则分类
  - 优化规则执行顺序
- 第二部分：表达式优化
  - 常量折叠
  - 表达式简化
  - 公共子表达式消除（CSE）
- 第三部分：逻辑优化
  - Filter Pushdown
  - Join Elimination
  - Limit Pushdown
- 第四部分：Join Order优化
  - 动态规划算法
  - Query Graph
  - 代价模型
- 第五部分：统计信息
  - 统计信息收集
  - HyperLogLog基数估计
  - 选择性估计
- 第六部分：实践项目
  - 实现Filter Pushdown
  - 实现Join Order优化器

**适用场景:** 深入学习查询优化器原理、数据库性能调优、实现自定义优化规则

---

#### 1️⃣8️⃣ [DuckDB深度课程_并发控制与事务处理.md](./DuckDB深度课程_并发控制与事务处理.md) 🔐
**并发控制与事务处理专题课程**
- 第一部分：事务基础
  - ACID特性详解
  - 事务隔离级别
  - 并发问题（脏读、不可重复读、幻读）
- 第二部分：MVCC实现
  - MVCC基本概念与架构
  - 版本链实现
  - 可见性判断逻辑
- 第三部分：并发控制机制
  - 锁机制（共享锁、排他锁）
  - 写偏问题
  - 乐观并发控制
- 第四部分：WAL与恢复
  - Write-Ahead Log实现
  - 崩溃恢复机制
  - Checkpoint机制
- 第五部分：性能优化
  - 并发性能优化技巧
  - 事务吞吐量优化
  - 批量提交与组提交
- 第六部分：实践项目
  - 实现简单MVCC系统
  - 并发场景测试

**适用场景:** 深入学习事务处理、并发编程、数据库ACID实现

---

#### 1️⃣8️⃣ [DuckDB深度课程_列存储与数据压缩.md](./DuckDB深度课程_列存储与数据压缩.md) 📦
**列存储与数据压缩专题课程**
- 第一部分：列存储架构
  - 行存储 vs 列存储对比
  - DuckDB列存储架构
  - RowGroup管理
- 第二部分：压缩算法
  - BitPacking（位打包）
  - FOR编码（Frame-Of-Reference）
  - RLE（行程编码）
  - Dictionary编码
- 第三部分：高级压缩算法
  - Gorillas算法（浮点数压缩）
  - Chimp算法（改进的浮点数压缩）
- 第四部分：存储优化
  - 自适应压缩选择
  - 分区裁剪
  - 延迟物化
- 第五部分：实践项目
  - 实现压缩算法测试
  - 性能基准测试

**适用场景:** 深入学习存储引擎、数据压缩、列式数据库实现

---

#### 1️⃣9️⃣ [DuckDB深度课程_向量化执行引擎.md](./DuckDB深度课程_向量化执行引擎.md) ⚡
**向量化执行引擎深度解析**
- 第一部分：向量化基础
  - 向量化 vs 行式执行
  - DataChunk 结构详解
  - Vector 内部实现
- 第二部分：Vector 类型系统
  - FLAT_VECTOR（平面向量）
  - CONSTANT_VECTOR（常量向量）
  - DICTIONARY_VECTOR（字典向量）
  - SEQUENCE_VECTOR（序列向量）
- 第三部分：零拷贝优化
  - SelectionVector 机制
  - ValidityMask NULL值处理
  - 引用语义与数据共享
- 第四部分：SIMD 优化
  - 向量化操作与 SIMD
  - 缓存友好设计
  - 批处理大小优化
- 第五部分：实践项目
  - 实现简单向量化算子
  - SIMD 性能测试

**适用场景:** 深入学习向量化执行引擎、SIMD优化、高性能计算

---

#### 2️⃣0️⃣ [DuckDB深度课程_执行算子实现详解.md](./DuckDB深度课程_执行算子实现详解.md) 🔧
**执行算子实现详解**
- 第一部分：PhysicalOperator 基类
  - Execute() 接口
  - State 管理模式
  - 算子类型分类
- 第二部分：Source 算子
  - TableScanPhysicalOperator
  - PhysicalTableScan 实现细节
- 第三部分：Operator 算子
  - FilterPhysicalOperator
  - ProjectionPhysicalOperator
  - PhysicalFilter 实现
- 第四部分：Sink 算子
  - HashJoinPhysicalOperator
  - PhysicalHashJoin 实现细节
  - HashAggregatePhysicalOperator
  - OrderByPhysicalOperator
- 第五部分：Pipeline 执行模型
  - PipelineExecutor 工作原理
  - Push-based vs Pull-based
  - 并行执行
- 第六部分：实践项目
  - 实现自定义 PhysicalOperator
  - 算子性能测试

**适用场景:** 深入学习执行算子、Pipeline执行、查询执行引擎

---

#### 2️⃣1️⃣ [DuckDB深度课程_Planner与Binder.md](./DuckDB深度课程_Planner与Binder.md) 📋
**Planner与Binder深度解析**
- 第一部分：SQL 处理流程
  - Parser → Transformer → Binder
  - Binder 职责与工作流程
  - Planner 规划查询计划
- 第二部分：Binder 符号解析
  - BindContext 上下文管理
  - 列名绑定
  - 表名解析
  - 子查询绑定
- 第三部分：LogicalOperator 层次结构
  - LogicalGet
  - LogicalFilter
  - LogicalProjection
  - LogicalJoin
  - LogicalAggregate
- 第四部分：表达式绑定
  - Bound_* 表达式类型
  - 类型推导
  - 函数绑定
- 第五部分：CTE 与子查询处理
  - CTE 物化策略
  - 相关子查询处理
  - 子查询去相关化
- 第六部分：实践项目
  - 实现简单 Binder
  - 实现逻辑计划生成

**适用场景:** 深入学习SQL处理流程、符号绑定、查询规划

---

#### 2️⃣2️⃣ [DuckDB深度课程_类型系统实现.md](./DuckDB深度课程_类型系统实现.md) 🎨
**类型系统实现详解**
- 第一部分：类型系统概览
  - LogicalType vs PhysicalType
  - 类型层次结构
  - 类型转换机制
- 第二部分：基础类型实现
  - 整数类型（TINYINT, SMALLINT, INTEGER, BIGINT, HUGEINT）
  - 浮点类型（FLOAT, DOUBLE）
  - 布尔类型（BOOLEAN）
- 第三部分：字符串类型
  - VARCHAR 实现
  - string_t 结构
  - 小字符串内联优化
  - BLOB 类型
- 第四部分：DECIMAL 类型
  - 精度与比例
  - 存储布局
  - 运算实现
- 第五部分：复合类型
  - LIST 类型
  - STRUCT 类型
  - ARRAY 类型
  - MAP 类型
- 第六部分：实践项目
  - 实现自定义类型
  - 类型转换测试

**适用场景:** 深入学习类型系统、数据表示、类型转换

---

#### 2️⃣3️⃣ [DuckDB深度课程_函数系统详解.md](./DuckDB深度课程_函数系统详解.md) ⚙️
**函数系统详解**
- 第一部分：函数系统概览
  - ScalarFunction（标量函数）
  - AggregateFunction（聚合函数）
  - TableFunction（表函数）
  - ScalarFunctionSet 和重载
- 第二部分：标量函数
  - 函数注册机制
  - 向量化执行
  - 参数验证
  - 返回值推导
- 第三部分：聚合函数
  - AggregateState 状态管理
  - Update 函数
  - Combine 函数（分布式）
  - Finalize 函数
  - 简单聚合 vs 有序聚合
- 第四部分：表函数
  - TableFunctionInfo
  - 表函数执行流程
  - 参数化表函数
  - Pragma 函数
- 第五部分：函数查找与绑定
  - 函数名解析
  - 类型匹配
  - 最佳函数选择
  - 隐式类型转换
- 第六部分：实践项目
  - 实现自定义标量函数
  - 实现自定义聚合函数

**适用场景:** 深入学习函数系统、扩展开发、UDF实现

---

#### 2️⃣4️⃣ [DuckDB深度课程_Catalog元数据管理.md](./DuckDB深度课程_Catalog元数据管理.md) 🗂️
**Catalog元数据管理详解**
- 第一部分：Catalog 系统架构
  - Catalog 层次结构
  - DatabaseCatalog
  - SchemaCatalogEntry
- 第二部分：表元数据
  - TableCatalogEntry
  - 列定义（ColumnDefinition）
  - 约束（Constraint）
  - 索引（Index）
- 第三部分：视图与序列
  - ViewCatalogEntry
  - SequenceCatalogEntry
  - 依赖管理
- 第四部分：函数元数据
  - ScalarFunctionCatalogEntry
  - 函数重载管理
- 第五部分：依赖管理
  - DependencyManager
  - 级联删除与更新
  - 依赖图构建
  - 导出顺序计算
- 第六部分：MVCC 支持
  - CatalogEntry 版本链
  - 事务可见性
  - DROP 与 ROLLBACK 处理
- 第七部分：实践项目
  - 实现简单 Catalog
  - 依赖管理测试

**适用场景:** 深入学习元数据管理、事务性元数据、Catalog实现

---

## 🗺️ 快速导航

### 按主题查找

#### 🏗️ 架构基础
- [整体架构](./DuckDB_30天学习课程.md#day-1-duckdb总体架构概览) - Day 1
- [类型系统](./DuckDB_30天学习课程.md#day-2-类型系统与向量化执行基础) - Day 2
- [数据结构](./DuckDB_30天学习课程.md#day-3-vector和datachunk---数据流动的载体) - Day 3

#### 💾 内存与存储
- [内存管理](./DuckDB_30天学习课程.md#day-4-内存管理与buffermanager) - Day 4
- [Buffer管理](./DuckDB_30天学习课程_总结.md#day-27-buffer管理与缓存策略) - Day 27
- [列式存储](./DuckDB_30天学习课程_第4周完整版.md#day-23-rowgroup与列存储详解) - Day 23
- [压缩算法](./DuckDB_30天学习课程_第4周完整版.md#day-24-压缩算法) - Day 24

#### 🔍 查询处理
- [SQL解析](./DuckDB_30天学习课程.md#day-5-sql解析器架构) - Day 5
- [表达式系统](./DuckDB_30天学习课程.md#day-6-表达式系统) - Day 6
- [Binder](./DuckDB_30天学习课程_第2-4周.md#day-8-binder与符号解析) - Day 8
- [逻辑计划](./DuckDB_30天学习课程_第2-4周.md#day-9-逻辑计划---logicaloperator) - Day 9
- [物理计划](./DuckDB_30天学习课程_第2-4周.md#day-10-物理计划---physicaloperator) - Day 10

#### ⚡ 执行算子
- [TableScan & Filter](./DuckDB_30天学习课程_第2-4周.md#day-11-算子实现---tablescan和filter) - Day 11
- [Hash Join](./DuckDB_30天学习课程_第2-4周.md#day-12-算子实现---hash-join) - Day 12
- [Aggregation](./DuckDB_30天学习课程_第2-4周.md#day-13-算子实现---aggregation) - Day 13

#### 🚀 查询优化
- [优化器架构](./DuckDB_30天学习课程_第3周_优化器.md#day-15-优化器架构与规则系统) - Day 15
- [Filter Pushdown](./DuckDB_30天学习课程_第3周_优化器.md#day-16-filter-pushdown优化) - Day 16
- [Join Order](./DuckDB_30天学习课程_第3周_优化器.md#day-17-join-order优化) - Day 17
- [统计信息](./DuckDB_30天学习课程_第3周_优化器.md#day-18-统计信息与基数估计) - Day 18
- [表达式优化](./DuckDB_30天学习课程_第3周_优化器.md#day-19-表达式优化与常量折叠) - Day 19
- [向量化执行](./DuckDB_30天学习课程_第3-4周.md#day-20-向量化执行深度解析) - Day 20

#### 🔐 事务与持久化
- [MVCC](./DuckDB_30天学习课程_第4周完整版.md#day-25-mvcc事务管理) - Day 25
- [WAL](./DuckDB_30天学习课程_第4周完整版.md#day-26-wal与持久化) - Day 26

#### 🔧 扩展开发
- [Extension系统](./DuckDB_30天学习课程_总结.md#day-29-扩展系统与函数注册) - Day 29

---

## 🎯 学习路径推荐

### 🌟 快速入门（5天）
适合想快速了解DuckDB核心概念的开发者：
1. Day 1: 整体架构
2. Day 3: Vector和DataChunk
3. Day 11: TableScan和Filter
4. Day 16: Filter Pushdown
5. Day 23: 列式存储

### 📈 全面学习（30天）
按顺序完成所有内容，每周完成综合项目

### 🔬 深度研究（按需）
根据兴趣选择特定主题深入研究：
- **性能优化方向：** Day 16-21
- **存储引擎方向：** Day 22-28
- **查询处理方向：** Day 8-14

### 🏆 大师课程（10周）
适合想从零实现数据库系统的开发者：
**[DuckDB大师课程_从零实现数据库系统.md](./DuckDB大师课程_从零实现数据库系统.md)**

```
Week 1-2: 存储引擎
  ├── 页面管理器
  ├── 缓冲池（LRU）
  └── B+树索引

Week 3-4: 执行引擎
  ├── SQL解析器
  ├── 查询计划器
  └── 向量化执行

Week 5-6: 事务系统
  ├── MVCC并发控制
  └── WAL预写日志

Week 7-8: 优化器
  ├── 逻辑优化
  └── 物理优化

Week 9-10: 高级特性
  ├── 压缩算法
  └── 并行执行
```

---

## 💻 实践项目清单

### 第一周项目
- ✅ **SimpleTable** - 实现基本的内存表
  - 文件：`DuckDB_30天学习课程.md` Day 7

### 第二周项目
- ✅ **SimpleQueryEngine** - 实现查询执行引擎
  - 文件：`DuckDB_30天学习课程_第2-4周.md` Day 14

### 第三周项目
- ✅ **SimpleOptimizer** - 实现查询优化器
  - 文件：`DuckDB_30天学习课程_第3-4周.md` Day 21

### 第四周项目
- ✅ **SimpleDiskStorage** - 实现持久化存储
  - 文件：`DuckDB_30天学习课程_总结.md` Day 28

### 最终项目
- ✅ **Mini-DuckDB** - 整合所有组件
  - 文件：`DuckDB_30天学习课程_总结.md` Day 30

---

## 📊 知识点速查表

### 核心数据结构
| 结构 | 用途 | 文件位置 | 学习章节 |
|------|------|----------|----------|
| Vector | 列向量 | `src/common/types/vector.hpp` | Day 3 |
| DataChunk | 批数据 | `src/common/types/data_chunk.hpp` | Day 3 |
| SelectionVector | 行索引 | `src/common/types/selection_vector.hpp` | Day 11 |
| ValidityMask | NULL标记 | `src/common/types/validity_mask.hpp` | Day 3 |

### 核心算子
| 算子 | 类型 | 文件位置 | 学习章节 |
|------|------|----------|----------|
| TableScan | Source | `src/execution/operator/scan/` | Day 11 |
| Filter | Operator | `src/execution/operator/filter/` | Day 11 |
| HashJoin | Sink | `src/execution/operator/join/` | Day 12 |
| HashAggregate | Sink | `src/execution/operator/aggregate/` | Day 13 |

### 优化技术
| 技术 | 效果 | 文件位置 | 学习章节 |
|------|------|----------|----------|
| Filter Pushdown | 2-10x | `src/optimizer/filter_pushdown.cpp` | Day 16 |
| Join Order | 10-100x | `src/optimizer/join_order/` | Day 17 |
| 向量化 | 2-8x | `src/common/vector_operations/` | Day 20 |
| 压缩 | 5-20x | `src/storage/compression/` | Day 24 |

---

## 🛠️ 环境准备

### 构建DuckDB
```bash
# 克隆仓库
git clone https://github.com/duckdb/duckdb.git
cd duckdb

# Debug构建（用于学习）
make debug

# 运行测试
./build/debug/test/unittest

# 运行DuckDB CLI
./build/debug/duckdb
```

### 推荐工具
- **IDE:** VSCode / CLion
- **调试器:** GDB / LLDB
- **性能分析:** perf / Instruments
- **文档:** Doxygen

---

## 📝 学习建议

### ✅ 推荐做法
1. **每天阅读对应章节** - 理解概念
2. **阅读引用的源码** - 深入细节
3. **完成实践任务** - 动手验证
4. **做笔记** - 记录关键点
5. **每周完成项目** - 整合知识

### ❌ 常见陷阱
1. ❌ 跳过基础章节直接看高级内容
2. ❌ 只看理论不写代码
3. ❌ 遇到困难就放弃
4. ❌ 不读源码只看课程
5. ❌ 不做练习项目

---

## 🤝 社区资源

### 官方资源
- **官网:** https://duckdb.org
- **文档:** https://duckdb.org/docs/
- **GitHub:** https://github.com/duckdb/duckdb
- **Blog:** https://duckdb.org/blog

### 社区支持
- **Discord:** https://discord.gg/tcvwpjfnZx
- **Twitter:** @duckdb
- **Stack Overflow:** [duckdb] tag

### 相关论文
1. "MonetDB/X100: Hyper-Pipelining Query Execution"
2. "Morsel-Driven Parallelism"
3. "Push-Based Execution in DuckDB"
4. "Everything You Always Wanted to Know About Compiled and Vectorized Queries"

---

## 📈 学习进度追踪

使用这个checklist追踪你的学习进度：

### 第一周：核心架构
- [ ] Day 1: 架构概览
- [ ] Day 2: 类型系统
- [ ] Day 3: Vector和DataChunk
- [ ] Day 4: 内存管理
- [ ] Day 5: SQL解析
- [ ] Day 6: 表达式系统
- [ ] Day 7: 实现SimpleTable ⭐

### 第二周：查询处理
- [ ] Day 8: Binder
- [ ] Day 9: LogicalOperator
- [ ] Day 10: PhysicalOperator
- [ ] Day 11: TableScan & Filter
- [ ] Day 12: Hash Join
- [ ] Day 13: Aggregation
- [ ] Day 14: 实现SimpleQueryEngine ⭐

### 第三周：优化器
- [ ] Day 15: 优化器架构
- [ ] Day 16: Filter Pushdown
- [ ] Day 17: Join Order
- [ ] Day 18: 统计信息
- [ ] Day 19: 表达式优化
- [ ] Day 20: 向量化执行
- [ ] Day 21: 实现SimpleOptimizer ⭐

### 第四周：存储引擎
- [ ] Day 22: 存储架构
- [ ] Day 23: RowGroup
- [ ] Day 24: 压缩算法
- [ ] Day 25: MVCC
- [ ] Day 26: WAL
- [ ] Day 27: Buffer管理
- [ ] Day 28: 实现SimpleDiskStorage ⭐

### 最终项目
- [ ] Day 29: Extension系统
- [ ] Day 30: Mini-DuckDB ⭐⭐⭐

---

## 🎓 结业标准

完成课程后，你应该能够：

✅ **理论掌握**
- 解释向量化执行的原理和优势
- 描述MVCC的工作机制
- 说明查询优化的各种技术
- 理解列式存储的设计原理

✅ **实践能力**
- 阅读并理解DuckDB源码
- 实现基本的查询算子
- 编写查询优化规则
- 设计简单的存储引擎
- 开发DuckDB扩展

✅ **项目经验**
- 完成SimpleTable实现
- 完成SimpleQueryEngine
- 完成SimpleOptimizer
- 完成SimpleDiskStorage
- 完成Mini-DuckDB

---

## 💡 常见问题

**Q: 需要什么基础知识？**
A: C++基础、数据结构、算法基础

**Q: 每天需要多少时间？**
A: 建议4-6小时（理论2小时 + 实践4小时）

**Q: 可以跳过某些章节吗？**
A: 不建议，内容是递进的

**Q: 如何获得帮助？**
A: 查看课程内容、阅读源码、加入Discord社区

**Q: 完成后的下一步？**
A: 贡献DuckDB、研究论文、构建自己的数据库

---

## 📞 反馈与贡献

如果你发现课程中的问题或有改进建议，欢迎：
1. 提交Issue
2. 提交PR
3. 在Discord讨论

---

## 📜 许可

本课程内容基于DuckDB MIT许可。

---

**祝学习愉快！Let's build databases! 🦆🚀**

最后更新：2026-01-17
