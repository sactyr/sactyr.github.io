---
title: "Projects"
layout: "projects-list"
url: "/projects/"
draft: false
---

{{ with .Scratch.Get "finalTimeline" }}
{{ . | safeHTML }}
{{ else }}
Below is a list of my active public repositories, pulled directly from my GitHub profile.
{{ end }}