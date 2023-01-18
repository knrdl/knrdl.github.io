---
title: "Miniflux Housekeeping"
date: "2023-01-18"
tags: [miniflux, sql]
lang: "en"
---

[Miniflux](https://miniflux.app/) is a feed reader. As adding feeds is easy, some bloat will appear.

## Article Age

List the top 10 feeds with the oldest articles. No updates, no need to keep a subscription.

```sql
select f.feed_url, f.title,
(select max(published_at) from public.entries where feed_id = f.id) last_published_at
from feeds f where f.user_id = (select id from users where username = 'knrdl')
order by last_published_at limit 10;
```

## Article Count

List the top 10 feeds with the fewest articles. An article count of 0 either means that the feed never produced articles since you subscribed to it. But it can also mean that the articles have been removed from the source after you read them (Miniflux marks old read articles as removed in the DB and deletes them if they disappear from the source). 

```sql
select f.feed_url, f.title,
(select count(*) from public.entries where feed_id = f.id) article_count
from feeds f where f.user_id = (select id from users where username = 'knrdl')
order by article_count limit 10;
```
