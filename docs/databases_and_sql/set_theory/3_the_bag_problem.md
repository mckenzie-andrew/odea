I have a confession to make.

For the last two modules, we have been hinting that the database table is a Set. I then defined a Set as a collection of **distinct** objects. However, life is not so simple.

This is just the idealized, academic version of the truth. In the real world of SQL and data engineering, tables are not Sets. Tables are considered **bags**.

## 3.1 Sets vs. Bags
In strict Set Theory, uniqueness is enforced by the laws of physics. If you put the number 5 into a set that already contains the number 5, nothing happens. The set absorbs it like a raindrop falling into the ocean.

However, standard SQL databases do not enforce this by default. They are happy to let you stack identical rows on top of each other until your hard drive explodes.

If we insert the number 5 into a database table that already has 5, we get:

| Value |
|:---|
| 5 |
| 5 |

This structure—a collection where duplicates are allowed—is formally called a **multiset**, but in computer science, we almost always call it a **bag**.

### The Multiplicity Function
To understand a bag mathematically, we have to tweak our definition of membership.

In a Set, membership is a Boolean: "Is $x$ in $A$?" (True/False). In a Bag, membership is a count: "How *many* times is $x$ in $A$?"

We call this the **multiplicity** of the element, denoted as $m(x)$.

Let's look at a Bag called $B$:

$$
B = \{\text{Apple, Apple, Apple, Orange}\}
$$

- **Set View**: $x \in B$ is True for Apple.
- **Bag View**: $m(\text{Apple}) = 3$.

This seemingly small difference is the root of almost every "Why is my sum wrong?" bug you will encounter in your career.

### Why Do We Allow This Heresy?
If Sets are so much cleaner and logically sound, why did the inventors of SQL decide to use Bags? Why allow duplicates at all?

There are two reasons: one is about **performance**, and the other is about **reality**.

#### 1. The Cost of Deduplication
Imagine you are a teacher collecting exam papers for 30 students. You have a stack of papers. A student walks up and hands you theirs. You just put it on top of the stack.

- **Effort**: Near zero.
- **Time**: Instant.

Now, imagine you are a strict Set Theorist. A student hands you a paper. Before you accept it, you must look through *every single paper currently in the stack* to make sure this student hasn't already turned one in.

- **Effort**: Massive.
- **Time**: Gets slower with every new paper.

In database terms, enforcing "Set-ness" (uniqueness) is expensive. The database has to scan, sort, or hash the existing data every time you write a new row. To make databases fast, we default to the "Bag" model: just throw the data on the pile. We'll sort it out later.

#### 2. Distinct Events, Identical Data
Sometimes, duplicates are real.

Imagine a retail store. I walk in at 10:00 AM and buy a **Mars bar**. At 10:05 AM, I realize I'm still hungry, walk back in, and buy another **Mars bar**.

The transaction table records this:

| Item | Price |
|:---|:---|
| Mars Bar | 1.50 |
| Mars Bar | 1.50 |

These rows are identical in content. But they represent two distinct transfer events of money. If we forced this table to be a Set, the database would delete my second purchase. The store would lose money, and inventory would be wrong.

### The Shopping List vs. The Receipt
The best way to mental-model this is the difference between planning and doing.

- **The Set is the Shopping List**: It says "Milk, Eggs, Bread." It doesn't matter if you write "Milk" twice on the list; you only need to go to the dairy aisle once. The goal is *existence*.
- **The Bag is the Receipt**: It says "Milk @ $3.00, Milk @ $3.00." The cashier scanned it twice. You paid for it twice. You are taking home two jugs. The goal is *accumulation*.

### The Trap
So, if Bags are necessary, what's the problem?

The issue is the standard Set Theory operations (Union, Intersection, Difference) behave differently on Bags.

If you have a Set of Users $\{\text{Alice, Bob}\}$ and you join it with another Set of Users $\{\text{Alice, Charlie}\}$, the Union is $\{\text{Alice, Alice, Bob, Charlie}\}$.

Suddenly, when you send out a marketing email blast, Alice gets it twice. She gets annoyed. She marks you as spam. Your domain reputation tanks. Your CTO yells at you.

All because you treated a Bag like a Set.

Now that we accept that our containers are full of duplicates, we need to learn how to manage them. We need to learn the art of **deduplication**.

## 3.2 Handling Duplicates
You are sitting at a restaurant. You order the steak. The waiter nods, writes it down, and walks away. Two minutes later, the waiter comes back. He looks a little confused. He looks at you and says, "You ordered the steak, right?" You say, "Yes." He writes it down again.

Now, here is the million-dollar question: **is the kitchen going to cook you two steaks?**

If the restaurant is running a smart system (a Set), the kitchen sees the second order, checks the table number, realizes "Table 4 already has a steak in the queue," and ignores the duplicate. If the restaurant is running a dumb system (a Bag), you are about to pay for two dinners.

