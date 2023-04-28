---
title: "Postgres: Fast row count estimates"
date: "2023-04-28"
tags: [postgres, sql, python]
lang: "en"
---

Counting rows in postgresql is as easy as `select count(1) from mytable`. This is a precise count. But it becomes slow on an increasing number of records, especially when a sequential scan of all table records is required, see `explain select count(1) from mytable`. But in the end the user often doesn't care if there are about 125_000_000 or exactly 124_756_849 results. Therefore, a fast row count estimate might be more desirable than a slow precise count.

Postgres runs internal statistics for the query planner to produce performant decisions. These statistics also include estimates of row counts. They are updated regularly by autovacuum or manually by `analyze mytable`. The later one is only necessary to get better results on very recent insert/delete batches.

This snippet uses the postgres statistics to estimate the row count:

```python3
import psycopg2
conn  = psycopg2.connect(os.getenv('DB_DSN'))
cur = conn.cursor()
cur.execute('explain (format json) select * from mytable;')
res = cur.fetchone()
row_count = res[0][0]['Plan']['Plan Rows']
```