---
name: "Library/MG/Mio_Diario_PhotoGallery"
tags: meta/library
description: "Collage fotografico responsive e viewer Virtual Page per le immagini del Diario tramite Synology Photos e PhotoGateway."
version: "0.2-01"
versionDate: 2026-09-03
pageDecoration.prefix: "📷 "
share.uri: "github:marco10x15/silverbullet-libraries/Mio_Diario_Photo_Gallery.md"
---

# Mio Diario — Photo Gallery

Libreria Space Lua per SilverBullet v2 che visualizza le fotografie archiviate in Synology Photos tramite PhotoGateway.

La versione **0.2-01** corregge il rendering del viewer introdotto nella 0.2-00: la Virtual Page non costruisce più HTML come stringa, ma delega il rendering a `widget.htmlBlock()` e `dom.*`, come già avviene nel collage.

## Architettura

Listing:

```text
SilverBullet Space Lua
    ↓ net.proxyFetch()
PhotoGateway 172.30.50.10:8080
    ↓
DSM FileStation.List
```

Rendering immagini:

```text
Browser
    ↓
https://sb2.fm-nas.net/diario/.proxy/172.30.50.10:8080/thumb
    ↓
PhotoGateway
    ↓
DSM FileStation.Thumb
```

Viewer:

```text
Collage
    ↓ click
photo:YYYY-MM-DD:N
    ↓
Virtual Page SilverBullet
    ↓
${photoGalleryViewer(...)}
    ↓
widget.htmlBlock() + dom.*
```

## Uso base

Su:

```text
Diario/2026-08-12
```

inserire:

```lua
${photoGallery()}
```

## Data esplicita

```lua
${photoGallery("2026-08-12")}
```

## Limitare le immagini nel collage

```lua
${photoGallery(nil, {maxImages = 30})}
```

## Diagnostica

```lua
${photoGalleryInfo()}
```

## Viewer Virtual Page

Formato:

```text
photo:YYYY-MM-DD:N
```

Esempio:

```text
photo:2026-08-06:1
```

La Virtual Page è read-only e non crea file fisici.

---

# Implementazione

