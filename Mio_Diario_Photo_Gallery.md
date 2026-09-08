---
name: "Library/MG/Mio_Diario_Photo_Gallery"
tags: meta/library
description: "Integrazione della PhotoGallery WebUI Python nelle pagine Diario."
version: "0.8-01"
versionDate: 2026-09-08
pageDecoration.prefix: "📷 "
share.uri: "github:marco10x15/silverbullet-libraries/Mio_Diario_Photo_Gallery.md"
---

# Mio Diario — Photo Gallery

**Versione:** 0.8-01 — 08 settembre 2026

Versione coordinata con **PhotoGallery WebUI Python 0.1-04**.

La libreria non registra più Bottom Widget autonomi. Espone il renderer
`photoGalleryWidget()` e gli helper di disponibilità/URL usati dal componente
unificato di `Mio_Diario`.

## Configurazione

```space-lua
photoGalleryConfig = photoGalleryConfig or {
  webUiBase =
    "https://sb2.fm-nas.net/gallery/",

  apiBase =
    "https://sb2.fm-nas.net/api/gallery/",

  diaryPagePrefix =
    "Diario/",
}


function photoGalleryDiaryDate(pageName)
  if type(pageName) ~= "string" then
    return nil
  end

  return string.match(
    pageName,
    "^" ..
    photoGalleryConfig.diaryPagePrefix ..
    "(%d%d%d%d%-%d%d%-%d%d)$"
  )
end


function photoGalleryEmbeddedUrl(date)
  return
    photoGalleryConfig.webUiBase ..
    date ..
    "?embedded=1"
end


function photoGalleryFullUrl(date)
  return
    photoGalleryConfig.webUiBase ..
    date
end


photoGalleryInfoCache =
  photoGalleryInfoCache
  or {}


function photoGalleryInfo(date)
  if not date or date == "" then
    return nil
  end

  if photoGalleryInfoCache[date] ~= nil then
    local cached =
      photoGalleryInfoCache[date]

    return cached ~= false
      and cached
      or nil
  end

  local response =
    net.proxyFetch(
      photoGalleryConfig.apiBase
        .. date
    )

  if not response
    or not response.ok
    or type(response.body) ~= "table"
  then
    photoGalleryInfoCache[date] = false
    return nil
  end

  photoGalleryInfoCache[date] =
    response.body

  return response.body
end


function photoGalleryHasPhotos(date)
  local info =
    photoGalleryInfo(date)

  if not info then
    return false
  end

  return info.exists ~= false
    and tonumber(
      info.count or 0
    ) > 0
end


function photoGalleryWidget(
  date,
  options
)
  options = options or {}

  date =
    date
    or photoGalleryDiaryDate(
      editor.getCurrentPage()
    )

  if not date then
    return nil
  end

  if options.requirePhotos ~= false
    and not photoGalleryHasPhotos(date)
  then
    return nil
  end

  local details = {
    class =
      "photo-gallery-details",

    dom.summary {
      class =
        "photo-gallery-summary",

      options.caption
        or "PhotoGallery",
    },

    dom.div {
      class =
        "photo-gallery-toolbar",

      dom.a {
        class =
          "photo-gallery-open",

        href =
          photoGalleryFullUrl(date),

        target =
          "_blank",

        rel =
          "noopener noreferrer",

        "Apri galleria a pagina intera ↗",
      },
    },

    dom.iframe {
      class =
        "photo-gallery-frame",

      src =
        photoGalleryEmbeddedUrl(date),

      title =
        "Galleria fotografica "
          .. date,

      loading =
        "lazy",
    },
  }

  if options.open == true then
    details.open = true
  end

  return widget.htmlBlock(
    dom.details(details)
  )
end


```

## Space Style

```space-style
.photo-gallery-details {
  width: 100%;
  margin: 0;
}

.photo-gallery-summary {
  cursor: pointer;
  font-weight: 600;
  padding: 0.15rem 0 0.25rem 0;
  user-select: none;
}

.photo-gallery-toolbar {
  display: flex;
  justify-content: flex-end;
  width: 100%;
  margin: -1.65rem 0 0.25rem 0;
  padding-right: 0.2rem;
}

.photo-gallery-open {
  font-size: 0.9em;
  text-decoration: none;
}

.photo-gallery-frame {
  display: block;
  width: 100%;
  height: 600px;
  margin: 0;
  padding: 0;
  border: 0;
  border-radius: 4px;
}

@media (max-width: 600px) {
  .photo-gallery-toolbar {
    position: static;
    justify-content: flex-start;
    margin: 0 0 0.25rem 0;
  }

  .photo-gallery-frame {
    height: 520px;
  }
}
```

## Comportamento

La libreria non crea più un Bottom Widget automatico.

`photoGalleryWidget(date, options)` è il renderer standalone canonico della
PhotoGallery. Il componente unificato `mediaLuoghiWidget()` usa gli helper
pubblici di questa libreria per:

* verificare se esistono fotografie;
* costruire URL embedded e full-page;
* mostrare `📷` soltanto quando la galleria contiene immagini.

L'opzione `options.open=true` riguarda esclusivamente l'uso standalone di
`photoGalleryWidget()`.

## Strategia di consolidamento PhotoGallery

Stato corrente:

* `photoGalleryWidget()` è il renderer standalone canonico;
* `mediaLuoghiWidget()` incorpora l'iframe nel proprio sandbox per consentire
  il cambio vista `📋 / 🗺️ / 🧭 / 📷`;
* `photoGalleryInfo()` mantiene una cache per data.

Per eliminare l'N+1 HTTP nelle pagine `Viaggi/...`, il passo successivo
consigliato è aggiungere a PhotoGateway un endpoint batch, ad esempio:

```text
GET /api/gallery?dates=2026-06-17,2026-06-18,2026-06-19
```

con risposta minima per data:

```json
{
  "2026-06-17": {"exists": true, "count": 24},
  "2026-06-18": {"exists": false, "count": 0}
}
```

La libreria Lua potrà così eseguire una sola `net.proxyFetch()` per viaggio e
riempire la cache già esistente. Questa soluzione evita richieste HTTP per ogni
giornata e non introduce dati persistenti nelle note.

## Changelog

### 0.8-01 — 2026-09-08

- rimossa la configurazione `defaultOpen`, non più utilizzata;
- chiarito che `photoGalleryWidget()` è il renderer standalone canonico;
- corretta la documentazione del comportamento corrente;
- documentata la strategia batch proposta per eliminare l'N+1 HTTP nei Viaggi.

### 0.8-00 — 2026-09-08

- rimosso il listener `hooks:renderBottomWidgets`;
- aggiunti `photoGalleryInfo()` e `photoGalleryHasPhotos()` con cache;
- la disponibilità usa `/api/gallery/YYYY-MM-DD` e `count > 0`;
- `photoGalleryWidget()` resta renderer autonomo.


### 0.7-04 TEST — 2026-09-07

- Bottom Widget invariato tecnicamente ma con summary `PhotoGallery`;
- `<details>` resta chiuso per default;
- nessuna modifica alla WebUI o al caricamento iframe.

### 0.7-03 TEST — 2026-09-06

- `<details>` realmente chiuso per default;
- rimosso `open=false` come attributo HTML;
- `open` viene aggiunto soltanto quando `defaultOpen == true`;
- coordinata con PhotoGallery WebUI 0.1-04.

### 0.7-02 TEST — 2026-09-06

- resize iframe delegato alla WebUI;
- link alla galleria completa;
- Virtual Page `photo:` assente.
