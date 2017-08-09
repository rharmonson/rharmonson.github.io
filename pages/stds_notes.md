Header for pages:
---
title: Article Title
keywords: linux centos system administration
last_updated: April 2, 2017
tags: [linux centos sysadmin]
summary: "The purpose of this article is to describe ."
layout: default_toc
sidebar:
permalink: filename.html
folder: mydoc
toc: false
published: true
---

Header for posts:
---
title: News Title
keywords: news, blog, updates, release notes, announcements
sidebar: mydoc_sidebar
permalink: news_archive.html
folder:
toc: false
---

Difference between keywords and tags?

For alerts use:
{% include note.html content="alert subject<br/>
Place alert content here."
%}

Available alerts: note, tip, important, and warning.

End every page with:
{% include links.html %}
