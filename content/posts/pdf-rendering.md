---
title: "How to make your program generate pdf files"
date: "2022-08-05"
tags: [pdf, webdev, latex]
lang: "en"
hackerNewsId: ""
---

PDF files are still highly popular. The user will find all information in a single, human-readable file which is quite durable.
However, the programmatic generation of pdf files can be a bit tricky. A rendering engine is needed to transform the source data into the target pdf file. The source data can be broken up into two parts: templates and dynamic data. The template is a skeleton of the pdf document. It contains static texts, images, styling and positioning rules etc. It also contains field definitions where the dynamic data will be filled in. The process of merging template and dynamic data is done by a templating engine.

There are multiple technologies applicable.

# Office Suites

It's pretty easy to generate a pdf file by using an office suite like LibreOffice or Google Docs. It's even possible to automate this with a headless LibreOffice instance. But working with  [OpenDocument](https://de.wikipedia.org/wiki/OpenDocument) files as templates for the pdf creation is not developer friendly (e.g. no git diffs ...). The implementation will be extra hard if the dynamic data also include complicated stuff like styling directives. On the other hand templates might be maintained in a WYSIWYG manner with user know software (basically just the office suite).

# Programmatic drawing

Drawing and writing elements programmatically on the pdf canvas is a pragmatic approach. The template is just a regular pdf file with some blanks. While this approach definitely works for simple documents, it gets harder with increasing document complexity. For example necessary line breaks and text alignments are easy to get wrong. In the long run a more sophisticated approach will be easier to handle.

# Latex

Latex is an extremely powerful and function-rich text processor which can also generate pdf files, of course. Using latex files as templates is also pretty easy because it is just text containing latex rules. The maintainers of the templates must be able to work with latex files, which could be quite a restriction. They need to learn latex first.

> The dynamic input is arbitrary data. To prevent **Latex Injections** the templating engine must escape all dynamic input. Otherwise, an attacker might tamper the generated pdf file or even execute commands on your server (e.g. when the latex flag `-shell-escape` is set)!
> {.warning }

PDF generation using latex is a solid and well-tested approach.

# Webbrowsers

Webbrowsers can render the DOM (web content) of a webpage to a pdf file. This function is regularly exposed in the user interface. It's also available via JavaScript as e.g. `window.print()`. However, this is still manual and based on user interaction.

But a headless browser engine (typically firefox or chromium) can also be executed in the backend. An adapter, called webdriver, is needed to control such an instance, e.g. Selenium, PhantomJs (deprecated) or Puppeteer. Now it's possible to get a pdf via the webdriver out of the headless browser. Keep in mind that it takes some time to spin up a browser instance, fully load and render the webpage and then generate the pdf.

This approach is really developer (especially web-developer) friendly. Templates might be maintained via a WYSIWYG webpage editor.

Problematic is the potentially high resource usage of a browser engine. Running too many parallel rendering jobs at the same time can eat up the servers resources. To prevent this a job queue should be used in conjunction with a limited number of parallel executors.

# Pseudo Webbrowsers

Full-featured browser engines like firefox and chromium are powerful but (relatively) slow. There are engines which only implement a subset of the web standards (e.g. only HTML4 with some CSS3 rules) but have a significant higher throughput and less resource usage. Examples are [WeasyPrint](https://weasyprint.org/) and [xhtml2pdf](https://xhtml2pdf.readthedocs.io/). The downside of this approach is that due to the non-standard conforming behaviour each engine interprets the html and css rules differently. So the templates will not be portable. It might also be difficult to figure out how the engine actually interprets positioning and styling rules.





