---
layout: page
title: Categories
permalink: /categories/
---

{% comment %} Catégories des liens externes {_data/links.yml} {% endcomment %}
{% assign link_categories = site.data.links.medium_articles | map: "category" | uniq %}

{% comment %} Noms des catégories internes issues des posts Jekyll {% endcomment %}
{% assign jekyll_cat_names = "" | split: "" %}
{% for pair in site.categories %}
  {% assign jekyll_cat_names = jekyll_cat_names | push: pair[0] %}
{% endfor %}

{% comment %} Fusion et tri des catégories internes et externes {% endcomment %}
{% assign all_cat_names = link_categories | concat: jekyll_cat_names | uniq | sort %}

{% for cat in all_cat_names %}
## {{ cat }}

<ul>
  {% comment %} Afficher les posts Jekyll dans cette catégorie {% endcomment %}
  {% assign posts_in_cat = site.categories[cat] %}
  {% if posts_in_cat %}
    {% for post in posts_in_cat %}
      {% if post.url %}
        <li><a href="{{ post.url }}">{{ post.title }}</a></li>
      {% endif %}
    {% endfor %}
  {% endif %}

  {% comment %} Afficher les articles externes (links.yml) {% endcomment %}
  {% for item in site.data.links.medium_articles %}
    {% if item.category == cat %}
      <li><a href="{{ item.url }}">{{ item.title }}</a></li>
    {% endif %}
  {% endfor %}
</ul>
{% endfor %}
