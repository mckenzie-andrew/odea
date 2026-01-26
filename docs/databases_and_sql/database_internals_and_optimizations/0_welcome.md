Welcome to the Database Internals module. If the SQL module taught you how to drive the car, this module is where we pop the hood and take the engine apart.

Many developers treat the database as a "black box"—you put a query in, and data comes out. But relying on magic is dangerous in engineering. In this course, we are going to shatter the illusion of the Cloud. We will learn that data obeys the laws of physics, that disk I/O is the enemy, and that every query you write has a tangible cost.

We are going to trace the lifecycle of a query from the moment it leaves your fingers to the moment it hits the spinning magnet or microchip. By the end of this module, you will no longer just be writing code that works; you will be writing code that scales.

## Prerequisites
This is an advanced module. Before we can discuss how the database optimizes a Join or ensures ACID compliance, you must be fluent in asking the database for data. This module relies entirely on the skills built in:
- Structured Query Language (SQL)

We will be looking at `EXPLAIN` plans and discussing query tuning. If you are still struggling with the difference between a `LEFT JOIN` and an `INNER JOIN`, or if the syntax for a `GROUP BY` feels shaky, please return to the SQL module first. We need your mental RAM free to focus on architecture, not syntax.

## The Practice
Theory is useless without observation. To understand performance, you have to measure it.

### Quizzes
Each chapter concludes with a comprehensive 10-question quiz. These are your diagnostic tools. They focus heavily on the "Why" and the "How"—asking you to predict behaviors regarding Indexing, Isolation Levels, and I/O costs. Take them as often as you need to cement the concepts.

### Guided Labs
This course contains a guided lab setup which can be found in the `/modules/database_internals_and_optimizations` folder of the companion [Github repo](https://github.com/OpenConduit/labs).

These labs are not intended for you to understand the code being used in the code cells. Just read through the text cells and review the visuals. I have provided comments in the code to help with those who can read Python, but I do not expect you to know it.

Also, these labs are extremely small scale. While all the material here is true at any scale, the performance improvements and differences we see may seem insignificant. They are not! Take them as percentages of a whole, and in your mind think about a 1-5 TB dataset. Imagine that percentage difference and what that would mean.