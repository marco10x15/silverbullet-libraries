---
name: "Library/MG/Mie_Configurazioni"
tags: meta/library
pageDecoration.prefix: "⚙️ "
---
# Configurazioni Personali dello spazio Silverbullet

## Nasconde frontmatter
Minimizza e quindi “nasconde” il frontmatter, che però rimane editabile.

Fonte: [Community](https://community.silverbullet.md/t/hiding-frontmatter/830?u=marco10x15)

```space-style
.sb-frontmatter.sb-line-frontmatter-outside:has(+ .sb-frontmatter) ~ .sb-frontmatter {
    display:none
}
```

## Imposta la larghezza della finestra di visualizzazione
Imposta la larghezza della finestra di visualizzazione per schermi più ampi.

```space-style
/* Imposta la larghezza della finestra di visualizzazioen a 1024 pixel */
html {
  /* Uncomment the next line to set the editor font to Courier */
  /* --editor-font: "Courier" !important; */
  /* Uncomment the next line to set the editor width to 1400px */
  --editor-width: 1024px !important; 
}
```
