If the previous chapters were about reading the library's catalog, this is the moment you pick up a pen and start writing your books.

Up until now, our relationship with the database has been somewhat passive. We've asked questions, filtered noise, and organized the answers. We've been observed. But databases aren't just archives; they are living, breathing repositories of state.

In this module, we enter the realm of **DML (Data Modification Language)**. We are going to change the world, or at least, the state of our tables.

Our first tool is the scalpel of data entry: the `INSERT` statement.

## 10.1 `INSERT`
Let's assume we just finished module 9 and ran a `CREATE TABLE` command to build a simple inventory for a fantasy RPG shop. Here is our empty container:

```sql
CREATE TABLE inventory (
    item_id INT,
    item_name VARCHAR(50),
    price DECIMAL(10, 2),
    quantity INT
);
```

Currently, the table is a ghost town. `SELECT * FROM inventory` returns nothing but a hollow echo. We need to populate it.

The command to add a single row looks like this:

```sql
INSERT INTO inventory (item_id, item_name, price, quantity)
VALUES (1, 'Rusty Dagger', 15.50, 10);
```

Let's break this down:

1. `INSERT INTO`: These two keywords (always used together) function as the verb. You are telling the database engine, "I have a package to deliver."
2. `inventory`: This is the destination. It's the table name.
3. `(item_id, …)`: This is the **column list**. It tells the database which specific attributes you are about to provide data for.
4. `VALUES`: This keyword acts as a separator. Everything before it is *metadata* (where is it going?); everything after it is the *actual* data.
5. `(1, 'Rusty Dagger', …)`: This is the payload. A comma-separated list of literals.

!!! example "Analogy: The Scantron Sheet"

    Think of a database row like a standardized test bubbling sheet (Scantron).

    - The **column list** tells you *which question numbers* you are answering.
    - The **VALUES** are the answers you bubble in.

    If you skip a question number in the list, the machine assumes you didn't answer it (NULL). If you try to bubble in "C" for question 4, but question 4 requires a number, the machine spits out the paper.

### The "Zipper" Alignment
The most critical concept in an `INSERT` statement is **positional correspondence**. The database does not know that `15.50` is a price just because it looks like money. It knows it is a price because it is the *third* item in the `VALUES` list, and `price` is the *third* item in the column list.

### Explicit vs. Implicit Columns
Technically, SQL allows you to be lazy. You can write an  insert statement without listing the columns, like this:

```sql
-- The lazy way
INSERT INTO inventory
VALUES (2, 'Healing Potion', 50.00, 5);
```

This works… for now. The database looks at the table definition, sees there are 4 columns defined in a specific order, and assumes your values match that exact order.

**Do not do this**.

!!! warning "The Schema Drift Trap"

    Imagine you write the "lazy" query above in your application code. It runs fine for months.

    Then, next year, a colleague adds a new column to the table: `created_at DATE`, and they put it as the *first* column in the table definition.

    Suddenly, your "lazy" query breaks. It tries to shove the integer `2` (your `item_id`) into the new `created_at` data column. The database throws an error, your application crashes, and you get a phone call at 3:00 AM.

    **Always list your columns explicitly**. It makes your code robust and readable.

### Handling Partial Data (NULLs)
What if we have an item, but we don't know the price yet? Or perhaps the `item_id` is an **identity** column (also called auto-increment) where the database generates the number for us?

We simply omit those columns from our list.

Let's say our table allows `price` to be empty (`NULL`), and `item_id` generates itself automatically. We only need to provide the name and quantity:

```sql
INSERT INTO inventory (item_name, quantity)
VALUES ('Mystery box', 3);
```

Here is the logic flow the database performs:

1. **Check Columns**: Okay, the user gave me `item_name` and `quantity`.
2. **Fill Gaps**: What about `price`? The user didn't mention it. Does the table definition allow `NULL`? Yes? Okay, insert a `NULL`.
3. **Generate Keys**: What about `item_id`? The user didn't mention it. Is it an auto-increment column? Yes? Okay, I'll generate the next number.

