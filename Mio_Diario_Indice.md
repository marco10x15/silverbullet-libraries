---
name: "Library/MG/Mio_Diario_Indice"
tags: meta/library
description: "Indice inline delle pagine Diario con data, titolo, immagine, snippet e filtro."
version: "0.1-18"
versionDate: 2026-09-02
pageDecoration.prefix: "📔 "
share.uri: "github:marco10x15/silverbullet-libraries/Mio_Diario_Indice.md"
---

# 📔 IndiceDiario

**IndiceDiario** visualizza direttamente in una pagina SilverBullet un indice compatto delle pagine del Diario.

**Versione:** 0.1-18 — 02.09.2026

La libreria è autonoma e non dipende da Journal Explorer.

## Main Features

* **Indice inline** — inseribile con `${widgets.IndiceDiario()}`.
* **Riepilogo annuale virtuale** — `diario:anno:YYYY`, costruito soltanto da attributi indicizzati del Diario.
* **Link annuale con fallback** — l'anno nell'indice apre `riepiloghi/YYYY` quando la pagina fisica esiste; altrimenti apre `Diario:YYYY`.
* **Riepilogo annuale incorporabile** — `${widgets.RiepilogoAnno()}` inserisce nella pagina fisica `riepiloghi/YYYY` lo stesso riepilogo automatico della Virtual Page.
* **Data compatta** — mese, giorno e giorno della settimana nello stile calendario.
* **Titolo** — `displayName` → nome pagina, visualizzato in grassetto e collegato alla pagina Diario.
* **Description** — `description` è l’unica fonte del testo sintetico; se manca viene mostrato `Descrizione non presente`.
* **Rendering progressivo** — all’apertura carica i metadati indicizzati del Diario e l’indice dei documenti `media/`, ma renderizza soltanto il primo batch; le schede successive vengono costruite durante lo scrolling.
* **Filtro globale indicizzato** — ricerca con logica AND su path, titolo visualizzato, `displayName`, `description`, tags, luoghi e Viaggio.
* **Livello dati riutilizzabile** — raccolta pagine e filtro indicizzato sono separati dal rendering e pronti per le Virtual Page.
* **Navigazione diretta** — i titoli usano `editor.open()` e restano raggiungibili anche dopo caricamento progressivo o filtro.
* **Raggruppamento mensile** — separazione delle pagine per mese e anno.
* **Virtual Page mensile** — il nome del mese apre `Diario:YYYY:MM`; la forma storica `diario:mese:YYYY-MM` resta disponibile.
* **Virtual Page di ricerca** — il filtro può essere aperto come `diario:ricerca:...`, ricalcolato esclusivamente sui dati indicizzati.
* **Tags e luoghi** — visualizzati nelle ultime due righe della scheda con carattere ridotto; i luoghi sono navigabili.
* **Galleria** — se esistono immagini `media/YYYYMMDD...` (`jpg`, `jpeg`, `png`, `webp`, `gif`), la scheda mostra `📷 Galleria` e apre la Virtual Page `gallery:...` già gestita da `Mio_Diario_Collage`.
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
  batchSize          = 10,
  showSnippets       = true,
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

`batchSize` determina soltanto quante schede vengono renderizzate alla volta. I metadati di tutte le pagine Diario vengono caricati con una sola query iniziale su `index.subPages("Diario")`.

Sono considerate giornate valide tutte le sottopagine di `Diario/` che possiedono l'attributo indicizzato `date`. Non viene più applicato alcun filtro sul formato del path.

Il filtro è applicato con un debounce di circa 280 ms per evitare una scansione completa a ogni singolo tasto premuto.

Per compatibilità con le revisioni precedenti, se `batchSize` non è definito viene ancora accettato `limit` come fallback per il solo rendering progressivo.

