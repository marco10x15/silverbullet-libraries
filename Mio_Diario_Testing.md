---
name: "Library/MG/Mio_Diario_Testing"
tags: meta/library
description: "Widget, funzioni, in fase di sviluppo. ATTENZIONE! PERICOLO!"
version: "0.0-01"
versionDate: 2026-09-01
pageDecoration.prefix: "⚠️ "
share.uri: "github:marco10x15/silverbullet-libraries/Mio_Diario_Testing.md"
---

# ⚠️️⚠️ Funzioni Diario in Sviluppo ⚠⚠

**Versione:** 0.0-01 — 01.09.2026

La libreria è autonoma.

## Main Features

- `diarioImages(pageName)` — seleziona automaticamente le immagini associate a una pagina Diario.
- `collage()` — visualizza automaticamente le immagini della pagina corrente come collage responsive.
- `collage(images)` — visualizza un elenco esplicito di immagini dello Space.
- Virtual Page `gallery:NomePagina` — visualizza le immagini associate alla pagina Diario a dimensione completa.
- Convenzione dati: le immagini del Diario sono memorizzate sotto `media/` e iniziano con il prefisso `YYYYMMDD`.
- Layout collage:
  - smartphone: 3 colonne
  - tablet: 4 colonne
  - desktop medio: 5 colonne
  - desktop largo: 6 colonne
- Nessuna modifica persistente delle pagine o dei file immagine.

## Configurazione

Configurazione minima consigliata in una pagina Diario:

```space-lua
${collage()}
```

Configurazione manuale:

```space-lua
${collage {
  "media/foto1.jpg",
  "media/foto2.jpg",
  "media/foto3.jpg"
}}
```

Il layout utilizza i valori definiti nello Space Style:

- smartphone: 3 colonne
- tablet: 4 colonne
- desktop medio: 5 colonne
- desktop largo: 6 colonne
- altezza immagini:
  - smartphone: `88px`
  - tablet: `110px`
  - desktop medio: `140px`
  - desktop largo: `160px`
- spazio tra immagini: `4px`
- adattamento immagine: `object-fit: cover`

## Uso

### Collage automatico

Per una pagina, ad esempio:

```text
Diario/20260830-descrizione
```

la funzione:

```space-lua
${collage()}
```

ricava il prefisso:

```text
20260830
```

e seleziona automaticamente i documenti:

```text
media/20260830...
```

### Collage manuale

```space-lua
${collage {
  "media/20260830-183349.jpeg",
  "media/20260830-183349 (1).jpeg",
  "media/20260830-183349 (2).jpeg"
}}
```

### Virtual Page

Link alla galleria completa:

```markdown
[[gallery:Diario/20260830-descrizione|📷 Galleria immagini]]
```

La Virtual Page:

```text
gallery:Diario/20260830-descrizione
```

ricava nuovamente il prefisso `20260830`, usa `diarioImages()` e visualizza tutte le immagini associate alla pagina.

### Ricerca immagini

La selezione automatica usa la convenzione:

```text
media/YYYYMMDD...
```

Sono considerate immagini le estensioni:

- `.jpg`
- `.jpeg`
- `.png`
- `.webp`
- `.gif`

Il prefisso `YYYYMMDD` viene ricavato dalle prime 8 cifre del nome della pagina.

Esempi validi:

```text
Diario/20260830
Diario/20260830-descrizione
Diario/20260830 Escursione
```

Vincolo adottato:

**la convenzione `media/YYYYMMDD...` fa parte del modello dati del Diario.**

Limiti noti:

- un'immagine senza il prefisso data non viene inclusa;
- più pagine Diario con lo stesso prefisso `YYYYMMDD` condividono le stesse immagini;
- qualsiasi immagine con lo stesso prefisso viene inclusa, anche se non è una fotografia;
- l'ordinamento è alfabetico sul path e, con nomi `YYYYMMDD-HHMMSS...`, coincide normalmente con l'ordine cronologico.

## Implementazione

```space-lua
local function diarioBasename(pageName)
  return (pageName or ""):match("([^/]+)$") or ""
end

local function diarioPrefix(pageName)
  return diarioBasename(pageName):match("^(%d%d%d%d%d%d%d%d)")
end

local function isImageDocument(name)
  local lower = string.lower(name or "")

  return lower:match("%.jpg$")
    or lower:match("%.jpeg$")
    or lower:match("%.png$")
    or lower:match("%.webp$")
    or lower:match("%.gif$")
end

local function sortByName(items)
  table.sort(items, function(a, b)
    return a < b
  end)

  return items
end

function diarioImages(pageName)
  pageName = pageName or editor.getCurrentPage()

  local prefix = diarioPrefix(pageName)
  if not prefix then
    return {}
  end

  local images = {}

  for _, doc in ipairs(index.documents()) do
    local name = doc.name or doc.ref or ""

    if name:match("^media/" .. prefix) and isImageDocument(name) then
      table.insert(images, name)
    end
  end

  return sortByName(images)
end

function collage(images)
  if images == nil then
    images = diarioImages()
  end

  if not images or #images == 0 then
    return widget.markdown("_Nessuna immagine trovata._")
  end

  local items = {}

  for _, path in ipairs(images) do
    table.insert(items,
      dom.div {
        class = "photo-collage-item",
        "![[" .. path .. "]]"
      }
    )
  end

  return widget.htmlBlock(
    dom.div {
      class = "photo-collage",
      table.unpack(items)
    }
  )
end

local function galleryMarkdown(pageName)
  local images = diarioImages(pageName)

  local lines = {
    "# 📷 Galleria immagini",
    "",
    "[[" .. pageName .. "|← Torna alla pagina]]",
    ""
  }

  if #images == 0 then
    table.insert(lines, "_Nessuna immagine trovata._")
    return table.concat(lines, "\n")
  end

  for _, path in ipairs(images) do
    table.insert(lines, "![[" .. path .. "]]")
    table.insert(lines, "")
  end

  return table.concat(lines, "\n")
end

virtualPage.define {
  pattern = "gallery:(.+)",
  run = function(pageName)
    return galleryMarkdown(pageName)
  end
}
```

```space-style
.photo-collage {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 4px;
  width: 100%;
}

.photo-collage-item {
  height: 88px;
  overflow: hidden;
  border-radius: 4px;
}

.photo-collage-item p {
  margin: 0;
  width: 100%;
  height: 100%;
}

.photo-collage-item img {
  display: block;
  width: 100%;
  height: 100%;
  object-fit: cover;
}

@media (min-width: 601px) {
  .photo-collage {
    grid-template-columns: repeat(4, minmax(0, 1fr));
  }

  .photo-collage-item {
    height: 110px;
  }
}

@media (min-width: 901px) {
  .photo-collage {
    grid-template-columns: repeat(5, minmax(0, 1fr));
  }

  .photo-collage-item {
    height: 140px;
  }
}

@media (min-width: 1201px) {
  .photo-collage {
    grid-template-columns: repeat(6, minmax(0, 1fr));
  }

  .photo-collage-item {
    height: 160px;
  }
}
```
