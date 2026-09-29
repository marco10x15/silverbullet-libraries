---
name: "Library/MG/Appunti_Readeck"
tags: meta/library
description: "Utility per gestire, navigare e aggregare appunti readeck in SilverBullet."
pageDecoration.prefix: "📔 "
share.uri: "github:marco10x15/silverbullet-libraries/Appunti_Readeck.md"
version: 0.08
versionDate: 2026-09-29
---

# Appunti Readeck

Libreria indipendente per integrare Readeck in SilverBullet 2.11+.

Quando una funzione non riceve una label esplicita, parte dalla pagina corrente e risale le pagine antenate fino alla prima che contiene nel frontmatter l'attributo `readeck`; il valore di tale attributo viene usato come label Readeck.

## Funzioni pubbliche

### Utilizzabili direttamente nelle pagine SilverBullet

```markdown
${readeck.catalogView()}
```

Renderizza il catalogo dei bookmark associati alla label contenuta nell'attributo frontmatter `readeck` della prima pagina corrente o antenata che lo definisce.

È ancora possibile forzare una label:

```markdown
${readeck.catalogView("Viaggio-Vicoforte")}
```

```markdown
${readeck.annotatedView()}
```

Renderizza solamente i bookmark che possiedono evidenziazioni Readeck e, per ciascuno, le parti evidenziate. Se Readeck restituisce una nota associata all'evidenziazione, viene visualizzata prima del testo evidenziato; se è disponibile anche il colore, il nome italiano del colore viene mostrato tra parentesi dopo `Nota:`. Il testo evidenziato viene reso senza blockquote o indentazione aggiunti dalla libreria.

È ancora possibile forzare una label:

```markdown
${readeck.annotatedView("Viaggio-Vicoforte")}
```

### API pubblica per altre librerie

```lua
readeck.bookmarks(label, limit)
readeck.bookmark(id)
readeck.create(url, title, labels)
readeck.annotations(id)
readeck.article(id)
```

Le altre funzioni della libreria sono implementative e restano locali al file.

## Configurazione

Aggiungere a `CONFIG`:

```space-lua
config.set("readeck", {
  apiUrl = "http://172.30.250.14:8000",
  webUrl = "https://readeck.example.net",
  tokenPage = "Library/MG/Mio_Viaggio/token",
})
```

La pagina indicata da `tokenPage` deve contenere esclusivamente il token, senza frontmatter né altro testo.

## Convenzione pagina con label Readeck

Una pagina può esporre la label Readeck con:

```yaml
readeck: Viaggio-Vicoforte
```

Tutte le funzioni che non ricevono una label esplicita partono dalla pagina corrente e risalgono progressivamente i parent fino alla prima pagina che contiene un attributo `readeck` non vuoto. Non è richiesto alcun tag specifico.

## Comandi

```text
Readeck: Test
Readeck: Inserisci catalogo
Readeck: Inserisci evidenziati
Readeck: Leggi articolo
```

`Readeck: Inserisci catalogo` inserisce:

```markdown
${readeck.catalogView()}
```

`Readeck: Inserisci evidenziati` inserisce:

```markdown
${readeck.annotatedView()}
```

`Readeck: Leggi articolo` mantiene il flusso pick → preview → `Ok` / `Cancel`; con `Ok` inserisce al cursore il link alla scheda Readeck e il contenuto visualizzato.

## Space Lua

