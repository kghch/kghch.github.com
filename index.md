---
layout: page
title: blank
tagline: 如果我不去规划我三十岁的人生，每天刷知乎微博朋友圈，无聊而舒服，三十岁时我又能得到什么呢？
---
{% include JB/setup %}

### Posts:

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
