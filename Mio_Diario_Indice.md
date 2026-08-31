---
name: "Library/MG/Mio_Diario_Indice"
tags: meta/library
description: "Indice inline delle pagine Diario con data, titolo, immagine, snippet e filtro."
version: "0.1-11"
versionDate: 2026-08-31
pageDecoration.prefix: "📔 "
share.uri: "github:marco10x15/silverbullet-libraries/Mio_Diario_Indice.md"
---

# 📔 IndiceDiario

**IndiceDiario** visualizza direttamente in una pagina SilverBullet un indice compatto delle pagine del Diario.

**Versione:** 0.1-11 — 31.08.2026

La libreria è autonoma e non dipende da Journal Explorer.

## Main Features

* **Indice inline** — inseribile con `${widgets.IndiceDiario()}`.
* **Riepilogo annuale virtuale** — `diario:anno:YYYY`, costruito soltanto da attributi indicizzati del Diario.
* **Data compatta** — mese, giorno e giorno della settimana nello stile calendario.
* **Titolo** — `displayName` → nome pagina, visualizzato in grassetto e collegato alla pagina Diario.
* **Immagine** — prima immagine trovata nella pagina, se presente.
* **Snippet** — estratto dal testo della pagina.
* **Caricamento progressivo reale** — all'apertura interroga solo il primo batch; i batch successivi vengono richiesti durante lo scrolling e l'intero dataset viene caricato solo quando serve al filtro.
* **Filtro globale indicizzato** — ricerca con logica AND su path, titolo visualizzato, `displayName`, `description`, tags, luoghi e Viaggio; lo snippet resta solo un elemento visuale.
* **Livello dati riutilizzabile** — raccolta pagine e filtro indicizzato sono separati dal rendering e pronti per le Virtual Page.
* **Navigazione diretta** — i titoli usano `editor.open()` e restano raggiungibili anche dopo caricamento progressivo o filtro.
* **Raggruppamento mensile** — separazione delle pagine per mese e anno.
* **Virtual Page mensile** — il titolo del mese apre `diario:mese:YYYY-MM`.
* **Virtual Page di ricerca** — il filtro può essere aperto come `diario:ricerca:...`, ricalcolato esclusivamente sui dati indicizzati.
* **Tags e luoghi** — visualizzati nelle ultime due righe della scheda con carattere ridotto; i luoghi sono navigabili.
* **Layout responsive** — su smartphone il calendario resta nell'header, mentre snippet, tags e luoghi sfruttano quasi tutta la larghezza della scheda.
* **Ricerca Luoghi** — `${widgets.CercaLuoghi()}` parte con elenco vuoto e mostra soltanto le pagine `luoghi/` che soddisfano la ricerca.

`entry.title` rimane il nome interno usato dal renderer; non corrisponde più a un attributo frontmatter `title` delle pagine Diario.

## Configurazione

Configurazione minima:

```space-lua
config.set("indiceDiario", {
  batchSize = 10,
})
```

Configurazione completa con i valori di default:

```space-lua
config.set("indiceDiario", {
  journalPathPattern = "Diario/#year#-#month#-#day#",
  batchSize          = 10,
  showThumbnails     = true,
  showSnippets       = true,
  snippetStartMarker = "",
  monthNames = {
    "Gennaio", "Febbraio", "Marzo", "Aprile",
    "Maggio", "Giugno", "Luglio", "Agosto",
    "Settembre", "Ottobre", "Novembre", "Dicembre"
  },
  dayNames = {
    "Lunedì", "Martedì", "Mercoledì", "Giovedì",
    "Venerdì", "Sabato", "Domenica"
  },
})
```

`batchSize` determina anche quante pagine vengono richieste all'indice in ogni batch durante la normale navigazione. Il filtro, quando utilizzato, carica invece l'intero insieme dei metadati indicizzati.

La raccolta dati usa `index.subPages("Diario")`; `journalPathPattern` continua a determinare quali sottopagine rappresentano giornate valide.

Il filtro è applicato con un debounce di circa 280 ms per evitare una scansione completa a ogni singolo tasto premuto.

Per compatibilità con la precedente `0.1 alpha`, se `batchSize` non è definito viene ancora accettato `limit` come fallback.

> **note** Lo snippet viene letto soltanto quando la relativa scheda viene renderizzata; non partecipa al filtro.

> **note** Il titolo delle pagine Diario viene letto esclusivamente da `displayName`; non viene interrogato `p.title` e non viene eseguita alcuna scansione di `index.headers()`.

## Uso

```space-lua
${widgets.IndiceDiario()}
```

Virtual Page:

```text
diario:mese:2026-08
diario:ricerca:torino vacanza
```

Ricerca luoghi:

```space-lua
${widgets.CercaLuoghi()}
```

## Implementazione

