---
title: "PostgreSQL: How UPDATE queries work"
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

## 1. Small Updates are Often Expensive

PostgreSQL does not update rows in-place.
This can lead to *write amplification*.
This means that a small update can lead to a large amount of data being written to disk.
Consider this example table that stores user profiles in a social network:

```sql
CREATE TABLE IF NOT EXISTS user_profile (id uuid, last_active timestamptz, description text);
```

The size of a single row (excluding metadata) is calculated as follows:
* `id` is 16 bytes
* `last_active` is 8 bytes
* `description` has variable length, so its size could be hundreds of bytes [^1].

If the `description` column takes 200 bytes on average, the write amplification factor when updating `last_active` is 28 [^2].
For each byte that is logically replaced, 28 bytes are written to disk.
This is because PostgreSQL writes the entire row to a new location on disk.
Considering row metadata, the real write amplification factor is even higher.

Write amplification can lead to increased disk I/O and wasted storage space.
There are other sources of write amplification:
* PostgreSQL uses a write-ahead log (WAL) is used to ensure durability.
* Indexes may have to be updated (even if the updated column is not indexed).
* If the row grows too large, some of its data may need to be moved to a separate physical table.

If disk I/O is approaching limits, it may be worth to investigate update queries.

---

[^1] If a row becomes larger than 2 kB, PostgreSQL will apply a storage technique known as [TOAST](https://www.postgresql.org/docs/current/storage-toast.html).
This technique moves some data to separate physical tables.
TOASTed data .
[^2] 16 bytes for `id` + 8 bytes for `last_active` + 200 bytes for `description` = 224 bytes.
