---
layout: page
title: Weekly Progress
icon: fas fa-calendar-week
order: 1
---

## Blog Posts

<div class="post-list">
  {% for post in site.posts %}
    <div style="margin-bottom: 2rem; padding-bottom: 2rem; border-bottom: 1px solid #e1e4e8;">
      <h3>
        <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      </h3>
      <p style="color: #666; font-size: 0.9rem;">
        <i class="far fa-calendar"></i> {{ post.date | date: "%B %d, %Y" }}
        {% if post.categories %}
          <span style="margin-left: 1rem;">
            <i class="fas fa-folder"></i>
            {% for category in post.categories %}
              <span class="category">{{ category }}</span>{% unless forloop.last %}, {% endunless %}
            {% endfor %}
          </span>
        {% endif %}
      </p>
      {% if post.excerpt %}
        <p>{{ post.excerpt | strip_html | truncatewords: 50 }}</p>
      {% endif %}
      <a href="{{ post.url | relative_url }}" style="color: #0366d6; text-decoration: none;">Read more →</a>
    </div>
  {% endfor %}
</div>

---

## Browse by Tag

Common tags: [#ur10](/final-year-blog/tags/ur10/), [#vision](/final-year-blog/tags/vision/), [#grasping](/final-year-blog/tags/grasping/), [#recovery](/final-year-blog/tags/recovery/), [#data](/final-year-blog/tags/data/)

---

*Posts are updated weekly throughout the project duration*
