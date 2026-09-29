---
name: "Library/MG/Mio_Viaggio"
tags: meta/library
description: "Libreria autonoma per pianificare e gestire Viaggi, Giorni, Attività, Luoghi, ricerche e mappe in SilverBullet 2.11+."
version: "0.4-00"
versionDate: 2026-09-29
pageDecoration.prefix: "🧳 "
share.uri: "github:marco10x15/silverbullet-libraries/Mio_Viaggio.md"
files:
  - Mio_Viaggio/Page Templates/Viaggio.md
  - Mio_Viaggio/Page Templates/Giorno.md
  - Mio_Viaggio/Page Templates/Luogo.md
---

# 🧳 Mio Viaggio

**Versione:** 0.4-00 — 29.09.2026

Libreria autonoma per SilverBullet 2.11+.

Non dipende da `Mio_Diario`, `PhotoGallery` o servizi fotografici esterni.
Mantiene il modello:

```text
Viaggi/<Nome viaggio>
├── Giorno 01
├── Giorno 02
├── Ricerche
├── Appunti Evidenziati
└── Escursioni/
```

Il modello logico è:

```text
Viaggio → Giorno → Attività
```

Le escursioni complesse possono essere organizzate sotto:

```text
Viaggi/<Nome>/Escursioni/<Nome>
```

## Page Templates distribuiti

La Library installa insieme al codice:

- `Library/MG/Mio_Viaggio/Page Templates/Viaggio`
- `Library/MG/Mio_Viaggio/Page Templates/Giorno`
- `Library/MG/Mio_Viaggio/Page Templates/Luogo`

## Configurazione Action Button

Per avere il pulsante **Viaggio** nella barra:

```lua

```space-lua
actionButton.define {
  icon = "map",
  description = "Viaggio",
  command = "Viaggio: Menu",
  priority = 2.5,
  dropdown = false,
}
```

## Implementazione

```space-lua
-- priority: 10

viaggio = viaggio or {}

local function trim(value)
  if value == nil then
    return ""
  end

  value = tostring(value)
  value = string.gsub(value, "^%s+", "")
  value = string.gsub(value, "%s+$", "")
  return value
end

local function basename(path)
  return string.match(path or "", "([^/]+)$") or path
end

local function parentPath(path)
  return string.match(path or "", "^(.*)/[^/]+$")
end

local function escapeYaml(value)
  value = tostring(value or "")
  value = string.gsub(value, "\\", "\\\\")
  value = string.gsub(value, '"', '\\"')
  return '"' .. value .. '"'
end

local function urlEncode(value)
  return string.gsub(
    tostring(value or ""),
    "([^%w%-_%.~])",
    function(c)
      return string.format("%%%02X", string.byte(c))
    end
  )
end

local function hasTag(meta, wanted)
  if not meta then
    return false
  end

  local tags = meta.tags or {}
  if type(tags) == "string" then
    return tags == wanted
  end

  for _, tag in ipairs(tags) do
    if tag == wanted then
      return true
    end
  end

  return false
end

local function notify(message)
  editor.flashNotification(message)
end

local function readMeta(pageName)
  local ok, meta = pcall(space.getPageMeta, pageName)
  if ok then
    return meta
  end
  return nil
end

function viaggio.currentTripPage(pageName)
  local current = pageName or editor.getCurrentPage()

  while current and current ~= "" do
    local meta = readMeta(current)
    if hasTag(meta, "viaggio") then
      return current, meta
    end
    current = parentPath(current)
  end

  return nil, nil
end

function viaggio.tripRoot(pageName)
  local root = viaggio.currentTripPage(pageName)
  return root
end

function viaggio.nextDayNumber(tripPage)
  local maxDay = 0

  for _, page in ipairs(index.subPages(tripPage) or {}) do
    local name = basename(page.name)
    local n = tonumber(string.match(name or "", "^Giorno%s+(%d+)$"))
    if n and n > maxDay then
      maxDay = n
    end
  end

  return maxDay + 1
