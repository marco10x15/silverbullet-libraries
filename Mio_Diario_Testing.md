---
name: "Library/MG/Mio_Diario_Testing"
tags: meta/library
description: "Widget, funzioni, in fase di sviluppo. ATTENZIONE! PERICOLO!"
version: "0.0-02"
versionDate: 2026-09-01
pageDecoration.prefix: "⚠️ "
share.uri: "github:marco10x15/silverbullet-libraries/Mio_Diario_Testing.md"
---

# ⚠️️⚠️ Funzioni Diario in Sviluppo ⚠⚠

**Versione:** 0.0-02 — 01.09.2026

La libreria è autonoma.

## Main Features

- `diarioImages(pageName)` — seleziona automaticamente le immagini associate a una pagina Diario.
- `collage()` — visualizza automaticamente le immagini della pagina corrente come collage responsive.
- `collage(images)` — visualizza un elenco esplicito di immagini dello Space.
- Il collegamento alla galleria viene generato automaticamente da `collage()`.
- Virtual Page `gallery:NomePagina` — visualizza le immagini della pagina Diario.
- La Virtual Page supporta sia la selezione automatica sia, quando l'elenco è scritto direttamente nella pagina, le immagini passate esplicitamente a `collage`.
- Convenzione dati automatica:
  - pagina Diario: `YYYY-MM-DD`
  - immagini: `media/YYYYMMDD...`
- Layout collage:
  - smartphone: 3 colonne
  - tablet: 4 colonne
  - desktop medio: 5 colonne
  - desktop largo: 6 colonne
- Nessuna modifica persistente delle pagine o dei file immagine.

## Configurazione

Configurazione minima, con selezione automatica delle immagini:

```space-lua
${collage()}
```

Configurazione con passaggio esplicito delle immagini:

```space-lua
${collage {
  "media/20260830-183349.jpeg",
  "media/20260830-183349 (1).jpeg",
  "media/20260830-183349 (2).jpeg"
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

### Selezione automatica

Per una pagina:

```text
Diario/2026-08-30
```

è sufficiente:

```space-lua
${collage()}
```

La funzione ricava:

```text
2026-08-30
        ↓
20260830
```

e seleziona dall'indice i documenti:

```text
media/20260830...
```

Il collage aggiunge automaticamente il link:

```text
📷 Galleria · N foto
```

che apre:

```text
gallery:Diario/2026-08-30
```

La Virtual Page utilizza la stessa pagina sorgente e ripete la selezione automatica.

### Passaggio esplicito delle immagini

È possibile definire esattamente quali immagini mostrare e in quale ordine:

```space-lua
${collage {
  "media/20260830-183349.jpeg",
  "media/20260830-183112 (2).jpeg",
  "media/20260830-183111.jpeg"
}}
```

In questo caso `collage(images)` non interroga l'indice: utilizza direttamente l'elenco ricevuto.

Il link automatico apre comunque:

```text
gallery:Diario/2026-08-30
```

La Virtual Page legge il Markdown della pagina sorgente e recupera l'elenco scritto nel blocco `${collage { ... }}`.

L'ordine delle immagini passate esplicitamente viene mantenuto.

### Virtual Page con immagini esplicite

La Virtual Page può quindi essere utilizzata anche quando le immagini sono passate esplicitamente, senza duplicare l'elenco.

Il funzionamento è:

```text
Pagina Diario
    ↓
${collage { "media/...", ... }}
    ↓
link automatico gallery:Pagina
    ↓
space.readPage(Pagina)
    ↓
estrazione dell'elenco dal blocco collage
    ↓
