# PostgreSQL Performance Lab

## Project Overview

This project demonstrates practical PostgreSQL database performance and transaction management techniques.

The lab covers:
- Creating and populating a large orders table
- Generating 2,000,000 records
- Measuring query performance with EXPLAIN ANALYZE
- Comparing sequential scan and indexed query plans
- Creating a targeted partial index
- Testing transaction isolation levels
- Configuring PgBouncer using transaction pooling

## 1. Large Dataset

Created the orders table with the following columns:

- id - BIGINT identity primary key
- customer_id - customer identifier
- amount - order amount
- status - order status
- created_at - order creation timestamp

The table was populated with 2,000,000 records.

The table was analyzed using:

```sql
ANALYZE orders;
```
---

## 2. Initial Query Performance

The following query was analyzed before creating the index:

EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, SUM(amount)
FROM orders
WHERE status = 'pending'
  AND created_at > now() - interval '30 days'
GROUP BY customer_id
ORDER BY SUM(amount) DESC
LIMIT 10;

Initial execution plan: Parallel Seq Scan

Observed execution time: approximately 286 ms.

Observed buffers:
- shared hit = 173
- shared read = 16571

This result was used as the baseline for comparison.

## 3. Targeted Partial Index

The following partial index was created:

CREATE INDEX idx_pending_recent
ON orders (created_at DESC, customer_id)
WHERE status = 'pending';

After creating the index, PostgreSQL used:

- Bitmap Index Scan
- Bitmap Heap Scan
- idx_pending_recent

Observed execution time: 445.804 ms.

The index changed the execution plan from a sequential scan to a bitmap index and bitmap heap scan. In this particular test run, the indexed query was slower than the baseline, demonstrating that an index does not automatically guarantee faster execution.

## 4. Transaction Isolation Testing

### READ COMMITTED

Session 1 initially read:

8888.00

Session 2 updated the amount to 9999 and committed.

Session 1 queried the row again and observed:

9999.00

This demonstrated that under READ COMMITTED, a new statement can see data committed after the previous statement.

### REPEATABLE READ

Session 1 started:

BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

Initial value:

9999.00

Session 2 changed the value to 7777.00 and committed.

Session 1 continued to see:

9999.00

After Session 1 committed and started a new transaction, it saw:

7777.00

This demonstrated that REPEATABLE READ maintains a consistent snapshot during the transaction.

## 5. PgBouncer Configuration

PgBouncer was installed in WSL Ubuntu and configured using transaction pooling.

Configuration:

[databases]
bootcamp = host=172.29.80.1 port=5432 dbname=bootcamp

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
listen_port = 6432
listen_addr = 127.0.0.1
auth_type = plain
auth_file = /etc/pgbouncer/userlist.txt

PgBouncer was tested using:

psql -h 127.0.0.1 -p 6432 -U postgres -d bootcamp

The following query was executed:

SELECT current_database(), inet_server_port();

Result:

current_database | inet_server_port
-----------------+------------------
bootcamp         | 5432

This confirmed that the client connected through PgBouncer on port 6432 and PgBouncer successfully forwarded the connection to PostgreSQL on port 5432.

## 6. Repository Files

The SQL file contains the commands used for:

- Creating the orders table
- Generating 2,000,000 records
- Running EXPLAIN ANALYZE
- Creating the targeted partial index
- Testing transaction isolation
- Documenting PgBouncer configuration

The README documents the experiment results, execution plans, performance observations, transaction isolation behavior, and PgBouncer configuration.

## 7. Conclusion

This lab provided practical hands-on experience with PostgreSQL database performance, indexing, transaction isolation, and connection pooling.

The experiments demonstrate how EXPLAIN ANALYZE can be used to inspect query execution plans, how indexes can change PostgreSQL execution strategies, how transaction isolation affects data visibility, and how PgBouncer can provide transaction-level connection pooling.