```space-lua
-- priority: 10

widgets = widgets or {}
indiceDiario = indiceDiario or {}
cercaLuoghi = cercaLuoghi or {}


-- ============================================================
-- CONFIGURAZIONE
-- ============================================================

config.define("indiceDiario", {
  type = "object",
  properties = {
    journalPathPattern = schema.string(),
    batchSize = schema.number(),
    showThumbnails = schema.boolean(),
    showSnippets = schema.boolean(),
    snippetStartMarker = schema.string(),
    monthNames = {
      type = "array",
      items = {
        type = "string"
      }
    },
    dayNames = {
      type = "array",
      items = {
        type = "string"
      }
    }
  }
})

local function indiceDiarioConfig()
  local c =
    config.get("indiceDiario")
    or {}

  return {
    PATTERN =
      c.journalPathPattern
      or "Diario/#year#-#month#-#day#",

    BATCH =
      math.max(
        1,
        math.floor(
          tonumber(c.batchSize)
          or tonumber(c.limit)
          or 10
        )
      ),

    THUMBNAILS =
      c.showThumbnails ~= false,

    SNIPPETS =
      c.showSnippets ~= false,

    SNIPPET_MARKER =
      c.snippetStartMarker
      or "",

    MONTHS =
      c.monthNames
      or {
        "Gennaio", "Febbraio", "Marzo", "Aprile",
        "Maggio", "Giugno", "Luglio", "Agosto",
        "Settembre", "Ottobre", "Novembre", "Dicembre"
      },

    DAYS =
      c.dayNames
      or {
        "Lunedì", "Martedì", "Mercoledì", "Giovedì",
        "Venerdì", "Sabato", "Domenica"
      }
  }
end


-- ============================================================
-- SELEZIONE DELLE PAGINE
-- ============================================================

local function indiceDiarioPattern(pattern)
  local p =
    pattern:gsub(
      "([%(%)%.%%%+%-%[%^%$%?%*])",
      "%%%1"
    )

  p = p:gsub("#weekdayfull#", "[%%a]+")
  p = p:gsub("#monthname#", "[%%a]+")
  p = p:gsub("#monthshort#", "[%%a]+")
  p = p:gsub("#weekday#", "[%%a]+")
  p = p:gsub("#ordinal#", "[%%a]+")
  p = p:gsub("#weekyear#", "%%d%%d%%d%%d")
  p = p:gsub("#weeknum#", "%%d%%d")
  p = p:gsub("#weeknumraw#", "%%d+")
  p = p:gsub("#year#", "%%d%%d%%d%%d")
  p = p:gsub("#month#", "%%d%%d")
  p = p:gsub("#day#", "%%d%%d")
  p = p:gsub("#YY#", "%%d%%d")
  p = p:gsub("#M#", "%%d+")
  p = p:gsub("#D#", "%%d+")
  p = p:gsub("#HH#", "%%d%%d")
  p = p:gsub("#hh#", "%%d%%d")
  p = p:gsub("#mm#", "%%d%%d")
  p = p:gsub("#ss#", "%%d%%d")
  p = p:gsub("#wildcard#", ".*")

  return "^" .. p .. "$"
end

local function indiceDiarioDateFromPath(pageName)
  local y, m, d =
    pageName:match(
      "(%d%d%d%d)%-(%d%d)%-(%d%d)"
    )

  if not y then
    y, m, d =
      pageName:match(
        "(%d%d%d%d)/(%d%d)/(%d%d)"
      )
  end

  if not y then
    return nil
  end

  return
    tonumber(y),
    tonumber(m),
    tonumber(d)
end

local function indiceDiarioWeekday(
  year,
  month,
  day,
  dayNames
)
  local y = year
  local m = month

  if m < 3 then
    m = m + 12
    y = y - 1
  end

  local k = y % 100
  local j =
    math.floor(y / 100)

  local h =
    (
      (
        day
        + math.floor(
          13 * (m + 1) / 5
        )
        + k
        + math.floor(k / 4)
        + math.floor(j / 4)
        - 2 * j
      ) % 7
      + 7
    ) % 7

  local remap =
    { 6, 7, 1, 2, 3, 4, 5 }

  local index =
    remap[h + 1]
    or 1

  return dayNames[index]
    or ""
end

local function indiceDiarioEntry(
  p,
  cfg
)
  if type(p.name) ~= "string" then
    return nil
  end

  local pattern =
    indiceDiarioPattern(
      cfg.PATTERN
    )

  if not p.name:match(pattern) then
    return nil
  end

  local y, m, d = nil, nil, nil
  local dateValue = nil

  if type(p.date) == "string"
    and p.date ~= ""
  then
    y, m, d =
      p.date:match(
        "^(%d%d%d%d)%-(%d%d)%-(%d%d)"
      )

    if y then
      y = tonumber(y)
      m = tonumber(m)
      d = tonumber(d)
      dateValue = p.date
    end
  end

  if not y then
    y, m, d =
      indiceDiarioDateFromPath(
        p.name
      )

    if y and m and d then
      dateValue =
        string.format(
          "%04d-%02d-%02d",
          y,
          m,
          d
        )
    end
  end

  if not y
    or not m
    or not d
    or not dateValue
  then
    return nil
  end

  local title =
    type(p.displayName) == "string"
    and p.displayName ~= ""
    and p.displayName
    or p.name:match(
      "([^/]+)$"
    )
    or p.name

  local entry = {
    path = p.name,
    date = dateValue,
    year = y,
    month = m,
    day = d,
    title = title,
    displayName = p.displayName,
    description = p.description,
    tags = p.tags,
    luoghi = p.luoghi,
    Viaggio = p.Viaggio,
    sortKey =
      y * 10000
      + m * 100
      + d
  }

  entry.indexedSearchText =
    indiceDiario.indexedSearchText(
      entry
    )

  return entry
end

function indiceDiario.entriesBatch(
  cfg,
  beforeDate
)
  cfg =
    cfg
    or indiceDiarioConfig()

  local batchSize = cfg.BATCH
  local pages = nil

  if beforeDate
    and beforeDate ~= ""
  then
    pages = query[[
      from p = index.subPages("Diario")
      where p.date != nil
        and p.date < beforeDate
      order by p.date desc
      limit batchSize
      select {
        name = p.name,
        date = p.date,
        displayName = p.displayName,
        description = p.description,
        tags = p.tags,
        luoghi = p.luoghi,
        Viaggio = p.Viaggio
      }
    ]]
  else
    pages = query[[
      from p = index.subPages("Diario")
      where p.date != nil
      order by p.date desc
      limit batchSize
      select {
        name = p.name,
        date = p.date,
        displayName = p.displayName,
        description = p.description,
        tags = p.tags,
        luoghi = p.luoghi,
        Viaggio = p.Viaggio
      }
    ]]
  end

  local entries = {}

  for _, p in ipairs(pages) do
    local entry =
      indiceDiarioEntry(
        p,
        cfg
      )

    if entry then
      table.insert(
        entries,
        entry
      )
    end
  end

  local cursor = nil

  if pages
    and #pages > 0
  then
    cursor =
      pages[#pages].date
  end

  return {
    entries = entries,
    cursor = cursor,
    hasMore =
      pages
      and #pages >= batchSize
      or false
  }
end

function indiceDiario.entries(cfg)
  cfg =
    cfg
    or indiceDiarioConfig()

  local pages = query[[
    from p = index.subPages("Diario")
    select {
      name = p.name,
      date = p.date,
      displayName = p.displayName,
      description = p.description,
      tags = p.tags,
      luoghi = p.luoghi,
      Viaggio = p.Viaggio
    }
  ]]

  local entries = {}

  for _, p in ipairs(pages) do
    local entry =
      indiceDiarioEntry(
        p,
        cfg
      )

    if entry then
      table.insert(
        entries,
        entry
      )
    end
  end

  table.sort(
    entries,
    function(a, b)
      if a.sortKey == b.sortKey then
        return a.path < b.path
      end

      return a.sortKey > b.sortKey
    end
  )

  return entries
end


-- ============================================================
-- NORMALIZZAZIONE VALORI E FILTRO
-- ============================================================

function indiceDiario.list(value)
  if type(value) == "table" then
    return value
  end

  if type(value) == "string"
    and value ~= ""
  then
    return { value }
  end

  return {}
end

function indiceDiario.valueText(value)
  if type(value) == "string" then
    return value
  end

  if type(value) ~= "table" then
    return ""
  end

  local parts = {}

  for _, item in ipairs(value) do
    if type(item) == "string" then
      table.insert(
        parts,
        item
      )
    elseif item ~= nil then
      table.insert(
        parts,
        tostring(item)
      )
    end
  end

  return table.concat(
    parts,
    " "
  )
end

local function indiceDiarioTerms(value)
  local terms = {}

  local queryText =
    string.lower(
      tostring(
        value
        or ""
      )
    )

  for term in
    queryText:gmatch("%S+")
  do
    table.insert(
      terms,
      term
    )
  end

  return terms
end

local function indiceDiarioMatches(
  text,
  terms
)
  for _, term in ipairs(terms) do
    if not string.find(
      text,
      term,
      1,
      true
    ) then
      return false
    end
  end

  return true
end

function indiceDiario.indexedSearchText(entry)
  local parts = {
    indiceDiario.valueText(
      entry.path
    ),
    indiceDiario.valueText(
      entry.title
    ),
    indiceDiario.valueText(
      entry.displayName
    ),
    indiceDiario.valueText(
      entry.description
    ),
    indiceDiario.valueText(
      entry.tags
    ),
    indiceDiario.valueText(
      entry.luoghi
    ),
    indiceDiario.valueText(
      entry.Viaggio
    )
  }

  return string.lower(
    table.concat(
      parts,
      " "
    )
  )
end

function indiceDiario.filter(
  entries,
  value,
  extraTextFn
)
  local terms =
    indiceDiarioTerms(value)

  if #terms == 0 then
    return entries
  end

  local filtered = {}

  for _, entry in ipairs(entries) do
    local searchText =
      entry.indexedSearchText
      or indiceDiario.indexedSearchText(
        entry
      )

    local matches =
      indiceDiarioMatches(
        searchText,
        terms
      )

    if not matches
      and extraTextFn
    then
      local extraText =
        extraTextFn(entry)

      searchText = searchText
        .. " "
        .. string.lower(
          tostring(
            extraText
            or ""
          )
        )

      matches =
        indiceDiarioMatches(
          searchText,
          terms
        )
    end

    if matches then
      table.insert(
        filtered,
        entry
      )
    end
  end

  return filtered
end

function indiceDiario.filterIndexed(
  entries,
  value
)
  return indiceDiario.filter(
    entries,
    value,
    nil
  )
end

function indiceDiario.month(
  entries,
  year,
  month
)
  local y = tonumber(year)
  local m = tonumber(month)
  local result = {}

  for _, entry in ipairs(entries) do
    if entry.year == y
      and entry.month == m
    then
      table.insert(
        result,
        entry
      )
    end
  end

  return result
end

function indiceDiario.renderVirtual(
  heading,
  entries
)
  local rows = {
    "# " .. heading,
    "",
    tostring(#entries)
      .. (
        #entries == 1
        and " pagina trovata."
        or " pagine trovate."
      ),
    ""
  }

  if #entries == 0 then
    table.insert(
      rows,
      "Nessun risultato."
    )

    return table.concat(
      rows,
      "\n"
    ) .. "\n"
  end

  for _, entry in ipairs(entries) do
    table.insert(
      rows,
      string.format(
        "## %02d/%02d/%04d — [[%s|%s]]",
        entry.day,
        entry.month,
        entry.year,
        entry.path,
        entry.title
      )
    )

    if type(entry.description) == "string"
      and entry.description ~= ""
    then
      table.insert(
        rows,
        entry.description
      )
    end

    local tags =
      indiceDiario.valueText(
        entry.tags
      )

    if tags ~= "" then
      table.insert(
        rows,
        "**Tags:** " .. tags
      )
    end

    local luoghi =
      indiceDiario.valueText(
        entry.luoghi
      )

    if luoghi ~= "" then
      table.insert(
        rows,
        "📍 " .. luoghi
      )
    end

    if type(entry.Viaggio) == "string"
      and entry.Viaggio ~= ""
    then
      table.insert(
        rows,
        "🧭 " .. entry.Viaggio
      )
    end

    table.insert(
      rows,
      ""
    )
  end

  return table.concat(
    rows,
    "\n"
  )
end


-- ============================================================
-- WIKILINK E RICERCA LUOGHI
-- ============================================================

local function indiceDiarioWikiLink(value)
  if type(value) ~= "string"
    or value == ""
  then
    return nil, nil
  end

  local target, label =
    value:match(
      "^%[%[([^]|]+)|([^]]+)%]%]$"
    )

  if target then
    return target, label
  end

  target =
    value:match(
      "^%[%[([^]]+)%]%]$"
    )

  if not target then
    return nil, nil
  end

  return
    target,
    target:match(
      "([^/]+)$"
    ) or target
end

local function cercaLuoghiFirstAlias(value)
  for _, alias in ipairs(
    indiceDiario.list(value)
  ) do
    if type(alias) == "string"
      and alias ~= ""
    then
      return alias
    end
  end

  return nil
end

-- Questa funzione riguarda esclusivamente luoghi/.
-- La semantica title dei Luoghi resta invariata.
local function cercaLuoghiLabel(p)
  if type(p.title) == "string"
    and p.title ~= ""
  then
    return p.title
  end

  if type(p.displayName) == "string"
    and p.displayName ~= ""
  then
    return p.displayName
  end

  local alias =
    cercaLuoghiFirstAlias(
      p.aliases
    )

  if alias then
    return alias
  end

  return
    p.name:match(
      "([^/]+)$"
    )
    or p.name
end

local function cercaLuoghiParentLabel(
  pageName,
  labelsByPath
)
  local relative =
    pageName:gsub(
      "^luoghi/",
      ""
    )

  local parts = {}

  for part in relative:gmatch(
    "[^/]+"
  ) do
    table.insert(
      parts,
      part
    )
  end

  if #parts <= 1 then
    return ""
  end

  local labels = {}
  local path = "luoghi"

  for i = 1, #parts - 1 do
    path = path
      .. "/"
      .. parts[i]

    table.insert(
      labels,
      labelsByPath[path]
      or parts[i]
    )
  end

  return table.concat(
    labels,
    " › "
  )
end

function cercaLuoghi.entries()
  local pages = query[[
    from p = index.subPages("luoghi")
    order by p.name
    select {
      name = p.name,
      title = p.title,
      displayName = p.displayName,
      aliases = p.aliases
    }
  ]]

  local labelsByPath = {}

  for _, p in ipairs(pages) do
    if type(p.name) == "string"
      and p.name ~= ""
    then
      labelsByPath[p.name] =
        cercaLuoghiLabel(p)
    end
  end

  local entries = {}

  for _, p in ipairs(pages) do
    if type(p.name) == "string"
      and p.name ~= ""
    then
      local label =
        labelsByPath[p.name]

      local pathLabel =
        cercaLuoghiParentLabel(
          p.name,
          labelsByPath
        )

      local entry = {
        path = p.name,
        title = label,
        displayName = p.displayName,
        aliases = p.aliases,
        pathLabel = pathLabel
      }

      entry.indexedSearchText =
        string.lower(
          table.concat(
            {
              indiceDiario.valueText(
                p.name
              ),
              indiceDiario.valueText(
                p.title
              ),
              indiceDiario.valueText(
                p.displayName
              ),
              indiceDiario.valueText(
                p.aliases
              ),
              indiceDiario.valueText(
                label
              ),
              indiceDiario.valueText(
                pathLabel
              )
            },
            " "
          )
        )

      table.insert(
        entries,
        entry
      )
    end
  end

  return entries
end

function cercaLuoghi.filter(
  entries,
  value
)
  return indiceDiario.filterIndexed(
    entries,
    value
  )
end


-- ============================================================
-- RIEPILOGO ANNUALE
-- ============================================================

-- Verifica la presenza esatta di un tag.
-- Supporta sia la forma corrente a stringa separata da spazi,
-- sia eventuali liste restituite dall'indice.
local function indiceDiarioHasTag(
  value,
  wanted
)
  if type(value) == "string" then
    for tag in value:gmatch("%S+") do
      if tag == wanted then
        return true
      end
    end

    return false
  end

  if type(value) == "table" then
    for _, tag in ipairs(value) do
      if tag == wanted then
        return true
      end
    end
  end

  return false
end


-- Estrae il target di un wikilink senza leggere la pagina.
local function indiceDiarioAnnualTarget(value)
  if type(value) ~= "string"
    or value == ""
  then
    return nil
  end

  return value:match(
    "^%[%[([^]|]+)"
  )
end


-- Recupera esclusivamente gli attributi indicizzati necessari
-- per il riepilogo di un anno.
function indiceDiario.yearEntries(year)
  local y =
    tonumber(year)

  if not y then
    return {}
  end

  local prefix =
    string.format(
      "%04d-",
      y
    )

  return query[[
    from p = index.subPages("Diario")
    where p.date
      and string.startsWith(
        p.date,
        prefix
      )
    order by p.date, p.name
    select {
      name = p.name,
      date = p.date,
      displayName = p.displayName,
      tags = p.tags,
      luoghi = p.luoghi,
      Viaggio = p.Viaggio
    }
  ]]
end


-- Costruisce il link visuale di una giornata.
local function indiceDiarioAnnualDayLink(p)
  local label =
    type(p.displayName) == "string"
    and p.displayName ~= ""
    and p.displayName
    or p.name:match(
      "([^/]+)$"
    )
    or p.name

  return string.format(
    "[[%s|%s]]",
    p.name,
    label
  )
end


-- Chiave geografica di ordinamento.
--
-- Il parent geografico ha precedenza; all'interno dello stesso
-- ramo i luoghi seguono la data della prima comparsa nell'anno.
local function indiceDiarioGeoParent(target)
  return target:match(
    "^(.*)/[^/]+$"
  ) or target
end


-- Renderizza il riepilogo annuale usando soltanto dati indicizzati.
function indiceDiario.renderYear(year)
  local y =
    tonumber(year)

  if not y then
    return "# Riepilogo\n\nAnno non valido.\n"
  end

  local entries =
    indiceDiario.yearEntries(y)

  local rows = {
    "# Riepilogo " .. tostring(y),
    "",
    tostring(#entries)
      .. (
        #entries == 1
        and " giornata"
        or " giornate"
      ),
    ""
  }

  -- Date da ricordare
  local ricorda = {}

  for _, p in ipairs(entries) do
    if indiceDiarioHasTag(
      p.tags,
      "ricorda"
    ) then
      table.insert(
        ricorda,
        string.format(
          "- %s — %s",
          date.format(p.date),
          indiceDiarioAnnualDayLink(p)
        )
      )
    end
  end

  table.insert(
    rows,
    "## Date da ricordare"
  )
  table.insert(rows, "")

  if #ricorda > 0 then
    table.insert(
      rows,
      table.concat(
        ricorda,
        "\n"
      )
    )
  else
    table.insert(
      rows,
      "Nessuna data registrata."
    )
  end

  table.insert(rows, "")

  -- Viaggi deduplicati, nell'ordine della prima giornata.
  local viaggi = {}
  local seenViaggi = {}

  for _, p in ipairs(entries) do
    local target =
      indiceDiarioAnnualTarget(
        p.Viaggio
      )

    if target
      and not seenViaggi[target]
    then
      table.insert(
        viaggi,
        p.Viaggio
      )

      seenViaggi[target] =
        true
    end
  end

  table.insert(
    rows,
    "## Viaggi"
  )
  table.insert(rows, "")

  if #viaggi > 0 then
    for _, viaggio in ipairs(viaggi) do
      table.insert(
        rows,
        "- " .. viaggio
      )
    end
  else
    table.insert(
      rows,
      "Nessun viaggio registrato."
    )
  end

  table.insert(rows, "")

  -- Luoghi deduplicati.
  --
  -- Per ogni luogo conserva la prima data di comparsa.
  -- L'ordinamento finale è:
  -- 1. ramo geografico;
  -- 2. prima data di visita;
  -- 3. path del luogo come spareggio stabile.
  local placesByTarget = {}

  for _, p in ipairs(entries) do
    if type(p.luoghi) == "table" then
      for _, luogo in ipairs(p.luoghi) do
        local target =
          indiceDiarioAnnualTarget(
            luogo
          )

        if target
          and not placesByTarget[target]
        then
          placesByTarget[target] = {
            target = target,
            link = luogo,
            firstDate = p.date,
            parent =
              indiceDiarioGeoParent(
                target
              )
          }
        end
      end
    elseif type(p.luoghi) == "string"
      and p.luoghi ~= ""
    then
      local target =
        indiceDiarioAnnualTarget(
          p.luoghi
        )

      if target
        and not placesByTarget[target]
      then
        placesByTarget[target] = {
          target = target,
          link = p.luoghi,
          firstDate = p.date,
          parent =
            indiceDiarioGeoParent(
              target
            )
        }
      end
    end
  end

  local places = {}

  for _, luogo in pairs(
    placesByTarget
  ) do
    table.insert(
      places,
      luogo
    )
  end

  table.sort(
    places,
    function(a, b)
      if a.parent ~= b.parent then
        return a.parent < b.parent
      end

      if a.firstDate ~= b.firstDate then
        return a.firstDate < b.firstDate
      end

      return a.target < b.target
    end
  )

  table.insert(
    rows,
    "## Luoghi visitati"
  )
  table.insert(rows, "")

  if #places > 0 then
    for _, luogo in ipairs(places) do
      table.insert(
        rows,
        "- " .. luogo.link
      )
    end
  else
    table.insert(
      rows,
      "Nessun luogo registrato."
    )
  end

  table.insert(rows, "")

  -- Record
  local records = {}

  for _, p in ipairs(entries) do
    if indiceDiarioHasTag(
      p.tags,
      "record"
    ) then
      table.insert(
        records,
        string.format(
          "- %s — %s",
          date.format(p.date),
          indiceDiarioAnnualDayLink(p)
        )
      )
    end
  end

  table.insert(
    rows,
    "## Record"
  )
  table.insert(rows, "")

  if #records > 0 then
    table.insert(
      rows,
      table.concat(
        records,
        "\n"
      )
    )
  else
    table.insert(
      rows,
      "Nessun record registrato."
    )
  end

  return table.concat(
    rows,
    "\n"
  ) .. "\n"
end


-- ============================================================
-- VIRTUAL PAGES
-- ============================================================

virtualPage.define {
  pattern =
    "^diario:anno:(%d%d%d%d)$",

  run = function(year)
    return indiceDiario.renderYear(
      year
    )
  end
}

virtualPage.define {
  pattern =
    "^diario:mese:(%d%d%d%d%-%d%d)$",

  run = function(period)
    local year, month =
      period:match(
        "^(%d%d%d%d)%-(%d%d)$"
      )

    local y = tonumber(year)
    local m = tonumber(month)

    if not y
      or not m
      or m < 1
      or m > 12
    then
      return "# Diario\n\nPeriodo non valido.\n"
    end

    local cfg =
      indiceDiarioConfig()

    local entries =
      indiceDiario.month(
        indiceDiario.entries(cfg),
        y,
        m
      )

    local monthName =
      cfg.MONTHS[m]
      or string.format(
        "%02d",
        m
      )

    return indiceDiario.renderVirtual(
      "Diario — "
        .. monthName
        .. " "
        .. tostring(y),
      entries
    )
  end
}

virtualPage.define {
  pattern =
    "^diario:ricerca:(.+)$",

  run = function(searchText)
    local cfg =
      indiceDiarioConfig()

    local entries =
      indiceDiario.filterIndexed(
        indiceDiario.entries(cfg),
        searchText
      )

    return indiceDiario.renderVirtual(
      "Ricerca Diario — "
        .. searchText,
      entries
    )
  end
}


-- ============================================================
-- CONTENUTO DELLE PAGINE
-- ============================================================

local function indiceDiarioExtractInfo(
  content,
  startMarker
)
  if not content then
    return "", nil, nil
  end

  local body = content

  local _, fmEnd =
    body:find(
      "^%-%-%-.-%-%-%-[\n\r]*"
    )

  if fmEnd then
    body =
      body:sub(
        fmEnd + 1
      )
  end

  local wikiImage =
    body:match(
      "!%[%[([^%]|]+)"
    )

  local markdownImage = nil

  if not wikiImage then
    markdownImage =
      body:match(
        "!%[[^%]]*%]%(([^%s%)]+)%)"
      )
  end

  local snippetBody = body
  local skipFirst = true

  if startMarker
    and startMarker ~= ""
  then
    local markerPos =
      body:find(
        startMarker,
        1,
        true
      )

    if markerPos then
      snippetBody =
        body:sub(
          markerPos
          + #startMarker
        )

      skipFirst = false
    end
  end

  local function cleanSnippetText(value)
    value =
      value:gsub(
        "!%[%[.-%]%]",
        "..."
      )

    value =
      value:gsub(
        "!%[[^%]]*%]%([^%)]+%)",
        "..."
      )

    value =
      value:gsub(
        "%[%[.-%]%]",
        "..."
      )

    value =
      value:gsub(
        "%[[^%]]*%]%([^%)]+%)",
        "..."
      )

    value =
      value:gsub(
        "%s+",
        " "
      )

    value =
      value:gsub(
        "%.%.%.%s+%.%.%.",
        "..."
      )

    return value:match(
      "^%s*(.-)%s*$"
    )
  end

  local parts = {}
  local skipped =
    not skipFirst

  for line in
    snippetBody:gmatch(
      "[^\r\n]+"
    )
  do
    local value =
      line:match(
        "^%s*(.-)%s*$"
      )

    if value
      and value ~= ""
    then
      if not skipped then
        skipped = true
      elseif not value:match(
        "^#+%s"
      ) then
        value =
          cleanSnippetText(value)

        if value
          and value ~= ""
        then
          table.insert(
            parts,
            value
          )

          if #table.concat(
            parts,
            " "
          ) >= 150
          then
            break
          end
        end
      end
    end
  end

  local snippet =
    table.concat(
      parts,
      " "
    )

  snippet =
    cleanSnippetText(snippet)

  if #snippet > 130 then
    snippet =
      snippet:sub(
        1,
        127
      )
      .. "…"
  end

  return
    snippet,
    wikiImage,
    markdownImage
end


-- ============================================================
-- RENDERING
-- ============================================================

local function indiceDiarioThumbnail(
  wikiImage,
  markdownImage
)
  if wikiImage then
    return dom.div {
      class = "id-thumb",
      "![[" .. wikiImage .. "]]"
    }
  end

  if markdownImage then
    return dom.div {
      class = "id-thumb",
      "![](" .. markdownImage .. ")"
    }
  end

  return nil
end

function widgets.IndiceDiario()
  local cfg =
    indiceDiarioConfig()

  local firstBatch =
    indiceDiario.entriesBatch(
      cfg,
      nil
    )

  if not firstBatch
    or not firstBatch.entries
    or #firstBatch.entries == 0
  then
    return widget.htmlBlock(
      dom.div {
        class = "id-index",
        "Nessuna pagina del Diario trovata."
      }
    )
  end

  local contentCache = {}

  local function ensureContent(entry)
    local cached =
      contentCache[entry.path]

    if cached then
      return cached
    end

    local raw =
      space.readPage(
        entry.path
      )

    local snippet,
      wikiImage,
      markdownImage =
        indiceDiarioExtractInfo(
          raw or "",
          cfg.SNIPPET_MARKER
        )

    cached = {
      snippet = snippet or "",
      wikiImage = wikiImage,
      markdownImage = markdownImage
    }

    contentCache[entry.path] =
      cached

    return cached
  end

  local function buildTagsNode(entry)
    local tags =
      indiceDiario.list(
        entry.tags
      )

    if #tags == 0 then
      return nil
    end

    local labels = {}

    for _, tag in ipairs(tags) do
      if type(tag) == "string"
        and tag ~= ""
      then
        if string.startsWith(
          tag,
          "#"
        ) then
          table.insert(
            labels,
            tag
          )
        else
          table.insert(
            labels,
            "#" .. tag
          )
        end
      end
    end

    if #labels == 0 then
      return nil
    end

    return dom.div {
      class = "id-meta-row id-tags",
      __rawText =
        table.concat(
          labels,
          " "
        )
    }
  end

  local function buildLuoghiNode(entry)
    local valori =
      indiceDiario.list(
        entry.luoghi
      )

    if #valori == 0 then
      return nil
    end

    local row =
      dom.div {
        class = "id-meta-row id-luoghi"
      }

    row.appendChild(
      dom.span {
        __rawText = "📍 "
      }
    )

    local added = 0

    for _, value in ipairs(valori) do
      local target, label =
        indiceDiarioWikiLink(value)

      if target then
        if added > 0 then
          row.appendChild(
            dom.span {
              __rawText = " · "
            }
          )
        end

        row.appendChild(
          dom.a {
            class = "id-meta-link",

            onclick = function()
              editor.open(target)
            end,

            __rawText = label
          }
        )

        added = added + 1
      elseif type(value) == "string"
        and value ~= ""
      then
        if added > 0 then
          row.appendChild(
            dom.span {
              __rawText = " · "
            }
          )
        end

        row.appendChild(
          dom.span {
            __rawText = value
          }
        )

        added = added + 1
      end
    end

    if added == 0 then
      return nil
    end

    return row
  end

  local function buildEntryNode(entry)
    local info =
      ensureContent(entry)

    local dayName =
      indiceDiarioWeekday(
        entry.year,
        entry.month,
        entry.day,
        cfg.DAYS
      )

    local monthShort =
      (
        cfg.MONTHS[entry.month]
        or tostring(
          entry.month
        )
      ):sub(
        1,
        3
      ):upper()

    local calendar =
      dom.div {
        class = "id-cal",

        dom.div {
          class = "id-cal-top",
          __rawText = monthShort
        },

        dom.div {
          class = "id-cal-num",
          __rawText =
            tostring(
              entry.day
            )
        },

        dom.div {
          class = "id-cal-bottom",
          __rawText =
            dayName:sub(
              1,
              3
            )
        }
      }

    local titleLink =
      dom.a {
        class = "id-title-link",

        onclick = function()
          editor.open(
            entry.path
          )
        end,

        dom.strong {
          __rawText =
            entry.title
        }
      }

    local titleNode =
      dom.div {
        class = "id-title",
        titleLink
      }

    local row =
      dom.div {
        class = "id-entry",
        calendar,
        titleNode
      }

    if cfg.SNIPPETS
      and info.snippet ~= ""
    then
      row.appendChild(
        dom.div {
          class = "id-snippet",
          __rawText =
            info.snippet
        }
      )
    end

    local tagsNode =
      buildTagsNode(entry)

    if tagsNode then
      row.appendChild(tagsNode)
    end

    local luoghiNode =
      buildLuoghiNode(entry)

    if luoghiNode then
      row.appendChild(luoghiNode)
    end

    if cfg.THUMBNAILS then
      local thumb =
        indiceDiarioThumbnail(
          info.wikiImage,
          info.markdownImage
        )

      if thumb then
        row.appendChild(thumb)
      end
    end

    return row
  end

  local root =
    dom.div {
      class = "id-index"
    }

  local filter =
    dom.input {
      class = "id-filter",
      type = "search",
      placeholder =
        "Filtra il Diario…"
    }

  local openSearch =
    dom.button {
      class = "id-open-search",
      type = "button",
      disabled = true,
      __rawText = "Apri risultati"
    }

  local filterBar =
    dom.div {
      class = "id-filter-bar",
      filter,
      openSearch
    }

  local list =
    dom.div {
      class = "id-list"
    }

  root.appendChild(filterBar)
  root.appendChild(list)

  local currentFilter = ""

  local streamEntries =
    firstBatch.entries

  local streamCursor =
    firstBatch.cursor

  local streamHasMore =
    firstBatch.hasMore

  local allEntries = nil
  local activeEntries =
    streamEntries

  local renderedCount = 0
  local lastMonthKey = nil
  local filtering = false

  local function appendMonth(entry)
    local monthKey =
      tostring(entry.year)
      .. "-"
      .. string.format(
        "%02d",
        entry.month
      )

    if monthKey == lastMonthKey then
      return
    end

    local monthName =
      cfg.MONTHS[entry.month]
      or string.format(
        "%02d",
        entry.month
      )

    list.appendChild(
      dom.div {
        class = "id-month",

        dom.a {
          class = "id-month-link",

          onclick = function()
            editor.open(
              "diario:mese:"
                .. monthKey
            )
          end,

          __rawText =
            monthName
            .. " "
            .. tostring(
              entry.year
            )
        }
      }
    )

    lastMonthKey = monthKey
  end

  local function renderNextBatch()
    if renderedCount >= #activeEntries then
      return
    end

    local last =
      math.min(
        renderedCount
          + cfg.BATCH,
        #activeEntries
      )

    for i =
      renderedCount + 1,
      last
    do
      local entry =
        activeEntries[i]

      appendMonth(entry)

      list.appendChild(
        buildEntryNode(entry)
      )
    end

    renderedCount = last
  end

  local function loadNextStreamBatch()
    if not streamHasMore
      or not streamCursor
      or streamCursor == ""
    then
      return
    end

    local batch =
      indiceDiario.entriesBatch(
        cfg,
        streamCursor
      )

    if not batch
      or not batch.entries
    then
      streamHasMore = false
      return
    end

    for _, entry in ipairs(
      batch.entries
    ) do
      table.insert(
        streamEntries,
        entry
      )
    end

    streamCursor = batch.cursor
    streamHasMore =
      batch.hasMore
      and #batch.entries > 0

    activeEntries = streamEntries
    renderNextBatch()
  end

  local function resetList(entries)
    activeEntries = entries
    renderedCount = 0
    lastMonthKey = nil

    list.replaceChildren()

    if #activeEntries == 0 then
      list.appendChild(
        dom.div {
          class = "id-empty",
          "Nessun risultato."
        }
      )

      return
    end

    renderNextBatch()
    list.scrollTop = 0
  end

  local function ensureAllEntries()
    if allEntries then
      return allEntries
    end

    allEntries =
      indiceDiario.entries(cfg)

    filter.placeholder =
      "Filtra "
      .. #allEntries
      .. " pagine…"

    return allEntries
  end

  list.addEventListener(
    "scroll",
    function()
      local remaining =
        list.scrollHeight
        - list.scrollTop
        - list.clientHeight

      if remaining >= 180 then
        return
      end

      if filtering then
        renderNextBatch()
        return
      end

      if renderedCount < #activeEntries then
        renderNextBatch()
        return
      end

      loadNextStreamBatch()
    end
  )

  local function applyFilter(value)
    local trimmed =
      tostring(
        value
        or ""
      ):match(
        "^%s*(.-)%s*$"
      )

    if not trimmed
      or trimmed == ""
    then
      filtering = false

      if allEntries then
        streamEntries = allEntries
        streamCursor = nil
        streamHasMore = false

        resetList(allEntries)
      else
        resetList(streamEntries)
      end

      return
    end

    filtering = true

    local entries =
      ensureAllEntries()

    local filtered =
      indiceDiario.filterIndexed(
        entries,
        trimmed
      )

    resetList(filtered)
  end

  local filterTimer = nil

  filter.addEventListener(
    "input",
    function(e)
      local value =
        tostring(
          e.target.value
          or ""
        )

      currentFilter = value

      openSearch.disabled =
        value:match(
          "^%s*$"
        ) ~= nil

      if filterTimer then
        js.window.clearTimeout(
          filterTimer
        )
      end

      filterTimer =
        js.window.setTimeout(
          function()
            applyFilter(value)
          end,
          280
        )
    end
  )

  openSearch.addEventListener(
    "click",
    function()
      local searchText =
        currentFilter:match(
          "^%s*(.-)%s*$"
        )

      if searchText
        and searchText ~= ""
      then
        editor.open(
          "diario:ricerca:"
            .. searchText
        )
      end
    end
  )

  renderNextBatch()

  return widget.htmlBlock(root)
end


-- ============================================================
-- WIDGET CERCA LUOGHI
-- ============================================================

function widgets.CercaLuoghi()
  local root =
    dom.div {
      class = "cl-index"
    }

  local filter =
    dom.input {
      class = "cl-filter",
      type = "search",
      placeholder = "Cerca un luogo…"
    }

  local status =
    dom.div {
      class = "cl-status"
    }

  local list =
    dom.div {
      class = "cl-list"
    }

  root.appendChild(filter)
  root.appendChild(status)
  root.appendChild(list)

  local allEntries = nil
  local activeEntries = {}
  local renderedCount = 0
  local batchSize = 25

  local function ensureEntries()
    if allEntries then
      return allEntries
    end

    allEntries =
      cercaLuoghi.entries()

    return allEntries
  end

  local function buildResultNode(entry)
    local row =
      dom.div {
        class = "cl-result",

        onclick = function()
          editor.open(
            entry.path
          )
        end,

        dom.div {
          class = "cl-title",
          __rawText =
            entry.title
        }
      }

    if entry.pathLabel
      and entry.pathLabel ~= ""
    then
      row.appendChild(
        dom.div {
          class = "cl-path",
          __rawText =
            entry.pathLabel
        }
      )
    end

    return row
  end

  local function renderNextBatch()
    if renderedCount >= #activeEntries then
      return
    end

    local last =
      math.min(
        renderedCount + batchSize,
        #activeEntries
      )

    for i =
      renderedCount + 1,
      last
    do
      list.appendChild(
        buildResultNode(
          activeEntries[i]
        )
      )
    end

    renderedCount = last
  end

  local function resetResults(entries)
    activeEntries = entries
    renderedCount = 0
    list.replaceChildren()

    if #entries == 0 then
      status.textContent =
        "Nessun luogo trovato."

      return
    end

    status.textContent =
      tostring(#entries)
      .. (
        #entries == 1
        and " luogo trovato"
        or " luoghi trovati"
      )

    renderNextBatch()
  end

  local function clearResults()
    activeEntries = {}
    renderedCount = 0
    list.replaceChildren()
    status.textContent = ""
  end

  list.addEventListener(
    "scroll",
    function()
      local remaining =
        list.scrollHeight
        - list.scrollTop
        - list.clientHeight

      if remaining < 160 then
        renderNextBatch()
      end
    end
  )

  local function applyFilter(value)
    local trimmed =
      tostring(
        value
        or ""
      ):match(
        "^%s*(.-)%s*$"
      )

    if not trimmed
      or trimmed == ""
    then
      clearResults()
      return
    end

    local entries =
      ensureEntries()

    local filtered =
      cercaLuoghi.filter(
        entries,
        trimmed
      )

    resetResults(filtered)
  end

  local filterTimer = nil

  filter.addEventListener(
    "input",
    function(e)
      local value =
        tostring(
          e.target.value
          or ""
        )

      if value:match("^%s*$") then
        if filterTimer then
          js.window.clearTimeout(
            filterTimer
          )

          filterTimer = nil
        end

        clearResults()
        return
      end

      if filterTimer then
        js.window.clearTimeout(
          filterTimer
        )
      end

      filterTimer =
        js.window.setTimeout(
          function()
            applyFilter(value)
          end,
          250
        )
    end
  )

  return widget.htmlBlock(root)
end
```

