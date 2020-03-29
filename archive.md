---
layout: page
---

{% for post in site.posts %}
{% raw %}
<h3>
    <a href="{{ post.url }}">
    {{ post.title }}
    <small>{{ post.date | date_to_string }}</small>
    </a>
</h3>
{% endraw %}
{% endfor %}