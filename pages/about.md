---
layout: page
title: About
description: resume
keywords: Hojun Kim
comments: false
menu: About
permalink: /about/
---

## Contact

<ul>
{% for website in site.data.social %}
<li>{{website.sitename }}：<a href="{{ website.url }}" target="_blank">@{{ website.name }}</a></li>
{% endfor %}
</ul>