In data engineering, we spend a shocking amount of our lives dealing with the stuttering waiter. We call this **deduplication**, or "deduping" if you want to sound cool.

### The "At Least Once" Guarantee
Why does data duplicate? Is the computer stupid?

Usually, no. The computer is just paranoid.

Most modern data pipelines (the systems that move data from A to B) operate on a principle called **"At Least Once" Delivery**. When system A sends a message to System B, it waits for a confirmation ("Ack"). If the network hiccups—if a squirrel chews through a fiber optic cable somewhere—System A might not hear the "Ack."

System A panics, "Oh no, they didn't get the data! I better send it again just to be safe."

Meanwhile, System B did get the data the first time. Then it gets it again. Now System B has two copies of the same record.

This is a feature, not a bug. It is safer to have two copies of a bank deposit than zero copies. But it shifts the responsibility to you, the data engineer, to clean up the mess.

### The Problem of "The Grain"
To remove duplicates, you must first answer a philosophical question: **What makes a row unique?**

This brings us to one of the most important concepts of data modeling: **The Grain**.

The Grain is the "atomic unit" of the table. It is the definition of what one row *represents*.

Let's look at that retry scenario again.

| EventID | User | Action | Timestamp |
|:---|:---|:---|:---|
| 500 | Bob | Click | 2023-01-01 10:00:01 |
| 501 | Bob | Click | 2023-01-01 10:00:05 |

Are these duplicates?

- **Scenario A**: Bob clicked the button. The app froze. bob got annoyed and clicked it again 4 seconds later.
    - **Verdict**: These are **distinct events**. Bob physically clicked twice. If we are counting "User Engagements," we count both.
- **Scenario B**: Bob clicked once. The mobile app sent the signal. The Signal got lost. The app retried sending the *same* click event 4 seconds later with a new ID.
    - **Verdict**: These are **duplicates**. In reality, Bob only performed one action. If we are counting "Purchases," we must delete one.

Without understanding the **Grain** (what actually happened in reality), you cannot know if you are looking at a Bag of duplicates or a Set of distinct events.

### The Survivor Logic
Once you identify that you have duplicates (say, two rows for "User 101" with different phone numbers), you have to decide how to fix it. You can't just "delete duplicates"—you have to decide *which one stays*.

This logic is often called **The Survivor**.

Imagine we have a Bag of update requests for a user's profile:

| User | Email | Updated_At |
|:---|:---|:---|
| 101 | bob@aol.com | 09:00 AM |
| 101 | bob@gmail.com | 09:05 AM |
| 101 | bob@yahoo.com | 08:55 AM |

If we want to convert this Bag into a Set of "Current User Emails," we need a rule. Usually, the rule is "Latest Wins."

1. Group the data by the identity (`User 101`).
2. Look at the `Updated_At` timestamps.
3. Keep the row with the Maximum timestamp (`09:05 AM`).
4. Discard the rest.

This operation transforms the Bag back into a Set: `[(101, bob@gmail.com)]`.

### Idempotency: The Holy Grail
There is a fancy mathematical word that you should learn so you can ask for a raise later: **idempotency**.

An operation is **idempotent** if applying it multiple times has the same result as applying it once.

$$
f(x) = f(f(x)) = f(f(f(x)))
$$

- **Not Idempotent**: "Add $10 to Bob's account."
    - Run once: Bob has +$10.
    - Run twice: Bob has +$20. (Bob is happy; the bank is not).
- **Idempotent**: "Set Bob's balance to $100."
    - Run once: Balance is $100.
    - Run twice: Balance is $100.

The goal of handling duplicates is to make your data pipeline idempotent. You want to build a system where, even if the "stuttering waiter" sends the data five times, your final bill still looks correct.

## 3.3 The Distinct Operator
We have a Bag. It is full of duplicates. It is noisy. We want a Set. We want peace and quiet.

In the world of logic and data, there is a specific operator designed to perform this alchemy. It takes a chaotic multiset and transmutes it into a pure Set.

In SQL, this command is literally called `DISTINCT`.

### The Collapse
Mathematically, the Distinct operator is a function that ignores multiplicity. It flattens the topography of your data.

If our Bag is a histogram where "Apple" is a skyscraper 50 stories tall (because it appears 50 times), the Distinct operator is a steamroller that crushes everything down to a single story.

$$
Distinct(\{A, A, B, C, C, C\}) \to \{A, B, C\}
$$

It seems simple, but there is a nuance here that catches people off guard. The Distinct operator works on the **entire tuple** (the whole row) by default.

Remember our chess board? If you have a bag of moves:

1. `(Pawn E4)`
2. `(Pawn, E4)`
3. `(Pawn, D4)`

If you say, "Give me the distinct pieces," the steamroller looks only at column 1:

- `(Pawn)`

If you say, "Give me the distinct moves," the steamroller looks at the combination (Piece, Square):

- `[(Pawn, E4), (Pawn, D4)]`

