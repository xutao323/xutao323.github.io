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

I'm a system performance engineer focusing on various CPU platform tunings. Rcently work at Sanechips and Alibaba Cloud OS team. Previously, I spent most time developing storage products at EMC and all flash startup. [Here](https://www.linkedin.com/in/tao-xu-49670733/) is my LinkedIn page.

## Contact

xutao323[AT]gmail.com OR xutao323[AT]hotmail.com

