---
title: "How to use IBM DB2 with async python"
date: "2024-12-06"
tags: [databases, python, fastapi]
lang: "en"
---

# Async Python

Async frameworks are nowadays the standard for web development with python. [FastAPI](https://fastapi.tiangolo.com/) didn't invent it, but is currently the most successful and popular web framework offering it. With the increasing popularity of async, the need for async libraries grew over time. Python's standard library is sync only for historic reasons. So [async packages filled the gap](https://github.com/aio-libs) and sometimes replaced the old sync libraries, e.g.:

* DNS: [std socket](https://docs.python.org/3/library/socket.html) **→** [aiodns](https://pypi.org/project/aiodns/)
* Docker Client: [docker](https://pypi.org/project/docker/) **→** [aiodocker](https://pypi.org/project/aiodocker/)
* File IO: [std open](https://docs.python.org/3/library/functions.html#open) **→** [aiofiles](https://pypi.org/project/aiofiles/)
* HTTP Client: [requests](https://pypi.org/project/requests/) **→** [aiohttp](https://pypi.org/project/aiohttp/), [httpx](https://pypi.org/project/httpx/)
* Postgres: [psycopg2](https://pypi.org/project/psycopg2/) **→** [psycopg3](https://pypi.org/project/psycopg/), [asyncpg](https://pypi.org/project/asyncpg/)
* SMTP Client: [std smtplib](https://docs.python.org/3/library/smtplib.html) **→** [aiosmtplib](https://pypi.org/project/aiosmtplib/)
* Webserver: [flask 1](https://pypi.org/project/Flask/) **→** [flask 2](https://pypi.org/project/Flask/), [fastapi](https://pypi.org/project/fastapi/)

# IBM DB2

However, some old-fashioned libraries haven't adapted yet and maybe never will, like the clunky [ibm-db](https://pypi.org/project/ibm-db/) package to connect to an IBM DB2 database. The package consists of a [C driver](https://github.com/ibmdb/python-ibmdb/blob/master/ibm_db.c) and [python interface](https://github.com/ibmdb/python-ibmdb/blob/master/ibm_db_dbi.py) (IMHO, the latter looks like it was poorly written by absolute python beginners together with naysayers to the idea that python code could be pythonic at all).

So the problem is to run the sync lib [*ibm-db*](https://pypi.org/project/ibm-db/) in an async context, like a FastAPI webserver. The standard approach is to use `asyncio.to_thread()` to delegate the sync tasks (db access operations) into additional threads to free the event loop on the main thread. As *ibm-db* is not subject to the [GIL](https://wiki.python.org/moin/GlobalInterpreterLock) there should be no disadvantage except the overhead of spawning threads. But *ibm-db* is also not thread-safe. Multiple threads interact with a DB2 instance result in race conditions, connection abortions and incomplete result set responses. So *ibm-db* cannot be used in async nor multithreaded environments. So the workaround is to run *ibm-db* in (at most) one dedicated thread, but never in the main thread to keep the event loop free. This can be achieved with Python's [`ThreadPoolExecutor`](https://docs.python.org/3/library/concurrent.futures.html#threadpoolexecutor) and the constraint `max_workers=1`. See code below.


## Old code (blocks the async loop)

```python
import ibm_db_dbi as db2

def run_query():
    conn: db2.Connection = db2.pconnect(dsn='...')
    with conn.cursor() as cursor:
        cursor.execute('select 1+1 from SYSIBM.SYSDUMMY1')
        result = cursor.fetchone()
    conn.close()
    return result

print(run_query())  # blocks event loop
print(run_query())  # blocks event loop
print(run_query())  # blocks event loop
```

## New code (frees the async loop)

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

import ibm_db_dbi as db2

_executor = ThreadPoolExecutor(max_workers=1, thread_name_prefix='db2')

async def run_query():  # can be invoked concurrently
    def do():
        conn: db2.Connection = db2.pconnect(dsn='...')
        with conn.cursor() as cursor:
            cursor.execute('select 1+1 from SYSIBM.SYSDUMMY1')
            result = cursor.fetchone()
        conn.close()
        return result

    return await asyncio.get_event_loop().run_in_executor(_executor, do)

print(await run_query())  # does not block the event loop
print(await run_query())  # does not block the event loop
print(await run_query())  # does not block the event loop
```