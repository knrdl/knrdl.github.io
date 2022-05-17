---
title: "Postgres: Create enum if not exists"
date: "2022-05-17"
tags: [postgres, sql]
lang: "en"
hackerNewsId: ""
---

In postgres there is nothing like `create table if not exists` for enums. Workaround:

```sql
DO
$$
    BEGIN
        CREATE TYPE request_type AS ENUM ('request_type1', 'request_type2');
    EXCEPTION
        WHEN duplicate_object THEN null;
    END
$$;
```
