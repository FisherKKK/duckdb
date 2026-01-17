# 📚 DuckDB 学习资源汇总

本文档汇总了所有DuckDB学习资源，包括课程文件、参考资料、外部资源和推荐阅读。

---

## 🎓 课程文件索引

### 主课程（按学习顺序）

#### 📘 第一周：核心架构与数据表示
**文件:** `DuckDB_30天学习课程.md`

- Day 1: DuckDB总体架构概览
- Day 2: 类型系统与向量化执行基础
- Day 3: Vector和DataChunk - 数据流动的载体
- Day 4: 内存管理与BufferManager
- Day 5: SQL解析器架构
- Day 6: 表达式系统
- Day 7: 第一周总结 - 实现SimpleTable

**关键概念:** Vector, DataChunk, ValidityMask, Allocator, AST, Expression

---

#### 📗 第二周：查询处理与算子实现
**文件:** `DuckDB_30天学习课程_第2-4周.md`

- Day 8: Binder与符号解析
- Day 9: 逻辑计划 - LogicalOperator
- Day 10: 物理计划 - PhysicalOperator
- Day 11: 算子实现 - TableScan和Filter
- Day 12: 算子实现 - Hash Join
- Day 13: 算子实现 - Aggregation
- Day 14: 第二周总结 - 实现基本查询执行

**关键概念:** Binder, LogicalOperator, PhysicalOperator, Pipeline, Source/Sink

---

#### 📙 第三周：优化器与性能
**文件:** `DuckDB_30天学习课程_第3周_优化器.md`, `DuckDB_30天学习课程_第3-4周.md`

- Day 15: 优化器架构与规则系统
- Day 16: Filter Pushdown优化
- Day 17: Join Order优化
- Day 18: 统计信息与基数估计
- Day 19: 表达式优化与常量折叠
- Day 20: 向量化执行深度解析
- Day 21: 第三周总结 - 实现优化规则

**关键概念:** Filter Pushdown, Join Order, HyperLogLog, 基数估计, CSE, SIMD

---

#### 📕 第四周：存储引擎
**文件:** `DuckDB_30天学习课程_第4周完整版.md`, `DuckDB_30天学习课程_总结.md`

- Day 22: 存储引擎架构
- Day 23: RowGroup与列存储详解
- Day 24: 压缩算法
- Day 25: MVCC事务管理
- Day 26: WAL与持久化
- Day 27: Buffer管理与缓存策略
- Day 28: 第四周总结 - 实现简单存储引擎
- Day 29: 扩展系统与函数注册
- Day 30: 总结与Mini-DuckDB项目

**关键概念:** 列式存储, FOR编码, Dictionary编码, MVCC, WAL, Checkpoint, LRU

---

### 辅助文档

#### 🔍 速查与参考

1. **DuckDB课程总索引.md**
   - 完整的课程导航
   - 主题快速查找
   - 学习路径推荐
   - 进度追踪清单

2. **DuckDB学习速查表.md**
   - 核心API速查
   - 常用代码模式
   - 数据结构速览
   - 性能优化Checklist

3. **DuckDB术语表.md**
   - 200+ 核心术语定义
   - A-Z字母索引
   - 术语关系说明
   - 示例代码

4. **DuckDB架构图解.md**
   - ASCII艺术架构图
   - 数据流可视化
   - 组件交互图
   - 执行流程图

5. **DuckDB调试指南.md**
   - GDB/LLDB调试技巧
   - 内存调试 (Valgrind, ASan)
   - 性能分析 (perf, Instruments)
   - 常见错误解决方案
   - 单元测试调试

---

#### 💻 实践材料

6. **DuckDB课程_代码示例集.md**
   - 完整可运行代码
   - CMakeLists.txt构建模板
   - 每周代码示例
   - 调试和编译说明

7. **DuckDB课程_练习题集.md**
   - 8个进阶练习
   - 详细解答
   - TPC-H Query 1实现
   - 学习检验清单

