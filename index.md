---
layout: archive
permalink: /
---

<div class="page-lead" style="background-image:url(/images/wood-texture-1600x800.jpg)">
    <div class="wrap page-lead-content">
    <h1>zhLinh</h1>
    <h2>Be the change you want to see the world.</h2>
    <a href="/about/" class="btn-inverse">Know About Me</a> &nbsp; or &nbsp; <a href="https://github.com/zhlinh" class="btn-inverse">View on GitHub</a>
    </div><!-- /.page-lead-content -->
</div><!-- /.page-lead -->

# Latest Posts

<div class="tiles">
{% for post in site.posts %}
	{% include post-grid.html %}
{% endfor %}
</div><!-- /.tiles -->
