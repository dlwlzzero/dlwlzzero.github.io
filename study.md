---
layout: page
title: Study
permalink: /study/
---

<input id="note-search" class="note-search" type="search" placeholder="Search notes…  (title:roofline to match titles only)" autocomplete="off">

<ul class="post-list" id="note-list">
{%- assign notes = site.study | sort: "date" | reverse -%}
{%- for note in notes %}
  <li data-title="{{ note.title | downcase | escape }}" data-body="{{ note.content | strip_html | downcase | escape }}">
    <span class="post-meta">{{ note.date | date: "%Y.%m.%d" }}{% if note.last_modified_at and note.last_modified_at != note.date %} · updated {{ note.last_modified_at | date: "%Y.%m.%d" }}{% endif %}</span>
    <h3><a class="post-link" href="{{ note.url | relative_url }}">{{ note.title }}</a></h3>
  </li>
{%- endfor %}
</ul>
<p class="note-search-empty" id="note-search-empty" hidden>No notes match.</p>

<script>
(function () {
  var q = document.getElementById('note-search'), items = document.querySelectorAll('#note-list > li'), empty = document.getElementById('note-search-empty');
  q.addEventListener('input', function () {
    var v = q.value.trim().toLowerCase(), titleOnly = v.indexOf('title:') === 0;
    if (titleOnly) v = v.slice(6).trim();
    var terms = v.split(/\s+/).filter(Boolean), shown = 0;
    items.forEach(function (li) {
      var hay = li.dataset.title + (titleOnly ? '' : ' ' + li.dataset.body);
      var ok = terms.every(function (t) { return hay.indexOf(t) !== -1; });
      li.hidden = !ok; if (ok) shown++;
    });
    empty.hidden = shown > 0;
  });
})();
</script>
