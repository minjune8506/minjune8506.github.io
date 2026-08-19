---
layout: page
title: Category
permalink: /categories/
---

{% assign all_categories = site.categories | sort %}

{% for category in all_categories %}
{% assign cat_name = category[0] %}
{% assign cat_posts = category[1] | sort: "date" %}

## {{ cat_name }} <span class="cat-count">({{ cat_posts.size }})</span>

{% assign series_names = "" | split: "," %}
{% for post in cat_posts %}
  {% if post.series %}
    {% unless series_names contains post.series %}
      {% assign series_names = series_names | push: post.series %}
    {% endunless %}
  {% endif %}
{% endfor %}

{% if series_names.size > 0 %}
  {% for series_name in series_names %}
  <div class="cat-series">
    <h3>{{ series_name }} 시리즈</h3>
    {% assign series_posts = cat_posts | where: "series", series_name %}
    {% assign tiers = "기본,심화,뇌절" | split: "," %}
    {% for tier in tiers %}
      {% assign tier_posts = series_posts | where: "tier", tier %}
      {% if tier_posts.size > 0 %}
      <p class="cat-tier">{{ tier }}</p>
      <ul>
        {% for post in tier_posts %}
        <li><a href="{{ post.url | relative_url }}">{{ post.title }}</a></li>
        {% endfor %}
      </ul>
      {% endif %}
    {% endfor %}
    {% assign tierless = series_posts | where_exp: "p", "p.tier == nil" %}
    {% if tierless.size > 0 %}
    <ul>
      {% for post in tierless %}
      <li><a href="{{ post.url | relative_url }}">{{ post.title }}</a></li>
      {% endfor %}
    </ul>
    {% endif %}
  </div>
  {% endfor %}
  {% assign no_series = cat_posts | where_exp: "p", "p.series == nil" %}
  {% if no_series.size > 0 %}
  <ul>
    {% for post in no_series %}
    <li><a href="{{ post.url | relative_url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
  {% endif %}
{% else %}
<ul>
  {% for post in cat_posts %}
  <li><a href="{{ post.url | relative_url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
{% endif %}

{% endfor %}
