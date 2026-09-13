# PySpark — Sept 13 (collect_list/set, Pivot, when/otherwise, Joins, Window Functions)

> Goal: understand what these functions actually do, when to use them, when **not** to use them, and the ugly edge cases that matter in real DE work.

---

## 0. Quick mental map

| Topic | Main purpose | Usually expensive? | Main thing to watch |
|---|---|---:|---|
| `collect_list()` | Gather values into an array, keeping duplicates | Yes | Arrays can become huge; order is not guaranteed after shuffle |
| `collect_set()` | Gather unique values into an array | Yes | Deduplication + potentially huge arrays; order is not guaranteed |
| `pivot()` | Turn row values into columns for reporting/analysis | Yes | Wide output, many columns, aggregation required |
| `when().otherwise()` | Conditional column logic (`if / elif / else`) | Usually no shuffle | NULL behavior and missing `otherwise()` |
| joins | Combine rows from DataFrames using keys/conditions | Often | Join type, duplicate keys, shuffle, skew, broadcast |
| window functions | Calculate across related rows **without collapsing them** | Often | Partition size, ordering, shuffle, and window definition |
| `row_number()` | Assign a unique sequence within each window partition | Often | Requires a meaningful `orderBy()` if you care which row is #1 |

---

# 1. `collect_list()` and `collect_set()`

## What they do

Both are aggregation functions that collect multiple values into an **array per group**.

```python
from pyspark.sql.functions import collect_list, collect_set

# Keep duplicates
df.groupBy("customer_id").agg(
    collect_list("product").alias("products")
)

# Remove duplicates
df.groupBy("customer_id").agg(
    collect_set("product").alias("unique_products")
)
```

Example input:

```text
customer_id | product
------------|--------
101         | Phone
101         | Case
101         | Phone
102         | Laptop
102         | Mouse
```

`collect_list()`:

```text
101 → [Phone, Case, Phone]
102 → [Laptop, Mouse]
```

`collect_set()`:

```text
101 → [Phone, Case]
102 → [Laptop, Mouse]
```

## Difference

| | `collect_list` | `collect_set` |
|---|---|---|
| Output | Array | Array |
| Duplicates | Kept | Removed |
| Typical use | Preserve all occurrences | Unique values only |
| Ordering | Not guaranteed | Not guaranteed |
| Aggregation | Yes | Yes |

## Good use cases

### `collect_list()`

Useful when the individual values need to be retained together:

- customer → all products purchased
- order → all item IDs
- user → all event types seen

### `collect_set()`

Useful when only distinct values matter:

- customer → unique product categories
- user → unique countries visited
- account → unique channels used

## The ugly part ⚠️

These functions can create **very large arrays**.

A customer with millions of events can produce a massive single grouped value. That can increase memory pressure and make the aggregation expensive.

Also, **do not assume the resulting array is sorted**.

If business logic needs a deterministic order, explicitly define how the data should be ordered rather than assuming `collect_list()` preserves input order.

## Important classification

Because these are aggregation functions used with `groupBy()`, they generally involve a **wide/shuffle dependency** when the grouping key is distributed across partitions.

```text
partitions
   ↓
local aggregation
   ↓
shuffle by grouping key
   ↓
final collection
```

## When NOT to use them

Avoid them when:

- you only need a count/sum/average
- the resulting arrays can become enormous
- you actually need one specific record rather than all values
- you need deterministic ordering but haven't explicitly designed for it

If the real requirement is **"give me the latest record per customer"**, `collect_list()` is usually the wrong tool. A window + `row_number()` is often a better fit.

---

# 2. `pivot()`

## What it does

`pivot()` turns distinct values from rows into **columns**.

Think:

```text
Rows → Columns
```

Example input:

```text
month | category | sales
------|----------|------
Jan   | Phone    | 100
Jan   | Laptop   | 200
Feb   | Phone    | 150
Feb   | Laptop   | 300
```

A pivot can produce:

```text
month | Laptop | Phone
------|--------|------
Jan   | 200    | 100
Feb   | 300    | 150
```