## Note della revisione 0.1-10

* `displayName` è il titolo canonico delle pagine `Diario/`.
* Eliminato ogni accesso a `p.title` nelle query rivolte al Diario.
* Eliminata la scansione globale di `index.headers()` usata per recuperare il primo H1.
* Fast path, filtro e Virtual Page usano la stessa sorgente indicizzata.
* Il fallback finale del titolo è l'ultimo segmento del path della pagina.
* `entry.title` resta soltanto un campo interno del renderer.
* `widgets.CercaLuoghi()` mantiene invariata la propria semantica `title → displayName → aliases → nome pagina`, perché opera sulle pagine `luoghi/` e non sul Diario.

## Note della revisione 0.1-09

Migliorata la presentazione gerarchica di `widgets.CercaLuoghi()`.

* la stessa query `index.subPages("luoghi")` costruisce una lookup locale `path -> label`;
* nessuna query aggiuntiva e nessun `space.readPage()`;
* il lookup viene calcolato una sola volta per istanza del widget, al primo utilizzo effettivo della ricerca;
* i codici dei livelli geografici vengono visualizzati usando `title` → `displayName` → `aliases` → nome pagina;
* il breadcrumb mostra soltanto gli antenati, escludendo il risultato corrente.

## Note della revisione 0.1-08

