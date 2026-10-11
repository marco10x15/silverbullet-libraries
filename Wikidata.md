---
name: "Library/MG/Wikidata"
tags: meta/library
description: "Ricerca dati e immagini di un luogo da Wikidata/Wikipedia, creazione e verifica delle pagine Luogo."
version: "0.03"
versionDate: 2026-10-11
pageDecoration.prefix: "🌐 "
share.uri: "github:marco10x15/silverbullet-libraries/Wikidata.md"
---

# 🌐 Wikidata

**Versione:** 0.03 — 11.10.2026 (test)

Libreria indipendente per cercare un luogo su Wikidata/Wikipedia, leggerne i dati (compresa l'immagine simbolo) e creare o verificare le pagine Luogo. Estratta da `Mio_Diario` (Configurazioni, blocchi 1-5) e da `Mio_Viaggio` (`viaggio.wikidata*`, `viaggio.newPlace`), senza modificarle.

Pensata per essere provata in uno Space di test: non richiede `Mio_Diario`, `Mio_Viaggio`, `DateFormat` né altre librerie.

Le pagine Markdown restano la fonte primaria. La libreria scrive solo con i comandi espliciti `Nuovo Luogo` e `Verifica Luogo`; l'unico stato è una cache in memoria degli elementi letti, eliminata a ogni ricarica dello Space Lua.

## Dipendenze

- `net.proxyFetch` verso `www.wikidata.org`, `<lingua>.wikipedia.org` e `commons.wikimedia.org`;
- API SilverBullet: `editor.prompt`, `editor.filterBox`, `editor.showPanel`, `editor.insertAtCursor`, `index.extractFrontmatter`, `index.patchFrontmatter`, `space.writePage`;
- `template.fromPage` solo se è configurato un template (opzionale);
- `js` solo se `proxyFetch` restituisce il body come stringa.

## Configurazione

```lua
config.set("wikidata.language", "it")            -- lingua di ricerca, etichette, alias, Wikipedia
config.set("wikidata.directory", "luoghi")       -- radice delle pagine Luogo
config.set("wikidata.luogoTemplate", nil)        -- pagina template per il corpo di un nuovo Luogo (opzionale)
config.set("wikidata.imagePriority", {"image", "coatOfArms", "flag"})  -- immagine simbolo
```

Con `wikidata.luogoTemplate` impostato (per esempio `Library/MG/Mio_Diario/Page Templates/Luogo`) il corpo della nuova pagina viene generato dal template, come in `Mio_Diario`. Senza template il corpo è `# <displayName>`.

## Funzioni

| Funzione | Restituisce |
|---|---|
| `wikidata.request(url)` | body JSON già decodificato; errore se HTTP non ok |
| `wikidata.search(testo, lingua, limite)` | lista di `{id, label, description}` |
| `wikidata.qidFromTitle(titolo, lingua)` | QID dal titolo di una voce Wikipedia |
| `wikidata.qidFromWikipediaSearch(testo)` | QID dalla ricerca testuale su Wikipedia |
| `wikidata.qid(input)` | QID da QID, URL Wikipedia o testo libero |
| `wikidata.entity(qid)` | elemento Wikidata (con cache) |
| `wikidata.claim(entity, property)` | valore del primo claim |
| `wikidata.values(entity, property)` | tutti i valori non deprecati, prima i preferiti |
| `wikidata.label(entity)` / `wikidata.aliases(entity)` | etichetta / alias |
| `wikidata.wikipedia(entity, lingua)` | URL della voce Wikipedia |
| `wikidata.coordinate(entity)` | `{lat, lon}` |
| `wikidata.adminType(entity)` | `tipoAmministrativo, tipoWikidata` |
| `wikidata.isoDivision(entity)` | codice ISO 3166-2 del primo antenato |
| `wikidata.images(entity)` | immagini disponibili: `{type, property, label, file, url, page}` |
| `wikidata.symbolImage(images)` | l'immagine simbolo secondo `wikidata.imagePriority` |
| `wikidata.fileName(valore)` | nome file Commons da nome, URL `Special:FilePath` o URL di pagina |
| `wikidata.imageUrl(file, larghezza)` | link permanente `Special:FilePath` |
| `wikidata.commonsPage(file)` | pagina Commons del file (licenza, autore) |
| `wikidata.imageInfo(file, larghezza)` | `{url, thumbUrl, page, mime, width, height, license, author, attributionRequired}` |
| `wikidata.chain(entity, max)` | catena amministrativa (P131): `{level, qid, label, tipo, tipoWikidata, iso}` |
| `wikidata.country(entity)` | stato (P17) e continenti (P30): `{qid, label, continents}` |
| `wikidata.summary(entity)` | dati base, senza altre richieste HTTP |
| `wikidata.place(input)` | dati completi del luogo |
| `wikidata.frontmatterLuogo(dati)` | frontmatter di una nuova pagina Luogo |
| `wikidata.creaPaginaLuogo(pagina, dati)` | scrive la pagina e la apre |
| `wikidata.nuovoLuogo()` | flusso interattivo di creazione |
| `wikidata.verificaLuogoDati()` | confronto frontmatter della pagina corrente / Wikidata |
| `wikidata.patchMancantiLuogo(verifica)` | operazioni `set-key` per i soli dati mancanti |
| `wikidata.verificaLuogo()` | verifica interattiva con pannello e applicazione dei dati mancanti |
| `wikidata.inserisciImmagine()` | inserisce nella pagina un'immagine scelta tra quelle disponibili |

`wikidata.place()` restituisce i nomi di attributo del frontmatter Luogo: `qid`, `input`, `displayName`, `description`, `tipoAmministrativo`, `tipoWikidata`, `divisioneIntermedia`, `wikipedia`, `coordinate`, `aliases`, `tags`, `immagine`, `immagini`. Senza etichetta `displayName` è il QID. Se non trova l'elemento restituisce `{error = ...}`; gli errori HTTP generano un errore Lua (usare `pcall`).

## Comandi

- `Wikidata: Nuovo Luogo`: sceglie la directory, cerca su Wikidata (più candidati: scelta da elenco, con la voce *Senza Wikidata* per una pagina vuota), poi crea la pagina.
- `Wikidata: Verifica Luogo`: confronta la pagina corrente con Wikidata in un pannello e, solo su scelta esplicita, inserisce i dati mancanti. Non sovrascrive mai valori presenti.
- `Wikidata: Immagini del Luogo`: elenca le immagini disponibili per il luogo della pagina corrente e inserisce quella scelta.
- `Wikidata: Catena`: diagnostica, non scrive nulla. Mostra in un pannello la catena amministrativa (P131) del luogo con QID, etichetta, tipo e codice ISO di ogni livello, più stato (P17) e continente (P30), e la stessa tabella in testo da copiare. Serve a verificare su dati reali come cambia la catena da paese a paese prima di definire la creazione dei padri.
- `Wikidata: Test`: notifica con i dati principali di un luogo, senza scrivere nulla.

## Immagini

### Tipi disponibili (`wikidata.imageTypes`)

| Chiave | Proprietà | Contenuto |
|---|---|---|
| `image` | P18 | immagine rappresentativa (foto) |
| `coatOfArms` | P94 | stemma |
| `flag` | P41 | bandiera |
| `seal` | P158 | sigillo |
| `locatorMap` | P242 | mappa di localizzazione |
| `collage` | P2716 | collage |
| `panorama` | P4291 | vista panoramica |
| `nightView` | P3451 | vista notturna |

Il valore di ogni proprietà è il nome di un file di Wikimedia Commons. Se una proprietà ha più valori vale l'ordine: rango *preferred*, poi *normal*; i *deprecated* sono ignorati. L'immagine simbolo è la prima trovata in `wikidata.imagePriority` (default: immagine, stemma, bandiera).

### Link da usare

- **Raccomandato:** `https://commons.wikimedia.org/wiki/Special:FilePath/<File>` con, nel corpo della pagina, `?width=<px>`. Si costruisce dal solo nome del file, senza hash né richieste, e Commons lo reindirizza al file corrente.
- **Attribuzione:** `https://commons.wikimedia.org/wiki/File:<File>` (licenza e autore); `wikidata.imageInfo()` li legge via API.
- **Sconsigliato:** `upload.wikimedia.org/...`, perché il percorso contiene l'hash MD5 del nome del file e per le miniature un secondo nome generato.
- **Limite:** è un collegamento a Commons, non una copia. Chiunque può rinominare o sostituire il file; Commons indica l'hotlinking come ammesso ma non generalmente raccomandato.

### Dove inserirlo

- **Frontmatter:** `immagine: "https://commons.wikimedia.org/wiki/Special:FilePath/<File>"`, senza larghezza. È un dato derivato, ricostruibile da Wikidata con `Verifica Luogo`.
- **Corpo della pagina:** `![<nome>](https://commons.wikimedia.org/wiki/Special:FilePath/<File>?width=800)` seguito da `*Fonte: [Wikimedia Commons](https://commons.wikimedia.org/wiki/File:<File>)*`. È ciò che inserisce `Wikidata: Immagini del Luogo`.

## Provenienza

| Elemento | Origine |
|---|---|
| ricerca, QID, entità, tipo, divisione ISO, coordinate, alias | `Mio_Diario` (Configurazioni, blocco 2), unificata con `viaggio.wikidata*` |
| scelta tra più candidati e creazione senza Wikidata | `viaggio.newPlace` |
| directory, `frontmatterLuogo`, `creaPaginaLuogo`, `nuovoLuogo` | `Mio_Diario` (Configurazioni, blocchi 3-4) |
| `verificaLuogoDati`, `patchMancantiLuogo`, `verificaLuogo` | `Mio_Diario` (Configurazioni, blocco 5) |
| `viaggio.verifyPlace` (solo notifica) | assorbita da `verificaLuogo` |

## Differenze rispetto alle librerie di origine

- Senza etichetta `displayName` è il QID (come `Mio_Viaggio`; `Mio_Diario` lasciava il campo vuoto).
- `urlEncode` e `urlDecode` restituiscono un solo valore (prima `gsub` restituiva anche il conteggio).
- URL Wikipedia: la lingua è letta dall'URL (prima solo `itwiki`) e `#frammento` / `?query` sono ignorati.
- `urlDecode` non converte più `+` in spazio.
- Il tag per un tipo non riconosciuto è inserito correttamente (prima `table.insert(tags, string.gsub(...))` passava due valori).
- Creazione: con testo libero si sceglie tra i candidati (prima `Mio_Diario` prendeva il primo risultato).
- Nuovo campo opzionale `immagine` in creazione e verifica; i campi esistenti non cambiano.
- I comandi hanno nomi propri (`Wikidata: …`) e non sostituiscono quelli di `Mio_Diario` / `Mio_Viaggio`, che restano invariati.

## Implementazione — Dati

```space-lua
-- priority: 10

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

-- Nome file con underscore, come richiesto dai link Commons
local function commonsName(file)
  return (string.gsub(wikidata.fileName(file), " ", "_"))
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

function wikidata.search(text, language, max)
  local data = wikidata.request(
    API .. "?action=wbsearchentities"
      .. "&search=" .. urlEncode(text)
      .. "&language=" .. (language or lang())
      .. "&uselang=" .. lang()
      .. "&type=item"
      .. "&limit=" .. (max or 8)
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

-- Ultimo fallback: ricerca testuale sulla Wikipedia della lingua configurata
function wikidata.qidFromWikipediaSearch(text)
  local data = wikidata.request(
    "https://" .. lang() .. ".wikipedia.org/w/api.php"
      .. "?action=query&list=search&srlimit=1"
      .. "&srsearch=" .. urlEncode(text)
      .. "&format=json&formatversion=2"
  )
  -- ["query"]: come nell'originale, evita ambiguità con la keyword query
  local result = data["query"] and data["query"].search and data["query"].search[1]

  if result and result.title then
    return wikidata.qidFromTitle(result.title)
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

  return wikidata.qidFromWikipediaSearch(input)
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

-- Tutti i valori non deprecati di una proprietà: prima i preferiti
function wikidata.values(entity, property)
  local result = {}
  local normal = {}
  local claims = entity and entity.claims and entity.claims[property] or {}

  for _, claim in ipairs(claims) do
    local snak = claim.mainsnak
    local value = snak and snak.datavalue and snak.datavalue.value

    if value ~= nil then
      if claim.rank == "preferred" then
        table.insert(result, value)
      elseif claim.rank ~= "deprecated" then
        table.insert(normal, value)
      end
    end
  end

  for _, value in ipairs(normal) do
    table.insert(result, value)
  end

  return result
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


-- Catena amministrativa: risale P131 per al massimo `max` livelli (default 8),
-- fermandosi su cicli. Livello 0 = l'elemento stesso.
function wikidata.chain(entity, max)
  local result = {}
  local seen = {}
  local current = entity

  for level = 0, max or 8 do
    if not current or seen[current.id] then
      break
    end
    seen[current.id] = true

    local tipo, tipoWikidata = wikidata.adminType(current)
    local iso = wikidata.claim(current, "P300")

    table.insert(result, {
      level = level,
      qid = current.id,
      label = wikidata.label(current) or current.id,
      tipo = tipo,
      tipoWikidata = tipoWikidata,
      iso = type(iso) == "string" and iso or nil
    })

    local parent = wikidata.claim(current, "P131")
    current = parent and parent.id and wikidata.entity(parent.id)
  end

  return result
end

-- Stato (P17) con i suoi continenti (P30), come etichette
function wikidata.country(entity)
  local value = wikidata.claim(entity, "P17")
  local country = value and value.id and wikidata.entity(value.id)

  if not country then
    return nil
  end

  local continents = {}

  for _, continent in ipairs(wikidata.values(country, "P30")) do
    local e = type(continent) == "table"
      and continent.id
      and wikidata.entity(continent.id)

    if e then
      table.insert(continents, wikidata.label(e) or e.id)
    end
  end

  return {
    qid = country.id,
    label = wikidata.label(country) or country.id,
    continents = continents
  }
end


-- ============================================================
-- IMMAGINI (Wikimedia Commons)
-- ============================================================

wikidata.imageTypes = {
  { key = "image", property = "P18", label = "Immagine" },
  { key = "coatOfArms", property = "P94", label = "Stemma" },
  { key = "flag", property = "P41", label = "Bandiera" },
  { key = "seal", property = "P158", label = "Sigillo" },
  { key = "locatorMap", property = "P242", label = "Mappa di localizzazione" },
  { key = "collage", property = "P2716", label = "Collage" },
  { key = "panorama", property = "P4291", label = "Vista panoramica" },
  { key = "nightView", property = "P3451", label = "Vista notturna" }
}

-- Nome file con spazi, da: nome Commons, URL Special:FilePath o URL di pagina
function wikidata.fileName(value)
  value = tostring(value or "")

  if string.match(value, "^https?://") then
    value = string.gsub(value, "[?#].*$", "")
    value = urlDecode(string.match(value, "([^/]+)$") or "")
  end

  value = string.gsub(value, "^[Ff]ile:", "")
  return (string.gsub(value, "_", " "))
end

-- Link permanente: segue il file corrente su Commons
function wikidata.imageUrl(file, width)
  local url = "https://commons.wikimedia.org/wiki/Special:FilePath/"
    .. urlEncode(commonsName(file))

  if width then
    url = url .. "?width=" .. math.floor(width)
  end

  return url
end

function wikidata.commonsPage(file)
  return "https://commons.wikimedia.org/wiki/File:"
    .. urlEncode(commonsName(file))
end

-- Immagini disponibili per l'elemento, nell'ordine di wikidata.imageTypes
function wikidata.images(entity)
  local result = {}

  for _, imageType in ipairs(wikidata.imageTypes) do
    for _, file in ipairs(wikidata.values(entity, imageType.property)) do
      if type(file) == "string" then
        table.insert(result, {
          type = imageType.key,
          property = imageType.property,
          label = imageType.label,
          file = wikidata.fileName(file),
          url = wikidata.imageUrl(file),
          page = wikidata.commonsPage(file)
        })
      end
    end
  end

  return result
end

-- Immagine simbolo: prima corrispondenza secondo wikidata.imagePriority
function wikidata.symbolImage(images)
  local priority = config.get(
    "wikidata.imagePriority",
    { "image", "coatOfArms", "flag" }
  )

  for _, key in ipairs(priority) do
    for _, image in ipairs(images or {}) do
      if image.type == key then
        return image
      end
    end
  end
end

-- Licenza, autore e URL del file da Commons (una richiesta HTTP)
function wikidata.imageInfo(file, width)
  local data = wikidata.request(
    "https://commons.wikimedia.org/w/api.php?action=query"
      .. "&titles=File:" .. urlEncode(commonsName(file))
      .. "&prop=imageinfo&iiprop=url%7Cmime%7Csize%7Cextmetadata"
      .. (width and ("&iiurlwidth=" .. math.floor(width)) or "")
      .. "&format=json&formatversion=2"
  )

  -- ["query"]: query è una parola riservata di Space Lua
  local pages = data["query"] and data["query"].pages
  local page = pages and pages[1]
  local info = page and not page.missing
    and page.imageinfo and page.imageinfo[1]

  if not info then
    return nil
  end

  local meta = info.extmetadata or {}

  local function metaValue(key)
    return meta[key] and meta[key].value
  end

  local artist = metaValue("Artist")

  return {
    url = info.url,
    thumbUrl = info.thumburl,
    page = info.descriptionurl,
    mime = info.mime,
    width = info.width,
    height = info.height,
    license = metaValue("LicenseShortName"),
    author = artist and (string.gsub(artist, "<[^>]*>", "")),
    attributionRequired = metaValue("AttributionRequired") == "true"
  }
end


-- ============================================================
-- DATI DEL LUOGO
-- ============================================================

-- Dati base di un elemento già letto (nessuna richiesta HTTP)
function wikidata.summary(entity)
  local images = wikidata.images(entity)
  local symbol = wikidata.symbolImage(images)

  return {
    qid = entity.id,
    displayName = wikidata.label(entity) or entity.id,
    description = localized(entity.descriptions),
    wikipedia = wikidata.wikipedia(entity),
    coordinate = wikidata.coordinate(entity),
    aliases = wikidata.aliases(entity),
    immagine = symbol and symbol.url,
    immagini = images
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
```

## Implementazione — Creazione e verifica Luogo

```space-lua
-- priority: 9

local MANUAL = "__manuale__"

local function trim(value)
  return (string.gsub(string.gsub(tostring(value or ""), "^%s+", ""), "%s+$", ""))
end

local function isEmpty(value)
  return value == nil or value == ""
end

local function basename(pageName)
  return string.match(pageName or "", "([^/]+)$") or pageName
end

local function yamlQuote(value)
  value = string.gsub(tostring(value or ""), "\\", "\\\\")
  value = string.gsub(value, "\"", "\\\"")
  return "\"" .. value .. "\""
end

local function asList(value)
  if isEmpty(value) then
    return {}
  end
  if type(value) == "table" then
    return value
  end
  return { value }
end

local function listContains(values, wanted)
  for _, value in ipairs(asList(values)) do
    if tostring(value) == tostring(wanted) then
      return true
    end
  end
  return false
end

local function missingListValues(current, proposed)
  local missing = {}
  for _, value in ipairs(asList(proposed)) do
    if not listContains(current, value) then
      table.insert(missing, value)
    end
  end
  return missing
end

local function mergeLists(current, proposed)
  local result = {}
  for _, value in ipairs(asList(current)) do
    table.insert(result, value)
  end
  for _, value in ipairs(missingListValues(current, proposed)) do
    table.insert(result, value)
  end
  return result
end

local function round5(value)
  if value == nil then
    return nil
  end
  return math.floor(value * 100000 + 0.5) / 100000
end

local function normalizzaCoordinate(value)
  if type(value) ~= "table" or not value[1] or not value[2] then
    return value
  end
  return { round5(value[1]), round5(value[2]) }
end

local function coordinateEqual(a, b)
  if type(a) ~= "table" or type(b) ~= "table" then
    return false
  end
  if not a[1] or not a[2] or not b[1] or not b[2] then
    return false
  end
  return math.abs(a[1] - b[1]) < 0.0001
    and math.abs(a[2] - b[2]) < 0.0001
end

local function valueText(value)
  if isEmpty(value) then
    return ""
  end
  if type(value) ~= "table" then
    return tostring(value)
  end
  local parts = {}
  for _, item in ipairs(value) do
    table.insert(parts, tostring(item))
  end
  return table.concat(parts, ", ")
end

local function htmlEscape(value)
  value = string.gsub(tostring(value or ""), "&", "&amp;")
  value = string.gsub(value, "<", "&lt;")
  value = string.gsub(value, ">", "&gt;")
  value = string.gsub(value, '"', "&quot;")
  return value
end

local function baseDirectory()
  return config.get("wikidata.directory", "luoghi")
end

local function inDirectory(pageName)
  local prefix = baseDirectory() .. "/"
  return pageName ~= nil and string.sub(pageName, 1, #prefix) == prefix
end

local function normalizzaDirectory(directory)
  local base = baseDirectory()

  directory = string.gsub(trim(directory), "^/+", "")
  directory = string.gsub(directory, "/+$", "")

  if directory == "" then
    return base
  end
  if directory ~= base and not inDirectory(directory .. "/") then
    directory = base .. "/" .. directory
  end
  return directory
end

local function currentFrontmatter()
  local extracted = index.extractFrontmatter(editor.getText())
  return extracted and extracted.frontmatter or {}
end


-- ============================================================
-- CREAZIONE PAGINA
-- ============================================================

local function directoryNuovoLuogo()
  local currentPage = editor.getCurrentPage()

  if inDirectory(currentPage) then
    local scelta = editor.filterBox(
      "Dove creare il nuovo Luogo?",
      {
        {
          name = "Figlia della pagina corrente",
          description = currentPage,
          value = currentPage
        },
        {
          name = "Altra directory",
          description = "Specifica una directory sotto " .. baseDirectory() .. "/",
          value = "__altra__"
        }
      },
      "Pagina corrente: " .. currentPage
    )

    if not scelta then
      return nil
    end
    if scelta.value ~= "__altra__" then
      return normalizzaDirectory(scelta.value)
    end
  end

  local directory = editor.prompt("Directory di destinazione", baseDirectory() .. "/")
  if not directory then
    return nil
  end
  return normalizzaDirectory(directory)
end

-- Input -> QID. QID e URL Wikipedia sono diretti; il testo libero mostra i
-- candidati con la voce "Senza Wikidata". Restituisce un QID, MANUAL o nil
-- (annullato).
local function sceglieQid(input)
  if string.match(input, "^Q%d+$")
    or string.find(input, "wikipedia.org/wiki/", 1, true)
  then
    local qid = wikidata.qid(input)
    if not qid then
      editor.flashNotification("Nessun elemento Wikidata trovato: " .. input, "error")
    end
    return qid
  end

  local results = wikidata.search(input, nil, 8)
  if #results == 0 then
    results = wikidata.search(input, "en", 8)
  end
  if #results == 0 then
    local qid = wikidata.qidFromWikipediaSearch(input)
    if qid then
      return qid
    end
  end

  local options = {}
  for _, result in ipairs(results) do
    table.insert(options, {
      name = result.label or result.id,
      description = (result.description or "") .. " (" .. result.id .. ")",
      value = result.id
    })
  end
  table.insert(options, {
    name = "Senza Wikidata",
    description = "Crea una pagina Luogo vuota",
    value = MANUAL
  })

  local scelta = editor.filterBox("Wikidata: " .. input, options, "Seleziona il luogo")
  return scelta and scelta.value
end

function wikidata.frontmatterLuogo(dati)
  local lines = {
    "---",
    "displayName: " .. yamlQuote(dati.displayName or "")
  }

  local function scalar(field)
    if dati[field] then
      table.insert(lines, field .. ": " .. yamlQuote(dati[field]))
    else
      table.insert(lines, field .. ":")
    end
  end

  scalar("tipoAmministrativo")
  scalar("divisioneIntermedia")

  table.insert(lines, "aliases:")
  for _, alias in ipairs(dati.aliases or {}) do
    table.insert(lines, "- " .. yamlQuote(alias))
  end

  scalar("wikipedia")

  local c = normalizzaCoordinate(dati.coordinate)
  if c and c[1] and c[2] then
    table.insert(
      lines,
      "coordinate: [" .. string.format("%.5f", c[1]) .. ", "
        .. string.format("%.5f", c[2]) .. "]"
    )
  else
    table.insert(lines, "coordinate:")
  end

  scalar("immagine")

  table.insert(lines, "tags:")
  for _, tag in ipairs(dati.tags or {}) do
    table.insert(lines, "- " .. tag)
  end

  table.insert(lines, "---")
  return table.concat(lines, "\n") .. "\n"
end

function wikidata.creaPaginaLuogo(pageName, dati)
  if space.pageExists(pageName) then
    editor.flashNotification("La pagina esiste già: " .. pageName, "error")
    return
  end

  local text = wikidata.frontmatterLuogo(dati)
  local templatePage = config.get("wikidata.luogoTemplate")

  if templatePage and space.pageExists(templatePage) then
    local tpl = template.fromPage(templatePage)

    space.writePage(pageName, text)
    editor.navigate({ kind = "page", page = pageName })
    editor.insertAtPos(tpl(), #text, true)
    return
  end

  space.writePage(
    pageName,
    text .. "\n# " .. (dati.displayName or basename(pageName)) .. "\n"
  )
  editor.navigate({ kind = "page", page = pageName })
end

function wikidata.nuovoLuogo()
  local directory = directoryNuovoLuogo()
  if not directory then
    return
  end

  local input = editor.prompt("Luogo da cercare", "")
  if not input or trim(input) == "" then
    return
  end
  input = trim(input)

  local qid = sceglieQid(input)
  if not qid then
    return
  end

  local dati = { displayName = input, aliases = {}, tags = {} }

  if qid ~= MANUAL then
    dati = wikidata.place(qid)
    if dati.error then
      editor.flashNotification(dati.error .. ": " .. input, "error")
      return
    end
  end

  local nomePagina = editor.prompt("Nome della pagina", dati.displayName or input)
  if not nomePagina or trim(nomePagina) == "" then
    return
  end

  wikidata.creaPaginaLuogo(directory .. "/" .. trim(nomePagina), dati)
end


-- ============================================================
-- VERIFICA PAGINA
-- ============================================================

local function scalarStatus(current, proposed)
  if isEmpty(proposed) then
    if isEmpty(current) then
      return "OK", ""
    end
    return "N/D", "mantieni"
  end

  if isEmpty(current) then
    return "MANCANTE", "inserisci"
  end

  if tostring(current) == tostring(proposed) then
    return "OK", ""
  end

  return "DIFFERENTE", "mantieni"
end

local function wikipediaStatus(current, proposed, qid)
  if isEmpty(current) then
    if isEmpty(proposed) then
      return "OK", ""
    end
    return "MANCANTE", "inserisci"
  end

  local currentQid = wikidata.qid(current)
  if currentQid and currentQid == qid then
    return "OK", ""
  end

  return "DIFFERENTE", "mantieni"
end

local function coordinateStatus(current, proposed)
  if proposed == nil then
    if current == nil then
      return "OK", ""
    end
    return "N/D", "mantieni"
  end

  if current == nil then
    return "MANCANTE", "inserisci"
  end

  if coordinateEqual(current, proposed) then
    return "OK", ""
  end

  return "DIFFERENTE", "mantieni"
end

-- Il file è lo stesso anche se il link è scritto in un altro modo
local function immagineStatus(current, proposed)
  if isEmpty(proposed) or isEmpty(current) then
    return scalarStatus(current, proposed)
  end

  if wikidata.fileName(current) == wikidata.fileName(proposed) then
    return "OK", ""
  end

  return "DIFFERENTE", "mantieni"
end

local function listStatus(current, proposed)
  local missing = missingListValues(current, proposed)
  if #missing == 0 then
    return "OK", "", missing
  end

  if #asList(current) == 0 then
    return "MANCANTE", "inserisci", missing
  end

  return "INCOMPLETO", "aggiungi " .. tostring(#missing), missing
end

function wikidata.verificaLuogoDati()
  local currentPage = editor.getCurrentPage()
  if not inDirectory(currentPage) then
    return { error = "Il comando è disponibile solo nelle pagine " .. baseDirectory() .. "/" }
  end

  local fm = currentFrontmatter()
  local pageName = basename(currentPage)

  local input = fm.wikipedia
  if isEmpty(input) then
    input = pageName
  end

  local dati = wikidata.place(input)

  if dati.error and not isEmpty(fm.wikipedia) then
    dati = wikidata.place(pageName)
  end

  if dati.error then
    return dati
  end

  dati.coordinate = normalizzaCoordinate(dati.coordinate)

  local rows = {}

  local function addRow(field, current, proposed, status, action, missing)
    table.insert(rows, {
      field = field,
      current = current,
      proposed = proposed,
      status = status,
      action = action,
      missing = missing
    })
  end

  local function addScalar(field)
    local status, action = scalarStatus(fm[field], dati[field])
    addRow(field, fm[field], dati[field], status, action)
  end

  local function addList(field)
    local status, action, missing = listStatus(fm[field], dati[field])
    addRow(field, asList(fm[field]), dati[field], status, action, missing)
  end

  addScalar("displayName")
  addScalar("tipoAmministrativo")
  addScalar("divisioneIntermedia")
  addList("aliases")

  local wStatus, wAction = wikipediaStatus(fm.wikipedia, dati.wikipedia, dati.qid)
  addRow("wikipedia", fm.wikipedia, dati.wikipedia, wStatus, wAction)

  local cStatus, cAction = coordinateStatus(fm.coordinate, dati.coordinate)
  addRow("coordinate", fm.coordinate, dati.coordinate, cStatus, cAction)

  local iStatus, iAction = immagineStatus(fm.immagine, dati.immagine)
  addRow("immagine", fm.immagine, dati.immagine, iStatus, iAction)

  addList("tags")

  return {
    page = currentPage,
    qid = dati.qid,
    input = input,
    frontmatter = fm,
    wikidata = dati,
    rows = rows
  }
end

function wikidata.verificaLuogoHtml(verifica)
  local cell = "<td style='vertical-align:top;border-bottom:1px solid #ddd;padding:.35rem'>"
  local head = "<th style='text-align:left;border-bottom:1px solid #888;padding:.35rem'>"

  local html = {
    "<div style='font-family:sans-serif;padding:1rem'>",
    "<h2>Verifica Luogo</h2>",
    "<p><strong>Pagina:</strong> " .. htmlEscape(verifica.page)
      .. "<br><strong>Wikidata:</strong> " .. htmlEscape(verifica.qid) .. "</p>",
    "<table style='width:100%;border-collapse:collapse'>",
    "<thead><tr>",
    head .. "Campo</th>",
    head .. "Pagina</th>",
    head .. "Wikidata</th>",
    head .. "Stato</th>",
    head .. "Azione</th>",
    "</tr></thead><tbody>"
  }

  for _, row in ipairs(verifica.rows) do
    table.insert(html,
      "<tr>"
        .. cell .. htmlEscape(row.field) .. "</td>"
        .. cell .. htmlEscape(valueText(row.current)) .. "</td>"
        .. cell .. htmlEscape(valueText(row.proposed)) .. "</td>"
        .. cell .. htmlEscape(row.status) .. "</td>"
        .. cell .. htmlEscape(row.action) .. "</td>"
        .. "</tr>"
    )
  end

  table.insert(html, "</tbody></table></div>")
  return table.concat(html, "")
end

-- Solo i dati mancanti: i valori già presenti non vengono mai toccati
function wikidata.patchMancantiLuogo(verifica)
  local fm = verifica.frontmatter or {}
  local dati = verifica.wikidata or {}
  local ops = {}

  local function setKey(field, value)
    table.insert(ops, { op = "set-key", path = field, value = value })
  end

  local function setIfEmpty(field)
    if isEmpty(fm[field]) and not isEmpty(dati[field]) then
      setKey(field, dati[field])
    end
  end

  local function mergeIfNew(field)
    local merged = mergeLists(fm[field], dati[field])
    if #merged > #asList(fm[field]) then
      setKey(field, merged)
    end
  end

  setIfEmpty("displayName")
  setIfEmpty("tipoAmministrativo")
  setIfEmpty("divisioneIntermedia")
  mergeIfNew("aliases")
  setIfEmpty("wikipedia")

  if fm.coordinate == nil and dati.coordinate ~= nil then
    setKey("coordinate", dati.coordinate)
  end

  setIfEmpty("immagine")
  mergeIfNew("tags")

  return ops
end

-- patchFrontmatter scrive le coordinate in forma lista: le riporta in linea
local function coordinateInlineText(text, coordinate)
  if type(coordinate) ~= "table" or not coordinate[1] or not coordinate[2] then
    return text
  end

  local inline = "coordinate: [" .. string.format("%.5f", coordinate[1])
    .. ", " .. string.format("%.5f", coordinate[2]) .. "]"

  return (string.gsub(
    text,
    "coordinate:%s*\n%s*%-%s*[-%d%.]+\n%s*%-%s*[-%d%.]+",
    inline,
    1
  ))
end

function wikidata.applicaMancantiLuogo(verifica)
  local ops = wikidata.patchMancantiLuogo(verifica)
  if #ops == 0 then
    editor.flashNotification("Nessun dato mancante da inserire")
    return false
  end

  local updated = index.patchFrontmatter(editor.getText(), ops)
  updated = coordinateInlineText(
    updated,
    verifica.wikidata and verifica.wikidata.coordinate
  )

  editor.setText(updated)
  editor.rebuildEditorState()
  editor.hidePanel("rhs")
  editor.flashNotification(
    "Inseriti " .. tostring(#ops) .. " aggiornamenti nel frontmatter"
  )
  return true
end

function wikidata.verificaLuogo()
  local verifica = wikidata.verificaLuogoDati()

  if verifica.error then
    editor.flashNotification(verifica.error, "error")
    return
  end

  editor.showPanel("rhs", 1, wikidata.verificaLuogoHtml(verifica), "")

  local ops = wikidata.patchMancantiLuogo(verifica)
  if #ops == 0 then
    editor.flashNotification("Verifica completata: nessun dato mancante")
    return verifica.rows
  end

  local scelta = editor.filterBox(
    "Verifica Luogo",
    {
      {
        name = "Solo verifica",
        description = "Non modificare la pagina",
        value = "check"
      },
      {
        name = "Inserisci solo dati mancanti",
        description = tostring(#ops) .. " aggiornamenti proposti",
        value = "apply"
      }
    },
    "Controlla la tabella nel pannello destro"
  )

  if scelta and scelta.value == "apply" then
    wikidata.applicaMancantiLuogo(verifica)
  end

  return verifica.rows
end


-- ============================================================
-- IMMAGINI NELLA PAGINA
-- ============================================================

-- Inserisce nel corpo l'immagine scelta, con link permanente e fonte
function wikidata.inserisciImmagine()
  local fm = currentFrontmatter()
  local input = fm.wikipedia
  if isEmpty(input) then
    input = basename(editor.getCurrentPage())
  end

  local dati = wikidata.place(input)
  if dati.error then
    editor.flashNotification(dati.error .. ": " .. tostring(input), "error")
    return
  end

  if #dati.immagini == 0 then
    editor.flashNotification("Nessuna immagine su Wikidata per " .. dati.displayName)
    return
  end

  local options = {}
  for i, image in ipairs(dati.immagini) do
    table.insert(options, {
      name = image.label,
      description = image.file,
      value = i
    })
  end

  local scelta = editor.filterBox("Immagini di " .. dati.displayName, options, "Seleziona l'immagine")
  if not scelta then
    return
  end

  local image = dati.immagini[scelta.value]

  editor.insertAtCursor(
    "![" .. dati.displayName .. "](" .. wikidata.imageUrl(image.file, 800) .. ")\n"
      .. "*Fonte: [Wikimedia Commons](" .. image.page .. ")*\n",
    false,
    true
  )
end


-- ============================================================
-- DIAGNOSTICA: CATENA AMMINISTRATIVA
-- ============================================================

-- Pannello con la catena P131 e, sotto, la stessa tabella in testo da copiare
function wikidata.catenaHtml(info)
  local cell = "<td style='vertical-align:top;border-bottom:1px solid #ddd;padding:.35rem'>"
  local head = "<th style='text-align:left;border-bottom:1px solid #888;padding:.35rem'>"

  local country = "non indicato (P17)"
  if info.country then
    country = info.country.label .. " (" .. info.country.qid .. ")"
      .. " — continente (P30): "
      .. (#info.country.continents > 0
        and table.concat(info.country.continents, ", ")
        or "non indicato")
  end

  local html = {
    "<div style='font-family:sans-serif;padding:1rem'>",
    "<h2>Catena amministrativa</h2>",
    "<p><strong>" .. htmlEscape(info.place) .. "</strong> (" .. htmlEscape(info.qid) .. ")<br>"
      .. "<strong>Stato:</strong> " .. htmlEscape(country) .. "</p>",
    "<table style='width:100%;border-collapse:collapse'><thead><tr>",
    head .. "Liv.</th>", head .. "QID</th>", head .. "Etichetta</th>",
    head .. "Tipo</th>", head .. "Tipo Wikidata</th>", head .. "ISO 3166-2</th>",
    "</tr></thead><tbody>"
  }

  local text = {}

  for _, row in ipairs(info.chain) do
    local cols = {
      tostring(row.level), row.qid, row.label,
      row.tipo or "-", row.tipoWikidata or "-", row.iso or "-"
    }

    local tr = { "<tr>" }
    for _, col in ipairs(cols) do
      table.insert(tr, cell .. htmlEscape(col) .. "</td>")
    end
    table.insert(tr, "</tr>")
    table.insert(html, table.concat(tr))

    table.insert(text, table.concat(cols, " | "))
  end

  table.insert(html, "</tbody></table>")
  table.insert(html, "<pre style='white-space:pre-wrap'>"
    .. htmlEscape(info.place .. " (" .. info.qid .. ") — " .. country
      .. "\n" .. table.concat(text, "\n")) .. "</pre></div>")

  return table.concat(html, "")
end

function wikidata.catena()
  local pageName = editor.getCurrentPage()
  local input = ""

  if inDirectory(pageName) then
    local fm = currentFrontmatter()
    input = isEmpty(fm.wikipedia) and basename(pageName) or fm.wikipedia
  end

  input = editor.prompt("Luogo (nome, QID o URL Wikipedia)", input)
  if not input or trim(input) == "" then
    return
  end

  local qid = wikidata.qid(trim(input))
  local entity = qid and wikidata.entity(qid)

  if not entity then
    editor.flashNotification("Nessun elemento Wikidata trovato: " .. trim(input), "error")
    return
  end

  editor.showPanel("rhs", 1, wikidata.catenaHtml({
    place = wikidata.label(entity) or qid,
    qid = qid,
    chain = wikidata.chain(entity),
    country = wikidata.country(entity)
  }), "")
end
```

## Implementazione — Comandi

```space-lua
-- priority: 8

command.define {
  name = "Wikidata: Nuovo Luogo",
  run = function()
    return wikidata.nuovoLuogo()
  end
}

command.define {
  name = "Wikidata: Verifica Luogo",
  run = function()
    return wikidata.verificaLuogo()
  end
}

command.define {
  name = "Wikidata: Immagini del Luogo",
  run = function()
    return wikidata.inserisciImmagine()
  end
}

command.define {
  name = "Wikidata: Catena",
  run = function()
    return wikidata.catena()
  end
}

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

    local text = place.displayName .. " — " .. place.qid

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

    text = text .. " — " .. #place.immagini .. " immagini"

    editor.flashNotification(text)
  end
}
```
