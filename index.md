<ul>
  {% assign sorted_posts = site.posts | sort: 'date' | reverse %}
  {% for post in sorted_posts %}
  <li>
    <span>{{ forloop.index }}.</span>
    <a href="{{ post.url }}">{{ post.title }}</a>
    <span>{{ post.date | date: "%B %d, %Y" }}</span>
    
    {% if post.url %}
      <p><a href="{{ post.url }}" target="_blank">Reference</a></p>
    {% endif %}
    
    {% if post.tags %}
      <p>Tags: 
        {% for tag in post.tags %}
          <span>{{ tag }}</span>{% unless forloop.last %}, {% endunless %}
        {% endfor %}
      </p>
    {% endif %}
  </li>
  {% endfor %}
</ul>
