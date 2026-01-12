Welcome to the power tools section of this course.

If you asked a veteran data analyst, "Which SQL feature changed your life the most?", 9 out of 10 would say **window functions**.

To understand why, we have to look at the flaw in `GROUP BY`. Think back to module 4. When you use `GROUP BY department`, you take 100 employee rows and **crush** them down into 5 department rows.

But what if you would rather not crush the rows? What if you wanted to keep all 100 employee rows but *also* see the department total next to each person?

## 8.1 The `OVER()` Clause and Partitioning
**The Scenario**: You want a report showing:

1. Employee name.
2. Their salary.
3. **The Average Salary of their Department** (so you can see if they are underpaid).

Before window functions, you would have to write a complex correlated subquery or a CTE with a join. It was messy. With window functions, it's trivial.

### The Syntax: `OVER()`
A window function is an aggregate function (like `SUM`, `AVG`, `COUNT`) that doesn't collapse rows. Instead, it looks through a "window" of relevant rows to calculate a value and then pastes that value onto the current row.

The magic keyword is `OVER()`.

```sql
SELECT
    first_name,
    department,
    salary
    AVG(salary) OVER (PARTITION BY department) AS dept_avg_salary
FROM employees;
```

**What just Happened?**

1. `AVG(salary)`: This is the calculation we want to perform.
2. `OVER (...)`: This tells SQL, "Do not collapse the rows! Keep the original rows."
3. `PARTITION BY department`: This defines the "window." It tells SQL to calculate the average *separately* for each department.

When the database looks at an employee in "Sales", the "window" is just the other sales people. When it looks at an "Engineer", the "window" slides over to include only Engineers.

!!! example "Analogy: The Reference Library"

    - `GROUP BY` is like taking all the books in the library, burning them, and just writing "Science: 500 books" on a sticky note. You lost the individual books.
    - `PARTITION BY` is like walking through the library. You pick up a specific book (the current row).  You look around at the shelf it belongs to (the partition). You write that number on a bookmark and put it inside the specific book. You put the book back on the shelf unharmed.

### `PARTITION BY` vs. The Whole Table
If you leave the parentheses empty (`OVER()`), the window becomes the entire table.

```sql
SELECT
    first_name,
    salary,
    SUM(salary) OVER() AS total_company_payroll,
    salary / SUM(salary) OVER() AS my_share_of_payroll
FROM employees;
```

Here, `SUM(salary) OVER()` adds up the salary of *every single row* in the result set and prints that massive number next to every employee. This allows you to easily calculate percentages (e.g., "I make 0.05% of the total company payroll") in a single step.

### Why is This Better than a Subquery?

1. **Performance**: The database engine is optimized to do this in one pass. Subqueries often require rescanning the table.
2. **Cleanliness**: You don't need joins or nested `SELECT`s.
3. **Flexibility**: You can have multiple windows in the same query!

```sql
SELECT
    first_name,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg,
    AVG(salary) OVER (PARTITION BY job_title) AS title_avg
FROm employees;
```

In this query, for every single row, SQL calculates the average for the department *and* the average for the job title simultaneously. Trying to do this with subqueries would be a nightmare.

## 8.2 Ranking Functions (`ROW_NUMBER`, `RANK`, `DENSE_RANK`)
In the previous section, we used `PARTITION BY` to create buckets of data for calculation. Now, we are going to add a new ingredient to the mix: **order**.

Business leaders love leaderboards. They want to know the "Top 3 Salespeople," the "Most Recent 5 Orders," or the "Highest Paid Employee per Department."

Standard SQL `LIMIT` or `TOP` is great for getting the top 5 of the *entire* table. But getting the top 5 *per group* (per department, per region, per year) is a nightmare without window functions.

Enter the **ranking functions**.

### The New Syntax: Adding `ORDER BY`
To rank things, the database needs to know, "Rank by what?" You can't have a first place winner if you don't know if you are judging by speed, height, or points.

