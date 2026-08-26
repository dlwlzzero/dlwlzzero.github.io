---
layout: page
title: Study
permalink: /study/
---

<ul class="post-list">
{%- assign notes = site.study | sort: "date" | reverse -%}
{%- for note in notes %}
  <li>
    <span class="post-meta">{{ note.date | date: "%b %-d, %Y" }}</span>
    <h3><a class="post-link" href="{{ note.url | relative_url }}">{{ note.title }}</a></h3>
    {{ note.excerpt }}
  </li>
{%- endfor %}
</ul>
