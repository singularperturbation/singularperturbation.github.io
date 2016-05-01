---
layout: default
title: Tags
permalink: /tags/
---

# Tags

Click to see list of posts with each tag.
{% for tag in site.tags %}
  {% assign tagname  = tag | first %}
  {% assign postlist = tag | last %}
### [{{tagname}}]({{site.baseurl}}/tag/{{tagname | jekyll_tagging_slug }})
  {{ postlist | size }} post(s) with this tag.
{% endfor %}
