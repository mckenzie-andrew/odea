In the previous module, we learned how to join tables **horizontally**. We took the `orders` table and the `customers` table and stitched them together side-by-side to create a wider row containing data from both.

But occasionally, you would rather not make the result *wider*. You want to make it *taller*.

Imagine you have two separate lists of email addresses:

1. `attendees_2022.csv`
2. `attendees_2023.csv`

You would rather not join them (finding people who attended *both*). You just want to stack them into one giant "master list" of emails.

In Set Theory, this is known as a **union**. In SQL, we call these **set operators**.

## 6.1 `UNION` vs. `UNION ALL`
The `UNION` operator allows you to stack the results of two or more `SELECT` queries on top of each other.

**The Rules of the Stack**

To stack two queries, they must be "union compatible."  You cannot stack a pile of bricks on top of a pile of oranges.

1. **Same Number of Columns**: If query A returns 3 columns, query B must return 3 columns.
2. **Same Data Types**: Column 1 in query A must be the same type (e.g., text) as column 1 in query B.
3. **Same Order**: The database stacks strictly by position, not by name.

### 1. `UNION ALL` (The Fast Stack)
`UNION ALL` is the simplest operator. It takes the rows from query A and blindly dumps the rows from query B right after them. It does **not** check for duplicates.

**Scenario**: We want a list of all transactions. Some are in the `sales` table; others are in the `refunds` table.

```sql
SELECT
    transaction_id,
    amount,
    'Sale' AS type
FROM sales
UNION ALL
SELECT
    refund_id,
    amount,
    'Refund' AS type
FROM refunds;
```

**Result**: If `sales` has 100 rows and `refunds` has 20 rows, the result will have exactly 120 rows.

### 2. `UNION` (The Unique Stack)
`UNION` (without the word `ALL`) does two things:

1. It stacks the rows.
2. It **removes duplicates**.

**Scenario**: We want a list of **all** unique client cities. We have clients in the `customers` table and the `suppliers` table.

```sql
SELECT
    city
FROM customers
UNION
SELECT
    city
FROM suppliers;
```

If "New York" appears in both tables, `UNION` will squash them into a single "New York" row in the final result.

!!! warning "Performance Cost"

    Just like `DISTINCT`, the standard `UNION` operator forces the database to sort and compare every single row to find duplicates. This is computationally expensive.

    **Best Practice**: Always default to `UNION ALL`. Only use `UNION` if you explicitly need to remove duplicates and know the cost is worth it.

### Visualizing the Difference
- `UNION ALL`: Appends table B to table A. (Preserves Duplicates).
- `UNION`: Appends B to A, then runs a `DISTINCT` filter.

## 6.2 `INTERSECT` and `EXCEPT`
These operators correspond directly to the classic Venn Diagrams you learned in math class.

### 1. `INTERSECT` (The Shared Area)
`INTERSECT` returns only the rows that appear in **BOTH** result sets.

**Scenario**: Which customers bought a product in 2022 **AND** also bought a product in 2023? (The "loyal" customers).

```sql
SELECT customer_id FROM orders_2022
INTERSECT
SELECT customer_id FROM orders_2023;
```

This effectively does the same work as an `INNER JOIN`, but the syntax is often cleaner when you are just comparing lists of IDs.

### 2. `EXCEPT` (The Difference)
`EXCEPT` returns rows that are in the **first** query but **NOT** in the second query. (*Note: Oracle calls this `MINUS`. Everyone else calls it `EXCEPT`).*

**Scenario**: Which customers bought something in 2022 but **did NOT** return in 2023? (The "lost" customer).

```sql
SELECT customer_id FROM orders_2022
EXCEPT
SELECT customer_id FROM orders_2023;
```

### Order matters
- `A EXCEPT B`: Rows in A that aren't in B.
- `B EXCEPT A`: Rows in B that aren't in A.

## Quiz

<quiz>
What is the primary difference between `UNION` and `UNION ALL`?
- [ ] `UNION` requires the columns to be named identically, while `UNION ALL` does not.
- [x] `UNION` removes duplicate rows, while `UNION ALL` keeps them.
- [ ] `UNION ALL` removes duplicate rows, while `UNION` keeps them.
- [ ] `UNION` can only combine two tables, while `UNION ALL` can combine many.

</quiz>

<quiz>
You have a query `SELECT id FROM table_A` (returns 10 rows) and `SELECT id FROM table_B` (returns 5 rows). If you use `UNION ALL`, how many rows will be in the result?
- [ ] It depends on how many duplicates exist.
- [x] 15
- [ ] 10
- [ ] 50

</quiz>

<quiz>
Which set operator corresponds to the overlapping section of a Venn diagram (rows present in **BOTH** queries)?
- [ ] `UNION`
- [ ] `EXCEPT`
- [ ] `CROSS APPLY`
- [x] `INTERSECT`

