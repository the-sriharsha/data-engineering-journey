# Complete Summary of Data Modeling & What's Left To Learn

**Date:** 2026-08-17
**Course:** Ansh Lamba — Database Design & Data Modeling Full Course For Beginners [2026]
**Status:** Video completed

---

# 1. Overall Understanding

Data modeling is the process of understanding and representing how data is structured, how different pieces of data relate to each other, what rules/constraints apply, and how that model will ultimately be implemented.

The biggest mental model from this course is that data modeling has multiple levels and that OLTP and OLAP solve different workload problems.

```text
DATA MODELING
│
├── Conceptual
│   └── Business-level understanding
│
├── Logical
│   └── Entities + attributes + relationships + keys + normalization
│
└── Physical
    └── Actual database implementation / SQL
```

Separately:

```text
DATA WORKLOAD
│
├── OLTP
│   └── Transactional / operational workloads
│
└── OLAP
    └── Analytical workloads
```

These are related but are not the same classification.

---

# 2. What I Learned

## Conceptual, Logical and Physical Modeling

### Conceptual

High-level business understanding:

- stakeholder requirements
- major business concepts/entities
- high-level relationships
- no concern yet for SQL syntax or exact implementation

### Logical

Detailed representation of the business data model:

- entities
- attributes
- relationships
- keys
- cardinality
- normalization
- constraints/business rules
- ER diagrams

### Physical

Actual implementation in a database:

- tables
- columns
- data types
- primary/foreign/unique constraints
- indexes
- DDL / CREATE TABLE
- database-specific implementation details

---

# 3. Entities, Attributes and Relationships

## Entity

A real-world thing or business concept about which we need to store information.

Examples:

- Customer
- Order
- Product
- Driver
- Vehicle
- Trip

An entity is not exactly the same thing as a table, although an entity is commonly represented by a table in a relational implementation.

## Attribute

A property/characteristic of an entity.

Example:

```text
Customer
├── customer_id
├── name
├── email
└── phone
```

Customer = entity.

name/email/phone = attributes.

## Relationship

Describes how entities are connected.

Example:

```text
Customer → places → Orders
```

---

# 4. ER Diagrams

ER = Entity Relationship.

ER diagrams visually represent entities and the relationships between them.

Crow's-foot notation can be used to represent relationship cardinality.

---

# 5. OLTP vs OLAP

## OLTP — Online Transaction Processing

Designed for operational/transactional workloads:

- frequent INSERT/UPDATE/DELETE
- many concurrent users/transactions
- low transaction latency
- strong data integrity/consistency
- usually small numbers of rows affected per transaction

Example:

```text
Create order
→ Update order status
→ Record payment
```

Important correction to my initial understanding:

**OLTP does not mean cheap storage.** The important distinction is the transactional workload.

## OLAP — Online Analytical Processing

Designed for analytical workloads:

- large reads
- historical analysis
- aggregations
- reporting
- dashboards
- queries that may scan large volumes of data

Example:

```text
Calculate revenue by city and month
for the last five years.
```

Important correction:

**OLAP does not simply mean more detailed or more normalized.** Its purpose is analytical workloads and the modeling/storage approach used to support them.

---

# 6. Keys

## Primary Key

Uniquely identifies each row.

- unique
- non-null
- can be single-column or composite

## Candidate Key

A minimal set of attributes that uniquely identifies a row and could therefore be selected as the primary key.

## Alternate Key

A candidate key that was not selected as the primary key.

## Super Key

Any set of one or more attributes that uniquely identifies a row.

A candidate key is a minimal super key.

## Foreign Key

A column or set of columns in a child table that references a key in a parent table.

Important:

**A foreign key does not have to be the child's primary key.**

Example:

```text
CUSTOMERS
customer_id PK

ORDERS
order_id PK
customer_id FK
```

## Composite Key

A key made from multiple columns.

Example:

```text
ORDER_ITEMS
order_id
product_id
quantity
```

`(order_id, product_id)` can uniquely identify an order item.

## UNIQUE Constraint