end

local function twoDigits(n)
  if n < 10 then
    return "0" .. tostring(n)
  end
  return tostring(n)
end

function viaggio.createDay(tripPage, number)
  local tripMeta = readMeta(tripPage) or {}
  local dayNumber = number or viaggio.nextDayNumber(tripPage)
  local dayLabel = "Giorno " .. twoDigits(dayNumber)
  local pageName = tripPage .. "/" .. dayLabel

  if space.pageExists(pageName) then
    editor.navigate(pageName)
    return pageName
  end

  local tripDisplay =
    trim(tripMeta.displayName) ~= ""
    and tripMeta.displayName
    or basename(tripPage)

  local text =
    "---\n"
    .. "displayName: " .. escapeYaml(dayLabel) .. "\n"
    .. "viaggio: " .. escapeYaml("[[" .. tripPage .. "]]") .. "\n"
    .. "giorno: " .. tostring(dayNumber) .. "\n"
    .. "luoghi: []\n"
    .. "tags:\n"
    .. "  - giorno\n"
    .. "pageDecoration.icon: calendar\n"
    .. "tree.priority: 800\n"
    .. "---\n\n"
    .. "# " .. dayLabel .. "\n"

  space.writePage(pageName, text)
  return pageName
end

function viaggio.createReadeckPages(tripPage)
  local ricerche = tripPage .. "/Ricerche"
  local appunti = tripPage .. "/Appunti Evidenziati"

  if not space.pageExists(ricerche) then
    space.writePage(
      ricerche,
      "---\n"
      .. 'displayName: "Ricerche da readeck"\n'
      .. "pageDecoration.icon: inbox\n"
      .. "tree.priority: 220\n"
      .. "---\n\n"
      .. "# Ricerche da Readeck\n\n"
      .. "${readeck.catalogView()}\n"
    )
  end

  if not space.pageExists(appunti) then
    space.writePage(
      appunti,
      "---\n"
      .. 'displayName: "Appunti Evidenziati"\n'
      .. "pageDecoration.icon: clipboard\n"
      .. "tree.priority: 210\n"
      .. "---\n\n"
      .. "# Appunti Evidenziati\n\n"
      .. "${readeck.annotatedView()}\n"
    )
  end
end

function viaggio.newTrip()
  local name = trim(editor.prompt("Nome del viaggio"))

  if name == "" then
    return
  end

  local pageName = "Viaggi/" .. name

  if space.pageExists(pageName) then
    notify("Viaggio già esistente: " .. name)
    editor.navigate(pageName)
    return
  end

  local text =
    "---\n"
    .. "displayName: " .. escapeYaml(name) .. "\n"
    .. "description: " .. escapeYaml(name) .. "\n"
    .. "tags:\n"
    .. "  - viaggio\n"
    .. "---\n\n"
    .. "# " .. name .. "\n"

  space.writePage(pageName, text)
  viaggio.createDay(pageName, 1)
  viaggio.createReadeckPages(pageName)

  editor.navigate(pageName)
  notify("Viaggio creato: " .. name)
end

function viaggio.newDay()
  local tripPage = viaggio.currentTripPage()

  if not tripPage then
    notify("Nessuna pagina Viaggio trovata nella pagina corrente o nei parent")
    return
  end

  local dayPage = viaggio.createDay(tripPage)
  editor.navigate(dayPage)
end

function viaggio.newChild()
  local current = editor.getCurrentPage()
  local name = trim(editor.prompt("Nome della nuova pagina figlia"))

  if name == "" then
    return
  end

  local pageName = current .. "/" .. name

  if space.pageExists(pageName) then
    editor.navigate(pageName)
    return
  end

  space.writePage(
    pageName,
    "---\n"
    .. "displayName: " .. escapeYaml(name) .. "\n"
    .. "---\n\n"
    .. "# " .. name .. "\n"
  )

  editor.navigate(pageName)
