---
layout: default
permalink: /quant/
pagination:
  enabled: true
  per_page: 5
---

<div class="home-page">
  <h1>퀀트 프로젝트</h1>
  
  <div class="posts">
    {% assign quant_posts = site.posts | where: "categories", "quant" %}
    {% for post in quant_posts %}
      <article class="post-preview">
        <header class="post-preview-header">
          <div class="post-title-row">
            <h2>{{ post.title }}</h2>
            {% if post.categories %}
              {% for category in post.categories %}
                <a href="/{{ category }}/" class="category-link">{{ category }}</a>
              {% endfor %}
            {% endif %}
            <time class="post-date" datetime="{{ post.date | date_to_xmlschema }}">
              {{ post.date | date: "%Y년 %m월 %d일" }}
            </time>
          </div>
        </header>
        
        <div class="post-content">
          {{ post.content }}
        </div>
      </article>
    {% endfor %}
  </div>

  <!-- 퀀트 카테고리 페이지네이션 -->
  {% if paginator.total_pages > 1 %}
    <nav class="pagination">
      <ul class="pagination-list">
        {% if paginator.previous_page %}
          <li class="pagination-item">
            <a href="{{ paginator.previous_page_path | relative_url }}" class="pagination-link">
              &lsaquo; 이전
            </a>
          </li>
        {% endif %}

        {% for page in paginator.page_trail %}
          {% if page.num == paginator.page %}
            <li class="pagination-item">
              <span class="pagination-link current">{{ page.num }}</span>
            </li>
          {% else %}
            <li class="pagination-item">
              <a href="{{ page.path | relative_url }}" class="pagination-link">
                {{ page.num }}
              </a>
            </li>
          {% endif %}
        {% endfor %}

        {% if paginator.next_page %}
          <li class="pagination-item">
            <a href="{{ paginator.next_page_path | relative_url }}" class="pagination-link">
              다음 &rsaquo;
            </a>
          </li>
        {% endif %}
      </ul>
    </nav>
  {% endif %}
</div>