Aggiunto `widgets.CercaLuoghi()` con caricamento lazy, ricerca indicizzata e rendering progressivo dei risultati.

## Note della revisione 0.1-07

Ottimizzato il caricamento iniziale: `widgets.IndiceDiario()` usa `indiceDiario.entriesBatch()` e richiede soltanto le prime `batchSize` pagine; il dataset completo viene caricato soltanto quando serve al filtro.

## Note della revisione 0.1-06

Aggiunte le Virtual Page `diario:mese:YYYY-MM` e `diario:ricerca:...`; il filtro usa esclusivamente gli attributi indicizzati.

## Note della revisione 0.1-05

Resa robusta la costruzione della stringa di ricerca tramite `indiceDiario.valueText()` e `table.concat()`.

## Note della revisione 0.1-04

Separato il livello dati/filtro dal rendering del widget; la sorgente delle pagine è `index.subPages("Diario")`.

## Note della revisione 0.1-03

Il rendering delle card usa CSS Grid con layout responsive per smartphone.

## Novità 0.1-11

Aggiunta la Virtual Page:

```text
diario:anno:YYYY
```

Il riepilogo annuale usa una sola query su `index.subPages("Diario")` e soltanto gli attributi indicizzati `date`, `displayName`, `tags`, `Viaggio` e `luoghi`.