```space-lua
readeck = readeck or {}

local function trim(value)
  if not value then
    return ""
  end

  value = string.gsub(value, "^%s+", "")
  value = string.gsub(value, "%s+$", "")

  return value
end

local function oneLine(value)
  if not value then
    return ""
  end

  value = string.gsub(value, "\r", " ")
  value = string.gsub(value, "\n", " ")
  value = string.gsub(value, "%s+", " ")

  return trim(value)
end

local function urlEncode(value)
  return string.gsub(
    value,
    "([^%w%-_%.~])",
    function(c)
      return string.format("%%%02X", string.byte(c))
    end
  )
end

local function normalizeBaseUrl(value)
  value = trim(value)
  value = string.gsub(value, "/+$", "")
  return value
end

local function getConfig()
  local cfg = config.get("readeck", {})

  if not cfg.apiUrl or trim(cfg.apiUrl) == "" then
    error("Readeck: apiUrl non configurato")
  end

  if not cfg.webUrl or trim(cfg.webUrl) == "" then
    error("Readeck: webUrl non configurato")
  end

  if not cfg.tokenPage or trim(cfg.tokenPage) == "" then
    error("Readeck: tokenPage non configurato")
  end

  cfg.apiUrl = normalizeBaseUrl(cfg.apiUrl)
  cfg.webUrl = normalizeBaseUrl(cfg.webUrl)
  cfg.tokenPage = trim(cfg.tokenPage)

  return cfg
end

local function getToken()
  local cfg = getConfig()

  if not space.pageExists(cfg.tokenPage) then
    error("Readeck: pagina token non trovata: " .. cfg.tokenPage)
  end

  local token = trim(space.readPage(cfg.tokenPage))

  if token == "" then
    error("Readeck: token vuoto")
  end

  return token
end

local function request(path, options)
  local cfg = getConfig()

  options = options or {}
  options.method = options.method or "GET"
  options.headers = options.headers or {}

  options.headers.Authorization =
    "Bearer " .. getToken()

  options.headers.Accept =
    options.headers.Accept or "application/json"

  local response = net.proxyFetch(
    cfg.apiUrl .. path,
    options
  )

  if not response then
    error("Readeck: nessuna risposta")
  end

  if not response.ok then
    error(
      "Readeck HTTP " ..
      tostring(response.status) ..
      ": " ..
      tostring(response.body)
    )
  end

  return response
end

local function labels()
  local response = request(
    "/api/bookmarks/labels"
  )

  return response.body or {}
end

function readeck.bookmarks(label, limit)
  limit = limit or 100

  if limit < 1 then
    limit = 1
  elseif limit > 100 then
    limit = 100
  end

  local path =
    "/api/bookmarks?limit=" .. tostring(limit)

  if label and trim(label) ~= "" then
    path =
      path ..
      "&labels=" ..
      urlEncode(trim(label))
  end

  local response = request(path)

  return response.body or {}
end

function readeck.bookmark(id)
  if not id or trim(id) == "" then
    error("Readeck: bookmark id mancante")
  end

  local response = request(
    "/api/bookmarks/" .. urlEncode(trim(id))
  )

  return response.body
end

function readeck.create(url, title, labels)
  url = trim(url)

  if url == "" then
    error("Readeck: URL mancante")
  end

  local body = {
    url = url,
  }

  title = trim(title)
  if title ~= "" then
    body.title = title
  end

  if labels and #labels > 0 then
    body.labels = labels
  end

  return request(
    "/api/bookmarks",
    {
      method = "POST",
      headers = {
        ["Content-Type"] = "application/json",
      },
      body = body,
    }
  )
end

function readeck.annotations(id)
  if not id or trim(id) == "" then
    error("Readeck: bookmark id mancante")
  end

  local response = request(
    "/api/bookmarks/" ..
    urlEncode(trim(id)) ..
    "/annotations"
  )

  return response.body or {}
end

function readeck.article(id)
  if not id or trim(id) == "" then
    error("Readeck: bookmark id mancante")
  end

  local response = request(
    "/api/bookmarks/" ..
    urlEncode(trim(id)) ..
    "/article.md",
    {
      responseEncoding = "text/markdown",
      headers = {
        Accept = "text/markdown",
      },
    }
  )

  return response.body or ""
end

local function bookmarkWebUrl(id)
  local cfg = getConfig()

  return cfg.webUrl ..
    "/bookmarks/" ..
    tostring(id)
end

local function parentPage(name)
  return string.match(name, "^(.*)/[^/]+$")
end

local function findFrontmatterFieldPage(field)
  field = trim(field)

  if field == "" then
    field = "readeck"
  end

  local pageName = editor.getCurrentPage()

  while pageName and pageName ~= "" do
    if space.pageExists(pageName) then
      local extracted = index.extractFrontmatter(
        space.readPage(pageName)
      )
      local frontmatter = extracted.frontmatter or {}
      local value = frontmatter[field]

      if type(value) == "table" then
        value = value[1]
      end

      if trim(value) ~= "" then
        return pageName, frontmatter
      end
    end

    pageName = parentPage(pageName)
  end

  return nil, nil
end

local function frontmatterLabel(field)
  field = trim(field)

  if field == "" then
    field = "readeck"
  end

  local pageName, frontmatter =
    findFrontmatterFieldPage(field)

  if not pageName then
    error(
      "Readeck: nessuna pagina corrente o antenata con attributo frontmatter '" ..
      field ..
      "'"
    )
  end

  local label = frontmatter[field]

  if type(label) == "table" then
    label = label[1]
  end

  label = trim(label)

  if label == "" then
    error(
      "Readeck: attributo frontmatter '" ..
      field ..
      "' vuoto nella pagina " ..
      pageName
    )
  end

  return label, pageName
end

local function resolveLabel(label)
  label = trim(label)

  if label ~= "" then
    return label
  end

  return frontmatterLabel("readeck")
end

local function catalogMarkdown(label)
  label = resolveLabel(label)

  local bookmarks = readeck.bookmarks(label)
  local result = {}

  if #bookmarks == 0 then
    table.insert(
      result,
      "_Nessun sito raccolto in Readeck con etichetta `" ..
      label ..
      "`._"
    )

    return table.concat(result, "\n")
  end

  for _, bookmark in ipairs(bookmarks) do
    local title =
      bookmark.title or
      bookmark.site_name or
      bookmark.site or
      "Scheda Readeck"

    local readeckUrl =
      bookmarkWebUrl(bookmark.id)

    table.insert(
      result,
      "### [" ..
      oneLine(title) ..
      "](" ..
      readeckUrl ..
      ")"
    )

    if bookmark.site_name and
       bookmark.site_name ~= "" then
      table.insert(
        result,
        "**" ..
        oneLine(bookmark.site_name) ..
        "**"
      )
    elseif bookmark.site and
           bookmark.site ~= "" then
      table.insert(
        result,
        "**" ..
        oneLine(bookmark.site) ..
        "**"
      )
    end

    if bookmark.description and
       bookmark.description ~= "" then
      table.insert(
        result,
        oneLine(bookmark.description)
      )
    end

    if bookmark.url and
       bookmark.url ~= "" then
      table.insert(
        result,
        "[Sito originale](" ..
        bookmark.url ..
        ")"
      )
    end

    table.insert(result, "")
  end

  return table.concat(result, "\n")
