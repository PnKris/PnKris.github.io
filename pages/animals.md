---
layout: page
permalink: "/animals/"
---


<ul>
  {% for animal in site.animals %}
    <li>
   <h1>   <a href="{{ animal.url }}">{{ animal.title }}</a>
      - {{ animal.headline }}
</h1>
      {{ animal.content }}
    </li>
  {% endfor %}
</ul>

