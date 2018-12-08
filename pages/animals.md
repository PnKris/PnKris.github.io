---
layout: page
title: "A list of animals"
permalink: "/animals/"
---

<ul>
  {% for animal in site.animals %}
    <li>
      <a href="{{ animal.url }}">{{ animal.title }}</a>
      - {{ animal.headline }}
    </li>
  {% endfor %}
</ul>
