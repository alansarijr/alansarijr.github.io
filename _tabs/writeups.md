---
layout: page
title: CTF Writeups
icon: fas fa-flag
order: 2
---

A collection of my Capture The Flag writeups, covering reverse engineering, cryptography, and more.

{% assign ctf_posts = site.posts | where_exp: "post", "post.categories contains 'CTF Writeups'" %}

{% if ctf_posts.size == 0 %}
  <p>No writeups published yet. Check back soon!</p>
{% else %}
  {% assign grouped = ctf_posts | group_by_exp: "post", "post.categories | last" %}
  {% for group in grouped %}
  <h2>{{ group.name }}</h2>
  <ul>
    {% for post in group.items %}
    <li>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      <span class="text-muted" style="font-size:0.85em;margin-left:0.5em;">— {{ post.date | date: "%b %d, %Y" }}</span>
      {% if post.description %}<br><small>{{ post.description }}</small>{% endif %}
    </li>
    {% endfor %}
  </ul>
  {% endfor %}
{% endif %}
