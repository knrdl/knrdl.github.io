---
title: "Strategies for Docker image tagging"
date: "2022-06-27"
tags: [docker]
lang: "en"
hackerNewsId: ""
---

Each docker image has a URI-style name like `domain.tld:port/directory/subdir:tag`. The image name `ubuntu` is just a shortcut for `docker.io/library/ubuntu:latest`. The default tag (if omitted) is `latest`. Docker (or <abbr title="Open Container Initiative">OCI</abbr>) images should be built automatically in <abbr title="Continuous Integration">CI</abbr>-Pipelines. There are multiple strategies for tagging those images:

1. **Docker Image Tagging via Git Commit Tags:**
    * Relevant git commits are tagged as versions, e.g. `v3.2.1`
    * The corresponding docker image gets tagged as `3.2.1`. If there are multiple image flavours, maybe also `3.2.1-alpine` or `v3.2.1-slim`
    * [Semantic Versioning](https://semver.org/) is also applicable: In addition to the image tag `3.2.1`, the tags `3.2` and `3` should also be assigned, if `3.2.1` is the newest version for both version series.
    * The image created from the newest git commit tag (the newest version) must also be tagged as `latest`
    * Docker images without associated git commit tag can be tagged as `edge` (convention) or something like `unstable`, `nightly` etc.
   
2. **Docker Image Tagging via Git Commit Hash:**
    * If there are no distinguishable software versions, the first 6 letters of the git commit hash (SHA) can be used to tag an image.
    * The `latest` tag gets assigned manually to a docker image.
    * Pro: It's easy to find the belonging image for a git commit. So an old version can easily be deployed if the newer one fails. Therefore, you must know which commit introduced the problem.
    * Contra:
        * Produces many tagged images => Docker Image Tag Deletion Policy must be enforced
        * Tags will only be sortable by their creation date (might be wrong in case of manual intervention)
        * It's hard to know which tags bring in small or big changes (missing [semantic versioning](https://semver.org/))
    * Strategy is only useful if there is no software versioning scheme in place and also git push's to the main branch are not too frequent.
   
3. **Docker Image Tagging for the Bleeding Edge:**
    * There is not dedicated tagging at all
    * The newest commit gets tagged as `latest`
    * Move fast and break things
    * To reproduce old versions the old image must be built again (make sure to pin the versions of all dependencies your software needs). Alternatively important old versions can be tagged manually.
   
4. **Docker Image Tagging with Timestamps:**
    * Each docker image tag is a timestamp of the build date.
    * The last run build is tagged as `latest`
    * Problematic if old versions are built via CI-Pipelines at a later time
    * Useful for nightly builds only. Produce tags like `nightly-YYYYMMDD`

Which strategy is right for me? It depends! Maybe a combination ...

---

## TL;DR / Opinion

* **Use git tags for software versioning**
    * If the newest git tag is `v3.2.1`, the built docker image should be tagged as `3.2.1`, `3.2` and `3`.
    * The newest software version (by git tag) additionally gets tagged as `latest`
* Building a docker image from a git commit without tag, the resulting docker image should be tagged as `edge`