end

function readeck.catalogView(label)
  return view.new {
    content = function()
      return catalogMarkdown(label)
    end,
  }
end

local function stripMarkdownImages(text)
  text = text or ""

  -- Immagini Markdown inline: ![alt](url "titolo")
  text = string.gsub(
    text,
    "!%[[^%]]*%]%([^%)]+%)",
    ""
  )

  -- Immagini Markdown reference: ![alt][ref]
  text = string.gsub(
    text,
    "!%[[^%]]*%]%[[^%]]*%]",
    ""
  )

  -- Immagini HTML semplici.
  text = string.gsub(
    text,
    "<img[^>]->",
    ""
  )

  -- Riduce blocchi vuoti prodotti dalla rimozione delle immagini.
  text = string.gsub(text, "\n\n\n+", "\n\n")

  return trim(text)
end

local function articleText(id)
  return stripMarkdownImages(
    readeck.article(id)
  )
end

local function annotationColorName(color)
  color = trim(color)

  if color == "" then
    return ""
  end

  local names = {
    yellow = "giallo",
    red = "rosso",
    blue = "blu",
    green = "verde",
    white = "bianco",
    neutral = "bianco",
  }

  return names[color] or color
end

local function annotationsMarkdown(id)
  local annotations = readeck.annotations(id)

  if #annotations == 0 then
    return nil
  end

  local result = {}

  for _, annotation in ipairs(annotations) do
    local text = trim(annotation.text)
    local note = trim(annotation.note or annotation.comment)
    local color = annotationColorName(annotation.color)

    if note ~= "" then
      local notePrefix = "**Nota:**"

      if color ~= "" then
        notePrefix =
          notePrefix ..
          " (" ..
          color ..
          ")"
      end

      table.insert(
        result,
        notePrefix .. " " .. note
      )
      table.insert(result, "")
    end

    if text ~= "" then
      -- Il testo Readeck viene restituito senza aggiungere
      -- blockquote o indentazioni artificiali.
      table.insert(result, text)
      table.insert(result, "")
    end
  end

  if #result == 0 then
    return nil
  end

  return table.concat(result, "\n")
