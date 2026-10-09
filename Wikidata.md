---
name: "Library/MG/Wikidata"
tags: meta/library
description: "Ricerca e lettura dei dati di un luogo da Wikidata/Wikipedia: QID, etichetta, coordinate, tipo amministrativo, divisione ISO 3166-2."
version: "0.00"
versionDate: 2026-10-09
pageDecoration.prefix: "🌐 "
share.uri: "github:marco10x15/silverbullet-libraries/Wikidata.md"
---

# 🌐 Wikidata

**Versione:** 0.00 — 09.10.2026 (test)

Libreria indipendente per cercare un luogo su Wikidata/Wikipedia e leggerne i dati. Estratta da `Mio_Diario` (Configurazioni, blocco 2) e unificata con le funzioni equivalenti di `Mio_Viaggio` (`viaggio.wikidata*`).

La libreria **non modifica pagine e non scrive stato**: restituisce solo dati. Creazione e verifica delle pagine Luogo restano in `Mio_Diario` / `Mio_Viaggio`. L'unico stato è una cache in memoria degli elementi letti, eliminata a ogni ricarica dello Space Lua.

## Dipendenze

- nessuna libreria personale;
- `net.proxyFetch` verso `www.wikidata.org` e `<lingua>.wikipedia.org`;
- `js` solo se `proxyFetch` restituisce il body come stringa.

## Configurazione

`wikidata.language` (default `it`): lingua di ricerca, etichette, alias e link Wikipedia. L'inglese resta il fallback per ricerca, etichetta e descrizione.

```lua
config.set("wikidata.language", "it")
```

## Funzioni

| Funzione | Restituisce |
|---|---|
| `wikidata.request(url)` | body JSON già decodificato; errore se HTTP non ok |
| `wikidata.search(testo, lingua, limite)` | lista di `{id, label, description}` |
| `wikidata.qidFromTitle(titolo, lingua)` | QID dal titolo di una voce Wikipedia |
| `wikidata.qid(input)` | QID da QID, URL Wikipedia o testo libero |
| `wikidata.entity(qid)` | elemento Wikidata (con cache) |
| `wikidata.claim(entity, property)` | valore del primo claim |
| `wikidata.label(entity)` | etichetta |
| `wikidata.aliases(entity)` | lista di alias |
| `wikidata.wikipedia(entity, lingua)` | URL della voce Wikipedia |
| `wikidata.coordinate(entity)` | `{lat, lon}` |
| `wikidata.adminType(entity)` | `tipoAmministrativo, tipoWikidata` |
| `wikidata.isoDivision(entity)` | codice ISO 3166-2 del primo antenato |
| `wikidata.summary(entity)` | dati base, senza altre richieste HTTP |
| `wikidata.place(input)` | dati completi del luogo |

`wikidata.place()` restituisce i nomi di attributo del frontmatter Luogo: `qid`, `input`, `displayName`, `description`, `tipoAmministrativo`, `tipoWikidata`, `divisioneIntermedia`, `wikipedia`, `coordinate`, `aliases`, `tags`. Se non trova l'elemento restituisce `{error = ...}`; gli errori HTTP generano un errore Lua (usare `pcall`).

Esempio:

```
${wikidata.place("Torino").divisioneIntermedia}
```

## Differenze rispetto alle librerie di origine

- `urlEncode` e `urlDecode` restituiscono un solo valore (prima `gsub` restituiva anche il conteggio).
- URL Wikipedia: la lingua è letta dall'URL (prima solo `itwiki`) e `#frammento` / `?query` sono ignorati.
- `urlDecode` non converte più `+` in spazio (in un percorso `/wiki/` il `+` è un carattere letterale).
- Link Wikipedia nel formato di `Mio_Diario` (codificato); `Mio_Viaggio` lo scriveva non codificato e con fallback `enwiki`, qui disponibile con `wikidata.wikipedia(entity, "en")`.
- `displayName` è `nil` senza etichetta (`Mio_Viaggio` usava il QID).
- Cache degli elementi: tipo e antenati si ripetono tra luoghi diversi.

## Implementazione

