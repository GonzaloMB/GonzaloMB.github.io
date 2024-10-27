---
layout: default
title: "Trial && Error"
---

<div class="post-list">
  {% assign sorted_posts = site.posts | sort: 'date' | reverse %}
  {% for post in sorted_posts %}
  <div class="post-item">
    <span class="post-number">{{ forloop.index }}.</span>
    <a href="{{ post.url }}" class="post-title">{{ post.title }}</a>
    <div class="post-meta">
      <span>{{ post.date | date: "%B %d, %Y" }}</span>
    </div>
  </div>
  {% endfor %}
</div>
