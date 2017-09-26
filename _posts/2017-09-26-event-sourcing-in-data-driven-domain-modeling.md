---
layout: post
title:  "Event sourcing in data-driven domain modeling"
date:   2017-09-26 04:32:16 +0300
categories: event sourcing command query response segregation domain modeling database design
image: /assets/article_images/2017-09-26-event-sourcing-in-data-driven-domain-modeling/shebang.jpg
---
A modern approach to domain modeling using hypermedia as a state of application.

-----

With the advent of the **OLTP** systems, changes made to the state of the
application could be stored in a different manner, which would make
analysis over data possible if carried in to **OLAP** systems.
Database sharding, data replication, partial eventual consistency support in
immediate consistent atomic databases, built-in multi-tenancy support features
made **OLTP** solutions easier to integrate into big data applications, without
too much hassle.
Analytical systems care about the current - valid data, furthermore, they also
care about its history in order to produce time-series analytics, which cares
significance in management and execution of business.

## Implementation

Event sourcing methodology changes usage of persistence back-end found in
**OLTP** subpart of the software solution.
The following parts of this section will make a superficial view of the
methodology.

### Traditional approach

Assume that we need to build a software solution for a bank, barely making
transactions between its own accounts for the sake of simplicity.
Every user should have a balance, and they could withdraw/deposit some money.
A withdrawal operation would reflected to user's balance as a negative change,
and a deposit operation would reflected to user's balance as a positive
change.
Henceforth, we could create a table for the users, `users` like so:

| **id** | name | balance | inserted_at |
| -: | :-: | :-: | ----------- |
| 1 | Hayden Panettiere | 24.72 USD | 2017-08-28 00:22:14.960193 |
| 2 | Megan Fox | 25.14 USD | 2017-08-28 00:23:14.123512 |

When *Hayden* wants to withdraw some bucks, we could do that with following
query in *PLpgSQL*:

```
UPDATE "users"
SET "balance" = "balance" - ('20.00 USD')::money
  WHERE "id" = 1 AND "balance" >= ('20.00 USD')::money;
```

That would decrement *Hayden*'s balance by *20.00 USD*.
Similarly, deposit operation could be performed with a simple `UPDATE` query.

However, when it comes to transfer, following we need to perform a simple
transaction.
We would like to transfer some funds from *Hayden* to *Megan*.

```
BEGIN
  DECLARE
    affected_rows int DEFAULT 0;
  BEGIN
    BEGIN TRANSACTION "transfer-VEVJW9WrThixnQr9vmR274LgltOC5EQoh5CYq2";

    SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

    UPDATE "users"
    SET "balance" = "balance" - ('20.00 USD')::money
      WHERE "id" = 1 AND "balance" >= ('20.00 USD')::money;

    affected_rows := SQL%ROWCOUNT; -- Get number of affected rows

    IF affected_rows = 1 THEN
      UPDATE "users"
      SET "balance" = "balance" + ('20.00 USD')::money
        WHERE "id" = 2;

      END TRANSACTION "transfer-VEVJW9WrThixnQr9vmR274LgltOC5EQoh5CYq2";
    ELSE
      ROLLBACK;
    END IF;
  END
END
```

Of course, this is not an elegant solution.
However, it shows a canonical way of doing it correctly.

Finally, we could get the current balance of *Hayden* with:

```
SELECT "balance"
FROM "users"
  WHERE "id" = 1;
```

##### Advantages

- Easy, straightforward implementation.
- Performant query.

##### Disadvantages

- Change history is not stored.
- There would be no rollback action other than transaction-level rollback.
- `READ COMMITTED` or `SERIALIZABLE` transaction isolation levels might
be required in order to achieve true atomicity over concurrency (*AoC*).

### The "Event Sourcing" approach

Traditional approach tries to store results of operations.
In *event sourcing* approach, we will store events (or artifacts) instead of
results.
With a powerful query language, like *PLpgSQL*, we could calculate results of
operations from those events.
A normalization could be performed in order to create required tables.

`users` table:

| **id** | name | inserted_at |
| -: | :-: | --- |
| 1 | Hayden Panettiere | 2017-08-28 00:22:14.960193 |
| 2 | Megan Fox | 2017-08-28 00:23:14.123512 |

`user_balance_changes` table:

| **id** | user_id | delta | inserted_at |
| -: | :-: | :-: | --- |
| 1 | 1 | + 30.00 USD | 2017-08-28 01:22:14.960193 |
| 2 | 2 | + 20.00 USD | 2017-08-28 01:23:14.123512 |
| 2 | 2 | - 5.00 USD | 2017-08-28 01:24:14.123512 |

`user_transfers` table:

| **id** | source_user_id | target_user_id | amount | inserted_at |
| -: | :-: | :-: | :-: | --- |
| 1 | 1 | 2 | + 2.50 USD | 2017-08-28 01:22:14.960193 |

Should start from simple one, getting current balance information of a user.

Again, using `PLpgSQL`, we could calculate balance of *Hayden* using an
aggregate function.

```
WITH (
  SELECT sum("delta")
  FROM "user_balance_changes"
    WHERE "user_id" = 1;
    GROUP BY "user_id"
) "balance_sum_aggregate", (
  SELECT sum("amount")
  FROM "user_transfers"
    WHERE "source_user_id" = 1
    GROUP BY "source_user_id"
) "incoming_transfers_sum_aggregate", (
  SELECT sum("amount")
  FROM "user_transfers"
    WHERE "target_user_id" = 1
    GROUP BY "target_user_id"
) "outgoing_transfers_sum_aggregate"
SELECT "user_balance_changes"."sum" + "incoming_transfers_sum_aggregate"."sum" - "outgoing_transfers_sum_aggregate"."sum";
```

Much more complicated.
Even though it seems like a `SELECT` query, it was an implicit `UNION`
query built with `PLpgSQL`s `WITH` construct.
A query with `UNION` clauses would be semantically equal.
However, performing a money transfer between those two ladies would be
simpler than former approach.

```
INSERT INTO "user_transfers" ("source_user_id", "target_user_id", "amount")
  VALUES (1, 2, '2.00 USD'::money);
```

Remembering that query with declaring a variable, beginning a read committed
transaction and doing sequential updates, this is much more convenient.
*Hayden* can also deposit some money to the bank:

```
INSERT INTO "user_balance_changes" ("user_id", "delta")
  VALUES (1, '2.00 USD'::money);
```

Very elegant, niché solution.

##### Advantages

- More data involved, open breath to analytics.
- Simple insertions.
- Similar to source control, makes available to *cherry pick*.

##### Disadvantages

- Complicated read queries, makes database design read biased.
- Functions involving those queries might need dramatically large number of
test cases.

## Analysis

Under the hood, we are converting our insertion overhead into a read overhead,
making our database design *read oriented*.
Having considered latest database technologies, those reads might sometimes
does not necessarily need to be consistent between different users.
Our example was not one of them, nevertheless, in social media applications,
those types of data is *endemic*.
Distinguishing several properties for your data, you may replicate that data
with an eventual consistent data persistence solution.

![BuyBuddy Platform Architecture](https://github.com/Chatatata/a1-Chatatata/raw/gh-pages/assets/multi-tenant%20arch.png)

*Figure: Data persistence architecture of [BuyBuddy](http://buybuddy.co) designed by myself.*

As you see, one of my architectures was using **OLTP** solutions with
homebrew multi-tenancy, whereas it was using an **OLAP** cluster to
process analytical queries to the master tenant.
That architecture allowed horizontal scaling for web servers, as well as
data persistence layers, and it does not limit further sharding and
replication applications to the distinct nodes.
As it is said, making insertion operations one-hit as possible has an
adverse affect of making the database *read oriented*, using different
scaling methods rescue us from long query times whereas protecting
all available data, which can be cumulated by analytical processing
engine.

## Summary

The examples were indeed in canonical form, yet available to show the
intended logic of the *event sourcing*.
A real life use case would include proper use of database indexes
(B-tree, hash, GiST, GreMAP etc.), vacuuming, perpetual trigger
functions and window functions in order to down-scale query times,
since performing a self join on unindexed column might be
*symbol of the death* for the whole stack.
Also, this is not a magic wand, on data types with high update rates
like real-time systems, such as beacon presence back-ends, the updates
should not be converted to insertions as that may create a redundancy
in data storage.
Considering time-series database solutions like *Riak-TS* could be
a good replacement for *event sourcing*.