## Syntax

```python
df.groupBy("month").pivot("category").sum("sales")
```

Multiple aggregations are possible depending on the API/use case, but the core pattern is:

```python
df.groupBy(<grouping_columns>) \
  .pivot(<column_to_turn_into_columns>) \
  .agg(<aggregation>)
```

## Why is aggregation involved?

Suppose you have:

```text
Jan | Phone | 100
Jan | Phone | 150
```

If `Phone` becomes a column, Spark still needs to decide what value belongs in the `Jan / Phone` cell.

That is why pivot is normally paired with an aggregation:

```python
.sum("sales")
.avg("sales")
.count()
```

## Good use cases

- reporting tables
- dashboard-friendly wide outputs
- month/category summaries
- cross-tab analysis
- converting categorical row values into a small, known set of columns

## The bad/ugly part ⚠️

Pivot can create **many columns** if the pivot column has high cardinality.

For example:

```text
pivot("user_id")
```

could attempt to create an enormous number of columns.

That is usually a terrible design.

Prefer pivoting on a **small, controlled categorical domain** such as:

```text
month
region
product_category
status
```

rather than something like:

```text
customer_id
transaction_id
email
```

## Explicit pivot values

If the possible values are known and limited, explicitly supplying them can be preferable:

```python
df.groupBy("month") \
  .pivot("category", ["Phone", "Laptop", "Tablet"]) \
  .sum("sales")
```

This makes the expected output schema clearer and avoids relying on Spark to discover all pivot values.

## Performance mental model

Pivot is not a simple column rename. It is an aggregation/restructuring operation and can involve substantial distributed work.

Treat it as a potentially expensive transformation, especially on large datasets or high-cardinality pivot columns.

---

# 3. `when()` / `otherwise()`

## What they do

These provide conditional logic for building a Spark `Column` expression.

Think:

```text
if / elif / else
```

## Basic syntax

```python
from pyspark.sql.functions import when, col

result = df.withColumn(
    "category",
    when(col("age") >= 60, "Senior")
    .when(col("age") >= 18, "Adult")
    .otherwise("Minor")
)
```

Conceptually:

```text
if age >= 60:
    Senior
elif age >= 18:
    Adult
else:
    Minor
```

## Multiple conditions

```python
df.withColumn(
    "status",
    when(col("amount") >= 10000, "High")
    .when(col("amount") >= 5000, "Medium")
    .otherwise("Low")
)
```

## Important: order matters

Conditions are evaluated in order.

```python
when(col("amount") > 100, "A") \
.when(col("amount") > 1000, "B")
```

The `> 1000` rows already satisfy `> 100`, so they can be classified as `A` before the second condition gets a chance.

Better ordering:

```python
when(col("amount") > 1000, "B") \
.when(col("amount") > 100, "A") \
.otherwise("C")
```

## `otherwise()` matters

If no condition matches and there is no `otherwise()`, the result can be `NULL`.

So if every row must receive a value, explicitly provide a fallback.

## NULLs

Be careful with conditions involving NULL:

```python
when(col("age") > 25, "Adult")
```

A NULL age does not satisfy `age > 25` as TRUE. If there is no later matching condition, it can fall through to `otherwise()` or become NULL if no `otherwise()` exists.

For explicit NULL handling:

```python
when(col("age").isNull(), "Unknown") \
.otherwise("Known")
```

## Good use cases

- categorization
- data-quality flags
- derived business status
- conditional transformations
- replacing simple nested Python `if` logic in DataFrame expressions

## When NOT to use

Don't build enormous, unreadable chains of `when()` conditions when the logic is really a lookup/mapping table. A small mapping DataFrame + join can be cleaner and more maintainable.

---

# 4. Joins

## What a join does

A join combines rows from two DataFrames using a matching condition.

Example:

```text
customers
customer_id | name
------------|------
1           | A
2           | B

orders
customer_id | amount
------------|-------
1           | 500
1           | 200
3           | 100
```

