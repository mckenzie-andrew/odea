We have spent the entire course conditioning your brain to look for **measures**.

I told you that dimensions are "Nouns" (text) and Facts are "verbs" (numbers). I told you that a fact table exists to sum things up: `SUM(sales_amount)`, `AVG(duration_seconds)`, `MAX(temperature)`.

But what happens when the business process is an event that has no magnitude?

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

## 12.2 Coverage Tables
if the factless fact table tracks "Events without numbers," the **coverage table** tracks "Non-events."

This is one of the hardest questions to answer in SQL, "Which products sold zero units today?"

It sounds simple. But think about the physical nature of your `fact_sales` table. It is a log of transactions.

- If a customer buys a croissant, we write a row.
- If a customer buys a latte, we write a row.
- If a customer *ignores* the bagel, we write… nothing. The database remains silent.

If you query `SUM(sales)` for the bagel, you don't get 0. You get `NULL` (or the row simply disappears from the result set). You cannot count what isn't there.

This is the **negative space problem**. To analyze what *didn't* happen, we need to first define the "Universe of What Could Have Happened."

### The "Denominator" Problem
