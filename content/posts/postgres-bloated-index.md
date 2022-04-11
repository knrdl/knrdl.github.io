---
title: "Cleanup bloated postgres index"
date: "2022-04-10"
tags: ['postgres', 'sql', 'devops']
lang: "en"
---

# 1. find db byte sizes

```sql
SELECT datname, pg_size_pretty(pg_database_size(datname))
FROM pg_database
ORDER by pg_database_size(datname) DESC;
```

# 2. find tables + indices sizes

```sql
select table_name, pg_size_pretty(pg_total_relation_size(quote_ident(table_name)))
from information_schema.tables
where table_schema = 'public'
order by pg_total_relation_size(quote_ident(table_name)) desc;
```

# 3. recreate index

```sql
REINDEX TABLE hungry;
```

# Conclusion

index shrank from ~12GiB to ~800MiB
