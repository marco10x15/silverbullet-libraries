---
name: "Library/MG/Mio_Diario_Indice"
tags: meta/library
description: "Indice inline delle pagine Diario con data, titolo, immagine, snippet e filtro."
version: "0.1-09"
versionDate: 2026-08-27
pageDecoration.prefix: "📔 "
---

# 📔 IndiceDiario

**IndiceDiario** visualizza direttamente in una pagina SilverBullet un indice compatto delle pagine del Diario.

**Versione:** 0.1-09 — 27.08.2026

La libreria è autonoma e non dipende da Journal Explorer.

## Main Features

* **Indice inline** — inseribile con `${widgets.IndiceDiario()}`.
* **Data compatta** — mese, giorno e giorno della settimana nello stile calendario.
* **Titolo** — priorità `title` → primo H1 → `displayName` → nome pagina, visualizzato in grassetto e collegato alla pagina Diario.
* **Immagine** — prima immagine trovata nella pagina, se presente.
* **Snippet** — estratto dal testo della pagina.
* **Caricamento progressivo reale** — all'apertura interroga solo il primo batch; i batch successivi vengono richiesti durante lo scrolling e l'intero dataset viene caricato solo quando serve al filtro.
* **Filtro globale indicizzato** — ricerca con logica AND su path, titolo, `displayName`, `description`, tags, luoghi e Viaggio; lo snippet resta solo un elemento visuale.
* **Livello dati riutilizzabile** — raccolta pagine e filtro indicizzato sono separati dal rendering e pronti per le future Virtual Page.
* **Navigazione diretta** — i titoli usano `editor.open()` e restano raggiungibili anche dopo caricamento progressivo o filtro.
* **Raggruppamento mensile** — separazione delle pagine per mese e anno.
* **Virtual Page mensile** — il titolo del mese apre `diario:mese:YYYY-MM`.
* **Virtual Page di ricerca** — il filtro può essere aperto come `diario:ricerca:...`, ricalcolato esclusivamente sui dati indicizzati.
* **Testo normale** — titolo e snippet ereditano la normale dimensione del carattere della pagina.
* **Tags e luoghi** — visualizzati nelle ultime due righe della scheda con carattere ridotto; i luoghi sono navigabili.
* **Layout responsive** — su smartphone il calendario resta nell'header, mentre snippet, tags e luoghi sfruttano quasi tutta la larghezza della scheda.
* **Ricerca Luoghi** — `${widgets.CercaLuoghi()}` parte con elenco vuoto e mostra soltanto le pagine `luoghi/` che soddisfano la ricerca.

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

Per compatibilità con la precedente `0.1 alpha`, se `batchSize` non è definito viene ancora accettato `limit` come fallback. La nuova configurazione consigliata è però `batchSize`.

> **note** Lo snippet viene letto soltanto quando la relativa scheda viene renderizzata; non partecipa più al filtro.

> **note** `batchSize = 10` limita soltanto il numero di righe aggiunte a ogni caricamento; non limita il numero totale di pagine disponibili né il filtro.

## Uso

Inserire nella pagina indice:

```space-lua
${widgets.IndiceDiario()}
```

Le Virtual Page possono essere aperte anche direttamente:

```text
diario:mese:2026-08
diario:ricerca:torino vacanza
```

Nel widget, il titolo di ogni mese apre automaticamente la relativa Virtual Page mensile. Quando il filtro contiene testo, il pulsante **Apri risultati** apre la Virtual Page della ricerca corrente.

Per inserire il widget di ricerca dei luoghi:

```space-lua
${widgets.CercaLuoghi()}
```

`CercaLuoghi` non mostra alcuna pagina all'apertura. Al primo testo inserito carica soltanto i metadati indicizzati sotto `luoghi/`, costruisce una lookup gerarchica in memoria e quindi filtra senza leggere i file Markdown. Il percorso visualizzato contiene solo le pagine antenate del risultato.

