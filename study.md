---
layout: page
title: Study
permalink: /study/
---

<input id="note-search" class="note-search" type="search" placeholder="Search notes…  (title:roofline to match titles only)" autocomplete="off">

{%- assign groups = "review,note" | split: "," -%}
{%- assign labels = "Paper Reviews,Notes" | split: "," -%}
{%- for g in groups -%}
{%- assign items = site.study | where: "type", g | sort: "date" | reverse -%}
{%- if items.size > 0 %}
<section class="note-group" id="group-{{ g }}">
  <h2>{{ labels[forloop.index0] }}</h2>
  <ul class="post-list">
  {%- for note in items %}
    <li data-title="{{ note.title | downcase | escape }}" data-body="{{ note.content | strip_html | downcase | escape }}">
      <span class="post-meta">{{ note.date | date: "%Y.%m.%d" }}{% if note.last_modified_at and note.last_modified_at != note.date %} · updated {{ note.last_modified_at | date: "%Y.%m.%d" }}{% endif %}</span>
      <h3><a class="post-link" href="{{ note.url | relative_url }}">{{ note.title }}</a></h3>
    </li>
  {%- endfor %}
  </ul>
</section>
{%- endif -%}
{%- endfor %}
<p class="note-search-empty" id="note-search-empty" hidden>No notes match.</p>

<script>
(function () {
  var q = document.getElementById('note-search'), empty = document.getElementById('note-search-empty');
  var groups = document.querySelectorAll('.note-group');
  q.addEventListener('input', function () {
    var v = q.value.trim().toLowerCase(), titleOnly = v.indexOf('title:') === 0;
    if (titleOnly) v = v.slice(6).trim();
    var terms = v.split(/\s+/).filter(Boolean), shown = 0;
    groups.forEach(function (g) {
      var visible = 0;
      g.querySelectorAll('li').forEach(function (li) {
        var hay = li.dataset.title + (titleOnly ? '' : ' ' + li.dataset.body);
        var ok = terms.every(function (t) { return hay.indexOf(t) !== -1; });
        li.hidden = !ok; if (ok) visible++;
      });
      g.hidden = visible === 0; shown += visible;
    });
    empty.hidden = shown > 0;
  });
})();
</script>
