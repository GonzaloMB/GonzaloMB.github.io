<ul>
  {% assign sorted_posts = site.posts | sort: 'date' | reverse %}
  {% for post in sorted_posts %}
  <li>
    <span>{{ forloop.index }}.</span>
    <a href="{{ post.url }}">{{ post.title }}</a>
    <span>{{ post.date | date: "%B %d, %Y" }}
        {% if post.tags %}  
        <span>
        (
        {% for tag in post.tags %}
          {{ tag }}
        {% unless forloop.last %}, {% endunless %}
        {% endfor %}
        )
        </span>
    {% endif %}
    </span>

  </li>
  {% endfor %}
</ul>
