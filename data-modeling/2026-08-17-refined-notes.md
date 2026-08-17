# Data Modeling — Refined Notes

**Date:** 2026-08-17
**Course:** Ansh Lamba — Database Design & Data Modeling Full Course For Beginners [2026]
**Scope:** Material covered up to 2:50:45

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

Example:

If `student_id` uniquely identifies a row:
- student_id = super key
- student_id + name = also a super key
- student_id + email = also a super key

But the larger combinations are not candidate keys because they are not minimal.

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

Bad:

customer_id | phone_numbers
---|---
1 | 9999, 8888

Better: store phone values as separate rows/related records.

### 2NF — Second Normal Form

2NF = 1NF + no partial dependency on a composite primary key.

Partial dependency occurs when a non-key attribute depends on only part of a composite key rather than the entire key.

This is primarily relevant when the primary key is composite.

### 3NF — Third Normal Form

3NF = 2NF + no transitive dependency.

A transitive dependency occurs when a non-key attribute depends on another non-key attribute instead of directly on the key.

Example:

`(order_id, product_id) → category_id`

`category_id → category_name`

Therefore category_name is transitively dependent on the original key.

### BCNF — Boyce-Codd Normal Form

BCNF is stricter than 3NF.

Core rule:

> Every determinant must be a super key.

BCNF is not automatically required for every practical design; 3NF is often sufficient depending on the requirements.

---

## 10. Key Mental Model

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