Prevents duplicate values in a column or column combination. Exact NULL behavior is DBMS-specific.

---

# 7. Cardinality

Cardinality describes how many instances of one entity can/must be related to another.

Common relationships:

- 1:1
- 1:M / M:1
- M:M

More precise participation can be represented as:

- 0..1
- 1..1
- 0..many
- 1..many

The zero vs one distinction tells us whether participation is optional or mandatory.

---

# 8. Many-to-Many Relationships

Relational databases normally resolve M:M relationships using a bridge/junction/associative table.

Example:

```text
ORDERS M : M PRODUCTS
```

becomes:

```text
ORDERS 1 : M ORDER_ITEMS M : 1 PRODUCTS
```

`ORDER_ITEMS` may contain:

- order_id FK
- product_id FK
- quantity

`(order_id, product_id)` can form the composite primary key.

---

# 9. Normalization

Normalization organizes relational data to reduce redundancy and prevent update/insert/delete anomalies while preserving meaningful relationships.

## 1NF

- rows must be uniquely identifiable
- values should be atomic/single values
- avoid storing multiple independent values in one cell

## 2NF

2NF = 1NF + no partial dependency on a composite primary key.

A non-key attribute should not depend on only part of a composite key.

## 3NF

3NF = 2NF + no transitive dependency.

A non-key attribute should not depend on another non-key attribute instead of directly on the key.

## BCNF

BCNF is stricter than 3NF.

Core rule:

> Every determinant must be a super key.

BCNF is not automatically required for every practical design; 3NF is often sufficient depending on requirements.

## Functional Dependency

`A → B` means A determines B.

Example:

```text
customer_id → customer_name
```

Functional dependency is important for understanding 2NF, 3NF and BCNF.

---

# 10. Data Types

Important SQL Server-oriented categories:

### Integer

- TINYINT
- SMALLINT
- INT
- BIGINT

### Exact numeric

- DECIMAL(p,s)
- NUMERIC(p,s)

### Approximate numeric

- FLOAT
- REAL

### Character

- CHAR
- VARCHAR
- NCHAR
- NVARCHAR

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

Important distinction:

**DECIMAL/NUMERIC are exact numeric types; FLOAT/REAL are approximate.**

---

# 11. OBT — One Big Table

OBT = One Big Table.

The idea is to combine the required data from multiple tables into one broad table so the required information is available in one place.

Conceptually:

```text
OLTP tables
    ↓
JOIN / transform
    ↓
One Big Table
```

OBT can then be used as an intermediate step when building a dimensional/analytical model.

```text
OLTP
 ↓
OBT
 ↓
Identify dimensions + measures
 ↓
Fact + Dimension tables
 ↓
Star schema
```

---

# 12. Dimensional Modeling

Dimensional modeling is a modeling approach for analytical/OLAP systems.

It organizes data around:

- business processes/facts
- descriptive dimensions
- measurable values
- analytical query patterns

Unlike highly normalized OLTP models, dimensional models generally accept some redundancy/denormalization to make analytical querying simpler and more efficient.

---

# 13. Fact Tables

A fact table represents a business process/event at a defined grain.

It is not limited to financial data.

Examples:

- sales
- orders
- trips
- payments
- cancellations

Fact tables commonly contain:

- foreign keys to dimensions
- measures
- identifiers related to the business event

The most important question is:

> **What does one row in this fact table represent?**

That answer defines the grain.

---

# 14. Dimension Tables

Dimension tables provide descriptive/contextual information used to analyze facts.

Examples:

- customer
- product
- date
- driver
- vehicle
- location

Dimensions commonly contain descriptive attributes and are generally denormalized in a dimensional model.

---

# 15. Grain

Grain means:

> **What does exactly one row represent?**

Examples:

```text
One row = one order
```

or:

```text
One row = one order line
```

or:

```text
One row = one trip
```

or:

```text
One row = one driver location observation
```

Grain is not simply "how many rows an entity has."

The grain must be clearly defined before deciding what belongs in a fact table.

---

