
{% include JB/setup %}

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

## About
I'm currently a system performance engineer at Alibaba Cloud OS team. Previously, I spent most time developing storage products at EMC and all flash startup. [Here](https://www.linkedin.com/in/tao-xu-49670733/) is my LinkedIn page.

## Contact
xutao323@gmail.com OR xutao323@hotmail.com
