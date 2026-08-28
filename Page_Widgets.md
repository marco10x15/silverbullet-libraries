---
name: "Library/MG/Page Widgets"
tags: meta/library
description: "Utility generali per ricavare il titolo visuale di una pagina."
version: "2.00"
versionDate: 2026-08-28
pageDecoration.prefix: "⚙️ "
---

# Page Widgets

Utility generali di presentazione delle pagine per SilverBullet v2.

**Versione:** 2.00 — 28.08.2026

Questa revisione separa nettamente le responsabilità tra le librerie:

* `PageNavigation.md` contiene `page.nome()`, `page.parents()`, `page.prec()`,
  `page.succ()`, `page.child()`, `page.sister()`, `page.up()`,
  `page.lev()` e `page.breadcrumb()`;
* `Page_Widgets.md` contiene `page.title()` e le eventuali future utility
  generali di presentazione che non appartengono alla navigazione.

La precedente `tag.define` che generava automaticamente l'attributo `title`
è stata rimossa. La libreria non crea né modifica attributi persistenti.

## `page.title(where)`

Restituisce il titolo visuale di una pagina.

`where` è opzionale. Se omesso viene utilizzata la pagina corrente.

### Priorità

1. `displayName`;
2. `title`, mantenuto come fallback per pagine legacy;
3. primo header H1 presente nella pagina;
4. stringa vuota se nessuna delle sorgenti precedenti è disponibile.

Per le pagine `Diario/` normalizzate la sorgente canonica è quindi
`displayName`. Il fallback `title` serve esclusivamente a mantenere
compatibilità con pagine non ancora migrate o con altri insiemi di pagine.

La ricerca dell'H1 usa gli oggetti `header` indicizzati da SilverBullet e
l'attributo `range`; non usa il precedente attributo `pos`.

### Esempi

```space-lua
page.title()
page.title("Diario/2026-05-09")
page.title("luoghi/ITA/21/Torino")
```

## Implementazione

```space-lua
page = page or {}


-- Restituisce true solo per stringhe non vuote.
local function pageWidgetsHasString(value)
  return type(value) == "string"
    and value ~= ""
end


-- Restituisce displayName e title indicizzati per una pagina.
local function pageWidgetsPageInfo(where)
  local pages = query[[
    from p = index.pages()
    where p.name == where
    select {
      displayName = p.displayName,
      title = p.title
    }
    limit 1
  ]]

  if pages
    and #pages > 0
  then
    return pages[1]
  end

  return nil
end


-- Restituisce il primo H1 in ordine di posizione nella pagina.
local function pageWidgetsFirstH1(where)
  local headers = query[[
    from h = index.headers()
    where h.page == where
      and h.level == 1
    select {
      text = h.text,
      range = h.range
    }
  ]]

  if not headers
    or #headers == 0
  then
    return ""
  end

  local firstText = ""
  local firstStart = nil

  for _, h in ipairs(headers) do
    if pageWidgetsHasString(h.text) then
      local start = nil

      if type(h.range) == "table" then
        start = h.range[1]
      end

      if type(start) == "number" then
        if firstStart == nil
          or start < firstStart
        then
          firstStart = start
          firstText = h.text
        end
      elseif firstText == "" then
        -- Fallback prudenziale nel caso in cui un oggetto header
        -- non esponga range.
        firstText = h.text
      end
    end
  end

  return firstText
end


function page.title(where)
  where =
    where
    or editor.getCurrentPage()

  if not pageWidgetsHasString(where) then
    return ""
  end

  local p =
    pageWidgetsPageInfo(where)

  if not p then
    return ""
  end

  if pageWidgetsHasString(
    p.displayName
  ) then
    return p.displayName
  end

  if pageWidgetsHasString(
    p.title
  ) then
    return p.title
  end

  return pageWidgetsFirstH1(where)
end
```

## Modifiche versione 2.00

* `page.title()` usa la priorità `displayName` → `title` legacy → primo H1 → `""`.
* Il primo H1 viene ricavato tramite `index.headers()` e `range`.
* Rimossa la `tag.define` che generava automaticamente `title`.
* Rimossa da questa libreria la definizione di `page.nome()`, ora appartenente
  esclusivamente a `PageNavigation.md`.
* Nessuna lettura diretta del Markdown e nessun attributo persistente aggiuntivo.
* La libreria usa esclusivamente API SilverBullet v2 correnti.