> **note** `description` è già disponibile nell’indice e non richiede letture del corpo Markdown. Se manca o è vuota viene mostrato `Descrizione non presente`.

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
    batchSize = schema.number(),
    showSnippets = schema.boolean(),
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
    BATCH =
      math.max(
        1,
        math.floor(
          tonumber(c.batchSize)
          or tonumber(c.limit)
          or 10
        )
      ),
    SNIPPETS =
      c.showSnippets ~= false,
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
  if type(p.name) ~= "string"
    or type(p.date) ~= "string"
    or p.date == ""
  then
    return nil
  end

  local y, m, d =
    p.date:match(
      "^(%d%d%d%d)%-(%d%d)%-(%d%d)"
    )

  if not y then
    return nil
  end

  y = tonumber(y)
  m = tonumber(m)
  d = tonumber(d)

  if not y
    or not m
    or not d
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
    date = p.date,
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

function indiceDiario.entries(cfg)
  cfg =
    cfg
    or indiceDiarioConfig()

  local pages = query[[
    from p = index.subPages("Diario")
    where p.date != nil
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

    local description =
      type(entry.description) == "string"
      and entry.description:match(
        "^%s*(.-)%s*$"
      )
      or ""

    table.insert(
      rows,
      description ~= ""
      and description
      or "Descrizione non presente"
    )

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


-- Verifica se una pagina Diario contiene il luogo richiesto
-- oppure un suo discendente geografico.
local function indiceDiarioEntryHasPlace(
  p,
  pageName
)
  if not p
    or type(pageName) ~= "string"
    or pageName == ""
  then
    return false
  end

  local values = p.luoghi

  if type(values) == "string" then
    values = { values }
  end

  if type(values) ~= "table" then
    return false
  end

  local prefix =
    pageName .. "/"

  for _, value in ipairs(values) do
    local target =
      indiceDiarioAnnualTarget(
        value
      )

    if target == pageName
      or (
        target
        and string.startsWith(
          target,
          prefix
        )
      )
    then
      return true
    end
  end

  return false
end


-- Restituisce il primo livello geografico sotto luoghi/.
-- Esempio:
-- luoghi/ITA/21/Torino -> luoghi/ITA
local function indiceDiarioCountryPath(target)
  if type(target) ~= "string" then
    return nil
  end

  local country =
    target:match(
      "^luoghi/([^/]+)"
    )

  if not country then
    return nil
  end

  return "luoghi/" .. country
end


-- Restituisce il label indicizzato di una pagina luogo.
local function indiceDiarioPlaceLabel(pageName)
  local pages = query[[
    from p = index.pages()
    where p.name == pageName
    select {
      name = p.name,
      displayName = p.displayName,
      aliases = p.aliases
    }
    limit 1
  ]]

  local p =
    pages
    and pages[1]
    or nil

  if p then
    if type(p.displayName) == "string"
      and p.displayName ~= ""
    then
      return p.displayName
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

  return pageName:match(
    "([^/]+)$"
  ) or pageName
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
-- Restituisce la pagina fisica riepiloghi/YYYY
-- soltanto quando esiste già nello Space.
function indiceDiario.yearSummaryPage(year)
  local y =
    tonumber(year)

  if not y then
    return nil
  end

  local pageName =
    "riepiloghi/"
    .. string.format(
      "%04d",
      y
    )

  local pages = query[[
    from p = index.pages()
    where p.name == pageName
    select {
      name = p.name
    }
    limit 1
  ]]

  if pages
    and pages[1]
    and pages[1].name == pageName
  then
    return pageName
  end

  return nil
end


-- Apre il riepilogo fisico dell'anno quando esiste.
-- In assenza della pagina Markdown apre la Virtual Page annuale.
function indiceDiario.openYear(year)
  local y =
    tonumber(year)

  if not y then
    return
  end

  local physical =
    indiceDiario.yearSummaryPage(
      y
    )

  if physical then
    editor.open(
      physical
    )
  else
    editor.open(
      "Diario:"
      .. string.format(
        "%04d",
        y
      )
    )
  end
end


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

  -- Stati visitati.
  --
  -- Viene mantenuto soltanto il primo livello sotto luoghi/.
  -- Ogni Stato apre la Virtual Page di dettaglio per l'anno.
  local countriesByPath = {}

  for _, p in ipairs(entries) do
    local values = p.luoghi

    if type(values) == "string" then
      values = { values }
    end

    if type(values) == "table" then
      for _, luogo in ipairs(values) do
        local target =
          indiceDiarioAnnualTarget(
            luogo
          )

        local countryPath =
          indiceDiarioCountryPath(
            target
          )

        if countryPath
          and not countriesByPath[countryPath]
        then
          countriesByPath[countryPath] = {
            path = countryPath,
            firstDate = p.date
          }
        end
      end
    end
  end

  local countries = {}

  for _, country in pairs(
    countriesByPath
  ) do
    table.insert(
      countries,
      country
    )
  end

  table.sort(
    countries,
    function(a, b)
      if a.firstDate ~= b.firstDate then
        return a.firstDate < b.firstDate
      end

      return a.path < b.path
    end
  )

  table.insert(
    rows,
    "## Luoghi visitati"
  )
  table.insert(rows, "")

  if #countries > 0 then
    for _, country in ipairs(countries) do
      local label =
        indiceDiarioPlaceLabel(
          country.path
        )

      table.insert(
        rows,
        string.format(
          "- [[diario:luogo:%04d:%s|%s]]",
          y,
          country.path,
          label
        )
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


-- Inserisce il riepilogo automatico dell'anno nella pagina
-- fisica riepiloghi/YYYY.
--
-- Non crea né modifica alcuna pagina.
function widgets.RiepilogoAnno()
  local current =
    editor.getCurrentPage()

  local year =
    type(current) == "string"
    and current:match(
      "^riepiloghi/(%d%d%d%d)$"
    )
    or nil

  if not year then
    return widget.markdownBlock(
      "_RiepilogoAnno è utilizzabile nelle pagine `riepiloghi/YYYY`._"
    )
  end

  return widget.markdownBlock(
    indiceDiario.renderYear(
      year
    )
  )
end


-- Costruisce il dettaglio annuale per un luogo e i suoi
-- discendenti geografici.
--
-- Esempio:
-- diario:luogo:2025:luoghi/ITA
function indiceDiario.renderYearPlace(
  year,
  pageName
)
  local y =
    tonumber(year)

  if not y
    or type(pageName) ~= "string"
    or not string.startsWith(
      pageName,
      "luoghi/"
    )
  then
    return "# Luogo visitato\n\nParametri non validi.\n"
  end

  local entries =
    indiceDiario.yearEntries(y)

  local filtered = {}

  for _, p in ipairs(entries) do
    if indiceDiarioEntryHasPlace(
      p,
      pageName
    ) then
      table.insert(
        filtered,
        p
      )
    end
  end

  local label =
    indiceDiarioPlaceLabel(
      pageName
    )

  local rows = {
    "# "
      .. label
      .. " — "
      .. tostring(y),
    ""
  }

  -- Viaggi deduplicati nell'ordine della prima giornata.
  local viaggi = {}
  local seenViaggi = {}

  for _, p in ipairs(filtered) do
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

  -- Luoghi deduplicati appartenenti al ramo richiesto.
  -- L'ordinamento resta geografico e poi temporale.
  local placesByTarget = {}
  local prefix =
    pageName .. "/"

  for _, p in ipairs(filtered) do
    local values = p.luoghi

    if type(values) == "string" then
      values = { values }
    end

    if type(values) == "table" then
      for _, luogo in ipairs(values) do
        local target =
          indiceDiarioAnnualTarget(
            luogo
          )

        if target
          and (
            target == pageName
            or string.startsWith(
              target,
              prefix
            )
          )
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
    "^diario:luogo:(.+)$",

  run = function(spec)
    local year, pageName =
      spec:match(
        "^(%d%d%d%d):(luoghi/.+)$"
      )

    return indiceDiario.renderYearPlace(
      year,
      pageName
    )
  end
}

virtualPage.define {
  pattern =
    "^diario:anno:(%d%d%d%d)$",

  run = function(year)
    return indiceDiario.renderYear(
      year
    )
  end
}

function indiceDiario.renderMonth(period)
  local year, month =
    tostring(
      period
      or ""
    ):match(
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


virtualPage.define {
  pattern =
    "^diario:mese:(%d%d%d%d%-%d%d)$",

  run = function(period)
    return indiceDiario.renderMonth(
      period
    )
  end
}


-- Alias compatti usati dall'indice visuale:
--
-- Diario:2026
-- Diario:2026:07
--
-- Le Virtual Page precedenti restano disponibili per compatibilità.
virtualPage.define {
  pattern =
    "^Diario:(.+)$",

  run = function(spec)
    local year =
      spec:match(
        "^(%d%d%d%d)$"
      )

    if year then
      return indiceDiario.renderYear(
        year
      )
    end

    local y, m =
      spec:match(
        "^(%d%d%d%d):(%d%d)$"
      )

    if y and m then
      return indiceDiario.renderMonth(
        y .. "-" .. m
      )
    end

    return "# Diario\n\nPeriodo non valido.\n"
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
-- GALLERIE DIARIO
-- ============================================================

local function indiceDiarioIsImageExtension(extension)
  if type(extension) ~= "string" then
    return false
  end

  local ext =
    string.lower(extension)

  return ext == "jpg"
    or ext == "jpeg"
    or ext == "png"
    or ext == "webp"
    or ext == "gif"
end


-- Costruisce una lookup YYYYMMDD -> true con una sola query
-- sull'indice dei documenti. Non legge i file media.
--
-- L'estensione usa direttamente il campo indicizzato
-- document.extension di SilverBullet.
function indiceDiario.galleryDates()
  local documents = query[[
    from d = index.documents()
    where string.startsWith(
      d.name,
      "media/"
    )
    select {
      name = d.name,
      extension = d.extension
    }
  ]]

  local result = {}

  for _, d in ipairs(documents) do
    if indiceDiarioIsImageExtension(
      d.extension
    ) then
      local dayKey =
        d.name:match(
          "^media/(%d%d%d%d%d%d%d%d)"
        )

      if dayKey then
        result[dayKey] = true
      end
    end
  end

  return result
end


local function indiceDiarioEntryDayKey(entry)
  if not entry
    or type(entry.date) ~= "string"
  then
    return nil
  end

  local y, m, d =
    entry.date:match(
      "^(%d%d%d%d)%-(%d%d)%-(%d%d)"
    )

  if not y then
    return nil
  end

  return y .. m .. d
end


-- ============================================================
-- RENDERING
-- ============================================================

function widgets.IndiceDiario()
  local cfg =
    indiceDiarioConfig()

  -- Metadati Diario e documenti media vengono letti soltanto
  -- dagli indici SilverBullet. Nessun corpo Markdown viene letto.
  local allEntries = nil
  local galleryDates = {}

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

  local function buildGalleryNode(entry)
    local dayKey =
      indiceDiarioEntryDayKey(
        entry
      )

    if not dayKey
      or not galleryDates[dayKey]
    then
      return nil
    end

    return dom.div {
      class = "id-meta-row id-gallery",

      dom.a {
        class = "id-meta-link",

        onclick = function()
          editor.open(
            "gallery:"
              .. entry.path
          )
        end,

        __rawText = "📷 Galleria"
      }
    }
  end

  local function buildEntryNode(entry)
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

    if cfg.SNIPPETS then
      local description =
        type(entry.description) == "string"
        and entry.description:match(
          "^%s*(.-)%s*$"
        )
        or ""

      if description == "" then
        description =
          "Descrizione non presente"
      end

      row.appendChild(
        dom.div {
          class = "id-snippet",
          __rawText =
            description
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

    local galleryNode =
      buildGalleryNode(entry)

    if galleryNode then
      row.appendChild(
        galleryNode
      )
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
      disabled = true,
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

  local loading =
    dom.div {
      class = "id-loading",
      __rawText =
        "Lettura pagine Diario in corso…"
    }

  local list =
    dom.div {
      class = "id-list"
    }

  root.appendChild(filterBar)
  root.appendChild(loading)
  root.appendChild(list)

  local currentFilter = ""
  local activeEntries = {}
  local renderedCount = 0
  local lastMonthKey = nil

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
              "Diario:"
                .. tostring(
                  entry.year
                )
                .. ":"
                .. string.format(
                  "%02d",
                  entry.month
                )
            )
          end,

          __rawText =
            monthName
        },

        dom.span {
          __rawText = " "
        },

        dom.a {
          class = "id-year-link",

          onclick = function()
            indiceDiario.openYear(
              entry.year
            )
          end,

          __rawText =
            tostring(
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

  list.addEventListener(
    "scroll",
    function()
      local remaining =
        list.scrollHeight
        - list.scrollTop
        - list.clientHeight

      if remaining < 180 then
        renderNextBatch()
      end
    end
  )

  local function applyFilter(value)
    if not allEntries then
      return
    end

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
      resetList(
        allEntries
      )

      return
    end

    local filtered =
      indiceDiario.filterIndexed(
        allEntries,
        trimmed
      )

    resetList(
      filtered
    )
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

  js.window.setTimeout(
    function()
      local entries =
        indiceDiario.entries(cfg)

      allEntries =
        entries
        or {}

      galleryDates =
        indiceDiario.galleryDates()

      loading.replaceChildren()

      if #allEntries == 0 then
        list.replaceChildren(
          dom.div {
            class = "id-empty",
            "Nessuna pagina del Diario trovata."
          }
        )

        return
      end

      filter.disabled = false

      filter.placeholder =
        "Filtra "
        .. #allEntries
        .. " pagine…"

      resetList(
        allEntries
      )
    end,
    20
  )

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

## Revisione 0.1-18

Semplificato `widgets.IndiceDiario()` eliminando ogni lettura del corpo delle pagine.

Modifiche principali:

* `description` è ora l’unica fonte del testo sintetico della scheda;
* se `description` manca o è vuota viene mostrato `Descrizione non presente`;
* eliminate completamente le thumbnail;
* eliminate tutte le chiamate `space.readPage()` dall’indice;
* rimossi `indiceDiarioExtractInfo()`, `contentCache`, `snippetStartMarker`, `showThumbnails` e la relativa configurazione;
* le Virtual Page mensili e di ricerca mostrano anch’esse `Descrizione non presente` quando necessario;
* aggiunta una sola query indicizzata su `index.documents()` per individuare le immagini sotto `media/`;
* sono riconosciute come immagini soltanto le estensioni già usate da `Mio_Diario_Collage`: `jpg`, `jpeg`, `png`, `webp`, `gif`;
* un documento `media/YYYYMMDD...` rende disponibile il link `📷 Galleria` per la giornata corrispondente;
* il link apre la Virtual Page già esistente `gallery:<pagina Diario>` senza duplicare il rendering della galleria.

La scheda dell’Indice è quindi costruita soltanto da dati indicizzati; il corpo Markdown non viene più letto.

## Revisione 0.1-17

Aggiunto `widgets.RiepilogoAnno()` per le pagine fisiche:

```text
riepiloghi/YYYY
```

Il widget ricava automaticamente l'anno dalla pagina corrente tramite `editor.getCurrentPage()` e riusa `indiceDiario.renderYear()`. La pagina fisica può quindi contenere semplicemente:

```space-lua
${widgets.RiepilogoAnno()}
```

Il link dell'anno nell'indice usa ora questa logica:

```text
riepiloghi/YYYY esiste
        ↓ sì
apri riepiloghi/YYYY

        ↓ no
apri Diario:YYYY
```

La verifica avviene al click tramite l'indice delle pagine; non viene creata né modificata alcuna pagina Markdown.

Dal frontmatter della libreria sono stati inoltre rimossi i metadati locali di sincronizzazione `share.hash` e `share.mode`.

`share.uri` rimane invariato.

## Revisione 0.1-16

Aggiunto un bootstrap visuale del widget: prima della query iniziale viene mostrato il messaggio **“Lettura pagine Diario in corso…”**. Il filtro resta disabilitato finché il dataset indicizzato non è disponibile.

L'intestazione mensile è ora composta da due link distinti:

```text
Luglio 2026
```

* `Luglio` → `Diario:2026:07`
* `2026` → `Diario:2026`

Sono state aggiunte le corrispondenti Virtual Page compatte:

```text
Diario:YYYY
Diario:YYYY:MM
```

Le forme già esistenti `diario:anno:YYYY` e `diario:mese:YYYY-MM` rimangono disponibili per compatibilità.

## Revisione 0.1-15

Semplificata la costruzione dell'indice eliminando completamente la paginazione delle query.

La nuova architettura è:

```text
index.subPages("Diario")
        ↓
una sola query dei metadati indicizzati
        ↓
ordinamento completo in memoria
        ↓
rendering di batchSize schede alla volta
```

Modifiche principali:

* eliminata `indiceDiario.entriesBatch()`;
* eliminati cursori, `streamEntries`, `streamCursor`, `streamHasMore` e le query successive durante lo scrolling;
* il filtro lavora direttamente sul dataset già caricato e non esegue più una query completa alla prima ricerca;
* lo scrolling mantiene il rendering progressivo e quindi `space.readPage()` continua a essere eseguito soltanto per le schede effettivamente visualizzate;
* rimosso `journalPathPattern`;
* una pagina Diario è ora valida quando appartiene a `Diario/` e possiede `p.date`;
* rimossi parsing e fallback della data dal nome della pagina.

Questo rende coerenti indice inline, ricerca e Virtual Page mensile anche per pagine con suffisso nel nome, ad esempio `Diario/YYYY-MM-DD-...`.

La revisione 0.1-14 e i precedenti tentativi di paginazione tramite cursore sono quindi superati.

## Correzione 0.1-14

Corretta la paginazione progressiva di `widgets.IndiceDiario()`.

Il cursore dei batch non usa più `p.date`, ma il nome univoco della pagina `p.name`.

La precedente logica:

```text
order by p.date desc
p.date < cursor
```

era fragile perché `date` non è necessariamente univoca: nel dataset corrente esistono più giornate con due pagine aventi la stessa data. Inoltre il cursore dipendeva dal tipo restituito dall'indice per `p.date`.

La nuova logica usa:

```text
order by p.name desc
p.name < cursor
```

Poiché le pagine Diario normalizzate hanno path `Diario/YYYY-MM-DD...`, l'ordine del nome coincide con l'ordine cronologico e, nello stesso giorno, rimane deterministico. `p.name` è inoltre univoco, quindi nessuna pagina può essere saltata tra due batch.

Non cambia il caricamento progressivo né il numero di pagine renderizzate per batch.

## Correzione 0.1-13

Corretta la cattura dei parametri della Virtual Page `diario:luogo:YYYY:luoghi/...`.

La definizione ora usa un solo gruppo catturato nella `virtualPage.define` e separa internamente:

```text
YYYY:luoghi/...
```

in:

```text
year
pageName
```

Questo evita che uno dei due parametri risulti `nil` durante la generazione della Virtual Page, mantenendo invariato il nome pubblico, ad esempio:

```text
diario:luogo:2023:luoghi/HUN
```

## Novità 0.1-12

Il riepilogo `diario:anno:YYYY` mostra ora nella sezione **Luoghi visitati** soltanto gli Stati, deduplicati nell'ordine della prima visita dell'anno.

Ogni Stato apre una nuova Virtual Page:

```text
diario:luogo:YYYY:luoghi/XXX
```

Esempio:

```text
diario:luogo:2025:luoghi/ITA
```

La pagina di dettaglio contiene:

* **Viaggi** — deduplicati nell'ordine della prima giornata;
* **Luoghi visitati** — tutti i luoghi appartenenti al ramo geografico richiesto, deduplicati e ordinati prima geograficamente e poi per data della prima visita.

La Virtual Page è generica e può quindi essere richiamata anche per livelli inferiori a uno Stato.

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

.id-loading {
  padding: 8px 2px;
  color: var(--id-muted);
  font-size: 0.86em;
  font-style: italic;
}

.id-loading:empty {
  display: none;
}

.id-year-link {
  color: inherit;
  text-decoration: none;
  cursor: pointer;
}

.id-year-link:hover {
  text-decoration: underline;
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
  grid-template-columns: 50px minmax(0, 1fr);
  grid-template-areas:
    "cal title"
    "cal snippet"
    "cal tags"
    "cal luoghi"
    "cal gallery";
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
.id-snippet p {
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

.id-gallery {
  grid-area: gallery;
}

.id-meta-link {
  color: inherit;
  text-decoration: none;
  cursor: pointer;
}

.id-meta-link:hover {
  text-decoration: underline;
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
    grid-template-columns: 50px minmax(0, 1fr);
    grid-template-areas:
      "cal title"
      "snippet snippet"
      "tags tags"
      "luoghi luoghi"
      "gallery gallery";
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

  .id-meta-row {
    font-size: 0.8em;
  }
}
```
