---
layout: page
title: Categories
permalink: /categories/
---

{% assign sorted_cats = site.categories | sort %}
{% for category in sorted_cats %}
## {{ category | first }}
<ul>
    {% for posts in category %}
        {% for post in posts %}
            {% if post.url %}
                <li><a href="{{ post.url }}">{{ post.title }}</a></li>
            {% endif %}
        {% endfor %}
    {% endfor %}
</ul>
{% endfor %}