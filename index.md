---
layout: default
---

# Recent Posts
---

<!--

{% for post in recent_posts %}
<article class="post">
    <div class="post-inside">
    {% assign pageImageCard_is_not_empty = post.pageImageCard | is_not_empty %}
    {% if pageImageCard_is_not_empty %}
    <a class="post-thumbnail" href="{{ post.url | relative_url }}"><img class="thumbnail"
        src="{{ post.pageImageCard | relative_url }}" alt="{{ post.title }}" /></a>
    {% endif %}
    <header class="post-header">
        <h3 class="post-title"><a href="{{ post.url | relative_url }}" rel="bookmark">{{ post.title }}</a></h3>
    </header>
    <div class="post-content">
        <p>{{ post.excerpt }}</p>
    </div>
    <footer class="post-meta">
        <time class="published"
        datetime="{{ post.date | date: '%Y-%m-%d %H:%M' }}">{{ post.date | date: '%B %d, %Y' }}</time>
    </footer>
    </div>
</article>
{% endfor %}


-->


{% for post in site.posts %}
  {% if post.quickbreach %} {% continue %} {% endif %}
  <article class="postListingBox">
    {% assign pageImageCard_is_not_empty = post.pageImageCard | is_not_empty %}
    {% if pageImageCard_is_not_empty %}
      <div class="postListingBoxLeftDiv">
          <div class="postListingBoxLeftDivThumb">
            <a href="{{ post.url }}">
              <img src="{{post.pageImageCard}}" />
            </a>
          </div>
      </div>
      <div class="postListingBoxRightDiv">
        <h3>
          <a href="{{ post.url }}">
            {{ post.title }}
          </a>
        </h3>
        <div class="postListingBoxDate">
          <time datetime="{{ post.date }}">{{ post.date |  date: "%m-%d-%Y" }}</time>
        </div>
        <p>{{ post.description }}</p>
    </div>
    {% else %}
      <div class="postListingBoxSingleDiv">
        <h3>
          <a href="{{ post.url }}">
            {{ post.title }}
          </a>
        </h3>
        <div class="postListingBoxDate">
          <time datetime="{{ post.date }}">{{ post.date |  date: "%m-%d-%Y" }}</time>
        </div>
        <p>{{ post.description }}</p>
      </div>
    {% endif %}
  </article>
{% endfor %}

---

#### Archived Quickbreach Posts
<div>
  <ul>
  {% for post in site.posts %}
    {% if post.quickbreach %}
      <a href="{{post.url}}"><li>{{post.title}}</li></a>
    {% endif %}
  {% endfor %}
  </ul>
</div>