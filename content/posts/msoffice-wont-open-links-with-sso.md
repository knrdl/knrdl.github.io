---
title: "Microsoft Office won't open links with SSO"
date: "2023-02-28"
tags: [microsoft, nginx]
lang: "en"
---

M$ Office is tightly bundled with Internet Explorer. So it used to open links in office documents in a separate IE8 instance. That was already problematic as cookies were not preserved. So the user has to login every time he clicks on a link in a excel sheet if the website has authentication in place.

Nowadays the Internet Explorer is finally dead (at least I hope so). When you click on a link in an excel sheet the default browser will pop up. But M$ office still preprocesses links. After you click the link in the excel sheet M$ office will trigger a HTTP request to the link target. If the request is successful the browser will open the link. If it fails with a 4xx or 5xx status code M$ office will just show a error message. The built in user agent will also follow http 3xx redirects. If these end up on a SSO portal with status code 4xx the problem is the same.

Micro$oft offers a * fantastic * solution: Just change a registry key on every machine which might open that excel file. What we can do instead is presenting to M$ office always a 200 response:
* When doing the request the M$ office user agent uses a dedicated User Agent String which contains the M$ product name. So a filter can be applied.
* Provide a dedicated domain for links in M$ office files. E.g. https://appname-msoffice.example.org instead of https://appname.example.org. A request to this dedicated domain can return a 200 and trigger a Javascript redirect. Nginx code snippet:

```
server {
    server_name appname-msoffice.example.org;
    location / {
        add_header Content-Type text/html;
        return 200 '<html><body>Please wait ...</body><script>window.location=window.location.href.replace("-msoffice", "")</script></html>';
    }
}
```

