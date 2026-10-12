---
title: "Proyectos"
layout: archive
permalink: /proyectos/
classes: wide
author_profile: false
---

{% assign proyectos = site.proyectos | where: "kind", "proyecto" %}
<div class="grid__wrapper">
{% for post in proyectos %}
  {% include archive-single.html type="grid" %}
{% endfor %}
</div>
