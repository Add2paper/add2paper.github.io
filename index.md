---
layout: default
title: Add2paper Engineering Blog
tagline:
class: index_page
---
{% include JB/setup %}

<div style="margin-bottom: 50px;">
    <div class="page-header">
      <h1><a href=" {{ root_url }}{{ site.posts.first.url }} "> {{ site.posts.first.title }} </a></h1>
    </div>

    <div class="row-fluid">
      <div class="span12">
        {{ site.posts.first.content }}
      </div>
    </div>
</div>

<hr />

<div style="margin-top: 50px;">
    <div class="page-header">
      <h1><a href=" {{ root_url }}{{ site.posts.last.url }} "> {{ site.posts.last.title }} </a></h1>
    </div>

    <div class="row-fluid">
      <div class="span12">
        {{ site.posts.last.content }}
      </div>
    </div>
</div>