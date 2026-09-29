---
tags: meta/template/page
description: "Crea una nuova pagina Diario"
version: "1.4-00"
versionDate: 2026-09-14
suggestedName: "Diario/${date.today()}"
confirmName: false
openIfExists: true
command: "Page: Nuova Pagina Diario"
priority: 30
frontmatter: |
  date: ${date.today()}
  displayName: ${date.today()}
  description:
  Viaggio:
  luoghi:
  tags:
---

# |^|
