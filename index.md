---
layout: home
---

Here is what I am building:

{% for category in site.categories %}
<ul>
  {% for post in category[1] %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      <small>({{ post.date | date: "%b %d, %Y" }})</small>
    </li>
  {% endfor %}
</ul>
{% endfor %}
