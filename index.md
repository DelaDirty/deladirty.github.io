---
layout: splash
title: ""
permalink: /          
hidden: true
header:
  overlay_color: "rgba(112, 171, 171, 0.32)"
  overlay_filter: 0.10
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
  - image_path: /assets/images/htb/monteverde/mv.png
    alt: "Monteverde"
    title: "Monteverde"
    excerpt: "A complete walkthrough of HTB’s Monteverde box covering AD enumeration, Azure abuse, and domain compromise."
    url: "https://deladirty.github.io/htb/monteverde/"
    btn_label: "Read Post"
    btn_class: "btn--primary"
---
### Featured Posts
{% include feature_row %}

{% for post in paginator.posts %}

{% endfor %}

{% if paginator.next_page %}
  <a href="{{ paginator.next_page_path }}" class="button">View More </a>
{% endif %}




### Recent Posts
{% for post in site.posts limit:3 %}
### [{{ post.title }}]({{ post.url | relative_url }})

<small>{{ post.date | date: "%-d %b %Y" }}</small>

{{ post.excerpt | strip_html | truncatewords: 40 }}

[Read More →]({{ post.url | relative_url }})

{% endfor %}