If you try to omit a column that is **NOT NULL** and has no **DEFAULT** value, the database will reject the insert with an error.

### Inserting Multiple Rows
Back in the old days, if you wanted to add three items, you wrote three `INSERT` statements. It was chatty and slow. Modern SQL allows you to batch these together in a single command.

You simply separate your value tuples with commas:

```sql
INSERT INTO inventory (item_name, price, quantity)
VALUES
    ('Iron Shield', 45.00, 2),
    ('Wooden Staff', 12.00, 5),
    ('Invisibility Cloak', 999.99, 1);
```

This is significantly faster because the database only has to parse the `INSERT INTO` command once and check permissions once. It just iterates through the data payload.

### The Power Move: `INSERT INTO … SELECT`
This is where your knowledge of set theory and querying pays off.

Occasionally you would rather not type values manually. Occasionally you want to copy data from one table to another, perhaps moving old orders to an archive table or creating a summary table for reporting.

You can feed the output of a `SELECT` statement directly into an `INSERT`.

Imagine we have a `new_shipment` table that acts as a loading dock. We want to move everything from the loading dock into the main `inventory`.

```mermaid
graph LR
    A[Table: new_shipment] -->|SELECT| B(Data Set)
    B -->|INSERT| C[Table: inventory]
```

The syntax replaces the `VALUES` keyword with a `SELECT` query:

```sql
INSERT INTO inventory (item_name, price, quantity)
SELECT
    product_name, -- Maps to item_name
    suggested_retail_price, -- Maps to price
    count_received -- Maps to quantity
FROM new_shipment
WHERE is_damaged = FALSE;
```

**Notice a few things here**:

1. **No `VALUES` keyword**: We aren't providing literals; we are providing a set.
2. **Column Mapping**: The column names in `new_shipment` (`product_name`) *do not have to match* the name in `inventory` (`item_name`). SQL only cares about the **data types** and the **order**.
3. **Filtering**: We used a `WHERE` clause. We are only inserting undamaged items. This is the beauty of set theory: we defined a specific subset of data and poured it into our table.

!!! info "Under the Hood: Transaction Logs"

    When you run a massive `INSERT … SELECT` that moves a million rows, the database writes every single change to a transaction log for safety. If your disk is full, the operation might fail. We'll discuss how to manage this later in the chapter with transactions.

## 10.2 `UPDATE`
We have successfully inserted data into our `inventory`. Our shop is stocked. The `Rusty Dagger` is sitting there at $15.50 with a quantity of 10.

But the world is not static. Inflation happens. Items get sold. We realize we misspelled "Dagger" as "Daggr."

If `INSERT` is writing a new page, `UPDATE` is the red pen. It allows you to modify existing records without deleting them and starting over.

### The Anatomy of a Modification
The `UPDATE` statement is powerful and dangerous. It doesn't ask, "Are you sure?" It just does what you tell it to do.

Here is the basic syntax to correct a spelling mistake:

```sql
UPDATE inventory
SET item_name = 'Rusty Dagger'
WHERE item_id = 1;
```

Let's decode the keywords:

1. `UPDATE inventory`: "I want to change the state of the *inventory* table."
2. `SET`: "Here is the list of changes I would like to make."
3. `item_name = 'Rusty Dagger'`: The assignment. "Make the `item_name` column equal to this new string."
4. `WHERE`: "Only apply this change to rows that match this condition."

### The "WHERE" Clause: Your Safety Catch
I cannot stress this enough: **The `WHERE` clause is the most important part of an `UPDATE` statement**.

If you omit the `WHERE` clause, the database assumes you want to update **every single row** in the table.

```sql
-- > Warning: DO NOT RUN THIS
UPDATE inventory
SET price = 0.00;
```

If you run the code above, congratulations: you just gave away your entire inventory for free. Every single row's price is now 0.00. There is no "undo" button in SQL.

