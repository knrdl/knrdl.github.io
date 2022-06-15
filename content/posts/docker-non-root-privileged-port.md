---
title: "Bind non-root process to privileged port inside a Docker container"
date: "2022-06-15"
tags: [docker, linux]
lang: "en"
hackerNewsId: ""
---

The Principle of Least Privilege (PoLP) should also be applied to applications inside a container! Running stuff as root
user is never a good idea. A custom user profile with fewer privileges should be created and used instead.

But such a user cannot bind a privileged port (<1024), e.g. start a webserver on port 80. Exposure via port binding to
the host is still unproblematic, e.g. `docker run -p 80:8080 ...` will bind port 8080 of the container to port 80 on the
host.

But what if two services share a Docker Network to reach each other? Then requests have to be made to
e.g. **http://service2:8080/api** (includes the port 8080). This is a bit ugly because **service1** now needs to contain
runtime information about **service2** (contradicts goal of loose coupling). An endpoint like **http://service2/api** (defaults to port 80) is preferable.

The following Dockerfile achieves that goal:

```dockerfile
FROM python:3-alpine

# allow non privileged user to run server on port 80
RUN apk add --no-cache libcap && \
    setcap 'cap_net_bind_service=+ep' "$(readlink -f "$(which python3)")" && \
    apk del libcap
    
EXPOSE 80/tcp

RUN adduser --home /home/appname --disabled-password --shell /bin/false --uid 1000 appname

# all commands after the USER-command will be executed as user `appname`
USER appname
RUN whoami

# now the app can run on port 80 as non root user
CMD python3 myapp.py
```

The program `setcap` is used to give the executable (e.g. `/usr/bin/python3` or `/bin/myapp`) the necessary capability (permission). It's not needed afterwards, so it might be deleted to keep the image small.

> If you work with <abbr title="Advanced Packaging Tool">APT</abbr> (Ubuntu/Debian images) just use the
> package `libcap2-bin` instead of `libcap`.
> {.info }

