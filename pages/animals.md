---
layout: page
permalink: "/animals/"
---

<ul>
  {% for animal in site.animals %}
    <li>
      <a href="{{ animal.url }}">{{ animal.title }}</a>
      - {{ animal.headline }}

      {{ animal.content }}
    </li>
  {% endfor %}
</ul>