end

function viaggio.wikidataSearch(input, language)
  language = language or "it"

  local url =
    "https://www.wikidata.org/w/api.php"
    .. "?action=wbsearchentities"
    .. "&search=" .. urlEncode(input)
    .. "&language=" .. urlEncode(language)
    .. "&uselang=it"
    .. "&type=item"
    .. "&limit=8"
    .. "&format=json"
    .. "&formatversion=2"

  local response = net.proxyFetch(url, {
    headers = { Accept = "application/json" },
  })

  if not response or not response.ok then
    error(
      "Wikidata HTTP "
      .. tostring(response and response.status or "errore")
    )
  end

  return response.body and response.body.search or {}
end

function viaggio.wikidataEntity(qid)
  local url =
    "https://www.wikidata.org/wiki/Special:EntityData/"
    .. urlEncode(qid)
    .. ".json"

  local response = net.proxyFetch(url, {
    headers = { Accept = "application/json" },
  })

  if not response or not response.ok then
    error(
      "Wikidata HTTP "
      .. tostring(response and response.status or "errore")
    )
  end

  local entities = response.body and response.body.entities or {}
  return entities[qid]
end

local function claimValue(entity, property)
  local claims = entity and entity.claims and entity.claims[property]
  if not claims or not claims[1] then
    return nil
  end

  local snak = claims[1].mainsnak
  return snak
    and snak.datavalue
    and snak.datavalue.value
    or nil
end

function viaggio.wikidataPlace(qid)
  local entity = viaggio.wikidataEntity(qid)
  if not entity then
    return nil
  end

  local labels = entity.labels or {}
  local label =
    labels.it and labels.it.value
    or labels.en and labels.en.value
    or qid

  local descriptions = entity.descriptions or {}
  local description =
    descriptions.it and descriptions.it.value
    or descriptions.en and descriptions.en.value
    or ""

  local coordinates = claimValue(entity, "P625")
  local article = nil

  if entity.sitelinks then
    if entity.sitelinks.itwiki then
      article = "https://it.wikipedia.org/wiki/"
        .. string.gsub(entity.sitelinks.itwiki.title, " ", "_")
    elseif entity.sitelinks.enwiki then
      article = "https://en.wikipedia.org/wiki/"
        .. string.gsub(entity.sitelinks.enwiki.title, " ", "_")
    end
  end

  return {
    qid = qid,
    displayName = label,
    description = description,
    wikipedia = article,
    coordinate = coordinates and {
      coordinates.latitude,
      coordinates.longitude,
    } or nil,
  }
end

function viaggio.pickWikidataPlace(input)
  local results = viaggio.wikidataSearch(input, "it")

  if #results == 0 then
    results = viaggio.wikidataSearch(input, "en")
  end

  if #results == 0 then
    return nil
  end

  return view.pick {
    title = "Wikidata",
    filter = false,
    source = function()
      return results
    end,
    presentation = {
      mode = "list",
      row = {
        primary = function(item)
          return item.label or item.id
        end,
        description = function(item)
          local d = trim(item.description)
          if d == "" then
            d = item.id or ""
          end
          return { text = d }
        end,
      },
    },
  }
end

