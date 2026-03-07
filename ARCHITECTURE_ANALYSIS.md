# Daft 架构分析文档

> 本文档深入分析 Daft 数据引擎的架构设计，帮助开发者理解系统的工作原理和代码组织方式。

## 目录

1. [架构概览](#架构概览)
2. [核心模块详解](#核心模块详解)
3. [数据流分析](#数据流分析)
4. [关键技术决策](#关键技术决策)
5. [扩展机制](#扩展机制)

---

## 架构概览

### 系统定位

Daft 是一个**混合型 Rust/Python 数据引擎**，专为 AI 和多模态工作负载设计：

- **高性能**：Rust 核心提供零成本抽象和并行执行
- **易用性**：Python API 提供直观的数据处理接口
- **多模态**：原生支持图像、音频、视频等非结构化数据
- **分布式**：支持本地和 Ray 分布式执行

### 技术栈分层

```
┌─────────────────────────────────────────────────────────────┐
│                     Python API Layer                        │
│  ┌──────────┬──────────┬──────────┬──────────┬──────────┐  │
│  │ DataFrame│Expression│  Series  │ Schema   │  UDF     │  │
│  └──────────┴──────────┴──────────┴──────────┴──────────┘  │
├─────────────────────────────────────────────────────────────┤
│                  PyO3 Binding Layer                         │
│            (Rust-Python 互操作，零拷贝数据传递)              │
├─────────────────────────────────────────────────────────────┤
│                     Rust Core Engine                        │
│  ┌──────────┬──────────┬──────────┬──────────┬──────────┐  │
│  │  Core    │  DSL     │ Logical  │  Local   │   I/O    │  │
│  │ DataType │  Expr    │   Plan   │ Execution│  Scan    │  │
│  └──────────┴──────────┴──────────┴──────────┴──────────┘  │
├─────────────────────────────────────────────────────────────┤
│                   Storage & Network                         │
│       (Parquet, CSV, JSON, S3, GCS, Azure, HTTP)           │
└─────────────────────────────────────────────────────────────┘
```

---

## 核心模块详解

### 1. 数据核心层 (daft-core)

**位置**: `src/daft-core/`

核心数据结构定义，是整个系统的基础：

```rust
// 主要组件
pub mod array;      // Arrow 数组封装
pub mod datatypes;  // DataType 类型系统
pub mod series;     // Series 列数据抽象
pub mod kernels;    // 计算内核
```

**关键设计**:

- **Series**: 列式数据抽象，内部使用 Arrow 数组
- **DataType**: 扩展的 Arrow 类型系统，支持嵌套类型、图像、张量等
- **零拷贝**: 与 Python/PyArrow 共享内存

### 2. 表达式系统 (daft-dsl)

**位置**: `src/daft-dsl/`

领域特定语言（DSL）实现，提供声明式查询接口：

```rust
pub mod expr;       // Expression 表达式树
pub mod functions;  // 函数注册表
pub mod optimization; // 表达式优化
```

**表达式树结构**:

```
Expr
├── Column(String)              # 列引用
├── Literal(ScalarValue)        # 字面量
├── BinaryOp { op, left, right } # 二元操作
├── Function { name, args }     # 函数调用
├── Agg(AggExpr)               # 聚合表达式
└── Window(WindowExpr)         # 窗口表达式
```

**函数系统架构**:

```rust
// 统一函数接口
trait ScalarUDF: Send + Sync {
    fn name(&self) -> &'static str;
    fn call(&self, inputs: FunctionArgs<Series>, ctx: &EvalContext) -> DaftResult<Series>;
    fn get_return_field(&self, inputs: FunctionArgs<ExprRef>, schema: &Schema) -> DaftResult<Field>;
}
```

### 3. 查询计划层

#### 3.1 逻辑计划 (daft-logical-plan)

**位置**: `src/daft-logical-plan/`

平台无关的查询逻辑表示：

```rust
pub enum LogicalPlan {
    Source(SourceInfo),      // 数据源
    Project(Vec<ExprRef>),   // 投影
    Filter(ExprRef),         // 过滤
    Aggregate { ... },       // 聚合
    Join { ... },            // 连接
    Sort { ... },            // 排序
    Limit { ... },           // 限制
    Explode { ... },         // 展开
    Distinct,               // 去重
    ...
}
```

**优化器**:

- 谓词下推（Predicate Pushdown）
- 投影下推（Projection Pushdown）
- 常量折叠（Constant Folding）
- 分区裁剪（Partition Pruning）

#### 3.2 本地执行计划 (daft-local-plan)

**位置**: `src/daft-local-plan/`

将逻辑计划转换为可执行的物理计划：

```rust
pub enum LocalPlan {
    PhysicalScan(PhysicalScan),
    Project(Project),
    Filter(Filter),
    HashAggregate(HashAggregate),
    Sort(Sort),
    HashJoin(HashJoin),
    ...
}
```

### 4. 执行引擎 (daft-local-execution)

**位置**: `src/daft-local-execution/`

基于异步流水线的执行引擎：

```
┌─────────────────────────────────────────────────────────────┐
│                    Pipeline Architecture                      │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐  │
│  │ Source  │───▶│  Op 1   │───▶│  Op 2   │───▶│  Sink   │  │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘  │
│       │              │              │              │        │
│       ▼              ▼              ▼              ▼        │
│   ┌──────┐      ┌──────┐      ┌──────┐      ┌──────┐       │
│   │Batch │      │Batch │      │Batch │      │Output│       │
│   └──────┘      └──────┘      └──────┘      └──────┘       │
└─────────────────────────────────────────────────────────────┘
```

**关键组件**:

- **Sources**: 数据输入（Scan, InMemory）
- **Intermediate Ops**: 中间操作（Filter, Project, Join）
- **Sinks**: 数据输出（Write, Collect）
- **Streaming Sink**: 流式输出支持

**内存管理**:

```rust
pub struct MemoryManager {
    // 基于权重的内存分配
    request_bytes(bytes: u64) -> MemoryPermit
}
```

### 5. I/O 系统 (daft-io)

**位置**: `src/daft-io/`

统一的对象存储接口：

```rust
// 支持的存储后端
enum SourceType {
    File(PathBuf),
    S3 { bucket, key },
    GCS { bucket, key },
    Azure { account, container, path },
    HTTP(Url),
}
```

**特性**:

- 基于 OpenDAL 的统一接口
- 异步 I/O 与 Tokio 运行时
- 自动重试和限流
- 缓存和预取优化

### 6. 文件格式支持

| 模块 | 位置 | 功能 |
|------|------|------|
| daft-parquet | `src/daft-parquet/` | Parquet 读写，谓词下推 |
| daft-csv | `src/daft-csv/` | CSV 读写，类型推断 |
| daft-json | `src/daft-json/` | JSON/JSONL 读写 |
| daft-warc | `src/daft-warc/` | WARC 网络存档格式 |

### 7. 函数库 (daft-functions-*)

**位置**: `src/daft-functions-*/`

模块化函数实现：

```
daft-functions-utf8/    # 字符串函数 (length, split, replace...)
daft-functions-list/    # 列表函数 (append, slice, contains...)
daft-functions-json/    # JSON 函数 (jq 查询)
daft-functions-temporal/ # 时间函数 (date_trunc, extract...)
daft-functions-binary/  # 二进制函数
daft-functions-uri/     # URI 处理函数
daft-functions-tokenize/ # 分词函数
daft-image/            # 图像处理函数
```

### 8. 目录系统 (daft-catalog)

**位置**: `src/daft-catalog/`

统一的元数据访问层：

```rust
trait Catalog {
    fn list_tables(&self) -> Vec<Table>;
    fn get_table(&self, name: &str) -> Result<Table>;
}

// 支持的目录
- Iceberg
- Delta Lake
- Lance
- Unity Catalog
- Glue
```

### 9. SQL 引擎 (daft-sql)

**位置**: `src/daft-sql/`

基于 sqlparser 的 SQL 解析和执行：

```rust
pub fn parse_sql(sql: &str) -> Result<LogicalPlan>
pub fn execute_sql(sql: &str) -> Result<DataFrame>
```

---

## 数据流分析

### 查询生命周期

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│   User   │────▶│  Python  │────▶│   Rust   │────▶│  Result  │
│  Query   │     │    API   │     │  Engine  │     │  Return  │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
      │                                │
      ▼                                ▼
# Python 层                      # Rust 层
df = daft.read_parquet(...)      LogicalPlanBuilder
    .filter(col('x') > 0)    ──▶   .filter(...)
    .select(col('y'))             .project(...)
    .collect()                    NativeExecutor::run()
```

### 内存布局

```
Python                                        Rust
┌─────────────────┐                          ┌─────────────────┐
│   PyArrow Array │◄────────────────────────►│   Arrow Array   │
│   (Python 对象) │   共享内存 (零拷贝)       │   (Rust 对象)   │
└─────────────────┘                          └─────────────────┘
       │                                            │
       ▼                                            ▼
┌─────────────────┐                          ┌─────────────────┐
│  Daft Series    │    PyO3 绑定              │  Daft Series    │
│  (Python 包装)  │◄────────────────────────►│  (Rust 原生)    │
└─────────────────┘                          └─────────────────┘
```

---

## 关键技术决策

### 1. 为什么选择 Rust？

| 优势 | 说明 |
|------|------|
| 性能 | 零成本抽象，接近 C/C++ 的性能 |
| 安全 | 内存安全保证，无数据竞争 |
| 并发 |  fearless 并发，易于编写高性能并行代码 |
| 生态 | Arrow-rs, Tokio, PyO3 等成熟库 |

### 2. 为什么保留 Python API？

- **生态整合**: 与 PyData 生态（pandas, numpy, PyTorch）无缝集成
- **开发效率**: Python 的快速迭代能力
- **用户友好**: 数据科学家熟悉的接口

### 3. 架构设计原则

1. **列式存储**: Arrow 格式最大化 SIMD 效率
2. **延迟执行**: 构建查询计划后批量执行
3. **流式处理**: 支持超出内存的数据集
4. **类型安全**: 编译时类型检查减少运行时错误

### 4. 性能优化策略

```rust
// 1. SIMD 向量化
pub fn numeric_add(&self, other: &Series) -> Series {
    // 使用 Arrow 的 SIMD 内核
}

// 2. 内存池
pub struct MemoryPool {
    // 重用内存分配
}

// 3. 延迟物化
pub fn filter_pushdown(&mut self) {
    // 将过滤条件下推到数据源
}

// 4. 并行执行
pub fn parallel_scan(&self) -> impl ParallelIterator {
    // 多线程数据扫描
}
```

---

## 扩展机制

### 1. 添加新函数

以字符串函数为例：

```rust
// 1. 在 daft-functions-utf8/src/ 创建模块
// src/daft-functions-utf8/src/my_function.rs

#[derive(Clone, Serialize, Deserialize, PartialEq, Eq, Hash)]
pub struct MyFunction;

#[typetag::serde]
impl ScalarUDF for MyFunction {
    fn name(&self) -> &'static str { "my_function" }
    
    fn call(&self, inputs: FunctionArgs<Series>, _ctx: &EvalContext) -> DaftResult<Series> {
        // 实现逻辑
    }
    
    fn get_return_field(&self, inputs: FunctionArgs<ExprRef>, schema: &Schema) -> DaftResult<Field> {
        // 返回类型推断
    }
}

// 2. 在 lib.rs 中注册
parent.add_fn(MyFunction);
```

### 2. 添加新数据源

```rust
// 1. 实现 ScanOperator
trait ScanOperator {
    fn schema(&self) -> Schema;
    fn partitioning_keys(&self) -> Vec<&str>;
    fn execute(&self) -> impl Iterator<Item = Result<RecordBatch>>;
}

// 2. 在 daft-scan 中注册
```

### 3. 自定义 UDF

```python
import daft
from daft import udf

@udf(return_dtype=daft.DataType.float())
def my_udf(x: daft.Series) -> daft.Series:
    # 自定义逻辑
    return x * 2

df = df.with_column("doubled", my_udf(df["value"]))
```

---

## 模块依赖图

```
daft-core
    ├── daft-dsl
    │       ├── daft-functions-*
    │       └── daft-sql
    ├── daft-logical-plan
    │       └── daft-local-plan
    │               └── daft-local-execution
    ├── daft-scan
    │       ├── daft-parquet
    │       ├── daft-csv
    │       └── daft-json
    └── daft-io

daft-catalog
    ├── daft-iceberg
    ├── daft-delta
    └── daft-lance
```

---

## 可运行示例

以下是直接从测试中提取的可运行代码示例，帮助理解 Daft 的使用方式。

### 示例 1: DataFrame 基本操作

```python
# 文件: examples/dataframe_basics.py
# 运行: python examples/dataframe_basics.py

from __future__ import annotations
import daft
from daft import col, lit

# 1. 从字典创建 DataFrame
df = daft.from_pydict({
    "name": ["Alice", "Bob", "Charlie", "Diana"],
    "age": [25, 30, 35, 28],
    "salary": [50000.0, 60000.0, 75000.0, 65000.0],
    "department": ["Engineering", "Sales", "Engineering", "Marketing"]
})

print("=== 原始数据 ===")
df.show()

# 2. 过滤数据
print("\n=== 过滤: age > 28 ===")
df.filter(col("age") > 28).show()

# 3. 选择列
print("\n=== 选择 name 和 salary 列 ===")
df.select(col("name"), col("salary")).show()

# 4. 添加计算列
print("\n=== 添加 bonus 列 (salary * 0.1) ===")
df.with_column("bonus", col("salary") * lit(0.1)).show()

# 5. 聚合操作
print("\n=== 按部门统计平均薪资 ===")
df.groupby(col("department")).agg(
    daft.mean(col("salary")).alias("avg_salary"),
    daft.count(col("name")).alias("employee_count")
).show()

# 6. 排序
print("\n=== 按年龄降序排列 ===")
df.sort(col("age"), desc=True).show()

# 7. 收集结果到 Python
collected = df.collect().to_pydict()
print("\n=== 收集为 Python 字典 ===")
print(collected)
```

### 示例 2: Series 字符串操作

```python
# 文件: examples/series_string_ops.py
# 运行: python examples/series_string_ops.py

from __future__ import annotations
from daft import Series

# 创建字符串 Series
s = Series.from_pylist(["hello", "WORLD", "RustPython", "DaftDataFrame", None])

print("=== 原始 Series ===")
print(s.to_pylist())

# 1. 转小写
print("\n=== 转小写 (lower) ===")
print(s.str.lower().to_pylist())

# 2. 转大写
print("\n=== 转大写 (upper) ===")
print(s.str.upper().to_pylist())

# 3. 首字母大写
print("\n=== 首字母大写 (capitalize) ===")
print(s.str.capitalize().to_pylist())

# 4. 字符串长度
print("\n=== 字符串长度 (length) ===")
print(s.str.length().to_pylist())

# 5. 包含子串
print("\n=== 包含 'Daft' ===")
pattern = Series.from_pylist(["Daft"])
print(s.str.contains(pattern).to_pylist())

# 6. 分割字符串
print("\n=== 按 '_' 分割 ===")
s2 = Series.from_pylist(["hello_world", "foo_bar_baz"])
print(s2.str.split("_").to_pylist())
```

### 示例 3: RecordBatch (MicroPartition) 底层操作

```python
# 文件: examples/recordbatch_ops.py
# 运行: python examples/recordbatch_ops.py

from __future__ import annotations
from daft.expressions import col
from daft.recordbatch import MicroPartition

# 1. 从字典创建 RecordBatch
table = MicroPartition.from_pydict({
    "col": ["foo", None, "barBaz", "quux", "1"],
    "num": [1, 2, 3, 4, 5]
})

print("=== 原始 RecordBatch ===")
print(table.to_pydict())

# 2. 评估表达式 (capitalize)
print("\n=== 执行 capitalize 表达式 ===")
result = table.eval_expression_list([col("col").capitalize()])
print(result.to_pydict())

# 3. 评估数值表达式
print("\n=== 执行数值表达式 (num * 2) ===")
result = table.eval_expression_list([col("num") * lit(2)])
print(result.to_pydict())

# 4. 过滤
print("\n=== 过滤 (num > 2) ===")
filtered = table.filter(col("num") > lit(2))
print(filtered.to_pydict())

# 5. Schema 信息
print("\n=== Schema 信息 ===")
print(table.schema())
```

### 示例 4: SQL 查询

```python
# 文件: examples/sql_example.py
# 运行: python examples/sql_example.py

from __future__ import annotations
import daft
from daft import sql

# 创建示例数据
df = daft.from_pydict({
    "x": [1, 2, 3, 4, 5],
    "y": [10, 20, 30, 40, 50],
    "z": ["a", "b", "c", "d", "e"]
})

# 注册为临时表
df.create_temp_table("my_table")

# 1. 简单查询
print("=== SQL 查询: SELECT * ===")
result = sql("SELECT * FROM my_table WHERE x > 2")
result.show()

# 2. 聚合查询
print("\n=== SQL 聚合 ===")
result = sql("""
    SELECT 
        COUNT(*) as count,
        AVG(y) as avg_y,
        MAX(x) as max_x
    FROM my_table
""")
result.show()

# 3. 条件查询
print("\n=== SQL 条件查询 ===")
result = sql("SELECT x, y FROM my_table WHERE z = 'c' OR z = 'd'")
result.show()
```

### 示例 5: 表达式操作

```python
# 文件: examples/expression_ops.py
# 运行: python examples/expression_ops.py

from __future__ import annotations
from daft.expressions import col, lit
from daft.recordbatch import MicroPartition
from daft.datatype import DataType

# 创建测试数据
table = MicroPartition.from_pydict({
    "a": [1, 2, 3, 4, 5],
    "b": [10.0, 20.0, 30.0, 40.0, 50.0],
    "name": ["Alice", "Bob", "Charlie", "Diana", "Eve"]
})

print("=== 算术表达式 ===")
result = table.eval_expression_list([
    (col("a") + col("b")).alias("add"),
    (col("a") * lit(2)).alias("multiply"),
    (col("b") / lit(10)).alias("divide"),
])
print(result.to_pydict())

print("\n=== 比较表达式 ===")
result = table.eval_expression_list([
    (col("a") > lit(3)).alias("gt_3"),
    (col("a") == lit(2)).alias("eq_2"),
])
print(result.to_pydict())

print("\n=== 数学函数 ===")
result = table.eval_expression_list([
    col("b").abs().alias("abs"),
    col("b").round().alias("round"),
    col("a").sqrt().alias("sqrt"),
])
print(result.to_pydict())

print("\n=== 字符串函数 ===")
result = table.eval_expression_list([
    col("name").str.upper().alias("upper"),
    col("name").str.length().alias("length"),
])
print(result.to_pydict())
```

### 如何运行示例

```bash
# 1. 确保环境已设置
source .venv/bin/activate

# 2. 创建示例目录
mkdir -p examples

# 3. 将示例代码保存到文件并运行
python examples/dataframe_basics.py

# 4. 或使用交互式 Python
ipython -i examples/dataframe_basics.py
```

### 调试技巧

```python
# 启用详细日志
import logging
logging.basicConfig(level=logging.DEBUG)
import daft
daft.refresh_logger()

# 查看执行计划
df.explain()

# 查看优化后的计划
df.explain(show_optimized=True)

# 逐步调试
df2 = df.filter(col("age") > 28)
print(df2._builder)  # 查看逻辑计划
```

---

## 参考资源

- [Daft 官方文档](https://docs.daft.ai)
- [Arrow 格式规范](https://arrow.apache.org/docs/format/Columnar.html)
- [PyO3 文档](https://pyo3.rs)
- [Tokio 文档](https://tokio.rs)

---

*最后更新: 2026-03-07*
