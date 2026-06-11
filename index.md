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

**Welcome** to the digital annex of **Lounge24**. This space serves as a dedicated archive for my journey through underground DJing, audio production, and sound design. While my [main studio site](https://www.aklein.studio) is the central command center, this archive is where I document the inspirations and creative workflows behind the sound.

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
