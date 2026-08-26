---
name: "Library/MG/Mio_Diario_Indice"
tags: meta/library
description: "Indice inline delle pagine Diario con data, titolo, immagine, snippet e filtro."
version: "0.1-03"
versionDate: 2026-08-26
pageDecoration.prefix: "📔 "
---

# 📔 IndiceDiario

**IndiceDiario** visualizza direttamente in una pagina SilverBullet un indice compatto delle pagine del Diario.

**Versione:** 0.1-03 — 26.08.2026

La libreria è autonoma e non dipende da Journal Explorer.

## Main Features

* **Indice inline** — inseribile con `${widgets.IndiceDiario()}`.
* **Data compatta** — mese, giorno e giorno della settimana nello stile calendario.
* **Titolo** — titolo H1 della pagina, visualizzato in grassetto e collegato alla pagina Diario.
* **Immagine** — prima immagine trovata nella pagina, se presente.
* **Snippet** — estratto dal testo della pagina.
* **Caricamento progressivo** — visualizza 10 pagine per volta e carica automaticamente le successive durante lo scrolling.
* **Filtro globale ottimizzato** — ricerca con logica AND su path, titolo, tags, luoghi e snippet; usa prima i metadati indicizzati e legge il Markdown solo quando necessario.
* **Navigazione diretta** — i titoli usano `editor.open()` e restano raggiungibili anche dopo caricamento progressivo o filtro.
* **Raggruppamento mensile** — separazione delle pagine per mese e anno.
* **Testo normale** — titolo e snippet ereditano la normale dimensione del carattere della pagina.
* **Tags e luoghi** — visualizzati nelle ultime due righe della scheda con carattere ridotto; i luoghi sono navigabili.
* **Layout responsive** — su smartphone il calendario resta nell'header, mentre snippet, tags e luoghi sfruttano quasi tutta la larghezza della scheda.

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

`batchSize` determina quante pagine vengono aggiunte a ogni blocco di caricamento. Il filtro opera invece sull'intero Diario.

Il filtro è applicato con un debounce di circa 280 ms per evitare una scansione completa a ogni singolo tasto premuto.

Per compatibilità con la precedente `0.1 alpha`, se `batchSize` non è definito viene ancora accettato `limit` come fallback. La nuova configurazione consigliata è però `batchSize`.

> **note** Il primo filtro che richiede la ricerca negli snippet può dover leggere pagine non ancora visualizzate. Le informazioni vengono poi mantenute in cache per l'intera vita del widget.

> **note** `batchSize = 10` limita soltanto il numero di righe aggiunte a ogni caricamento; non limita il numero totale di pagine disponibili né il filtro.

## Uso

Inserire nella pagina indice:

```space-lua
${widgets.IndiceDiario()}
```

## Implementazione

