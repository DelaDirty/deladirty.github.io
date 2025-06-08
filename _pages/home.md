---
layout: splash
title: "DelaDirty’s Pentest Playbook"
permalink: /                 # makes this file the real homepage
hidden: true                 # hides the page title in the UI
header:
  overlay_color: "#000"
  overlay_filter: "0.4"
  overlay_image: /assets/images/banner.jpg   # replace or delete
  actions:
    - label: "About Me"
      url: "/about/"
# --- FEATURE CARDS GO HERE ---
feature_row:
  - image_path: /assets/images/shellcode.png
    alt: "Shellcode-Encryption"
    title: "Shellcode-Encryption"
    excerpt: "Encrypt C# shellcode with AES-256 or XOR to evade static AV."
    url: "https://github.com/DelaDirty/Shellcode-Encryption"
    btn_label: "GitHub"
    btn_class: "btn--primary"
  # add more cards with "- image_path: …" if you like
---
## Featured Work
{% include feature_row %}

## Latest Posts
{% include posts_list.html limit="5" %}