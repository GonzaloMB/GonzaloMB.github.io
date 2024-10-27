<link rel="stylesheet" href="/assets/css/custom.css">
<ul>
  {% assign sorted_posts = site.posts | sort: 'date' | reverse %}
  {% for post in sorted_posts %}
  <li>
    <span>{{ forloop.index }}.</span>
    <a href="{{ post.url }}">{{ post.title }}</a>
    <span>{{ post.date | date: "%B %d, %Y" }}
        {% if post.tags %} 
        {% for tag in post.tags %}
          <span>{{ tag }}</span>{% unless forloop.last %}, {% endunless %}
        {% endfor %}
    {% endif %}
    </span>
  </li>
  {% endfor %}
</ul>