```space-lua
wikidata = wikidata or {}
wikidata._entities = wikidata._entities or {}

local API = "https://www.wikidata.org/w/api.php"

local function lang()
  return config.get("wikidata.language", "it")
end

local function urlEncode(value)
  return (string.gsub(tostring(value or ""), "([^%w%-_%.~])", function(c)
    return string.format("%%%02X", string.byte(c))
  end))
end

local function urlDecode(value)
  return (string.gsub(value, "%%(%x%x)", function(hex)
    return string.char(tonumber(hex, 16))
  end))
end

-- Valore localizzato (lingua configurata, poi inglese) da labels/descriptions
local function localized(values)
  values = values or {}
  local item = values[lang()] or values.en
  return item and item.value
end

-- Ricerca testuale sulla Wikipedia della lingua configurata (ultimo fallback)
local function wikipediaTitle(text)
  local data = wikidata.request(
    "https://" .. lang() .. ".wikipedia.org/w/api.php"
      .. "?action=query&list=search&srlimit=1"
      .. "&srsearch=" .. urlEncode(text)
      .. "&format=json&formatversion=2"
  )
  -- ["query"]: come nell'originale, evita ambiguità con la keyword query
  local result = data["query"] and data["query"].search and data["query"].search[1]
  return result and result.title
end


-- ============================================================
-- RICHIESTE E RICERCA
-- ============================================================

function wikidata.request(url)
  local response = net.proxyFetch(url, {
    headers = { Accept = "application/json" }
  })

  if not response or not response.ok then
    error("HTTP " .. tostring(response and response.status or "errore"))
  end

  local body = response.body

  if type(body) == "string" then
    body = js.tolua(js.window.JSON.parse(body))
  end

  return body
end

function wikidata.search(text, language, limit)
  local data = wikidata.request(
    API .. "?action=wbsearchentities"
      .. "&search=" .. urlEncode(text)
      .. "&language=" .. (language or lang())
      .. "&uselang=" .. lang()
      .. "&type=item"
      .. "&limit=" .. (limit or 8)
      .. "&format=json&formatversion=2"
  )

  return data.search or {}
end

function wikidata.qidFromTitle(title, language)
  local data = wikidata.request(
    API .. "?action=wbgetentities"
      .. "&sites=" .. (language or lang()) .. "wiki"
      .. "&titles=" .. urlEncode(title)
      .. "&props=&format=json&formatversion=2"
  )

  for qid in pairs(data.entities or {}) do
    if string.match(qid, "^Q%d+$") then
      return qid
    end
  end
end

-- Input: QID, URL di una voce Wikipedia o testo libero
function wikidata.qid(input)
  input = tostring(input or "")

  if string.match(input, "^Q%d+$") then
    return input
  end

  local site, title = string.match(
    input,
    "([%w%-]+)%.wikipedia%.org/wiki/([^#?]+)"
  )

  if title then
    title = string.gsub(urlDecode(title), "_", " ")
    return wikidata.qidFromTitle(title, site)
  end

  local found = wikidata.search(input, lang(), 1)[1]
    or wikidata.search(input, "en", 1)[1]

  if found then
    return found.id
  end

  title = wikipediaTitle(input)
  return title and wikidata.qidFromTitle(title)
end


-- ============================================================
-- LETTURA ELEMENTO
-- ============================================================

function wikidata.entity(qid)
  local cache = wikidata._entities

  if not cache[qid] then
    local data = wikidata.request(
      "https://www.wikidata.org/wiki/Special:EntityData/"
        .. urlEncode(qid) .. ".json"
    )
    cache[qid] = data.entities and data.entities[qid]
  end

  return cache[qid]
end

function wikidata.claim(entity, property)
  local claims = entity and entity.claims and entity.claims[property]
  local snak = claims and claims[1] and claims[1].mainsnak
  return snak and snak.datavalue and snak.datavalue.value
end

function wikidata.label(entity)
  return localized(entity and entity.labels)
end

function wikidata.aliases(entity)
  local result = {}
  local list = entity and entity.aliases and entity.aliases[lang()] or {}

  for _, alias in ipairs(list) do
    table.insert(result, alias.value)
  end

  return result
end

function wikidata.wikipedia(entity, language)
  language = language or lang()

  local sitelink = entity
    and entity.sitelinks
    and entity.sitelinks[language .. "wiki"]

  if not sitelink or not sitelink.title then
    return nil
  end

  return "https://" .. language .. ".wikipedia.org/wiki/"
    .. urlEncode(string.gsub(sitelink.title, " ", "_"))
end

function wikidata.coordinate(entity)
  local value = wikidata.claim(entity, "P625")

  if value and value.latitude and value.longitude then
    return { value.latitude, value.longitude }
  end
end


-- ============================================================
-- DATI AMMINISTRATIVI
-- ============================================================

-- Primo "istanza di" (P31): tipo normalizzato e etichetta Wikidata.
-- Se l'etichetta manca restituisce nil e il QID del tipo.
function wikidata.adminType(entity)
  local instance = wikidata.claim(entity, "P31")

  if not instance or not instance.id then
    return nil, nil
  end

  local label = wikidata.label(wikidata.entity(instance.id))

  if not label then
    return nil, instance.id
  end

  local lower = string.lower(label)

  local function has(text)
    return string.find(lower, text, 1, true) ~= nil
  end

  if has("comune") or has("municipality") then
    return "comune", label
  end

  if has("stato sovrano")
    or lower == "stato"
    or lower == "country"
    or lower == "stato federato degli stati uniti d'america"
  then
    return "stato", label
  end

  if lower == "grande città" then
    return "città", label
  end

  if has("regione") then return "regione", label end
  if has("provincia") then return "provincia", label end
  if has("dipartimento") then return "dipartimento", label end
  if has("distretto") then return "distretto", label end

  return nil, label
end

-- Risale la catena "ubicato nell'entità amministrativa" (P131), al massimo
-- 6 livelli, fino al primo elemento con codice ISO 3166-2 (P300).
function wikidata.isoDivision(entity)
  local current = entity

  for _ = 1, 6 do
    local parent = wikidata.claim(current, "P131")
    current = parent and parent.id and wikidata.entity(parent.id)

    if not current then
      return nil
    end

    local iso = wikidata.claim(current, "P300")

    if type(iso) == "string" then
      return iso
    end
  end
end


-- ============================================================
-- DATI DEL LUOGO
-- ============================================================

-- Dati base di un elemento già letto (nessuna richiesta HTTP)
function wikidata.summary(entity)
  return {
    qid = entity.id,
    displayName = wikidata.label(entity),
    description = localized(entity.descriptions),
    wikipedia = wikidata.wikipedia(entity),
    coordinate = wikidata.coordinate(entity),
    aliases = wikidata.aliases(entity)
  }
end

-- Dati completi: summary + tipo amministrativo + divisione ISO + tag
function wikidata.place(input)
  local qid = wikidata.qid(input)

  if not qid then
    return { error = "Nessun elemento Wikidata trovato", input = input }
  end

  local entity = wikidata.entity(qid)

  if not entity then
    return { error = "Elemento Wikidata non leggibile", qid = qid }
  end

  local place = wikidata.summary(entity)
  place.input = input
  place.qid = qid
  place.tipoAmministrativo, place.tipoWikidata = wikidata.adminType(entity)
  place.divisioneIntermedia = wikidata.isoDivision(entity)
  place.tags = {}

  if not place.tipoAmministrativo and place.tipoWikidata then
    table.insert(place.tags, (string.gsub(place.tipoWikidata, "%s+", "_")))
  end

  return place
end


-- ============================================================
-- COMANDO DI TEST
-- ============================================================

command.define {
  name = "Wikidata: Test",
  run = function()
    local input = editor.prompt("Luogo da cercare su Wikidata", "Torino")

    if not input or input == "" then
      return
    end

    local ok, place = pcall(wikidata.place, input)

    if not ok then
      editor.flashNotification("Wikidata: " .. tostring(place), "error")
      return
    end

    if place.error then
      editor.flashNotification(place.error .. ": " .. input, "error")
      return
    end

    local text = (place.displayName or "?") .. " — " .. place.qid

    if place.tipoAmministrativo then
      text = text .. " — " .. place.tipoAmministrativo
    end

    if place.divisioneIntermedia then
      text = text .. " — " .. place.divisioneIntermedia
    end

    if place.coordinate then
      text = text .. string.format(
        " — %.5f, %.5f", place.coordinate[1], place.coordinate[2]
      )
    end

    editor.flashNotification(text)
  end
}
```
