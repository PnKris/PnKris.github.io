---
layout: page
permalink: "/animals/"
---

{% for animal in site.animals %}
    
      <a href="{{ animal.url }}">{{ animal.title }}</a>
	
      {{ animal.content }}
    
{% endfor %}


