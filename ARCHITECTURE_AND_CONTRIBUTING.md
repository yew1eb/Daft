# Daft 架构分析与贡献指南

> 本文档深入分析 Daft 数据引擎的架构设计，并提供完整的新手贡献指南，帮助开发者快速理解系统并开始贡献代码。

## 目录

1. [快速开始](#快速开始)
2. [架构概览](#架构概览)
3. [核心模块详解](#核心模块详解)
4. [数据流分析](#数据流分析)
5. [可运行示例](#可运行示例)
6. [新手友好的任务](#新手友好的任务)
7. [第一个 PR：分步指南](#第一个-pr分步指南)
8. [常见问题](#常见问题)
9. [学习资源](#学习资源)

---

## 快速开始

### 1. 环境搭建

```bash
# 1. 克隆仓库
git clone https://github.com/Eventual-Inc/Daft.git
cd Daft

# 2. 设置 Python 环境（使用 uv）
make .venv
source .venv/bin/activate

# 3. 构建 Rust 扩展
make build

# 4. 运行测试验证
export DAFT_RUNNER=native
make test EXTRA_ARGS="-v tests/test_import.py"
```

### 2. 项目结构速览

```
.
├── src/                    # Rust 核心代码
│   ├── daft-core/         # 核心数据类型
│   ├── daft-dsl/          # 表达式系统
│   ├── daft-logical-plan/ # 逻辑查询计划
│   ├── daft-local-execution/ # 执行引擎
│   ├── daft-functions-*/  # 各类函数实现
│   └── daft-io/           # I/O 系统
├── daft/                   # Python API
│   ├── dataframe/         # DataFrame 实现
│   ├── expressions/       # 表达式 API
│   ├── functions/         # 函数包装
│   └── io/                # I/O 连接器
└── tests/                  # 测试套件
    ├── unit/              # 单元测试
    └── integration/       # 集成测试
```

### 3. 开发工作流

```bash
# 修改代码后重新构建
make build

# 运行测试
export DAFT_RUNNER=native
make test

# 格式化代码
make format

# 运行 lint
make lint

# 提交前检查
make precommit
```

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

将逻辑计划转换为可执行的物理计划。

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

### 5. I/O 系统 (daft-io)

**位置**: `src/daft-io/`

统一的对象存储接口，支持 File/S3/GCS/Azure/HTTP。

### 6. 函数库 (daft-functions-*)

**位置**: `src/daft-functions-*/`

模块化函数实现：

```
daft-functions-utf8/    # 字符串函数 (length, split, replace...)
daft-functions-list/    # 列表函数 (append, slice, contains...)
daft-functions-json/    # JSON 函数 (jq 查询)
daft-functions-temporal/ # 时间函数 (date_trunc, extract...)
daft-functions-binary/  # 二进制函数
daft-functions-uri/     # URI 处理函数
daft-image/            # 图像处理函数
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

## 可运行示例

以下是直接从测试中提取的可运行代码示例，保存后可直接运行：`python example.py`

### 示例 1: 快速验证安装

```python
from __future__ import annotations
import daft
from daft import col, lit

print("✅ Daft 安装成功!")
print(f"版本: {daft.get_version()}")

# 创建简单 DataFrame
df = daft.from_pydict({
    "name": ["Alice", "Bob", "Charlie"],
    "age": [25, 30, 35],
    "score": [85.5, 90.0, 78.5]
})

print("\n=== 原始数据 ===")
df.show()

# 简单过滤
print("\n=== 年龄 > 28 ===")
df.filter(col("age") > 28).show()

print("\n✅ 所有测试通过!")
```

### 示例 2: Series 操作（底层数据结构）

```python
from __future__ import annotations
from daft import Series
import pyarrow as pa

# 1. 从 Python 列表创建 Series
print("=== 创建 Series ===")
s = Series.from_pylist([1, 2, 3, 4, 5])
print(f"数据: {s.to_pylist()}")
print(f"类型: {s.datatype()}")

# 2. 字符串 Series 操作
print("\n=== 字符串操作 ===")
str_series = Series.from_pylist(["hello", "WORLD", "Daft"])
print(f"原始: {str_series.to_pylist()}")
print(f"大写: {str_series.str.upper().to_pylist()}")
print(f"小写: {str_series.str.lower().to_pylist()}")
print(f"长度: {str_series.str.length().to_pylist()}")

# 3. 处理 Null 值
print("\n=== Null 值处理 ===")
s_with_null = Series.from_pylist([1, None, 3, None, 5])
print(f"含 Null: {s_with_null.to_pylist()}")
print(f"填充 Null: {s_with_null.fill_null(0).to_pylist()}")
```

### 示例 3: 表达式操作（核心概念）

```python
from __future__ import annotations
from daft.expressions import col, lit
from daft.recordbatch import MicroPartition

# 创建测试数据
table = MicroPartition.from_pydict({
    "a": [1, 2, 3, 4, 5],
    "b": [10.0, 20.0, 30.0, 40.0, 50.0],
    "name": ["alice", "bob", "charlie", "diana", "eve"]
})

print("=== 原始数据 ===")
print(table.to_pydict())

# 1. 算术表达式
print("\n=== 算术表达式 ===")
result = table.eval_expression_list([
    (col("a") + col("b")).alias("add"),
    (col("a") * lit(2)).alias("multiply"),
])
print(result.to_pydict())

# 2. 比较表达式
print("\n=== 比较表达式 ===")
result = table.eval_expression_list([
    (col("a") > lit(3)).alias("gt_3"),
    (col("a") == lit(2)).alias("eq_2"),
])
print(result.to_pydict())

# 3. 字符串表达式
print("\n=== 字符串表达式 ===")
result = table.eval_expression_list([
    col("name").str.upper().alias("upper"),
    col("name").str.length().alias("length"),
])
print(result.to_pydict())
```

### 示例 4: DataFrame 完整工作流

```python
from __future__ import annotations
import daft
from daft import col, lit

# 创建 DataFrame
df = daft.from_pydict({
    "name": ["Alice", "Bob", "Charlie", "Diana", "Eve"],
    "department": ["Engineering", "Sales", "Engineering", "Marketing", "Sales"],
    "salary": [80000.0, 60000.0, 90000.0, 70000.0, 65000.0],
    "years_exp": [5, 3, 7, 4, 2]
})

print("=== 创建 DataFrame ===")
df.show()

# 过滤
print("\n=== 过滤: 薪资 > 65000 ===")
df.filter(col("salary") > 65000).show()

# 分组聚合
print("\n=== 按部门分组 ===")
df.groupby(col("department")).agg(
    daft.count(col("name")).alias("employee_count"),
    daft.mean(col("salary")).alias("avg_salary"),
).show()

# 排序
print("\n=== 按薪资降序 ===")
df.sort(col("salary"), desc=True).show()
```

### 示例 5: SQL 查询

```python
from __future__ import annotations
import daft
from daft import sql

# 创建数据
df = daft.from_pydict({
    "id": [1, 2, 3, 4, 5],
    "product": ["A", "B", "A", "C", "B"],
    "quantity": [10, 5, 8, 3, 12],
    "price": [100.0, 200.0, 100.0, 300.0, 200.0]
})

# 注册为临时表
df.create_temp_table("sales")

# 简单查询
print("=== 简单查询 ===")
sql("SELECT * FROM sales WHERE quantity > 5").show()

# 聚合查询
print("\n=== 聚合查询 ===")
sql("""
    SELECT 
        product,
        COUNT(*) as total_orders,
        SUM(quantity) as total_quantity
    FROM sales
    GROUP BY product
""").show()
```

### 示例 6: 调试技巧

```python
from __future__ import annotations
import daft
from daft import col

# 启用调试日志
import logging
logging.basicConfig(level=logging.INFO)
daft.refresh_logger()

df = daft.from_pydict({
    "x": [1, 2, 3, 4, 5],
    "y": ["a", "b", "c", "d", "e"]
})

# 查看执行计划
print("=== 执行计划 ===")
df2 = df.filter(col("x") > 2).select(col("y"))
df2.explain()

print("\n=== 优化后的计划 ===")
df2.explain(show_optimized=True)
```

---

## 新手友好的任务

### 🟢 难度：入门

#### 1. 添加字符串函数

**任务**: 实现 `reverse()` 字符串函数

**参考实现**:
```rust
// src/daft-functions-utf8/src/capitalize.rs
#[derive(Clone, Serialize, Deserialize, PartialEq, Eq, Hash)]
pub struct Reverse;

#[typetag::serde]
impl ScalarUDF for Reverse {
    fn name(&self) -> &'static str { "reverse" }
    fn call(&self, inputs: FunctionArgs<Series>, _ctx: &EvalContext) -> DaftResult<Series> {
        unary_utf8_evaluate(inputs, reverse_impl)
    }
    fn docstring(&self) -> &'static str { "Reverse a UTF-8 string" }
}
```

**类似任务**: `trim()`, `md5()`, `initcap()`

#### 2. 改进错误信息

**示例**:
```rust
// 之前
Err(DaftError::TypeError("Expected Int64, got Utf8"))

// 改进后
Err(DaftError::TypeError(format!(
    "Type mismatch: column '{}' expected '{}' but got '{}'", 
    column_name, expected, actual
)))
```

#### 3. 添加文档和示例

为现有函数添加文档字符串示例（参考示例代码）。

#### 4. 修复 TODO 注释

```bash
# 查找 TODO
grep -r "TODO" src/ --include="*.rs" | head -20
```

### 🟡 难度：中等

- 实现新的表达式函数
- 添加边界条件测试
- 性能基准测试

---

## 第一个 PR：分步指南

### 步骤 1：创建分支

```bash
git checkout -b feature/add-string-reverse-function
```

### 步骤 2：实现功能（以 reverse 为例）

#### 2.1 Rust 实现

```rust
// src/daft-functions-utf8/src/reverse.rs
use common_error::DaftResult;
use daft_core::{
    prelude::{DataType, Field, Schema},
    series::{IntoSeries, Series},
};
use daft_dsl::{
    ExprRef,
    functions::{FunctionArgs, ScalarUDF, scalar::ScalarFn},
};
use serde::{Deserialize, Serialize};
use crate::utils::{unary_utf8_evaluate, unary_utf8_to_field};

#[derive(Clone, Serialize, Deserialize, PartialEq, Eq, Hash)]
pub struct Reverse;

#[typetag::serde]
impl ScalarUDF for Reverse {
    fn name(&self) -> &'static str { "reverse" }
    
    fn call(&self, inputs: FunctionArgs<Series>, _ctx: &EvalContext) -> DaftResult<Series> {
        unary_utf8_evaluate(inputs, reverse_impl)
    }

    fn get_return_field(&self, inputs: FunctionArgs<ExprRef>, schema: &Schema) -> DaftResult<Field> {
        unary_utf8_to_field(inputs, schema, self.name(), DataType::Utf8)
    }

    fn docstring(&self) -> &'static str { "Reverse a UTF-8 string." }
}

pub fn reverse(e: ExprRef) -> ExprRef {
    ScalarFn::builtin(Reverse, vec![e]).into()
}

fn reverse_impl(s: &Series) -> DaftResult<Series> {
    s.with_utf8_array(|u| {
        u.unary_broadcasted_op(|val| val.chars().rev().collect::<String>())
            .map(IntoSeries::into_series)
    })
}
```

#### 2.2 注册函数

```rust
// src/daft-functions-utf8/src/lib.rs
mod reverse;
pub use reverse::*;

// 在 register 函数中添加
parent.add_fn(Reverse);
```

#### 2.3 Python API

```python
# daft/expressions/expressions.py
class ExpressionStringNamespace:
    def reverse(self) -> Expression:
        """Reverse the string."""
        return Expression._from_pyexpr(self._expr.utf8_reverse())
```

#### 2.4 编写测试

```python
# tests/recordbatch/utf8/test_reverse.py
import pytest
from daft import Series

def test_utf8_reverse():
    daft_series = Series.from_pylist(["abc", "12345", "", "Hello"])
    result = daft_series.utf8_reverse()
    assert result.to_pylist() == ["cba", "54321", "", "olleH"]

def test_utf8_reverse_unicode():
    daft_series = Series.from_pylist(["世界", "你好"])
    result = daft_series.utf8_reverse()
    assert result.to_pylist() == ["界世", "好你"]
```

### 步骤 3：测试和验证

```bash
make build
make test EXTRA_ARGS="-v tests/recordbatch/utf8/test_reverse.py"
make format
make lint
```

### 步骤 4：提交 PR

```bash
git add .
git commit -m "feat: add utf8 reverse function

- Add reverse() function for reversing UTF-8 strings
- Handle unicode characters correctly
- Add comprehensive tests"

git push origin feature/add-string-reverse-function
```

---

## 常见问题

### Q1: 构建失败 "rust using incorrect toolchain"

```bash
rustup show active-toolchain  # 应显示: nightly-2025-09-03
rustup install nightly-2025-09-03
rustup default nightly-2025-09-03
```

### Q2: Python 导入错误

```bash
make build
ls -la daft/daft*.so  # 检查 .so 文件
make clean && make build  # 重新构建
```

### Q3: 测试挂起

```bash
export DAFT_RUNNER=native
make test EXTRA_ARGS="-n 2"
```

### Q4: 调试 Rust 代码

```python
import logging
logging.basicConfig(level=logging.DEBUG)
import daft
daft.refresh_logger()
```

---

## 学习资源

### Rust 学习

- [The Rust Book](https://doc.rust-lang.org/book/)
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/)
- [Arrow Rust 文档](https://docs.rs/arrow/latest/arrow/)

### 项目资源

- [Daft 官方文档](https://docs.daft.ai)
- [AGENTS.md](./AGENTS.md) - 开发工作流
- [CONTRIBUTING.md](./CONTRIBUTING.md) - 贡献指南

### 社区支持

- [GitHub Discussions](https://github.com/Eventual-Inc/Daft/discussions)
- [Slack 社区](https://www.daft.ai/slack)
- [GitHub Issues](https://github.com/Eventual-Inc/Daft/issues)

### 贡献方向推荐

| 方向 | 难度 | 前置知识 | 相关文件 |
|------|------|----------|----------|
| 字符串函数 | ⭐ | 基础 Rust | `src/daft-functions-utf8/` |
| 文档改进 | ⭐ | Python | `daft/` 各模块 |
| 测试添加 | ⭐⭐ | Python | `tests/` |
| 数值函数 | ⭐⭐ | Rust | `src/daft-functions/` |
| I/O 连接器 | ⭐⭐⭐ | Rust + 存储 | `src/daft-io/` |

---

## 总结

欢迎来到 Daft 社区！

- 🎯 **从小任务开始** - 积累信心和经验
- 📚 **多阅读代码** - 学习项目风格和模式
- 🤝 **积极交流** - 社区乐意帮助新人
- 🔄 **迭代改进** - PR 审查是学习和改进的机会

期待你的第一次贡献！

---

*最后更新: 2026-03-07*