You must always ask yourself, "Distinct what?"

### The Hidden Cost of Cleanliness
New data engineers love the `DISTINCT` keyword. It feels like a magic "Fix My Bugs" button. You wrote a query, you messed up the joins, and suddenly your report says you have 10 million users instead of 100.

You panic. You add `SELECT DISTINCT`. The number drops back to 100. You wipe the sweat from your brow and push the code.

**Do not do this**.

Using `DISTINCT` is not free. In fact, it is one of the most expensive operations you can ask a database to perform.

Think about how you would do it physically. I hand you a shuffled deck of 1,000 cards and ask you to remove the duplicates. You can't just glance at the pile.  You have two options:

1. **Sorting**: You have to lay every single card out on the floor and sort them by suit and number. Only when two identical cards are sitting right next to each other can you spot the duplicate and throw one away.
    - *Computer Cost*: Sorting is $O(N \log{N})$. It eats CPU. (If you're not familiar with Big O notation, we cover this in a different course).
2. **Hashing**: You get a set of buckets. You pick up a card, calculate which bucket it belongs in, and check if there's already a card in that bucket.
    - *Computer Cost*: This east RAM.

When you type `DISTINCT`, you are forcing the computer to stop, pick up every single row of your data, and shuffle it around until it finds the twins. If you do this on a billion rows, your query will hang, your server will overheat, and your pager will go off during dinner.

### The "Code Smell"
Because of this cost, `DISTINCT` is often considered a "Code Smell" in data engineering.

If you are calculating the "Distinct Users who logged in today," that is a valid use case. You expect duplicates (a user logs in twice), and you want to count the unique humans.

But if you are querying a table of `Users` (which should have a primary key and be unique already) and you find yourself needing `DISTINCT` to remove duplicates, **you have broken the logic somewhere else**.

You shouldn't need to steamroll a parking lot that is already flat. If you do, it means you accidentally dumped a pile of dirt on it earlier in your script.

## Quiz

<quiz>
What is the fundamental difference between a strict Mathematical Set and a SQL Table (a Bag)?
- [ ] Sets can contain NULLs; Bags cannot.
- [x] Sets cannot contain duplicates; Bags can.
- [ ] Bags are infinite; Sets are finite.
- [ ] Sets have a defined order; Bags do not.

</quiz>

<quiz>
In a Bag (Multiset), how is membership defined differently than in a Set?
- [ ] Membership is based on Timestamp.
- [ ] Membership is a Probability (0 to 1).
- [x] Membership is a Count (Multiplicity).
- [ ] Membership is a Boolean (True/False).

</quiz>

<quiz>
Why do most database systems default to the "Bag" model instead of enforcing strict "Set" uniqueness?
- [x] It is computationally expensive to check for uniqueness on every insert.
- [ ] SQL was designed before Set Theory was invented.
- [ ] Developers prefer seeing duplicate data.
- [ ] Hard drives cannot store unique indexes.

</quiz>

<quiz>
Which common distributed system guarantee is a primary cause of "technical duplicates" (retries)?
- [x] A Least Once Delivery
- [ ] Eventual Consistency
- [ ] At Most Once Delivery
- [ ] Exactly Once Delivery

</quiz>

<quiz>
To distinguish between a "True Duplicate" (error) and a "Distinct Event" (reality), you must define what concept?
- [ ] The Index.
- [ ] The Schema.
- [ ] The Foreign Key.
- [x] The Grain.

</quiz>

<quiz>
What does it mean for a data operation to be "idempotent"?
- [ ] It automatically deletes duplicates.
- [ ] It can never fail.
- [ ] It runs faster the second time.
- [x] Applying the operation multiple times yields the same result as applying it once.

</quiz>

<quiz>
When you use the `DISTINCT` operator, what effectively happens to the "topography" of your data?
- [ ] It removes all NULL values.
- [ ] It is sorted into a pyramid.
- [x] It flattens the multiplicity to 1.
- [ ] It expands the set to include missing values.

</quiz>

<quiz>
Why is `SELECT DISTINCT` considered a "Code Smell" when used excessively?
- [ ] It is not standard SQL.
- [ ] It cannot handle string data.
- [ ] It returns results in random order.
- [x] It indicates a failure in upstream logic (e.g., bad joins) and is computationally expensive.

</quiz>

<quiz>
In "Survivor Logic," if you have three rows for User 101, which attribute is most commonly used to pick the winner?
- [ ] The shortest email address.
- [ ] The lowest numeric value.
- [ ] Random selection.
- [x] The timestamp (latest wins).

</quiz>

<quiz>
If Set $A = \{x\}$ and Set $B = \{x\}$, what is the result of a **Bag** Union?
- [ ] $\emptyset$
- [x] $\{x, x\}$
- [ ] $\{2x\}$
- [ ] $\{x\}$

</quiz>

<!-- mkdocs-quiz results -->