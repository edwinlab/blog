---
layout: post
title: "Partitioning in PostgreSQL"
description: "How to implement partitioning in postgresql to improve huge row of table."
tags: [PostgreSQL, partitioned]
---

## What is table partitioning in PostgreSQL?
1 logical table = n smaller physical tables

## What are the benefits?
improved query performance for big tables*

*works best when results are coming from single partition

## When to consider partitioning?
Big table that cannot be queried efficiently, as the index is enormous as well.

Rule of thumb: table does not fit into memory

## How is data split?
By ranges:
- year > 2010 AND year <= 2012
- created_at >= DATE '2017-08-01' AND created_at < DATE '2017-09-01'

By lists of key values: company_id = 5


## How does it work?
At the heart of it lie 2 mechanisms:
- table inheritance
- table CHECK constraints

## Table inheritance: schema and create constraint index

### Add scheme
```sql
CREATE SCHEMA payments_partitions;
```

### Add table and index
```sql
CREATE TABLE "payments_partitions"."p201708" (
  CHECK (created_at >= '2017-08-01' AND created_at < '2017-09-01')
) INHERITS (payments);
CREATE UNIQUE INDEX  "p201708_id_udx" ON "payments_partitions"."p201708"  ("id");
CREATE  INDEX  "p201708_created_at_idx" ON "payments_partitions"."p201708"  ("created_at");
```

### Add trigger when insert payment
```sql
CREATE OR REPLACE FUNCTION fn_insert() RETURNS TRIGGER AS $$
BEGIN
IF ( NEW.created_at >= DATE '2017-08-01' AND NEW.created_at < DATE '2017-09-01' ) THEN
  INSERT INTO p201708 VALUES (NEW.*);
ELSE
  RAISE EXCEPTION 'Date out of range';
END IF;
  RETURN NULL;
END;
$$
LANGUAGE plpgsql;

CREATE TRIGGER tr_insert BEFORE INSERT ON public.payments
FOR EACH ROW EXECUTE PROCEDURE fn_insert();
```

## Explain

### Select

```sql
EXPLAIN ANALYZE SELECT * FROM payments;
```

#### Explain result
```
Seq Scan on payments  (cost=0.00..3.39 rows=39 width=12222) (actual time=0.006..0.035 rows=43 loops=1)
Planning time: 0.136 ms
Execution time: 0.066 ms
```

#### After partitioned
```sql
EXPLAIN ANALYZE SELECT * FROM payments;
```

#### Explain result
```
Append  (cost=0.00..43.39 rows=43 width=12842) (actual time=0.010..0.055 rows=43 loops=1)
->  Seq Scan on payments  (cost=0.00..3.39 rows=39 width=12222) (actual time=0.010..0.044 rows=43 loops=1)
->  Seq Scan on p201708  (cost=0.00..10.00 rows=1 width=18860) (actual time=0.001..0.001 rows=0 loops=1)
->  Seq Scan on p201709  (cost=0.00..10.00 rows=1 width=18860) (actual time=0.004..0.004 rows=0 loops=1)
->  Seq Scan on p201710  (cost=0.00..10.00 rows=1 width=18860) (actual time=0.002..0.002 rows=0 loops=1)
->  Seq Scan on p201711  (cost=0.00..10.00 rows=1 width=18860) (actual time=0.001..0.001 rows=0 loops=1)
Planning time: 2.520 ms
Execution time: 0.285 ms
```

#### Explain with filter
```sql
EXPLAIN ANALYZE SELECT * FROM "payments"  WHERE "created_at" = '2017-08-01';
```

#### Explain result
```
Append  (cost=0.00..11.63 rows=2 width=15539) (actual time=0.049..0.049 rows=0 loops=1)
->  Seq Scan on payments  (cost=0.00..3.49 rows=1 width=12222) (actual time=0.045..0.045 rows=0 loops=1)
    Filter: (created_at = '2017-08-01 00:00:00+07'::timestamp with time zone)
    Rows Removed by Filter: 43
->  Index Scan using p201708_created_at_idx on p201708  (cost=0.12..8.14 rows=1 width=18860) (actual time=0.002..0.002 rows=0 loops=1)
    Index Cond: (created_at = '2017-08-01 00:00:00+07'::timestamp with time zone)
Planning time: 0.670 ms
Execution time: 0.156 ms
```

#### Explain with date range
```sql
EXPLAIN ANALYZE SELECT "payments".* FROM "payments"  WHERE ("payments"."created_at" BETWEEN '2017-08-01' AND '2017-08-31');
```

#### Explain result
```
Append  (cost=0.00..11.73 rows=40 width=12391) (actual time=0.014..0.082 rows=43 loops=1)
->  Seq Scan on payments  (cost=0.00..3.58 rows=39 width=12222) (actual time=0.013..0.068 rows=43 loops=1)
    Filter: ((created_at >= '2017-08-01 00:00:00+07'::timestamp with time zone) AND (created_at <= '2017-08-31 00:00:00+07'::timestamp with time zone))
->  Index Scan using p201708_created_at_idx on p201708  (cost=0.12..8.14 rows=1 width=18860) (actual time=0.002..0.002 rows=0 loops=1)
    Index Cond: ((created_at >= '2017-08-01 00:00:00+07'::timestamp with time zone) AND (created_at <= '2017-08-31 00:00:00+07'::timestamp with time zone))
Planning time: 0.764 ms
Execution time: 0.172 ms
```

### Insert

```sql
EXPLAIN ANALYZE INSERT INTO payments ("created_at") VALUES ('2017-08-18 14:09:03.763977') RETURNING id;
```

#### Explain result
```
Insert on payments  (cost=0.00..0.01 rows=1 width=1464) (actual time=0.711..0.711 rows=0 loops=1)
->  Result  (cost=0.00..0.01 rows=1 width=1464) (actual time=0.023..0.023 rows=1 loops=1)
Planning time: 0.167 ms
Trigger tr_insert: time=0.660 calls=1
Execution time: 0.869 ms
```

### Delete

```sql
EXPLAIN ANALYZE DELETE FROM payments WHERE id = 69090;
```

#### Explain result
```
Delete on payments  (cost=0.00..11.63 rows=2 width=6) (actual time=0.061..0.061 rows=0 loops=1)
Delete on payments
Delete on p201708
->  Seq Scan on payments  (cost=0.00..3.49 rows=1 width=6) (actual time=0.028..0.028 rows=0 loops=1)
    Filter: (id = 69090)
    Rows Removed by Filter: 43
->  Index Scan using p201708_id_udx on p201708  (cost=0.12..8.14 rows=1 width=6) (actual time=0.004..0.005 rows=1 loops=1)
    Index Cond: (id = 69090)
Planning time: 0.850 ms
Execution time: 0.123 ms
```

## Resource
- [Partitioning](https://www.postgresql.org/docs/9.3/static/ddl-partitioning.html)