---
id: sql
title: SQL implementation
sidebar_label: SQL implementation
---

## Introduction
QuestDB attempts to implement standard ANSI SQL. We also attempt to be PostgresSQL compatible, although some of it is work in progress. QuestDB implements the following clauses in this execution order:

1. FROM
2. ON
3. JOIN
4. WHERE
5. LATEST BY
6. GROUP BY (implicit)
7. WITH
8. HAVING (implicit)
9. SELECT
10. DISTINCT
11. ORDER BY
12. LIMIT

We also implemented sub-queries. They can be used anywhere table name is used. Our sub-query implementation adds 
virtually zero execution cost to SQL. We encourage their use to add flavours of functional language to old-school SQL. 

## Important differences from standard SQL

### `SELECT * FROM` Optionality

In QuestDB `select * from` is optional.

```sql
select * from tab
```

and

```sql
tab
```

achieves the same effect.

While `select * from` makes SQL look more complete on a single time, there are examples where its optionality makes things a lot easier to read. See examples in `GROUP BY` section.

### Lack of `GROUP BY ... HAVING`

We do not support explicit `GROUP BY` clause. Instead QuestDB optimiser derives group-by implementation from `SELECT` clause. For example:

```sql
select a, b, c, d, sum(e) from tab group by a, b, c, d
```

We find enumerating subset of `SELECT` columns in `GROUP BY` clause redundant and therefore unnecessary. The same SQL in QuestDB SQL-dialect will look like:

```sql
select a, b, c, d, sum(e) from tab
```

Lets see how we replace
```sql
select a, b, c, d, sum(e) 
from tab 
group by a, b, c, d 
having sum(e) > 100
```
Here `select * from` optionality and featherweight sub-queries come to the rescue:

```sql
(select a, b, c, d, sum(e) sum from tab) where sum > 100
```

Here we avoided repetitive aggregation expressions without extra furniture syntax.

## SQL extensions

We have extended SQL language to support our data storage model and simplify semantics of time series queries.

- `LATEST BY` (find out more **[here](startCRUD.md)**)
- `SAMPLE BY`

Please follow the links for detailed description of these clauses.