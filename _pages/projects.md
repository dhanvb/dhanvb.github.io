---
title: "Projects"
permalink: /projects/
layout: single
author_profile: true
---

{% assign engineering_projects = site.projects | sort: "date" | reverse %}

{% if engineering_projects.size > 0 %}

<div class="entries-list">

{% for project in engineering_projects %}
  <article class="archive__item">
    <h2 class="archive__item-title no_toc">
      <a href="{{ project.url | relative_url }}">{{ project.title }}</a>
    </h2>

    {% if project.excerpt %}
    <p class="archive__item-excerpt">
      {{ project.excerpt | markdownify | strip_html | truncate: 220 }}
    </p>
    {% endif %}
  </article>
{% endfor %}

</div>

{% else %}

No engineering projects are currently published.

{% endif %}
