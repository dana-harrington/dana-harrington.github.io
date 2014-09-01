---
layout: page
title: Monads in the Enterprise
tagline: Scala, Play Framework, Akka and functional programming in the e-Commerce world
---
{% include JB/setup %}
<h2>Articles on applied functional programming</h2>

These article are intended to show the practical utility of monads and other functional ideas in solving real world issues. The topics covered have been motivated by issues that I've encountered in my professional experience as a software developer. No theoretical knowledge is required.

<ul class="posts">
  {% for post in site.posts %}
    <li> <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a>
	(<span>{{ post.date | date_to_string }}</span>)
	<div class="post_excerpt">{{ post.excerpt }}</div>
    </li>
  {% endfor %}
</ul>



