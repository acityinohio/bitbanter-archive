---
layout: page
title: Archive
---

{% for post in site.posts %}
<a href="{{ post.url }}" target="_blank">{{ post.title }}</a>  
<span class="archive-post-date">{{ post.date | date: '%B %-d, %Y' }}</span>
{% endfor %}

##Long(ish) Reads

<a href="http://flailfast.com/" target="_blank">Flailing Fast</a>  
<a href="http://twitter.com/ACityInOhio" target="_blank">My Tweet Stream</a>  
Another one coming soon...
