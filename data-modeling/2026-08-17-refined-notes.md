# Data Modeling — Refined Notes

**Date:** 2026-08-17
**Course:** Ansh Lamba — Database Design & Data Modeling Full Course For Beginners [2026]
**Scope:** Complete video

## 1. Data Modeling

Data modeling is the process of translating business requirements into a structured representation of data, its attributes, relationships, constraints, and implementation.

It is useful to think about three levels:

### Conceptual Model

Business-level view of the data.

Focuses on:
- business requirements
- major business concepts/entities
- high-level relationships

No concern yet for SQL syntax, exact data types, indexes, etc.

### Logical Model

Turns business concepts into a detailed data model.

Focuses on:
- entities
- attributes
- relationships
- primary/candidate/foreign keys
- cardinality
- normalization
- constraints/business rules
- ER diagrams

### Physical Model

Implementation of the logical model in a specific database system.

Focuses on:
- tables and columns
- SQL data types
- primary/foreign/unique constraints
- indexes
- database-specific implementation details
- DDL / CREATE TABLE statements

---

## 2. OLTP vs OLAP

These describe different workload/system patterns. Do not confuse them with conceptual/logical/physical modeling layers.

### OLTP — Online Transaction Processing

Designed for operational/transactional workloads:
- frequent INSERT/UPDATE/DELETE
- many concurrent users/transactions
- low transaction latency
- strong data integrity/consistency
- usually small numbers of rows affected per transaction

Example: creating an order, updating an order status, recording a payment.

**Important:** OLTP does not simply mean “cheap storage.” The key distinction is the transactional workload.

### OLAP — Online Analytical Processing

Designed for analytical workloads:
- large reads
- historical analysis
- aggregations
- reporting
- dashboards
- queries that may scan large volumes of data

Example: calculating revenue by city and month over five years.

**Important:** OLAP does not simply mean “more detailed” or “more normalized.” Its key distinction is the analytical workload and the modeling/storage approach used to support it.

---

## 3. ER Diagrams

ER = Entity Relationship.

An ER diagram represents entities and the relationships between them.

### Entity

An entity is a real-world thing or business concept about which we need to store information.

Examples:
- Customer
- Order
- Product
- Payment
- Driver
- Trip

An entity is a conceptual/logical idea; in a relational implementation it is commonly represented by a table.

### Attribute

An attribute is a property/characteristic of an entity.

Example:

Customer
- customer_id
- name
- email
- phone

Customer = entity.
name/email/phone = attributes.

### Relationship

A relationship describes how entities are connected.

Example:

Customer → places → Orders

---

## 4. Data Types

Common SQL Server-oriented types:

### Integer

- TINYINT
- SMALLINT
- INT
- BIGINT

### Exact numeric

- DECIMAL(p,s)
- NUMERIC(p,s)

`p` = total precision/digits.
`s` = digits after the decimal point.

Use exact numeric types when exact decimal representation matters, such as monetary values.

### Approximate numeric

- FLOAT
- REAL

These are approximate representations, unlike DECIMAL/NUMERIC.

### Character/string

- CHAR — fixed length
- VARCHAR — variable length
- NCHAR — fixed-length Unicode
- NVARCHAR — variable-length Unicode

SQL Server does not use a generic `STRING` type like Python does.

### Date/time

- DATE
- TIME
- DATETIME
- DATETIME2
- DATETIMEOFFSET

### Other useful types

- BIT
- UNIQUEIDENTIFIER
- BINARY / VARBINARY

---

## 5. Keys

### Primary Key

A constraint/key that uniquely identifies each row.

Properties:
- unique
- non-null
- can consist of one or multiple columns

### Candidate Key

A minimal set of attributes that uniquely identifies a row and therefore could be selected as the primary key.

Example:

If both `user_id` and `email` uniquely identify users, both can be candidate keys.

### Alternate Key

A candidate key that was not selected as the primary key.

Example:

- user_id → primary key
- email → alternate key

### Super Key

Any set of one or more attributes that uniquely identifies a row.

A candidate key is a **minimal super key**.

### Foreign Key

A column or set of columns in a child table that references a key in a parent table, establishing a relationship between the tables.

Important: a foreign key does **not** have to be the child table's primary key.

Example:

CUSTOMERS
- customer_id PK

ORDERS
- order_id PK
- customer_id FK

`orders.customer_id` references `customers.customer_id`.

### Composite Key

A key made from multiple columns.

Example:

ORDER_ITEMS
- order_id
- product_id
- quantity

If neither `order_id` nor `product_id` is unique alone, `(order_id, product_id)` can form a composite primary key.

### UNIQUE Constraint / Key

Prevents duplicate values in a column or column combination. Exact NULL behavior should be treated as DBMS-specific rather than assumed to be universal SQL behavior.

---

## 6. Cardinality

Cardinality describes how many instances of one entity can/must be related to another entity.

Common relationships:
- 1:1
- 1:M / M:1
- M:M

More precise participation notation can include:
- 0..1
- 1..1
- 0..many
- 1..many

