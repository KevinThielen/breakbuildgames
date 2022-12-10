{% if extra %}
{% set additions = "," ~ extra %}
{% else %}
{% set additions = "" %}
{% endif %}



{% if not language %}
{% set language = "rust" %}
{% endif %}





{% if not name and path %}
{% set name = path | split(pat="/") | last %}
{% endif %}


{% if not name %}
{% set name = language  %}
{% endif %}



{% if path %}
{% set full_path = "content/" ~ path %}
{% set source = load_data(path=full_path) %}

<div class="code_with_header">
<div class="codeheader">{{name}}</div>

```{{ language }}{{additions}}
{{ source | safe}}
```

</div>

{% else %}

<div class="code_with_header">
<div class="codeheader">{{name}}</div>

{{ source }}

</div>

{% endif %}
