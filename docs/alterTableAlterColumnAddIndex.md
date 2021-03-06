---
id: alterTableAlterColumnAddIndex
title: ALTER TABLE COLUMN ADD INDEX
sidebar_label: ALTER TABLE COLUMN ADD INDEX
---

## Synopsis

Adds index to existing column

## Syntax

![alt-text](assets/alter-table.svg)
![alt-text](assets/alter-table-add-index.svg)

## Description
Adds new index to column of type `symbol`. Adding index is an atomic, non-blocking and non-waiting operation. Once complete optimiser will start using new index for SQL executions.

> For more information about indexes please refer to the **[INDEX section](sqlINDEX.md)**.

## Example
```sql
ALTER TABLE trades ALTER COLUMN symbol ADD INDEX
```

