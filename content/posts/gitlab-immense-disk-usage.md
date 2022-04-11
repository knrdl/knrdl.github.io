---
title: "Fix gitlab immense disk usage"
date: "2022-02-28"
tags: ['gitlab', 'docker', 'devops']
lang: "en"
---

The gitlab docker registry has no cleanup job per default. If an image tag gets overwritten (updated) then the original
image layer blobs will be kept as orphans. To clean those up run:

```shell
gitlab-ctl registry-garbage-collect --delete-untagged --delete-manifest
```

# Automation

* run the command via Cron (the gitlab image contains `go-crond`)
* set command in env var `GITLAB_POST_RECONFIGURE_SCRIPT` (executed at least on each container start)

# Conclusion

170GiB server disk storage freed