!!! example "Analogy: The Sniper vs. The Nuke"

    - **With `WHERE`**: You are a sniper. You pick a specific target (`item_id = 1`) and make the precise change.
    - **Without `WHERE`**: You are a nuke. You change everything in the blast radius (the whole table).

    **Pro-Tip**: Before writing an `UPDATE`, experienced developers often write a `SELECT` first using the same `WHERE` clause to verify exactly which rows they are about to hit.

### Updating Multiple Columns
You can change multiple attributes of a record at the same time. Just separate the assignments with a comma.

Let's say we want to rebrand the 'Rusty Dagger.' We will clean it up, raise the price, and change the name.

```sql
UPDATE inventory
SET
    item_name = 'Polished Dagger',
    price = 25.00
WHERE item_id = 1;
```

### Relative Updates (Math on the Fly)
This is where `UPDATE` shines. You don't always set a value to a static number (like `25.00`). Typically, you want to set a value *relative to what it currently is*.

Imagine a customer buys one "Polished Dagger." We need to lower the quantity by 1. We don't need to know that the current quantity is 10. We just need to tell the database, "Whatever the quantity is right now, make it one less."

```sql
UPDATE inventory
SET quantity = quantity - 1
WHERE item_id = 1;
```

**How the Engine Thinks**:

1. **Locate Row**: Find the row where `item_id` is 1.
2. **Read Current Value**: Look at the `quantity` column. It's `10`.
3. **Calculate**: Do the math: `10 - 1 = 9`.
4. **Write**: Overwrite `10` with `9`.

You can use this for store-wide price adjustments too. Let's apply a 10% inflation hike on all items:

```sql
-- Notice: No WHERE clause!
-- We actually WANT to affect every row here.
UPDATE inventory
SET price = price * 1.10;
```

## 10.3 `DELETE` vs. `TRUNCATE`
Data has a lifecycle. Occasionally an order is cancelled, a user account is closed, or you simply made a mistake during testing. In SQL, there are two primary ways to remove data, and confusing them is a rite of passage for every junior engineer.

One is a scalpel; the other is a sledgehammer.

### The Scalpel: `DELETE`
The `DELETE` statement is used to remove specific rows from a table. It is precise, careful, and polite.

#### Syntax

```sql
DELETE FROM inventory
WHERE item_id = 1;
```

Let's look at the keywords:

1. `DELETE FROM inventory`: "I want to remove rows from the inventory table."
2. `WHERE`: "But only the ones that match this condition."

#### The Mechanism

When you run a `DELETE` statement, the database engine goes *row by row*. It looks at the row, checks if it matches your criteria, deletes it, and writes an entry in the transaction log saying, "I deleted row #1." Then it moves to the next row.

Because it does this work row-by-row, `DELETE` can be relatively slow if you are deleting millions of records.

!!! warning "The `WHERE` Clause Trap"

    Just like `UPDATE`, if you omit the `WHERE` clause, **you delete everything**.

    ```sql
    -- This empties the table, but keeps the table structure.
    DELETE FROM inventory;
    ```

    The table wrapper still exists (columns, data types), but it is now an empty shell.

### The Sledgehammer: `TRUNCATE`
Occasionally, you would rather not delete specific items. You want to wipe the slate clean. You would like to dump the entire contents of the table and start fresh, perhaps to reload a test environment.

For this, we use `TRUNCATE`.

#### The Syntax

```sql
TRUNCATE TABLE inventory;
```

#### The Mechanism

While `DELETE` goes row-by-row saying, "Delete this… okay, log it. Delete this… okay, log it," `TRUNCATE` is much more aggressive.

`TRUNCATE` tells the database, "I don't care what is inside this table. Deallocate the data storage pages entirely."

It doesn't scan the rows. It just drops the storage containers holding the data. Because of this, it is **instantaneous**, even if the table has 100 million rows.

### The "Soft Delete" Pattern
Here is a secret from the industry: **We rarely actually delete data**.

Storage is cheap. Data is valuable. If a user "deletes" their account, we usually want to keep the record for analytics, legal auditing, or the inevitable "Oops, I didn't mean to do that" support ticket.

Instead of running a `DELETE` statement, we use a technique called a **soft delete**.

