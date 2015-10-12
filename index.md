---
layout: home
permalink: /
image:
  feature: wood-texture-1600x800.jpg
---

<div class="tiles">

<div class="tile">
  <h2 class="post-title">Humanity</h2>
  <p class="post-excerpt">
    Had I the heaven's embroidered cloths, Inwrought with gold and silver light.   
  </p>
</div><!-- /.tile -->

<div class="tile">
  <h2 class="post-title">Technology</h2>
  <p class="post-excerpt">
    The blue and the dim and the dark cloths Of night and light and the half-light, I would spread the cloths under your feet.
  </p>
</div><!-- /.tile -->

<div class="tile">
  <h2 class="post-title">Day dreams</h2>
  <p class="post-excerpt">
    But I, being poor, have only my dreams.
    I have spread my dreams under your feet. Tread softly because you tread on my dreams.
  </p>
</div><!-- /.tile -->

<div class="tile">
  <h2 class="post-title">Inspired</h2>
  <p class="post-excerpt">
    And every day, everywhere, our children spread their dreams beneath our feet. And we should tread softly. 
  </p>
</div><!-- /.tile -->

</div><!-- /.tiles -->
<div class="tiles">
{% for post in site.posts %}
	{% include post-grid.html %}
{% endfor %}
</div><!-- /.tiles -->
