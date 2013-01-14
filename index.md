---
layout: page
---
{% include JB/setup %}

<center>
  <div style="width: 400px">
    <img src="http://rubiojr.frameos.org/images/aaron.jpg"/>
    <div style="font-weight: bold">Aaron Swartz</div>
    <div>
      <i>"World wanderers, we have lost a wise elder. Hackers for right, we are one down. Parents all, we have lost a child. Let us weep."</i>
    </div>
    <div style="text-align:right">
      Tim Berners-Lee.
    </div>
  </div>
</center>


## Writings

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>


