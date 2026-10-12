---
title: "Construcción de un ordenador desde 0"
permalink:/proyectos/orndeador-desde-0
kind: proyect
excerpt: "¡Estudio, analisis y cacharreo con el clásico procesador 6502!"
---

# Introducción 
Probando probando...

{% assign notes = site.proyectos | where: "project", "ordenador-desde-0" | where: "kind", "note" | sort: "order" %}
<ul>
{% for n in notes %}
  <li><a href="{{ n.url | relative_url }}">{{ n.title }}</a></li>
{% endfor %}
</ul>
