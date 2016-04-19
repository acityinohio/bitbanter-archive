---
layout: page
title: Archive
---

{% for post in site.posts %}
<a href="{{site.url}}{{ post.url }}" target="_blank">{{ post.title }}</a>  
<span class="archive-post-date">{{ post.date | date: '%B %-d, %Y' }}</span>
{% endfor %}

##Long(ish) Reads

<a href="http://www.flailfast.com/" target="_blank">Flailing Fast</a>  
<a href="https://twitter.com/acityinohio" target="_blank">My Tweet Stream</a>  