```python
customers.join(
    orders,
    customers.customer_id == orders.customer_id,
    "inner"
)
```

Result for an inner join:

```text
customer_id | name | amount
------------|------|-------
1           | A    | 500
1           | A    | 200
```

## Join types

| Join | Keeps |
|---|---|
| `inner` | Matching rows from both sides |
| `left` / `left_outer` | All left rows + matching right rows |
| `right` / `right_outer` | All right rows + matching left rows |
| `full` / `full_outer` | All rows from both sides |
| `left_semi` | Left rows that have a match; no right columns |
| `left_anti` | Left rows that have no match |
| `cross` | Cartesian product; every left row with every right row |

## Syntax

```python
left.join(right, "customer_id", "inner")
```

or:

```python
left.join(
    right,
    left.customer_id == right.customer_id,
    "inner"
)
```

## The big gotcha: join cardinality

A join does **not** necessarily preserve the row count of the left DataFrame.

If:

```text
left:  customer_id = 1 appears 2 times
right: customer_id = 1 appears 3 times
```

an inner join can produce:

```text
2 × 3 = 6 rows
```

This is one of the most important practical join bugs: **duplicate keys can multiply rows.**

Always know the grain/cardinality of both sides before joining.

## Why joins can be expensive

If both sides are large and Spark cannot use a suitable broadcast strategy, the join may require a **shuffle** so matching keys are brought together.

Conceptually:

```text
Left partitions ──┐
                  ├── shuffle by join key ──→ join
Right partitions ─┘
```

So joins can be wide/expensive.

## Broadcast joins

If one side is small enough, Spark can often broadcast it to executors so the large side doesn't need the same kind of shuffle.

Example:

```python
from pyspark.sql.functions import broadcast

large_df.join(
    broadcast(small_df),
    "customer_id",
    "left"
)
```

The key idea:

```text
small table → copied/broadcast to executors
large table → processed locally against it
```

This can be much cheaper than shuffling two large datasets.

**Do not blindly broadcast huge DataFrames.** Broadcasting a table that is too large can create memory pressure or failures.

## Join checklist

Before joining, ask:

1. What is the grain of each DataFrame?
2. Is the join key unique on either side?
3. What join type do I actually need?
4. Can duplicate keys multiply rows?
5. Is one side small enough for broadcast?
6. Is the join key skewed?
7. Do I actually need all columns from both sides?
8. Can I filter/project before the join to reduce data?

## `left_semi` and `left_anti`

These are especially useful for existence logic.

```python
# Keep customers that have at least one order
customers.join(orders, "customer_id", "left_semi")
```

```python
# Keep customers with no matching order
customers.join(orders, "customer_id", "left_anti")
```

They return columns from the left side rather than duplicating right-side columns.

## When NOT to use a join

Don't join just because two DataFrames contain related-looking columns.

If the requirement is only an existence check, `left_semi` / `left_anti` may be more appropriate.

If the lookup is tiny, a broadcast join may be preferable.

If you need the latest record per key, don't blindly join a table containing many historical rows; first establish the desired grain, often with a window.

---

# 5. Window functions

## The core idea

A window function calculates something **across related rows while keeping the individual rows**.

This is the critical difference from `groupBy()`.

### `groupBy()` collapses rows

Input:

```text
customer | amount
---------|-------
A        | 100
A        | 200
B        | 300
```

```python
df.groupBy("customer").sum("amount")
```

Result:

```text
customer | total
---------|------
A        | 300
B        | 300
```

The individual transaction rows are gone.

### Window keeps rows

```python
from pyspark.sql.window import Window
from pyspark.sql.functions import sum, col

w = Window.partitionBy("customer")

df.withColumn("customer_total", sum("amount").over(w))
```

Result conceptually:

```text
customer | amount | customer_total
---------|--------|---------------
A        | 100    | 300
A        | 200    | 300
B        | 300    | 300
```

**Same rows, additional calculation.**

That's the mental model:

> `groupBy` → collapse rows into groups.
>
> Window → calculate across a group while retaining the rows.

