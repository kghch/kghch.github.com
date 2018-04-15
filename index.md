---
layout: page
tagline: 三十而立，四十不惑，五十知天命
---
{% include JB/setup %}

### Posts:

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>