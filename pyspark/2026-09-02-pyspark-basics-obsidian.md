# PySpark Basics — Sept 2

> [!summary] Core idea
> **Reading = build a DataFrame. Display/inspection of rows = execution. Actions = trigger Spark to execute the lazy plan.**
>
> Most important mental model: **syntax is not execution**. Ask: *Does this execute? What gets scanned/shuffled? Does data come to the driver?*

---

## 1. Reading DataFrames

PySpark uses `spark.read` (`DataFrameReader`) to create DataFrames.

### Common ways

```python
# Simple / quick
df = spark.read.csv(path, header=True, inferSchema=True)

# Generic format + options
df = (
    spark.read
        .format("csv")
        .option("header", True)
        .option("inferSchema", True)
        .load(path)
)

# Several options
df = (
    spark.read
        .options(header=True, inferSchema=True, delimiter=",")
        .csv(path)
)

# Explicit schema — preferred when schema is known
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

schema = StructType([
    StructField("column1", StringType(), True),
    StructField("column2", IntegerType(), True)
])

df = spark.read.csv(path, header=True, schema=schema)
```

### Side-by-side

| Method | Use when | Important point |
|---|---|---|
| `read.csv()` | Simple CSV loading | Shortest syntax |
| `.format().load()` | Generic/reusable pattern | Works across formats |
| `.options().csv()` | Many reader options | Cleaner configuration |
| `inferSchema=True` | Exploration / unknown schema | Requires schema inference work; avoid when schema is known |
| Explicit `schema=` | Production / known schema | Predictable; avoids inference |
| List of files | Specific files | Reads them as one logical DataFrame |
| `*.csv` / directory | Batch of files | Watch for the **small-files problem** |

Common formats: CSV, JSON, Parquet, ORC, Delta.

Useful CSV options: `header`, `inferSchema`, `schema`, `sep`/`delimiter`, `quote`, `escape`, `nullValue`, `dateFormat`, `timestampFormat`.

> [!tip] Production rule
> **Known schema → explicit schema. Unknown/exploration → inference.**
>
> Different reader syntaxes generally do **not** mean different Spark execution engines; they are mostly different ways to configure the same reader.

---

## 2. Display & Inspect DataFrames

First separate **metadata inspection** from **row retrieval**.

### Side-by-side

| Method | Executes? | Driver gets rows? | Use |
|---|---:|---:|---|
| `df.printSchema()` | No | No | Inspect schema |
| `df.schema` | No | No | Programmatic schema |
| `df.columns` / `df.dtypes` | No | No | Column metadata |
| `df.show(10)` | **Yes** | Limited | Standard PySpark preview |
| `df.display()` | **Yes** | Limited | Databricks interactive preview |
| `df.limit(10)` | No* | No* | Limit the logical plan |
| `df.take(10)` | **Yes** | 10 rows | Get small result in Python |
| `df.head(10)` | **Yes** | 10 rows | Similar to `take()` |
| `df.first()` | **Yes** | 1 row | Get first row |
| `df.count()` | **Yes** | Only count | Row count; can still be expensive |
| `df.sample(...)` | No* | No* | Create a sample plan |

\* Becomes part of execution when followed by an action.

### `show()` vs `display()`

- `show()` → standard PySpark API; good for scripts/notebooks.
- `display()` → **Databricks-specific** interactive visualization.
- Both execute when used to retrieve/display rows.
- Use explicit limits for notebook exploration.

```python
df.show(10)
df.show(20, truncate=False)

df.limit(100).display()
df.take(5)
df.first()
```

### Sampling

```python
df.sample(withReplacement=False, fraction=0.1, seed=42).show()
```

Sampling has execution cost; don't assume it is free.

```python
df.orderBy(rand()).limit(1000)
```

This can be expensive because global ordering is expensive. Don't use it as a cheap random sample.

### `count()` warning

`df.count()` returns only one number, so **driver-memory risk is low**, but it can still require substantial computation/full relevant scan.

```python
df.count()
df.filter(condition).count()
df.select("column").distinct().count()
```

> **Low driver memory ≠ low compute cost.**

