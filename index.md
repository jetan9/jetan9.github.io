---
layout: page
title: Hello World!
tagline: Supporting tagline
---
{% include JB/setup %}

Read [Jekyll Quick Start](http://jekyllbootstrap.com/usage/jekyll-quick-start.html)

Complete usage and documentation available at: [Jekyll Bootstrap](http://jekyllbootstrap.com)

## Update Author Attributes

In `_config.yml` remember to specify your own data:
    
    title : My Blog =)
    
    author :
      name : Name Lastname
      email : blah@email.test
      github : username
      twitter : username

The theme should reference these variables whenever needed.
    
## Sample Posts

This blog contains sample posts which help stage pages and blog data.
When you don't need the samples anymore just delete the `_posts/core-samples` folder.

    $ rm -rf _posts/core-samples

Here's a sample "posts list".

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

<div id="clustrmaps-widget"></div><script type="text/javascript">var _clustrmaps = {'url' : 'http://jetan9.github.io', 'user' : 1159877, 'server' : '4', 'id' : 'clustrmaps-widget', 'version' : 1, 'date' : '2015-02-24', 'lang' : 'zh', 'corners' : 'square' };(function (){ var s = document.createElement('script'); s.type = 'text/javascript'; s.async = true; s.src = 'http://www4.clustrmaps.com/counter/map.js'; var x = document.getElementsByTagName('script')[0]; x.parentNode.insertBefore(s, x);})();</script><noscript><a href="http://www4.clustrmaps.com/user/00c11b2c5"><img src="http://www4.clustrmaps.com/stats/maps-no_clusters/jetan9.github.io-thumb.jpg" alt="Locations of visitors to this page" /></a></noscript>

## To-Do

This theme is still unfinished. If you'd like to be added as a contributor, [please fork](http://github.com/plusjade/jekyll-bootstrap)!
We need to clean up the themes, make theme usage guides with theme-specific markup examples.


