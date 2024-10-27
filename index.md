<ul>
  {% assign sorted_posts = site.posts | sort: 'date' | reverse %}
  {% for post in sorted_posts %}
  <li>
    <span>{{ forloop.index }}.</span>
    <a href="{{ post.url }}">{{ post.title }}</a>
    <span>{{ post.date | date: "%B %d, %Y" }}</span>
  </li>
  {% endfor %}
</ul>