The distinction between zero and one matters because it tells us whether participation is optional or mandatory.

---

## 7. Many-to-Many Relationships

A relational database normally resolves an M:M relationship using a bridge/junction/associative table.

Example:

ORDERS M : M PRODUCTS

becomes:

ORDERS 1 : M ORDER_ITEMS M : 1 PRODUCTS

ORDER_ITEMS may contain:
- order_id FK
- product_id FK
- quantity

`(order_id, product_id)` can be the composite primary key.

---

## 8. Functional Dependency

Functional dependency is important for understanding normalization.

`A → B` means that A determines B.

Example:

`customer_id → customer_name`

If a customer ID identifies exactly one customer, knowing customer_id determines customer_name.

The determining attribute is called the **determinant**.

---

## 9. Normalization

Normalization organizes relational data to reduce redundancy and prevent update/insert/delete anomalies while preserving meaningful relationships.

Each normal form builds on the previous one.

### 1NF — First Normal Form

Core ideas:
- rows must be uniquely identifiable
- each column should contain atomic/single values
- avoid storing multiple independent values in one cell

### 2NF — Second Normal Form

2NF = 1NF + no partial dependency on a composite primary key.

Partial dependency occurs when a non-key attribute depends on only part of a composite key rather than the entire key.

This is primarily relevant when the primary key is composite.

### 3NF — Third Normal Form

3NF = 2NF + no transitive dependency.

A transitive dependency occurs when a non-key attribute depends on another non-key attribute instead of directly on the key.

### BCNF — Boyce-Codd Normal Form

BCNF is stricter than 3NF.

Core rule:

> Every determinant must be a super key.

BCNF is not automatically required for every practical design; 3NF is often sufficient depending on the requirements.

---

# 10. OLAP / Dimensional Modeling

After the OLTP/normalized modeling section, the course moves into **dimensional modeling**, which is primarily used to model data for OLAP/analytical workloads.

Dimensional modeling organizes analytical data around:

- **fact tables** — measurable business events/processes
- **dimension tables** — descriptive/contextual information

The goal is to make analytical queries easier and more efficient, often accepting some denormalization/redundancy in dimensions.

---

## 11. OBT — One Big Table

OBT = **One Big Table**.

The basic idea is to combine data from multiple source/normalized tables into a broad table containing the columns needed for a particular analytical purpose.

Conceptually:

```text
Normalized / OLTP data
        ↓
      OBT
        ↓
  analytical modeling
        ↓
Facts + Dimensions
```

OBT is not the same thing as the final dimensional model. It can be an intermediate/working representation used before deriving facts and dimensions.

---

## 12. Fact Tables

A fact table represents a measurable business process/event at a defined **grain**.

Examples:
- sales
- orders
- payments
- trips
- cancellations

Fact tables generally contain:
- foreign keys to dimensions
- numeric measures
- sometimes other event-level attributes

Example:

```text
fact_sales
-----------
order_date_sk
customer_sk
product_sk
quantity
sales_amount
```

### Important

A fact table is **not simply a table containing financial information**.

The important idea is that it represents a business process/event and contains values that can be analyzed, often through aggregation.

---

## 13. Dimension Tables

Dimension tables provide **context** for facts.

Examples:
- customer
- product
- date
- location
- driver
- vehicle

They typically contain descriptive attributes used to filter, group, and explain facts.

Example:

```text
dim_customer
------------
customer_sk
customer_id
customer_name
city
state
segment
```

Dimensions are commonly denormalized in dimensional models to reduce the number of joins required by analytical queries.

---

## 14. Grain

**Grain = what exactly one row in a fact table represents.**

This is one of the most important concepts in dimensional modeling.

Examples:

> One row = one order.

or:

> One row = one order line item.

or:

> One row = one trip.

These are different grains and produce different fact tables.

A useful question is:

> "What does one row represent?"

Do not define grain as "how many records per user/policy." It is the level of detail represented by **one row**.

---

## 15. Measures

Measures are quantitative values stored in or derived from fact data that can be analyzed/aggregated.

Examples:
- quantity
- sales_amount
- revenue
- discount_amount
- distance
- trip_duration

Typical operations include:
- SUM
- AVG
- MIN
- MAX
- COUNT

Not every numeric column is automatically a measure; its meaning and behavior at the fact's grain matter.

---

## 16. Star Schema

A star schema has:

- a central fact table
- dimension tables surrounding it

Example:

```text
              dim_customer
                   │
                   │
dim_date ─── fact_sales ─── dim_product
                   │
                   │
             dim_location
```

It is called a **star schema** because the visual structure resembles a star.

Star schemas are commonly preferred for analytical workloads because they provide a simple query structure with relatively few joins.

---

## 17. Snowflake Schema

A snowflake schema is a dimensional model in which one or more dimensions are further normalized into related sub-dimensions.

Example:

```text
fact_sales
    │
    ▼
dim_product
    │
    ▼
dim_category
```

Compared with a star schema, this introduces additional joins but can reduce some dimension redundancy.

Mental model:

```text
STAR
Fact ←→ Dimensions

SNOWFLAKE
Fact ←→ Dimensions ←→ Sub-dimensions
```

---

## 18. Date Dimension

A date dimension provides reusable analytical time context.

Example attributes:

```text
date_sk
date
 day
month
quarter
year
```

A fact can reference the date dimension using a date surrogate key, allowing consistent time-based analysis across the warehouse.

---

## 19. Surrogate Keys

A surrogate key is a system-generated key used to identify a dimension record, typically independent of the business/natural key.

Example:

```text
dim_customer
------------
customer_sk   ← surrogate key
customer_id   ← business/natural key
customer_name
```

A surrogate key is **not simply an encrypted or more readable version of a long key**.

It is usually an artificial/system-generated identifier such as `101`, `102`, `103`, etc.

One major reason surrogate keys become important is historical tracking, especially with SCD Type 2.

---

# 20. Slowly Changing Dimensions (SCD)

Dimensions can change over time.

Example:

```text
Customer address:
Bangalore → Hyderabad
```

SCD techniques define how those changes should be represented while balancing current-state needs and historical requirements.

---

## SCD Type 0

**Do not change the original value.**

Example:

```text
date_of_birth
```

The original value is retained permanently.

---

## SCD Type 1

**Overwrite the old value.**

Example:

Before:

```text
customer_id = 1
address = Bangalore
```

After:

```text
customer_id = 1
address = Hyderabad
```

The old Bangalore value is lost.

This is commonly implemented using an upsert/overwrite approach.

Use Type 1 when historical changes do not need to be preserved.

---

## SCD Type 2

**Preserve full history by creating a new dimension version/row.**

Typical columns include:

```text
customer_sk
customer_id
address
valid_from
valid_to
is_current
```

Example:

```text
customer_sk | customer_id | address   | valid_from | valid_to   | is_current
------------|-------------|-----------|------------|------------|-----------
101         | 1           | Bangalore | 2026-01-01 | 2027-02-01 | 0
205         | 1           | Hyderabad | 2027-02-01 | NULL       | 1
```

This allows historical facts to be interpreted using the dimension state that was valid at the relevant time.

**SCD Type 2 is one of the most important SCD patterns for Data Engineering.**

---

## SCD Type 3

Store limited previous/current history in additional columns.

Example:

```text
customer_id
current_address
previous_address
```

If the customer changes address repeatedly, this design generally retains only the current value and a limited previous value rather than unlimited historical versions.

---

## SCD Types 4–6

The course also introduces additional SCD patterns/hybrid approaches.

For practical interview preparation, prioritize:

1. Type 0
2. Type 1
3. Type 2
3. Type 3

You should recognize Types 4–6, but do not spend disproportionate study time memorizing them before Type 2 is completely solid.

---

# 21. SCD Implementation Concepts

SCD implementations can be handled using approaches such as:

- SQL `MERGE`
- Spark SQL
- PySpark
- warehouse/platform-specific features

This becomes especially relevant later when working with:

**PySpark + Databricks + Delta Lake**.

---

# 22. OLTP → OBT → Dimensional Model Mental Model

A useful end-to-end mental model from the course is:

```text
Operational / OLTP data
        ↓
      OBT
        ↓
Identify business processes
        ↓
Determine grain
        ↓
Identify measures
        ↓
Identify descriptive context
        ↓
Dimensions + Facts
        ↓
Star / Snowflake schema
        ↓
OLAP / analytical workloads
```

This is not the only possible architecture, but it is a useful way to connect the concepts taught in the video.

---

# 23. What This Video Covered vs What It Did Not

This video gives a strong introduction to **database design and dimensional modeling**, including:

- conceptual/logical/physical modeling
- ER modeling
- keys
- cardinality
- normalization
- OLTP vs OLAP
- OBT
- dimensional modeling
- facts
- dimensions
- grain
- measures
- star schema
- snowflake schema
- surrogate keys
- SCD

It does **not** mean Data Warehousing is now completely mastered.

Topics such as deeper ETL/ELT architecture, incremental processing, CDC, Bronze/Silver/Gold implementation, partitioning, orchestration, data quality, performance engineering, and production lakehouse design need separate study.

---

# 24. Key Mental Model

```text
DATA MODELING
│
├── Conceptual
│   └── business concepts + requirements
│
├── Logical
│   ├── entities
│   ├── attributes
│   ├── relationships
│   ├── keys
│   ├── cardinality
│   ├── normalization
│   └── ER diagrams
│
└── Physical
    ├── SQL tables
    ├── data types
    ├── constraints
    ├── indexes
    └── DBMS-specific implementation
```

Separately:

```text
              DATA WORKLOAD
                    │
          ┌─────────┴─────────┐
          │                   │
         OLTP                OLAP
          │                   │
    transactional          analytical
      workloads             workloads
          │                   │
   frequent writes       large reads/
   low latency            aggregations
   concurrency            historical data
```

**Do not merge these two classifications.**

Conceptual/logical/physical describe levels of modeling and implementation. OLTP/OLAP describe workload/system patterns and the modeling approaches used to support them.
