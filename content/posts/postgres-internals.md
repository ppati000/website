---
title: "PostgreSQL: Things I learned about UPDATE Queries"
author: "Patrick Petrovic"
date: 2023-12-26T08:30:21+02:00
draft: true
---

I recently finished reading [PostgreSQL 14 Internals](https://postgrespro.com/community/books/internals) by Egor Rogov.
This book might be the best PostgreSQL resource besides the official documentation.
It is one of the rare books that manage to connect computer science theory with a real-world implementation.

One particularly relevant topic is PostgreSQL's implementation of `UPDATE` queries.
Updates in a relational database are naturally complex.
The database needs to find, read, and write rows.
These operations cannot break ACID guarantees.

These constraints led to some interesting and sometimes confusing behavior in PostgreSQL.

## 1. Small Updates can be Expensive

PostgreSQL does not update rows in-place due to its implementation of multi-version concurrency control ([MVCC](https://www.postgresql.org/docs/7.1/mvcc.html)).
This can lead to *write amplification*: a small update can lead to a large amount of data being written to disk.
Consider a table that stores user profiles in a social networking service:

```sql
CREATE TABLE IF NOT EXISTS user_profile (
    id UUID,
    last_active TIMESTAMPTZ,
    description TEXT
);
```

Each column has a different size:
* `id` takes 16 bytes
* `last_active` takes 8 bytes
* `description` has a variable length, which could be tens to hundreds of bytes[^1].

If the `description` column takes 200 bytes on average, the write amplification factor when updating `last_active` is 28.
For each logical byte that needs to be updated, 28 bytes are written to disk[^2].
This is because PostgreSQL writes the whole updated row to a different location on disk.
Considering row metadata, the real write amplification factor is even higher.

Write amplification can lead to increased disk I/O and wasted storage space.
There are more sources of write amplification:
* PostgreSQL uses a write-ahead log (WAL) for durability.
* Indexes may have to be updated, even if the updated column is not indexed.
* If the row grows too large, some of its data may need to be moved to a separate physical table (TOAST).

If disk I/O is approaching limits, it may be worth to investigate row sizes, update queries, and unused indexes [^3].

## 2. Lost Updates are Confusing

The SQL standard states that lost updates must not occur at any isolation level.
On the other hand, it seems trivial to create a lost update in PostgreSQL.
The following assumes the default `READ COMMITTED` isolation level.

1. Begin a transaction.
2. Read the current value of a row (e.g., `SELECT followers_count FROM user_profile WHERE id = '...'`).
3. Compute a new value in application logic (e.g., `followers_count + 1`).
4. Write the new value to the database (e.g., `UPDATE user_profile SET followers_count = <new value> WHERE id = '...'`).
5. Commit the transaction.

In step 2, two concurrent transactions may read the old value, which does not include the update from the other transaction.
This effectively causes a lost update, but arguably not a violation of the SQL standard.
The application logic which calculates the new value is not part of the database transaction.
The database does not know how the new value was calculated. It merely stores the new value.
Note that this anomaly would not occur if the transactions run sequentially.

In higher isolation levels, PostgreSQL prevents lots updates entirely.
Consider the following commands in two separate sessions using the REPEATABLE READ isolation level:

```sql
BEGIN;
SELECT followers_count FROM user_profile WHERE id = '...';
-- Returns 300. Compute new value in application: 300 + 1 = 301
UPDATE user_profile SET followers_count = 301 WHERE id = '...';
```

Now, only one of the transactions can commit.

```sql
-- Session 1
COMMIT;
-- => COMMIT

-- Session 2
COMMIT;
-- => ERROR:  could not serialize access due to concurrent update
```

The application can handle this error by retrying the transaction.
However, a simpler solution is to let the database compute the new value:

```sql
UPDATE user_profile
SET followers_count = followers_count + 1
WHERE id = '...';
```

This statement prevents lost updates at the `READ COMMITTED` isolation level.
The second update waits for the first transaction to abort or commit, and then proceeds execution.
Therefore, no error is raised, and the overhead of retrying transactions can be avoided [^4].

[^1]: If a row becomes larger than 2 kB (default setting), PostgreSQL will apply a storage technique known as [TOAST](https://www.postgresql.org/docs/current/storage-toast.html).
This technique moves partial data to separate physical tables.
TOASTed data is usually not rewritten unless it has changed.

[^2]: 16 bytes for `id` + 8 bytes for `last_active` + 200 bytes for `description` = 224 bytes, vs 8 bytes that were logically updated.

[^3]: Uber somewhat famously [migrated from PostgreSQL to MySQL](https://www.uber.com/en-DE/blog/postgres-to-mysql-migration/) in 2016.
According to the blog post, this decision was partly made due to write amplification.

[^4]: Nonetheless, updating the same row in multiple concurrent transactions can lead to performance degradation.
As of PostgreSQL 14, there is no efficient queueing mechanism for such queries. See section 13.4 (Wait Queues) in PostgreSQL 14 Internals.
