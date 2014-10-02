---
layout: page
title: Articles
permalink: /tags/
---

<ul>
{% for tag in site.tags %}
    <li>
        <strong><i class="fa fa-tags"> {{ tag[0] }}</i></strong> <small>({{ tag[1].size }} article{% if tag[1].size > 1 %}s{% endif %})</small>
        <ul>
{% for post in tag[1] %}
            <li><a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %}
        </ul>
    </li>
{% endfor %}
</ul>