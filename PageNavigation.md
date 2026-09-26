---
name: "Library/MG/PageNavigation"
tags: meta/library
version: "1.06"
versionDate: 2026-09-25
pageDecoration.prefix: "📃 "
share.uri: "github:marco10x15/silverbullet-libraries/PageNavigation.md"
---


# Page Navigation Functions

Funzioni generali di navigazione gerarchica delle pagine per SilverBullet 2.11.1.

La libreria utilizza le API correnti SilverBullet v2 e separa:

* gestione dei path;
* individuazione di figli e sorelle;
* navigazione precedente/successiva;
* costruzione del breadcrumb;
* configurazione delle pagine escluse dal breadcrumb automatico.
    

Le funzioni accettano opzionalmente un `path`. Se omesso viene utilizzata la pagina corrente.

Le etichette del breadcrumb utilizzano esclusivamente `displayName`; se l'attributo non è presente viene utilizzato l'ultimo segmento del path.

## Funzioni

| Funzione | Restituisce | Descrizione |
| --- | --- | --- |
| `page.parents(path)` | table | Genitori della pagina |
| `page.sister(path)` | table | Sorelle dirette, comprese quelle al livello root |
| `page.child(path)` | table | Figli diretti; restituisce sempre una table |
| `page.up(path)` | string | Path della pagina padre |
| `page.prec(path)` | string | Sorella precedente |
| `page.succ(path)` | string | Sorella successiva |
| `page.nome(path)` | string | Ultimo segmento del path |
| `page.lev(path)` | number | Livello della pagina |
| `page.breadcrumb(path, includeNav)` | string | Breadcrumb della pagina indicata |
| `breadcrumb()` | string/nil | Breadcrumb automatico della pagina corrente |

## Configurazione

Le esclusioni dal breadcrumb automatico sono configurate nello Space e non nella libreria.

Ogni voce esclude:

* la pagina indicata;
* tutte le sue sottopagine.
    

Esempio per **Mio Diario**:

```
config.set("pageNavigation.breadcrumbExclude", {
  "Diario",
  "index",
  "luoghi",
})
```

La configurazione precedente esclude quindi:

* `Diario` e tutto il ramo `Diario/...`;
* `index`;
* `luoghi` e tutto il ramo `luoghi/...`.
    

`page.breadcrumb(path, includeNav)` rimane sempre richiamabile direttamente: le esclusioni riguardano soltanto il breadcrumb automatico renderizzato nella parte superiore della pagina.

## Implementation

