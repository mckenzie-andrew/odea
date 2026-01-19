We have spent the entire course conditioning your brain to look for **measures**.

I told you that dimensions are "Nouns" (text) and Facts are "verbs" (numbers). I told you that a fact table exists to sum things up: `SUM(sales_amount)`, `AVG(duration_seconds)`, `MAX(temperature)`.

But what happens when the business process is an event that has no magnitude?

## 13.1 Tracking Events without Numbers

What if the CEO asks, "Who attended the 'Advanced Latte Art' training session yesterday?"

You look at the event.

- **Who**: Bob (the barista).
- **What**: Advanced Latte Art (the course).
- **When**: Monday.
- **How Much**: … ?

There is no "dollar amount." There is no "quantity sold." Bob didn't "produce" 5 units of attendance. He just… showed up.

This is the domain of the **factless fact table**. It is a table that contains nothing but foreign keys. To the untrained eye, it looks like a mistake—a bridge to nowhere. But to a data engineer, it is the only way to track **existence**.

### The "Attendance" Schema
Let's model the training program at **Omni-Coffee**. We want to track which baristas have completed which certifications.

If we tried to force this into a standard transaction fact table, we'd be searching for a number that doesn't exist. Instead, we build a table that looks like this:

**Table: `fact_training_attendance`**

| date_key (FK) | employee_key (FK) | course_key (FK) | location_key (FK) |
|:---|:---|:---|:---|
| 20240501 | 101 (Bob) | 50 (Latte Art) | 01 (Seattle HQ) |
| 20240501 | 102 (Alice) | 50 (Latte Art) | 01 (Seattle HQ) |
| 20240502 | 101 (Bob) | 60 (Safety) | 01 (Seattle HQ) |

**What is missing?** There are no columns on the right side. No `amount`. No `score`.

### How to Query "Nothing"
You might ask, "If there are no numbers, how do we do math?"

The math here is simple: **the row itself is the metric.**

If we want to know how many people took the "Latte Art" course, we don't sum a column. We count the rows.

```sql
SELECT
    c.course_name,
    COUNT(*) AS total_attendees
FROM fact_training_attendance AS f
JOIN dim_course AS c ON f.course_key = c.course_key
GROUP BY c.course_name;
```

### The "Dummy" Metric Pattern
Some BI tools (Tableau, Power BI, Looker) get confused by tables that don't have a numeric column. They explicitly look for a "Measure" to drag into the "Values" box.

To appease these tools (and to make life easier for analysts), many data engineers add a dummy column, often called `counter` or `quantity`, and hard-code it to `1`.

**Table: `fact_training_attendance` (with dummy)**

| date_key | employee_key | course_key | counter |
|:---|:---|:---|:---|
| 20240501 | 101 | 50 | 1 |
| 20240501 | 102 | 50 | 1 |

Now, instead of `COUNT(*)`, the analyst can `SUM(counter)`. It is mathematically identical but psychologically more comfortable for users who expect a fact table to "add up" to something.

### Why Not Just Use a Dimension?
A common beginner mistake is to try to shove this data into a dimension, "Why not just add a `training_courses_taken` column to the employee dimension?"

Because: **cardinality**.

- Bob takes 1 course? Fine.
- Bob takes 50 courses over 10 years?
- Bob takes the *same* course twice (recertification)?

If you store this as a list inside the employee table, you violate First Normal Form (no lists in columns). If you try to make a "Many-to-Many" link, you are essentially building a factless fact table anyway—you just haven't named it correctly.

