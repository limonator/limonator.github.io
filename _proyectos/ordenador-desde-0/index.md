---
title: "Construcción de un ordenador desde 0"
permalink: /proyectos/ordenador-desde-0
kind: proyecto
excerpt: "¡Estudio, analisis y cacharreo con el clásico procesador 6502!"
header:
    teaser: /assets/images/proyects/6502/top-6502.jpeg
---

Probando probando...

{% assign notes = site.proyectos | where: "project", "ordenador-desde-0" | where: "kind", "note" | sort: "order" %}
<ul>
{% for n in notes %}
  <li><a href="{{ n.url | relative_url }}">{{ n.title }}</a></li>
{% endfor %}
</ul>