So, we add an `ORDER BY` clause *inside* the `OVER()` parentheses.

```sql
SELECT
    department,
    last_name,
    salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_rank
FROM employees;
```

**How to Read This**:

1. `PARTITION BY department`: "Split the employees into groups based on their department."
2. `ORDER BY salary DESC`: "Inside each group, line them up from highest salary to lowest."
3. `RANK()`: "Walk down the line and hand out medals: 1, 2, 3..."
4. **Reset**: When the database finishes one department and moves to the next, the rank counter resets to 1.

### The Three Contenders
There isn't just one way to rank data. The problem arises when two people cross the finish line at the same time. How do you handle **tiers**?

SQL provides three specific functions for this. They all look similar until you hit a tie.

1. `ROW_NUMBER()`: The ruthless pragmatist. It demands unique numbers. If two people tie, it arbitrarily picks one to be #1 and the other to be #2.
2. `RANK()`: The Olympic Judge. If two people tie for Gold (#1), they both get gold. The next person gets Bronze (#3). Silver (#2) is skipped.
3. `DENSE_RANK()`: The "Everyone Gets a Medal" approach. If two people tie for gold (#1), they both get gold. The next person gets silver (#2). No numbers are skipped.

Let's see this in action. Imagine a Sales team where Alice and Bob are tied with $5000 in sales.

| Name | Sales | ROW_NUMBER() | RANK() | DENSE_RANK() |
|:---|:---|:---|:---|:---|
| Alice | 5000 | 1 | 1 | 1 |
| Bob | 5000 | 2 | 1 | 1 |
| Charlie | 4000 | 3 | 3 | 2 |
| David | 3000 | 4 | 4 | 3 |

**Key Observations**:

- `ROW_NUMBER` forced Bob to be #2 (even though he sold the same as Alice).
- `RANK` gave them both #1, but **Charlie got punished**. He is #3 because two people beat him. The rank #2 does not exist.
- `DENSE_RANK` gave them both #1, and Charlie got #2. It is "dense" because there are no gaps in the numbering sequence.

!!! warning "The Arbitrary `ROW_NUMBER`"

    If you use `ROW_NUMBER()` and you have ties, the database engine decides who comes first based on random internal storage order. If you run the query twice, Alice and Bob might swap places.

    If deterministic results matter, add a tiebreaker to your order clause: `ORDER BY sales DESC, last_name ASC`.


### A Real-World Use Case: "Get the Top N per Group"
This is one of the most common interview questions and real-world tasks.

**Task**: "Find the highest-paid employee in every department."

We can use a CTE combined with a window function to solve this elegantly.

```sql
WITH RankedEmployees AS (
    SELECT
        first_name,
        last_name,
        department,
        salary,
        DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rank_num
    FROM employees
)
SELECT * FROM RankedEmployees
WHERE rank_num = 1;
```

**Why do we need the CTE?** Remember the order of execution from module 4? The `WHERE` clause runs *before* the `SELECT` clause. You cannot say `WHERE rank_num = 1` in the same query because `rank_num` hasn't been calculated yet!

So, we wrap the calculation in a CTE (step 1) and then filter the result (step 2).

## 8.3 Offset Functions (`LEAD`, `LAG`)
We have mastered the art of looking at the "group" (aggregate) and looking at the "rank" (ranking). Now, we are going to master the art of looking at the **neighbors**.

Data is often sequential. To understand a data point, you usually need to compare it to what happened immediately before it.

- "Did our sales go up **compared to last month**?"
- "How long was the website down **between this error and the next one**?"
- "What is the difference between this meter reading and the **previous reading**?"

In Standard SQL (without window functions), comparing row 5 to row 4 is incredibly difficult. You have to do a "self join" where you join the table to itself on `ID = ID - 1`. It's slow, brittle, and ugly.

With window functions, we just use **offset functions**.

### The Rearview Mirror: `LAG()`
`LAG()` allows you to access data from a previous row *without leaving the current row*. It reaches back in time and pulls a value forward.

**The Syntax**: `LAG(column_name, offset_amount, default_value)`

- `column_name`: What value do you want to grab?
- `offset_amount`: How many rows back? (default is 1).
- `default_value`: What if there is no previous row? (default is `NULL`).

**Scenario**: We have a table of monthly sales. We want to see the growth.

```sql
SELECT
    month,
    sales,
    -- Look back 1 row to get last month's sales
    LAG(sales, 1) OVER (ORDER BY month) AS previous_sales,
    -- Calculate the difference
    sales - LAG(sales, 1) OVER (ORDER BY month) AS monthly_growth
FROM monthly_sales;
```

**The Output**:

| Month | Sales | Previous_Sales | Monthly_Growth |
|:---|:---|:---|:---|
| Jan | 100 | NULL | NULL |
| Feb | 120 | 100 | +20 |
| Mar | 110 | 120 | -10 |


**Notice**:

1. **January's `Previous_Sales` is NULL.** There is no "Month 0" in our table, so `LAG` returns `NULL`.
2. **February's `Previous_Sales` is 100.** It successfully grabbed the value from the row above.
3. **The Calculation**: Once we have `sales` (120) and `previous_sales` (100) on the same row, simple subtraction (`120 - 100`) gives us the growth.

### The Telescope: `LEAD()`
`LEAD()` is simply the reverse of `LAG()`. It looks **forward** to the next row.

It is useful for calculating durations. If you have a log of "Event Start Times," the duration of event A is essentially `(Event B Start Time) - (Event A Start Time)`.

```sql
SELECT
    event_name,
    start_time,
    LEAD(start_time) OVER (ORDER BY start_time) AS next_event_start,
    -- Calculate how long this event lasted before the next one began.
    DATEDIFF(minute, start_time, LEAD(start_time) OVER (ORDER BY start_time)) AS duration_minutes
FROM schedule;
```

### The Null Problem (and Solution)
The first row (for `LAG`) or the last row (for `LEAD`) will always result in a `NULL` because there is no neighbor to talk to.

Occasionally this is fine. Other times, it breaks your math (because `100 - NULL = NULL`).

You can provide a **default value** as the third argument to fix this.

```sql
-- If there is no previous month, assume sales were 0.
LAG(sales, 1, 0) OVER (ORDER BY month)
```

Now, the calculation for January becomes `100 - 0 = +100` growth.

### Combining Partitioning and Ordering
Just like with ranking, you can use `PARTITION BY` to reset the logic.

If you are tracking sales **per employee**, you don't want Alice's first sale to be compared to Bob's last sale. You want to restart the calculation for every person.

```sql
SELECT
    employee_name,
    sale_date,
    amount,
    -- Compare this sale to THIS employee's previous sale
    LAG(amount) OVER (PARTITION BY employee_name ORDER BY sale_date) AS prev_amt
FROm sales;
```

When the database switches from Alice to Bob, `LAG` sees a "new partition" and returns `NULL` (or your default) for Bob's first row, ensuring data integrity.

## Quiz

<quiz>
What is the primary difference between `GROUP BY` and a window function using `OVER()`?
- [ ] Window functions collapse multiple rows into a single summary row, while `GROUP BY` keeps them separate.
- [x] `GROUP BY` collapses rows into a summary, whereas window functions calculate aggregates while retaining the original individual rows.
- [ ] `GROUP By` is faster than window functions.
- [ ] Window functions cannot calculate averages or sums.

</quiz>

<quiz>
In the clause `OVER (PARTITION BY department)`, what does the `PARTITION BY` keyword do?
- [x] It divides the result set into specific groups (windows) for calculation, resetting the function for each group.
- [ ] It sorts the results by department.
- [ ] It removes duplicate department rows.
- [ ] It hides columns that are not related to the department.

</quiz>

<quiz>
Which ranking function handles ties by assigning the same rank to tied rows and leaving a gap in the number sequence (e.g., 1, 1, 3)?
- [ ] `DENSE_RANK()`
- [x] `RANK()`
- [ ] `ROW_NUMBER()`
- [ ] `LEAD()`

</quiz>

<quiz>
If you want to assign a unique sequential integer to every row, distinguishing even between exact duplicates arbitrarily, which function should you use?
- [x] `ROW_NUMBER()`
- [ ] `RANK()`
- [ ] `COUNT()`
- [ ] `DENSE_RANK()`

</quiz>

<quiz>
What is the result of `LAG(sales, 1)` for the very first row in a partition?
- [ ] 0
- [x] NULL
- [ ] The value of the current row.
- [ ] An error.

</quiz>

<quiz>
You want to calculate "Month-over-Month" growth. Which combination of window function and math is correct?
- [ ] `LAG(sales, 1) - LEAD(sales, 1)`
- [ ] `sales - LEAD(sales, 1)`
- [x] `sales - LAG(sales, 1)`
- [ ] `sales - RANK()`

</quiz>

<quiz>
In the expression `RANK() OVER (PARTITION BY dept ORDER BY salary DESC)`, when does the rank counter reset to 1?
- [x] When the `department` changes.
- [ ] When the `salary` changes.
- [ ] Every time the query is run.
- [ ] It never resets; it ranks the whole table.

</quiz>

<quiz>
Which function allows you to access data from the *next* row in the result set?
- [ ] `LAG()`
- [x] `LEAD()`
- [ ] `NEXT()`
- [ ] `FORWARD()`

</quiz>

<quiz>
What happens if you use an aggregate function like `SUM(salary) OVER()` without a `PARTITION BY` clause?
- [x] It calculates the sum of the entire result set and displays that same number on every row.
- [ ] It calculates the sum for the current row only.
- [ ] It calculates the running total row by row.
- [ ] It returns an error because partitions are required.

</quiz>

<quiz>
Why would you use `DENSE_RANK()` instead of `RANK()`?
- [x] To ensure there are no gaps in the ranking sequence (e.g., 1, 2, 3).
- [ ] To skip numbers after a tie.
- [ ] To improve query performance.
- [ ] To break ties arbitrarily.

</quiz>

<!-- mkdocs-quiz results -->

## Lab
Please complete module 8's labs in the companion GitHub repository.

## Lab Solutions

!!! warning "Don't Cheat Yourself"

    Before viewing any of the solutions below, please ensure you have given the challenge an honest try. The worst thing you can do to yourself while learning is to not "accept the struggle." The struggle is what cements the information. Discovering the answer through trial and error is the only way to truly learn.

??? note "Challenge 1 Solution"

    ```sql
    SELECT
        order_id,
        order_item_id,
        price,
        SUM(price) OVER (PARTITION BY order_id) AS order_total
    FROm order_items;
    ```

??? note "Challenge 2 Solution"

    ```sql
    SELECT
        product_id,
        product_category_name,
        product_weight_g,
        AVG(product_weight_g) OVER (PARTITION BY product_category_name) AS avg_category_weight
    FROM products
    WHERE product_category_name IS NOT NULL;
    ```

??? note "Challenge 3 Solution"

    ```sql
    SELECT
        order_id,
        order_item_id,
        freight_value,
        freight_value / NULLIF(SUM(freight_value) OVER (PARTITION BY order_id), 0) AS freight_ratio
    FROM order_items;
    ```

??? note "Challenge 4 Solution"

    ```sql
    SELECT
        order_id,
        order_item_id,
        price,
        ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY price DESC) AS item_rank
    FROM order_items;
    ```

??? note "Challenge 5 Solution"

    ```sql
    SELECT
        seller_id,
        seller_city,
        seller_zip_code_prefix,
        RANK() OVER (PARTITION BY seller_city ORDER BY seller_zip_code_prefix) AS city_rank
    FROM sellers;
    ```

??? note "Challenge 6 Solution"

    ```sql
    SELECT
        product_id,
        product_category_name,
        product_length_cm,
        DENSE_RANK() OVER(PARTITION BY product_category_name ORDER BY product_length_cm) AS length_rank
    FROM products;
    ```

??? note "Challenge 7 Solution"

    ```sql
    SELECT
        c.customer_unique_id,
        o.order_id,
        o.order_purchase_timestamp,
        ROW_NUMBER() OVER (PARTITION BY c.customer_unique_id ORDER BY o.order_purchase_timestamp DESC) AS latest_order_rank
    FROM orders AS o
    JOIN customers AS c
        ON o.customer_id = c.customer_id;
    ```

??? note "Challenge 8 Solution"

    ```sql
    WITH ProductRanks AS (
        SELECT
            product_id,
            product_category_name,
            product_weight_g,
            RANK() OVER (PARTITION BY product_category_name ORDER BY product_weight_g DESC) AS product_rank
        FROM products
    )
    SELECT *
    FROM ProductRanks
    WHERE product_rank <= 3;
    ```

??? note "Challenge 9 Solution"

    ```sql
    SELECT
        order_id,
        order_item_id,
        price,
        SUM(price) OVER (PARTITION BY order_id ORDER BY order_item_id) AS running_total
    FROM order_items;
    ```

??? note "Challenge 10 Solution"

    ```sql
    SELECT
        seller_id,
        shipping_limit_date,
        price,
        LAG(price) OVER (PARTITION BY seller_id ORDER BY shipping_limit_date) AS previous_price
    FROM order_items;
    ```

??? note "Challenge 11 Solution"

    ```sql
    SELECT
        c.customer_unique_id,
        o.order_id,
        o.order_estimated_delivery_date,
        LEAD(o.order_estimated_delivery_date) OVER(PARTITION BY c.customer_unique_id ORDER BY o.order_estimated_delivery_date) as next_estimated_delivery
    FROM orders AS o
    JOIN customers AS c
        ON c.customer_id = o.customer_id;
    ```

??? note "Challenge 12 Solution"

    ```sql
    WITH OrderSequence AS (
        SELECT 
            c.customer_unique_id,
            o.order_purchase_timestamp,
            LAG(o.order_purchase_timestamp) OVER(PARTITION BY c.customer_unique_id ORDER BY o.order_purchase_timestamp) as prev_order_date
        FROM orders o
        JOIN customers c ON o.customer_id = c.customer_id
    )
    SELECT 
        customer_unique_id,
        order_purchase_timestamp,
        prev_order_date,
        DATE_PART('day', order_purchase_timestamp - prev_order_date) as days_since_last_order
    FROM OrderSequence
    WHERE prev_order_date IS NOT NULL;
    ```

??? note "Challenge 13 Solution"

    ```sql
    WITH MonthlySales AS (
        SELECT 
            DATE_PART('year', o.order_purchase_timestamp) as sales_year,
            DATE_PART('month', o.order_purchase_timestamp) as sales_month,
            SUM(oi.price) as total_revenue
        FROM orders o
        JOIN order_items oi ON o.order_id = oi.order_id
        GROUP BY 1, 2
    )
    SELECT 
        sales_year,
        sales_month,
        total_revenue,
        LAG(total_revenue) OVER(ORDER BY sales_year, sales_month) as prev_month_revenue
    FROM MonthlySales;
    ```

??? note "Challenge 14 Solution"

    ```sql
    WITH AvgOrderPrice AS (
        
        SELECT
            order_id,
            price,
            AVG(price) OVER (PARTITION BY order_id) AS avg_order_price
        FROM order_items
    )

    SELECT *
    FROM AvgOrderPrice
    WHERE price > avg_order_price * 1.5;
    ```
