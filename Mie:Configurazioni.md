---
name: "Library/MG/Mie_Configurazioni"
tags: meta/library
pageDecoration.prefix: "⚙️ "
---
# Configurazioni Personali dello spazio Silverbullet

## Hiding frontmatter
**Scopo**
Minimizza e quindi “nasconde” il frontmatter, che però rimane editabile.

Fonte: [Community](https://community.silverbullet.md/t/hiding-frontmatter/830?u=marco10x15)

```space-style
.sb-frontmatter.sb-line-frontmatter-outside:has(+ .sb-frontmatter) ~ .sb-frontmatter {
    display:none
}
```
