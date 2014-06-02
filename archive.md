---
layout: page
title: Archive
---
##Posts

{% for post in site.posts %}
<a href="{{ post.url }}" target="_blank">{{ post.title }}</a>  
<span class="archive-post-date">{{ post.date | date: '%B %-d, %Y' }}</span>
{% endfor %}

##Long(ish) Reads

[Flailing Fast](http://flailfast.com/)  
[My Tweet Stream](http://twitter.com/acityinohio)  
Another one coming soon...
