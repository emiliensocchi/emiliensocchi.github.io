---
permalink: /blog/
layout: archive
---

<h3 class="archive__subtitle"><i class="fas fa-bookmark"></i> {{ site.data.ui-text[site.locale].recent_posts | default: "All Posts" }}</h3>

{% assign entries_layout = page.entries_layout | default: 'list' %}
<div class="entries-{{ entries_layout }}">
  {% assign postsByYear = site.posts | group_by_exp:"post", "post.date | date: '%Y'" %}
  {% for year in postsByYear %}
    <h2>{{ year.name }}</h2>
    <hr>
    {% for post in year.items %}
      {% include archive-single.html type=entries_layout %}
    {% endfor %}
  {% endfor %}
</div>
