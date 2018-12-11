---
layout: page
permalink: "/animals/"
---

{% for animal in site.animals %}
    <h1>
      <a href="{{ animal.url }}">{{ animal.title }}</a>
	</h1>
      {{ animal.content }}
    
{% endfor %}