---

# 6. `row_number()`

## What it does

`row_number()` assigns a unique sequential number to rows within each window partition according to the specified ordering.

Classic use case:

> **Keep the latest record for each business key.**

Example:

```text
customer_id | updated_at | status
------------|------------|-------
101         | Jan 1      | A
101         | Feb 1      | B
101         | Mar 1      | C
102         | Jan 5      | A
```

Define the window:

```python
from pyspark.sql.window import Window
from pyspark.sql.functions import row_number, col

w = Window.partitionBy("customer_id") \
          .orderBy(col("updated_at").desc())
```

Then:

```python
df.withColumn("rn", row_number().over(w))
```

Conceptually:

```text
customer_id | updated_at | status | rn
------------|------------|--------|---
101         | Mar 1      | C      | 1
101         | Feb 1      | B      | 2
101         | Jan 1      | A      | 3
102         | Jan 5      | A      | 1
```

Then:

```python
df.withColumn("rn", row_number().over(w)) \
  .filter(col("rn") == 1) \
  .drop("rn")
```

gives the latest row per customer.

## Why this is better than `dropDuplicates()` for this requirement

`dropDuplicates(["customer_id"])` can keep an arbitrary record when multiple records have the same customer ID.

If the requirement is:

> "Keep the most recent record"

then you need to express that ordering explicitly.

`row_number()` does that.

## The ugly part: ties

Suppose:

```text
customer_id | updated_at | status
------------|------------|-------
101         | Mar 1      | A
101         | Mar 1      | B
```

Both have the same ordering value.

If you only do:

```python
.orderBy(col("updated_at").desc())
```

you have **not fully defined a deterministic winner**.

Add a tie-breaker when business logic requires it:

```python
w = Window.partitionBy("customer_id") \
          .orderBy(
              col("updated_at").desc(),
              col("event_id").desc()
          )
```

Now Spark has a deterministic ordering assuming `event_id` itself is suitable as a unique tie-breaker.

## `row_number()` vs `rank()` / `dense_rank()`

These are different concepts.

| Function | Ties | Example order values `100, 100, 90` |
|---|---|---|
| `row_number()` | Gives every row a unique number | `1, 2, 3` |
| `rank()` | Same rank for ties, leaves gaps | `1, 1, 3` |
| `dense_rank()` | Same rank for ties, no gaps | `1, 1, 2` |

Use `row_number()` when you need **exactly one winning row per partition**.

Use `rank()` / `dense_rank()` when ties themselves are meaningful.

---

# 7. Window performance mental model

Window functions are powerful, but they aren't free.

For:

```python
Window.partitionBy("customer_id").orderBy("updated_at")
```

Spark may need to:

1. bring rows for the same `customer_id` together
2. order rows within those groups
3. calculate the window expression

That can involve substantial shuffle and sort work.

### Watch for huge partitions

If one customer has an enormous number of rows, that partition can become expensive.

This is another place where **data skew** matters.

### Filter early when logically safe

If you can reduce the input before the window without changing the required result, do it.

For example, don't carry irrelevant columns/rows into a window operation unnecessarily.

But be careful: filtering before a window can change the set of rows the window sees, so it must be logically valid for the requirement.

---

# 8. The most important comparisons

## `groupBy` vs window

| | `groupBy` | Window |
|---|---|---|
| Collapses rows? | Yes | No |
| Keeps original rows? | No | Yes |
| Can calculate per-group totals? | Yes | Yes |
| Can assign row numbers? | No | Yes |
| Typical use | Aggregated output | Per-row analytics using group context |

## `dropDuplicates` vs `row_number`

| Requirement | Better fit |
|---|---|
| Remove duplicate full rows | `dropDuplicates()` / `distinct()` |
| Keep one arbitrary row per key | `dropDuplicates(subset=[...])` if arbitrary is genuinely acceptable |
| Keep latest row per key | `row_number()` + ordered window |
| Keep highest-value row per key | `row_number()` + descending value |
| Keep all tied winners | `rank()` / `dense_rank()` depending on desired ranking semantics |