---

## 🗂️ 文件快速访问

```
duckdb/
├── CLAUDE.md                          # 开发者指南
├── DuckDB课程总索引.md                 # 📌 从这里开始！
├── DuckDB_30天学习课程.md              # Week 1
├── DuckDB_30天学习课程_第2-4周.md      # Week 2
├── DuckDB_30天学习课程_第3周_优化器.md # Week 3 Part 1
├── DuckDB_30天学习课程_第3-4周.md      # Week 3 Part 2
├── DuckDB_30天学习课程_第4周完整版.md  # Week 4 Part 1
├── DuckDB_30天学习课程_总结.md         # Week 4 Part 2 & Summary
├── DuckDB学习速查表.md                 # 🚀 常用参考
├── DuckDB调试指南.md                   # 🐛 遇到问题看这里
├── DuckDB术语表.md                     # 📖 术语查询
├── DuckDB架构图解.md                   # 🎨 可视化架构
├── DuckDB课程_代码示例集.md            # 💻 实践代码
└── DuckDB课程_练习题集.md              # ✏️ 练习题
```

---

## 🎯 学习路径推荐

### 🌟 快速入门 (5天)
适合想快速了解DuckDB核心概念的开发者

**Day 1:** DuckDB_30天学习课程.md - Day 1 (整体架构)
**Day 2:** DuckDB_30天学习课程.md - Day 3 (Vector和DataChunk)
**Day 3:** DuckDB_30天学习课程_第2-4周.md - Day 11 (TableScan和Filter)
**Day 4:** DuckDB_30天学习课程_第3周_优化器.md - Day 16 (Filter Pushdown)
**Day 5:** DuckDB_30天学习课程_第4周完整版.md - Day 23 (列式存储)

**辅助材料:**
- DuckDB架构图解.md - 建立整体认知
- DuckDB学习速查表.md - 常用API参考

---

### 📈 系统学习 (30天)
完整的30天课程，适合深入学习

**Week 1-4:** 按顺序学习所有主课程文件
**每周:** 完成对应周的练习题 (DuckDB课程_练习题集.md)
**每周项目:**
- Week 1: SimpleTable
- Week 2: SimpleQueryEngine
- Week 3: SimpleOptimizer
- Week 4: SimpleDiskStorage

**辅助材料:**
- 每天参考 DuckDB学习速查表.md
- 遇到问题查阅 DuckDB调试指南.md
- 不理解术语查询 DuckDB术语表.md
- 定期回顾 DuckDB架构图解.md

**最终项目:** Mini-DuckDB (DuckDB_30天学习课程_总结.md - Day 30)

---

### 🔬 专题深入 (按需)
根据兴趣选择特定主题

#### 性能优化方向
1. Day 16: Filter Pushdown
2. Day 17: Join Order Optimization
3. Day 18: Statistics & Cardinality Estimation
4. Day 19: Expression Optimization
5. Day 20: Vectorization & SIMD
6. Day 24: Compression

**实践:** 实现SimpleOptimizer项目

---

#### 存储引擎方向
1. Day 22: Storage Architecture
2. Day 23: RowGroup & Columnar Storage
3. Day 24: Compression Algorithms
4. Day 25: MVCC
5. Day 26: WAL & Persistence
6. Day 27: Buffer Management

**实践:** 实现SimpleDiskStorage项目

---

#### 查询处理方向
1. Day 8: Binder
2. Day 9: Logical Plan
3. Day 10: Physical Plan
4. Day 11: TableScan & Filter
5. Day 12: Hash Join
6. Day 13: Aggregation

**实践:** 实现SimpleQueryEngine项目

---

## 📖 DuckDB官方资源

### 核心资源

1. **官方网站**
   - URL: https://duckdb.org
   - 内容: 产品介绍、下载、新闻

