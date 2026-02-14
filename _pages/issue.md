---
layout: default
permalink: /issue/
---

<div class="home-page">
  <h1>Issue</h1>

  <div class="posts">
    {% assign issue_posts = site.posts | where: "categories", "issue" %}
    {% for post in issue_posts %}
      <article class="post-preview">
        <header class="post-preview-header">
          <div class="post-title-row">
            <div class="post-title">{{ post.title }}</div>
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
</div>
