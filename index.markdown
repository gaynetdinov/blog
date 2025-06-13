---
layout: default
---

# Big-Elephants no more

Below are the blog posts I authored during my time at Adjust.
Originally they were published on big-elephants.com, but apparently Adjust decided to shut the site down, that's why
I re-publish them here.

Enjoy!

## Latest Posts

<ul class="post-list">
  {% for post in site.posts %}
    <li>
      <h3>
        <a class="post-link" href="{{ post.url | relative_url }}">
           {{ post.title }}
        </a>
      </h3>
      <p>{{ post.excerpt }}</p>
    </li>
  {% endfor %}
</ul>