2. **官方文档**
   - URL: https://duckdb.org/docs/
   - 内容: SQL参考、API文档、配置指南
   - 重点章节:
     - Architecture Overview
     - SQL Introduction
     - Data Import/Export
     - Extensions

3. **GitHub仓库**
   - URL: https://github.com/duckdb/duckdb
   - 内容: 源代码、Issues、Pull Requests
   - 推荐阅读:
     - README.md - 项目概览
     - CONTRIBUTING.md - 贡献指南
     - src/ - 源代码

4. **官方博客**
   - URL: https://duckdb.org/news/
   - 内容: 技术博文、版本发布、案例分析
   - 推荐文章:
     - "Announcing DuckDB" - 项目介绍
     - "Push-Based Execution" - 执行模型
     - "Why DuckDB" - 设计理念

---

### 社区资源

5. **Discord社区**
   - URL: https://discord.gg/tcvwpjfnZx
   - 用途: 实时交流、问题求助、讨论

6. **Twitter/X**
   - 账号: @duckdb
   - 内容: 新闻、更新、社区动态

7. **Stack Overflow**
   - 标签: [duckdb]
   - URL: https://stackoverflow.com/questions/tagged/duckdb
   - 用途: 技术问答

8. **YouTube**
   - 搜索: "DuckDB"
   - 内容: 演讲视频、教程、会议演示

---

## 📄 学术论文

### DuckDB相关论文

1. **"Push-Based Execution in DuckDB" (2023)**
   - 主题: DuckDB的执行模型
   - 链接: https://duckdb.org/pdf/CIDR2023-DuckDB-Push.pdf
   - 关键点: Push-based pipeline, Morsel-driven parallelism

2. **"DuckDB: an Embeddable Analytical Database" (2019)**
   - 主题: DuckDB的设计和架构
   - 会议: SIGMOD 2019
   - 关键点: Vectorized execution, Columnar storage

---

### 相关数据库论文

3. **"MonetDB/X100: Hyper-Pipelining Query Execution" (2005)**
   - 主题: 向量化执行引擎
   - 作者: Peter Boncz et al.
   - 关键点: Vectorization的开创性工作

4. **"Morsel-Driven Parallelism: A NUMA-Aware Query Evaluation Framework" (2014)**
   - 主题: 并行查询执行
   - 会议: SIGMOD 2014
   - 关键点: 细粒度并行、动态负载均衡

5. **"Everything You Always Wanted to Know About Compiled and Vectorized Queries But Were Afraid to Ask" (2018)**
   - 主题: 编译执行 vs 向量化执行
   - 会议: VLDB 2018
   - 关键点: 两种执行模型的对比

6. **"Integrating Compression and Execution in Column-Oriented Database Systems" (2006)**
   - 主题: 列式存储和压缩
   - 会议: SIGMOD 2006
   - 关键点: 压缩算法、直接在压缩数据上执行

7. **"An Empirical Evaluation of Columnar Storage Formats" (2017)**
   - 主题: 列式存储格式对比
   - 会议: VLDB 2017
   - 关键点: Parquet, ORC, Arrow等格式

---

## 📚 推荐书籍

### 数据库系统

1. **"Database System Concepts" (7th Edition)**
   - 作者: Silberschatz, Korth, Sudarshan
   - 内容: 数据库系统全面教材
   - 相关章节: Query Processing, Query Optimization, Storage

2. **"Database Internals"**
   - 作者: Alex Petrov
   - 内容: 存储引擎和分布式数据库
   - 相关章节: B-Trees, LSM Trees, MVCC

3. **"Designing Data-Intensive Applications"**
   - 作者: Martin Kleppmann
   - 内容: 数据系统设计
   - 相关章节: Storage Engines, Encoding, Replication

---

### 查询优化

4. **"Architecture of a Database System"**
   - 作者: Hellerstein, Stonebraker, Hamilton
   - 内容: 数据库架构全景
   - 免费: http://db.cs.berkeley.edu/papers/fntdb07-architecture.pdf

