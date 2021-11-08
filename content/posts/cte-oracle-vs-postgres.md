---
title: "CTE Oracle vs Postgres"
date: 2018-09-15T15:37:17+03:00
draft: false
tags: ["SQL", "Oracle", "Postgres"]
---
UPDATE(2021-11-08): Described here has changed in [Postgres 12](https://www.postgresql.org/docs/12/queries-with.html) (more recent versions have even better documentation on this).

Major part of my career to the moment passed working with databases and mostly with Oracle and Postgres. Several years ago I was really surprised by the difference in handling CTE (common table expressions) in these databases. From that time I spread this knowledge and recreated a sample for it several times. The last one was this week and I decided to store it somewhere - this mostly dead blog seems like a good place ;-) 
So here is a really simple case of using CTE on Oracle. 

```
create table test_table(
  id int primary key,
  foo varchar2(10)
);

insert into test_table(id, foo)
select rownum, 'val'||rownum from dual
   connect by rownum<=100000;
   
explain plan for select * from (select * from test_table) sel where id=42;
select * from table(dbms_xplan.display);
--|   0 | SELECT STATEMENT            |               |     1 |    20 |     1   (0)| 00:00:01 |
--|   1 |  TABLE ACCESS BY INDEX ROWID| TEST_TABLE    |     1 |    20 |     1   (0)| 00:00:01 |
--|*  2 |   INDEX UNIQUE SCAN         | SYS_C00328137 |     1 |       |     1   (0)| 00:00:01 |

explain plan for with sel as (select * from test_table) select * from sel where id=42;
select * from table(dbms_xplan.display);
--|   0 | SELECT STATEMENT            |               |     1 |    20 |     1   (0)| 00:00:01 |
--|   1 |  TABLE ACCESS BY INDEX ROWID| TEST_TABLE    |     1 |    20 |     1   (0)| 00:00:01 |
--|*  2 |   INDEX UNIQUE SCAN         | SYS_C00328137 |     1 |       |     1   (0)| 00:00:01 |
```

You can see that from the resulting execution plan there is no difference between CTE and sub-select. 
But here is similar sample on Postgres:

```
create table test_table(
  id serial primary key,
  foo text
);

insert into test_table(foo)
select 'val'||generate_series from generate_series(1, 100000);

explain analyze select * from (select * from test_table) as sel where id=42;
-- Index Scan using test_table_pkey on test_table  (cost=0.29..8.31 rows=1 width=12) (actual time=0.037..0.038 rows=1 loops=1)
-- Index Cond: (id = 42)
-- Planning time: 0.053 ms
-- Execution time: 0.048 ms


explain analyze with sel as (select * from test_table) select * from sel where id=42;
-- CTE Scan on sel  (cost=1541.00..3791.00 rows=500 width=36) (actual time=0.025..32.644 rows=1 loops=1)
-- Filter: (id = 42)
-- Rows Removed by Filter: 99999
-- CTE sel
-- ->  Seq Scan on test_table  (cost=0.00..1541.00 rows=100000 width=12) (actual time=0.016..5.859 rows=100000 loops=1)
-- Planning time: 0.069 ms
-- Execution time: 35.305 ms
```

Difference is dramatic. [Quoting Tom Lane](https://www.postgresql.org/message-id/24672.1319561025%40sss.pgh.pa.us) (real Postgres expert) "CTEs act as optimization fences. This is a feature, not a bug. Use them when you want to isolate the evaluation of a subquery." But a case when CTE should be used in Postgres is a theme for another post (maybe, so you here in another 4 or 5 years :-))
