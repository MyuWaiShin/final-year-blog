---
layout: page
title: Weekly Progress
icon: fas fa-calendar-week
order: 1
---

<div class="post-list" style="display: flex; flex-direction: column; gap: 20px; margin-top: 20px;">
  {% for post in site.posts %}
  <a href="{{ post.url | relative_url }}" class="post-card-link" style="text-decoration: none !important; color: inherit; display: block;">
    <div class="post-card" style="border: 1px solid #eaecef; padding: 24px; border-radius: 8px; background: white; transition: all 0.2s ease-in-out;">
      <h3 style="margin-top: 0; font-size: 1.4em; margin-bottom: 12px; font-weight: 600; color: #24292e;">
        {{ post.title | escape }}
      </h3>

      <p style="color: #586069; line-height: 1.6; margin-bottom: 16px; font-size: 1rem;">
        {{ post.excerpt | strip_html | truncatewords: 35 }}
      </p>

      <div class="post-meta" style="color: #868e96; font-size: 0.85rem; display: flex; align-items: center; gap: 20px;">
        <span class="post-date" style="display: flex; align-items: center;">
          <i class="far fa-calendar" style="margin-right: 6px;"></i> {{ post.date | date: "%b %d, %Y" }}
        </span>
        {% if post.categories.size > 0 %}
        <span class="post-category" style="display: flex; align-items: center;">
          <i class="far fa-folder" style="margin-right: 6px;"></i> {{ post.categories | join: ", " }}
        </span>
        {% endif %}
      </div>
    </div>
  </a>
  {% endfor %}
</div>

<style>
  /* Remove default headers if any */
  h2#blog-posts { display: none; }

  /* Card hover effects */
  .post-card-link:hover .post-card {
    transform: translateY(-2px);
    box-shadow: 0 8px 20px rgba(0,0,0,0.06);
    border-color: #d1d5da;
  }
  
  /* Ensure blue title on hover */
  .post-card-link:hover h3 {
    color: #0366d6 !important;
  }

  /* Force remove underlines used by some themes on anchors */
  a.post-card-link {
    text-decoration: none !important;
    box-shadow: none !important;
    border-bottom: none !important;
  }
</style>


## Browse by Tag

Common tags: [#ur10](/final-year-blog/tags/ur10/), [#vision](/final-year-blog/tags/vision/), [#grasping](/final-year-blog/tags/grasping/), [#recovery](/final-year-blog/tags/recovery/), [#data](/final-year-blog/tags/data/)

---

*Posts are updated weekly throughout the project duration*