### Practical inspection flow

```python
df.printSchema()       # cheap metadata inspection
df.limit(10).show()    # small preview
# df.count()           # only when the count is actually needed
```

For large/production datasets, don't repeatedly `display()`/`count()` just to inspect data.

---

## 3. Actions

### The core rule

**Transformations are lazy. Actions trigger execution of the DAG.**

Examples of transformations:

`select`, `filter`, `withColumn`, `join`, `groupBy`, `sample`, `limit`, `union`, `intersect`, `subtract`

Examples of actions:

`show`, `count`, `collect`, `take`, `first`, `write`

> [!warning] Action ≠ cheap
> An action means **execute**. It does not mean the operation is inexpensive.

### Actions / execution-related operations by category

| Category | Examples | Driver risk | Key point |
|---|---|---|---|
| **Retrieve rows** | `show`, `take`, `head`, `first` | Low → medium | Limited results, but computation still happens |
| **Collect everything** | `collect()` | 🔴 High | Entire result comes to driver; OOM risk |
| **Count** | `count()` | Low memory | Can still be a costly scan |
| **Stats** | `describe()`, `summary()`, `agg()` | Usually low | Computation happens on cluster |
| **Write** | `write.csv`, `write.parquet`, `saveAsTable`, Delta writes | Low driver risk | Distributed execution + I/O |
| **Iteration / side effects** | `foreach`, `foreachPartition` | Medium | Use carefully; prefer built-in Spark functions for normal transformations |
| **Local conversion** | `toPandas()` | 🔴 High | Collects data to driver |
| **Local iteration** | `toLocalIterator()` | Medium | Streams partitions to driver rather than collecting all at once |
| **RDD-style** | `reduce`, `fold`, `treeReduce` | Medium | Mostly relevant to RDD APIs |
| **SQL** | `spark.sql(...).show()` | Depends | SQL query is lazy; `show()` triggers it |

### Operations commonly confused with actions

These **do not execute by themselves**:

```python
df.limit(100)
df.sample(0.1)
df.groupBy("country")
df.union(other_df)
df.intersect(other_df)
df.subtract(other_df)
```

`groupBy()` returns a `GroupedData`; an aggregation such as:

```python
df.groupBy("country").count()
```

builds the aggregation, and a terminal action such as `.show()` triggers execution.

### Cache / persist

`cache()` and `persist()` are **not actions**.

```python
df.cache()
df.count()     # first action can materialize/cache it
df.show()      # later actions may reuse cached data
```

Don't cache everything. Cache when the same expensive DataFrame is reused and the storage/memory cost is justified.

### Checkpoint

```python
df.checkpoint()
df.localCheckpoint()
```

Used to materialize data and/or cut long lineage.

- `checkpoint()` → reliable checkpoint storage.
- `localCheckpoint()` → executor-local storage; less reliable.

### SQL / temporary views

```python
df.createOrReplaceTempView("rides")

result = spark.sql("""
    SELECT country, COUNT(*)
    FROM rides
    GROUP BY country
""")

result.show()   # action → execution
```

Creating the temp view itself is not a terminal execution of the query.

### `explain()`

```python
df.explain()
df.explain(True)
```

**Not an action.** It helps inspect the logical/physical plan and becomes important for performance debugging.

---

## ⚡ What to remember

1. **DataFrame creation is usually lazy.**
2. **Transformations build a plan; actions execute it.**
3. `printSchema()` / metadata inspection is cheap.
4. `show()` / `display()` execute to retrieve rows.
5. `count()` returns one value but can still be expensive.
6. `collect()` / `toPandas()` can move huge amounts of data to the **driver** → OOM risk.
7. `limit()` is lazy; `limit(...).show()` executes.
8. `groupBy()` is not itself an action.
9. `cache()` / `persist()` are not actions.
10. `explain()` is not an action.
11. **Always think about scan, shuffle, partitions, and driver-vs-executor movement—not just the Python syntax.**

### Next connections

**Driver & Executors → Partitions → Jobs/Stages/Tasks → DAG → Narrow/Wide → Shuffle → Joins → `explain()` → Performance**
