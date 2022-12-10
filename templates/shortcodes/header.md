
{%- set source = load_data(path=path) %}

<span class="code_with_header">
<span class="codeheader">{{name}}</span>

```rust
{{ source | safe }}
```

</span>
