---
name: "Library/MG/Mio_Diario_PhotoGallery"
tags: meta/library
description: "Gallery fotografica automatica del Diario con collage responsive, PhotoGateway e viewer Virtual Page."
version: "0.5-01"
versionDate: 2026-09-03
pageDecoration.prefix: "📷 "
share.uri: "github:marco10x15/silverbullet-libraries/Mio_Diario_Photo_Gallery.md"
---

# Mio Diario — Photo Gallery

Versione **0.5-01 TEST**.

Questa versione mantiene la gallery automatica già validata e semplifica
radicalmente il viewer:

- la **Virtual Page** contiene solo i comandi di navigazione e le informazioni;
- la fotografia viene renderizzata tramite **`hooks:renderBottomWidgets`**;
- nessun top widget per il viewer;
- nessun tentativo di nascondere parti di CodeMirror;
- nessuna Virtual Page vuota;
- nessun placeholder residuo;
- nessuna scrollbar interna nel viewer grazie a un'altezza immagine compatibile
  con il limite nativo dei bottom widget SilverBullet;
- altezza viewer desktop aumentata a 470 px.

---

# Architettura

## Gallery del Diario

```text
Diario/YYYY-MM-DD
    ↓
hooks:renderBottomWidgets
    ↓
photoGalleryAutoBottom()
    ↓
collage
```

## Viewer fotografia

```text
photo:YYYY-MM-DD:N
    │
    ├── Virtual Page
    │     └── navigazione + indice + ora
    │
    └── hooks:renderBottomWidgets
          └── fotografia singola
```

---

# Gallery automatica

Viene visualizzata solo su:

```text
Diario/YYYY-MM-DD
```

Per disabilitarla:

```yaml
PhotoGallery: false
```

La chiamata manuale resta disponibile:

```lua
${photoGallery()}
```

---

# Dimensionamento automatico Gallery

Obiettivo:

- massimo **4 righe** senza scrollbar;
- colonne minime: **6**;
- colonne massime: **10**;
- altezza miniature calcolata in funzione delle righe;
- oltre 40 immagini non vengono ulteriormente ridotte:
  si accetta lo scrolling del bottom widget.

Thumbnail Synology:

- fino a 24 immagini: `medium`;
- oltre 24 immagini: `small`;
- `thumbSize` esplicito mantiene la precedenza.

---

# Viewer

La Virtual Page contiene solo una riga di controllo:

```text
← Precedente · ← Diario · 4 / 35 · 09:24:56 · Successiva →
```

La fotografia è visualizzata sotto, nel bottom widget.

Il nome file non viene mostrato.

L'ora viene ricavata, quando possibile, da filename nel formato:

```text
YYYYMMDD-HHMMSS
YYYYMMDD_HHMMSS
```

Esempio:

```text
20260830-092456.JPG
```

→

```text
09:24:56
```

---

# Implementazione

