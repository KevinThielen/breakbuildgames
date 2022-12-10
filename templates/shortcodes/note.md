{% if not kind %}
{% set kind = "Note" %}
{% endif %}


{% if not title %}
{% set title = kind %}
{% endif %}

<span class="admonition {{ kind | lower }} title"> {{ title }} </span>
<span class="admonition {{ kind | lower }}"> {{ body | safe }}  </span>
