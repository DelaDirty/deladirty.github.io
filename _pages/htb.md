---
title: "HTB"
permalink: /htb/
layout: category
taxonomy: htb          
---
<div class="feature__wrapper">
  {% for post in site.posts %}
  <div class="feature__item">
    <div class="archive__item-teaser">
      <img src="{{ post.header.image | default: '/assets/images/default.jpg' }}" alt="{{ post.title }}">
    </div>
    <div class="archive__item-body">
      <h2 class="archive__item-title">
        <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      </h2>
      <p class="archive__item-excerpt">{{ post.excerpt | strip_html | truncate: 140 }}</p>
      <a href="{{ post.url | relative_url }}" class="btn btn--primary">Read More</a>
    </div>
  </div>
  {% endfor %}
</div>

I hope these walkthroughs help you out on your journey. 