```space-lua
-- ============================================================
-- Mio Diario PhotoGallery 0.2-01
-- SilverBullet v2 / Space Lua
-- ============================================================

photoGalleryConfig = photoGalleryConfig or {
  gatewayBase = "http://172.30.50.10:8080",

  -- Necessario URL assoluto per il rendering immagini nel browser.
  proxyBase = "https://sb2.fm-nas.net/diario/.proxy/172.30.50.10:8080",

  diaryPagePrefix = "Diario/",
  viewerPrefix = "photo:",

  defaultThumbSize = "medium",
  defaultOpenSize = "large",

  maxImages = nil,
}


-- ============================================================
-- UTILITY
-- ============================================================

local function photoGalleryExtractDate(pageName)
  if type(pageName) ~= "string" then
    return nil
  end

  local y, m, d =
    string.match(pageName, "(%d%d%d%d)%-(%d%d)%-(%d%d)$")

  if not y then
    return nil
  end

  return y .. "-" .. m .. "-" .. d
end


local function photoGalleryCurrentDate()
  return photoGalleryExtractDate(editor.getCurrentPage())
end


local function photoGalleryReplaceSize(thumbPath, size)
  if type(thumbPath) ~= "string" then
    return nil
  end

  return string.gsub(
    thumbPath,
    "([?&]size=)[^&]+",
    "%1" .. size
  )
end


local function photoGalleryError(message)
  return widget.htmlBlock(
    dom.div {
      class = "photo-gallery-message photo-gallery-error",
      message,
    }
  )
end


local function photoGalleryEmpty(message)
  return widget.htmlBlock(
    dom.div {
      class = "photo-gallery-message photo-gallery-empty",
      message,
    }
  )
end


local function photoGalleryFetch(date)
  local response = net.proxyFetch(
    photoGalleryConfig.gatewayBase ..
    "/gallery/" ..
    date
  )

  if not response then
    return nil, "nessuna risposta da PhotoGateway"
  end

  if not response.ok then
    return nil,
      "PhotoGateway HTTP " ..
      tostring(response.status or "?")
  end

  if type(response.body) ~= "table" then
    return nil, "risposta PhotoGateway non valida"
  end

  return response.body, nil
end


-- ============================================================
-- COLLAGE
-- ============================================================

function photoGallery(date, options)
  options = options or {}

  date = date or photoGalleryCurrentDate()

  if not date then
    return photoGalleryError(
      "PhotoGallery: impossibile ricavare una data YYYY-MM-DD dalla pagina corrente."
    )
  end

  local thumbSize =
    options.thumbSize or
    photoGalleryConfig.defaultThumbSize

  local maxImages = options.maxImages

  if maxImages == nil then
    maxImages = photoGalleryConfig.maxImages
  end

  local data, err = photoGalleryFetch(date)

  if not data then
    return photoGalleryError(
      "PhotoGallery: " .. tostring(err)
    )
  end

  if data.exists == false then
    return photoGalleryEmpty(
      "Nessuna directory fotografica per " ..
      date ..
      "."
    )
  end

  local images = data.images

  if type(images) ~= "table" or #images == 0 then
    return photoGalleryEmpty(
      "Nessuna fotografia per " ..
      date ..
      "."
    )
  end

  local children = {
    class = "photo-gallery",
  }

  local shown = 0

  for index, image in ipairs(images) do
    if not maxImages or shown < maxImages then
      local thumbPath =
        photoGalleryReplaceSize(
          image.thumb,
          thumbSize
        )

      if thumbPath then
        local thumbUrl =
          photoGalleryConfig.proxyBase ..
          thumbPath

        local viewerPage =
          photoGalleryConfig.viewerPrefix ..
          date ..
          ":" ..
          tostring(index)

        table.insert(
          children,
          dom.a {
            class = "photo-gallery-item",
            href = viewerPage,
            title = image.name or "",

            dom.img {
              class = "photo-gallery-image",
              src = thumbUrl,
              alt = image.name or "",
              loading = "lazy",
              decoding = "async",
            },
          }
        )

        shown = shown + 1
      end
    end
  end

  if shown == 0 then
    return photoGalleryEmpty(
      "Nessuna fotografia visualizzabile per " ..
      date ..
      "."
    )
  end

  return widget.htmlBlock(
    dom.div(children)
  )
end


-- ============================================================
-- DIAGNOSTICA
-- ============================================================

function photoGalleryInfo(date)
  date = date or photoGalleryCurrentDate()

  if not date then
    return "PhotoGallery: data non disponibile"
  end

  local data, err =
    photoGalleryFetch(date)

  if not data then
    return "PhotoGallery: " ..
      tostring(err)
  end

  return "PhotoGallery " ..
    date ..
    ": " ..
    tostring(data.count or 0) ..
    " immagini"
end


-- ============================================================
-- VIEWER DOM
-- ============================================================

function photoGalleryViewer(date, index)
  index = tonumber(index)

  local data, err = photoGalleryFetch(date)

  if not data then
    return photoGalleryError(
      "PhotoGallery: " .. tostring(err)
    )
  end

  local images = data.images or {}
  local total = #images

  if total == 0 then
    return photoGalleryEmpty(
      "Nessuna fotografia per " .. date .. "."
    )
  end

  if not index or index < 1 or index > total then
    return photoGalleryError(
      "Indice fotografia non valido."
    )
  end

  local image = images[index]

  local largePath =
    photoGalleryReplaceSize(
      image.thumb,
      photoGalleryConfig.defaultOpenSize
    )

  if not largePath then
    return photoGalleryError(
      "Thumbnail non disponibile."
    )
  end

  local largeUrl =
    photoGalleryConfig.proxyBase ..
    largePath

  local diaryPage =
    photoGalleryConfig.diaryPagePrefix ..
    date

  local previousPage = nil
  local nextPage = nil

  if index > 1 then
    previousPage =
      photoGalleryConfig.viewerPrefix ..
      date ..
      ":" ..
      tostring(index - 1)
  end

  if index < total then
    nextPage =
      photoGalleryConfig.viewerPrefix ..
      date ..
      ":" ..
      tostring(index + 1)
  end

  local children = {
    class = "photo-viewer",
  }


  -- Toolbar
  local toolbar = {
    class = "photo-viewer-toolbar",
  }

  table.insert(
    toolbar,
    dom.a {
      class = "photo-viewer-back",
      href = diaryPage,
      "← Diario",
    }
  )

  table.insert(
    toolbar,
    dom.span {
      class = "photo-viewer-counter",
      tostring(index) ..
      " / " ..
      tostring(total),
    }
  )

  table.insert(
    toolbar,
    dom.span {
      class = "photo-viewer-toolbar-spacer",
      "",
    }
  )

  table.insert(
    children,
    dom.div(toolbar)
  )


  -- Immagine
  local imageNode =
    dom.img {
      class = "photo-viewer-image",
      src = largeUrl,
      alt = image.name or "",
    }

  local stageContent

  if nextPage then
    stageContent =
      dom.a {
        class = "photo-viewer-image-link",
        href = nextPage,
        imageNode,
      }
  else
    stageContent = imageNode
  end

  table.insert(
    children,
    dom.div {
      class = "photo-viewer-stage",
      stageContent,
    }
  )


  -- Nome file
  table.insert(
    children,
    dom.div {
      class = "photo-viewer-filename",
      image.name or "",
    }
  )


  -- Navigazione
  local navigation = {
    class = "photo-viewer-navigation",
  }

  if previousPage then
    table.insert(
      navigation,
      dom.a {
        class = "photo-viewer-nav photo-viewer-prev",
        href = previousPage,
        "← Precedente",
      }
    )
  else
    table.insert(
      navigation,
      dom.span {
        class = "photo-viewer-nav photo-viewer-disabled",
        "← Precedente",
      }
    )
  end

  table.insert(
    navigation,
    dom.a {
      class = "photo-viewer-nav photo-viewer-close",
      href = diaryPage,
      "Chiudi",
    }
  )

  if nextPage then
    table.insert(
      navigation,
      dom.a {
        class = "photo-viewer-nav photo-viewer-next",
        href = nextPage,
        "Successiva →",
      }
    )
  else
    table.insert(
      navigation,
      dom.span {
        class = "photo-viewer-nav photo-viewer-disabled",
        "Successiva →",
      }
    )
  end

  table.insert(
    children,
    dom.div(navigation)
  )

  return widget.htmlBlock(
    dom.div(children)
  )
end


-- ============================================================
-- VIRTUAL PAGE
-- ============================================================

virtualPage.define {
  pattern = "photo:(%d%d%d%d%-%d%d%-%d%d):(%d+)",

  run = function(date, index)
    return '${photoGalleryViewer("' ..
      date ..
      '", ' ..
      tostring(index) ..
      ')}'
  end
}
```

