---
 layout: page
 title: Hello World!
 tagline: Supporting tagline
 title: Welcome to Shako's Home
 tagline: Enjoy Life & For Your OnePiece
 titleline: Home
 links: 
  
---
{% include JB/setup %}
{% include xh/logo %}
<div>
<ul class="posts">
<h2 id="index_titleline">{{page.titleline}}</h2>
<hr id="index_line" />

<div class="span6 pull-left">
  <ul class="posts index_posts span8">
  <!-- for post in site.posts -->
  {% for post in site.posts %}
   
   <li>
<div class="index_intro span6">
<h3 class="index_title"><a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></h3>
<p class="index_excerpt">{{ post.excerpt }}</p>
</div>
  <span class="index_date span2">{{ post.date | date_to_string }}</span>
  </li>
   {% endfor %}
</ul>

</div>

<div id="aside" class="well sidebar-nav">
<ul class="nav nav-list">
 <li class="nav-header">About me</li>
<img width="160" height="160" target="_blank" src = "{{ ASSET_PATH }}/images/avatar.jpg"/>
<li style="margin-top:9px">微博：<a href="http://weibo.com/ecnushako">盘子里想家的鱼</a></li>
<li style="margin-top:9px">博客：<a href="http://sorafantasy.blogspot.com">空の幻想</a></li>  
  <li class="nav-header">Categories</li>
  {% assign categories_list = site.categories %}
  {% include JB/categories_list %}
 
  <li class="nav-header">Tags</li>
  {% assign tags_list = site.tags %}
  {% if tags_list.first[0] == null %}
  {% for tag in tags_list %} 
  <li class="index_tags"><a href="{{ BASE_PATH }}{{ site.JB.tags_path }}#{{ tag }}-ref">{{ tag }} <span>{{ site.tags[tag].size }}</span></a></li>
  {% endfor %}
  {% else %}
  {% for tag in tags_list %} 
  <li class="index_tags"><a href="{{ BASE_PATH }}{{ site.JB.tags_path }}#{{ tag[0] }}-ref">{{ tag[0] }} <span>{{ tag[1].size }}</span></a></li>
  {% endfor %}
  {% endif %}
  {% assign tags_list = nil %}

  <li class="clear"></li>
</ul>
</div>
  </ul>
  <ul class="pager">
   {% if site.previous_page %}
   <li class="previous">
   {% if site.page == 2 %}
   <a href="/" rel="bookmark">Previous Page</a>
   {% else %}
   <a href="/page{{ site.previous_page }}" rel="bookmark">Previous Page</a>
   {% endif %}
   </li>
   {% endif %}
   {% if site.next_page %}
   <li class="next">
   <a href="/page{{ site.next_page }}" rel="bookmark" style="float:right">Next Page</a>
   </li>
   {% endif %}

  </ul>

</div>
 


