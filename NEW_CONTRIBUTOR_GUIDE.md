# Daft 新手贡献指南

> 专为首次参与 Daft 项目贡献的开发者准备的入门指南。

## 目录

1. [快速开始](#快速开始)
2. [新手友好的任务](#新手友好的任务)
3. [第一个 PR：分步指南](#第一个-pr分步指南)
4. [常见问题](#常见问题)
5. [学习资源](#学习资源)

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

## 可运行示例

以下是可以直接运行的代码示例，帮助理解 Daft 的 API 和内部工作原理。

### 示例 1: 快速验证安装

```python
# 保存为 test_install.py 并运行: python test_install.py

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

### 示例 2: 理解 Series (底层数据结构)

```python
# 保存为 test_series.py 并运行: python test_series.py

from __future__ import annotations
from daft import Series
import pyarrow as pa

# 1. 从 Python 列表创建 Series
print("=== 创建 Series ===")
s = Series.from_pylist([1, 2, 3, 4, 5])
print(f"数据: {s.to_pylist()}")
print(f"类型: {s.datatype()}")
print(f"长度: {len(s)}")

# 2. 字符串 Series 操作
print("\n=== 字符串操作 ===")
str_series = Series.from_pylist(["hello", "WORLD", "Daft"])
print(f"原始: {str_series.to_pylist()}")
print(f"大写: {str_series.str.upper().to_pylist()}")
print(f"小写: {str_series.str.lower().to_pylist()}")
print(f"长度: {str_series.str.length().to_pylist()}")

# 3. 从 PyArrow 创建 (零拷贝)
print("\n=== PyArrow 集成 ===")
arrow_array = pa.array([10, 20, 30, 40])
series_from_arrow = Series.from_arrow(arrow_array)
print(f"From Arrow: {series_from_arrow.to_pylist()}")

# 4. 处理 Null 值
print("\n=== Null 值处理 ===")
s_with_null = Series.from_pylist([1, None, 3, None, 5])
print(f"含 Null: {s_with_null.to_pylist()}")
print(f"填充 Null: {s_with_null.fill_null(0).to_pylist()}")
```

### 示例 3: 理解表达式 (Expression)

```python
# 保存为 test_expressions.py 并运行: python test_expressions.py

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
    (col("b") - col("a")).alias("subtract"),
])
print(result.to_pydict())

# 2. 比较表达式
print("\n=== 比较表达式 ===")
result = table.eval_expression_list([
    col("a"),
    (col("a") > lit(3)).alias("gt_3"),
    (col("a") == lit(2)).alias("eq_2"),
    (col("a") != lit(5)).alias("ne_5"),
])
print(result.to_pydict())

# 3. 字符串表达式
print("\n=== 字符串表达式 ===")
result = table.eval_expression_list([
    col("name").str.upper().alias("upper"),
    col("name").str.length().alias("length"),
])
print(result.to_pydict())

# 4. 数学函数
print("\n=== 数学函数 ===")
result = table.eval_expression_list([
    col("a"),
    col("a").abs().alias("abs"),
    col("b").round().alias("round"),
    col("a").sqrt().alias("sqrt"),
])
print(result.to_pydict())
```

### 示例 4: RecordBatch/MicroPartition 操作

```python
# 保存为 test_recordbatch.py 并运行: python test_recordbatch.py

from __future__ import annotations
from daft.expressions import col
from daft.recordbatch import MicroPartition

# 1. 创建 RecordBatch
print("=== 创建 RecordBatch ===")
table = MicroPartition.from_pydict({
    "col": ["foo", None, "barBaz", "quux", "1"],
    "num": [1, 2, 3, 4, 5]
})
print(table.to_pydict())
print(f"列名: {table.column_names()}")
print(f"行数: {len(table)}")

# 2. 评估表达式
print("\n=== 执行表达式 ===")
result = table.eval_expression_list([col("col").capitalize()])
print(result.to_pydict())

# 3. 过滤
print("\n=== 过滤 ===")
filtered = table.filter(col("num") > 2)
print(filtered.to_pydict())

# 4. 获取单列
print("\n=== 获取单列 ===")
col_data = table.get_column_by_name("num")
print(f"列数据: {col_data.to_pylist()}")
print(f"列类型: {col_data.datatype()}")

# 5. Schema 信息
print("\n=== Schema 信息 ===")
print(table.schema())
```

### 示例 5: DataFrame 完整工作流

```python
# 保存为 test_dataframe.py 并运行: python test_dataframe.py

from __future__ import annotations
import daft
from daft import col, lit

# 1. 创建 DataFrame
print("=== 创建 DataFrame ===")
df = daft.from_pydict({
    "name": ["Alice", "Bob", "Charlie", "Diana", "Eve"],
    "department": ["Engineering", "Sales", "Engineering", "Marketing", "Sales"],
    "salary": [80000.0, 60000.0, 90000.0, 70000.0, 65000.0],
    "years_exp": [5, 3, 7, 4, 2]
})
df.show()

# 2. 过滤
print("\n=== 过滤: 薪资 > 65000 ===")
df.filter(col("salary") > 65000).show()

# 3. 选择列
print("\n=== 选择列 ===")
df.select(col("name"), col("salary")).show()

# 4. 添加计算列
print("\n=== 添加计算列 ===")
df.with_column("bonus", col("salary") * lit(0.1)).show()

# 5. 分组聚合
print("\n=== 按部门分组 ===")
df.groupby(col("department")).agg(
    daft.count(col("name")).alias("employee_count"),
    daft.mean(col("salary")).alias("avg_salary"),
    daft.max(col("salary")).alias("max_salary"),
).show()

# 6. 排序
print("\n=== 按薪资降序 ===")
df.sort(col("salary"), desc=True).show()

# 7. 收集结果
print("\n=== 收集为 Python 字典 ===")
result = df.collect().to_pydict()
print(result)
```

### 示例 6: SQL 查询

```python
# 保存为 test_sql.py 并运行: python test_sql.py

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

# 1. 简单查询
print("=== 简单查询 ===")
result = sql("SELECT * FROM sales WHERE quantity > 5")
result.show()

# 2. 聚合查询
print("\n=== 聚合查询 ===")
result = sql("""
    SELECT 
        product,
        COUNT(*) as total_orders,
        SUM(quantity) as total_quantity,
        AVG(price) as avg_price
    FROM sales
    GROUP BY product
""")
result.show()

# 3. 条件查询
print("\n=== 条件查询 ===")
result = sql("SELECT id, product FROM sales WHERE price >= 200 AND quantity > 5")
result.show()
```

### 示例 7: 调试技巧

```python
# 保存为 test_debug.py 并运行: python test_debug.py

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

# 查看逻辑计划
print("\n=== 逻辑计划详情 ===")
print(df2._builder)
```

### 运行示例的方法

```bash
# 1. 确保环境激活
source .venv/bin/activate

# 2. 确保已构建
make build

# 3. 运行单个示例
python test_install.py

# 4. 使用 IPython 交互式运行
ipython -i test_dataframe.py

# 5. 运行测试（从测试目录）
export DAFT_RUNNER=native
pytest tests/recordbatch/utf8/test_capitalize.py -v
```

### 常见测试模式

```python
# 模式 1: 测试 Series 函数
def test_my_function():
    from daft import Series
    
    # 准备输入
    input_series = Series.from_pylist(["hello", "world"])
    
    # 执行操作
    result = input_series.str.my_function()
    
    # 验证结果
    assert result.to_pylist() == ["expected", "result"]

# 模式 2: 测试表达式
def test_my_expression():
    from daft.expressions import col
    from daft.recordbatch import MicroPartition
    
    # 创建测试数据
    table = MicroPartition.from_pydict({
        "col": ["foo", "bar"]
    })
    
    # 评估表达式
    result = table.eval_expression_list([col("col").my_function()])
    
    # 验证
    assert result.to_pydict() == {"col": ["expected", "result"]}

# 模式 3: 测试 DataFrame
def test_my_dataframe_op():
    import daft
    from daft import col
    
    df = daft.from_pydict({"x": [1, 2, 3]})
    result = df.my_operation().collect().to_pydict()
    
    assert result == {"x": ["expected"]}
```

---

## 新手友好的任务

### 🟢 难度：入门 (Good First Issues)

#### 1. 添加字符串函数

**背景**: Daft 的字符串函数库需要扩展更多标准函数。

**任务示例**: 实现 `reverse()` 字符串函数

**涉及的文件**:
- `src/daft-functions-utf8/src/reverse.rs` (新建)
- `src/daft-functions-utf8/src/lib.rs` (注册函数)
- `daft/expressions/expressions.py` (Python 绑定)
- `tests/recordbatch/utf8/test_reverse.py` (测试)

**参考实现**: 查看 `src/daft-functions-utf8/src/capitalize.rs`

```rust
// 基本结构
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
    
    fn docstring(&self) -> &'static str { "Reverse a UTF-8 string" }
}
```

**类似任务**:
- [ ] 实现 `trim()` 函数
- [ ] 实现 `md5()` 哈希函数
- [ ] 实现 `initcap()` 函数（每个单词首字母大写）

---

#### 2. 改进错误信息

**背景**: 让错误信息更友好，帮助用户快速定位问题。

**任务示例**: 改进类型不匹配的错误提示

**涉及的文件**:
- `src/common/error/src/lib.rs`
- 各模块的错误处理代码

**示例改进**:
```rust
// 之前
Err(DaftError::TypeError("Expected Int64, got Utf8"))

// 改进后
Err(DaftError::TypeError(format!(
    "Type mismatch: column '{}' expected type '{}' but got '{}'", 
    column_name, expected, actual
)))
```

**类似任务**:
- [ ] 为 SQL 解析错误添加更详细的上下文
- [ ] 改进文件未找到错误，显示搜索路径
- [ ] 优化函数参数错误提示

---

#### 3. 添加文档和示例

**背景**: 完善的文档对用户体验至关重要。

**任务示例**: 为现有函数添加文档字符串示例

**涉及的文件**:
- `daft/functions/*.py`
- `docs/api/*.md`

**示例**:
```python
def length_bytes(self) -> Expression:
    """Returns the length of the string in bytes.
    
    Examples:
        >>> import daft
        >>> df = daft.from_pydict({"text": ["hello", "世界"]})
        >>> df = df.with_column("byte_len", df["text"].str.length_bytes())
        >>> df.show()
        ╭────────┬──────────╮
        │ text   ┆ byte_len │
        ╞════════╪══════════╡
        │ hello  ┆ 5        │
        │ 世界   ┆ 6        │
        ╰────────┴──────────╯
    """
```

**类似任务**:
- [ ] 为 DataFrame 方法添加使用示例
- [ ] 创建教程 Notebook
- [ ] 完善 API 文档中的类型提示

---

#### 4. 修复简单的 TODO 注释

**背景**: 代码中有许多标记为 TODO 的小改进点。

**查找 TODO**:
```bash
grep -r "TODO" src/ --include="*.rs" | head -20
```

**入门级别的 TODO**:
- `src/daft-logical-plan/src/display/json.rs`: 验证 JSON 序列化正确性
- `src/daft-functions-uri/src/upload.rs`: 优化大文件迭代器
- `src/daft-writers/src/utils.rs`: 改进 URL 构建函数

---

### 🟡 难度：中等

#### 5. 实现新的表达式函数

**任务**: 添加数值函数或时间函数

**示例**: 实现 `degrees()` 函数（弧度转角度）

**参考**:
- `src/daft-functions/src/numeric.rs`
- `src/daft-functions-temporal/src/`

#### 6. 添加测试用例

**任务**: 为边界条件添加测试

**涉及的目录**:
- `tests/recordbatch/`
- `tests/dataframe/`
- `tests/expressions/`

**示例测试**:
```python
def test_length_bytes_empty():
    """Test length_bytes with empty strings."""
    daft_series = Series.from_pylist(["", "hello", ""])
    result = daft_series.utf8_length_bytes()
    assert result.to_pylist() == [0, 5, 0]

def test_length_bytes_unicode():
    """Test length_bytes with unicode characters."""
    daft_series = Series.from_pylist(["世界", "hello"])
    result = daft_series.utf8_length_bytes()
    assert result.to_pylist() == [6, 5]  # UTF-8 编码
```

#### 7. 性能基准测试

**任务**: 为新功能添加性能测试

**涉及的目录**:
- `tests/microbenchmarks/`

---

## 第一个 PR：分步指南

### 步骤 1：选择任务

推荐从以下开始：
1. 添加一个简单的字符串函数（如 `reverse`）
2. 修复文档中的拼写错误
3. 添加测试用例

### 步骤 2：创建分支

```bash
git checkout -b feature/add-string-reverse-function
```

### 步骤 3：实现功能

**以 reverse 函数为例**:

#### 3.1 创建 Rust 实现

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
    fn name(&self) -> &'static str {
        "reverse"
    }
    
    fn call(
        &self,
        inputs: daft_dsl::functions::FunctionArgs<Series>,
        _ctx: &daft_dsl::functions::scalar::EvalContext,
    ) -> DaftResult<Series> {
        unary_utf8_evaluate(inputs, reverse_impl)
    }

    fn get_return_field(
        &self,
        inputs: FunctionArgs<ExprRef>,
        schema: &Schema,
    ) -> DaftResult<Field> {
        unary_utf8_to_field(inputs, schema, self.name(), DataType::Utf8)
    }

    fn docstring(&self) -> &'static str {
        "Reverse a UTF-8 string."
    }
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

#### 3.2 注册函数

```rust
// src/daft-functions-utf8/src/lib.rs
mod reverse;
pub use reverse::*;

// 在 register 函数中添加
parent.add_fn(Reverse);
```

#### 3.3 添加 Python API

```python
# daft/expressions/expressions.py
class ExpressionStringNamespace:
    def reverse(self) -> Expression:
        """Reverse the string.
        
        Examples:
            >>> import daft
            >>> df = daft.from_pydict({"s": ["abc", "123"]})
            >>> df.select(df["s"].str.reverse()).show()
            ╭───────╮
            │ s     │
            ╞═══════╡
            │ cba   │
            │ 321   │
            ╰───────╯
        """
        return Expression._from_pyexpr(self._expr.utf8_reverse())
```

#### 3.4 添加 Rust-Python 绑定

在相应的 Rust 文件中添加 PyO3 绑定：

```rust
// 在 daft-dsl 或相关模块中
#[pymethods]
impl PyExpr {
    fn utf8_reverse(&self) -> PyResult<Self> {
        Ok(utf8_reverse(self.into()).into())
    }
}
```

#### 3.5 编写测试

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


def test_utf8_reverse_nulls():
    daft_series = Series.from_pylist(["abc", None, "def"])
    result = daft_series.utf8_reverse()
    assert result.to_pylist() == ["cba", None, "fed"]
```

### 步骤 4：测试和验证

```bash
# 构建
make build

# 运行相关测试
make test EXTRA_ARGS="-v tests/recordbatch/utf8/test_reverse.py"

# 运行所有测试确保没有破坏
make test

# 代码格式化
make format

# Lint 检查
make lint
```

### 步骤 5：提交 PR

```bash
git add .
git commit -m "feat: add utf8 reverse function

- Add reverse() function for reversing UTF-8 strings
- Handle unicode characters correctly
- Add comprehensive tests"

git push origin feature/add-string-reverse-function
```

**PR 检查清单**:
- [ ] 代码遵循项目风格
- [ ] 所有测试通过
- [ ] 添加了新测试
- [ ] 更新了文档（如果需要）
- [ ]  commit 消息遵循 [Conventional Commits](https://www.conventionalcommits.org/)

---

## 常见问题

### Q1: 构建失败 "rust using incorrect toolchain"

```bash
# 检查当前工具链
rustup show active-toolchain

# 应该显示: nightly-2025-09-03

# 如果没有，安装正确的工具链
rustup install nightly-2025-09-03
rustup default nightly-2025-09-03
```

### Q2: Python 导入错误

```bash
# 确保已正确构建
make build

# 检查 .so 文件是否存在
ls -la daft/daft*.so

# 如果仍然失败，尝试 clean 后重新构建
make clean && make build
```

### Q3: 测试挂起（特别是 Ray 测试）

```bash
# 设置 runner 为 native
export DAFT_RUNNER=native

# 限制并行度
make test EXTRA_ARGS="-n 2"
```

### Q4: 如何调试 Rust 代码？

```bash
# 设置日志级别
export DAFT_LOG=debug

# Python 中
import logging
logging.basicConfig(level=logging.DEBUG)
import daft
daft.refresh_logger()
```

### Q5: 如何理解代码中的生命周期标注？

推荐阅读:
- [Rust Book - Lifetimes](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html)
- [Arrow Rust 文档](https://docs.rs/arrow/latest/arrow/)

---

## 学习资源

### Rust 学习

- [The Rust Book](https://doc.rust-lang.org/book/) - 官方教程
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/) - 实例学习
- [Arrow Rust 文档](https://docs.rs/arrow/latest/arrow/)

### 项目特定

- [ARCHITECTURE_ANALYSIS.md](./ARCHITECTURE_ANALYSIS.md) - 架构分析
- [AGENTS.md](./AGENTS.md) - 开发工作流
- [CONTRIBUTING.md](./CONTRIBUTING.md) - 贡献指南

### 社区支持

- [GitHub Discussions](https://github.com/Eventual-Inc/Daft/discussions) - 提问和交流
- [Slack 社区](https://www.daft.ai/slack) - 实时交流
- [GitHub Issues](https://github.com/Eventual-Inc/Daft/issues) - 报告问题

### 推荐的第一个贡献方向

| 方向 | 难度 | 前置知识 | 相关文件 |
|------|------|----------|----------|
| 字符串函数 | ⭐ | 基础 Rust | `src/daft-functions-utf8/` |
| 文档改进 | ⭐ | Python | `daft/` 各模块 |
| 测试添加 | ⭐⭐ | Python | `tests/` |
| 数值函数 | ⭐⭐ | Rust | `src/daft-functions/` |
| I/O 连接器 | ⭐⭐⭐ | Rust + 存储 | `src/daft-io/` |

---

## 贡献建议

### 新手最佳实践

1. **从小开始**: 先修文档、拼写错误，熟悉流程
2. **阅读测试**: 通过测试理解代码行为
3. **复制修改**: 找到类似功能的实现，复制后修改
4. **积极提问**: 在 Discussion 或 Slack 提问
5. **关注 Review**: 认真处理代码审查意见

### 代码审查准备

```bash
# 提交前运行完整检查
make precommit

# 确保测试通过
make test

# 检查测试覆盖率（如果有变化）
```

### 寻求帮助

如果遇到问题：

1. 查看 [GitHub Issues](https://github.com/Eventual-Inc/Daft/issues) 是否有类似问题
2. 在 [Discussions](https://github.com/Eventual-Inc/Daft/discussions) 提问
3. 在 Slack #help 频道提问
4. 在 PR 中 @maintainer

---

## 总结

欢迎来到 Daft 社区！记住：

- 🎯 **从小任务开始** - 积累信心和经验
- 📚 **多阅读代码** - 学习项目风格和模式  
- 🤝 **积极交流** - 社区乐意帮助新人
- 🔄 **迭代改进** - PR 审查是学习和改进的机会

期待你的第一次贡献！

---

*最后更新: 2026-03-07*