1. We add a column to the table called `is_active` (boolean) or `deleted_at` (timestamp).
2. When a user deletes an item, we actually run an `UPDATE`.

```sql
-- The user clicked "delete," but we are actually updating.
UPDATE inventory
SET is_active = FALSE
WHERE item_id = 1;
```

3. Then, in all our `SELECT` queries, we filter out the dead rows:

```sql
SELECT * FROM inventory
WHERE is_active = TRUE;
```

This preserves history while effectively removing the item from the user's view.

## 10.4 Managing Transactions
We have arrived at the most critical safety feature in SQL: **The Transaction**.

Up until this moment, we have been operating in what is known as **auto-commit** mode. This means that as soon as you hit "run" on an `INSERT`, `UPDATE`, or `DELETE`, the change is permanent. It is written to the disk; if you accidentally deleted the wrong row, it is gone forever.

But what if you could treat a series of SQL commands like a draft? What if you could run five dangerous commands, look at the results to ensure they are perfect, and *then* decide whether to save them or scrap them?

This is a transaction. It is the "undo" button.

### The Bank Transaction Analogy (ACID)
To understand why transactions exist, we don't look at a shop inventory; we look at a bank.

Imagine you are transferring $100 from Alice to Bob. This is actually two separate operations:

1. Subtract $100 from Alice.
2. Add $100 to Bob.

Now, imagine the power goes out right after step 1 but before step 2.

Alice has lost $100. Bob hasn't received it. The money has vanished into the void. This is a database administrator's nightmare.

We need a guarantee that says, "Either BOTH of these things happen, or NEITHER of them happen." This concept is called **atomicity** (the "A" in ACID compliance).

### The Three Magic Keywords
To control this manually, we use three specific keywords.

#### 1. `BEGIN` (or `START TRANSACTION`)

This disables auto-commit. It tells the database, "Everything I type from now on is provisional. Don't write it to the permanent storage ledger yet. Just keep it in a scratchpad."

#### 2. `ROLLBACK`

This is the panic button. It tells the database, "I made a mistake. Crumple up the scratchpad and throw it away. Pretend I never typed anything since the last `BEGIN`."

#### 3. `COMMIT`

This is the save button. It tells the database, "Everything looks good. Write these changes to the permanent ledger."

### The Workflow in Action
Let's try a dangerous operation. We are going to fire an employee (delete them) and reassign their laptop to the inventory pool (update).

```sql
-- Step 1: Open the safety bubble
BEGIN TRANSACTION;

-- Step 2: Delete the employee (Dangerous!)
DELETE FROM employees
WHERE employee_id = 101;


-- Step 3: Update the laptop inventory
UPDATE inventory
SET quantity = quantity + 1
WHERE item_name = 'ThinkPad X1';

-- Pause here
```

At this exact moment, if you run `SELECT * FROM employees`, you will see that employee 101 is gone. **However**, you are the *only* person who sees this.

If your colleague queries the database from their computer, they *still see* employee 101. The database is isolating your changes in a "sandbox" until you confirm them.

**Scenario A: The "Oops" Moment**: You realize you just deleted the CEO (ID 101) instead of the intern.

```sql
ROLLBACK;
-- Phew, employee 101 is back. The inventory update is undone.
-- It's as if nothing ever happened.
```

**Scenario B: The "All Clear"**: You verify the IDs are correct.

```sql
COMMIT;
-- BAM. The changes are now visible to everyone.
-- There is no going back now.
```

### The "All or Nothing" Rule
Transactions are essential for data integrity. If you have a script that does 50 complex updates, wrap them in a transaction.

If an error occurs on command #49 (e.g., the server crashes or you violate a data constraint), the database engine will automatically **ROLLBACK** the entire transaction. It ensures you never end up with half-finished data (like Alice losing money that Bob never gets).

!!! warning "The Lockout Trap"

    When you begin a transaction and modify a row, you **lock** that row. No one else can update that row until you finish.

    If you type `BEGIN`, update a row, and then go to lunch without typing `COMMIT`, you might block the entire company from working on that data. **Rule of thumb**: Keep your transactions short and sweet.

