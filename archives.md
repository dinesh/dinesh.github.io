---
layout: page
title: Archives
---

{% for post in site.posts %}
  <div class="masthead-title">
    <a href="{{ post.url | prepend: site.baseurl}}">
      <span> {{post.title}} -- {{ post.date | date: "%d %b, %Y"}}</span>
      <div> <small> {{post.excerpt | strip_html | truncatewords:35 }}</small> ... </div>
    </a>
  </div>
{% endfor %}
