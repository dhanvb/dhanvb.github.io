---

title: "Engineering Articles"
permalink: /posts/
layout: single
author_profile: true
--------------------

These articles document engineering decisions, implementation patterns, failure modes, operational controls, troubleshooting, and lessons from infrastructure and platform engineering work.

The emphasis is on **why a control or architecture exists**, how it behaves when something goes wrong, and what evidence proves that the intended state was achieved.

{% assign engineering_posts = site.posts %}

{% if engineering_posts.size > 0 %}

<div class="entries-list">

{% for post in engineering_posts %}

  <article class="archive__item">

```
<h2 class="archive__item-title no_toc">
  <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
</h2>

{% if post.date %}
<p class="page__meta">
  {{ post.date | date: "%B %-d, %Y" }}
</p>
{% endif %}

{% if post.excerpt %}
<p class="archive__item-excerpt">
  {{ post.excerpt | markdownify | strip_html | truncate: 260 }}
</p>
{% endif %}
```

  </article>
{% endfor %}

</div>

{% else %}

No engineering articles are currently published.

{% endif %}