5. **"Query Optimization" (Foundations and Trends in Databases)**
   - 作者: Surajit Chaudhuri
   - 内容: 查询优化综述
   - 关键点: Cost-based optimization, Join algorithms

---

### 向量化和性能

6. **"Computer Architecture: A Quantitative Approach"**
   - 作者: Hennessy, Patterson
   - 内容: 计算机体系结构
   - 相关章节: SIMD, Cache, Memory Hierarchy

---

## 🛠️ 开发工具

### 必备工具

1. **IDE/编辑器**
   - **VSCode** (推荐)
     - 扩展: C/C++, CMake Tools, GitLens
   - **CLion**
     - JetBrains的C++ IDE
   - **Vim/Neovim**
     - 配合coc.nvim或LSP

2. **调试器**
   - **GDB** (Linux)
     - 教程: https://sourceware.org/gdb/documentation/
   - **LLDB** (macOS)
     - 教程: https://lldb.llvm.org/use/tutorial.html

3. **性能分析**
   - **perf** (Linux)
     - Wiki: https://perf.wiki.kernel.org/
   - **Instruments** (macOS)
     - Xcode自带
   - **Valgrind**
     - 内存调试
     - URL: https://valgrind.org/

4. **构建系统**
   - **Make**
     - DuckDB默认构建工具
   - **Ninja**
     - 更快的构建 (`GEN=ninja make`)
   - **CMake**
     - 底层构建配置

---

### 推荐扩展

5. **DuckDB扩展**
   - **httpfs**: HTTP/S3文件系统
   - **parquet**: Parquet文件支持
   - **json**: JSON支持
   - **spatial**: 地理空间数据

6. **开发辅助**
   - **clang-format**: 代码格式化
   - **clang-tidy**: 静态分析
   - **cpplint**: Google C++风格检查

---

## 🎓 在线课程

### 数据库课程

1. **CMU 15-445/645: Database Systems**
   - 讲师: Andy Pavlo
   - URL: https://15445.courses.cs.cmu.edu/
   - 内容: 数据库系统全面课程
   - 包含: Lecture视频, Project (实现数据库)

2. **CMU 15-721: Advanced Database Systems**
   - 讲师: Andy Pavlo
   - URL: https://15721.courses.cs.cmu.edu/
   - 内容: 高级数据库主题
   - 包含: OLAP系统, Query Optimization

3. **Stanford CS245: Principles of Data-Intensive Systems**
   - URL: http://web.stanford.edu/class/cs245/
   - 内容: 数据密集型系统

---

### 系统编程

