---
title: RichTech Website Practices
published: false
tags:
keywords:
last_updated: August 10, 2017
summary: "A page to establish and reference RichTech Site Practices."
permalink: practices.html
toc: false
---

## Templates

For <b>pages</b> use the template found at pages/mydoc/template_page.md.

for <b>posts</b> use the template found at pages/mydoc/template_post.md.

## tags & keywords

Difference between tags and keywords?

<b>Tags</b> are used to group related articles and present them in a variety of formats for the user. They must be predefined and have correlating html files.

Currently defined tags: centos fedora  hardware linux security service sysadmin virtualization windows

Tags data file _data/tags.yml.

Tags html files pages/tags/tag_[tag].md.

<b>Keywords</b> are used to facilitate the Search feature. Search uses the frontmatter including title, tags, keywords, url, and summary to identify articles to include in the search results.


## Alerts

For alerts use:

```
{% include note.html content="alert subject<br/>
Place alert content here."
%}
```

Available alerts: `note, tip, important, and warning.`

## End of Page

End every page and post with:

```
{% include links.html %}
```

{% include links.html %}
