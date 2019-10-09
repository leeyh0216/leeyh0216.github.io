---
layout: post
title: Tags
permalink: /tags/
---

{% for tag in site.tags %}
* [{{ tag | first }}]({{ site.baseurl }}/tag/{{ tag | first }})
{% endfor %}
