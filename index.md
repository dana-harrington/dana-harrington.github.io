---
layout: page
title: Monads in the Enterprise
tagline: Scala, Play Framework, Akka and functional programming in the e-Commerce world
---
{% include JB/setup %}
<h2>Articles on applied functional programming</h2>

These articles are intended to demonstrate the application of monads and other functional ideas to solve real world issues that arise in mainstream software development. 
My hope is to demonstrate to working programmers how ideas from functional programming can be applied to solve real problems, while achieving clean, testable and maintainable code. My target audience is working developers who have a basic familiarity with Scala, and who have an interest in functional programming. If you appreciate the concise expressiveness of Scala, but don't really 'get' monads, this is the place for you.


<ul class="posts">
  {% for post in site.posts %}
    <li> <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a>
	{% if post.github != null %}
	[<a href="http://github.com/dana-harrington/{{ post.github }}/"> code on github </a>]
	{% endif %}
	<div class="post_excerpt">{{ post.excerpt }}</div>
    </li>
  {% endfor %}
</ul>