function viaggio.newPlace()
  local mode = view.pick {
    title = "Nuovo luogo",
    filter = false,
    source = function()
      return {
        {
          name = "Cerca con Wikidata",
          description = "Cerca e precompila i dati del luogo",
          value = "wikidata",
        },
        {
          name = "Senza ricerca",
          description = "Crea una pagina Luogo vuota",
          value = "manual",
        },
      }
    end,
    presentation = {
      mode = "list",
      row = {
        primary = function(item)
          return item.name
        end,
        description = function(item)
          return { text = item.description }
        end,
      },
    },
  }

  if not mode then
    return
  end

  local input = trim(editor.prompt("Nome o percorso del luogo"))
  if input == "" then
    return
  end

  local data = nil

  if mode.value == "wikidata" then
    local picked = viaggio.pickWikidataPlace(input)
    if not picked then
      notify("Nessun risultato Wikidata")
      return
    end
    data = viaggio.wikidataPlace(picked.id)
  end

  local displayName =
    data and data.displayName
    or basename(input)

  local pageName =
    string.startsWith(input, "luoghi/")
    and input
    or "luoghi/" .. input

  if space.pageExists(pageName) then
    editor.navigate(pageName)
    return
  end

  local rows = {
    "---",
    "displayName: " .. escapeYaml(displayName),
    "tipoAmministrativo:",
    "divisioneIntermedia:",
    "aliases: []",
  }

  if data and data.wikipedia then
    table.insert(rows, "wikipedia: " .. escapeYaml(data.wikipedia))
  else
    table.insert(rows, "wikipedia:")
  end

  if data and data.coordinate then
    table.insert(
      rows,
      string.format(
        "coordinate: [%.5f, %.5f]",
        data.coordinate[1],
        data.coordinate[2]
      )
    )
  else
    table.insert(rows, "coordinate: []")
  end

  table.insert(rows, "tags: []")
  table.insert(rows, "---")
  table.insert(rows, "")
  table.insert(rows, "# " .. displayName)
  table.insert(rows, "")

  space.writePage(pageName, table.concat(rows, "\n"))
  editor.navigate(pageName)
end

function viaggio.verifyPlace()
  local pageName = editor.getCurrentPage()

  if pageName ~= "luoghi"
    and not string.startsWith(pageName, "luoghi/")
  then
    notify("La pagina corrente non è un Luogo")
    return
  end

  local meta = readMeta(pageName) or {}
  local name = trim(meta.displayName)

  if name == "" then
    name = basename(pageName)
  end

  local results = viaggio.wikidataSearch(name, "it")
  if #results == 0 then
    results = viaggio.wikidataSearch(name, "en")
  end

  if #results == 0 then
    notify("Nessun risultato Wikidata per " .. name)
    return
  end

  local first = results[1]
  local place = viaggio.wikidataPlace(first.id)

  if not place then
    notify("Impossibile leggere il risultato Wikidata")
    return
  end

  local coordinate = ""
  if place.coordinate then
    coordinate = string.format(
      " — %.5f, %.5f",
      place.coordinate[1],
      place.coordinate[2]
    )
  end

  notify(
    (place.displayName or name)
    .. " — "
    .. tostring(place.qid)
    .. coordinate
  )
end

function viaggio.searchInformation()
  local queryText = trim(
    editor.prompt(
      "Ricerca informazioni",
      basename(editor.getCurrentPage())
    )
  )

  if queryText == "" then
    return
  end

  local results = viaggio.wikidataSearch(queryText, "it")
  if #results == 0 then
    results = viaggio.wikidataSearch(queryText, "en")
  end

  if #results == 0 then
    notify("Nessun risultato")
    return
  end

  local picked = view.pick {
    title = "Risultati",
    filter = false,
    source = function()
      return results
    end,
    presentation = {
      mode = "list",
      row = {
        primary = function(item)
          return item.label or item.id
        end,
        description = function(item)
          local text = trim(item.description)
          if text == "" then
            text = item.id or ""
          end
          return { text = text }
        end,
      },
    },
  }

  if not picked then
    return
  end

  local place = viaggio.wikidataPlace(picked.id)
  if not place then
    return
  end

  local lines = {
    "## " .. (place.displayName or picked.label or picked.id),
    "",
  }

  if trim(place.description) ~= "" then
    table.insert(lines, place.description)
    table.insert(lines, "")
  end

  table.insert(lines, "Wikidata: https://www.wikidata.org/wiki/" .. picked.id)

  if place.wikipedia then
    table.insert(lines, "Wikipedia: " .. place.wikipedia)
  end

  if place.coordinate then
    table.insert(
      lines,
      string.format(
        "Coordinate: %.5f, %.5f",
        place.coordinate[1],
        place.coordinate[2]
      )
    )
  end

  editor.insertAtCursor(
    table.concat(lines, "\n") .. "\n",
    false,
    true
  )
