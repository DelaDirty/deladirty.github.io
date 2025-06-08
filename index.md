---
layout: splash
title: "Featured Posts"
permalink: /          
hidden: true
header:
  overlay_color: "#a28089"
  overlay_filter: "0.4"
feature_row:
  - image_path: /assets/images/shellcode.png
    alt: "Shellcode-Encryption"
    title: "Shellcode-Encryption"
    excerpt: "Encrypt C# shellcode with AES-256 or XOR to evade static AV."
    url: "https://deladirty.github.io/blog/encryption/"
    btn_label: "Read Post"
    btn_class: "btn--primary"
---

{% include feature_row type="left" %}



### Recent Posts
{% for post in site.posts limit:3 %}
### [{{ post.title }}]({{ post.url | relative_url }})

<small>{{ post.date | date: "%-d %b %Y" }}</small>

{{ post.excerpt | strip_html | truncatewords: 40 }}

[Read More →]({{ post.url | relative_url }})

{% endfor %}