4. **CS:APP (Computer Systems: A Programmer's Perspective)**
   - 书籍配套课程
   - 内容: 系统级编程、内存、缓存

---

## 💡 学习技巧

### 有效学习方法

1. **主动学习**
   - ✅ 阅读源码并做笔记
   - ✅ 实现课程项目
   - ✅ 完成练习题
   - ❌ 只是被动阅读课程

2. **循序渐进**
   - Week 1 → Week 2 → Week 3 → Week 4
   - 不要跳过基础章节
   - 每周完成项目巩固

3. **理论结合实践**
   - 学习概念后立即写代码验证
   - 使用调试器单步执行理解流程
   - 性能测试对比优化效果

4. **利用工具**
   - GDB调试源码
   - perf分析性能
   - EXPLAIN查看执行计划

5. **社区互动**
   - Discord提问
   - GitHub阅读Issues/PRs
   - Stack Overflow查找答案

---

### 常见陷阱

1. ❌ 跳过基础直接看高级内容
2. ❌ 只看课程不写代码
3. ❌ 不做练习项目
4. ❌ 遇到困难就放弃
5. ❌ 不读DuckDB源码

---

## 🔗 相关项目

### 类似数据库系统

1. **SQLite**
   - URL: https://sqlite.org/
   - 特点: 嵌入式关系数据库
   - 对比: OLTP vs OLAP (DuckDB)

2. **MonetDB**
   - URL: https://www.monetdb.org/
   - 特点: 列式OLAP数据库
   - 关系: DuckDB的灵感来源

3. **Hyper**
   - 商业产品 (现为Tableau一部分)
   - 特点: 高性能OLAP引擎

4. **ClickHouse**
   - URL: https://clickhouse.com/
   - 特点: 列式OLAP数据库
   - 对比: 更适合大规模分布式场景

---

### 数据格式

5. **Apache Parquet**
   - URL: https://parquet.apache.org/
   - 关系: DuckDB原生支持Parquet

6. **Apache Arrow**
   - URL: https://arrow.apache.org/
   - 关系: DuckDB支持Arrow格式

---

## 📝 学习检查清单

### 第一周
- [ ] 理解Vector和DataChunk结构
- [ ] 能够创建和操作Vector
- [ ] 理解ValidityMask处理NULL
- [ ] 完成SimpleTable项目
- [ ] 练习题1-3

### 第二周
- [ ] 理解Pipeline执行模型
- [ ] 能够实现简单算子
- [ ] 理解Source/Sink区别
- [ ] 完成SimpleQueryEngine项目
- [ ] 练习题4-5

### 第三周
- [ ] 理解Filter Pushdown
- [ ] 掌握Join Order优化
- [ ] 理解向量化执行
- [ ] 完成SimpleOptimizer项目
- [ ] 练习题6-7

### 第四周
- [ ] 理解列式存储
- [ ] 掌握压缩算法
- [ ] 理解MVCC机制
- [ ] 完成SimpleDiskStorage项目
- [ ] 练习题8

### 最终
- [ ] 完成Mini-DuckDB项目
- [ ] 能够阅读DuckDB源码
- [ ] 能够调试DuckDB
- [ ] 能够开发DuckDB扩展

---

## 🎯 下一步行动

### 完成课程后可以:

1. **贡献DuckDB**
   - 修复Bug
   - 实现新功能
   - 改进文档
   - 提交PR: https://github.com/duckdb/duckdb/pulls

2. **深入研究**
   - 阅读学术论文
   - 研究其他数据库系统
   - 对比不同实现

3. **构建自己的项目**
   - 基于DuckDB的应用
   - 自定义扩展
   - 甚至自己的数据库原型

4. **分享知识**
   - 写博客
   - 做演讲
   - 帮助他人学习

---

## 📞 获取帮助

### 遇到问题时

1. **查阅本课程文档**
   - DuckDB调试指南.md
   - DuckDB术语表.md
   - DuckDB学习速查表.md

2. **搜索已知问题**
   - GitHub Issues
   - Stack Overflow
   - Discord历史消息

3. **阅读源码**
   - 相关的.cpp/.hpp文件
   - 单元测试代码

4. **提问**
   - Discord: 实时讨论
   - GitHub Issues: Bug报告
   - Stack Overflow: 技术问题

5. **调试**
   - 使用GDB/LLDB
   - 添加打印语句
   - 查看EXPLAIN输出

---

## 🌟 成功案例

### DuckDB被使用的场景

- **数据分析**: Pandas, Polars集成
- **数据科学**: Jupyter notebooks
- **BI工具**: Tableau, Superset
- **数据工程**: ETL pipeline
- **嵌入式分析**: 移动应用、桌面应用

学习DuckDB不仅是学习一个数据库，更是掌握现代OLAP系统的核心技术！

---

## 📅 保持更新

### 关注更新

- **DuckDB Blog**: 定期查看新文章
- **GitHub Watch**: 关注仓库更新
- **Discord**: 加入社区讨论
- **Twitter**: 关注@duckdb

---

**祝你学习愉快！Happy Learning! 🦆🚀**

---

最后更新：2026-01-17