end

local function contentMarkdown(bookmark)
  if not bookmark or not bookmark.id then
    error("Readeck: bookmark non valido")
  end

  local annotations = annotationsMarkdown(bookmark.id)

  if annotations then
    return annotations, "annotations"
  end

  local article = articleText(bookmark.id)

  if article == "" then
    return nil, nil
  end

  return article, "article"
end

local function readerMarkdown(bookmark)
  if not bookmark or not bookmark.id then
    error("Readeck: bookmark non valido")
  end

  local title =
    bookmark.title or
    bookmark.site_name or
    bookmark.site or
    "Scheda Readeck"

  local content, contentType =
    contentMarkdown(bookmark)

  local result = {
    "# " .. oneLine(title),
    "",
  }

  if contentType == "annotations" then
    table.insert(result, "## Annotazioni")
    table.insert(result, "")
    table.insert(result, content)
  elseif contentType == "article" then
    table.insert(result, "## Articolo")
    table.insert(result, "")
    table.insert(result, content)
  else
    table.insert(
      result,
      "_Nessun contenuto testuale disponibile._"
    )
  end

  return table.concat(result, "\n")
end

local function insertMarkdown(bookmark)
  local content, contentType =
    contentMarkdown(bookmark)

  if not content then
    editor.flashNotification(
      "Readeck: nessun contenuto testuale disponibile"
    )
    return false
  end

  local source =
    "[Fonte Readeck](" ..
    bookmarkWebUrl(bookmark.id) ..
    ")"

  editor.insertAtCursor(
    source .. "\n\n" .. content .. "\n",
    false,
    true
  )

  if contentType == "annotations" then
    editor.flashNotification(
      "Readeck: fonte e annotazioni inserite nella pagina"
    )
  else
    editor.flashNotification(
      "Readeck: fonte e articolo inseriti nella pagina"
    )
  end

  return true
end

local function annotatedMarkdown(label)
  label = resolveLabel(label)

  local bookmarks = readeck.bookmarks(label)
  local result = {}
  local count = 0

  for _, bookmark in ipairs(bookmarks) do
    local annotated = annotationsMarkdown(bookmark.id)

    if annotated then
      count = count + 1

      local title =
        bookmark.title or
        bookmark.site_name or
        bookmark.site or
        "Scheda Readeck"

      table.insert(
        result,
        "### [" ..
        oneLine(title) ..
        "](" ..
        bookmarkWebUrl(bookmark.id) ..
        ")"
      )

      table.insert(result, "")
      table.insert(result, annotated)
    end
  end

  if count == 0 then
    return
      "_Nessuna evidenziazione Readeck per l'etichetta `" ..
      label ..
      "`._"
  end

  return table.concat(result, "\n")
end

function readeck.annotatedView(label)
  return view.new {
    content = function()
      return annotatedMarkdown(label)
    end,
  }
end

