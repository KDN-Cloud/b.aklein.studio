---
layout: default
title: Tone's Music 🎵 Blog
author: Anthony Klein
description: A technical log documenting workflows in audio production, sound design, and underground DJ sets.
---

Welcome to the technical annex of Lounge24. This space serves as a dedicated log for my journey through audio production, DJing, and sound design. While my [main studio site](https://www.aklein.studio) serves as the command center, this blog is where I document the workflows and inspirations behind the music.

This site is a lightweight, high-availability build hosted on GitHub Pages and powered by Jekyll, utilizing a GitOps-inspired workflow for rapid static deployment.

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url }}) — <small>{{ post.date | date: "%b %d, %Y" }}</small>
{% endfor %}

