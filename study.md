---
layout: page
title: Study
permalink: /study/
---

읽은 논문과 시스템 공부 기록. 대부분 한국어, 인용과 용어는 원문 그대로.

<ul class="post-list">
{%- assign notes = site.study | sort: "date" | reverse -%}
{%- for note in notes %}
  <li>
    <span class="post-meta">{{ note.date | date: "%Y.%m.%d" }}{% if note.last_modified_at and note.last_modified_at != note.date %} · updated {{ note.last_modified_at | date: "%Y.%m.%d" }}{% endif %}</span>
    <h3><a class="post-link" href="{{ note.url | relative_url }}">{{ note.title }}</a></h3>
  </li>
{%- endfor %}
</ul>
