---
---
{::options parse_block_html="true" /}
{% assign pages = site.pages | sort: 'date' | reverse %}
{% for page in pages %}
{% if page.title %}

<h1><a href="{{ page.url }}" style="text-decoration: none; color:inherit;">{{ page.title }}</a></h1>

{% if page.description %}
### {{ page.description }}
{% endif %}

{{ page.date | date_to_string }}

{% if forloop.last == false %}
&nbsp;
{% endif %}

{% endif %}
{% endfor %}
