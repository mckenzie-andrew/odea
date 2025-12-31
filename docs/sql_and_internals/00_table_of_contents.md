# Welcome to SQL and Database Internals
**The Mission**: To transition the student from "Writing SQL that works" to "Writing SQL that scales." We are not just teaching syntax; we are teaching the engineering consequences of every query they write.

### The Approach:
- **Comparative Learning**: Every concept is tested on PostgreSQL (Row Store / OLTP) and DuckDB (Column Store / OLAP) simultaneously to highlight architectural trade-offs.
- **Internals First**: We explain how the engine retrieves data (Scanning, Parsing, Planning) so the syntax makes sense logically, not just memorized.
- **Muscle Memory**: Heavy emphasis on "Drills"—repetitive, real-world scenarios using messy data to build instinct.

## Learning Goals by Phase
### Phase 1: Syntax & The Relational Model
- **Goal**: Master the Declarative nature of SQL.
- **Outcome**: Students stop thinking in "Loops" (imperative) and start thinking in "Sets" (declarative).
- **Key Skill**: Writing strict, performant queries using `WHERE`, `GROUP BY`, and `HAVING` without triggering common "gotchas" like `NULL` traps or Cartesian products.

### Phase 2: Relational Logic & Modeling
- **Goal**: Understand how to stitch isolated data points into a cohesive narrative.
- **Outcome**: Students can comfortably navigate normalized (3NF) schemas and understand the cost of joining data.
- **Key Skill**: Visualizing `JOINS` as filtered Cartesian products and understanding the difference between `UNION` (Set) and `JOIN` (Relational) operations.

### Phase 3: The Engine Room (Internals)
- **Goal**: Demystify the "Black Box" of the database.
- **Outcome**: Students understand why their query is slow. They can explain the physical difference between how a Row Store reads 8kb pages vs. how a Column Store reads vectors.
- **Key Skill**: Understanding ACID compliance, the Write-Ahead Log (WAL), and MVCC (why `DELETE` doesn't actually delete data immediately).

### Phase 4: Performance & Optimization
- **Goal**: Learn the art of "Database Tuning."
- **Outcome**: Students can read an `EXPLAIN ANALYZE` plan, identify "Sequential Scans," and fix them using Indexes or query rewrites.
- **Key Skill**: Implementing B-Tree, GIN, and BRIN indexes effectively, and knowing when not to index.

### Phase 5: Modern Data Engineering
- **Goal**: Bridge the gap between the Database and the Data Lake.
- **Outcome**: Students are ready for the modern stack. They can move data between Postgres and Parquet/Arrow without friction.
- **Key Skill**: Building a hybrid workflow that leverages Postgres for transactions and DuckDB for heavy analytical lifting on the same dataset.

## Table of Contents
*Part 1: The Syntax Bootcamp (Zero to Competent)*

- **Chapter 1: Hello, Data (Your First Query)**
    - 1.1 The "Select Star" Pattern (Anatomy of a Query)
    - 1.2 Selecting Specific Columns (Aliasing & Basic Math)
- **Chapter 2: Filtering (Finding the Needle in the Haystack)**
    - 2.1 The WHERE Clause (Text & Numbers)
    - 2.2 Logical Operators (AND, OR, NOT)
    - 2.3 Pattern Matching (LIKE, Wildcards, IN)
    - 2.4 The "Nothing" Problem (Handling NULL Correctly)
- **Chapter 3: Organizing Your Results**
    - 3.1 Sorting (ORDER BY, ASC/DESC, NULLS LAST)
    - 3.2 Limiting Results (LIMIT & OFFSET)
    - 3.3 Removing Duplicates (DISTINCT)
- **Chapter 4: The Math of SQL (Aggregates)**
    - 4.1 The Aggregate Functions (COUNT, SUM, AVG)
    - 4.2 The GROUP BY Clause (Bucketing Data)
    - 4.3 Filtering Aggregates (HAVING vs WHERE)
- **Chapter 5: Putting It Together (Order of Execution)**
    - 5.1 The Logical Order (Lexical vs. Logical Execution Flow)

*Part 2: Relational Logic (Connecting Data)*

- **Chapter 6: Joins (The Heart of SQL)**
    - 6.1 The Primary & Foreign Key Relationship
    - 6.2 INNER JOIN (Finding Overlaps)
    - 6.3 LEFT JOIN (Preserving Records)
    - 6.4 RIGHT and FULL OUTER Joins (Edge Cases)
- **Chapter 7: Set Operations (Vertical Combinations)**
    - 7.1 UNION vs UNION ALL (Stacking Data)
    - 7.2 INTERSECT and EXCEPT (Set Differences)

*Part 3: The Engine Room (Internals & Architecture)*

- **Chapter 8: Storage Engines (Row vs. Column)**
    - 8.1 The Row Store (Postgres: Heaps, Tuples, Pages)
    - 8.2 The Column Store (DuckDB: Compression, Vectorization)
- **Chapter 9: Transaction Management (ACID)**
    - 9.1 The Write-Ahead Log (WAL & Durability)
    - 9.2 MVCC (Multi-Version Concurrency Control & Dead Tuples)
    - 9.3 Isolation Levels (Read Committed vs. Serializable)

*Part 4: Performance Tuning & Optimization*

- **Chapter 10: The Query Planner & Optimizer**
    - 10.1 Reading the Plan (EXPLAIN ANALYZE & Cost Models)
    - 10.2 Scan Types (Seq Scan vs. Index Scan)
    - 10.3 Join Algorithms (Nested Loop, Hash Join, Merge Join)
- **Chapter 11: Indexing Strategies**
    - 11.1 The B-Tree (Logarithmic Search)
    - 11.2 Specialized Indexes (GIN, GiST, BRIN)
    - 11.3 Zone Maps (DuckDB's Automatic Indexing)
- **Chapter 12: Maintenance & Hygiene**
    - 12.1 VACUUM (Garbage Collection in Postgres)
    - 12.2 Statistics (ANALYZE & The Planner's Estimates)

*Part 5: Expert Engineering & Modern Workflows*

- **Chapter 13: Interoperability & The "Data Lake"**
    - 13.1 The Parquet Format (Row Groups & Column Chunks)
    - 13.2 Apache Arrow (In-Memory Zero-Copy Transfer)
    - 13.3 Foreign Data Wrappers (Querying External Systems)
- **Chapter 14: Final Capstone Project**
    - 14.1 Scenario: The Taxi Data Challenge
    - 14.2 Task 1: Ingest & Optimize (Postgres)
    - 14.3 Task 2: Transform & Benchmark (DuckDB)
    - 14.4 Task 3: Performance Analysis Report