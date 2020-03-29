---
layout: page
---

# Archive

{% for post in site.related_posts limit:3 %}
{% raw %}
<h3>
    <a href="{{ post.url }}">
    {{ post.title }}
    <small>{{ post.date | date_to_string }}</small>
    </a>
</h3>
{% endraw %}
{% endfor %}