Treat "Attendance," "Clicks," "Logins," and "Assignments" as distinct events. Give them their own fact table. It costs almost nothing in storage (since it's just skinny integers), but it gives you a clean, joinable history of every event.

But what about the events that *didn't* happen? That is a much harder problem.

## 13.2 Coverage Tables
If the factless fact table tracks "Events without numbers," the **coverage table** tracks "Non-events."

This is one of the hardest questions to answer in SQL: "Which products sold zero units today?"

It sounds simple. But think about the physical nature of your `fact_sales` table. It is a log of transactions.

- If a customer buys a croissant, we write a row.
- If a customer buys a latte, we write a row.
- If a customer *ignores* the bagel, we write… nothing. The database remains silent.

If you query `SUM(sales)` for the bagel, you don't get 0. You get `NULL` (or the row simply disappears from the result set). You cannot count what isn't there.

This is the **negative space problem**. To analyze what *didn't* happen, we need to first define the "Universe of What Could Have Happened."

### The "Denominator" Problem
Let's look at the concrete Omni-Coffee crisis.

The marketing director calls you, "The spicy Pumpkin Latte is failing in Miami. Sales are zero. Why do they hate it?"

You check `fact_sales` Sure enough, Miami sales for that product are nonexistent. But there are two possible reasons for this:

1. **Rejection**: The product was available, but customers chose not to buy it. (Demand issue).
2. **Absence**: The store manager forgot to order the pumpkin syrup. The product was never on the menu. (Supply issue).

The `fact_sales` table cannot tell the difference between "hated" and "missing." Both look like empty space.

To differentiate, we need a **coverage table** (often called an inventory snapshot or product availability table). This table records the *opportunity* for a sale.

**Table: `fact_daily_product_availability`**

| date_key | store_key | product_key | is_on_menu | is_in_stock |
|:---|:---|:---|:---|:---|
| 20241031 | 05 (Miami) | 99 (Pumpkin Latte) | True | False |
| 22041031 | 06 (Orlando) 99 (Pumpkin Latte) | True | True |

Now we have our answer:

- **Miami**: `is_in_stock = False`. Sales were zero because we couldn't sell it.
- **Orlando**: `is_in_stock = True`. If sales were zero, then the customers genuinely hate it.

### Designing the Coverage Table
A coverage table is dense. Unlike a sales fact table (which is "sparse"—only recording actual events), a coverage table typically records a row for **every product** in **every store** for **every day**.

If you have 1,000 stores and 5,000 products, you are generating **5 million rows a day**, even if no one buys anything.

Because of this weight, we design them carefully:

1. **The Grain**: Snapshot periodic (usually daily).
2. **The measure**: Often just a boolean flag (`1` for available, `0` for unavailable) or a simple integer (`quantity_on_hand`).

### The Magic of the Cross Join
When you combine a **coverage table** with a **sales table**, you unlock the "Sell-Through Rate," one of the most powerful metrics in retail.

- **Numerator (from Fact Sales)**: How many did we sell?
- **Denominator (from Coverage)**: How many could we have sold?

If you don't have a coverage table, you can sometimes simulate one using a **cross join** (Cartesian Product) in SQL to generate the "Universe of Possibility" on the fly:

```sql
-- The "Theoretical" Universe
WITH AllPossibilities AS (
    SELECT d.date_key, s.store_key, p.product_key
    FROM dim_date AS d
    CROSS JOIN dim_store  AS s
    CROSS JOIN dim_product AS p
    WHERE d.date_key = '20241031'
)

-- Left Join to see what actually happened
SELECT
    ap.store_key,
    ap.product_key,
    COALESCE(f.sales_amount, 0) AS actual_sales
FROM AllPossibilities AS ap
LEFT JOIN fact_sales AS f
    ON ap.store_key = f.store_key
    AND ap.product_key = f.product_key
```

This query forces the database to acknowledge the zeros. By creating the scaffolding of "All Possibilities" first, the `LEFT JOIN` exposes the holes where sales data is missing.

## Quiz

<quiz>
What distinguishes a 'factless' fact table from a standard transaction fact table?
- [ ] It is used exclusively for storage efficiency and cannot be queried.
- [ ] It is a temporary table used during the ETL process but never stored.
- [ ] It contains no foreign keys, only text descriptions.
- [x] It contains only foreign keys and no numeric measure columns.

</quiz>

<quiz>
If a 'factless' fact table has no numeric column to sum, how do you calculate the volume of events (e.g., total attendees)?
- [x] You count the rows.
- [ ] You cannot calculate volume; you can only list the participants.
- [ ] You multiply the foreign keys together.
- [ ] You must join it to a dimension table and sum the IDs.

</quiz>

<quiz>
Why is it generally poor design to track 'Training Courses Taken' as a list inside the Employee Dimension  (e.g., `courses: 'Latte Art, Safety, Management'`)?
- [x] It violates first normal form and makes cardinality difficult to manage.
- [ ] Dimensions are read-only and cannot be updated with new courses.
- [ ] Text columns take up too much storage space compared to integers.
- [ ] It prevents the use of surrogate keys.

</quiz>

<quiz>
What is the primary purpose of a 'Coverage Table'?
- [ ] To map many-to-many relationships.
- [ ] To list all insurance policies for employees.
- [ ] To summarize large fact tables into smaller aggregates.
- [x] To track the 'Negative Space' or things that didn't happen (but could have).

</quiz>

<quiz>
Why can't a standard sales fact table differentiate between 'No Demand' and 'Out of Stock'?
- [ ] Because it doesn't have a 'Store' dimension linked to it.
- [ ] Because SQL cannot handle zero values.
- [ ] Because NULL values in the Amount column are ambiguous.
- [x] Because it only records events that actually occurred (sparse data).

</quiz>

<quiz>
What is the 'Denominator Problem' in the context of retail analysis?
- [ ] Managing the large size of fact tables.
- [x] Calculating a sell-through rate requires knowing how many items *could* have been sold.
- [ ] The challenge of summing up additive facts.
- [ ] The difficulty in dividing by zero in SQL.

</quiz>

<quiz>
A coverage table is typically 'Dense.' What does this mean?
- [x] It contains a row for every combination of dimension (e.g., Product/Store/Day), even if nothing happened.
- [ ] It stores data in a compressed binary format.
- [ ] It is difficult to understand.
- [ ] It has more columns than a standard fact table.

</quiz>

<quiz>
Which SQL operation is useful for generating a 'Theoretical Universe' of possibilities if you don't have a physical coverage table?
- [ ] `UNION ALL`
- [ ] `INNER JOIN`
- [x] `CROSS JOIN`
- [ ] `GROUP BY`

</quiz>

<quiz>
What is a 'Dummy Metric' in a factless fact table?
- [ ] A placeholder for future data.
- [x] A column (usually equal to 1) added to allow BI tools to perform a SUM operation.
- [ ] A metric that is known to be inaccurate.
- [ ] A column filled with random numbers for testing.

</quiz>

<quiz>
The lesson of module 12 is that data engineering is not just about recording transactions but about…
- [ ] Creating pretty charts.
- [ ] Writing complex Python scripts.
- [ ] Minimizing storage costs.
- [x] Modeling reality.

</quiz>

<!-- mkdocs-quiz results -->