end

function viaggio.collectMapPoints(pageName)
  pageName = pageName or editor.getCurrentPage()

  local pages = { readMeta(pageName) }

  for _, p in ipairs(index.subPages(pageName) or {}) do
    table.insert(pages, p)
  end

  local points = {}

  for _, p in ipairs(pages) do
    if p
      and type(p.coordinate) == "table"
      and tonumber(p.coordinate[1])
      and tonumber(p.coordinate[2])
    then
      table.insert(points, {
        name = p.displayName or basename(p.name),
        page = p.name,
        lat = tonumber(p.coordinate[1]),
        lon = tonumber(p.coordinate[2]),
      })
    end

    if p and type(p.luoghi) == "table" then
      for _, link in ipairs(p.luoghi) do
        local target = string.match(link, "^%[%[([^]|]+)")
        if target then
          local lp = readMeta(target)
          if lp
            and type(lp.coordinate) == "table"
            and tonumber(lp.coordinate[1])
            and tonumber(lp.coordinate[2])
          then
            table.insert(points, {
              name = lp.displayName or basename(target),
              page = target,
              lat = tonumber(lp.coordinate[1]),
              lon = tonumber(lp.coordinate[2]),
            })
          end
        end
      end
    end
  end

  return points
end

function viaggio.mapCurrentTree()
  local points = viaggio.collectMapPoints()

  if #points == 0 then
    notify("Nessun luogo con coordinate trovato")
    return
  end

  local items = {}
  for _, p in ipairs(points) do
    table.insert(
      items,
      string.format(
        "- [[%s|%s]] — %.5f, %.5f",
        p.page or "",
        p.name or "",
        p.lat,
        p.lon
      )
    )
  end

  view.define {
    name = "viaggio.mapPoints",
    title = "Mappa pagina e sottopagine",
    dock = "modal",
    supportedDocks = { "modal", "rhs", "bhs" },
    content = function()
      return table.concat(items, "\n")
    end,
  }

  view.open("viaggio.mapPoints")
end