</quiz>

<quiz>
The order of queries matters for the `EXCEPT` operator (i.e., `Query A EXCEPT Query B` produces the same result as `Query B EXCEPT Query A`).
- [ ] True
- [x] False

</quiz>

<quiz>
Which of the following is NOT a requirement for two queries to be 'union compatible'?
- [ ] The columns must be in the same order.
- [x] The columns must have the same name.
- [ ] The columns must have compatible data types.
- [ ] They must have the same number of columns.

</quiz>

<quiz>
You want to find customers who placed an order in 2022 BUT did NOT place an order in 2023. Which operator should you use?
- [x] `EXCEPT`
- [ ] `INTERSECT`
- [ ] `INNER JOIN`
- [ ] `UNION`

</quiz>

<quiz>
Which operator is generally faster in terms of performance?
- [ ] `UNION`
- [x] `UNION ALL`

</quiz>

<quiz>
In Oracle SQL, the `EXCEPT` operator is known by a different keyword. What is it?
- [ ] `SUBTRACT`
- [ ] `DELETE`
- [x] `MINUS`
- [ ] `REMOVE`

</quiz>

<quiz>
Why would the following query fail? `SELECT name, age FROM students UNION SELECT name FROM teachers;`
- [x] Because the number of columns does not match.
- [ ] Because `name` is text and `age` is a number.
- [ ] Because the tables have different names.
- [ ] Because you cannot UNION people from different professions.

</quiz>

<quiz>
When stacking data with `UNION`, where does the final result set get its column names from?
- [ ] The second query.
- [x] The first query.
- [ ] It generates generic names like `col1`, `col2`.
- [ ] It combines the names (e.g., `name_teacher_student`).

</quiz>

<!-- mkdocs-quiz results -->

## Lab
Please complete module 6's labs in the companion GitHub repository.

## Lab Solutions

!!! warning "Don't Cheat Yourself"

    Before viewing any of the solutions below, please ensure you have given the challenge an honest try. The worst thing you can do to yourself while learning is to not "accept the struggle." The struggle is what cements the information. Discovering the answer through trial and error is the only way to truly learn.

??? note "Challenge 1 Solution"

    ```sql
    SELECT customer_state FROM customers
    UNION
    SELECT seller_state FROM sellers;
    ```

??? note "Challenge 2 Solution"

    ```sql
    SELECT customer_state from customers
    UNION ALL
    SELECT seller_state from sellers;
    ```

??? note "Challenge 3 Solution"

    ```sql
    SELECT
        customer_zip_code_prefix
    FROM customers
    UNION
    SELECT
        seller_zip_code_prefix
    FROM sellers;
    ```

??? note "Challenge 4 Solution"

    ```sql
    SELECT
        customer_city,
        customer_state
    FROM customers
    INTERSECT
    SELECT
        seller_city,
        seller_state
    FROM sellers;
    ```

??? note "Challenge 5 Solution"

    ```sql
    SELECT
        customer_city,
        customer_state
    FROM customers
    EXCEPT
    SELECT
        seller_city,
        seller_state
    FROM sellers;
    ```

??? note "Challenge 6 Solution"

    ```sql
    SELECT
        customer_id,
        'Customer' AS title
    FROM customers
    UNION ALL
    SELECT
        seller_id,
        'Seller' AS title
    FROM sellers;
    ```

??? note "Challenge 7 Solution"

    ```sql
    SELECT customer_city
    FROM customers
    WHERE customer_state = 'RJ'
    UNION
    SELECT seller_city
    FROM sellers
    WHERE seller_state = 'RJ';
    ```

??? note "Challenge 8 Solution"

    ```sql
    SELECT
        customer_city
    FROM customers
    GROUP BY customer_city
    HAVING COUNT(*) > 2000
    UNION
    SELECT
        seller_city
    FROM sellers
    GROUP BY seller_city
    HAVING COUNT(*) > 50;
    ```

??? note "Challenge 9 Solution"

    ```sql
    SELECT
        customer_city,
        customer_state
    FROM customers
    UNION
    SELECT
        seller_city,
        seller_state
    FROM sellers
    ORDER BY customer_city ASC, customer_state ASC;
    ```

??? note "Challenge 10 Solution"

    ```sql
    (
        SELECT 
            customer_city, 
            customer_state 
        FROM customers
        EXCEPT 
        SELECT
            seller_city, 
            seller_state 
        FROM sellers
    )
        UNION
    (
        SELECT 
            seller_city, 
            seller_state 
        FROM sellers
        EXCEPT 
        SELECT 
            customer_city, 
            customer_state 
        FROM customers
    );
    ```