```space-lua
page = page or {}


-- ============================================================
-- CONFIGURAZIONE
-- ============================================================

-- Elenco delle pagine e delle gerarchie escluse dal breadcrumb
-- automatico.
--
-- La libreria non definisce esclusioni specifiche di uno Space.
-- Le eventuali esclusioni devono essere configurate nel CONFIG
-- dello Space tramite:
--
-- config.set("pageNavigation.breadcrumbExclude", {
--   "Pagina",
--   "AltraPagina",
-- })
--
-- Ogni voce esclude sia la pagina indicata sia tutte le sue
-- sottopagine.
config.define(
  "pageNavigation.breadcrumbExclude",
  {
    description =
      "Pagine e gerarchie escluse dal breadcrumb automatico.",
    type = "array",
    items = {
      type = "string",
    },
    default = {},
  }
)


-- Verifica se il breadcrumb automatico deve essere escluso
-- per il path indicato.
--
-- Esempio:
-- excluded = "Diario"
--
-- vengono esclusi:
-- Diario
-- Diario/2026
-- Diario/2026/2026-09-25
--
-- non vengono escluse pagine come:
-- DiarioTest
local function pageNavigationBreadcrumbExcluded(path)
  local exclusions =
    config.get(
      "pageNavigation.breadcrumbExclude"
    )
    or {}

  for _, excluded in ipairs(exclusions) do
    if path == excluded
      or string.startsWith(
        path,
        excluded .. "/"
      )
    then
      return true
    end
  end

  return false
end


-- ============================================================
-- PATH
-- ============================================================

-- Restituisce i genitori della pagina dal livello più alto
-- fino al parent diretto.
--
-- Esempio:
-- page.parents("A/B/C")
--
-- restituisce:
-- {
--   "A",
--   "A/B",
-- }
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
--
-- Per una pagina al livello root restituisce stringa vuota.
--
-- Esempi:
-- page.up("A/B/C") -> "A/B"
-- page.up("A")     -> ""
function page.up(path)
  path =
    path
    or editor.getCurrentPage()

  return string.match(
    path,
    "^(.*)/[^/]+$"
  ) or ""
end


-- Restituisce l'ultimo segmento del path.
--
-- Esempio:
-- page.nome("A/B/C") -> "C"
function page.nome(path)
  path =
    path
    or editor.getCurrentPage()

  return string.match(
    path,
    "([^/]+)$"
  ) or path
end


-- Restituisce il livello gerarchico della pagina.
--
-- Esempi:
-- page.lev("A")     -> 1
-- page.lev("A/B")   -> 2
-- page.lev("A/B/C") -> 3
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
-- PAGINE FIGLIE E SORELLE
-- ============================================================

-- Restituisce esclusivamente i figli diretti del path indicato.
--
-- Questa funzione privata centralizza la definizione di
-- "figlio diretto", utilizzata da:
--
-- page.child()
-- page.sister()
-- page.prec()
-- page.succ()
--
-- Per path == "" restituisce esclusivamente le pagine al livello
-- root dello Space.
--
-- Per tutti gli altri path utilizza index.subPages() e rimuove
-- le pagine appartenenti ai livelli gerarchici più profondi.
local function pageNavigationDirectChildren(path)
  if path == "" then
    return query[[
      from p = index.pages()
      where not string.find(
        p.name,
        "/"
      )
      order by p.name
    ]]
  end

  return query[[
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
end


-- Restituisce le pagine sorelle dirette della pagina indicata.
--
-- Funziona anche per le pagine al livello root.
--
-- Restituisce sempre una table.
function page.sister(path)
  path =
    path
    or editor.getCurrentPage()

  return pageNavigationDirectChildren(
    page.up(path)
  )
end


-- Restituisce i figli diretti della pagina indicata.
--
-- Restituisce sempre una table:
-- se non esistono figli restituisce {}.
function page.child(path)
  path =
    path
    or editor.getCurrentPage()

  return pageNavigationDirectChildren(
    path
  )
end


-- Restituisce il path della sorella precedente secondo
-- l'ordinamento alfabetico del path.
--
-- Restituisce stringa vuota se:
-- * la pagina è la prima;
-- * la pagina non viene trovata tra le sorelle.
--
-- Funziona anche per le pagine al livello root.
function page.prec(path)
  path =
    path
    or editor.getCurrentPage()

  local pages =
    pageNavigationDirectChildren(
      page.up(path)
    )

  for i, p in ipairs(pages) do
    if p.name == path then
      if i > 1 then
        return pages[i - 1].name
      end

      return ""
    end
  end

  return ""
end


-- Restituisce il path della sorella successiva secondo
-- l'ordinamento alfabetico del path.
--
-- Restituisce stringa vuota se:
-- * la pagina è l'ultima;
-- * la pagina non viene trovata tra le sorelle.
--
-- Funziona anche per le pagine al livello root.
function page.succ(path)
  path =
    path
    or editor.getCurrentPage()

  local pages =
    pageNavigationDirectChildren(
      page.up(path)
    )

  for i, p in ipairs(pages) do
    if p.name == path then
      if i < #pages then
        return pages[i + 1].name
      end

      return ""
    end
  end

  return ""
end


-- ============================================================
-- BREADCRUMB
-- ============================================================

-- Restituisce l'etichetta visuale di una pagina indicizzata.
--
-- Viene utilizzato esclusivamente displayName.
--
-- Se displayName non esiste o è vuoto viene restituito nil;
-- page.breadcrumb() utilizzerà quindi page.nome() come fallback.
--
-- title e aliases non vengono utilizzati intenzionalmente.
local function pageNavigationLabel(p)
  if p
    and type(p.displayName) == "string"
    and p.displayName ~= ""
  then
    return p.displayName
  end

  return nil
end


-- Costruisce il breadcrumb della pagina indicata.
--
-- path:
--   opzionale;
--   se omesso utilizza editor.getCurrentPage().
--
-- includeNav:
--   true/nil -> aggiunge precedente e successiva;
--   false    -> produce soltanto il percorso gerarchico.
--
-- La funzione può essere richiamata direttamente anche per pagine
-- escluse dal breadcrumb automatico.
--
-- Tutte le etichette vengono recuperate con una sola query
-- all'indice.
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

  -- Recupera in una sola query i displayName di tutti i livelli
  -- che compongono il breadcrumb.
  local indexedPages = query[[
    from p = index.pages()
    where table.includes(
      paths,
      p.name
    )
    select {
      name = p.name,
      displayName = p.displayName
    }
  ]]

  local labels = {}

  for _, p in ipairs(indexedPages) do
    labels[p.name] =
      pageNavigationLabel(p)
  end

  -- Navigazione precedente / successiva.
  --
  -- Dalla versione 1.06 vengono utilizzate direttamente
  -- page.prec() e page.succ(); non esistono più dipendenze dalle
  -- vecchie funzioni page.navp() e page.navs().
  local navp = nil
  local navs = nil

  if includeNav ~= false then
    local navpRaw =
      page.prec(path)

    local navsRaw =
      page.succ(path)

    navp =
      navpRaw ~= ""
      and (
        "[["
        .. navpRaw
        .. "|👈]]"
      )
      or nil

    navs =
      navsRaw ~= ""
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
      -- La pagina corrente non è un link.
      -- Se disponibile viene anteposta la navigazione precedente.
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
      -- Tutti i livelli superiori sono link navigabili.
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

  -- Se disponibile aggiunge la navigazione successiva.
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


-- ============================================================
-- BREADCRUMB AUTOMATICO
-- ============================================================

-- Wrapper compatibile con l'uso storico della libreria.
--
-- Verifica una sola volta le esclusioni configurate e, se la pagina
-- è ammessa, restituisce il breadcrumb completo di navigazione
-- precedente/successiva.
--
-- Le esclusioni non vengono duplicate nel renderTopWidgets.
function breadcrumb()
  local path =
    editor.getCurrentPage()

  if pageNavigationBreadcrumbExcluded(
    path
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

-- Renderizza automaticamente il breadcrumb nella parte superiore
-- della pagina.
--
-- La decisione di visualizzarlo o meno è delegata interamente a
-- breadcrumb(), evitando controlli duplicati delle esclusioni.
event.listen {
  name = "hooks:renderTopWidgets",

  run = function()
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

## Modifiche 1.06

* sostituite nel breadcrumb le dipendenze da `page.navp()` e `page.navs()` con le funzioni già pubbliche `page.prec(path)` e `page.succ(path)`;
* `page.child()` restituisce ora sempre una table e quindi `{}` quando non esistono figli;
* centralizzata nella funzione privata `pageNavigationDirectChildren()` la logica utilizzata per identificare i figli diretti;
* aggiunto il supporto alle pagine al livello root per `page.sister()`, `page.prec()` e `page.succ()`;
* il breadcrumb utilizza ora esclusivamente `displayName`, con fallback all'ultimo segmento del path; rimossi completamente `title` e `aliases`;
* eliminata la doppia verifica delle esclusioni tra `breadcrumb()` e `hooks:renderTopWidgets`;
* introdotta la configurazione `pageNavigation.breadcrumbExclude`, permettendo a ogni Space di definire autonomamente le pagine e le gerarchie escluse;
* le esclusioni sono gerarchiche ma rispettano i confini del path: l'esclusione `"index"` non esclude, ad esempio, `"indexTest"`;
* mantenuta una sola query `index.pages()` per recuperare i `displayName` necessari alla costruzione del breadcrumb;
* verificato il funzionamento della configurazione, della navigazione e del breadcrumb nello Space **Mio Diario** con SilverBullet 2.11.1.
    

## Modifiche 1.02

* introdotta `page.breadcrumb(path, includeNav)` come funzione riutilizzabile;
* `includeNav=false` produce il solo percorso gerarchico, adatto all'inserimento dentro altri widget;
* il wrapper globale `breadcrumb()` rimane disponibile per compatibilità;
* il top widget non renderizza più il breadcrumb sulle pagine `luoghi` e `luoghi/...`: la visualizzazione viene delegata alla scheda geografica di `Mio_Diario_1.03`.
    

## Ottimizzazioni 1.01

* sostituite le query legacy `index.tag "page"` con `index.subPages()`;
* `page.prec()` e `page.succ()` accettano opzionalmente il path, mantenendo il comportamento precedente se omesso;
* `page.up()` non usa più una variabile globale temporanea;
* `breadcrumb()` controllava le esclusioni prima di eseguire query;
* rimossa la query inutilizzata `page.child()` dal breadcrumb;
* rimossa la funzione locale `getDisplayName()` inutilizzata;
* il breadcrumb risolve tutti i label con una sola query `index.pages()` anziché chiamare `page.title()` per ogni livello;
* il breadcrumb utilizza soltanto attributi già indicizzati.
