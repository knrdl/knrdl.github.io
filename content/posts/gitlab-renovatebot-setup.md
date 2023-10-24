---
title: "Gitlab: RenovateBot Setup"
date: "2023-10-24"
tags: [gitlab, devops]
lang: "en"
hackerNewsId: ""
---

1. Create a gitlab user "renovate.bot":
    * External User: Yes
    * Personal projects limit: 0
    * Can create top level groups: No

2. Impersonate "renovate.bot" (or login as this user) and create a `Settings` → `Access Tokens`:
    * Expiration: 1 year (place a reminder to renew this!)
    * Scopes: api, read_api, read_repository, write_repository, read_registry

3. Create a repository "renovate.bot" and add the user "renovate.bot" as Maintainer

4. Set a pipeline schedule (how often the renovate bot will run), e.g. every day at 5am: `0 5 * * *`

5. `Settings` → `CI/CD` → `Variables`: add a variable to supply the access token:
    * Key: `RENOVATE_TOKEN`
    * Value: the access token of the "renovate.bot" user (step 2)
    * Protect and mask the variable

> Keep in mind that every gitlab user with (at least) Maintainer role in the "renovate.bot" repository can see the `RENOVATE_TOKEN`. For this reason every maintainer can see and do everything the "renovate.bot" user is capable of.
> {.warning }

6. Add these two files to the repo:

* `.gitlab-ci.yml`:

```yaml
renovate:
  image: renovatebot/renovate:37-slim
  variables:
    RENOVATE_LOG_FILE: renovate-log.ndjson
    RENOVATE_LOG_FILE_LEVEL: debug
    LOG_LEVEL: info   # console logging should be less verbose than the logfile  because pipeline output is kept forever, logfiles can expire
    RENOVATE_BASE_DIR: $CI_PROJECT_DIR
    RENOVATE_ENDPOINT: $CI_API_V4_URL
    RENOVATE_PLATFORM: gitlab
    RENOVATE_ALLOW_SCRIPTS: 'true'
    RENOVATE_AUTODISCOVER: 'true'  # all repos where the "renovate.bot" user is (at least) developer are handled
    RENOVATE_ONBOARDING: 'false'  # no initial MR required
    RENOVATE_REPOSITORY_CACHE: 'enabled'
    RENOVATE_REQUIRE_CONFIG: 'optional'  # the renovate.json file in each handled repo is not required
    RENOVATE_TOKEN: "$RENOVATE_TOKEN"
    RENOVATE_PR_HOURLY_LIMIT: '50'  # limit the creation of MRs
    RENOVATE_CONFIG_FILE: 'config.json'  # path to renovate global config in "renovate.bot" repo
    NODE_OPTIONS: --use-openssl-ca
    SSL_CERT_DIR: /etc/ssl/certs
    REQUESTS_CA_BUNDLE: /etc/ssl/certs/ca-certificates.crt
  cache:
    key: ${CI_COMMIT_REF_SLUG}-renovate
    paths:
      - cache
  script:
    - renovate
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
  artifacts:
    when: always
    expire_in: 1d
    paths:
      - '$RENOVATE_LOG_FILE'  # remove the renovate logfile from each run after one day to reduce disk usage
```

* `config.json`:

```json
{
    "$schema": "https://docs.renovatebot.com/renovate-schema.json",
    "packageRules": [
        {
            "matchDatasources": [
                "docker"
            ],
            "registryUrls": [
                "https://docker-registry-mirror.example.org"
            ]
        },
        {
            "matchUpdateTypes": [
                "minor",
                "patch"
            ],
            "enabled": true
        },
        {
            "matchUpdateTypes": [
                "major"
            ],
            "enabled": false
        }
    ]
}
```

This will use a docker registry mirror and open MRs for minor and patch updates (no breaking changes). Unless `"automerge": true` is specified, these MRs will stay open for manual intervention.

7. Add the "renovate.bot" user to your repositories:
    * As **Developer** if the renovate bot should open MRs *but not* automerge them
    * As **Maintainer** if the renovate bot should open MRs *and* automerge them (on protected branches)
    * As **Reporter** if the renovate bot depends on this repo to do the version checks for another repo (e.g. another repo is based on the docker image in the gitlab docker registry of this repo)

8. Trigger a run of the scheduled pipeline to test the setup.

9. View all MRs (for your repositories) opened by the renovate bot: https://git.example.org/dashboard/merge_requests?scope=all&state=opened&author_username=renovate.bot