## Quiz

<quiz>
Why is it considered a best practice to explicitly list the column names when writing an `INSERT` statement (e.g., `INSERT INTO table (col1, col2) …`)?
- [ ] It is required by the SQL standard; the query will strictly fail without it.
- [ ] It makes the database engine process the insertion significantly faster.
- [ ] It allows you to bypass `NOT NULL` constraints defined on the table.
- [x] It prevents errors if the table structure changes (e.g., a new column is added) or columns are reordered.

</quiz>

<quiz>
You need to copy all rows from a `new_shipments` table into your main `inventory` table. Which statement structure is most efficient?
- [ ] `UPDATE inventory SET .. FROM new_shipments`
- [ ] `SELECT * INTO inventory FROM new_shipments`
- [ ] `INSERT INTO inventory VALUES (SELECT * FROM new_shipments)`
- [x] `INSERT INTO inventory SELECT .. FROM new_shipments`

</quiz>

<quiz>
What is the result of executing the following query? `UPDATE employees SET salary = 500000;`
- [ ] It prompts the user for confirmation before proceeding.
- [x] The salary for every single row in the employees table is set to 500,000.
- [ ] It updates only the first row it finds.
- [ ] The database throws an error because a `WHERE` clause is mandatory.

</quiz>

<quiz>
You want to increase the price of a specific item by $5.00, regardless of its current price. Which syntax is correct?
- [x] `SET price = price + 5.00`
- [ ] `SET price = NEW(price) + 5.00`
- [ ] `SET price += 5.00`
- [ ] `SET price = 5.00`

</quiz>

<quiz>
Which of the following describes the key difference between `DELETE` and `TRUNCATE`?
- [ ] `DELETE` is faster than `TRUNCATE` because it scans the table first.
- [x] `TRUNCATE` is the entire table, whereas `DELETE` is specific rows designated by the query.
- [ ] `TRUNCATE` allows you to use a `WHERE` clause.
- [ ] `DELETE` can be rolled back, but `TRUNCATE` cannot.

</quiz>

<quiz>
Why is `TRUNCATE` generally much faster than `DELETE` when clearing a large table?
- [ ] It does not update the transaction log at all.
- [x] It deallocates the data pages entirely rather than logging every individual row deletion.
- [ ] It ignores foreign key constraints.
- [ ] It executes in parallel, whereas `DELETE` is single-threaded.

</quiz>

<quiz>
What is the primary mechanism of a "Soft Delete"?
- [ ] Removing the data but keeping the index pointers.
- [ ] Moving the data to a temporary table.
- [ ] Using the `DELETE SOFT` keyword.
- [x] Updating a flag column (e.g., `is_deleted = true`) instead of removing the row.

</quiz>

<quiz>
In a transaction, what command is used to save the changes permanently to the database?
- [ ] `SAVE`
- [ ] `PUSH`
- [x] `COMMIT`
- [ ] `ROLLBACK`

</quiz>

<quiz>
If you perform an `INSERT` and omit a column that allows NULLs and has no default value, what happens?
- [ ] The database inserts the number 0 or an empty string.
- [ ] The database prompts you to provide a value.
- [ ] The `INSERT` fails with an error.
- [x] The database inserts a `NULL` value into that column.

</quiz>

<quiz>
What happens if a power failure occurs in the middle of a transaction (after `BEGIN`, but before `COMMIT`)?
- [ ] The transaction pauses and resumes when power is restored.
- [x] The database automatically performs a `ROLLBACK` when it restarts, undoing the partial changes.
- [ ] The changes made up to that point are saved.
- [ ] The database enters a corrupted state and requires manual repair.

</quiz>

<!-- mkdocs-quiz results -->

## Lab
Please complete module 10 labs in the companion GitHub repository.

## Lab Solutions

