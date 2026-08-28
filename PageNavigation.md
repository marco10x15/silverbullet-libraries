---
name: "Library/MG/PageNavigation"
tags: meta/library
version: "1.02"
versionDate: 2026-08-28
pageDecoration.prefix: "📃 "
share.uri: "github:marco10x15/silverbullet-libraries/PageNavigation.md"
---

# Page Navigation Functions

Funzioni di navigazione pagine e gestione metadati.

Questa revisione usa esclusivamente le API correnti SilverBullet v2 per le query sulle pagine e riduce le query ripetute del breadcrumb.

## Funzioni

| Funzione | Restituisce | Descrizione |
|---|---|---|
| `page.parents(path)` | table | Genitori della pagina |
| `page.sister(path)` | table | Sorelle dirette |
| `page.child(path)` | table/nil | Figli diretti |
| `page.up(path)` | string | Pagina padre |
| `page.prec(path)` | string | Sorella precedente |
| `page.succ(path)` | string | Sorella successiva |
| `page.nome(path)` | string | Nome dall'ultimo segmento del path |
| `page.lev(path)` | number | Livello della pagina |
| `breadcrumb()` | string | Breadcrumb fino alla pagina corrente |

## Implementation

```space-lua
page = page or {}


-- ============================================================
-- PATH
-- ============================================================

-- Restituisce i genitori della pagina, dal livello più alto
-- fino al parent diretto.
function page.parents(path)
  path =
    path
    or editor.getCurrentPage()

  local livelli =
    string.split(
      path,
      "/"
    )

  local genitori = {}
  local currentPath = ""

  for i = 1, #livelli - 1 do
    if i == 1 then
      currentPath =
        livelli[i]
    else
      currentPath =
        currentPath
        .. "/"
        .. livelli[i]
    end

    table.insert(
      genitori,
      currentPath
    )
  end

  return genitori
end


-- Restituisce il path della pagina padre.
function page.up(path)
  path =
    path
    or editor.getCurrentPage()

  return string.match(
    path,
    "^(.*)/[^/]+$"
  ) or ""
end


-- Estrae l'ultimo segmento di un path.
function page.nome(path)
  path =
    path
    or editor.getCurrentPage()

  return string.match(
    path,
    "([^/]+)$"
  ) or path
end


-- Restituisce il livello della pagina.
function page.lev(path)
  path =
    path
    or editor.getCurrentPage()

  return #string.split(
    path,
    "/"
  )
end


-- ============================================================
-- PAGINE SORELLE E FIGLIE
-- ============================================================

-- Restituisce le pagine sorelle dirette.
function page.sister(path)
  path =
    path
    or editor.getCurrentPage()

  local parent =
    page.up(path)

  if parent == "" then
    return {}
  end

  return query[[
    from p = index.subPages(parent)
    where not string.find(
      string.sub(
        p.name,
        #parent + 2
      ),
      "/"
    )
    order by p.name
  ]]
end


-- Restituisce i figli diretti della pagina.
function page.child(path)
  path =
    path
    or editor.getCurrentPage()

  local children = query[[
    from p = index.subPages(path)
    where not string.find(
      string.sub(
        p.name,
        #path + 2
      ),
      "/"
    )
    order by p.name
  ]]

  if not children
    or #children == 0
  then
    return nil
  end

  return children
end


-- Restituisce la sorella precedente.
--
-- La query usa direttamente index.subPages(parent) e restituisce
-- soltanto il primo risultato utile.
function page.prec(path)
  path =
    path
    or editor.getCurrentPage()

  local parent =
    page.up(path)

  if parent == "" then
    return ""
  end

  local pages = query[[
    from p = index.subPages(parent)
    where not string.find(
      string.sub(
        p.name,
        #parent + 2
      ),
      "/"
    )
      and p.name < path
    order by p.name desc
    select {
      name = p.name
    }
    limit 1
  ]]

  if not pages
    or #pages == 0
  then
    return ""
  end

  return pages[1].name
end


-- Restituisce la sorella successiva.
function page.succ(path)
  path =
    path
    or editor.getCurrentPage()

  local parent =
    page.up(path)

  if parent == "" then
    return ""
  end

  local pages = query[[
    from p = index.subPages(parent)
    where not string.find(
      string.sub(
        p.name,
        #parent + 2
      ),
      "/"
    )
      and p.name > path
    order by p.name
    select {
      name = p.name
    }
    limit 1
  ]]

  if not pages
    or #pages == 0
  then
    return ""
  end

  return pages[1].name
end


-- ============================================================
-- BREADCRUMB
-- ============================================================

-- Nome visuale ricavato esclusivamente dagli attributi indicizzati.
--
-- Priorità:
-- displayName -> title -> aliases[1] -> basename.
local function pageNavigationLabel(p)
  if p then
    if type(p.displayName) == "string"
      and p.displayName ~= ""
    then
      return p.displayName
    end

    if type(p.title) == "string"
      and p.title ~= ""
    then
      return p.title
    end

    if type(p.aliases) == "string"
      and p.aliases ~= ""
    then
      return p.aliases
    end

    if type(p.aliases) == "table"
      and #p.aliases > 0
      and type(p.aliases[1]) == "string"
      and p.aliases[1] ~= ""
    then
      return p.aliases[1]
    end
  end

  return nil
end


-- Costruisce il breadcrumb con una sola query ai metadati delle
-- pagine coinvolte.
--
-- `path` è opzionale; `includeNav=false` esclude le frecce di
-- navigazione e consente il riuso del breadcrumb dentro altri widget.
function page.breadcrumb(
  path,
  includeNav
)
  path =
    path
    or editor.getCurrentPage()

  local parts =
    string.split(
      path,
      "/"
    )

  local paths = {}
  local currentPath = ""

  for i, part in ipairs(parts) do
    if i == 1 then
      currentPath =
        part
    else
      currentPath =
        currentPath
        .. "/"
        .. part
    end

    table.insert(
      paths,
      currentPath
    )
  end

  local indexedPages = query[[
    from p = index.pages()
    where table.includes(
      paths,
      p.name
    )
    select {
      name = p.name,
      title = p.title,
      displayName = p.displayName,
      aliases = p.aliases
    }
  ]]

  local labels = {}

  for _, p in ipairs(indexedPages) do
    labels[p.name] =
      pageNavigationLabel(p)
  end

  local navp = nil
  local navs = nil

  if includeNav ~= false then
    local navpRaw =
      page.navp
      and page.navp()
      or nil

    local navsRaw =
      page.navs
      and page.navs()
      or nil

    navp =
      navpRaw
      and navpRaw ~= ""
      and (
        "[["
        .. navpRaw
        .. "|👈]]"
      )
      or nil

    navs =
      navsRaw
      and navsRaw ~= ""
      and (
        "[["
        .. navsRaw
        .. "|👉]]"
      )
      or nil
  end

  local breadcrumbs = {}

  for i, pagePath in ipairs(paths) do
    local label =
      labels[pagePath]
      or page.nome(pagePath)

    if i == #paths then
      if navp then
        table.insert(
          breadcrumbs,
          navp
        )
      end

      table.insert(
        breadcrumbs,
        "**"
          .. label
          .. "**"
      )
    else
      table.insert(
        breadcrumbs,
        string.format(
          "[[%s|%s]]",
          pagePath,
          label
        )
      )
    end
  end

  if navs then
    table.insert(
      breadcrumbs,
      navs
    )
  end

  return table.concat(
    breadcrumbs,
    "|"
  )
end


-- Wrapper compatibile con l'uso precedente.
function breadcrumb()
  local path =
    editor.getCurrentPage()

  if string.startsWith(
    path,
    "Diario/"
  ) then
    return
  end

  if string.startsWith(
    path,
    "index"
  ) then
    return
  end

  return page.breadcrumb(
    path,
    true
  )
end


-- ============================================================
-- ATTIVAZIONE
-- ============================================================

event.listen {
  name = "hooks:renderTopWidgets",

  run = function(e)
    local pageName =
      editor.getCurrentPage()

    if string.startsWith(
      pageName,
      "Diario/"
    ) then
      return
    end

    if string.startsWith(
      pageName,
      "index"
    ) then
      return
    end

    if pageName == "luoghi"
      or string.startsWith(
        pageName,
        "luoghi/"
      )
    then
      return
    end

    local text =
      breadcrumb()

    if not text
      or text == ""
    then
      return
    end

    return widget.new {
      markdown =
        "\n\n"
        .. text
        .. "\n\n"
    }
  end
}
```

## Modifiche 1.02

* introdotta `page.breadcrumb(path, includeNav)` come funzione riutilizzabile;
* `includeNav=false` produce il solo percorso gerarchico, adatto all'inserimento dentro altri widget;
* il wrapper globale `breadcrumb()` rimane disponibile per compatibilità;
* il top widget non renderizza più il breadcrumb sulle pagine `luoghi` e `luoghi/...`: la visualizzazione viene delegata alla scheda geografica di `Mio_Diario_1.03`.

## Ottimizzazioni 1.01

* sostituite le query legacy `index.tag "page"` con `index.subPages()`;
* `page.prec()` e `page.succ()` accettano ora opzionalmente il path, mantenendo il comportamento precedente se omesso;
* `page.up()` non usa più una variabile globale temporanea;
* `breadcrumb()` controlla subito le esclusioni `Diario/` e `index`, prima di eseguire query;
* rimossa la query inutilizzata `page.child()` dal breadcrumb;
* rimossa la funzione locale `getDisplayName()` inutilizzata;
* il breadcrumb risolve tutti i label con una sola query `index.pages()` anziché chiamare `page.title()` per ogni livello;
* il breadcrumb usa soltanto attributi già indicizzati: `displayName`, `title`, `aliases`.
