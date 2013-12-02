---
layout: default
title: News
permalink: /news/index.html
---




<div class="newsentries">
  {% for post in site.posts %}
  <a href="http:/scala-blitz.github.com/{{ post.url }}">
    <br/>
    <br/>
    <h1 class="newstitle">
      {{ post.title | upcase }}
    </h1>
  </a>
  <hr class="newstitle"/>
  <div class="newsinfo">
    <img width="15" height="15" src="{{ homedir }}/resources/images/calendar.png"/>&nbsp; {{ post.date | date: "%d.%m.%Y." }}
  </div> 
  <br/>
  {{ post.content }}
  {% endfor %}
</div>






