---
name: "Library/MG/Mio_Diario_PhotoGallery"
tags: meta/library
description: "Integrazione della PhotoGallery WebUI Python nelle pagine Diario."
version: "0.7-03"
versionDate: 2026-09-06
pageDecoration.prefix: "📷 "
share.uri: "github:marco10x15/silverbullet-libraries/Mio_Diario_Photo_Gallery.md"
---

# Mio Diario — Photo Gallery

**Versione:** 0.7-03 TEST — 06 settembre 2026

Versione coordinata con **PhotoGallery WebUI Python 0.1-04**.

Il Bottom Widget parte **chiuso per default**. Quando `defaultOpen=false`
l'elemento `<details>` viene creato senza attributo HTML `open`.

## Configurazione

```space-lua
photoGalleryConfig = photoGalleryConfig or {
  webUiBase =
    "https://sb2.fm-nas.net/gallery/",

  diaryPagePrefix =
    "Diario/",

  defaultOpen =
    false,
}


local function photoGalleryDiaryDate(pageName)
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


local function photoGalleryEmbeddedUrl(date)
  return
    photoGalleryConfig.webUiBase ..
    date ..
    "?embedded=1"
end


local function photoGalleryFullUrl(date)
  return
    photoGalleryConfig.webUiBase ..
    date
end


function photoGalleryAutoBottom()
  local date =
    photoGalleryDiaryDate(
      editor.getCurrentPage()
    )

  if not date then
    return nil
  end

  local meta =
    editor.getCurrentPageMeta()

  if not meta or
     meta.PhotoGallery == false then
    return nil
  end


  local details = {
    class =
      "photo-gallery-details",

    dom.summary {
      class =
        "photo-gallery-summary",

      "📷 Foto della giornata",
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
        "Galleria fotografica " ..
        date,

      loading =
        "lazy",
    },
  }


  -- Gli attributi booleani HTML dipendono dalla presenza:
  -- quando false NON aggiungiamo affatto "open".
  if photoGalleryConfig.defaultOpen == true then
    details.open = true
  end


  return widget.htmlBlock(
    dom.details(details)
  )
end


event.listen {
  name =
    "hooks:renderBottomWidgets",

  run =
    function(e)
      return photoGalleryAutoBottom()
    end
}
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

Per default:

```text
▶ 📷 Foto della giornata
```

Solo dopo il click:

```text
▼ 📷 Foto della giornata
   [PhotoGallery embedded]
```

Per aprirlo esplicitamente per default:

```space-lua
photoGalleryConfig.defaultOpen = true
```

Per disabilitarlo su una singola pagina:

```yaml
PhotoGallery: false
```

## Changelog

### 0.7-03 TEST — 2026-09-06

- `<details>` realmente chiuso per default;
- rimosso `open=false` come attributo HTML;
- `open` viene aggiunto soltanto quando `defaultOpen == true`;
- coordinata con PhotoGallery WebUI 0.1-04.

### 0.7-02 TEST — 2026-09-06

- resize iframe delegato alla WebUI;
- link alla galleria completa;
- Virtual Page `photo:` assente.
