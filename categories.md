---
layout: page
title: Categories
permalink: /categories/
---

{% for category in site.categories | sort %}
## {{ category | first }}
<ul>
    {% for posts in category %}
        {% for post in posts %}
            <li><a href="{{ post.url }}">{{ post.title }}</a></li>
        {% endfor %}
    {% endfor %}
</ul>
{% endfor %}