```space-lua
-- priority: 10

widgets = widgets or {}
indiceDiario = indiceDiario or {}


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


-- Ricava la parte iniziale statica del path pattern.
--
-- Esempio:
-- Diario/#year#-#month#-#day#
-- -> Diario/
--
-- Il prefisso permette di limitare le query native all'area
-- pertinente dello Space prima del controllo del pattern completo.
local function indiceDiarioPrefix(pattern)
  return pattern:match("^([^#]*)")
    or ""
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


-- Costruisce la lista delle pagine candidate usando gli indici
-- nativi SilverBullet.
--
-- index.pages() fornisce path e frontmatter.
-- index.headers() viene interrogato una sola volta per recuperare
-- il primo H1 di ciascuna pagina senza query N+1.
--
-- Per il titolo la priorità è:
-- frontmatter title -> primo H1 -> displayName -> nome pagina.
--
-- La data del frontmatter ha priorità; il path è il fallback.
local function indiceDiarioEntries(cfg)
  local pattern =
    indiceDiarioPattern(
      cfg.PATTERN
    )

  local prefix =
    indiceDiarioPrefix(
      cfg.PATTERN
    )

  local pages = query[[
    from p = index.pages()
    where prefix == ""
      or string.startsWith(
        p.name,
        prefix
      )
    select {
      name = p.name,
      date = p.date,
      title = p.title,
      displayName = p.displayName,
      tags = p.tags,
      luoghi = p.luoghi
    }
  ]]

  local headers = query[[
    from h = index.headers()
    where h.level == 1
      and (
        prefix == ""
        or string.startsWith(
          h.page,
          prefix
        )
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
    if type(p.name) == "string"
      and p.name:match(pattern)
    then
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

      if y and m and d
        and dateValue
      then
        local title = nil

        -- Per il Diario il frontmatter title, quando presente,
        -- ha priorità sul titolo H1 della pagina.
        if type(p.title) == "string"
          and p.title ~= ""
        then
          title = p.title

        elseif type(firstH1[p.name]) == "string"
          and firstH1[p.name] ~= ""
        then
          title = firstH1[p.name]

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

        table.insert(
          entries,
          {
            path = p.name,
            date = dateValue,
            year = y,
            month = m,
            day = d,
            title = title,
            tags = p.tags,
            luoghi = p.luoghi,
            sortKey =
              y * 10000
              + m * 100
              + d
          }
        )
      end
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
-- METADATI INDICIZZATI
-- ============================================================


-- Normalizza un attributo singolo o multivalore in una lista.
local function indiceDiarioList(value)
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


-- Converte una lista di valori in testo ricercabile.
--
-- Viene usato dal filtro prima di leggere il Markdown completo.
local function indiceDiarioSearchText(value)
  local parts = {}

  for _, item in ipairs(
    indiceDiarioList(value)
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
-- CONTENUTO DELLE PAGINE
-- ============================================================


-- Estrae snippet e prima immagine dal Markdown.
--
-- Durante la visualizzazione viene eseguita soltanto sulle pagine
-- realmente renderizzate. Quando viene usato il filtro, le pagine
-- non ancora lette vengono analizzate una sola volta e poi mantenute
-- in cache per completare la ricerca sull'intero Diario.
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
-- Il filtro viene applicato direttamente agli elementi DOM già
-- caricati e usa logica AND: tutte le parole inserite devono
-- comparire in titolo, snippet o path.
--
-- Titolo e snippet non impostano una dimensione font propria e
-- quindi ereditano il normale carattere della pagina SilverBullet.
function widgets.IndiceDiario()
  local cfg =
    indiceDiarioConfig()

  local allEntries =
    indiceDiarioEntries(cfg)

  if not allEntries
    or #allEntries == 0
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
  -- Ogni pagina viene letta al massimo una volta. Lo scrolling
  -- carica soltanto le schede richieste; il filtro usa prima i
  -- metadati indicizzati e ricorre allo snippet solo se necessario.
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


  -- Costruisce la riga compatta dei tags.
  local function buildTagsNode(entry)
    local tags =
      indiceDiarioList(
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


  -- Costruisce la riga dei luoghi.
  --
  -- I wikilink vengono trasformati in elementi DOM con editor.open()
  -- così rimangono navigabili anche nei batch creati dinamicamente.
  local function buildLuoghiNode(entry)
    local valori =
      indiceDiarioList(
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


  -- Costruisce una singola scheda dell'indice.
  --
  -- Gli elementi principali sono fratelli diretti della card.
  -- Questa struttura consente al CSS Grid di disporli diversamente
  -- su desktop e smartphone senza duplicare il DOM.
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

  -- Prepara una stringa di ricerca usando esclusivamente dati già
  -- presenti nell'indice.
  --
  -- Questo passaggio è economico e permette a molte ricerche per
  -- titolo, tag o luogo di evitare space.readPage().
  for _, entry in ipairs(allEntries) do
    entry.indexedSearchText =
      string.lower(
        entry.path
        .. " "
        .. entry.title
        .. " "
        .. indiceDiarioSearchText(
          entry.tags
        )
        .. " "
        .. indiceDiarioSearchText(
          entry.luoghi
        )
      )
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
        "Filtra "
        .. #allEntries
        .. " pagine…"
    }

  local list =
    dom.div {
      class = "id-list"
    }

  root.appendChild(filter)
  root.appendChild(list)

  local activeEntries =
    allEntries

  local renderedCount = 0
  local lastMonthKey = nil


  -- Aggiunge il batch successivo senza ricostruire il DOM esistente.
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

      local monthKey =
        tostring(entry.year)
        .. "-"
        .. string.format(
          "%02d",
          entry.month
        )

      if monthKey
        ~= lastMonthKey
      then
        local monthName =
          cfg.MONTHS[entry.month]
          or string.format(
            "%02d",
            entry.month
          )

        list.appendChild(
          dom.div {
            class = "id-month",
            __rawText =
              monthName
              .. " "
              .. tostring(
                entry.year
              )
          }
        )

        lastMonthKey =
          monthKey
      end

      list.appendChild(
        buildEntryNode(entry)
      )
    end

    renderedCount =
      last
  end


  -- Ricostruisce soltanto la parte visibile dell'elenco.
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


  -- Caricamento progressivo dei batch successivi.
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


  -- Esegue il filtro globale.
  --
  -- Prima prova esclusivamente path, titolo, tags e luoghi.
  -- Solo quando almeno un termine non è presente nei metadati
  -- indicizzati viene letto lo snippet della pagina.
  local function applyFilter(value)
    local queryText =
      string.lower(
        tostring(
          value
          or ""
        )
      )

    local terms = {}

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

    if #terms == 0 then
      resetList(
        allEntries
      )

      return
    end

    local filtered = {}

    for _, entry in ipairs(
      allEntries
    )
    do
      local indexedText =
        entry.indexedSearchText

      local metadataMatch =
        true

      for _, term in ipairs(
        terms
      )
      do
        if not string.find(
          indexedText,
          term,
          1,
          true
        ) then
          metadataMatch =
            false
          break
        end
      end

      if metadataMatch then
        table.insert(
          filtered,
          entry
        )

      else
        local info =
          ensureContent(entry)

        local fullText =
          indexedText
          .. " "
          .. string.lower(
            info.snippet
            or ""
          )

        local matches =
          true

        for _, term in ipairs(
          terms
        )
        do
          if not string.find(
            fullText,
            term,
            1,
            true
          ) then
            matches = false
            break
          end
        end

        if matches then
          table.insert(
            filtered,
            entry
          )
        end
      end
    end

    resetList(
      filtered
    )
  end


  -- Debounce del campo di ricerca.
  --
  -- Evita di scandire fino a centinaia di pagine per ogni singola
  -- battuta quando l'utente sta ancora componendo il filtro.
  local filterTimer = nil

  filter.addEventListener(
    "input",
    function(e)
      local value =
        tostring(
          e.target.value
          or ""
        )

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

  renderNextBatch()

  return widget.htmlBlock(root)
end
```

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

.id-filter {
  width: 100%;
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
@media (max-width: 520px) {
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
