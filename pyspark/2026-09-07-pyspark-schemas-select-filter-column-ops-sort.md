# PySpark — Sept 7 (Schemas, Select, Filter, Column Ops, Sort)

> [!summary] Core idea
> Today was about **shape and content control**: defining schema precisely, selecting/renaming/deriving columns, filtering/sorting rows.
>
> Advanced thread: **what can Spark push toward the source, and what must happen inside Spark?**

---

## 1. Reading JSON — `multiLine`

```python
df_json = (spark.read.format('json')
    .option('inferSchema', True)
    .option('multiLine', False)
    .load(path))
```

| Setting | Expected shape | Parallelism |
|---|---|---|
| `multiLine=False` (default) | JSON Lines / NDJSON — one object per line | Generally splittable / parallel-friendly |
| `multiLine=True` | One JSON structure spanning multiple lines | Harder to split; can reduce parallelism for large files |

> [!tip]
> `multiLine=True` changes the parsing/file-splitting problem; it isn't merely a formatting switch. For large datasets, **NDJSON is generally friendlier to distributed processing**.

---

## 2. Schemas — DDL vs `StructType`

```python
schema = "Item_Identifier STRING, Item_MRP DOUBLE, Outlet_Establishment_Year INT"
df = spark.read.csv(path, header=True, schema=schema)
```

```python
from pyspark.sql.types import *

schema = StructType([
    StructField('Item_Identifier', StringType(), True),
    StructField('Item_MRP', DoubleType(), True),
    StructField('Outlet_Establishment_Year', IntegerType(), True)
])
df = spark.read.csv(path, header=True, schema=schema)
```

| DDL string | `StructType` |
|---|---|
| Compact, SQL-like | More verbose, programmatic |
| Good for straightforward schemas | Better for nested/programmatically generated schemas |
| Less convenient for field metadata | `StructField` supports metadata |

`True` in `StructField(..., True)` = **nullable**; it means the field may contain nulls.

### Important types

- Text → `StringType`
- Integers → `ByteType`, `ShortType`, `IntegerType`, `LongType`
- Decimal → `DecimalType(precision, scale)`
- Floating point → `FloatType`, `DoubleType`
- Date/time → `DateType`, `TimestampType`, `TimestampNTZType`
- Boolean → `BooleanType`
- Nested → `ArrayType`, `MapType`, `StructType`
- Raw bytes → `BinaryType`

> [!warning]
> **Money → prefer `DecimalType`, not Float/Double.** Binary floating point can introduce rounding error.

---

## 3. `select()` — projection

```python
df.select('Item_Fat_Content', 'Item_Identifier')
df.select(col('Item_Identifier').alias('Item_ID'))
df.selectExpr('Item_MRP * 2 AS double_mrp')
```

`select()` narrows columns and is a **narrow transformation** — no shuffle.

- Strings → simple column references
- `col()` → useful for expressions/aliases/casts
- `expr()` / `selectExpr()` → SQL-style expressions

### Projection pushdown

| Source | Effect of selecting fewer columns |
|---|---|
| **Parquet / ORC / Delta** | Columnar storage can read only required columns → real I/O savings |
| **CSV / JSON** | Rows still need to be parsed; logical pruning doesn't provide the same read savings |
| **JDBC / remote sources** | Column list can often be pushed into remote query |

**Key idea:** logical projection ≠ guaranteed physical I/O reduction; it depends on the source.

---

## 4. `filter()` / `where()`

`filter()` and `where()` are aliases.

```python
df.filter(col('Item_Fat_Content') == 'Regular')

df.filter(
    (col('Item_Type') == 'Soft Drinks') &
    (col('Item_Weight') < 10)
)

df.filter(
    col('Outlet_Size').isNull() &
    col('Outlet_Location_Type').isin(['Tier 1', 'Tier 2'])
)
```

> [!danger] Spark Column expressions ≠ Python booleans
> Use `&`, `|`, `~` — **not** Python `and`, `or`, `not`.
>
> Wrap each comparison in parentheses.

