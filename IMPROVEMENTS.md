# Daft 改进分析报告

> 生成日期：2026-03-19
> 分析范围：Python API、Rust 执行引擎、SQL 函数、IO 层
> 目标：对齐 PySpark 功能，提升性能与生产就绪性

---

## 目录

1. [总览](#总览)
2. [Python DataFrame API 缺口](#python-dataframe-api-缺口)
3. [内置函数缺口](#内置函数缺口)
4. [Rust 执行引擎性能优化](#rust-执行引擎性能优化)
5. [IO 层改进](#io-层改进)
6. [SQL 支持改进](#sql-支持改进)
7. [优先级路线图](#优先级路线图)

---

## 总览

Daft 是具备 Python API + Rust 核心的高性能分布式数据框架，功能丰富但与 PySpark 尚有以下主要差距：

| 维度 | 现状评分 | 关键缺口 |
|------|---------|---------|
| DataFrame API | 85% | 缓存API、rollup/cube、na对象、统计方法 |
| 内置函数 | 75% | 日期差值函数、字符串格式化、数组高级操作 |
| 执行引擎 | 65% | 无 spill-to-disk、无 broadcast join、CBO 不完整 |
| IO / 格式 | 80% | 缺 ORC、Avro，Hive Metastore 未集成 |
| SQL 支持 | 70% | 窗口函数部分缺失、若干 Spark SQL 函数缺失 |

---

## Python DataFrame API 缺口

### 1. 缓存与性能提示 (HIGH PRIORITY)

**缺失方法：**

```python
# PySpark API                    # Daft 现状
df.cache()                       # ✗ 内部有 _result_cache 但无公共 API
df.persist(StorageLevel.DISK)    # ✗ 完全缺失
df.unpersist()                   # ✗ 完全缺失
df.hint("broadcast")             # ✗ 完全缺失
broadcast(df)                    # △ 仅作为 join(strategy="broadcast") 存在
```

**建议实现位置：** `daft/dataframe/dataframe.py`

```python
def cache(self) -> "DataFrame":
    """缓存 DataFrame 到内存，避免重复计算。"""
    # 复用现有 _result_cache 机制
    ...

def persist(self) -> "DataFrame":
    """持久化 DataFrame 到内存/磁盘。"""
    ...

def hint(self, name: str, *parameters) -> "DataFrame":
    """向查询优化器提供执行提示（broadcast、repartition 等）。"""
    ...
```

---

### 2. GroupBy 高级聚合 (MEDIUM PRIORITY)

**缺失方法（`GroupedDataFrame` 类中）：**

```python
# PySpark API                                  # Daft 现状
df.groupby("dept").rollup("dept","year").sum()  # ✗ 无 rollup
df.groupby("dept").cube("dept","year").count()  # ✗ 无 cube
```

**建议位置：** `daft/dataframe/dataframe.py` 第 5061 行附近的 `GroupedDataFrame` 类

`rollup` 实现思路：
1. 生成多层次分组组合（全部列、去掉最后一列、……、空）
2. 对每个组合执行 groupby 聚合
3. UNION ALL 所有结果，缺失的分组维度填 NULL

---

### 3. NA 处理对象 (MEDIUM PRIORITY)

**缺失：**

```python
# PySpark API               # Daft 现状
df.na.fill(0)                # ✗ 无 .na 属性，需用 col("x").fill_null(0)
df.na.drop()                 # ✗ 需用 df.drop_null()
df.na.replace(1, 2)          # ✗ 完全缺失
df.fillna({"col": 0})        # ✗ 无 fillna() 快捷方法
```

**建议实现：**

```python
# daft/dataframe/na.py（新文件）
class DataFrameNaFunctions:
    def fill(self, value, subset=None) -> "DataFrame": ...
    def drop(self, how="any", thresh=None, subset=None) -> "DataFrame": ...
    def replace(self, to_replace, value, subset=None) -> "DataFrame": ...

# dataframe.py 中添加属性
@property
def na(self) -> DataFrameNaFunctions:
    return DataFrameNaFunctions(self)
```

---

### 4. 随机采样与统计 (MEDIUM PRIORITY)

**缺失方法：**

```python
# PySpark API
df.randomSplit([0.8, 0.2], seed=42)     # ✗ 缺失（有 sample() 但无分割）
df.sampleBy("col", fractions={0: 0.1})  # ✗ 缺失（分层采样）
df.freqItems(["col1", "col2"])          # ✗ 缺失（频繁项）
df.approxQuantile("col", [0.25, 0.75], 0.05)  # △ 有 approx_percentiles() 表达式但无 DataFrame 方法
df.stat.crosstab("col1", "col2")        # ✗ 缺失
```

**建议：**

```python
def randomSplit(
    self, weights: list[float], seed: int | None = None
) -> list["DataFrame"]:
    """将 DataFrame 随机分割为多个子集。"""
    ...

def approxQuantile(
    self, col: str | list[str], probabilities: list[float], relativeError: float
) -> list[float] | list[list[float]]:
    """计算近似分位数（包装 approx_percentiles 聚合）。"""
    ...
```

---

### 5. 其他缺失方法 (LOW PRIORITY)

| 方法 | PySpark 用法 | 建议 Daft 实现 |
|------|-------------|--------------|
| `toDF(*cols)` | `rdd.toDF(["a","b"])` | `with_columns_renamed()` 的别名 |
| `coalesce(n)` | `df.coalesce(2)` | `into_partitions(n)` 的别名 |
| `printSchema()` | `df.printSchema()` | `print(df.schema)` 的包装 |
| `df.stat` | 返回 `DataFrameStatFunctions` | 新增统计函数对象 |

---

## 内置函数缺口

### 1. 日期/时间函数 (HIGH PRIORITY)

PySpark 有但 Daft 缺失：

| 函数 | Spark SQL | 说明 | 实现位置 |
|------|-----------|------|--------|
| `datediff` | `datediff(end, start)` | 返回天数差 | `src/daft-functions-temporal/` |
| `date_diff` | `date_diff(unit, start, end)` | 指定单位的日期差 | 同上 |
| `add_months` | `add_months(date, n)` | 加减月份 | 同上 |
| `months_between` | `months_between(d1, d2)` | 返回月份差（小数） | 同上 |
| `date_add` | `date_add(date, days)` | 日期加天数 | 同上 |
| `date_sub` | `date_sub(date, days)` | 日期减天数 | 同上 |
| `next_day` | `next_day(date, "Mon")` | 下一个指定星期几 | 同上 |
| `last_day` | `last_day(date)` | 本月最后一天 | 同上 |
| `from_unixtime` | `from_unixtime(ts, fmt)` | Unix 时间戳转字符串 | 同上 |
| `unix_timestamp` | `unix_timestamp(s, fmt)` | 字符串转 Unix 时间戳 | 同上 |

**实现思路（Rust）：**
```rust
// src/daft-functions-temporal/src/lib.rs
// 参考现有 date_trunc 实现模式
fn datediff(end: &Series, start: &Series) -> DaftResult<Series> {
    // 转为日期后相减，返回整数天数
}
```

---

### 2. 字符串函数 (MEDIUM PRIORITY)

| 函数 | Spark SQL | 说明 |
|------|-----------|------|
| `format_string` | `format_string("%s is %d", c1, c2)` | printf 格式化字符串 |
| `levenshtein` | `levenshtein(s1, s2)` | 编辑距离 |
| `soundex` | `soundex(str)` | 发音相似编码 |
| `ascii` | `ascii(str)` | 第一个字符的 ASCII 码 |
| `chr` | `chr(65)` | ASCII 码转字符 |
| `concat_ws` | `concat_ws(",", c1, c2)` | 带分隔符的字符串连接 |
| `str_to_map` | `str_to_map(str, ",", ":")` | 字符串解析为 Map |
| `initcap` | `initcap(str)` | 每词首字母大写（已有 titlecase） |
| `trim` | `trim(str)` | 两端去空格（已有 lstrip/rstrip） |
| `instr` | `instr(str, substr)` | 子串位置（已有 find，可加别名） |

---

### 3. 数学函数 (LOW PRIORITY)

| 函数 | Spark SQL | 说明 |
|------|-----------|------|
| `bround` | `bround(x, 2)` | 银行家舍入（四舍六入五成双） |
| `factorial` | `factorial(5)` | 阶乘 |
| `gcd` | `gcd(a, b)` | 最大公约数 |
| `lcm` | `lcm(a, b)` | 最小公倍数 |
| `pmod` | `pmod(a, b)` | 正模数（结果永远非负） |
| `conv` | `conv(num, 10, 16)` | 进制转换 |
| `approx_median` | `percentile_approx(col, 0.5)` | 近似中位数 |

---

### 4. 数组函数 (MEDIUM PRIORITY)

Daft 已有较完整的 list 函数，以下是缺失的：

| 函数 | Spark SQL | 说明 |
|------|-----------|------|
| `array_intersect` | `array_intersect(a1, a2)` | 数组交集 |
| `array_union` | `array_union(a1, a2)` | 数组并集 |
| `array_except` | `array_except(a1, a2)` | 数组差集 |
| `arrays_zip` | `arrays_zip(a1, a2)` | 多数组压缩为结构体数组 |
| `array_position` | `array_position(arr, val)` | 元素位置（1-based） |
| `array_remove` | `array_remove(arr, val)` | 删除所有匹配元素 |
| `flatten` | `flatten(arr)` | 二维数组展平 |
| `sequence` | `sequence(1, 5)` | 生成整数序列数组 |
| `zip_with` | `zip_with(a1,a2, (x,y)->x+y)` | 高阶函数 |

**实现位置：** `src/daft-functions/src/list/` 或新建 `src/daft-functions-list/`

---

### 5. Map 函数 (MEDIUM PRIORITY)

Daft map 函数较弱，缺失：

| 函数 | Spark SQL | 说明 |
|------|-----------|------|
| `map_keys` | `map_keys(m)` | 返回所有键的数组 |
| `map_values` | `map_values(m)` | 返回所有值的数组 |
| `map_entries` | `map_entries(m)` | 返回 (key, value) 结构体数组 |
| `map_from_entries` | `map_from_entries(arr)` | 结构体数组转 Map |
| `map_concat` | `map_concat(m1, m2)` | 合并两个 Map |
| `element_at` | `element_at(m, key)` | 取 Map 中某 key 的值 |
| `map_filter` | `map_filter(m, (k,v)->v>0)` | 过滤 Map 条目 |

---

### 6. 窗口函数 (MEDIUM PRIORITY)

已有：`row_number`, `rank`, `dense_rank`, `lag`, `lead`

缺失：

| 函数 | Spark SQL | 说明 |
|------|-----------|------|
| `ntile` | `ntile(4)` | 分位分桶 |
| `first_value` | `first_value(col)` | 窗口第一个值 |
| `last_value` | `last_value(col)` | 窗口最后一个值 |
| `nth_value` | `nth_value(col, 2)` | 窗口第 N 个值 |
| `percent_rank` | `percent_rank()` | 百分比排名 |
| `cume_dist` | `cume_dist()` | 累积分布 |

**实现位置：** `src/daft-functions/src/` 中的 window 相关模块

---

### 7. 聚合函数 (LOW PRIORITY)

| 函数 | Spark SQL | 说明 |
|------|-----------|------|
| `collect_list` | `collect_list(col)` | 收集为列表（已有 `list_agg`） |
| `collect_set` | `collect_set(col)` | 收集为去重集合（已有 `list_agg_distinct`） |
| `first` | `first(col, ignorenulls)` | 分组第一个值（已有 `any_value`） |
| `last` | `last(col, ignorenulls)` | 分组最后一个值 |
| `percentile_approx` | `percentile_approx(col, 0.5)` | 近似百分位数（已有 `approx_percentiles`） |
| `kurtosis` | `kurtosis(col)` | 峰度 |
| `regr_slope` | `regr_slope(y, x)` | 线性回归斜率 |
| `regr_intercept` | `regr_intercept(y, x)` | 线性回归截距 |

---

## Rust 执行引擎性能优化

### P0 - 紧急（会导致功能缺失/OOM）

#### 1. Spill-to-Disk 机制（最高优先级）

**问题：** Sort 和 Hash Join 无溢出处理，大数据集直接 OOM。

```
# 受影响文件
src/daft-local-execution/src/sinks/sort.rs       # Building(Vec<..>) 无大小限制
src/daft-local-execution/src/join/hash_join.rs   # tables: Vec<RecordBatch> 无限制
src/daft-local-execution/src/resource_manager.rs  # 内存不足直接报错
```

**建议方案：**

```rust
// sort.rs：实现外排序
pub(crate) enum SortState {
    Building {
        partitions: Vec<Arc<MicroPartition>>,
        memory_used: usize,          // 新增：内存追踪
        spilled_files: Vec<PathBuf>, // 新增：溢出文件列表
    },
    Done,
}

// 当 memory_used > threshold 时，将已有数据排序后写入临时文件
// 最终做多路归并
```

#### 2. Broadcast Join 自动检测

**问题：** 无自动广播小表机制，多表 join 性能差。

```
# 受影响文件
src/daft-logical-plan/src/optimization/rules/reorder_joins/
src/daft-logical-plan/src/stats.rs
```

**建议方案：**

```rust
// 新增优化规则：broadcast_join_detection.rs
struct BroadcastJoinDetection;

impl OptimizerRule for BroadcastJoinDetection {
    fn apply(&self, plan: LogicalPlan) -> Result<LogicalPlan> {
        // 如果 build 侧估计行数 < 1_000_000 且大小 < 1GB
        // 自动将 join 策略改为 broadcast
    }
}
```

#### 3. Join 顺序优化（O(n!) → DP）

**问题：** 现有暴力搜索算法对 5+ 表 join 退化。

```
# 受影响文件
src/daft-logical-plan/src/optimization/rules/reorder_joins/brute_force_join_order.rs
```

**建议：** 实现动态规划 join 顺序（Selinger 算法），对 4 表以下用暴力，5 表以上用 DP。

---

### P1 - 高优先级（影响性能 10x+）

#### 4. 列级统计信息（CBO 基础）

**问题：** `ApproxStats` 仅有 `num_rows` 和 `size_bytes`，无列统计，CBO 无法工作。

```
# 受影响文件
src/daft-logical-plan/src/stats.rs  第106-129行
```

**建议扩展：**

```rust
pub struct ColumnStats {
    pub min_value: Option<ScalarValue>,
    pub max_value: Option<ScalarValue>,
    pub null_count: Option<usize>,
    pub ndv: Option<usize>,          // Number of Distinct Values
    pub selectivity: f64,
}

pub struct ApproxStats {
    pub num_rows: CountMode,
    pub size_bytes: CountMode,
    pub column_stats: HashMap<String, ColumnStats>,  // 新增
    pub acc_selectivity: f64,
}
```

#### 5. 分区修剪改进

**问题：** 当前分区修剪仅基于 Hive 分区路径，无统计感知。

```
# 受影响文件
src/daft-scan/src/pushdowns.rs
```

**建议：** 添加 `partition_statistics` 字段，利用 Parquet/Iceberg 页面级统计跳过数据文件。

---

### P2 - 中优先级（性能提升 2-5x）

#### 6. 自适应聚合策略

```
# 受影响文件
src/daft-local-execution/src/sinks/grouped_aggregate.rs
```

**问题：** 分组策略阈值硬编码（第70行），无运行时自适应。

**建议：** 基于实时内存使用和数据分布动态选择 `AggThenPartition` vs `PartitionThenAgg`。

#### 7. Bloom Filter 下推

在扫描层增加 Bloom Filter 谓词，跳过不匹配的 Row Group：

```
# 受影响文件
src/daft-scan/src/pushdowns.rs        # 添加 bloom_filters 字段
src/daft-parquet/src/read.rs          # 利用 Parquet Bloom Filter 元数据
```

#### 8. Sort-Merge Join 优化

```
# 受影响文件
src/daft-local-execution/src/join/sort_merge_join.rs
```

**建议：** 如果输入已按连接键排序，跳过重排序步骤（利用 `SortInfo` 传播）。

---

### P3 - 低优先级（长期改进）

#### 9. 表达式预编译缓存

相同表达式多次执行时缓存编译结果，减少解析开销。

#### 10. SIMD 向量化

Join 等值检查、聚合累加等热路径加入 SIMD 优化：

```
# 受影响文件
src/daft-recordbatch/src/ops/joins/hash_join.rs  # 等值检查
src/daft-recordbatch/src/ops/agg.rs              # 聚合累加
```

---

## IO 层改进

### 1. 缺失的文件格式

| 格式 | 优先级 | 理由 | 实现思路 |
|------|--------|------|---------|
| **ORC** | HIGH | Hive 生态标准、PySpark 默认写入格式之一 | 新建 `src/daft-orc/`，使用 `orc-rust` crate |
| **Avro** | MEDIUM | Kafka Schema Registry 标准、Schema Evolution | 新建 `src/daft-avro/`，使用 `apache-avro` crate |
| **Excel (.xlsx)** | MEDIUM | 商业数据交换场景广泛 | Python 层集成 `openpyxl` |
| **XML** | LOW | 企业系统集成 | Python 层集成 |

**ORC 实现路径：**
```
src/daft-orc/
├── Cargo.toml
├── src/
│   ├── lib.rs
│   ├── reader.rs   # 读取：ORC → RecordBatch
│   └── writer.rs   # 写入：RecordBatch → ORC

daft/io/_orc.py     # Python API
```

---

### 2. 写入功能增强

#### 2.1 压缩支持扩展

```python
# 当前 Parquet 写入只有 snappy
# 建议支持：
df.write_parquet(path, compression="zstd")    # ✓ 已部分支持
df.write_csv(path, compression="gzip")        # ✗ 缺失
df.write_json(path, compression="gzip")       # ✗ 缺失
```

**实现位置：** `daft/io/writer.py` 中的 `CSVFileWriter` 和 JSON 写入器

#### 2.2 小文件合并

**问题：** 多分区并行写入会产生大量小文件，查询性能差。

```python
# 建议添加 coalesce_files 参数
df.write_parquet(path, max_file_size_bytes=128 * 1024 * 1024)
```

**实现思路：** 在 `src/daft-writers/src/` 中实现 `CoalescingWriter`，缓冲多个 MicroPartition 直到达到目标大小。

#### 2.3 Hive Metastore 集成

```
# 缺失 Catalog 实现
daft/catalog/__hive.py   # 新文件
```

**建议：** 通过 PyHive / `pymetastore` 连接 Hive Metastore，读取表元数据和分区信息。

---

### 3. Schema Evolution 改进

#### Delta Lake `merge` 模式

```python
# 当前：schema_mode="merge" 会报 NotImplementedError
# daft/dataframe/dataframe.py:1421-1422
df.write_deltalake(table, mode="append", schema_mode="merge")  # ✗
```

**建议：** 接入 `deltalake` Python 包的 `MERGE schema evolution` 支持。

#### Iceberg Schema Evolution

完整支持列新增、列重命名、类型提升等 Iceberg schema evolution 操作。

---

### 4. 读取功能增强

#### 分区统计利用

读取 Parquet/Iceberg 时利用行组统计（min/max/null_count）跳过不满足谓词的行组：

```
# 受影响文件
src/daft-parquet/src/read.rs  # 添加 row_group_skip_stats 逻辑
src/daft-scan/src/pushdowns.rs
```

#### 流式 Kafka 读取

```python
# 当前：只有有界读取 read_kafka()
# 建议：添加流式读取 API（实验性）
daft.read_kafka_stream(brokers, topic, schema)
```

---

## SQL 支持改进

### 1. 缺失的 SQL 函数

以下函数在 Spark SQL 中有但 Daft SQL 解析器尚未支持：

#### 日期函数
```sql
-- 缺失
SELECT datediff('2024-01-10', '2024-01-01')     -- 返回 9
SELECT date_add('2024-01-01', 10)               -- 返回 '2024-01-11'
SELECT add_months('2024-01-31', 1)              -- 返回 '2024-02-29'
SELECT months_between('2024-03-01', '2024-01-01') -- 返回 2.0
SELECT last_day('2024-02-15')                   -- 返回 '2024-02-29'
SELECT next_day('2024-01-01', 'MON')            -- 返回 '2024-01-07'
```

#### 字符串函数
```sql
-- 缺失
SELECT format_string('%s scored %d', name, score)
SELECT levenshtein('kitten', 'sitting')   -- 编辑距离 3
SELECT soundex('Smith')                   -- 返回 'S530'
SELECT concat_ws(',', col1, col2, col3)
```

#### 条件函数
```sql
-- 建议完善
SELECT nullif(col1, col2)   -- col1 == col2 时返回 NULL
SELECT ifnull(col1, 0)      -- col1 IS NULL 时返回 0（Spark 别名）
SELECT nvl(col1, 0)         -- 同上
SELECT nvl2(col1, v1, v2)   -- col1 非 NULL 返回 v1，否则 v2
```

---

### 2. SQL 窗口函数完善

```sql
-- 已支持
SELECT row_number() OVER (PARTITION BY dept ORDER BY salary DESC) ...
SELECT rank(), dense_rank(), lag(col, 1), lead(col, 1) ...

-- 缺失
SELECT ntile(4) OVER (ORDER BY score) ...           -- 分桶排名
SELECT percent_rank() OVER (ORDER BY score) ...     -- 百分比排名
SELECT cume_dist() OVER (ORDER BY score) ...        -- 累积分布
SELECT first_value(col) OVER (...) ...              -- 窗口第一值
SELECT last_value(col) OVER (...) ...               -- 窗口最后值
SELECT nth_value(col, 2) OVER (...) ...             -- 窗口第 N 值
```

---

### 3. ROLLUP / CUBE SQL 语法

```sql
-- 缺失
SELECT dept, year, SUM(sales)
FROM t
GROUP BY ROLLUP(dept, year)

SELECT dept, year, SUM(sales)
FROM t
GROUP BY CUBE(dept, year)

SELECT dept, GROUPING_ID(dept, year), SUM(sales)
FROM t
GROUP BY GROUPING SETS ((dept), (year), ())
```

---

## 优先级路线图

### 阶段一：紧急修复（1-2 周内）

| 功能 | 类型 | 影响 | 实现文件 |
|------|------|------|---------|
| Spill-to-disk（Sort） | 引擎 | 大数据集可用性 | `sinks/sort.rs` |
| Spill-to-disk（Hash Join） | 引擎 | 大数据集可用性 | `join/hash_join.rs` |
| `cache()` / `persist()` | Python API | 用户体验 | `dataframe.py` |
| `datediff()` / `date_diff()` | 函数 | PySpark 迁移 | `daft-functions-temporal/` |

---

### 阶段二：重要功能（2-4 周）

| 功能 | 类型 | 影响 |
|------|------|------|
| Broadcast Join 自动检测 | 引擎优化 | Join 性能 10x+ |
| 列级统计信息（CBO 基础） | 引擎优化 | 查询计划质量 |
| ORC 格式读写 | IO | Hive 生态对齐 |
| `rollup()` / `cube()` | Python API | PySpark 对齐 |
| 日期函数完整实现 | 函数 | PySpark 迁移 |
| `.na` 对象 | Python API | PySpark 迁移 |

---

### 阶段三：增强功能（1-2 月）

| 功能 | 类型 | 影响 |
|------|------|------|
| DP Join 顺序优化 | 引擎优化 | 多表 join 性能 |
| Bloom Filter 下推 | 引擎优化 | 扫描跳过率提升 |
| Avro 格式支持 | IO | Kafka 生态对齐 |
| Hive Metastore 集成 | IO | 企业部署 |
| 字符串/数组函数完整性 | 函数 | PySpark 对齐 |
| `randomSplit()` / `approxQuantile()` | Python API | PySpark 对齐 |
| ROLLUP/CUBE SQL 语法 | SQL | PySpark 对齐 |

---

### 阶段四：长期优化（3 月+）

| 功能 | 类型 | 说明 |
|------|------|------|
| 小文件合并写入器 | IO | 生产就绪 |
| 外排序完整实现 | 引擎 | 无限数据集支持 |
| SIMD 向量化（Join/Agg 热路径） | 引擎 | 单核性能 2x |
| 表达式 JIT | 引擎 | 复杂查询加速 |
| 跨 Worker 动态统计 | 分布式 | 分布式 CBO |

---

## 附录：关键文件路径索引

### Python 层
```
daft/dataframe/dataframe.py    # DataFrame 类（145 个方法），5304 行
daft/expressions/expressions.py # Expression 类（318 个方法）
daft/functions/__init__.py     # 函数导出（200+ 个）
daft/functions/datetime.py     # 日期时间函数
daft/functions/str.py          # 字符串函数
daft/functions/list.py         # 列表函数
daft/functions/agg.py          # 聚合函数
daft/functions/window.py       # 窗口函数
daft/io/writer.py              # 所有写入器实现
daft/io/sink.py                # 自定义 Sink 接口
daft/catalog/__init__.py       # Catalog 主接口
```

### Rust 层（优化重点）
```
src/daft-local-execution/src/sinks/sort.rs             # 排序（需 spill）
src/daft-local-execution/src/join/hash_join.rs         # Hash Join（需 spill）
src/daft-local-execution/src/resource_manager.rs       # 内存管理
src/daft-logical-plan/src/stats.rs                     # 统计信息（需扩展）
src/daft-logical-plan/src/optimization/rules/          # 优化规则（30+ 条）
src/daft-scan/src/pushdowns.rs                         # 谓词下推
src/daft-functions-temporal/src/lib.rs                 # 时间函数（需扩展）
src/daft-recordbatch/src/ops/joins/hash_join.rs        # 底层 join 实现
```