```space-lua
-- ============================================================
-- Mio Diario PhotoGallery 0.5-01 TEST
-- SilverBullet v2
-- ============================================================

photoGalleryConfig = photoGalleryConfig or {
  gatewayBase =
    "http://172.30.50.10:8080",

  proxyBase =
    "https://sb2.fm-nas.net/diario/.proxy/172.30.50.10:8080",

  diaryPagePrefix =
    "Diario/",

  viewerPrefix =
    "photo:",

  defaultThumbSize =
    "medium",

  defaultOpenSize =
    "large",

  -- Gallery automatica
  galleryMaxVisibleRows = 4,
  galleryMinColumns = 6,
  galleryMaxColumns = 10,
  galleryUsableHeight = 430,
  galleryGap = 5,
  galleryMaxThumbHeight = 140,

  -- Viewer
  viewerImageMaxHeight = 470,

  maxImages =
    nil,
}


-- ============================================================
-- UTILITY
-- ============================================================

local function photoGalleryExtractDate(pageName)
  if type(pageName) ~= "string" then
    return nil
  end

  local y, m, d =
    string.match(
      pageName,
      "(%d%d%d%d)%-(%d%d)%-(%d%d)$"
    )

  if not y then
    return nil
  end

  return
    y .. "-" ..
    m .. "-" ..
    d
end


local function photoGalleryDiaryDate(pageName)
  if type(pageName) ~= "string" then
    return nil
  end

  return string.match(
    pageName,
    "^Diario/(%d%d%d%d%-%d%d%-%d%d)$"
  )
end


local function photoGalleryParseViewerPage(pageName)
  if type(pageName) ~= "string" then
    return nil, nil
  end

  local date, indexText =
    string.match(
      pageName,
      "^photo:(%d%d%d%d%-%d%d%-%d%d):(%d+)$"
    )

  if not date or not indexText then
    return nil, nil
  end

  return date, tonumber(indexText)
end


local function photoGalleryCurrentDate()
  return photoGalleryExtractDate(
    editor.getCurrentPage()
  )
end


local function photoGalleryReplaceSize(
  thumbPath,
  size
)
  if type(thumbPath) ~= "string" then
    return nil
  end

  return string.gsub(
    thumbPath,
    "([?&]size=)[^&]+",
    "%1" .. size
  )
end


local function photoGalleryFetch(date)
  local response =
    net.proxyFetch(
      photoGalleryConfig.gatewayBase ..
      "/gallery/" ..
      date
    )

  if not response then
    return nil,
      "nessuna risposta da PhotoGateway"
  end

  if not response.ok then
    return nil,
      "PhotoGateway HTTP " ..
      tostring(response.status or "?")
  end

  if type(response.body) ~= "table" then
    return nil,
      "risposta PhotoGateway non valida"
  end

  return response.body, nil
end


local function photoGalleryError(message)
  return widget.htmlBlock(
    dom.div {
      class =
        "photo-gallery-message photo-gallery-error",

      message,
    }
  )
end


local function photoGalleryEmpty(message)
  return widget.htmlBlock(
    dom.div {
      class =
        "photo-gallery-message photo-gallery-empty",

      message,
    }
  )
end


local function photoGalleryShotTime(fileName)
  if type(fileName) ~= "string" then
    return nil
  end

  local hh, mm, ss =
    string.match(
      fileName,
      "^%d%d%d%d%d%d%d%d[-_](%d%d)(%d%d)(%d%d)"
    )

  if not hh then
    return nil
  end

  local h =
    tonumber(hh)

  local m =
    tonumber(mm)

  local s =
    tonumber(ss)

  if not h or
     not m or
     not s or
     h > 23 or
     m > 59 or
     s > 59 then
    return nil
  end

  return
    hh .. ":" ..
    mm .. ":" ..
    ss
end


local function photoGalleryLayout(imageCount)
  local maxRows =
    photoGalleryConfig.galleryMaxVisibleRows

  local minCols =
    photoGalleryConfig.galleryMinColumns

  local maxCols =
    photoGalleryConfig.galleryMaxColumns

  local gap =
    photoGalleryConfig.galleryGap

  local usableHeight =
    photoGalleryConfig.galleryUsableHeight

  local maxThumbHeight =
    photoGalleryConfig.galleryMaxThumbHeight


  local columns =
    math.ceil(
      imageCount / maxRows
    )

  if columns < minCols then
    columns = minCols
  end

  if columns > maxCols then
    columns = maxCols
  end


  local rows =
    math.ceil(
      imageCount / columns
    )


  local sizingRows =
    rows

  if sizingRows > maxRows then
    sizingRows = maxRows
  end


  local thumbHeight =
    math.floor(
      (
        usableHeight -
        ((sizingRows - 1) * gap)
      ) /
      sizingRows
    )

  if thumbHeight > maxThumbHeight then
    thumbHeight = maxThumbHeight
  end


  return {
    columns = columns,
    rows = rows,
    thumbHeight = thumbHeight,
    scrollExpected = rows > maxRows,
  }
end


-- ============================================================
-- COLLAGE
-- ============================================================

function photoGallery(date, options)
  options =
    options or {}

  date =
    date or
    photoGalleryCurrentDate()

  if not date then
    if options.silent then
      return nil
    end

    return photoGalleryError(
      "PhotoGallery: impossibile ricavare una data YYYY-MM-DD."
    )
  end


  local requestedThumbSize =
    options.thumbSize

  local maxImages =
    options.maxImages

  if maxImages == nil then
    maxImages =
      photoGalleryConfig.maxImages
  end


  local data, err =
    photoGalleryFetch(date)

  if not data then
    if options.silent then
      return nil
    end

    return photoGalleryError(
      "PhotoGallery: " ..
      tostring(err)
    )
  end


  if data.exists == false then
    if options.silent then
      return nil
    end

    return photoGalleryEmpty(
      "Nessuna directory fotografica per " ..
      date ..
      "."
    )
  end


  local images =
    data.images

  if type(images) ~= "table" or
     #images == 0 then

    if options.silent then
      return nil
    end

    return photoGalleryEmpty(
      "Nessuna fotografia per " ..
      date ..
      "."
    )
  end


  local layout =
    photoGalleryLayout(
      #images
    )


  local thumbSize =
    requestedThumbSize or
    ((#images > 24) and "small") or
    photoGalleryConfig.defaultThumbSize


  local grid = {
    class =
      "photo-gallery",

    style =
      "--photo-gallery-cols:" ..
      tostring(layout.columns) ..
      ";" ..
      "--photo-gallery-thumb-height:" ..
      tostring(layout.thumbHeight) ..
      "px;",
  }


  local shown =
    0


  for index, image in ipairs(images) do
    if not maxImages or
       shown < maxImages then

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
          grid,

          dom.button {
            type =
              "button",

            class =
              "photo-gallery-item",

            title =
              image.name or "",

            onclick =
              function()
                editor.navigate(
                  viewerPage
                )
              end,

            dom.img {
              class =
                "photo-gallery-image",

              src =
                thumbUrl,

              alt =
                image.name or "",

              loading =
                "lazy",

              decoding =
                "async",
            },
          }
        )


        shown =
          shown + 1
      end
    end
  end


  if shown == 0 then
    if options.silent then
      return nil
    end

    return photoGalleryEmpty(
      "Nessuna fotografia visualizzabile per " ..
      date ..
      "."
    )
  end


  local wrapper = {
    class =
      "photo-gallery-wrapper",
  }


  if options.title then
    table.insert(
      wrapper,

      dom.div {
        class =
          "photo-gallery-title",

        options.title,
      }
    )
  end


  table.insert(
    wrapper,
    dom.div(grid)
  )


  return widget.htmlBlock(
    dom.div(wrapper)
  )
end


-- ============================================================
-- DIAGNOSTICA
-- ============================================================

function photoGalleryInfo(date)
  date =
    date or
    photoGalleryCurrentDate()

  if not date then
    return
      "PhotoGallery: data non disponibile"
  end


  local data, err =
    photoGalleryFetch(date)

  if not data then
    return
      "PhotoGallery: " ..
      tostring(err)
  end


  return
    "PhotoGallery " ..
    date ..
    ": " ..
    tostring(
      data.count or 0
    ) ..
    " immagini"
end


-- ============================================================
-- BOTTOM WIDGET: GALLERY DIARIO
-- ============================================================

function photoGalleryAutoBottom()
  local pageName =
    editor.getCurrentPage()

  local date =
    photoGalleryDiaryDate(
      pageName
    )

  if not date then
    return nil
  end


  local meta =
    editor.getCurrentPageMeta()

  if not meta then
    return nil
  end


  if meta.PhotoGallery == false then
    return nil
  end


  return photoGallery(
    date,
    {
      silent = true,
      title =
        "📷 Foto della giornata",
    }
  )
end


-- ============================================================
-- BOTTOM WIDGET: VIEWER FOTO
-- ============================================================

function photoGalleryViewerBottom()
  local pageName =
    editor.getCurrentPage()

  local date, index =
    photoGalleryParseViewerPage(
      pageName
    )

  if not date or
     not index then
    return nil
  end


  local data, err =
    photoGalleryFetch(date)

  if not data then
    return photoGalleryError(
      "PhotoGallery: " ..
      tostring(err)
    )
  end


  local images =
    data.images or {}

  local total =
    #images


  if total == 0 then
    return photoGalleryEmpty(
      "Nessuna fotografia per " ..
      date ..
      "."
    )
  end


  if index < 1 or
     index > total then
    return photoGalleryError(
      "Indice fotografia non valido."
    )
  end


  local image =
    images[index]


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


  local nextPage =
    nil

  if index < total then
    nextPage =
      photoGalleryConfig.viewerPrefix ..
      date ..
      ":" ..
      tostring(index + 1)
  end


  local imageNode =
    dom.img {
      class =
        "photo-viewer-bottom-image",

      src =
        largeUrl,

      alt =
        image.name or "",
    }


  local content

  if nextPage then
    content =
      dom.button {
        type =
          "button",

        class =
          "photo-viewer-bottom-button",

        title =
          "Fotografia successiva",

        onclick =
          function()
            editor.navigate(
              nextPage
            )
          end,

        imageNode,
      }
  else
    content =
      dom.div {
        class =
          "photo-viewer-bottom-button photo-viewer-bottom-final",

        imageNode,
      }
  end


  return widget.htmlBlock(
    dom.div {
      class =
        "photo-viewer-bottom",

      content,
    }
  )
end


-- ============================================================
-- UNICO LISTENER BOTTOM
-- ============================================================

event.listen {
  name =
    "hooks:renderBottomWidgets",

  run =
    function(e)
      local pageName =
        editor.getCurrentPage()

      if photoGalleryDiaryDate(pageName) then
        return photoGalleryAutoBottom()
      end

      local date, index =
        photoGalleryParseViewerPage(
          pageName
        )

      if date and index then
        return photoGalleryViewerBottom()
      end

      return nil
    end
}


-- ============================================================
-- VIRTUAL PAGE
-- ============================================================

virtualPage.define {
  pattern =
    "photo:(.+)",

  run =
    function(spec)
      local date, indexText =
        string.match(
          tostring(spec or ""),
          "^(%d%d%d%d%-%d%d%-%d%d):(%d+)$"
        )

      if not date or
         not indexText then
        return
          "PhotoGallery viewer non valido."
      end


      local index =
        tonumber(indexText)


      local data, err =
        photoGalleryFetch(date)

      if not data then
        return
          "PhotoGallery: " ..
          tostring(err)
      end


      local images =
        data.images or {}

      local total =
        #images


      if total == 0 then
        return
          "Nessuna fotografia per " ..
          date ..
          "."
      end


      if not index or
         index < 1 or
         index > total then
        return
          "Indice fotografia non valido."
      end


      local image =
        images[index]


      local shotTime =
        photoGalleryShotTime(
          image.name
        )


      local diaryPage =
        photoGalleryConfig.diaryPagePrefix ..
        date


      local controls = {}


      -- Precedente
      if index > 1 then
        table.insert(
          controls,

          "← [[" ..
          photoGalleryConfig.viewerPrefix ..
          date ..
          ":" ..
          tostring(index - 1) ..
          "|Precedente]]"
        )
      else
        table.insert(
          controls,
          "← Precedente"
        )
      end


      -- Diario
      table.insert(
        controls,

        "[[" ..
        diaryPage ..
        "|← Diario]]"
      )


      -- Indice
      table.insert(
        controls,

        "**" ..
        tostring(index) ..
        " / " ..
        tostring(total) ..
        "**"
      )


      -- Ora
      if shotTime then
        table.insert(
          controls,
          shotTime
        )
      end


      -- Successiva
      if index < total then
        table.insert(
          controls,

          "[[" ..
          photoGalleryConfig.viewerPrefix ..
          date ..
          ":" ..
          tostring(index + 1) ..
          "|Successiva]] →"
        )
      else
        table.insert(
          controls,
          "Successiva →"
        )
      end


      return table.concat(
        controls,
        " · "
      )
    end
}
```