## Implementazione

```space-lua
-- priority: 10

widgets = widgets or {}
indiceDiario = indiceDiario or {}
cercaLuoghi = cercaLuoghi or {}


-- ============================================================
-- CONFIGURAZIONE
-- ============================================================


-- Definisce un namespace di configurazione autonomo.
--
-- Nessun valore viene letto da Journal Explorer o da clientStore:
-- la configurazione è interamente contenuta in indiceDiario.
config.define("indiceDiario", {
  type = "object",
  properties = {
    journalPathPattern = schema.string(),
    batchSize          = schema.number(),
    showThumbnails     = schema.boolean(),
    showSnippets       = schema.boolean(),
    snippetStartMarker = schema.string(),
    monthNames = {
      type = "array",
      items = { type = "string" }
    },
    dayNames = {
      type = "array",
      items = { type = "string" }
    },
  }
})


-- Carica la configurazione applicando valori di default locali.
--
-- La funzione viene richiamata al rendering del widget, così le
-- modifiche alla CONFIG vengono applicate senza stato persistente
-- aggiuntivo.
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
      },
  }
end


-- ============================================================
-- SELEZIONE DELLE PAGINE
-- ============================================================


-- Converte journalPathPattern in un pattern Lua.
--
-- Mantiene i placeholder principali di Journal Explorer per
-- consentire di adattare la libreria a strutture Diario diverse
-- senza introdurre una seconda logica di configurazione.
local function indiceDiarioPattern(pattern)
  local p =
    pattern:gsub(
      "([%(%)%.%%%+%-%[%^%$%?%*])",
      "%%%1"
    )

  p = p:gsub("#weekdayfull#", "[%%a]+")
  p = p:gsub("#monthname#",   "[%%a]+")
  p = p:gsub("#monthshort#",  "[%%a]+")
  p = p:gsub("#weekday#",     "[%%a]+")
  p = p:gsub("#ordinal#",     "[%%a]+")
  p = p:gsub("#weekyear#",    "%%d%%d%%d%%d")
  p = p:gsub("#weeknum#",     "%%d%%d")
  p = p:gsub("#weeknumraw#",  "%%d+")
  p = p:gsub("#year#",        "%%d%%d%%d%%d")
  p = p:gsub("#month#",       "%%d%%d")
  p = p:gsub("#day#",         "%%d%%d")
  p = p:gsub("#YY#",          "%%d%%d")
  p = p:gsub("#M#",           "%%d+")
  p = p:gsub("#D#",           "%%d+")
  p = p:gsub("#HH#",          "%%d%%d")
  p = p:gsub("#hh#",          "%%d%%d")
  p = p:gsub("#mm#",          "%%d%%d")
  p = p:gsub("#ss#",          "%%d%%d")
  p = p:gsub("#wildcard#",    ".*")

  return "^" .. p .. "$"
end


-- Ricava una data dal path quando il frontmatter date non è
-- disponibile.
--
-- Sono supportate le forme YYYY-MM-DD e YYYY/MM/DD.
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


-- Calcola il nome del giorno della settimana.
--
-- Usa Zeller e restituisce un indice lunedì-domenica coerente
-- con la lista dayNames configurata.
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


-- Converte un oggetto pagina indicizzato in una voce Diario.
--
-- `firstH1` è opzionale. Nel caricamento rapido iniziale può essere
-- omesso per evitare una query globale sugli header.
local function indiceDiarioEntry(
  p,
  cfg,
  firstH1
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

  local title = nil

  if type(p.title) == "string"
    and p.title ~= ""
  then
    title = p.title

  elseif type(firstH1) == "string"
    and firstH1 ~= ""
  then
    title = firstH1

  elseif type(p.displayName) == "string"
    and p.displayName ~= ""
  then
    title = p.displayName

  else
    title =
      p.name:match(
        "([^/]+)$"
      )
      or p.name
  end

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


-- Recupera un singolo batch cronologico dal Diario.
--
-- È usato soltanto dal widget per il fast path iniziale e per lo
-- scrolling progressivo. Il cursore è la data ISO dell'ultima pagina
-- ricevuta; il Diario normalizzato usa una pagina per giornata.
--
-- Non interroga index.headers(): per il fast path il titolo usa
-- `title` -> `displayName` -> nome pagina. Il caricamento completo,
-- usato dalle Virtual Page e dal filtro, conserva invece il fallback H1.
function indiceDiario.entriesBatch(
  cfg,
  beforeDate
)
  cfg =
    cfg
    or indiceDiarioConfig()

  local batchSize =
    cfg.BATCH

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
        title = p.title,
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
        title = p.title,
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
        cfg,
        nil
      )

    if entry then
      table.insert(
        entries,
        entry
      )
    end
  end

  local nextCursor = nil

  if #pages > 0 then
    nextCursor =
      pages[#pages].date
  end

  return {
    entries = entries,
    cursor = nextCursor,
    hasMore =
      #pages >= batchSize
  }
end


-- Costruisce la lista delle pagine Diario usando gli indici
-- nativi SilverBullet.
--
-- index.subPages("Diario") limita semanticamente la collection alle
-- sole sottopagine del Diario. journalPathPattern stabilisce poi
-- quali sottopagine rappresentano effettivamente giornate valide.
--
-- index.headers() viene interrogato una sola volta per recuperare
-- il primo H1 senza query N+1.
--
-- Per il titolo la priorità è:
-- frontmatter title -> primo H1 -> displayName -> nome pagina.
--
-- La data del frontmatter ha priorità; il path è il fallback.
function indiceDiario.entries(cfg)
  cfg =
    cfg
    or indiceDiarioConfig()

  local pages = query[[
    from p = index.subPages("Diario")
    select {
      name = p.name,
      date = p.date,
      title = p.title,
      displayName = p.displayName,
      description = p.description,
      tags = p.tags,
      luoghi = p.luoghi,
      Viaggio = p.Viaggio
    }
  ]]

  local headers = query[[
    from h = index.headers()
    where h.level == 1
      and string.startsWith(
        h.page,
        "Diario/"
      )
    order by h.page, h.pos
    select {
      page = h.page,
      name = h.name
    }
  ]]

  local firstH1 = {}

  for _, h in ipairs(headers) do
    if type(h.page) == "string"
      and type(h.name) == "string"
      and h.name ~= ""
      and not firstH1[h.page]
    then
      firstH1[h.page] =
        h.name
    end
  end

  local entries = {}

  for _, p in ipairs(pages) do
    local entry =
      indiceDiarioEntry(
        p,
        cfg,
        firstH1[p.name]
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

      return a.sortKey
        > b.sortKey
    end
  )

  return entries
end

-- ============================================================
-- DATI E FILTRO RIUTILIZZABILI
-- ============================================================


-- Normalizza un attributo singolo o multivalore in una lista.
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


-- Converte un valore singolo o multivalore in testo ricercabile.
function indiceDiario.valueText(value)
  local parts = {}

  for _, item in ipairs(
    indiceDiario.list(value)
  ) do
    if type(item) == "string"
      and item ~= ""
    then
      table.insert(
        parts,
        item
      )
    end
  end

  return table.concat(
    parts,
    " "
  )
end


-- Scompone il filtro in termini separati.
--
-- I termini sono combinati con logica AND.
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
    queryText:gmatch(
      "%S+"
    )
  do
    table.insert(
      terms,
      term
    )
  end

  return terms
end


-- Verifica se tutti i termini sono presenti nel testo.
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


-- Costruisce il testo di ricerca utilizzando soltanto attributi
-- già presenti nell'indice SilverBullet.
--
-- Questo testo è condiviso dal widget e dalle future Virtual Page.
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


-- Filtro generico.
--
-- Se extraTextFn è nil usa esclusivamente i dati indicizzati.
-- Se extraTextFn è presente viene interrogato soltanto per le
-- pagine che non soddisfano già interamente il filtro indicizzato.
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

      searchText =
        searchText
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


-- Filtro esclusivamente indicizzato.
--
-- È il punto di riuso previsto per le future Virtual Page:
-- nessuna lettura del Markdown e nessuna dipendenza dagli snippet.
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


-- Restituisce le pagine di un singolo mese.
--
-- La lista mantiene l'ordinamento già prodotto da entries():
-- più recente -> più vecchia.
function indiceDiario.month(
  entries,
  year,
  month
)
  local y = tonumber(year)
  local m = tonumber(month)

  if not y
    or not m
    or m < 1
    or m > 12
  then
    return {}
  end

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


-- Rende una lista di pagine Diario come Markdown.
--
-- È volutamente più semplice delle card DOM del widget:
-- una Virtual Page può contenere molti risultati e deve restare
-- leggera. Usa esclusivamente dati già presenti nell'indice.
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


-- Estrae target e label da un wikilink SilverBullet.
--
-- Restituisce nil per valori non riconosciuti.
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


-- ============================================================
-- CERCA LUOGHI — DATI
-- ============================================================


-- Restituisce il primo alias testuale disponibile.
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


-- Restituisce il nome visuale preferito di una pagina luogo.
--
-- Priorità:
-- title -> displayName -> aliases -> ultimo segmento del path.
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


-- Costruisce il breadcrumb visuale usando le pagine antenate.
--
-- L'ultimo segmento non viene incluso perché è già il titolo
-- del risultato corrente.
--
-- Esempio:
-- luoghi/ITA/21/Torino/Superga
-- -> Italia › Piemonte › Torino
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
    path =
      path
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


-- Costruisce il dataset indicizzato delle pagine sotto luoghi/.
--
-- La query viene eseguita una sola volta al primo utilizzo del
-- filtro. Il lookup dei nomi gerarchici viene poi costruito in
-- memoria sulla stessa collection, senza ulteriori query e senza
-- leggere il Markdown.
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

  -- Primo passaggio:
  -- lookup path canonico -> nome visuale.
  local labelsByPath = {}

  for _, p in ipairs(pages) do
    if type(p.name) == "string"
      and p.name ~= ""
    then
      labelsByPath[p.name] =
        cercaLuoghiLabel(p)
    end
  end

  -- Secondo passaggio:
  -- costruzione dei risultati e dei breadcrumb tradotti.
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

      -- La ricerca comprende sia il path canonico sia i nomi
      -- visuali ricavati esclusivamente da attributi indicizzati.
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


-- Ricerca AND sui soli dati indicizzati dei luoghi.
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
-- VIRTUAL PAGE
-- ============================================================


-- Tutte le pagine Diario di un mese.
--
-- Esempio:
-- diario:mese:2026-08
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


-- Ricerca riproducibile sulle sole proprietà indicizzate.
--
-- Esempio:
-- diario:ricerca:torino vacanza
--
-- Non legge il Markdown e non utilizza lo snippet.
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


-- Estrae snippet e prima immagine dal Markdown.
--
-- Viene eseguita soltanto sulle pagine realmente renderizzate.
-- Lo snippet non partecipa al filtro: la ricerca usa esclusivamente
-- gli attributi già presenti nell'indice SilverBullet.
--
-- Se snippetStartMarker è configurato, lo snippet comincia dopo
-- quella stringa; in caso contrario salta il primo testo utile,
-- normalmente il titolo H1.
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

  -- Le immagini vengono individuate prima della pulizia dello snippet.
  local wikiImage =
    body:match(
      "!%[%[([^%]|]+)"
    )

  local markdownImage =
    nil

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


  -- Sostituisce nel testo dello snippet immagini e link con "...".
  --
  -- Sono riconosciuti:
  -- ![[media/...]]
  -- ![testo](url)
  -- [[link SilverBullet]]
  -- [[link SilverBullet|alias]]
  -- [testo](https://...)
  --
  -- Le immagini vengono trattate prima dei link, per evitare
  -- che resti il carattere "!" davanti al placeholder.
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

    -- Normalizza spazi e placeholder consecutivi.
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
          cleanSnippetText(
            value
          )

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
    cleanSnippetText(
      snippet
    )

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


-- Costruisce la miniatura della pagina.
--
-- Per gli embed SilverBullet viene riutilizzato il wikilink
-- originale; per immagini Markdown viene generato il relativo
-- markup standard.
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


-- Costruisce il widget completo dell'indice.
--
-- Il widget delega raccolta e filtro al namespace indiceDiario.
-- Il rendering DOM resta separato dal livello dati.
--
-- Titolo e snippet non impostano una dimensione font propria e
-- quindi ereditano il normale carattere della pagina SilverBullet.
function widgets.IndiceDiario()
  local cfg =
    indiceDiarioConfig()

  -- Fast path:
  -- all'apertura vengono richieste soltanto le prime cfg.BATCH pagine.
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

  -- Cache del contenuto valida per questa istanza del widget.
  --
  -- Il Markdown viene letto solo quando la relativa card viene
  -- effettivamente costruita.
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
        indiceDiarioWikiLink(
          value
        )

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
      row.appendChild(
        tagsNode
      )
    end

    local luoghiNode =
      buildLuoghiNode(entry)

    if luoghiNode then
      row.appendChild(
        luoghiNode
      )
    end

    if cfg.THUMBNAILS then
      local thumb =
        indiceDiarioThumbnail(
          info.wikiImage,
          info.markdownImage
        )

      if thumb then
        row.appendChild(
          thumb
        )
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

  -- Modalità cronologica progressiva.
  local streamEntries =
    firstBatch.entries

  local streamCursor =
    firstBatch.cursor

  local streamHasMore =
    firstBatch.hasMore

  -- Dataset completo: viene popolato soltanto quando serve al filtro.
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

    if monthKey
      == lastMonthKey
    then
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

    lastMonthKey =
      monthKey
  end


  -- Renderizza fino a cfg.BATCH elementi già presenti nel dataset attivo.
  local function renderNextBatch()
    if renderedCount
      >= #activeEntries
    then
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

    renderedCount =
      last
  end


  -- Aggiunge al flusso cronologico un nuovo batch richiesto all'indice.
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

    streamCursor =
      batch.cursor

    streamHasMore =
      batch.hasMore
      and #batch.entries > 0

    activeEntries =
      streamEntries

    renderNextBatch()
  end


  local function resetList(entries)
    activeEntries =
      entries

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


  -- Il dataset completo viene caricato solo al primo uso del filtro.
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

      if renderedCount
        < #activeEntries
      then
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
        streamEntries =
          allEntries

        streamCursor = nil
        streamHasMore = false

        resetList(
          allEntries
        )
      else
        resetList(
          streamEntries
        )
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


  -- Le prime schede sono già disponibili dal fast path.
  renderNextBatch()

  return widget.htmlBlock(root)
end


-- ============================================================
-- WIDGET CERCA LUOGHI
-- ============================================================


-- Ricerca interattiva delle pagine sotto luoghi/.
--
-- Comportamento:
-- - elenco vuoto all'apertura;
-- - caricamento del dataset solo al primo filtro non vuoto;
-- - ricerca AND su path, title, displayName e aliases;
-- - nessuna lettura dei file Markdown;
-- - rendering progressivo dei risultati.
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


  -- Il dataset viene interrogato una sola volta per istanza del widget
  -- e solo dopo l'inserimento del primo filtro.
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
    if renderedCount
      >= #activeEntries
    then
      return
    end

    local last =
      math.min(
        renderedCount
          + batchSize,
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

    renderedCount =
      last
  end


  local function resetResults(entries)
    activeEntries =
      entries

    renderedCount = 0

    list.replaceChildren()
    list.scrollTop = 0

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
    local searchText =
      tostring(
        value
        or ""
      ):match(
        "^%s*(.-)%s*$"
      )

    if not searchText
      or searchText == ""
    then
      clearResults()
      return
    end

    local entries =
      ensureEntries()

    local filtered =
      cercaLuoghi.filter(
        entries,
        searchText
      )

    resetResults(
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

      if value:match(
        "^%s*$"
      ) then
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

## Note della revisione 0.1-09

Migliorata la presentazione gerarchica di `widgets.CercaLuoghi()`.

- la stessa query `index.subPages("luoghi")` costruisce una lookup locale `path -> label`;
- nessuna query aggiuntiva e nessun `space.readPage()`;
- il lookup viene calcolato una sola volta per istanza del widget, al primo utilizzo effettivo della ricerca;
- i codici dei livelli geografici vengono visualizzati usando `title` → `displayName` → `aliases` → nome pagina;
- il breadcrumb mostra soltanto gli antenati, escludendo il risultato corrente.

Esempio:

```text
luoghi/ITA/21/Torino/Superga
```

viene visualizzato come risultato:

```text
Superga
Italia › Piemonte › Torino
```

I nomi gerarchici tradotti vengono inclusi anche nella stringa di ricerca, quindi una ricerca come `Piemonte Superga` può trovare la pagina senza modificare il path canonico.

## Note della revisione 0.1-08

Aggiunto `widgets.CercaLuoghi()`.

- all'apertura il widget mostra soltanto il campo di ricerca;
- nessuna query viene eseguita finché il filtro è vuoto;
- al primo utilizzo viene caricato una sola volta il dataset di `index.subPages("luoghi")`;
- la ricerca usa esclusivamente `name`, `title`, `displayName` e `aliases`;
- il path gerarchico viene mostrato come breadcrumb testuale;
- i risultati vengono renderizzati 25 alla volta durante lo scrolling;
- il click apre direttamente la pagina luogo;
- nessun `space.readPage()` e nessuna scansione del Markdown.

La logica AND del filtro riusa `indiceDiario.filterIndexed()` senza modificare il comportamento già stabile di `IndiceDiario`.

## Note della revisione 0.1-07

Ottimizzato il caricamento iniziale del widget.

- all'apertura `widgets.IndiceDiario()` usa `indiceDiario.entriesBatch()` e interroga soltanto le prime `batchSize` pagine;
- i batch successivi vengono richiesti a `index.subPages("Diario")` durante lo scrolling;
- il dataset completo viene caricato soltanto al primo utilizzo effettivo del filtro;
- le Virtual Page continuano a usare `indiceDiario.entries()` e rimangono autonome;
- il Markdown viene letto soltanto quando una card viene realmente renderizzata;
- il fast path iniziale non esegue la query globale su `index.headers()`.

Nel fast path il titolo usa `title` → `displayName` → nome pagina. Il fallback al primo H1 resta disponibile nel caricamento completo usato dal filtro e dalle Virtual Page.

La paginazione cronologica usa `p.date` come cursore ISO; questo è coerente con il Diario normalizzato, dove `date` è presente nel frontmatter delle giornate.

## Note della revisione 0.1-06

Aggiunte due Virtual Page basate sul livello dati introdotto nella 0.1-04:

- `diario:mese:YYYY-MM` mostra tutte le giornate del mese;
- `diario:ricerca:...` ricalcola la ricerca usando esclusivamente gli attributi indicizzati.

Il filtro del widget è stato allineato alla Virtual Page di ricerca: non usa più lo snippet. Lo snippet rimane visualizzato nelle card e viene letto solo quando la scheda viene effettivamente renderizzata.

Nel widget:
- il titolo del mese apre la relativa Virtual Page mensile;
- il pulsante **Apri risultati** apre la ricerca indicizzata corrente.

Le Virtual Page restituiscono Markdown leggero e non duplicano il rendering DOM delle card.

## Note della revisione 0.1-05

Corretto `indiceDiario.indexedSearchText()`: la stringa di ricerca non viene più costruita con una concatenazione `..` di attributi indicizzati potenzialmente assenti. Ogni valore passa ora da `indiceDiario.valueText()` e il risultato viene composto con `table.concat()`.

La modifica non cambia i campi ricercati né il comportamento del filtro; rende soltanto più robusta la normalizzazione dei valori `nil`.

## Note della revisione 0.1-04

Questa revisione separa il livello **dati/filtro** dal rendering del widget.

- `indiceDiario.entries()` raccoglie e normalizza le pagine del Diario.
- `indiceDiario.filterIndexed()` filtra esclusivamente gli attributi già indicizzati.
- `indiceDiario.filter()` consente al widget di aggiungere lo snippet come sorgente supplementare senza duplicare la logica AND.
- la sorgente delle pagine è ora `index.subPages("Diario")`.
- il filtro indicizzato comprende `path`, titolo, `displayName`, `description`, `tags`, `luoghi` e `Viaggio`.

`indiceDiario.filterIndexed()` non esegue `space.readPage()` ed è il punto di riuso previsto per le future Virtual Page.

Il cambio da `index.pages() + startsWith` a `index.subPages("Diario")` non dovrebbe produrre oggi un'accelerazione significativa: nell'implementazione corrente SilverBullet `subPages()` applica internamente un filtro sul prefisso della collection `page`. Il vantaggio principale è semantico, manutentivo e di compatibilità con eventuali future ottimizzazioni dell'API.

## Note della revisione 0.1-03

Il rendering delle card usa ora CSS Grid. La stessa struttura DOM viene riordinata via CSS sotto i 520 px: non vengono duplicate schede, query o letture Markdown.

Su smartphone il calendario resta accanto al titolo; snippet e foto passano nella riga successiva, mentre tags e luoghi occupano tutta la larghezza. Lo snippet mobile è limitato visivamente a tre righe.

L'impatto computazionale rispetto alla 0.1-02 è trascurabile: cambia principalmente il layout CSS, non la pipeline dati.

## Note prestazionali della revisione 0.1-02

La query iniziale legge dall'indice anche `tags` e `luoghi`. Questo non introduce letture aggiuntive dei file Markdown.

Il filtro utilizza prima `path + title + tags + luoghi`; `space.readPage()` viene chiamato soltanto quando serve verificare lo snippet. Il campo di ricerca applica inoltre un debounce di circa 280 ms.

Le schede continuano a essere create in batch da `batchSize`, quindi tags e luoghi aggiungono soltanto pochi nodi DOM alle pagine effettivamente visualizzate.

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

.id-filter {
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

.id-filter:focus {
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

.id-list::-webkit-scrollbar {
  width: 5px;
}

.id-list::-webkit-scrollbar-track {
  background: transparent;
}

.id-list::-webkit-scrollbar-thumb {
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

/*
  Smartphone:
  - calendario e titolo formano l'header;
  - lo snippet passa sotto e usa quasi tutta la larghezza;
  - la foto resta a destra dello snippet;
  - tags e luoghi occupano l'intera larghezza della card.
*/
.cl-index {
  display: flex;
  flex-direction: column;
  gap: 7px;
  font-size: inherit;
}

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

.cl-filter:focus {
  border-color: var(--id-accent);
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

.cl-list::-webkit-scrollbar {
  width: 5px;
}

.cl-list::-webkit-scrollbar-track {
  background: transparent;
}

.cl-list::-webkit-scrollbar-thumb {
  background: var(--id-border);
  border-radius: 3px;
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
