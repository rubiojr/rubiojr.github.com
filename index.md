---
layout: page
---
{% include JB/setup %}

<center>
<h1>rbel.co</h1>
<div style="font-size: 10px; color: gray;padding-bottom: 40px;">
de los CO de toda la vida, maño (41.648791,-0.889581)
</div>
<img src="http://rbel.frameos.org/images/rbel.png"/>
<div style="font-size: 10px; color: gray;padding-bottom: 40px;">
Original Scirocco Picture from <a style="text-decoration: none; color: #333;" href="http://www.flickr.com/photos/mk1archive/1438507780/">mk1archive</a>
</div>
</center>

## Writings

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>


