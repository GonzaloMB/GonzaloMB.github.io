<div id="search-container" style="margin: 1rem 0;">
  <input id="post-search" type="text" placeholder="Search by title, tag, or date (e.g., 2025-10 or October)" style="width: 100%; padding: 0.5rem; border-radius: 6px; border: 1px solid #565f89; background: transparent; color: inherit;" />
  <small id="search-help">Showing <span id="post-count"></span> posts</small>
  <script>
    (function () {
      const input = document.getElementById('post-search');
      const list = document.getElementById('post-list');
      if (!list) return;
      const items = Array.from(list.querySelectorAll('li'));
      const countEl = document.getElementById('post-count');
      function filter() {
        const q = (input.value || '').trim().toLowerCase();
        let visible = 0;
        items.forEach(li => {
          if (!q) { li.style.display = ''; visible++; return; }
          const t = (li.dataset.title || '');
          const tags = (li.dataset.tags || '');
          const date = (li.dataset.date || '');
          const datestr = (li.dataset.datestr || '');
          const match = t.includes(q) || tags.includes(q) || date.includes(q) || datestr.includes(q);
          li.style.display = match ? '' : 'none';
          if (match) visible++;
        });
        if (countEl) countEl.textContent = String(visible);
      }
      input.addEventListener('input', filter);
      // initialize count on load
      filter();
    })();
  </script>
</div>

<ul id="post-list">
  {% assign sorted_posts = site.posts | sort: 'date' | reverse %}
  {% for post in sorted_posts %}
  <li class="post-item"
      data-title="{{ post.title | downcase }}"
      data-tags="{% if post.tags %}{{ post.tags | join: ' ' | downcase }}{% endif %}"
      data-date="{{ post.date | date: '%Y-%m-%d' }}"
      data-datestr="{{ post.date | date: '%B %d, %Y' | downcase }}">
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