Nulls:
- `isNull()` / `isNotNull()` → correct null checks
- Don't rely on Python-style `== None` semantics
- `isin([...])` → SQL-style `IN`

### Chained filters

```python
df.filter(col('A') == 1).filter(col('B') == 2)
```

and

```python
df.filter((col('A') == 1) & (col('B') == 2))
```

are logically equivalent. Catalyst will typically merge consecutive filters during optimization, but verify with `explain()` when performance matters.

### Predicate pushdown

Spark can sometimes push filters closer to the source, reducing data read/transfer.

| Source | Possible optimization |
|---|---|
| Parquet / ORC / Delta | File/row-group skipping using statistics when predicates are selective |
| JDBC / Snowflake | Predicate can often become part of the remote query |
| CSV / JSON | Rows generally still need to be read/parsed; no comparable file-stat skipping |

> [!warning]
> A Python UDF generally cannot be translated into a source query, so it can prevent pushdown. **Use built-in Spark functions where possible.**

Check `explain()` for physical-plan evidence such as `PushedFilters` rather than assuming pushdown occurred.

---

## 5. Column operations

```python
df.withColumnRenamed('Item_Weight', 'Item_Wt')

df.withColumn('flag', lit('new'))

df.withColumn('revenue', col('Item_Weight') * col('Item_MRP'))

df.withColumn('Item_Weight', col('Item_Weight').cast('double'))
```

| Operation | Meaning |
|---|---|
| `withColumnRenamed()` | Rename a column; no data transformation/shuffle |
| `withColumn()` | Add a column, or replace one if the name already exists |
| `lit()` | Turns a Python literal into a Spark `Column` expression |
| `cast()` | Converts the expression to another Spark type |
| `regexp_replace()` | String replacement using regex |

Chained replacements happen **in order**, so later replacements see earlier changes.

> [!warning] Casts can create silent nulls
> In normal non-ANSI behavior, a value such as `"abc"` cast to `int` can become `null` instead of raising an exception. Validate cast results when input quality is uncertain.

> [!info] Many `withColumn()` calls
> A few are fine. Hundreds chained in a loop can make logical-plan analysis unnecessarily large. For large-scale uniform column transformations, consider constructing expressions and using one `select()`.

---

## 6. Sorting — `sort()` / `orderBy()`

```python
df.sort(col('Item_Weight').desc())
df.sort(['Item_Weight', 'Item_Visibility'], ascending=[0, 1])
```

`sort()` and `orderBy()` are aliases.

A **global sort is a wide transformation** and normally requires a shuffle because rows must be redistributed to establish global ordering.

> [!tip]
> `orderBy(...).limit(10)` can be optimized into a Top-K style execution instead of a full global sort. Check `explain()` for the actual physical plan.

---

## 7. `limit()` and `drop()`

```python
df.limit(10)
df.drop('Item_Visibility')
```

- `limit()` is lazy until a terminal action follows.
- `drop()` is a projection-like narrow operation.
- As with `select()`, actual I/O savings from dropping columns depend on the source format.

---

## ⚡ What to remember

- `multiLine=True` can reduce file-splitting/parallelism for large JSON; NDJSON is usually better for distributed processing.
- DDL and `StructType` describe the same schema; `StructType` is better for programmatic/nested schemas.
- `nullable=True` means **may contain null**, not "must be provided."
- `select()` / `drop()` don't automatically mean less disk I/O — source format matters.
- Use `&` / `|` / `~` + parentheses for Spark filter expressions.
- `isNull()` / `isNotNull()` for null checks.
- Built-in Spark functions are preferable to Python UDFs when possible because Catalyst can optimize them and pushdown may remain possible.
- `cast()` can silently produce nulls; validate untrusted input.
- `select`, `filter`, `withColumn`, `drop` are narrow; global `sort`/`orderBy` is wide and normally shuffles.
- **Use `explain()` to verify optimizer behavior instead of assuming it.**

## Next connections

`Narrow vs Wide` → `Shuffle internals` → `Partitions` → `repartition/coalesce` → `Joins` → `Aggregations` → `Windows` → `explain()`