function viaggio.diagnostics()
  local current = editor.getCurrentPage()
  local trip, tripMeta = viaggio.currentTripPage(current)
  local points = viaggio.collectMapPoints(current)

  local lines = {
    "# Diagnostica Mio Viaggio",
    "",
    "- Pagina corrente: `" .. tostring(current) .. "`",
    "- Viaggio rilevato: `" .. tostring(trip or "nessuno") .. "`",
    "- Sottopagine: " .. tostring(#(index.subPages(current) or {})),
    "- Punti con coordinate: " .. tostring(#points),
    "- Readeck disponibile: " .. tostring(
      type(readeck) == "table"
      and type(readeck.catalogView) == "function"
    ),
  }

  view.define {
    name = "viaggio.diagnostics",
    title = "Diagnostica Mio Viaggio",
    dock = "modal",
    supportedDocks = { "modal", "rhs", "bhs" },
    content = function()
      return table.concat(lines, "\n")
    end,
  }

  view.open("viaggio.diagnostics")
end

function viaggio.testHttpWikidata()
  local response = net.proxyFetch(
    "https://www.wikidata.org/w/api.php"
      .. "?action=wbsearchentities"
      .. "&search=Torino"
      .. "&language=it"
      .. "&type=item"
      .. "&limit=1"
      .. "&format=json"
      .. "&formatversion=2",
    {
      headers = { Accept = "application/json" },
    }
  )

  if not response or not response.ok then
    notify(
      "Test HTTP/Wikidata fallito — HTTP "
      .. tostring(response and response.status or "errore")
    )
    return
  end

  local item =
    response.body
    and response.body.search
    and response.body.search[1]

  if not item then
    notify("HTTP OK — Wikidata senza risultati")
    return
  end

  local place = viaggio.wikidataPlace(item.id)
  local coordinate = ""

  if place and place.coordinate then
    coordinate = string.format(
      " — %.5f, %.5f",
      place.coordinate[1],
      place.coordinate[2]
    )
  end

  notify(
    "HTTP/Wikidata OK — "
    .. tostring(item.label or item.id)
    .. " — "
    .. tostring(item.id)
    .. coordinate
  )
end

local function menuItem(name, description, commandName)
  return {
    name = name,
    description = description,
    command = commandName,
  }
end

function viaggio.menu()
  local items = {
    menuItem(
      "Nuovo viaggio",
      "Operative — crea Viaggio, Giorno 01 e pagine di supporto",
      "Viaggio: Nuovo viaggio"
    ),
    menuItem(
      "Nuovo giorno",
      "Operative — crea il prossimo Giorno del viaggio corrente",
      "Viaggio: Nuovo giorno"
    ),
    menuItem(
      "Nuova pagina figlia",
      "Operative — crea una pagina sotto quella corrente",
      "Viaggio: Nuova pagina figlia"
    ),
    menuItem(
      "Ricerca informazioni",
      "Operative — ricerca Wikidata e inserisce le informazioni scelte",
      "Viaggio: Ricerca informazioni"
    ),
    menuItem(
      "Nuovo luogo",
      "Luoghi — crea un Luogo con o senza Wikidata",
      "Viaggio: Nuovo luogo"
    ),
    menuItem(
      "Verifica Luogo",
      "Luoghi — verifica la pagina corrente con Wikidata",
      "Viaggio: Verifica Luogo"
    ),
    menuItem(
      "Mappa pagina e sottopagine",
      "Luoghi — raccoglie i luoghi con coordinate",
      "Viaggio: Mappa pagina e sottopagine"
    ),
    menuItem(
      "Diagnostica",
      "Tecniche — riepilogo della pagina corrente",
      "Viaggio: Diagnostica"
    ),
    menuItem(
      "Test HTTP e Wikidata",
      "Tecniche — verifica net.proxyFetch e Wikidata",
      "Viaggio: Test HTTP e Wikidata"
    ),
  }

  local picked = view.pick {
    title = "Viaggio",
    filter = false,
    source = function()
      return items
    end,
    presentation = {
      mode = "list",
      row = {
        primary = function(item)
          return item.name
        end,
        description = function(item)
          return { text = item.description }
        end,
      },
    },
  }

  if picked and picked.command then
    editor.invokeCommand(picked.command)
  end
end

command.define {
  name = "Viaggio: Menu",
  run = viaggio.menu,
}

command.define {
  name = "Viaggio: Nuovo viaggio",
  run = viaggio.newTrip,
}

command.define {
  name = "Viaggio: Nuovo giorno",
  run = viaggio.newDay,
}

command.define {
  name = "Viaggio: Nuova pagina figlia",
  run = viaggio.newChild,
}

command.define {
  name = "Viaggio: Ricerca informazioni",
  run = viaggio.searchInformation,
}

command.define {
  name = "Viaggio: Nuovo luogo",
  run = viaggio.newPlace,
}

command.define {
  name = "Viaggio: Verifica Luogo",
  run = viaggio.verifyPlace,
}

command.define {
  name = "Viaggio: Mappa pagina e sottopagine",
  run = viaggio.mapCurrentTree,
}

command.define {
  name = "Viaggio: Diagnostica",
  run = viaggio.diagnostics,
}

command.define {
  name = "Viaggio: Test HTTP e Wikidata",
  run = viaggio.testHttpWikidata,
}
```