# 16. Measures

Measures are numerical values in fact tables that can be analyzed/aggregated depending on their properties.

Examples:

- quantity
- revenue
- amount
- distance
- duration
- number of items

Important future topic:

- additive measures
- semi-additive measures
- non-additive measures

---

# 17. Star Schema

A star schema has a central fact table connected directly to dimension tables.

```text
              DIM_CUSTOMER
                   │
                   │
DIM_DATE ───── FACT_SALES ───── DIM_PRODUCT
                   │
                   │
              DIM_SHIPMENT
```

The fact table is in the center and dimensions surround it like a star.

Star schemas are simple and convenient for analytical querying.

---

# 18. Snowflake Schema

A snowflake schema is similar to a star schema, but dimensions can be further normalized into additional related tables.

Example:

```text
FACT_SALES
    │
    └── DIM_PRODUCT
             │
             └── DIM_CATEGORY
```

Compared with star schema:

- Star → dimensions are generally denormalized.
- Snowflake → some dimension data is further normalized.

---

# 19. Surrogate Keys

A surrogate key is a system-generated/artificial key used to identify a record, especially in dimensional models.

Example:

```text
customer_sk = 101
customer_id = original business/customer identifier
```

Important correction to my initial understanding:

**A surrogate key is not an encrypted or "readable version" of a long key.**

It is an artificial identifier, often an integer, that is independent of the business/natural key.

Surrogate keys become particularly important when managing historical dimension records such as SCD Type 2.

---

# 20. Date Dimension

A reusable date dimension can provide analytical time context.

Example attributes:

- date
- day
- month
- quarter
- year
- other calendar/fiscal attributes

Instead of repeatedly deriving these properties from every fact table's date column, analytical models can use a shared date dimension.

---

# 21. Slowly Changing Dimensions

SCD = Slowly Changing Dimensions.

Dimension attributes can change over time.

Example:

```text
Customer changes address
```

The key question is:

> Do we overwrite the old value or preserve history?

## Type 0

Never change the original value.

Example:

```text
date_of_birth
```

## Type 1

Overwrite the old value.

No historical version is retained.

Example:

```text
Before: address = A
After:  address = B
```

The row simply contains B.

Often implemented as an upsert/overwrite pattern.

## Type 2

Preserve full historical versions.

Typical columns include:

- surrogate key
- business/natural key
- valid_from/start_date
- valid_to/end_date
- current_flag

Example:

```text
customer_sk | customer_id | address | valid_from | valid_to | current
------------|-------------|---------|------------|----------|--------
101         | 1           | A       | 2026-01-01 | 2027-01-01 | false
205         | 1           | B       | 2027-01-01 | NULL       | true
```

## Type 3

Keep limited history using additional columns.

Example:

```text
current_address
previous_address
```

If the customer changes address repeatedly, this model does not preserve unlimited historical versions; it keeps limited previous/current information.

## Type 4

Current dimension and historical data are separated into different tables.

## Type 5

A hybrid approach combining dimensional modeling/SCD techniques to retain current and historical information.

## Type 6

A hybrid SCD approach combining multiple SCD strategies.

### Practical priority

Know Type 0, Type 1, Type 2 and Type 3 well.

Type 2 is especially important for Data Engineering because preserving dimension history is a common warehouse/lakehouse requirement.

---

# 22. SCD Implementation

SCDs can be implemented using approaches such as:

- SQL
- MERGE statements
- PySpark / Spark SQL
- platform-specific tools/patterns

This becomes particularly relevant later when working with:

- Databricks
- Delta Lake
- PySpark

---

# 23. What This Video Taught Me

After completing the video, I understand the overall progression as:

```text
Business Requirements
        ↓
Conceptual Model
        ↓
Logical Model
        ↓
ER Diagram
        ↓
Keys + Relationships + Cardinality
        ↓
Normalization
        ↓
Physical SQL Implementation
        ↓
OLTP / Operational Data
        ↓
OBT
        ↓
Dimensional Modeling
        ↓
Facts + Dimensions
        ↓
Grain + Measures
        ↓
Star / Snowflake Schema
        ↓
Surrogate Keys
        ↓
SCDs / Historical Dimensions
```

