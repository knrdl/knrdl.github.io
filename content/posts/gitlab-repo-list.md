---
title: "Gitlab: Generate repository list report"
date: "2023-05-12"
tags: [gitlab, python]
lang: "en"
---

1. You must be admin to get a list of all repositories
2. You need a personal access token: Gitlab WebUI → Preferences → Access Tokens → Select scopes: `api` or `read_api` → Create personal access token
3. Run the script:

```python
import json
import requests
import csv

token = 'YOUR_PERSONAL_ACCESS_TOKEN'
page = 1

repos = []

users = {}

while True:
    print('Page:', page)
    res = requests.get(f'https://gitlab.example.org/api/v4/projects?page={page}&per_page=100&private_token={token}')
    assert res.status_code == 200
    records = res.json()
    if records:
        for record in records:
            creator_id = record['creator_id']
            if creator_id:  # attach creator user record to the repo
                if creator_id not in users:
                    user_res = requests.get(f'https://gitlab.example.org/api/v4/users/{creator_id}?private_token={token}')
                    assert user_res.status_code == 200
                    users[creator_id] = user_res.json()
                record['creator'] = users[creator_id]
            else:
                record['creator'] = None
            repos.append(record)
    else:
        break
    page+=1


with open('repos.json', 'w') as f:
    json.dump(repos, f)


with open('repos.csv', 'w') as f:
    csvw = csv.writer(f, delimiter=',', quotechar='"', quoting=csv.QUOTE_MINIMAL)

    csvw.writerow(['Title', 'Url', 'Creator', 'Created', 'Last Activity', 'Private'])
    for repo in repos:
        if not repo['archived'] and not repo['empty_repo']:  # filter out archived repos and repos without files (e.g. issue only repos)
            csvw.writerow([
                repo['name_with_namespace'], 
                repo['web_url'], 
                f"{repo['creator']['name']} ({repo['creator']['username']})" if repo['creator'] else '', 
                repo['created_at'].split('T')[0], 
                repo['last_activity_at'].split('T')[0], 
                repo['visibility'] == 'private' and repo['namespace']['path'] == repo.get('owner', {}).get('username')
            ])
```