```space-style
/* ============================================================
   Mio Diario PhotoGallery 0.5-01 TEST
   ============================================================ */


/* ------------------------------------------------------------
   GALLERY
   ------------------------------------------------------------ */

.photo-gallery-wrapper {
  width: 100%;
  margin: 1rem 0 0 0;
}

.photo-gallery-title {
  margin: 0 0 0.55rem 0;
  font-weight: 600;
}

.photo-gallery {
  display: grid;

  grid-template-columns:
    repeat(
      var(--photo-gallery-cols, 6),
      minmax(0, 1fr)
    );

  gap: 5px;
  width: 100%;
}

.photo-gallery-item {
  appearance: none;
  -webkit-appearance: none;

  display: block;
  position: relative;

  width: 100%;
  min-width: 0;

  height:
    var(--photo-gallery-thumb-height, 120px);

  overflow: hidden;

  margin: 0;
  padding: 0;

  border: 0;
  border-radius: 4px;

  background: transparent;
  cursor: pointer;
}

.photo-gallery-image {
  display: block;

  width: 100%;
  height: 100%;

  object-fit: cover;

  border: 0;
  margin: 0;
  padding: 0;

  transition:
    transform 120ms ease;
}

.photo-gallery-item:hover
.photo-gallery-image {
  transform: scale(1.025);
}

.photo-gallery-item:focus-visible {
  outline:
    2px solid currentColor;
  outline-offset: 2px;
}


/* ------------------------------------------------------------
   VIEWER BOTTOM
   ------------------------------------------------------------ */

.photo-viewer-bottom {
  display: flex;

  align-items: center;
  justify-content: center;

  width: 100%;

  margin: 0;
  padding: 0;
}

.photo-viewer-bottom-button {
  appearance: none;
  -webkit-appearance: none;

  display: flex;

  align-items: center;
  justify-content: center;

  width: 100%;

  margin: 0;
  padding: 0;

  border: 0;

  background:
    rgba(0, 0, 0, 0.92);

  cursor: pointer;
}

.photo-viewer-bottom-final {
  cursor: default;
}

.photo-viewer-bottom-image {
  display: block;

  width: auto;
  height: auto;

  max-width: 100%;
  max-height: 470px;

  object-fit: contain;

  margin: 0;
  padding: 0;

  border: 0;
}


/* ------------------------------------------------------------
   MESSAGGI
   ------------------------------------------------------------ */

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


/* ------------------------------------------------------------
   RESPONSIVE
   ------------------------------------------------------------ */

@media (max-width: 900px) {
  .photo-viewer-bottom-image {
    max-height: 450px;
  }
}

@media (max-width: 600px) {
  .photo-gallery {
    gap: 4px;
  }

  .photo-viewer-bottom-image {
    max-height: 420px;
  }
}
```

