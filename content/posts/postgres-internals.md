---
title: "PostgreSQL Internals: 3 Things to Know About UPDATE Statements"
author: "Patrick Petrovic"
date: 2023-12-27T12:15:00+01:00
draft: false
---

I recently finished reading [PostgreSQL 14 Internals](https://postgrespro.com/community/books/internals) by Egor Rogov.
It is one of the great books that teach applied computer science without watering down the difficult parts.

One particularly interesting topic is PostgreSQL's execution of `UPDATE` statements.
Updates in relational databases are inherently complex.
They can be affected by conflicts, anomalies, and deadlocks.
However, some behavior is specific to PostgreSQL. This can lead to surprises.

## 1. Write Amplification

PostgreSQL generally does not update rows in-place.
This is a design decision in PostgreSQL's implementation of multi-version concurrency control ([MVCC](https://www.postgresql.org/docs/7.1/mvcc.html)).
The design can lead to *write amplification*: a small update can lead to a larger amount of data being written to disk.

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
* `description` has a variable length, which could be hundreds of bytes [^1].

If the `description` column takes 200 bytes on average, the write amplification factor when updating `last_active` is approximately 28.
For each logical byte that needs to be updated, 28 bytes are written to disk [^2].
This is because PostgreSQL writes the whole updated row to a different location on disk.
Considering row metadata, the real write amplification factor is even higher.

Write amplification can lead to increased disk I/O and wasted storage space.
There are more sources of write amplification:
* Indexes may have to be updated. This can be the case even if the updated column is not indexed [^5].
* PostgreSQL writes all changes to a write-ahead log (WAL) to ensure durability [^6].

If disk I/O is approaching limits, it may be worth investigating row sizes, update statements, and unused indexes [^3].

## 2. Lost Updates

The SQL standard mandates that lost updates be prevented at any isolation level [^8].
Nonetheless, it is trivial to simulate a lost update in PostgreSQL.
Consider an additional `followers_count` column and two transactions (Read Committed) that perform the following steps:

1. `BEGIN`
2. Read the row (e.g., `SELECT followers_count AS curr FROM user_profile WHERE id = '...'`).
3. Compute a new value *in application logic* (e.g., `curr + 1`).
4. Write the new value to the database (e.g., `UPDATE user_profile SET followers_count = <new value> WHERE id = '...'`).
5. `COMMIT`

In step 2, two concurrent transactions may read the original `followers_count` value.
This causes a lost update because the count is effectively only incremented once.

This behavior may well be interpreted as a violation of the SQL standard.
However, the application logic which calculates the new value is not considered part of the transaction.
The database merely stores the new value as requested.
Still, the behavior is clearly an anomaly: it cannot occur if the transactions run sequentially [^9].

When using a higher isolation level, PostgreSQL fully prevents lost updates.
Consider two concurrent transactions executing the following statements using the Repeatable Read isolation level:

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
-- => ERROR:  could not serialize access due to concurrent update
```

The application can handle this error by retrying the transaction.
A simpler solution using the weaker Read Committed level is to let the database compute the new value:

```sql
UPDATE user_profile
SET followers_count = followers_count + 1
WHERE id = '...';
```

This statement prevents lost updates at the Read Committed isolation level [^10].
The second transaction waits for the first transaction to abort or commit, and then proceeds with execution.
Therefore, no error is raised, and the overhead of retrying transactions can be avoided [^4].

## 3. Deadlocks

Deadlocks are another source of confusion in PostgreSQL.
In theory, deadlocks are often explained using the [four required conditions](https://en.wikipedia.org/wiki/Deadlock): mutual exclusion, hold and wait, no preemption, and circular wait.
This would be the case if
* Transaction A updates Row 1, then wants to update Row 2.
* Transaction B updates Row 2, then wants to update Row 1.

Until one of the transactions aborts, there is a circular dependency and hence a deadlock.
The textbook scenario requires two update statements in each transaction, but in reverse order.
Surprisingly, deadlocks can also occur with two transactions that each contain only a single update statement.
The reason is that row locks are acquired in a non-deterministic order.

Consider the following transactions:
    
```sql
-- Session 1
UPDATE user_profile
SET followers_count = 0
WHERE followers_count IS NULL;

-- Session 2
UPDATE user_profile
SET last_active = NOW()
WHERE id = ANY (/* list of ids */);
```

The first transaction runs a sequential scan and updates all rows that match the `WHERE` clause.
Affected rows are locked in the order of the sequential scan.
The second transaction can use the index on `id` to find the rows to update.
An index scan might access affected rows in a different order.
Hence, it is possible that a circular dependency between locks arises.

This particular scenario is most likely to occur when updating a large number of rows in a single transaction.
Updating rows in smaller batches can help to avoid deadlocks.
Batching can also fix performance issues caused by holding locks for an unnecessarily long time.
If performance is not an issue, it may be enough to retry transactions that have been aborted because of a deadlock.

## Conclusion

Many engineers have a good theoretical understanding of isolation levels, anomalies, and deadlocks.
However, PostgreSQL—like any other database—is a specific implementation with its own quirks.

*PostgreSQL 14 Internals* offers many more insights beyond the execution of update statements.
The book is a valuable resource for anyone working with PostgreSQL.
It covers data storage, query execution, caching, and indexing in great detail.
I recommend it to anyone who uses PostgreSQL in production.

<hr>

→ *Discuss this post on [Hacker News](https://news.ycombinator.com/item?id=38781442).*

[^1]: If a row becomes larger than 2 kB (default setting), PostgreSQL will apply a storage technique known as [TOAST](https://www.postgresql.org/docs/current/storage-toast.html).
This technique moves partial data to separate physical tables.
TOASTed data is usually not rewritten unless it has changed.
However, small rows that eventually grow too large can cause additional write amplification due to TOASTing.

[^2]: To be precise, PostgreSQL uses a page size of 8 kB, and only entire pages are written to disk.
However, PostgreSQL writes [row-level data](https://www.postgresql.org/docs/15/runtime-config-wal.html#GUC-FULL-PAGE-WRITES) to the WAL.

[^3]: Debugging write amplification is tricky.
Uber even [migrated from PostgreSQL to MySQL](https://www.uber.com/en-DE/blog/postgres-to-mysql-migration/) in 2016, citing write amplification as a major reason.

[^4]: Nonetheless, updating the same row in multiple concurrent transactions can lead to performance degradation.
As of PostgreSQL 14, there is no efficient queueing mechanism for such transactions. See section 13.4 (Wait Queues) in _PostgreSQL 14 Internals_.

[^5]: This is because indexes point to the database page that contains the row, and the updated row can be written to a different page.
PostgreSQL has a mechanism called [HOT updates](https://www.postgresql.org/docs/current/storage-hot.html) to avoid this.
This mechanism is also explained in _PostgreSQL 14 Internals_.

[^6]: As explained in the [documentation](https://www.postgresql.org/docs/current/wal-intro.html), the WAL reduces the amount of disk writes in practice.
The reason is that the WAL makes it unnecessary to flush data pages to disk after each transaction commit.

[^8]: [SQL-92](https://www.contrib.andrew.cmu.edu/~shadow/sql/sql1992.txt):
"The four isolation levels guarantee that [...] no updates will be lost."

[^9]: The same issue can occur without using transactions.
The first time I fixed a similar issue was at [Jodel](https://jodel.com/), in MongoDB.
Post vote counts were incremented in application logic instead of using MongoDB's `$inc` operator.
This caused about 1 in 1000 votes to be lost in posts with high activity.

[^10]: As readers on [Hacker News](https://news.ycombinator.com/item?id=38781442) have pointed out, a more widely applicable solution is [SELECT FOR UPDATE](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE).
This clause lets the application safely perform arbitrary updates.
However, throughput can suffer because the row must be locked during at least one round trip between database and application.