local function pickBookmarkByFrontmatter(field)
  local label = frontmatterLabel(field)
  local bookmarks = readeck.bookmarks(label)

  if #bookmarks == 0 then
    editor.flashNotification(
      "Readeck: nessun articolo con etichetta " .. label
    )
    return nil
  end

  return view.pick {
    title = "Scegli articolo Readeck",
    source = function()
      return bookmarks
    end,
    presentation = {
      mode = "list",
      row = {
        primary = function(bookmark)
          return
            bookmark.title or
            bookmark.site_name or
            bookmark.site or
            "Scheda Readeck"
        end,
        description = function(bookmark)
          local description = oneLine(bookmark.description)

          if description == "" then
            description =
              bookmark.site_name or
              bookmark.site or
              bookmark.url or
              ""
          end

          if description == "" then
            return nil
          end

          return { text = description }
        end,
      },
    },
  }
end

local function openArticleFromFrontmatter(field)
  local bookmark = pickBookmarkByFrontmatter(field)

  if not bookmark then
    return
  end

  local title =
    bookmark.title or
    bookmark.site_name or
    bookmark.site or
    "Articolo Readeck"

  local preview = readerMarkdown(bookmark)

  view.define {
    name = "readeck.articleReader",
    title = oneLine(title),
    dock = "modal",
    supportedDocks = { "modal", "rhs", "bhs" },
    content = function()
      return preview
    end,
  }

  view.open("readeck.articleReader")

  if not editor.confirm(
    "Inserire nella pagina il contenuto visualizzato?"
  ) then
    return
  end

  insertMarkdown(bookmark)
end

command.define {
  name = "Readeck: Test",

  run = function()
    local labels = labels()

    editor.flashNotification(
      "Readeck OK - " ..
      tostring(#labels) ..
      " label disponibili"
    )
  end,
}

command.define {
  name = "Readeck: Inserisci catalogo",

  run = function()
    frontmatterLabel("readeck")

    editor.insertAtCursor(
      "${readeck.catalogView()}",
      false,
      true
    )
  end,
}

command.define {
  name = "Readeck: Inserisci evidenziati",

  run = function()
    frontmatterLabel("readeck")

    editor.insertAtCursor(
      "${readeck.annotatedView()}",
      false,
      true
    )
  end,
}

command.define {
  name = "Readeck: Leggi articolo",

  run = function()
    openArticleFromFrontmatter("readeck")
  end,
}
```



## Modifiche versione 0.08

- Libreria rinominata **Appunti Readeck**.
- Eliminata la dipendenza logica dal tag `viaggio`: la label viene ricercata nella prima pagina corrente/antenata che contiene un attributo frontmatter `readeck` non vuoto.
- `readeck.catalogView()` e `readeck.annotatedView()` mantengono il parametro opzionale per forzare una label esplicita.
- Le evidenziazioni non vengono più renderizzate come blockquote.
- La nota precede il testo evidenziato.
- Quando Readeck restituisce `color`, il colore viene mostrato dopo `Nota:` con traduzione italiana per giallo, rosso, blu, verde e bianco.
- `Readeck: Leggi articolo`, `Readeck: Inserisci catalogo` e `Readeck: Inserisci evidenziati` usano lo stesso meccanismo di risalita basato sull'attributo `readeck`.

## API esposta

La libreria espone intenzionalmente soltanto:

```lua
readeck.catalogView()
readeck.annotatedView()
readeck.bookmarks()
readeck.bookmark()
readeck.create()
readeck.annotations()
readeck.article()
```

Le funzioni di configurazione, token, request HTTP, ricerca della pagina corrente/antenata con attributo `readeck`, rendering Markdown, pick, preview e inserimento sono `local`.

## Nota sulle note associate alle evidenziazioni

Nell'API Readeck verificata, `annotationInfo` espone ufficialmente il testo evidenziato (`text`) e i dati necessari a localizzarlo nell'articolo, ma non documenta un campo separato per una nota testuale dell'utente.

La libreria legge gli eventuali campi `note` o `comment` e il campo `color` restituiti da Readeck. `readeck.annotatedView()` e la preview mostrano `Nota:` prima dell'evidenziazione e, quando disponibile, il colore tradotto in italiano tra parentesi. Il testo evidenziato non viene trasformato in blockquote.
