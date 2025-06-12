---
layout: splash
title: ""
permalink: /          
hidden: true
header:
  overlay_filter: rgba(110, 163, 173, 0.3)
feature_row:
  - image_path: /assets/images/shellcode.png
    alt: "Shellcode-Encryption For OSEP"
    title: "Shellcode-Encryption For OSEP"
    excerpt: "Encrypt C# shellcode with AES-256 or XOR to evade static AV."
    url: "https://deladirty.github.io/blog/encryption/"
    btn_label: "Read Post"
    btn_class: "btn--primary"
  - image_path: /assets/images/oscp2.jpg
    alt: "OSCP Journey"
    title: "My OSCP journey"
    excerpt: "Lessons, failures, and practical tips for the OSCP."
    url: "https://deladirty.github.io/blog/oscp/"
    btn_label: "Read Post"
    btn_class: "btn--primary"
  - image_path: /assets/images/htb/administrator/admin.jpg
    alt: "Administrator"
    title: "Administrator"
    excerpt: "A full walkthrough of HTB’s Administrator box covering AD enumeration, DACL abuse, and domain compromise."
    url: "https://deladirty.github.io/htb/Administrator/"
    btn_label: "Read Post"
    btn_class: "btn--primary"
---
### Featured Posts
{% include feature_row %}




### Recent Posts
{% for post in site.posts limit:3 %}
### [{{ post.title }}]({{ post.url | relative_url }})

<small>{{ post.date | date: "%-d %b %Y" }}</small>

{{ post.excerpt | strip_html | truncatewords: 40 }}

[Read More →]({{ post.url | relative_url }})

{% endfor %}