Le sezioni sono:

* **Date da ricordare** — giornate con tag esatto `ricorda`;
* **Viaggi** — `Viaggio` deduplicato, mantenendo l'ordine della prima giornata;
* **Luoghi visitati** — `luoghi` deduplicati; ordinamento per ramo geografico e, all'interno dello stesso ramo, per prima data di visita;
* **Record** — giornate con tag esatto `record`.

Non vengono letti H1, corpo Markdown, immagini o gallerie.

## Space Style

```space-style
:root {
  --id-border: var(--modal-border-color);
  --id-bg: var(--top-background-color);
  --id-text: var(--root-color);
  --id-muted: color-mix(
    in srgb,
    var(--root-color) 65%,
    transparent
  );
  --id-accent: var(--ui-accent-color);
  --id-accent-text: var(
    --ui-accent-contrast-color,
    white
  );
  --id-card: color-mix(
    in srgb,
    var(--top-background-color) 92%,
    var(--root-color) 8%
  );
  --id-hover: color-mix(
    in srgb,
    var(--top-background-color) 86%,
    var(--root-color) 14%
  );
}

.id-index {
  display: flex;
  flex-direction: column;
  gap: 8px;
  font-size: inherit;
}

.id-filter-bar {
  display: grid;
  grid-template-columns: minmax(0, 1fr) auto;
  gap: 7px;
}

.id-filter,
.cl-filter {
  width: 100%;
  min-width: 0;
  box-sizing: border-box;
  padding: 7px 9px;
  border: 1px solid var(--id-border);
  border-radius: 7px;
  background: var(--id-bg);
  color: var(--id-text);
  font: inherit;
  outline: none;
}

.id-filter:focus,
.cl-filter:focus {
  border-color: var(--id-accent);
}

.id-open-search {
  box-sizing: border-box;
  padding: 7px 10px;
  border: 1px solid var(--id-border);
  border-radius: 7px;
  background: var(--id-card);
  color: var(--id-text);
  font: inherit;
  cursor: pointer;
}

.id-open-search:hover:not(:disabled) {
  background: var(--id-hover);
}

.id-open-search:disabled {
  opacity: 0.45;
  cursor: default;
}

.id-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
  max-height: min(60vh, 560px);
  overflow-y: auto;
  overflow-x: hidden;
  padding-right: 3px;
  scrollbar-gutter: stable;
}

.id-list::-webkit-scrollbar,
.cl-list::-webkit-scrollbar {
  width: 5px;
}

.id-list::-webkit-scrollbar-track,
.cl-list::-webkit-scrollbar-track {
  background: transparent;
}

.id-list::-webkit-scrollbar-thumb,
.cl-list::-webkit-scrollbar-thumb {
  background: var(--id-border);
  border-radius: 3px;
}

.id-empty {
  padding: 18px 8px;
  color: var(--id-muted);
  text-align: center;
}

.id-month {
  margin-top: 5px;
  padding: 3px 1px;
  color: var(--id-muted);
  font-size: inherit;
  font-weight: 700;
  letter-spacing: 0.08em;
  text-transform: uppercase;
}

.id-month-link {
  color: inherit;
  text-decoration: none;
  cursor: pointer;
}

.id-month-link:hover {
  text-decoration: underline;
}

.id-entry {
  display: grid;
  grid-template-columns: 50px minmax(0, 1fr) auto;
  grid-template-areas:
    "cal title thumb"
    "cal snippet thumb"
    "cal tags thumb"
    "cal luoghi thumb";
  column-gap: 10px;
  row-gap: 4px;
  align-items: center;
  min-width: 0;
  padding: 8px 9px;
  border: 1px solid var(--id-border);
  border-radius: 8px;
  background: var(--id-card);
}

.id-entry:hover {
  background: var(--id-hover);
}

.id-hidden {
  display: none;
}

.id-cal {
  grid-area: cal;
  width: 50px;
  min-height: 62px;
  align-self: center;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  border: 1px solid var(--id-border);
  border-radius: 7px;
}

.id-cal-top {
  padding: 3px 2px 2px;
  background: var(--id-accent);
  color: var(--id-accent-text);
  font-size: 0.65em;
  font-weight: 800;
  line-height: 1;
  letter-spacing: 0.07em;
  text-align: center;
}

.id-cal-num {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  color: var(--id-text);
  background: var(--id-bg);
  font-size: 1.45em;
  font-weight: 700;
  line-height: 1;
}

.id-cal-bottom {
  padding: 2px 2px 3px;
  color: var(--id-muted);
  background: var(--id-bg);
  font-size: 0.65em;
  font-weight: 700;
  line-height: 1;
  letter-spacing: 0.05em;
  text-align: center;
}

.id-title {
  grid-area: title;
  min-width: 0;
  font-size: inherit;
  font-weight: 700;
  line-height: 1.35;
  overflow-wrap: anywhere;
}

.id-title-link {
  color: inherit;
  text-decoration: none;
  cursor: pointer;
}

.id-title-link:hover {
  text-decoration: underline;
}

.id-title p,
.id-snippet p,
.id-thumb p {
  margin: 0;
}

.id-snippet {
  grid-area: snippet;
  min-width: 0;
  color: var(--id-muted);
  font-size: inherit;
  line-height: 1.4;
  overflow-wrap: anywhere;
}

.id-meta-row {
  min-width: 0;
  color: var(--id-muted);
  font-size: 0.78em;
  line-height: 1.3;
  overflow-wrap: anywhere;
}

.id-tags {
  grid-area: tags;
  margin-top: 2px;
}

.id-luoghi {
  grid-area: luoghi;
}

.id-meta-link {
  color: inherit;
  text-decoration: none;
  cursor: pointer;
}

.id-meta-link:hover {
  text-decoration: underline;
}

.id-thumb {
  grid-area: thumb;
  width: 56px;
  height: 56px;
  align-self: center;
  overflow: hidden;
  border: 1px solid var(--id-border);
  border-radius: 6px;
  background: var(--id-bg);
}

.id-thumb img {
  display: block;
  width: 100%;
  height: 100%;
  margin: 0;
  object-fit: cover;
}

.cl-index {
  display: flex;
  flex-direction: column;
  gap: 7px;
  font-size: inherit;
}

.cl-status {
  min-height: 1.2em;
  color: var(--id-muted);
  font-size: 0.8em;
}

.cl-list {
  display: flex;
  flex-direction: column;
  gap: 5px;
  max-height: min(60vh, 560px);
  overflow-y: auto;
  overflow-x: hidden;
  padding-right: 3px;
  scrollbar-gutter: stable;
}

.cl-list:empty {
  display: none;
}

.cl-result {
  min-width: 0;
  padding: 7px 9px;
  border: 1px solid var(--id-border);
  border-radius: 7px;
  background: var(--id-card);
  cursor: pointer;
}

.cl-result:hover {
  background: var(--id-hover);
}

.cl-title {
  min-width: 0;
  font-weight: 700;
  line-height: 1.3;
  overflow-wrap: anywhere;
}

.cl-path {
  margin-top: 2px;
  color: var(--id-muted);
  font-size: 0.78em;
  line-height: 1.3;
  overflow-wrap: anywhere;
}

@media (max-width: 520px) {
  .cl-result {
    padding: 8px 9px;
  }

  .cl-path {
    font-size: 0.8em;
  }

  .id-filter-bar {
    grid-template-columns: 1fr;
  }

  .id-open-search {
    width: 100%;
  }

  .id-entry {
    grid-template-columns: 50px minmax(0, 1fr) auto;
    grid-template-areas:
      "cal title title"
      "snippet snippet thumb"
      "tags tags tags"
      "luoghi luoghi luoghi";
    column-gap: 8px;
    row-gap: 5px;
    align-items: start;
  }

  .id-cal {
    align-self: start;
  }

  .id-title {
    align-self: start;
  }

  .id-snippet {
    display: -webkit-box;
    -webkit-box-orient: vertical;
    -webkit-line-clamp: 3;
    overflow: hidden;
  }

  .id-thumb {
    width: 48px;
    height: 48px;
    align-self: start;
  }

  .id-meta-row {
    font-size: 0.8em;
  }
}
```
