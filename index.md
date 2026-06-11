---
layout: default
title: "TONE // AUDIO_LOG"
author: Anthony Klein
description: A music studio log covering deep house, underground DJ culture, home studio gear, production experiments, Lounge24 Radio, and self-hosted audio infrastructure.
keywords:
  - deep house DJ
  - underground electronic music
  - jungle music
  - music production workflows
  - DJ workflows
  - DJ studio setup
  - home studio
  - sound design
  - internet radio
  - self-hosted radio
  - Lounge24
  - AzuraCast
  - Rekordbox
  - Ableton
  - live streaming for DJs
---

**Welcome** to the digital annex of **Lounge24**. This is where I keep the notes behind the music, the gear, the radio side, and the studio build as it keeps evolving. My <a href="https://www.aklein.studio" target="_blank" rel="noopener noreferrer">main studio site</a> is still home base, but this is the side room where I break down what is shaping the sound.

---

| Category | Focus & Objectives |
| :--- | :--- |
| **Selections** | Curated tracklists, Jungle, and Deep House sets. |
| **Studio** | Documentation of my journey into production and hardware. |
| **Underground** | Notes on culture, gear, and sonic inspiration. |

---

### [ LATEST_TRANSMISSIONS ]

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url }}) — <small>{{ post.date | date: "%b %d, %Y" }}</small>
{% endfor %}


---
*This site is a lightweight, high-availability build hosted on **GitHub Pages** and powered by **Jekyll**, utilizing a GitOps-inspired workflow for rapid static deployment.*