galleria completa
```

Vincolo della modalità esplicita:

la Virtual Page può ricostruire l'elenco solo quando i file sono scritti direttamente nel blocco:

```space-lua
${collage {
  "media/foto1.jpg",
  "media/foto2.jpg"
}}
```

Non può ricostruire automaticamente una lista creata dinamicamente, ad esempio:

```space-lua
${collage(miaLista)}
```

perché il valore runtime di `miaLista` non è persistito nel Markdown.

Se nella stessa pagina sono presenti più blocchi `${collage { ... }}`, la Virtual Page unisce le immagini dei blocchi nell'ordine in cui compaiono, eliminando eventuali duplicati.

### Ricerca automatica immagini

La modalità automatica usa la convenzione:

```text
Pagina:    Diario/YYYY-MM-DD
Immagini:  media/YYYYMMDD...
```

Esempio:

```text
Diario/2026-08-30
```

corrisponde a:

```text
media/20260830-183111.jpeg
media/20260830-183112.jpeg
media/20260830-183349.jpeg
```

Sono considerate immagini le estensioni:

- `.jpg`
- `.jpeg`
- `.png`
- `.webp`
- `.gif`

Il prefisso `YYYYMMDD` viene ricavato da una data `YYYY-MM-DD` posta all'inizio del nome della pagina.

Esempi validi:

```text
Diario/2026-08-30
Diario/2026-08-30-descrizione
Diario/2026-08-30 Escursione
```

Vincolo adottato:

**la convenzione `media/YYYYMMDD...` fa parte del modello dati del Diario.**

Limiti noti:

- un'immagine senza il prefisso data non viene inclusa nella selezione automatica;
- più pagine Diario con la stessa data iniziale condividono la stessa selezione automatica;
- qualsiasi immagine con lo stesso prefisso viene inclusa, anche se non è una fotografia;
- nella selezione automatica l'ordinamento è alfabetico sul path e, con nomi `YYYYMMDD-HHMMSS...`, coincide normalmente con l'ordine cronologico;
- nella modalità esplicita la Virtual Page riconosce gli elenchi scritti direttamente come `${collage { ... }}`;
- liste Lua costruite dinamicamente non sono ricostruibili dalla Virtual Page.

## Implementazione

```space-lua
local function diarioBasename(pageName)
  return (pageName or ""):match("([^/]+)$") or ""
end

local function diarioPrefix(pageName)
  local name = diarioBasename(pageName)
  local y, m, d = name:match("^(%d%d%d%d)%-(%d%d)%-(%d%d)")

  if not y then
    return nil
  end

  return y .. m .. d
end

local function isImageDocument(name)
  local lower = string.lower(name or "")

  return lower:match("%.jpg$")
    or lower:match("%.jpeg$")
    or lower:match("%.png$")
    or lower:match("%.webp$")
    or lower:match("%.gif$")
end

local function diarioImagesByPrefix(prefix)
  if not prefix then
    return {}
  end

  local images = {}

  for _, doc in ipairs(index.documents()) do
    local name = doc.name or ""

    if name:match("^media/" .. prefix) and isImageDocument(name) then
      table.insert(images, name)
    end
  end

  table.sort(images)

  return images
end

function diarioImages(pageName)
  pageName = pageName or editor.getCurrentPage()
  return diarioImagesByPrefix(diarioPrefix(pageName))
end

local function explicitCollageImages(pageName)
  local ok, text = pcall(function()
    return space.readPage(pageName)
  end)

  if not ok or not text then
    return {}
  end

  local images = {}
  local seen = {}

  for body in text:gmatch("%$%{%s*collage%s*%{(.-)%}%s*%}") do
    for path in body:gmatch('"(media/[^"]+)"') do
      if isImageDocument(path) and not seen[path] then
        table.insert(images, path)
        seen[path] = true
      end
    end
  end

  return images
end

local function galleryImages(pageName)
  local explicit = explicitCollageImages(pageName)

  if #explicit > 0 then
    return explicit
  end

  return diarioImages(pageName)
end

local function galleryLink(pageName, count)
  return dom.div {
    class = "photo-collage-link",
    "[[gallery:" .. pageName .. "|📷 Galleria · " .. count .. " foto]]"
  }
end

function collage(images)
  local pageName = editor.getCurrentPage()

  if images == nil then
    images = diarioImages(pageName)
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
      class = "photo-collage-wrapper",

      dom.div {
        class = "photo-collage",
        table.unpack(items)
      },

      galleryLink(pageName, #images)
    }
  )
end

local function galleryMarkdown(pageName)
  local images = galleryImages(pageName)

  local lines = {
    "# 📷 Galleria immagini",
    "",
    "[[" .. pageName .. "|← Torna alla pagina]]",
    "",
    "**" .. #images .. " immagini**",
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
.photo-collage-wrapper {
  width: 100%;
}

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

.photo-collage-link {
  margin-top: 6px;
  text-align: right;
  font-size: 0.9em;
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