## `collect_list` vs window

| Requirement | Better fit |
|---|---|
| Build an array of all values per group | `collect_list()` |
| Build an array of unique values | `collect_set()` |
| Select one specific row per group | Window + `row_number()` |
| Keep rows while adding group-level information | Window |

---

# 9. Interview-level mental models

### `collect_list()` / `collect_set()`

> **"I am collapsing many rows into an array per group."**

### `pivot()`

> **"I am turning values in a column into output columns, usually with an aggregation."**

### `when().otherwise()`

> **"I am building conditional column logic inside Spark's expression API."**

### Join

> **"I am combining two datasets based on a relationship, and I need to understand both sides' grain/cardinality before I do it."**

### Window

> **"I want to calculate across related rows without collapsing those rows."**

### `row_number()`

> **"I want a deterministic sequence within each group based on an ordering, often to choose one specific row."**

---

# 10. Practical patterns to know cold

## Latest record per business key

```python
w = Window.partitionBy("business_key") \
          .orderBy(col("updated_at").desc(), col("event_id").desc())

latest = (
    df
    .withColumn("rn", row_number().over(w))
    .filter(col("rn") == 1)
    .drop("rn")
)
```

## Unique values per group

```python
df.groupBy("customer_id").agg(
    collect_set("category").alias("categories")
)
```

## All values per group

```python
df.groupBy("customer_id").agg(
    collect_list("product").alias("products")
)
```

## Conditional classification

```python
df.withColumn(
    "risk_band",
    when(col("score") >= 800, "Low")
    .when(col("score") >= 600, "Medium")
    .otherwise("High")
)
```

## Pivot for reporting

```python
df.groupBy("month") \
  .pivot("category", ["Phone", "Laptop", "Tablet"]) \
  .sum("sales")
```

## Existence check

```python
customers.join(orders, "customer_id", "left_semi")
```

## Find missing matches

```python
customers.join(orders, "customer_id", "left_anti")
```

---

# 11. What I should be able to answer after learning this

- Why can `collect_list()` / `collect_set()` become memory-heavy?
- Why is `collect_set()` different from `collect_list()`?
- Why should I not assume collection order?
- Why does pivot require an aggregation?
- Why can a high-cardinality pivot be dangerous?
- Why does `when()` condition order matter?
- What happens when no `when()` condition matches?
- How does NULL interact with conditional expressions?
- What happens to row count in a many-to-many join?
- How do duplicate keys cause row multiplication?
- When can a join involve a shuffle?
- Why can a broadcast join be cheaper?
- When should I use `left_semi` / `left_anti`?
- What is the difference between `groupBy` and a window?
- Why is `row_number()` useful for latest-record logic?
- Why isn't `dropDuplicates()` appropriate when I need the latest record?
- Why do I need a tie-breaker in a `row_number()` ordering?
- What makes a window operation expensive?

---

# 12. Quiz integration

These topics are now part of the **interactive PySpark learning track** alongside the earlier material.

The quiz should test these in multiple ways — not just "what is the syntax?":

1. Explain the concept in plain English.
2. Write the syntax from a requirement.
3. Predict the output.
4. Identify a bug.
5. Explain row-count changes.
6. Identify narrow/wide/shuffle implications.
7. Choose between `groupBy`, window, join, `collect_*`, or `pivot` based on the requirement.
8. Handle NULLs and ties.
9. Explain why a seemingly valid solution is dangerous at scale.
10. Design a real-world solution from a business requirement.

## Core rule

> **Don't memorize the function. Understand the shape of the data before and after the function.**

For every new PySpark operation, ask:

```text
What is the input grain?
        ↓
What does this operation do to rows?
        ↓
What does it do to columns?
        ↓
Does it require grouping/shuffle/sort?
        ↓
Can it increase/decrease row count?
        ↓
Can it create huge intermediate data?
        ↓
What happens with NULLs / duplicates / ties?
```