The biggest conceptual distinction I should retain is:

```text
Conceptual / Logical / Physical
        = levels of modeling/implementation

OLTP / OLAP
        = workload/system patterns
```

---

# 24. What I Still Need To Learn

This video gave me a foundation, but it did **not** complete Data Modeling or Data Warehousing.

## A. Normalization — needs deeper practice

I need to become comfortable with:

- functional dependencies
- candidate keys
- partial dependency
- transitive dependency
- 1NF
- 2NF
- 3NF
- BCNF
- identifying anomalies
- deciding when normalization is appropriate
- practical normalization exercises

## B. Dimensional Modeling — needs deeper practice

I need to practice:

- identifying the business process
- identifying the grain
- identifying facts
- identifying dimensions
- choosing measures
- additive vs semi-additive vs non-additive measures
- conformed dimensions
- role-playing dimensions
- degenerate dimensions
- junk dimensions
- factless fact tables
- accumulating snapshot facts
- periodic snapshot facts
- transaction facts
- multiple fact tables
- fact-to-fact problems

## C. Slowly Changing Dimensions — needs implementation practice

Especially:

- SCD Type 1 implementation
- SCD Type 2 implementation
- MERGE patterns
- surrogate key generation
- effective dates
- current flags
- handling late-arriving changes
- handling duplicate updates
- implementing SCD2 in PySpark
- implementing SCD2 using Delta Lake

## D. Data Warehousing — separate topic to learn

This video introduced dimensional modeling but did not cover the whole data warehouse subject.

Still to learn deeply:

- OLTP vs data warehouse architecture
- ETL vs ELT
- staging layers
- Bronze / Silver / Gold
- incremental loads
- full loads
- CDC
- idempotency
- late-arriving data
- schema evolution
- data quality
- partitioning
- warehouse/lakehouse architecture
- fact and dimension loading strategies

## E. Physical Database Design

Still need deeper understanding of:

- indexes
- clustered vs non-clustered indexes
- query performance
- execution plans
- partitioning
- constraints
- storage considerations
- database-specific optimization

## F. Real-world Modeling Practice

Need to model real systems independently rather than only following examples.

Practice domains:

- Uber / ride sharing
- Insurance
- E-commerce
- Banking/payments
- Customer analytics

For every model, explicitly define:

1. Business requirements
2. Entities
3. Attributes
4. Relationships
5. Cardinality
6. Primary/candidate/foreign keys
7. Grain
8. Normalization decisions
9. OLTP model
10. OLAP model
11. Facts
12. Dimensions
13. SCD strategy
14. Physical implementation

---

# 25. Priority For My Data Engineering Goal

## Must Know Deeply

1. Keys
2. Relationships
3. Cardinality
4. Normalization
5. Grain
6. Fact vs dimension
7. Dimensional modeling
8. Star schema
9. SCD Type 1 and Type 2
10. Surrogate keys
11. OLTP vs OLAP
12. Data warehouse fundamentals

## Need Working Knowledge

- Snowflake schema
- OBT
- SCD Type 0/3
- Date dimensions
- Composite keys
- Functional dependencies
- Physical database design

## Later / Lower Priority

- SCD Type 4/5/6 in depth
- Advanced warehouse modeling patterns
- Rare normalization edge cases

---

# 26. Current Status

**Data Modeling:** Foundation established; needs hands-on modeling practice.

**Normalization:** Understood conceptually; needs dedicated practice.

**Dimensional Modeling:** Foundation established; needs deeper fact/dimension/grain practice.

**SCD:** Basic conceptual understanding established; Type 2 needs implementation practice.

**Data Warehousing:** Not complete. Dimensional modeling is one component, not the whole subject.

**Next logical step:** Practice data modeling with real business scenarios, then connect the models to SQL and eventually to the Data Engineering stack: PySpark + Databricks + Delta Lake + ADF.