!!! warning "Don't Cheat Yourself"

    Before viewing any of the solutions below, please ensure you have given the challenge an honest try. The worst thing you can do to yourself while learning is to not "accept the struggle." The struggle is what cements the information. Discovering the answer through trial and error is the only way to truly learn.

??? note "Challenge 1 Solution"

    ```sql
    CREATE TABLE product_experiments AS
    SELECT *
    FROM products
    WHERE 1 = 0;
    ```

??? note "Challenge 2 Solution"

    ```sql
    INSERT INTO product_experiments (
        product_id,
        product_category_name,
        product_weight_g, 
        product_length_cm, 
        product_height_cm, 
        product_width_cm
    )
    VALUES (
        'PROTOTYPE-001', 
        'experimental_tech', 
        500, 20, 10, 15
    );
    ```

??? note "Challenge 3 Solution"

    ```sql
    INSERT INTO product_experiments (
        product_id,
        product_category_name,
        product_weight_g,
        product_length_cm,
        product_height_cm,
        product_width_cm
    )
    VALUES 
        ('PROTOTYPE-002', 'experimental_tech', 600, 22, 12, 15),
        ('PROTOTYPE-003', 'experimental_Tech', 550, 20, 10, 15);
    ```

??? note "Challenge 4 Solution"

    ```sql
    INSERT INTO product_experiments
    SELECT * FROM products
    WHERE product_category_name = 'moveis_escritorio';
    ```

??? note "Challenge 5 Solution"

    ```sql
    INSERT INTO product_experiments (
        product_id,
        product_category_name,
        product_weight_g,
        product_length_cm,
        product_height_cm,
        product_width_cm
    )
    VALUES ('DIGITAL-001', 'software', NULL, NULL, NULL, NULL);
    ```

??? note "Challenge 6 Solution"

    ```sql
    UPDATE product_experiments
    SET product_category_name = 'rd_lab'
    WHERE product_category_name = 'experimental_tech';
    ```

??? note "Challenge 7 Solution"

    ```sql
    UPDATE product_experiments
    SET product_length_cm = product_length_cm + 2
    WHERE product_category_name = 'moveis_escritorio';
    ```

??? note "Challenge 8 Solution"

    ```sql
    UPDATE product_experiments
    set 
        product_weight_g = 520,
        product_height_cm = 12
    WHERE product_id = 'PROTOTYPE-001';
    ```

??? note "Challenge 9 Solution"

    ```sql
    UPDATE product_experiments
    SET product_weight_g = 0
    WHERE product_id = 'DIGITAL-001';
    ```

??? note "Challenge 10 Solution"

    ```sql
    DELETE FROM product_experiments
    WHERE product_id = 'PROTOTYPE-002';
    ```

??? note "Challenge 11 Solution"

    ```sql
    DELETE FROM product_experiments
    WHERE product_category_name = 'software';
    ```

??? note "Challenge 12 Solution"

    ```sql
    -- Cell 1:
    DELETE FROM orders_temp
    WHERE customer_id = '00012a2ce6f8dcda20d059ce98491703';

    -- Cell 2:
    DELETE FROM customers_temp
    WHERE customer_id = '00012a2ce6f8dcda20d059ce98491703';
    ```

??? note "Challenge 13 Solution"

    ```sql
    TRUNCATE TABLE product_experiments;
    ```

??? note "Challenge 14 Solution"

    ```sql
    BEGIN;

    INSERT INTO orders_temp (order_id, order_status)
        VALUES ('ORD-999', 'created');

    COMMIT;
    ```

??? note "Challenge 15 Solution"

    ```sql
    BEGIN;

    UPDATE product_experiments
    SET product_weight_g = 0;

    ROLLBACK;
    ```

??? note "Challenge 16 Solution"

    ```sql
    BEGIN;

    DELETE FROM product_experiments
    WHERE product_id = 'PROTOTYPE-003';

    INSERT INTO products (product_id)
    VALUES ('PROTOTYPE-003');

    COMMIT;
    ```

??? note "Challenge 17 Solution"

    ```sql
    DROP TABLE product_experiments, customers_temp, orders_temp, sellers_copy;
    ```