---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
style: /assets/css/SCP-bar.css
---

{% include SCP-bar.html %}

<br />

# Blog Sections

{% for blog_category in site.blog_categories %}
* [{{ blog_category.name }}]({{ blog_category.url }})
{% endfor %}