---
# the default layout is 'page'
layout: archives
icon: fas fa-info-circle
order: 5
---

<ul>
{% assign pwn_articles = site.posts | where: 'tags', 'pwn' %}
{% for article in pwn_articles %}
  <li><a href="{{ article.url }}">{{ article.title }}</a></li>
{% endfor %}
</ul>
