---
layout: page
title: Categories
permalink: /categories/
---

{% sorted_for category in site.categories reversed sort_by:title case_sensitive:true %}
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