```space-style
/* ============================================================
   Mio Diario PhotoGallery 0.2-01
   ============================================================ */


/* COLLAGE */

.photo-gallery {
  display: grid;
  grid-template-columns: repeat(6, minmax(0, 1fr));
  gap: 6px;
  width: 100%;
  margin: 0.5rem 0 1rem 0;
}

.photo-gallery-item {
  display: block;
  position: relative;
  min-width: 0;
  overflow: hidden;
  border-radius: 4px;
  text-decoration: none;
  aspect-ratio: 4 / 3;
}

.photo-gallery-image {
  display: block;
  width: 100%;
  height: 100%;
  object-fit: cover;
  border: 0;
  margin: 0;
  padding: 0;
  transition: transform 120ms ease;
}

.photo-gallery-item:hover .photo-gallery-image {
  transform: scale(1.025);
}

.photo-gallery-message {
  padding: 0.65rem 0.8rem;
  margin: 0.5rem 0;
  border-radius: 4px;
  font-size: 0.95em;
}

.photo-gallery-empty {
  opacity: 0.75;
}

.photo-gallery-error {
  font-weight: 600;
}


/* VIEWER */

.photo-viewer {
  width: 100%;
  max-width: 1400px;
  margin: 0 auto;
}

.photo-viewer-toolbar {
  display: grid;
  grid-template-columns: 1fr auto 1fr;
  align-items: center;
  gap: 1rem;
  margin: 0 0 0.75rem 0;
}

.photo-viewer-back {
  justify-self: start;
  text-decoration: none;
}

.photo-viewer-counter {
  justify-self: center;
  font-variant-numeric: tabular-nums;
  opacity: 0.8;
}

.photo-viewer-toolbar-spacer {
  justify-self: end;
}

.photo-viewer-stage {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
  min-height: 50vh;
  padding: 8px;
  box-sizing: border-box;
  background: rgba(0, 0, 0, 0.92);
  border-radius: 6px;
  overflow: hidden;
}

.photo-viewer-image-link {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
  text-decoration: none;
}

.photo-viewer-image {
  display: block;
  max-width: 100%;
  max-height: 78vh;
  width: auto;
  height: auto;
  object-fit: contain;
  border: 0;
  margin: 0;
  padding: 0;
}

.photo-viewer-filename {
  margin-top: 0.5rem;
  text-align: center;
  font-size: 0.85em;
  opacity: 0.7;
  overflow-wrap: anywhere;
}

.photo-viewer-navigation {
  display: grid;
  grid-template-columns: 1fr auto 1fr;
  align-items: center;
  gap: 1rem;
  margin-top: 0.75rem;
}

.photo-viewer-nav {
  text-decoration: none;
}

.photo-viewer-prev {
  justify-self: start;
}

.photo-viewer-close {
  justify-self: center;
}

.photo-viewer-next {
  justify-self: end;
}

.photo-viewer-disabled {
  opacity: 0.3;
  pointer-events: none;
}

.photo-viewer-navigation .photo-viewer-disabled:last-child {
  justify-self: end;
}


/* RESPONSIVE */

@media (max-width: 900px) {
  .photo-gallery {
    grid-template-columns: repeat(4, minmax(0, 1fr));
  }

  .photo-viewer-image {
    max-height: 75vh;
  }
}

@media (max-width: 600px) {
  .photo-gallery {
    grid-template-columns: repeat(3, minmax(0, 1fr));
    gap: 4px;
  }

  .photo-viewer-toolbar {
    gap: 0.5rem;
  }

  .photo-viewer-stage {
    min-height: 40vh;
    padding: 4px;
  }

  .photo-viewer-image {
    max-height: 70vh;
  }

  .photo-viewer-navigation {
    gap: 0.4rem;
    font-size: 0.9em;
  }

  .photo-viewer-filename {
    font-size: 0.75em;
  }
}
```

