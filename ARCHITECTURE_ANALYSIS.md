# Daft 架构深度分析

> 文档生成日期：2026-03-19
> 代码库版本：`ai-1` 分支

---

## 目录

1. [项目概述](#1-项目概述)
2. [整体架构](#2-整体架构)
3. [查询执行流水线](#3-查询执行流水线)
4. [Rust 核心层详解](#4-rust-核心层详解)
5. [Python API 层详解](#5-python-api-层详解)
6. [逻辑计划与优化器](#6-逻辑计划与优化器)
7. [执行引擎](#7-执行引擎)
8. [I/O 层与存储后端](#8-io-层与存储后端)
9. [函数库体系](#9-函数库体系)
10. [扩展系统](#10-扩展系统)
11. [依赖关系图](#11-依赖关系图)
12. [核心设计模式](#12-核心设计模式)
13. [关键设计决策分析](#13-关键设计决策分析)

---

## 1. 项目概述

Daft 是一个**高性能分布式 DataFrame 引擎**，设计目标是在 Python 生态系统中提供接近原生性能的大规模数据处理能力。

### 核心特性

| 特性 | 描述 |
|------|------|
| **多语言实现** | Python API + Rust 核心，通过 PyO3 桥接 |
| **懒求值** | 操作构建逻辑计划，触发执行时才运算 |
| **多执行模式** | 本地 Native 执行 + Ray 分布式执行 |
| **优化器** | 31 条基于规则的查询优化 pass |
| **统一 I/O** | 26+ 存储后端（S3、GCS、Azure、本地等） |
| **SQL 支持** | 内置 SQL 解析器和执行层 |

### 技术栈

```
Python 3.8+  ←→  PyO3 0.28  ←→  Rust (stable)
                                    ↓
                           Apache Arrow 57.1.0
                           Tokio 1.48.0 (async)
                           OpenDAL 0.55 (I/O)
```

---

## 2. 整体架构

Daft 的代码库分为两大部分：Python API 层和 Rust 核心层，通过 PyO3 + Maturin 编译为 `.so` 动态库（`daft.daft` 模块）。

```
┌─────────────────────────────────────────────────────────────┐
│                     用户代码 (Python)                        │
│  df = daft.read_parquet("s3://...")                         │
│     .filter(col("age") > 18)                                │
│     .select("name", "age")                                  │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    Python API 层                             │
│  daft/dataframe/dataframe.py  (239KB)                       │
│  daft/expressions/expressions.py  (102KB)                   │
│  daft/functions/  (26 modules)                              │
│  daft/io/  (35+ I/O modules)                                │
└──────────────────────────┬──────────────────────────────────┘
                           │  PyO3 FFI
┌──────────────────────────▼──────────────────────────────────┐
│                    Rust 核心层 (daft.abi3.so)                │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐ │
│  │ 逻辑计划层  │  │  优化器层    │  │    物理计划层      │ │
│  │daft-logical-│  │ 31条规则     │  │ daft-local-plan    │ │
│  │plan         │  │ 优化 pass    │  │ daft-distributed   │ │
│  └─────────────┘  └──────────────┘  └────────────────────┘ │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐ │
│  │  执行引擎   │  │   数据层     │  │     I/O 层         │ │
│  │daft-local-  │  │ daft-core    │  │ daft-io (26 后端)  │ │
│  │execution    │  │ daft-schema  │  │ daft-parquet       │ │
│  │daft-distrib.│  │ daft-dsl     │  │ daft-csv           │ │
│  └─────────────┘  └──────────────┘  └────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. 查询执行流水线

一个 Daft 查询从 Python API 到最终结果，经历以下 6 个阶段：

```
阶段 1: Python API 调用
    df.filter(col("age") > 18).select("name")
         ↓
阶段 2: 构建逻辑计划 (LazyFrame 语义)
    LogicalPlanBuilder → LogicalPlan (Rust 树结构)
    Project
    └── Filter (age > 18)
        └── Source (parquet scan)
         ↓
阶段 3: 规则优化器 (31 条规则)
    - push_down_filter: 谓词下推到 Scan
    - push_down_projection: 投影列裁剪
    - eliminate_cross_join: 消除笛卡尔积
    - 等 28+ 条规则
         ↓
阶段 4: 翻译为物理计划
    ┌─────────────────────┐
    │  Native Runner?     │ → LocalPhysicalPlan (daft-local-plan)
    │  Ray Runner?        │ → DistributedPipeline (daft-distributed)
    └─────────────────────┘
         ↓
阶段 5: 执行
    Native:       Tokio 异步流水线执行
                  Sources → IntermediateOps → Sinks
                  Morsel 驱动 (批量 micro-partition)

    Ray:          任务调度到 Ray Worker
                  每个 Worker 使用 LocalExecutionEngine
         ↓
阶段 6: 结果收集
    Vec<MicroPartition> → Python RecordBatch → Pandas/Arrow/...
```

### 关键数据流单元

| 单元 | 定义 | 用途 |
|------|------|------|
| `Series` | 单列数据，Arrow 内存格式 | 最细粒度计算单元 |
| `RecordBatch` | 一组命名 `Series` | 单 partition 表示 |
| `MicroPartition` | 包裹多个 `RecordBatch` | 运行时数据单元，跨网络传输 |

---

## 4. Rust 核心层详解

### 4.1 Crate 总览（54 个）

Rust 代码库分为 5 个层次：

#### 层次 1：数据类型层

```
src/daft-schema/          # DataType 定义，Schema 管理
src/daft-core/            # DataArray<T>、Series（Arrow 封装）
src/daft-recordbatch/     # RecordBatch（命名 Series 集合）
src/daft-micropartition/  # MicroPartition（运行时数据单元）
```

`daft-core` 的核心类型层次：

```
DataType (enum)  ─┐
                  ├→ DataArray<T>  →  Series  →  RecordBatch  →  MicroPartition
Arrow Array ──────┘
```

`src/daft-core/src/series/ops/` 包含 38 个操作模块（数学、比较、聚合、时间、字符串等）。

#### 层次 2：DSL 与表达式层

```
src/daft-dsl/
├── expr/          # Expr enum 定义（Column、Literal、BinaryOp、Function 等）
├── functions/     # 函数注册表
├── arithmetic/    # 算术表达式求值
├── python_udf/    # Python UDF 集成
└── visitor.rs     # 表达式树遍历
```

`Expr` 枚举是表达式系统的核心，表示所有可能的计算：

```rust
pub enum Expr {
    Column(Arc<str>),
    Literal(LiteralValue),
    BinaryOp { op: Operator, left, right },
    Function { func: FunctionExpr, inputs },
    Agg(AggExpr),
    WindowFunction(WindowExpr),
    ...
}
```

#### 层次 3：计划与优化层

见[第 6 节详解](#6-逻辑计划与优化器)。

#### 层次 4：执行层

见[第 7 节详解](#7-执行引擎)。

#### 层次 5：公共工具层

```
src/common/
├── error/           # 错误类型系统
├── io-config/       # I/O 配置（S3 凭证等）
├── file-formats/    # 文件格式枚举
├── runtime/         # Tokio 异步运行时封装
├── partitioning/    # 分区策略
├── resource-request/# 资源规格（CPU/内存/GPU）
├── metrics/         # 性能指标采集
├── tracing/         # OpenTelemetry 追踪
├── macros/          # 过程宏
└── treenode/        # 树节点遍历工具
```

### 4.2 关键依赖

```toml
# 内存模型
arrow = "57.1.0"       # Apache Arrow 列式内存格式
parquet = "57.1.0"     # Parquet 格式（与 Arrow 同版本）

# 异步运行时
tokio = "1.48.0"       # 异步执行框架
futures = "0.3.30"     # Future 组合子

# Python 绑定
pyo3 = "0.28.2"        # Rust-Python FFI

# 并行计算
rayon = "*"            # 数据并行
dashmap = "*"          # 并发 HashMap

# 存储
opendal = "0.55"       # 统一存储抽象层
jemalloc = "*"         # 高性能内存分配器
```

---

## 5. Python API 层详解

### 5.1 入口结构

```
daft/
├── __init__.py            # 公共 API 重导出（read_*, col, lit, etc.）
├── dataframe/
│   └── dataframe.py       # DataFrame 类（239KB，核心文件）
├── expressions/
│   └── expressions.py     # Expression 类（102KB）
├── functions/             # 26 个内置函数模块
├── io/                    # 35+ I/O 读写模块
├── series.py              # Series Python 封装
├── datatype.py            # DataType 定义（58KB）
├── context.py             # Session 管理（14KB）
├── runners/               # 运行器配置
├── catalog/               # Catalog 类型
├── sql/                   # sql() 和 sql_expr() 入口
├── udf/                   # UDF 装饰器
└── daft.abi3.so           # 编译的 Rust 扩展
```

### 5.2 DataFrame 类

`daft/dataframe/dataframe.py` 是用户主要交互界面：

- **懒求值**：所有转换操作返回新 `DataFrame` 对象，不触发计算
- **触发执行**：`.collect()`、`.to_pandas()`、`.show()` 等操作触发执行
- **计划构建**：每次操作调用 `LogicalPlanBuilder` 在 Rust 侧构建节点

主要方法分类：

| 类别 | 方法 |
|------|------|
| 读取 | `read_parquet`、`read_csv`、`read_json`、`read_delta`、`from_pydict` 等 |
| 投影 | `select`、`with_column`、`exclude`、`rename` |
| 过滤 | `filter`、`limit`、`sample` |
| 聚合 | `agg`、`groupby`、`sum`、`count` 等 |
| Join | `join`、`cross_join`、`semi_join`、`anti_join` |
| 排序 | `sort`、`top_k` |
| 重分区 | `repartition`、`into_partitions` |
| 窗口 | `with_column` + 窗口表达式 |
| 写出 | `write_parquet`、`write_csv`、`write_delta`、`write_iceberg` |
| 执行 | `collect`、`to_pandas`、`show`、`explain` |

### 5.3 Expression 系统

`daft/expressions/expressions.py` 提供链式 API：

```python
# 示例表达式
col("date").dt.year()           # 时间访问器
col("name").str.upper()         # 字符串访问器
col("tags").list.lengths()      # 列表访问器
col("data").struct.get("field") # 结构体访问器
```

命名空间设计：

| 命名空间 | Python 访问器 | 对应 Rust crate |
|----------|--------------|----------------|
| 时间函数 | `.dt.*` | `daft-functions-temporal` |
| 字符串函数 | `.str.*` | `daft-functions-utf8` |
| 列表函数 | `.list.*` | `daft-functions-list` |
| 结构体函数 | `.struct.*` | `daft-functions` |
| JSON 函数 | `.json.*` | `daft-functions-json` |
| URL 函数 | `.url.*` | `daft-functions-uri` |
| 二进制函数 | `.binary.*` | `daft-functions-binary` |
| 图像函数 | `.image.*` | `daft-image` |

### 5.4 函数库模块

`daft/functions/` 目录包含 26 个模块：

```
agg.py          聚合函数 (sum, count, mean, stddev, ...)
datetime.py     日期时间 (58KB)
str.py          字符串 (50KB)
list.py         列表/数组 (23KB)
misc.py         杂项 (34KB，hash, monotonically_increasing_id 等)
numeric.py      数值函数
binary.py       二进制数据
bitwise.py      位运算
struct.py       结构体操作
url.py          URL 处理
image.py        图像处理
audio.py        音频处理
video.py        视频处理
llm.py          大语言模型集成
distance.py     向量距离
similarity.py   相似度计算
columnar.py     列操作
process.py      进程/系统调用
file_.py        文件操作
...
```

---

## 6. 逻辑计划与优化器

### 6.1 逻辑计划算子（27 种）

`src/daft-logical-plan/src/ops/` 目录包含所有逻辑算子：

| 算子 | 文件 | 功能 |
|------|------|------|
| Source | `source.rs` | 数据读取（Scan、InMemory 等） |
| Filter | `filter.rs` | 行过滤（WHERE 子句） |
| Project | `project.rs` (31KB) | 列投影（SELECT） |
| Agg | `agg.rs` | 聚合（GROUP BY + 聚合函数） |
| Join | `join.rs` (14KB) | 各类 Join |
| Sort | `sort.rs` | 排序 |
| Limit | `limit.rs` | 行数限制 |
| Repartition | `repartition.rs` | 重分区 |
| Explode | `explode.rs` | 展开数组/结构体 |
| Unpivot | `unpivot.rs` | 行列转换 |
| Pivot | `pivot.rs` | 列行转换 |
| Distinct | `distinct.rs` | 去重 |
| Concat | `concat.rs` | 并集合并 |
| SetOperations | `set_operations.rs` (19KB) | UNION/INTERSECT/EXCEPT |
| TopN | `top_n.rs` | Top-K 优化 |
| Window | `window.rs` (7KB) | 窗口函数 |
| Sink | `sink.rs` | 数据写出 |
| UDF | `udf.rs` | 用户自定义函数 |
| Offset | `offset.rs` | 行偏移（OFFSET 子句） |
| Sample | `sample.rs` | 采样 |
| Shard | `shard.rs` | 数据分片 |
| IntoBatches | `into_batches.rs` | 批次重组 |
| IntoPartitions | `into_partitions.rs` | 分区重组 |
| Summarize | `summarize.rs` | 窗口摘要 |
| MonotonicallyIncreasingId | `monotonically_increasing_id.rs` | 全局唯一 ID |
| VLLM | `vllm.rs` | vLLM 模型推理集成 |

### 6.2 优化器（31 条规则）

优化器位于 `src/daft-logical-plan/src/optimization/rules/`，采用**树节点遍历**模式，从下到上、从上到下多趟重写计划。

**关键规则分析：**

#### 谓词下推 (`push_down_filter.rs`, 52KB — 最大规则文件)

```
原始计划:
  Filter (age > 18)
  └── Join (user_id)
      ├── Users
      └── Orders

优化后:
  Join (user_id)
  ├── Filter (age > 18)   ← 谓词下推到 Users
  │   └── Users
  └── Orders
```

#### 投影下推 (`push_down_projection.rs`, 40KB)

消除不必要的列读取，减少 I/O 和内存使用。

#### 消除笛卡尔积 (`eliminate_cross_join.rs`, 26KB)

将 `CROSS JOIN + WHERE` 转化为等价的 `INNER JOIN`。

#### 分裂 UDF (`split_udfs.rs`, 59KB — 最大优化文件)

将含有 UDF 的复合投影拆分，使 UDF 可以独立调度和并行执行。

#### 连接重排序 (`reorder_joins/`)

基于统计信息选择最优的 Join 执行顺序（最小化中间结果大小）。

#### 限制下推 (`push_down_limit.rs`, 18KB)

将 LIMIT 尽量下推，减少不必要的数据处理。

#### 其他重要规则

| 规则文件 | 功能 |
|----------|------|
| `granular_projections.rs` | 细粒度投影优化 |
| `push_down_aggregation.rs` | 聚合下推 |
| `push_down_anti_semi_join.rs` | Anti/Semi Join 优化 |
| `simplify_expressions.rs` | 表达式化简（常量折叠等） |
| `extract_window_function.rs` | 窗口函数提取 |
| `unnest_subquery.rs` | 子查询展平 |
| `enrich_with_stats.rs` | 注入统计信息 |
| `rewrite_count_distinct.rs` | COUNT DISTINCT 改写 |

---

## 7. 执行引擎

### 7.1 Native Runner（本地执行引擎）

`src/daft-local-execution/` 实现基于 **Morsel 驱动**的异步流水线执行。

#### 核心概念：Morsel 驱动执行

```
Morsel = 小批量 MicroPartition（自适应大小）

Source ──[morsel]──→ IntermediateOp ──[morsel]──→ Sink
         (生产者)                                  (消费者)

每个 morsel 通过 channel 传递，实现反压（backpressure）
```

#### 执行组件分类

**Sources（数据源，8 种）**

```
empty_scan         空扫描
glob_scan          文件系统 Glob 扫描（Parquet/CSV/JSON 等）
in_memory          内存数据源
scan_task          Scan Task 执行器
flight_shuffle_read Arrow Flight 混洗读取
...
```

**Intermediate Operators（流式算子，9 种）**

```
project                  列投影（无状态）
filter                   行过滤（无状态）
explode                  数组展开
unpivot                  行列转换
into_batches             批次整形
udf                      UDF 执行
distributed_actor_pool_project  分布式 Actor Pool UDF
```

**Join Implementations（Join 算子，11 种）**

```
hash_join            哈希 Join（主要实现）
sort_merge_join      排序合并 Join（有序输入）
inner_join           内连接
left_right_join      左/右外连接
outer_join           全外连接
anti_semi_join       Anti/Semi Join
cross_join           笛卡尔积
```

**Sinks（汇算子，19 种）**

```
aggregate            全局聚合（阻塞）
grouped_aggregate    分组聚合（阻塞）
sort                 全局排序（阻塞）
top_n                Top-K（阻塞）
dedup                去重
pivot                透视
repartition          重分区
into_partitions      分区重组
window_*             窗口函数系列
commit_write         写出提交
flight_shuffle_write Arrow Flight 混洗写出
```

#### 资源管理

`resource_manager.rs` (7KB) 管理：
- 内存配额控制
- CPU 核心分配
- GPU 资源调度（用于 AI/向量化工作负载）

### 7.2 Ray Runner（分布式执行引擎）

`src/daft-distributed/` 负责 Ray 集群上的分布式执行。

#### 分布式流水线节点（31 种）

与逻辑计划算子一一对应，每个节点负责在 Ray Worker 上调度执行。

#### 执行模式

```
Driver (Python 进程)
  ↓
DistributedPhysicalPlan  ←  daft-distributed (Rust)
  ↓
Ray 任务调度
  ↓
Ray Workers × N
  ↓
各 Worker 执行 LocalExecutionEngine (daft-local-execution)
  ↓
结果通过 Arrow Flight 传输回 Driver
```

### 7.3 两种执行模式对比

| 维度 | Native Runner | Ray Runner |
|------|--------------|-----------|
| 适用场景 | 单机/多核 | 多节点集群 |
| 通信机制 | 共享内存 Channel | Arrow Flight |
| 调度粒度 | Morsel 级 | Task 级 |
| 故障恢复 | 无 | Ray 任务重试 |
| 启动开销 | 极低 | 需要 Ray 集群 |
| 扩展性 | 受单机资源限制 | 水平扩展 |

---

## 8. I/O 层与存储后端

### 8.1 I/O 抽象层

`src/daft-io/` 通过 **OpenDAL**（统一存储抽象）实现 26 个存储后端：

```
daft-io/src/
├── s3_like.rs         (64KB)  S3/MinIO/Ceph
├── azure_blob.rs      (27KB)  Azure Blob Storage
├── google_cloud.rs    (22KB)  Google Cloud Storage
├── tos.rs             (45KB)  腾讯云对象存储
├── gravitino.rs       (25KB)  Gravitino 目录
├── huggingface.rs     (25KB)  HuggingFace Hub
├── http.rs            (15KB)  HTTP/HTTPS
├── local.rs           (17KB)  本地文件系统
├── unity.rs                   Unity Catalog
├── opendal_source.rs  (16KB)  OpenDAL 通用后端
└── object_store_glob.rs (29KB) Glob 模式匹配
```

### 8.2 文件格式支持

| 格式 | Rust Crate | 读取 | 写入 | 备注 |
|------|-----------|------|------|------|
| Parquet | `daft-parquet` | ✓ | ✓ | 主要格式，支持下推 |
| CSV | `daft-csv` | ✓ | ✓ | |
| JSON (JSONL) | `daft-json` | ✓ | ✓ | |
| 纯文本 | `daft-text` | ✓ | - | 全文读取 |
| WARC | 内置 | ✓ | - | Common Crawl 格式 |
| Kafka | Python 层 | ✓ | ✓ | 流式读取 (29KB) |
| Delta Lake | Python 层 | ✓ | ✓ | 事务支持 |
| Apache Iceberg | Python 层 | ✓ | ✓ | 表格式 |
| Apache Hudi | Python 层 | ✓ | ✓ | 增量处理 |
| Lance | Python 层 | ✓ | ✓ | 向量数据库 |

**注：ORC 格式目前不支持**（已在 IMPROVEMENTS.md 中记录为缺口）。

### 8.3 目录集成

通过 `daft-catalog` 模块集成以下数据目录：

```
Apache Hive Metastore   BigTable
Unity Catalog           Gravitino
AWS Glue                Hugging Face
S3 Tables               Apache Polaris
```

### 8.4 下推优化

Scan 层（`daft-scan`）支持以下下推：

| 下推类型 | 说明 |
|----------|------|
| 谓词下推 | 将 WHERE 条件传递给 Parquet row group 过滤 |
| 投影下推 | 只读取需要的列（Parquet 列式存储优化） |
| 统计信息 | 利用 Parquet footer 统计跳过 row group |
| 分区裁剪 | Hive 分区目录的分区剪裁 |

---

## 9. 函数库体系

### 9.1 Rust 函数 Crate（11 个专用 crate）

```
daft-functions/                  核心函数注册表
daft-functions-temporal/         时间日期函数（宏驱动注册）
daft-functions-utf8/             字符串函数
daft-functions-binary/           二进制数据函数
daft-functions-json/             JSON 操作
daft-functions-list/             列表/数组函数
daft-functions-uri/              URL/URI 处理
daft-functions-tokenize/         文本分词
daft-functions-serde/            序列化/反序列化
daft-image/                      图像处理
daft-ai/                         AI 集成（OpenAI、Google、Transformers、vLLM）
```

### 9.2 函数注册机制

以时间函数为例（`src/daft-functions-temporal/src/lib.rs`）：

```rust
// 通过宏批量注册
register_temporal_function!(year, Year);
register_temporal_function!(month, Month);
register_temporal_function!(day, Day);
// ...

// 每个函数实现 ScalarUDF trait
impl ScalarUDF for Year {
    fn evaluate(&self, inputs: &[Series]) -> DaftResult<Series> {
        inputs[0].dt_year()  // 调用 Series 的方法
    }
}
```

调用链：

```
Python: col("date").dt.year()
  ↓
daft/expressions/expressions.py: ExpressionDatetimeNamespace.year()
  ↓
daft-dsl: Expr::Function { func: Year, inputs }
  ↓
daft-functions-temporal: Year::evaluate(series)
  ↓
daft-core: Series::dt_year()  (src/daft-core/src/series/ops/time.rs)
  ↓
DataArray<TimestampType>::year() → DataArray<Int32Type>
```

### 9.3 UDF 系统

`daft/udf/` 提供 4 种 UDF 装饰器：

| 装饰器 | 用途 |
|--------|------|
| `@udf` | 基础 UDF，按行/批处理 |
| `@func` | 函数式 UDF |
| `@cls` | 类方法 UDF（有状态，支持模型加载） |
| `@method` | 实例方法 UDF |

GPU UDF 示例（`daft-udf-tuning` 相关）：

```python
@daft.udf(return_dtype=daft.DataType.float32(), num_gpus=1)
def embed(text: daft.Series) -> list:
    model = load_model()  # 模型在 Worker 上缓存
    return model.encode(text.to_pylist())
```

---

## 10. 扩展系统

### 10.1 插件架构（`daft-ext`）

`src/daft-ext/` 提供**稳定 ABI**，允许外部 Rust 扩展接入 Daft，无需重新编译 Daft 本身：

```
daft-ext-abi/      ABI 稳定层（C 兼容接口）
daft-ext-core/     扩展核心 trait 定义
daft-ext-macros/   过程宏（简化扩展开发）
```

### 10.2 SQL 层（`daft-sql`）

独立的 SQL 解析器，将 SQL 翻译为 Daft 逻辑计划：

```sql
-- 支持的 SQL 特性
SELECT a, SUM(b) FROM t
WHERE a > 10
GROUP BY a
HAVING SUM(b) > 100
ORDER BY a
LIMIT 50
```

与 Spark SQL 语义高度兼容（见 `galaxy-spark-optimizer` skill）。

### 10.3 目录与会话（`daft-catalog`）

```
Session ──→ Catalog ──→ Table
  ↓           ↓           ↓
执行上下文   命名空间   Schema + ScanTask
```

---

## 11. 依赖关系图

```
                        ┌──────────────┐
                        │   用户代码   │
                        └──────┬───────┘
                               │
                        ┌──────▼───────┐
                        │  daft-runners│ ← Runner 选择 (native/ray)
                        └──────┬───────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
   ┌──────▼──────┐    ┌────────▼────────┐  ┌───────▼───────┐
   │daft-logical-│    │  daft-catalog   │  │   daft-sql    │
   │    plan     │    └─────────────────┘  └───────────────┘
   └──────┬──────┘
          │ 优化器
   ┌──────▼──────┐
   │daft-local-  │──→ daft-distributed (Ray)
   │    plan     │
   └──────┬──────┘
          │
   ┌──────▼──────┐
   │daft-local-  │
   │  execution  │
   └──────┬──────┘
          │
   ┌──────▼──────────────────────────────────────┐
   │              daft-micropartition             │
   └──────┬──────────────────────────────────────┘
          │
   ┌──────▼──────┐  ┌─────────────┐  ┌──────────────┐
   │daft-record- │  │  daft-scan  │  │   daft-io    │
   │   batch     │  └──────┬──────┘  └──────┬───────┘
   └──────┬──────┘         │                 │
          │          ┌─────▼──────┐  ┌───────▼────────┐
   ┌──────▼──────┐   │daft-parquet│  │OpenDAL/S3/GCS/  │
   │  daft-core  │   │ daft-csv   │  │Azure/HTTP/...   │
   └──────┬──────┘   │ daft-json  │  └────────────────┘
          │          └────────────┘
   ┌──────▼──────┐
   │ daft-schema │
   └──────┬──────┘
          │
   ┌──────▼──────┐
   │  Apache     │
   │  Arrow      │
   └─────────────┘
```

---

## 12. 核心设计模式

### 12.1 Builder 模式（逻辑计划构建）

```python
# Python 层调用
df.filter(col("a") > 1).select("a", "b")

# 内部 Rust LogicalPlanBuilder 调用链
LogicalPlanBuilder::filter(expr)
  → LogicalPlanBuilder::select(exprs)
    → LogicalPlan::Project { ... }
```

### 12.2 Visitor 模式（计划遍历与优化）

每个优化规则实现 `TreeNodeRewriter` trait，对计划树进行变换：

```rust
trait TreeNodeRewriter {
    fn f_down(&mut self, node: &LogicalPlan) -> Result<Transformed<LogicalPlan>>;
    fn f_up(&mut self, node: &LogicalPlan) -> Result<Transformed<LogicalPlan>>;
}
```

### 12.3 策略模式（执行引擎选择）

`daft-runners` 通过 `RunnerConfig` 在运行时选择 `NativeRunner` 或 `RayRunner`，两者实现相同的 `Runner` trait。

### 12.4 Morsel 驱动流水线

借鉴 DuckDB 的 Morsel-Driven Parallelism 思想：

- 数据分割为大小自适应的 morsel（micro-batch）
- Source 生产 morsel，通过有界 channel 传递
- 中间算子流式处理每个 morsel
- 阻塞算子（Sort、Agg）积累所有 morsel 后处理
- 反压机制防止内存溢出

### 12.5 PyO3 集成模式

```rust
// Rust 侧：通过 register_modules 导出
#[pymodule]
fn daft_core(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_class::<PySeries>()?;
    m.add_class::<PyDataType>()?;
    Ok(())
}

// Python 侧使用
from daft.daft import PySeries  # 直接使用 Rust 对象
```

---

## 13. 关键设计决策分析

### 13.1 为什么选择 Rust + Python？

| 方面 | 决策理由 |
|------|----------|
| **Rust 核心** | 内存安全、零开销抽象、WASM/native 兼容 |
| **Python API** | 数据科学生态系统兼容（Pandas、NumPy、MLflow 等） |
| **PyO3** | 比 ctypes/CFFI 更安全，比 CPython C API 更易用 |
| **Arrow 内存格式** | 零拷贝与其他系统交换数据（Spark、DuckDB、Polars） |

### 13.2 为什么选择 Morsel 驱动执行？

相比 MapReduce 风格（Spark）：

- **更低延迟**：流水线执行，无 shuffle 同步屏障
- **更好内存效率**：反压防止 OOM
- **更高并发**：CPU 核心持续饱和，无等待

### 13.3 Native + Ray 双执行路径的好处

- 开发环境使用 Native（无 Ray 依赖，调试方便）
- 生产环境无缝切换到 Ray（相同 API）
- 每个 Ray Worker 仍使用 Native 执行引擎（最大化单节点性能）

### 13.4 当前架构的潜在改进点

根据 `IMPROVEMENTS.md` 分析：

| 改进项 | 优先级 | 描述 |
|--------|--------|------|
| Spill-to-disk | 高 | 防止大数据集 OOM |
| Broadcast Join 自动检测 | 高 | 小表自动广播，减少 shuffle |
| ORC 格式支持 | 中 | Hadoop 生态系统互操作 |
| SQL ROLLUP/CUBE | 中 | 多维聚合支持 |
| DataFrame `.na` 对象 | 低 | Pandas 兼容性 |

---

## 附录：关键文件速查

| 文件 | 大小 | 描述 |
|------|------|------|
| `daft/dataframe/dataframe.py` | 239KB | Python DataFrame 主类 |
| `daft/expressions/expressions.py` | 102KB | 表达式 DSL |
| `src/daft-logical-plan/src/optimization/rules/push_down_filter.rs` | 52KB | 谓词下推（最大规则） |
| `src/daft-logical-plan/src/optimization/rules/split_udfs.rs` | 59KB | UDF 拆分（最大优化文件） |
| `src/daft-local-execution/src/pipeline.rs` | 53KB | 流水线核心 |
| `src/daft-io/src/s3_like.rs` | 64KB | S3 I/O（最大 I/O 文件） |
| `src/daft-core/src/datatypes/infer_datatype.rs` | 39KB | 类型推断引擎 |
| `daft/datatype.py` | 58KB | Python 类型定义 |
| `daft/functions/datetime.py` | 58KB | 时间函数 Python 封装 |

---

*本文档基于 Daft `ai-1` 分支代码库自动分析生成。*
