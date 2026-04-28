---
layout: default
title: Tao Xu's homepage
---

## Blogs

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date: "%Y-%m-%d" }}</span> &raquo; <a href="{{ site.url }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

## About

Hello, I'm Tao Xu, a system performance engineer and storage developer. Here is my personal blog.

## Contact

xutao323[AT]gmail.com OR xutao323[AT]outlook.com