---

# Test 0.2-01

## 1. Viewer diretto

Aprire:

```text
photo:2026-08-06:1
```

Atteso:

- immagine `large`;
- contatore `1 / 9`;
- nome file sotto l'immagine;
- navigazione sotto lo stage nero;
- nessun testo sovrapposto alla fotografia.

## 2. Navigazione

Verificare:

```text
photo:2026-08-06:2
```

Atteso:

- precedente attivo;
- successiva attiva;
- `Chiudi` ritorna a `Diario/2026-08-06`.

## 3. Click sull'immagine

Da una fotografia non finale, fare click sull'immagine.

Atteso:

- fotografia successiva.

## 4. Compatibilità collage

Su:

```text
Diario/2026-08-06
```

usare:

```lua
${photoGallery()}
```

Atteso:

- collage invariato;
- click su miniatura → Virtual Page viewer.

---

# Changelog

## 0.2-01 — 2026-09-03

- corretto rendering del viewer Virtual Page;
- eliminata costruzione HTML manuale tramite stringhe;
- viewer ricostruito con `widget.htmlBlock()` e `dom.*`;
- mantenuto URL assoluto HTTPS per il proxy immagini;
- Virtual Page ridotta a router verso `photoGalleryViewer()`;
- mantenuta compatibilità con tutte le chiamate 0.1/0.2.

## 0.2-00 — 2026-09-03

- aggiunto viewer Virtual Page;
- aggiunta navigazione precedente/successiva;
- aggiunto contatore;
- aggiunto ritorno al Diario.

## 0.1-00 — 2026-09-01

- prima versione basata su PhotoGateway;
- collage responsive;
- thumbnail Synology;
- diagnostica `photoGalleryInfo()`.