---

# Test 0.5-00

## 1. Gallery Diario

Aprire:

```text
Diario/2026-08-30
```

Verificare:

- gallery automatica;
- layout per righe invariato;
- `PhotoGallery: false` ancora funzionante.

## 2. Viewer

Cliccare una fotografia.

Atteso:

```text
photo:2026-08-30:4
```

Nel corpo della Virtual Page:

```text
← Precedente · ← Diario · 4 / 35 · 09:24:56 · Successiva →
```

Sotto:

- fotografia;
- nessun filename;
- nessun placeholder;
- nessun commento;
- nessun top widget;
- nessuna scrollbar interna.

## 3. Click immagine

Cliccare la fotografia.

Atteso:

- passaggio alla successiva.

## 4. Prima fotografia

Atteso:

- `← Precedente` non cliccabile.

## 5. Ultima fotografia

Atteso:

- `Successiva →` non cliccabile;
- click sulla fotografia non cambia pagina.

---

# Changelog

## 0.5-01 TEST — 2026-09-03

- aumentata altezza massima fotografia desktop da 430 px a 470 px;
- tablet: 450 px;
- smartphone: 420 px;
- mantenuto margine compatibile con il limite nativo di 500 px del bottom widget;
- corretto link Precedente in `← [[...|Precedente]]`;
- corretto link Successiva in `[[...|Successiva]] →`;
- nessuna modifica alla struttura Virtual Page + bottom widget;
- nessuna modifica alla logica Gallery basata sulle righe.


## 0.5-00 TEST — 2026-09-03

- eliminato il viewer top widget;
- Virtual Page resa non vuota con controlli reali;
- fotografia spostata in `hooks:renderBottomWidgets`;
- Virtual Page contiene solo navigazione, indice e ora;
- eliminati placeholder, commenti e zero-width space;
- viewer bottom limitato a 430 px per restare entro il limite nativo dei widget;
- mantenuto click sull'immagine = successiva;
- mantenuta logica Gallery basata sulle righe;
- mantenuto `PhotoGallery: false`;
- mantenuto PhotoGateway invariato.
