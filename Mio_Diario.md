---
name: "Library/MG/Mio_Diario"
tags: meta/library
description: "Libreria consolidata Markdown-first per Diario, Luoghi, Viaggi, Indice, Mappe/GPX, PhotoGallery, Riepiloghi e configurazione delle nuove pagine."
version: "1.4-00"
versionDate: 2026-09-14
pageDecoration.prefix: "📔 "
share.uri: "github:marco10x15/silverbullet-libraries/Mio_Diario.md"
files:
  - Mio_Diario/Page Templates/Diario.md
  - Mio_Diario/Page Templates/Luogo.md
---

# 📔 Mio Diario

**Versione consolidata:** 1.4-00 — 14.09.2026

Questa pagina è la sola libreria `Mio_Diario` da installare. Riunisce le implementazioni definitive contenute nelle precedenti librerie `Mio_Diario_*` senza cambiare il modello dati Markdown.

## Invarianti

- le pagine Markdown restano la fonte primaria dei dati;
- `Diario/`, `luoghi/` e `Viaggi/` mantengono struttura e frontmatter già adottati;
- le relazioni geografiche derivano prioritariamente dai path `luoghi/...`;
- `divisioneIntermedia` conserva il codice ISO 3166-2 quando il livello amministrativo non è materializzato nel path;
- le viste, mappe, GPX, PhotoGallery e riepiloghi sono dati derivati e non richiedono persistenza aggiuntiva;
- non vengono introdotte modifiche automatiche alle note oltre ai comandi espliciti di creazione/verifica Luogo già presenti.

## Componenti consolidate

| Componente precedente | Stato in 1.4-00 | Destinazione |
|---|---|---|
| `Mio_Diario.md` | incorporata | questa libreria |
| `Mio_Diario_Configurazioni.md` | incorporata | questa libreria |
| `Mio_Diario_Indice.md` | incorporata | questa libreria |
| `Mio_Diario_LuoghiMappa.md` | incorporata | questa libreria |
| `Mio_Diario_Photo_Gallery.md` | incorporata | questa libreria |
| `Mio_Diario_Riepiloghi.md` | incorporata | questa libreria |
| `Mio_Diario_Configurazioni/Page Templates/Diario` | spostata | `Library/MG/Mio_Diario/Page Templates/Diario` |
| `Mio_Diario_Configurazioni/Page Templates/Luogo` | spostata | `Library/MG/Mio_Diario/Page Templates/Luogo` |

Dopo il collaudo della versione consolidata le vecchie librerie `Mio_Diario_*` non devono restare attive nello Space: definirebbero nuovamente comandi, listener, widget e Virtual Page già presenti qui.

## Dipendenze definitive

### Interna personale obbligatoria

- `Library/MG/DateFormat`: fornisce `date.format()` ed è volutamente riutilizzata anziché duplicata.

`Library/MG/PageNavigation` non è più una dipendenza di Mio Diario: l'unico uso precedente, `page.nome()`, è sostituito dall'equivalente già interno `luoghi.basename()`.

### Esterne per funzionalità specifiche

- dataset ISO 3166-2 configurato in `luoghi.iso3166_2.url`; in caso di indisponibilità resta il fallback al codice ISO;
- Wikipedia/Wikidata tramite `net.proxyFetch` per creazione e verifica esplicita delle pagine Luogo;
- Leaflet 1.9.4 e tile OpenStreetMap nel sandbox delle mappe;
- servizio PhotoGallery WebUI/API configurato in `photoGalleryConfig`;
- documenti GPX presenti nello Space e indicizzati da SilverBullet.

L'assenza di una dipendenza accessoria deve degradare soltanto la funzione interessata; non modifica il Markdown del Diario.

## Configurazione PhotoGallery canonica

`photoGalleryConfig.webUiBase` è la sorgente unica dell'URL full-page usato sia dalle API PhotoGallery sia dai link 📷 del Diario. Gli endpoint API restano `photoGalleryConfig.apiBase` e `photoGalleryConfig.apiBatch`.

## Page Template

I due Page Template fanno parte della componente consolidata e devono essere installati con lo stesso numero di versione:

- `Library/MG/Mio_Diario/Page Templates/Diario`
- `Library/MG/Mio_Diario/Page Templates/Luogo`

La configurazione attuale implementa la creazione di **Diario** e **Luogo**. Non viene aggiunto un comando/template Viaggio perché non era presente nell'implementazione allegata.

## Implementazione — Core Diario / Luoghi / Viaggi

```space-lua
luoghi = luoghi or {}
diario = diario or {}
widgets = widgets or {}



-- ============================================================
-- FUNZIONI GENERALI
-- ============================================================

function luoghi.hasString(value)
  return type(value) == "string"
    and value ~= ""
end


function luoghi.hasList(value)
  return type(value) == "table"
    and #value > 0
end


function luoghi.basename(pageName)
  return string.match(
    pageName,
    "([^/]+)$"
  ) or pageName
end


function luoghi.parentName(pageName)
  return string.match(
    pageName,
    "^(.*)/[^/]+$"
  )
end


function luoghi.isoDate(value)
  return string.match(
    tostring(value or ""),
    "(%d%d%d%d%-%d%d%-%d%d)"
  )
end


function luoghi.wikilinkTarget(link)
  if not luoghi.hasString(link) then
    return nil
  end

  return string.match(
    link,
    "^%[%[([^]|]+)"
  )
end


function luoghi.wikilinkLabel(link)
  if not luoghi.hasString(link) then
    return nil
  end

  local target, label = string.match(
    link,
    "^%[%[([^]|]+)|([^]]+)%]%]$"
  )

  if target then
    return label
  end

  target = luoghi.wikilinkTarget(link)
  return target and luoghi.basename(target) or nil
end


function luoghi.label(p, fallback)
  if p then
    if luoghi.hasString(p.displayName) then
      return p.displayName
    end

    if luoghi.hasString(p.title) then
      return p.title
    end

    if luoghi.hasList(p.aliases)
      and luoghi.hasString(p.aliases[1])
    then
      return p.aliases[1]
    end
  end

  return fallback
end


function luoghi.pageLink(p)
  return string.format(
    "[[%s|%s]]",
    p.name,
    luoghi.label(
      p,
      luoghi.basename(p.name)
    )
  )
end


function luoghi.mapLink(pageName)
  return string.format(
    "[[Mappa/%s|🗺️]]",
    pageName
  )
end


function luoghi.getPage(pageName)
  local pages = query[[
    from p = index.pages()
    where p.name == pageName
    select {
      name = p.name,
      title = p.title,
      displayName = p.displayName,
      aliases = p.aliases,
      continente = p.continente,
      tipoAmministrativo = p.tipoAmministrativo,
      divisioneIntermedia = p.divisioneIntermedia,
      wikipedia = p.wikipedia,
      coordinate = p.coordinate,
      tags = p.tags,
      luoghi = p.luoghi
    }
    limit 1
  ]]

  if pages and #pages > 0 then
    return pages[1]
  end

  return nil
end


function luoghi.children(pageName)
  return query[[
    from p = index.subPages(pageName)
    where not string.find(
      string.sub(
        p.name,
        #pageName + 2
      ),
      "/"
    )
    order by p.name
    select {
      name = p.name,
      title = p.title,
      displayName = p.displayName,
      aliases = p.aliases,
      continente = p.continente,
      tipoAmministrativo = p.tipoAmministrativo,
      divisioneIntermedia = p.divisioneIntermedia,
      wikipedia = p.wikipedia,
      coordinate = p.coordinate,
      tags = p.tags
    }
  ]]
end


function luoghi.resolvePages(values)
  if values == nil then
    return {}
  end

  if type(values) ~= "table" then
    values = {values}
  end

  local names = {}
  local seen = {}
  local supplied = {}

  for _, value in ipairs(values) do
    local name = nil

    if type(value) == "table"
      and luoghi.hasString(value.name)
    then
      name = value.name
      supplied[name] = value
    else
      name =
        luoghi.wikilinkTarget(value)
        or tostring(value or "")
    end

    if name ~= ""
      and not seen[name]
    then
      table.insert(names, name)
      seen[name] = true
    end
  end

  if #names == 0 then
    return {}
  end

  local missing = {}

  for _, name in ipairs(names) do
    if not supplied[name] then
      table.insert(missing, name)
    end
  end

  if #missing > 0 then
    local rows = query[[
      from p = index.pages()
      where table.includes(
        missing,
        p.name
      )
      select {
        name = p.name,
        title = p.title,
        displayName = p.displayName,
        aliases = p.aliases,
        continente = p.continente,
        tipoAmministrativo = p.tipoAmministrativo,
        divisioneIntermedia = p.divisioneIntermedia,
        wikipedia = p.wikipedia,
        coordinate = p.coordinate,
        tags = p.tags
      }
    ]]

    for _, row in ipairs(rows or {}) do
      supplied[row.name] = row
    end
  end

  local result = {}

  for _, name in ipairs(names) do
    if supplied[name] then
      table.insert(
        result,
        supplied[name]
      )
    end
  end

  return result
end


function luoghi.contieneLuogo(
  valori,
  pageName
)
  if not luoghi.hasList(valori)
    or not luoghi.hasString(pageName)
  then
    return false
  end

  local prefix =
    pageName .. "/"

  for _, value in ipairs(valori) do
    if luoghi.hasString(value) then
      local target =
        luoghi.wikilinkTarget(value)
        or value

      if target == pageName
        or string.startsWith(
          target,
          prefix
        )
      then
        return true
      end
    end
  end

  return false
end


function luoghi.diarioLabel(info)
  if luoghi.hasString(
    info.displayName
  ) then
    return info.displayName
  end

  return luoghi.basename(info.page)
end


-- ============================================================
-- RADICE LUOGHI
-- ============================================================

function luoghi.renderRoot()
  local children =
    luoghi.children("luoghi")

  if not children
    or #children == 0
  then
    return ""
  end

  local ordineContinenti = {
    "Africa",
    "Asia",
    "Europa",
    "Nord America",
    "Sud America",
    "Oceania",
    "Antartide"
  }

  local gruppi = {}

  for _, continente in ipairs(
    ordineContinenti
  ) do
    gruppi[continente] = {}
  end

  local nonClassificati = {}

  for _, p in ipairs(children) do
    local item = {
      label = luoghi.label(
        p,
        luoghi.basename(p.name)
      ),
      link = luoghi.pageLink(p)
    }

    if luoghi.hasString(p.continente)
      and gruppi[p.continente]
    then
      table.insert(
        gruppi[p.continente],
        item
      )
    else
      table.insert(
        nonClassificati,
        item
      )
    end
  end

  local function ordina(lista)
    table.sort(
      lista,
      function(a, b)
        return a.label < b.label
      end
    )
  end

  local text =
    "## Paesi visitati\n"

  for _, continente in ipairs(
    ordineContinenti
  ) do
    local stati =
      gruppi[continente]

    if #stati > 0 then
      ordina(stati)

      local links = {}

      for _, stato in ipairs(stati) do
        table.insert(
          links,
          stato.link
        )
      end

      text = text
        .. "\n### "
        .. continente
        .. "\n\n"
        .. table.concat(links, ", ")
        .. "\n"
    end
  end

  if #nonClassificati > 0 then
    ordina(nonClassificati)

    local links = {}

    for _, stato in ipairs(
      nonClassificati
    ) do
      table.insert(
        links,
        stato.link
      )
    end

    text = text
      .. "\n### Da verificare\n\n"
      .. table.concat(links, ", ")
      .. "\n"
  end

  return text
end


-- ============================================================
-- ISO 3166-2
-- ============================================================

function luoghi.isoIndex()
  if luoghi._isoAttempted then
    return luoghi._isoByCode
  end

  luoghi._isoAttempted = true

  local url = config.get(
    "luoghi.iso3166_2.url",
    "https://salsa.debian.org/iso-codes-team/iso-codes/-/raw/main/data/iso_3166-2.json"
  )

  local response =
    net.proxyFetch(
      url,
      {
        headers = {
          Accept = "application/json"
        }
      }
    )

  if not response
    or not response.ok
  then
    return nil
  end

  local body = response.body

  if type(body) == "string" then
    body = js.tolua(
      js.window.JSON.parse(body)
    )
  end

  if type(body) ~= "table"
    or not body["3166-2"]
  then
    return nil
  end

  local byCode = {}

  for _, item in ipairs(
    body["3166-2"]
  ) do
    if item.code then
      byCode[item.code] = item
    end
  end

  luoghi._isoByCode = byCode

  return byCode
end


function luoghi.isoDivisione(code)
  local indexIso =
    luoghi.isoIndex()

  if not indexIso then
    return nil
  end

  return indexIso[code]
end


function luoghi.divisioneLabel(code)
  local divisione =
    luoghi.isoDivisione(code)

  if not divisione then
    return code
  end

  local prefissi = {
    ["Province"] = "Provincia di ",
    ["Metropolitan city"] = "Città metropolitana di ",
    ["Autonomous province"] = "Provincia autonoma di ",
    ["Free municipal consortium"] = "Libero consorzio comunale di ",
    ["Decentralized regional entity"] = "Ente di decentramento regionale di ",
    ["Department"] = "Dipartimento di ",
    ["Metropolitan department"] = "Dipartimento di ",
    ["County"] = "Contea di ",
    ["District"] = "Distretto di ",
    ["Canton"] = "Cantone di "
  }

  local prefisso =
    prefissi[divisione.type]

  if prefisso then
    return prefisso
      .. divisione.name
  end

  return divisione.name
    or code
end


function luoghi.divisioneLink(code)
  return string.format(
    "[[geo:divisione:%s|%s]]",
    code,
    luoghi.divisioneLabel(code)
  )
end


function luoghi.renderDivisione(code)
  local divisione =
    luoghi.isoDivisione(code)

  local label =
    luoghi.divisioneLabel(code)

  local pages = query[[
    from p = index.subPages("luoghi")
    where p.divisioneIntermedia == code
    order by p.name
    select {
      name = p.name,
      title = p.title,
      displayName = p.displayName,
      aliases = p.aliases
    }
  ]]

  local text =
    "# "
    .. label
    .. "\n\n`"
    .. code
    .. "`"

  if divisione
    and luoghi.hasString(
      divisione.type
    )
  then
    text = text
      .. " — "
      .. divisione.type
  end

  local parentNames = {}
  local parentSeen = {}

  for _, p in ipairs(pages or {}) do
    local parentName =
      luoghi.parentName(p.name)

    if parentName
      and not parentSeen[parentName]
    then
      table.insert(
        parentNames,
        parentName
      )

      parentSeen[parentName] = true
    end
  end

  local parentsByName = {}

  if #parentNames > 0 then
    local parents = query[[
      from p = index.subPages("luoghi")
      where table.includes(
        parentNames,
        p.name
      )
      select {
        name = p.name,
        title = p.title,
        displayName = p.displayName,
        aliases = p.aliases
      }
    ]]

    for _, parent in ipairs(
      parents or {}
    ) do
      parentsByName[parent.name] =
        parent
    end
  end

  local parentLinks = {}

  for _, parentName in ipairs(
    parentNames
  ) do
    local parent =
      parentsByName[parentName]

    if parent then
      table.insert(
        parentLinks,
        luoghi.pageLink(parent)
      )
    else
      table.insert(
        parentLinks,
        string.format(
          "[[%s|%s]]",
          parentName,
          luoghi.basename(parentName)
        )
      )
    end
  end

  if #parentLinks > 0 then
    text = text
      .. "\n\nAppartiene a "
      .. table.concat(
        parentLinks,
        ", "
      )
      .. "."
  end

  if pages and #pages > 0 then
    local links = {}

    for _, p in ipairs(pages) do
      table.insert(
        links,
        luoghi.pageLink(p)
      )
    end

    text = text
      .. "\n\nIn cui ci sono "
      .. #pages
      .. " luoghi: "
      .. table.concat(links, ", ")
  else
    text = text
      .. "\n\nNessun luogo dello Space utilizza questa divisione."
  end

  return text .. "\n"
end


virtualPage.define {
  pattern = "geo:divisione:(.+)",

  run = function(code)
    return luoghi.renderDivisione(code)
  end
}


-- ============================================================
-- PRESENTAZIONE PAGINE LUOGO
-- ============================================================

function luoghi.renderTipo(p)
  local elementi = {}

  if luoghi.hasString(
    p.tipoAmministrativo
  ) then
    table.insert(
      elementi,
      "`"
        .. p.tipoAmministrativo
        .. "`"
    )
  end

  if luoghi.hasString(p.wikipedia) then
    table.insert(
      elementi,
      "[wikipedia]("
        .. p.wikipedia
        .. ")"
    )
  end

  if luoghi.hasList(p.tags) then
    for _, tag in ipairs(p.tags) do
      if luoghi.hasString(tag) then
        table.insert(
          elementi,
          "#" .. tag
        )
      end
    end
  elseif luoghi.hasString(p.tags) then
    table.insert(
      elementi,
      "#" .. p.tags
    )
  end

  return table.concat(
    elementi,
    " "
  )
end


function luoghi.renderStato(p)
  local parts = {}

  if luoghi.hasString(p.continente) then
    table.insert(
      parts,
      "Si trova in **"
        .. p.continente
        .. "**."
    )
  end

  local children =
    luoghi.children(p.name)

  if children and #children > 0 then
    local links = {}

    for _, child in ipairs(children) do
      table.insert(
        links,
        luoghi.pageLink(child)
      )
    end

    table.insert(
      parts,
      luoghi.mapLink(p.name)
        .. " In cui ci sono "
        .. #children
        .. " suddivisioni: "
        .. table.concat(links, ", ")
    )
  end

  return table.concat(parts, "\n\n")
end


function luoghi.renderSuddivisione(p)
  local parts = {}

  local children =
    luoghi.children(p.name)

  local divisioni = {}
  local divisioniSeen = {}
  local altriLuoghi = {}

  for _, child in ipairs(children or {}) do
    if luoghi.hasString(
      child.divisioneIntermedia
    )
    then
      local code =
        child.divisioneIntermedia

      if not divisioniSeen[code] then
        table.insert(
          divisioni,
          code
        )

        divisioniSeen[code] = true
      end
    else
      table.insert(
        altriLuoghi,
        child
      )
    end
  end

  table.sort(divisioni)

  local mapPrefix =
    luoghi.mapLink(p.name)
      .. " "

  if #divisioni > 0 then
    local links = {}

    for _, code in ipairs(divisioni) do
      table.insert(
        links,
        luoghi.divisioneLink(code)
      )
    end

    table.insert(
      parts,
      mapPrefix
        .. "In cui sono rappresentate "
        .. #divisioni
        .. " divisioni intermedie: "
        .. table.concat(links, ", ")
    )

    mapPrefix = ""
  end

  if #altriLuoghi > 0 then
    local links = {}

    for _, child in ipairs(
      altriLuoghi
    ) do
      table.insert(
        links,
        luoghi.pageLink(child)
      )
    end

    local label =
      #divisioni > 0
      and "Altri luoghi direttamente nella suddivisione: "
      or "In cui ci sono "
        .. #altriLuoghi
        .. " luoghi: "

    table.insert(
      parts,
      mapPrefix
        .. label
        .. table.concat(links, ", ")
    )
  end

  return table.concat(parts, "\n\n")
end


function luoghi.renderLuogo(p)
  local parts = {}

  local parentName =
    luoghi.parentName(p.name)

  local parent = nil

  if parentName then
    parent =
      luoghi.getPage(parentName)
  end

  if parent then
    table.insert(
      parts,
      luoghi.label(
        p,
        luoghi.basename(p.name)
      )
        .. " si trova in "
        .. luoghi.pageLink(parent)
    )
  end

  local children =
    luoghi.children(p.name)

  if children and #children > 0 then
    local links = {}

    for _, child in ipairs(children) do
      table.insert(
        links,
        luoghi.pageLink(child)
      )
    end

    table.insert(
      parts,
      luoghi.mapLink(p.name)
        .. " In cui ci sono "
        .. #children
        .. " luoghi: "
        .. table.concat(links, ", ")
    )
  end

  return table.concat(parts, "\n\n")
end


function luoghi.breadcrumb(pageName)
  local parts =
    string.split(pageName, "/")

  local links = {
    "[[luoghi|🌍]]"
  }

  local paths = {}
  local path = "luoghi"

  for i = 2, #parts - 1 do
    path =
      path .. "/" .. parts[i]
    table.insert(paths, path)
  end

  local byName = {}

  for _, p in ipairs(
    luoghi.resolvePages(paths)
  ) do
    byName[p.name] = p
  end

  for i, ancestorPath in ipairs(paths) do
    local p = byName[ancestorPath]

    if p then
      table.insert(
        links,
        luoghi.pageLink(p)
      )
    else
      table.insert(
        links,
        string.format(
          "[[%s|%s]]",
          ancestorPath,
          parts[i + 1]
        )
      )
    end
  end

  return table.concat(links, "|")
end


function luoghi.withBreadcrumb(
  pageName,
  text
)
  local trail =
    luoghi.breadcrumb(pageName)

  if not luoghi.hasString(text) then
    return trail
  end

  return trail
    .. "\n\n"
    .. text
end


function widgets.linkedInfoLuoghi(pageName)
  pageName =
    pageName
    or editor.getCurrentPage()

  if pageName == "luoghi" then
    return luoghi.renderRoot()
  end

  if not string.startsWith(
    pageName,
    "luoghi/"
  ) then
    return ""
  end

  local p =
    luoghi.getPage(pageName)

  if not p then
    return ""
  end

  local parts =
    string.split(pageName, "/")

  local text = nil

  if #parts == 2 then
    text = luoghi.renderStato(p)
  elseif #parts == 3 then
    text = luoghi.renderSuddivisione(p)
  else
    text = luoghi.renderLuogo(p)
  end

  return luoghi.withBreadcrumb(
    pageName,
    text
  )
end


-- ============================================================
-- MEDIA DELLE GIORNATE
-- ============================================================

function widgets.diarioMediaCatalog(
  diarioInfo
)
  local catalog = {
    gpxByDate = {}
  }

  if type(gpxCatalogByDate) == "function" then
    catalog.gpxByDate =
      gpxCatalogByDate()
      or {}
  end

  return catalog
end


function widgets.mediaLinksForDate(
  value,
  catalog
)
  local iso =
    luoghi.isoDate(value)

  if not iso then
    return ""
  end

  local links = {}

  local hasGpx = false

  if catalog
    and catalog.gpxByDate
  then
    hasGpx =
      catalog.gpxByDate[iso]
      ~= nil
  elseif type(gpxPathForDate) == "function" then
    hasGpx =
      gpxPathForDate(iso)
      ~= nil
  end

  if hasGpx then
    table.insert(
      links,
      "[[GPX/"
        .. iso
        .. "|🧭]]"
    )
  end

  local photoUrl =
    photoGalleryFullUrl(iso)

  if photoUrl then
    table.insert(
      links,
      "[📷]("
        .. photoUrl
        .. ")"
    )
  end

  return table.concat(links, " ")
end


function widgets.diarioRow(
  info,
  options
)
  options = options or {}

  local giorno =
    date.format(info.date)

  local titolo =
    luoghi.diarioLabel(info)

  local dateLink =
    string.format(
      "[[%s|%s]]",
      info.page,
      giorno
    )

  if options.boldDate then
    dateLink =
      "**" .. dateLink .. "**"
  end

  local row =
    dateLink .. " " .. titolo

  local media =
    widgets.mediaLinksForDate(
      info.date,
      options.catalog
    )

  if media ~= "" then
    row = row
      .. " "
      .. media
  end

  return row
end


-- ============================================================
-- SIAMO STATI QUI
-- ============================================================

function luoghi.diarioPerLuogo(pageName)
  if not luoghi.hasString(pageName) then
    return {}
  end

  return query[[
    from p = index.subPages("Diario")
    where p.date
      and p.luoghi
      and luoghi.contieneLuogo(
        p.luoghi,
        pageName
      )
    order by p.date desc, p.name
    select {
      page = p.name,
      date = p.date,
      displayName = p.displayName,
      Viaggio = p.Viaggio
    }
  ]]
end


function luoghi.viaggioLink(viaggio)
  local target =
    luoghi.wikilinkTarget(viaggio)

  if not target then
    return nil
  end

  return string.format(
    "[[%s|%s]]",
    target,
    luoghi.basename(target)
  )
end


function luoghi.renderViaggiPerLuogo(
  pageName,
  diarioInfo
)
  diarioInfo =
    diarioInfo
    or luoghi.diarioPerLuogo(
      pageName
    )

  if not diarioInfo
    or #diarioInfo == 0
  then
    return ""
  end

  local viaggi = {}
  local seen = {}

  for _, info in ipairs(diarioInfo) do
    local target =
      luoghi.wikilinkTarget(
        info.Viaggio
      )

    if target
      and not seen[target]
    then
      local link =
        luoghi.viaggioLink(
          info.Viaggio
        )

      if link then
        table.insert(viaggi, link)
      end

      seen[target] = true
    end
  end

  if #viaggi == 0 then
    return ""
  end

  return "## 🧳 In questi viaggi\n\n"
    .. table.concat(viaggi, "\n")
end


function luoghi.renderGiorniPerLuogo(
  pageName,
  diarioInfo,
  catalog
)
  diarioInfo =
    diarioInfo
    or luoghi.diarioPerLuogo(
      pageName
    )

  if not diarioInfo
    or #diarioInfo == 0
  then
    return ""
  end

  catalog =
    catalog
    or widgets.diarioMediaCatalog(
      diarioInfo
    )

  local giorni = {}

  for _, info in ipairs(diarioInfo) do
    table.insert(
      giorni,
      widgets.diarioRow(
        info,
        {
          boldDate = true,
          catalog = catalog
        }
      )
    )
  end

  return "## In questi giorni\n\n"
    .. table.concat(giorni, "\n")
end


function luoghi.siamoStatiQuiEnabled(
  pageName
)
  if not string.startsWith(
    pageName,
    "luoghi/"
  ) then
    return false
  end

  local parts =
    string.split(pageName, "/")

  return #parts >= 2
end


function widgets.siamoStatiQuiViaggi(
  pageName,
  diarioInfo
)
  pageName =
    pageName
    or editor.getCurrentPage()

  if not luoghi.siamoStatiQuiEnabled(
    pageName
  ) then
    return ""
  end

  return luoghi.renderViaggiPerLuogo(
    pageName,
    diarioInfo
  )
end


function widgets.siamoStatiQuiGiorni(
  pageName,
  diarioInfo,
  catalog
)
  pageName =
    pageName
    or editor.getCurrentPage()

  if not luoghi.siamoStatiQuiEnabled(
    pageName
  ) then
    return ""
  end

  return luoghi.renderGiorniPerLuogo(
    pageName,
    diarioInfo,
    catalog
  )
end


-- ============================================================
-- VIAGGI
-- ============================================================

function luoghi.contieneViaggio(
  viaggio,
  pageName
)
  if not viaggio then
    return false
  end

  local target =
    luoghi.wikilinkTarget(viaggio)

  return target == pageName
end


function viaggiDiarioInfo(pageName)
  if not string.startsWith(
    pageName,
    "Viaggi/"
  ) then
    return {}
  end

  return query[[
    from p = index.subPages("Diario")
    where p.date
      and p.Viaggio
      and luoghi.contieneViaggio(
        p.Viaggio,
        pageName
      )
    order by p.date, p.name
    select {
      page = p.name,
      date = p.date,
      displayName = p.displayName,
      luoghi = p.luoghi
    }
  ]]
end


function widgets.infoViaggioLuoghiList(
  pageName,
  diarioInfo
)
  diarioInfo =
    diarioInfo
    or viaggiDiarioInfo(pageName)

  if not diarioInfo
    or #diarioInfo == 0
  then
    return {}
  end

  local values = {}
  local seen = {}

  for _, info in ipairs(diarioInfo) do
    if luoghi.hasList(info.luoghi) then
      for _, value in ipairs(info.luoghi) do
        local target =
          luoghi.wikilinkTarget(value)
          or value

        if not seen[target] then
          table.insert(values, value)
          seen[target] = true
        end
      end
    end
  end

  return values
end


function widgets.infoViaggioLuoghi(
  pageName,
  diarioInfo
)
  local values =
    widgets.infoViaggioLuoghiList(
      pageName,
      diarioInfo
    )

  local text =
    "**Luoghi visitati** "
      .. luoghi.mapLink(pageName)

  if #values > 0 then
    text = text
      .. "\n\n"
      .. table.concat(values, ", ")
  end

  return text
end


function widgets.infoViaggioDiario(
  pageName,
  diarioInfo,
  catalog
)
  diarioInfo =
    diarioInfo
    or viaggiDiarioInfo(pageName)

  if not diarioInfo
    or #diarioInfo == 0
  then
    return ""
  end

  catalog =
    catalog
    or widgets.diarioMediaCatalog(
      diarioInfo
    )

  local giorni = {}

  for _, info in ipairs(diarioInfo) do
    table.insert(
      giorni,
      widgets.diarioRow(
        info,
        {
          catalog = catalog
        }
      )
    )
  end

  return "## "
    .. #diarioInfo
    .. " pagine del diario per questo viaggio\n\n"
    .. table.concat(giorni, "\n")
    .. "\n"
end


-- ============================================================
-- TOP WIDGET LUOGO
-- ============================================================

function widgets.topLuogo(pageName)
  pageName =
    pageName
    or editor.getCurrentPage()

  if not string.startsWith(
    pageName,
    "luoghi/"
  ) then
    return ""
  end

  local p =
    luoghi.getPage(pageName)

  if not p then
    return ""
  end

  local titolo =
    luoghi.hasString(p.displayName)
    and p.displayName
    or luoghi.basename(pageName)

  local tipo =
    luoghi.renderTipo(p)

  if tipo ~= "" then
    return "# "
      .. titolo
      .. "\n"
      .. tipo
      .. "\n\n---"
  end

  return "# "
    .. titolo
    .. "\n\n---"
end


-- ============================================================
-- NAVIGAZIONE DIARIO SPECIALIZZATA
-- ============================================================

-- Il Diario consolidato è piatto sotto Diario/.
-- Mantiene due query mirate con limit 1, senza il filtro
-- generico sulla profondità usato da PageNavigation.
function diario.prec(path)
  path = path or editor.getCurrentPage()

  local pages = query[[
    from p = index.subPages("Diario")
    where p.name < path
    order by p.name desc
    select {
      name = p.name
    }
    limit 1
  ]]

  if not pages or #pages == 0 then
    return ""
  end

  return pages[1].name
end


function diario.succ(path)
  path = path or editor.getCurrentPage()

  local pages = query[[
    from p = index.subPages("Diario")
    where p.name > path
    order by p.name
    select {
      name = p.name
    }
    limit 1
  ]]

  if not pages or #pages == 0 then
    return ""
  end

  return pages[1].name
end


-- Media del Top Diario: GPX verificato nell'indice locale;
-- PhotoGallery è un link deterministico e non esegue preflight HTTP.
function widgets.topMediaLinksForDate(value)
  local iso = luoghi.isoDate(value)

  if not iso then
    return ""
  end

  local links = {}

  if type(gpxPathForDate) == "function"
    and gpxPathForDate(iso) ~= nil
  then
    table.insert(
      links,
      "[[GPX/" .. iso .. "|🧭]]"
    )
  end

  local photoUrl =
    photoGalleryFullUrl(iso)

  if photoUrl then
    table.insert(
      links,
      "[📷]("
        .. photoUrl
        .. ")"
    )
  end

  return table.concat(links, " ")
end


-- ============================================================
-- TOP WIDGET DIARIO
-- ============================================================

function wTopDiario(path)
  local pageName =
    path
    or editor.getCurrentPage()

  if not string.startsWith(
    pageName,
    "Diario/"
  ) then
    return
  end

  local extracted =
    index.extractFrontmatter(
      editor.getText()
    )

  local p =
    extracted
    and extracted.frontmatter
    or {}

  local titolo =
    luoghi.hasString(p.displayName)
    and p.displayName
    or luoghi.basename(pageName)

  local previousPage =
    diario.prec(pageName)

  local nextPage =
    diario.succ(pageName)

  local righe = {
    string.format(
      "# [[%s|⬅️]] %s [[%s|➡️]]",
      previousPage,
      titolo,
      nextPage
    )
  }

  if luoghi.hasString(p.description) then
    table.insert(
      righe,
      "📒 " .. p.description
    )
  end

  local media =
    widgets.topMediaLinksForDate(
      p.date
    )

  local info = {}

  if luoghi.hasString(p.Viaggio) then
    table.insert(
      info,
      "🧳 " .. p.Viaggio
    )
  end

  if luoghi.hasList(p.luoghi) then
    table.insert(
      info,
      luoghi.mapLink(pageName)
        .. " "
        .. table.concat(
          p.luoghi,
          ", "
        )
    )
  end

  if media ~= "" then
    table.insert(info, media)
  end

  if #info > 0 then
    table.insert(
      righe,
      table.concat(info, " · ")
    )
  end

  return table.concat(righe, "\n")
    .. "\n"
end


-- ============================================================
-- RENDERER COMPOSITI MARKDOWN
-- ============================================================

function widgets.siamoStatiQui(pageName)
  pageName = pageName or editor.getCurrentPage()

  if not luoghi.siamoStatiQuiEnabled(pageName) then
    return ""
  end

  local diarioInfo = luoghi.diarioPerLuogo(pageName)
  local catalog = widgets.diarioMediaCatalog(diarioInfo)
  local viaggi = widgets.siamoStatiQuiViaggi(pageName, diarioInfo)
  local giorni = widgets.siamoStatiQuiGiorni(pageName, diarioInfo, catalog)

  luoghi._siamoStatiQuiRender = {
    pageName = pageName,
    viaggi = viaggi,
    giorni = giorni
  }

  local sections = {}

  if viaggi ~= "" then
    table.insert(sections, viaggi)
  end

  if giorni ~= "" then
    table.insert(sections, giorni)
  end

  return table.concat(sections, "\n\n")
end


function widgets.siamoStatiQuiRender(pageName)
  pageName = pageName or editor.getCurrentPage()

  local cached = luoghi._siamoStatiQuiRender

  if cached
    and cached.pageName == pageName
  then
    return cached
  end

  widgets.siamoStatiQui(pageName)
  return luoghi._siamoStatiQuiRender
end


function widgets.infoViaggio(pageName)
  pageName = pageName or editor.getCurrentPage()

  if not string.startsWith(pageName, "Viaggi/") then
    return ""
  end

  local cached = widgets._infoViaggioRender
  if cached
    and cached.pageName == pageName
  then
    return cached.text or ""
  end

  local diarioInfo = viaggiDiarioInfo(pageName)

  if not diarioInfo or #diarioInfo == 0 then
    widgets._infoViaggioRender = {
      pageName = pageName,
      text = ""
    }
    return ""
  end

  local catalog = widgets.diarioMediaCatalog(diarioInfo)
  local luoghiText = widgets.infoViaggioLuoghi(pageName, diarioInfo)
  local diarioText = widgets.infoViaggioDiario(pageName, diarioInfo, catalog)

  local text = table.concat(
    {luoghiText, diarioText},
    "\n\n"
  )

  widgets._infoViaggioRender = {
    pageName = pageName,
    text = text
  }

  return text
end


-- ============================================================
-- LISTENER
-- ============================================================

-- La cache è solo runtime e viene ricostruita a ogni caricamento pagina.
local function resetSiamoStatiQuiRender()
  luoghi._siamoStatiQuiRender = nil
  widgets._infoViaggioRender = nil
end


event.listen {
  name = "editor:pageLoaded",
  run = resetSiamoStatiQuiRender
}



-- Un solo listener Top Widget per Luogo e Diario.
event.listen {
  name = "hooks:renderTopWidgets",

  run = function(e)
    local pageName = editor.getCurrentPage()

    if string.startsWith(pageName, "luoghi/") then
      local text = widgets.topLuogo(pageName)
      if text ~= "" then
        return widget.markdownBlock(text)
      end
      return
    end

    if config.get("std.widgets.wTopDiario.enabled", true)
      and string.startsWith(pageName, "Diario/")
    then
      local text = wTopDiario(pageName)
      if text and text ~= "" then
        return widget.markdownBlock(text)
      end
    end
  end
}


if config.get("std.widgets.linkedInfoLuoghi.enabled", true) then
  event.listen {
    name = "hooks:renderBottomWidgets",

    run = function(e)
      local pageName = editor.getCurrentPage()

      if pageName ~= "luoghi"
        and not string.startsWith(pageName, "luoghi/")
      then
        return
      end

      local text = widgets.linkedInfoLuoghi(pageName)
      if text ~= "" then
        return widget.markdownBlock(text)
      end
    end
  }
end


-- Due Bottom Widget distinti, con acquisizione Diario condivisa.
if config.get("std.widgets.siamoStatiQui.enabled", true) then
  event.listen {
    name = "hooks:renderBottomWidgets",

    run = function(e)
      local pageName = editor.getCurrentPage()

      if not luoghi.siamoStatiQuiEnabled(pageName) then
        return
      end

      local rendered = widgets.siamoStatiQuiRender(pageName)
      local text = rendered and rendered.viaggi or ""

      if text ~= "" then
        return widget.markdownBlock(text)
      end
    end
  }

  event.listen {
    name = "hooks:renderBottomWidgets",

    run = function(e)
      local pageName = editor.getCurrentPage()

      if not luoghi.siamoStatiQuiEnabled(pageName) then
        return
      end

      local rendered = widgets.siamoStatiQuiRender(pageName)
      local text = rendered and rendered.giorni or ""

      if text ~= "" then
        return widget.markdownBlock(text)
      end
    end
  }
end


-- Un solo Bottom Widget per Viaggio: Luoghi visitati + pagine Diario.
if config.get("std.widgets.infoViaggio.enabled", true) then
  event.listen {
    name = "hooks:renderBottomWidgets",

    run = function(e)
      local pageName = editor.getCurrentPage()
      local text = widgets.infoViaggio(pageName)

      if text ~= "" then
        return widget.markdownBlock(text)
      end
    end
  }
end
```

## Implementazione — Configurazioni e creazione pagine

### Configurazioni — blocco 1

```space-lua
-- priority: 10

mioDiarioConfigurazioni = mioDiarioConfigurazioni or {}

function mioDiarioConfigurazioni.nuovaPagina()
  local scelta = editor.filterBox(
    "Nuova Pagina",
    {
      {
        name = "Diario",
        description = "Crea una nuova pagina del Diario",
        value = "Page: Nuova Pagina Diario"
      },
      {
        name = "Luogo",
        description = "Crea una nuova pagina Luogo",
        value = "Page: Nuova Pagina Luogo"
      }
    },
    "Seleziona il tipo di pagina"
  )

  if not scelta or not scelta.value then
    return
  end

  editor.invokeCommand(scelta.value)
end

command.define {
  name = "Page: Nuova Pagina Luogo",
  run = function()
    return mioDiarioConfigurazioni.nuovaPaginaLuogo()
  end
}

command.define {
  name = "Page: Verifica Luogo",
  run = function()
    return mioDiarioConfigurazioni.verificaLuogo()
  end
}

actionButton.define {
  icon = "plus",
  description = "Nuova Pagina",
  priority = 1.2,
  run = mioDiarioConfigurazioni.nuovaPagina
}
```

### Configurazioni — blocco 2

```space-lua
-- priority: 9

function mioDiarioConfigurazioni.urlEncode(value)
  return string.gsub(value, "([^%w%-_%.~])", function(c)
    return string.format("%%%02X", string.byte(c))
  end)
end

function mioDiarioConfigurazioni.urlDecode(value)
  value = string.gsub(value, "+", " ")
  return string.gsub(value, "%%(%x%x)", function(hex)
    return string.char(tonumber(hex, 16))
  end)
end

function mioDiarioConfigurazioni.wikidataRequest(url)
  local response = net.proxyFetch(url, {
    headers = {
      Accept = "application/json"
    }
  })

  if not response.ok then
    error("Wikidata HTTP " .. tostring(response.status))
  end

  return response.body
end

function mioDiarioConfigurazioni.qidDaWikipediaTitle(title)
  local url = "https://www.wikidata.org/w/api.php" ..
    "?action=wbgetentities" ..
    "&sites=itwiki" ..
    "&titles=" .. mioDiarioConfigurazioni.urlEncode(title) ..
    "&props=" ..
    "&format=json" ..
    "&formatversion=2"

  local data = mioDiarioConfigurazioni.wikidataRequest(url)
  for qid, _ in pairs(data.entities or {}) do
    if string.match(qid, "^Q%d+$") then
      return qid
    end
  end
end

function mioDiarioConfigurazioni.cercaQid(input, language)
  local url = "https://www.wikidata.org/w/api.php" ..
    "?action=wbsearchentities" ..
    "&search=" .. mioDiarioConfigurazioni.urlEncode(input) ..
    "&language=" .. language ..
    "&uselang=it" ..
    "&type=item" ..
    "&limit=1" ..
    "&format=json" ..
    "&formatversion=2"

  local data = mioDiarioConfigurazioni.wikidataRequest(url)
  if data.search and data.search[1] then
    return data.search[1].id
  end
end

function mioDiarioConfigurazioni.cercaTitoloWikipedia(input)
  local url = "https://it.wikipedia.org/w/api.php" ..
    "?action=query" ..
    "&list=search" ..
    "&srsearch=" .. mioDiarioConfigurazioni.urlEncode(input) ..
    "&srlimit=1" ..
    "&format=json" ..
    "&formatversion=2"

  local data = mioDiarioConfigurazioni.wikidataRequest(url)
  local queryResult = data["query"]
  local result = queryResult and queryResult.search and queryResult.search[1]
  return result and result.title
end

function mioDiarioConfigurazioni.qidDaInput(input)
  if string.match(input, "^Q%d+$") then
    return input
  end

  local wikiTitle = string.match(input, "wikipedia%.org/wiki/(.+)$")
  if wikiTitle then
    wikiTitle = mioDiarioConfigurazioni.urlDecode(wikiTitle)
    wikiTitle = string.gsub(wikiTitle, "_", " ")
    return mioDiarioConfigurazioni.qidDaWikipediaTitle(wikiTitle)
  end

  local qid = mioDiarioConfigurazioni.cercaQid(input, "it")
  if qid then
    return qid
  end

  qid = mioDiarioConfigurazioni.cercaQid(input, "en")
  if qid then
    return qid
  end

  local title = mioDiarioConfigurazioni.cercaTitoloWikipedia(input)
  if title then
    return mioDiarioConfigurazioni.qidDaWikipediaTitle(title)
  end
end

function mioDiarioConfigurazioni.entitaWikidata(qid)
  local data = mioDiarioConfigurazioni.wikidataRequest(
    "https://www.wikidata.org/wiki/Special:EntityData/" .. qid .. ".json"
  )
  return data.entities and data.entities[qid]
end

function mioDiarioConfigurazioni.claimValue(entity, property)
  local claims = entity and entity.claims and entity.claims[property]
  local claim = claims and claims[1]
  local snak = claim and claim.mainsnak
  local datavalue = snak and snak.datavalue
  return datavalue and datavalue.value
end

function mioDiarioConfigurazioni.label(entity)
  if not entity or not entity.labels then
    return nil
  end
  if entity.labels.it then
    return entity.labels.it.value
  end
  if entity.labels.en then
    return entity.labels.en.value
  end
end

function mioDiarioConfigurazioni.aliases(entity)
  local result = {}
  local aliases = entity and entity.aliases and entity.aliases.it or {}
  for _, alias in ipairs(aliases) do
    table.insert(result, alias.value)
  end
  return result
end

function mioDiarioConfigurazioni.wikipedia(entity)
  local sitelink = entity and entity.sitelinks and entity.sitelinks.itwiki
  if not sitelink or not sitelink.title then
    return nil
  end
  local title = string.gsub(sitelink.title, " ", "_")
  return "https://it.wikipedia.org/wiki/" .. mioDiarioConfigurazioni.urlEncode(title)
end

function mioDiarioConfigurazioni.coordinate(entity)
  local value = mioDiarioConfigurazioni.claimValue(entity, "P625")
  if not value or not value.latitude or not value.longitude then
    return nil
  end
  return {value.latitude, value.longitude}
end

function mioDiarioConfigurazioni.tipo(entity)
  local value = mioDiarioConfigurazioni.claimValue(entity, "P31")
  local qid = value and value.id
  if not qid then
    return nil, nil
  end

  local typeEntity = mioDiarioConfigurazioni.entitaWikidata(qid)
  local label = mioDiarioConfigurazioni.label(typeEntity)
  if not label then
    return nil, qid
  end

  local lower = string.lower(label)
  if string.find(lower, "comune", 1, true) or string.find(lower, "municipality", 1, true) then
    return "comune", label
  end
  if string.find(lower, "stato sovrano", 1, true)
      or lower == "stato"
      or lower == "country"
      or lower == "stato federato degli stati uniti d'america" then
    return "stato", label
  end
  if lower == "grande città" then
    return "città", label
  end
  if string.find(lower, "regione", 1, true) then
    return "regione", label
  end
  if string.find(lower, "provincia", 1, true) then
    return "provincia", label
  end
  if string.find(lower, "dipartimento", 1, true) then
    return "dipartimento", label
  end
  if string.find(lower, "distretto", 1, true) then
    return "distretto", label
  end

  return nil, label
end

function mioDiarioConfigurazioni.divisioneIntermedia(entity)
  local current = entity

  for _ = 1, 6 do
    local parent = mioDiarioConfigurazioni.claimValue(current, "P131")
    local qid = parent and parent.id
    if not qid then
      return nil
    end

    current = mioDiarioConfigurazioni.entitaWikidata(qid)
    if not current then
      return nil
    end

    local iso = mioDiarioConfigurazioni.claimValue(current, "P300")
    if type(iso) == "string" then
      return iso
    end
  end
end

function mioDiarioConfigurazioni.leggiLuogo(input)
  local qid = mioDiarioConfigurazioni.qidDaInput(input)
  if not qid then
    return {error = "Nessun elemento Wikidata trovato", input = input}
  end

  local entity = mioDiarioConfigurazioni.entitaWikidata(qid)
  if not entity then
    return {error = "Elemento Wikidata non leggibile", qid = qid}
  end

  local tipoAmministrativo, tipoWikidata = mioDiarioConfigurazioni.tipo(entity)

  local tags = {}
  if not tipoAmministrativo and tipoWikidata then
    table.insert(tags, string.gsub(tipoWikidata, "%s+", "_"))
  end

  return {
    input = input,
    qid = qid,
    displayName = mioDiarioConfigurazioni.label(entity),
    tipoAmministrativo = tipoAmministrativo,
    tipoWikidata = tipoWikidata,
    divisioneIntermedia = mioDiarioConfigurazioni.divisioneIntermedia(entity),
    wikipedia = mioDiarioConfigurazioni.wikipedia(entity),
    coordinate = mioDiarioConfigurazioni.coordinate(entity),
    aliases = mioDiarioConfigurazioni.aliases(entity),
    tags = tags
  }
end

function mioDiarioConfigurazioni.testLuogo(input)
  return mioDiarioConfigurazioni.leggiLuogo(input)
end
```

### Configurazioni — blocco 3

```space-lua
-- priority: 8

function mioDiarioConfigurazioni.trim(value)
  if not value then
    return ""
  end
  return string.gsub(string.gsub(value, "^%s+", ""), "%s+$", "")
end

function mioDiarioConfigurazioni.normalizzaDirectoryLuoghi(directory)
  directory = mioDiarioConfigurazioni.trim(directory)
  directory = string.gsub(directory, "^/+", "")
  directory = string.gsub(directory, "/+$", "")

  if directory == "" then
    return "luoghi"
  end
  if directory ~= "luoghi" and not string.match(directory, "^luoghi/") then
    directory = "luoghi/" .. directory
  end
  return directory
end

function mioDiarioConfigurazioni.directoryNuovoLuogo()
  local currentPage = editor.getCurrentPage()

  if currentPage and string.match(currentPage, "^luoghi/") then
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
          description = "Specifica una directory sotto luoghi/",
          value = "__altra__"
        }
      },
      "Pagina corrente: " .. currentPage
    )

    if not scelta then
      return nil
    end
    if scelta.value ~= "__altra__" then
      return mioDiarioConfigurazioni.normalizzaDirectoryLuoghi(scelta.value)
    end
  end

  local directory = editor.prompt("Directory di destinazione", "luoghi/")
  if not directory then
    return nil
  end
  return mioDiarioConfigurazioni.normalizzaDirectoryLuoghi(directory)
end

function mioDiarioConfigurazioni.yamlQuote(value)
  value = tostring(value or "")
  value = string.gsub(value, "\\", "\\\\")
  value = string.gsub(value, "\"", "\\\"")
  return "\"" .. value .. "\""
end

```

### Configurazioni — blocco 4

```space-lua
-- priority: 7

function mioDiarioConfigurazioni.frontmatterLuogo(dati)
  local lines = {
    "---",
    "displayName: " .. mioDiarioConfigurazioni.yamlQuote(dati.displayName or "")
  }

  if dati.tipoAmministrativo then
    table.insert(lines, "tipoAmministrativo: " .. mioDiarioConfigurazioni.yamlQuote(dati.tipoAmministrativo))
  else
    table.insert(lines, "tipoAmministrativo:")
  end

  if dati.divisioneIntermedia then
    table.insert(lines, "divisioneIntermedia: " .. mioDiarioConfigurazioni.yamlQuote(dati.divisioneIntermedia))
  else
    table.insert(lines, "divisioneIntermedia:")
  end

  table.insert(lines, "aliases:")
  for _, alias in ipairs(dati.aliases or {}) do
    table.insert(lines, "- " .. mioDiarioConfigurazioni.yamlQuote(alias))
  end

  if dati.wikipedia then
    table.insert(lines, "wikipedia: " .. mioDiarioConfigurazioni.yamlQuote(dati.wikipedia))
  else
    table.insert(lines, "wikipedia:")
  end

  local c = mioDiarioConfigurazioni.normalizzaCoordinate(dati.coordinate)
  if c and c[1] and c[2] then
    table.insert(
      lines,
      "coordinate: [" ..
        string.format("%.5f", c[1]) .. ", " ..
        string.format("%.5f", c[2]) .. "]"
    )
  else
    table.insert(lines, "coordinate:")
  end

  table.insert(lines, "tags:")
  for _, tag in ipairs(dati.tags or {}) do
    table.insert(lines, "- " .. tag)
  end

  table.insert(lines, "---")
  return table.concat(lines, "\n") .. "\n"
end

function mioDiarioConfigurazioni.creaPaginaLuogo(pageName, dati)
  if space.pageExists(pageName) then
    editor.flashNotification("La pagina esiste già: " .. pageName, "error")
    return
  end

  local templatePage = "Library/MG/Mio_Diario/Page Templates/Luogo"
  local tpl = template.fromPage(templatePage)
  local initialText = mioDiarioConfigurazioni.frontmatterLuogo(dati)

  space.writePage(pageName, initialText)
  editor.navigate({kind = "page", page = pageName})
  editor.insertAtPos(tpl(), #initialText, true)
end

function mioDiarioConfigurazioni.nuovaPaginaLuogo()
  local directory = mioDiarioConfigurazioni.directoryNuovoLuogo()
  if not directory then
    return
  end

  local input = editor.prompt("Luogo da cercare", "")
  if not input or mioDiarioConfigurazioni.trim(input) == "" then
    return
  end

  local dati = mioDiarioConfigurazioni.leggiLuogo(input)
  if dati.error then
    editor.flashNotification(dati.error .. ": " .. input, "error")
    return
  end

  local nomePagina = editor.prompt("Nome della pagina", dati.displayName or input)
  if not nomePagina or mioDiarioConfigurazioni.trim(nomePagina) == "" then
    return
  end

  nomePagina = mioDiarioConfigurazioni.trim(nomePagina)
  local pageName = directory .. "/" .. nomePagina

  mioDiarioConfigurazioni.creaPaginaLuogo(pageName, dati)
end
```

### Configurazioni — blocco 5

```space-lua
-- priority: 6

function mioDiarioConfigurazioni.isEmpty(value)
  return value == nil or value == ""
end

function mioDiarioConfigurazioni.pageBaseName(pageName)
  return string.match(pageName or "", "([^/]+)$") or pageName
end

function mioDiarioConfigurazioni.asList(value)
  if value == nil or value == "" then
    return {}
  end
  if type(value) == "table" then
    return value
  end
  return {value}
end

function mioDiarioConfigurazioni.listContains(values, wanted)
  for _, value in ipairs(mioDiarioConfigurazioni.asList(values)) do
    if tostring(value) == tostring(wanted) then
      return true
    end
  end
  return false
end

function mioDiarioConfigurazioni.missingListValues(current, proposed)
  local missing = {}
  for _, value in ipairs(mioDiarioConfigurazioni.asList(proposed)) do
    if not mioDiarioConfigurazioni.listContains(current, value) then
      table.insert(missing, value)
    end
  end
  return missing
end

function mioDiarioConfigurazioni.mergeLists(current, proposed)
  local result = {}
  for _, value in ipairs(mioDiarioConfigurazioni.asList(current)) do
    table.insert(result, value)
  end
  for _, value in ipairs(mioDiarioConfigurazioni.missingListValues(current, proposed)) do
    table.insert(result, value)
  end
  return result
end

function mioDiarioConfigurazioni.round5(value)
  if value == nil then
    return nil
  end
  return math.floor(value * 100000 + 0.5) / 100000
end

function mioDiarioConfigurazioni.normalizzaCoordinate(value)
  if type(value) ~= "table" or not value[1] or not value[2] then
    return value
  end
  return {
    mioDiarioConfigurazioni.round5(value[1]),
    mioDiarioConfigurazioni.round5(value[2])
  }
end

function mioDiarioConfigurazioni.coordinateEqual(a, b)
  if type(a) ~= "table" or type(b) ~= "table" then
    return false
  end
  if not a[1] or not a[2] or not b[1] or not b[2] then
    return false
  end
  return math.abs(a[1] - b[1]) < 0.0001
    and math.abs(a[2] - b[2]) < 0.0001
end

function mioDiarioConfigurazioni.valueText(value)
  if value == nil or value == "" then
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

function mioDiarioConfigurazioni.htmlEscape(value)
  value = tostring(value or "")
  value = string.gsub(value, "&", "&amp;")
  value = string.gsub(value, "<", "&lt;")
  value = string.gsub(value, ">", "&gt;")
  value = string.gsub(value, '"', "&quot;")
  return value
end

function mioDiarioConfigurazioni.scalarStatus(current, proposed)
  if mioDiarioConfigurazioni.isEmpty(proposed) then
    if mioDiarioConfigurazioni.isEmpty(current) then
      return "OK", ""
    end
    return "N/D", "mantieni"
  end

  if mioDiarioConfigurazioni.isEmpty(current) then
    return "MANCANTE", "inserisci"
  end

  if tostring(current) == tostring(proposed) then
    return "OK", ""
  end

  return "DIFFERENTE", "mantieni"
end

function mioDiarioConfigurazioni.wikipediaStatus(current, proposed, qid)
  if mioDiarioConfigurazioni.isEmpty(current) then
    if mioDiarioConfigurazioni.isEmpty(proposed) then
      return "OK", ""
    end
    return "MANCANTE", "inserisci"
  end

  local currentQid = mioDiarioConfigurazioni.qidDaInput(current)
  if currentQid and currentQid == qid then
    return "OK", ""
  end

  return "DIFFERENTE", "mantieni"
end

function mioDiarioConfigurazioni.coordinateStatus(current, proposed)
  if proposed == nil then
    if current == nil then
      return "OK", ""
    end
    return "N/D", "mantieni"
  end

  if current == nil then
    return "MANCANTE", "inserisci"
  end

  if mioDiarioConfigurazioni.coordinateEqual(current, proposed) then
    return "OK", ""
  end

  return "DIFFERENTE", "mantieni"
end

function mioDiarioConfigurazioni.listStatus(current, proposed)
  local missing = mioDiarioConfigurazioni.missingListValues(current, proposed)
  if #missing == 0 then
    return "OK", "", missing
  end

  if #mioDiarioConfigurazioni.asList(current) == 0 then
    return "MANCANTE", "inserisci", missing
  end

  return "INCOMPLETO", "aggiungi " .. tostring(#missing), missing
end

function mioDiarioConfigurazioni.verificaLuogoDati()
  local currentPage = editor.getCurrentPage()
  if not currentPage or not string.match(currentPage, "^luoghi/") then
    return {error = "Il comando è disponibile solo nelle pagine luoghi/"}
  end

  local fmInfo = index.extractFrontmatter(editor.getText())
  local fm = fmInfo and fmInfo.frontmatter or {}
  local pageName = mioDiarioConfigurazioni.pageBaseName(currentPage)

  local input = fm.wikipedia
  if mioDiarioConfigurazioni.isEmpty(input) then
    input = pageName
  end

  local dati = mioDiarioConfigurazioni.leggiLuogo(input)

  if dati.error and not mioDiarioConfigurazioni.isEmpty(fm.wikipedia) then
    dati = mioDiarioConfigurazioni.leggiLuogo(pageName)
  end

  if dati.error then
    return dati
  end

  local rows = {}

  local function addScalar(field, current, proposed)
    local status, action = mioDiarioConfigurazioni.scalarStatus(current, proposed)
    table.insert(rows, {
      field = field,
      current = current,
      proposed = proposed,
      status = status,
      action = action
    })
  end

  addScalar("displayName", fm.displayName, dati.displayName)
  addScalar("tipoAmministrativo", fm.tipoAmministrativo, dati.tipoAmministrativo)
  addScalar("divisioneIntermedia", fm.divisioneIntermedia, dati.divisioneIntermedia)

  local aliasStatus, aliasAction, aliasMissing =
    mioDiarioConfigurazioni.listStatus(fm.aliases, dati.aliases)
  table.insert(rows, {
    field = "aliases",
    current = mioDiarioConfigurazioni.asList(fm.aliases),
    proposed = dati.aliases,
    status = aliasStatus,
    action = aliasAction,
    missing = aliasMissing
  })

  local wikipediaStatus, wikipediaAction =
    mioDiarioConfigurazioni.wikipediaStatus(fm.wikipedia, dati.wikipedia, dati.qid)
  table.insert(rows, {
    field = "wikipedia",
    current = fm.wikipedia,
    proposed = dati.wikipedia,
    status = wikipediaStatus,
    action = wikipediaAction
  })

  dati.coordinate = mioDiarioConfigurazioni.normalizzaCoordinate(dati.coordinate)

  local coordinateStatus, coordinateAction =
    mioDiarioConfigurazioni.coordinateStatus(fm.coordinate, dati.coordinate)
  table.insert(rows, {
    field = "coordinate",
    current = fm.coordinate,
    proposed = dati.coordinate,
    status = coordinateStatus,
    action = coordinateAction
  })

  local tagsStatus, tagsAction, tagsMissing =
    mioDiarioConfigurazioni.listStatus(fm.tags, dati.tags)
  table.insert(rows, {
    field = "tags",
    current = mioDiarioConfigurazioni.asList(fm.tags),
    proposed = dati.tags,
    status = tagsStatus,
    action = tagsAction,
    missing = tagsMissing
  })

  return {
    page = currentPage,
    qid = dati.qid,
    input = input,
    frontmatter = fm,
    wikidata = dati,
    rows = rows
  }
end

function mioDiarioConfigurazioni.verificaLuogoHtml(verifica)
  local html = {
    "<div style='font-family:sans-serif;padding:1rem'>",
    "<h2>Verifica Luogo</h2>",
    "<p><strong>Pagina:</strong> " ..
      mioDiarioConfigurazioni.htmlEscape(verifica.page) ..
      "<br><strong>Wikidata:</strong> " ..
      mioDiarioConfigurazioni.htmlEscape(verifica.qid) ..
      "</p>",
    "<table style='width:100%;border-collapse:collapse'>",
    "<thead><tr>",
    "<th style='text-align:left;border-bottom:1px solid #888;padding:.35rem'>Campo</th>",
    "<th style='text-align:left;border-bottom:1px solid #888;padding:.35rem'>Pagina</th>",
    "<th style='text-align:left;border-bottom:1px solid #888;padding:.35rem'>Wikidata</th>",
    "<th style='text-align:left;border-bottom:1px solid #888;padding:.35rem'>Stato</th>",
    "<th style='text-align:left;border-bottom:1px solid #888;padding:.35rem'>Azione</th>",
    "</tr></thead><tbody>"
  }

  for _, row in ipairs(verifica.rows) do
    table.insert(html,
      "<tr>" ..
      "<td style='vertical-align:top;border-bottom:1px solid #ddd;padding:.35rem'>" ..
        mioDiarioConfigurazioni.htmlEscape(row.field) .. "</td>" ..
      "<td style='vertical-align:top;border-bottom:1px solid #ddd;padding:.35rem'>" ..
        mioDiarioConfigurazioni.htmlEscape(
          mioDiarioConfigurazioni.valueText(row.current)
        ) .. "</td>" ..
      "<td style='vertical-align:top;border-bottom:1px solid #ddd;padding:.35rem'>" ..
        mioDiarioConfigurazioni.htmlEscape(
          mioDiarioConfigurazioni.valueText(row.proposed)
        ) .. "</td>" ..
      "<td style='vertical-align:top;border-bottom:1px solid #ddd;padding:.35rem'>" ..
        mioDiarioConfigurazioni.htmlEscape(row.status) .. "</td>" ..
      "<td style='vertical-align:top;border-bottom:1px solid #ddd;padding:.35rem'>" ..
        mioDiarioConfigurazioni.htmlEscape(row.action) .. "</td>" ..
      "</tr>"
    )
  end

  table.insert(html, "</tbody></table></div>")
  return table.concat(html, "")
end

function mioDiarioConfigurazioni.patchMancantiLuogo(verifica)
  local fm = verifica.frontmatter or {}
  local dati = verifica.wikidata or {}
  local ops = {}

  if mioDiarioConfigurazioni.isEmpty(fm.displayName)
      and not mioDiarioConfigurazioni.isEmpty(dati.displayName) then
    table.insert(ops, {
      op = "set-key",
      path = "displayName",
      value = dati.displayName
    })
  end

  if mioDiarioConfigurazioni.isEmpty(fm.tipoAmministrativo)
      and not mioDiarioConfigurazioni.isEmpty(dati.tipoAmministrativo) then
    table.insert(ops, {
      op = "set-key",
      path = "tipoAmministrativo",
      value = dati.tipoAmministrativo
    })
  end

  if mioDiarioConfigurazioni.isEmpty(fm.divisioneIntermedia)
      and not mioDiarioConfigurazioni.isEmpty(dati.divisioneIntermedia) then
    table.insert(ops, {
      op = "set-key",
      path = "divisioneIntermedia",
      value = dati.divisioneIntermedia
    })
  end

  local aliases = mioDiarioConfigurazioni.mergeLists(fm.aliases, dati.aliases)
  if #aliases > #mioDiarioConfigurazioni.asList(fm.aliases) then
    table.insert(ops, {
      op = "set-key",
      path = "aliases",
      value = aliases
    })
  end

  if mioDiarioConfigurazioni.isEmpty(fm.wikipedia)
      and not mioDiarioConfigurazioni.isEmpty(dati.wikipedia) then
    table.insert(ops, {
      op = "set-key",
      path = "wikipedia",
      value = dati.wikipedia
    })
  end

  if fm.coordinate == nil and dati.coordinate ~= nil then
    table.insert(ops, {
      op = "set-key",
      path = "coordinate",
      value = dati.coordinate
    })
  end

  local tags = mioDiarioConfigurazioni.mergeLists(fm.tags, dati.tags)
  if #tags > #mioDiarioConfigurazioni.asList(fm.tags) then
    table.insert(ops, {
      op = "set-key",
      path = "tags",
      value = tags
    })
  end

  return ops
end

function mioDiarioConfigurazioni.coordinateInlineText(text, coordinate)
  if type(coordinate) ~= "table" or not coordinate[1] or not coordinate[2] then
    return text
  end

  local lat = string.format("%.5f", coordinate[1])
  local lon = string.format("%.5f", coordinate[2])
  local inline = "coordinate: [" .. lat .. ", " .. lon .. "]"

  text = string.gsub(
    text,
    "coordinate:%s*\n%s*%-%s*[-%d%.]+\n%s*%-%s*[-%d%.]+",
    inline,
    1
  )

  return text
end

function mioDiarioConfigurazioni.applicaMancantiLuogo(verifica)
  local ops = mioDiarioConfigurazioni.patchMancantiLuogo(verifica)
  if #ops == 0 then
    editor.flashNotification("Nessun dato mancante da inserire")
    return false
  end

  local updated = index.patchFrontmatter(editor.getText(), ops)
  updated = mioDiarioConfigurazioni.coordinateInlineText(
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

function mioDiarioConfigurazioni.verificaLuogo()
  local verifica = mioDiarioConfigurazioni.verificaLuogoDati()

  if verifica.error then
    editor.flashNotification(verifica.error, "error")
    return
  end

  editor.showPanel(
    "rhs",
    1,
    mioDiarioConfigurazioni.verificaLuogoHtml(verifica),
    ""
  )

  local ops = mioDiarioConfigurazioni.patchMancantiLuogo(verifica)
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
    mioDiarioConfigurazioni.applicaMancantiLuogo(verifica)
  end

  return verifica.rows
end
```

## Implementazione — Indice e ricerca

```space-lua
-- priority: 10

indiceDiario = indiceDiario or {}
cercaLuoghi = cercaLuoghi or {}
widgets = widgets or {}



-- ============================================================
-- CONFIGURAZIONE
-- ============================================================

config.define("indiceDiario", {
  type = "object",
  properties = {
    batchSize = schema.number(),
    showSnippets = schema.boolean(),
    monthNames = {type = "array", items = {type = "string"}},
    dayNames = {type = "array", items = {type = "string"}}
  }
})

local function indiceDiarioConfig()
  local c = config.get("indiceDiario") or {}

  return {
    BATCH = math.max(1, math.floor(tonumber(c.batchSize) or 10)),
    SNIPPETS = c.showSnippets ~= false,
    MONTHS = c.monthNames or {
      "Gennaio", "Febbraio", "Marzo", "Aprile",
      "Maggio", "Giugno", "Luglio", "Agosto",
      "Settembre", "Ottobre", "Novembre", "Dicembre"
    },
    DAYS = c.dayNames or {
      "Lunedì", "Martedì", "Mercoledì", "Giovedì",
      "Venerdì", "Sabato", "Domenica"
    }
  }
end


-- ============================================================
-- DATI DIARIO E FILTRO
-- ============================================================

local function indiceDiarioWeekday(year, month, day, dayNames)
  local y = year
  local m = month

  if m < 3 then
    m = m + 12
    y = y - 1
  end

  local k = y % 100
  local j = math.floor(y / 100)
  local h = ((day + math.floor(13 * (m + 1) / 5) + k
    + math.floor(k / 4) + math.floor(j / 4) - 2 * j) % 7 + 7) % 7
  local remap = {6, 7, 1, 2, 3, 4, 5}

  return dayNames[remap[h + 1] or 1] or ""
end


local function indiceDiarioEntry(p)
  if type(p.name) ~= "string"
    or type(p.date) ~= "string"
    or p.date == ""
  then
    return nil
  end

  local y, m, d = p.date:match("^(%d%d%d%d)%-(%d%d)%-(%d%d)")
  y, m, d = tonumber(y), tonumber(m), tonumber(d)

  if not y or not m or not d then
    return nil
  end

  return {
    path = p.name,
    page = p.name,
    date = p.date,
    year = y,
    month = m,
    day = d,
    title = luoghi.hasString(p.displayName)
      and p.displayName
      or luoghi.basename(p.name),
    displayName = p.displayName,
    description = p.description,
    tags = p.tags,
    luoghi = p.luoghi,
    Viaggio = p.Viaggio
  }
end


function indiceDiario.entries()
  local pages = query[[
    from p = index.subPages("Diario")
    where p.date != nil
    order by p.date desc, p.name
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

  for _, p in ipairs(pages or {}) do
    local entry = indiceDiarioEntry(p)
    if entry then
      table.insert(entries, entry)
    end
  end

  return entries
end


function indiceDiario.list(value)
  if type(value) == "table" then
    return value
  end
  if type(value) == "string" and value ~= "" then
    return {value}
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
      table.insert(parts, item)
    elseif item ~= nil then
      table.insert(parts, tostring(item))
    end
  end
  return table.concat(parts, " ")
end


local function indiceDiarioTerms(value)
  local terms = {}
  local text = string.lower(tostring(value or ""))
  for term in text:gmatch("%S+") do
    table.insert(terms, term)
  end
  return terms
end


function indiceDiario.indexedSearchText(entry)
  return string.lower(table.concat({
    indiceDiario.valueText(entry.path),
    indiceDiario.valueText(entry.title),
    indiceDiario.valueText(entry.displayName),
    indiceDiario.valueText(entry.description),
    indiceDiario.valueText(entry.tags),
    indiceDiario.valueText(entry.luoghi),
    indiceDiario.valueText(entry.Viaggio)
  }, " "))
end


function indiceDiario.filter(entries, value)
  local terms = indiceDiarioTerms(value)
  if #terms == 0 then
    return entries
  end

  local filtered = {}

  for _, entry in ipairs(entries or {}) do
    entry.indexedSearchText = entry.indexedSearchText
      or indiceDiario.indexedSearchText(entry)

    local match = true
    for _, term in ipairs(terms) do
      if not string.find(entry.indexedSearchText, term, 1, true) then
        match = false
        break
      end
    end

    if match then
      table.insert(filtered, entry)
    end
  end

  return filtered
end


function indiceDiario.month(entries, year, month)
  local y, m = tonumber(year), tonumber(month)
  local result = {}

  for _, entry in ipairs(entries or {}) do
    if entry.year == y and entry.month == m then
      table.insert(result, entry)
    end
  end

  return result
end


-- ============================================================
-- MARKDOWN PER VIRTUAL PAGE
-- ============================================================

local function indiceDiarioTagLinks(value)
  local tags = {}

  if type(value) == "string" then
    for tag in value:gmatch("%S+") do
      tag = tag:gsub("^#", "")
      if tag ~= "" then
        table.insert(tags, "[[tag:" .. tag .. "|#" .. tag .. "]]")
      end
    end
  elseif type(value) == "table" then
    for _, rawTag in ipairs(value) do
      if type(rawTag) == "string" and rawTag ~= "" then
        local tag = rawTag:gsub("^#", "")
        table.insert(tags, "[[tag:" .. tag .. "|#" .. tag .. "]]")
      end
    end
  end

  return table.concat(tags, " ")
end


local function indiceDiarioWikiList(value)
  local parts = {}
  for _, item in ipairs(indiceDiario.list(value)) do
    if type(item) == "string" and item ~= "" then
      table.insert(parts, item)
    end
  end
  return table.concat(parts, " · ")
end


function indiceDiario.renderPeriod(heading, entries)
  local rows = {
    "# " .. heading,
    "",
    tostring(#entries) .. (#entries == 1 and " pagina trovata." or " pagine trovate."),
    ""
  }

  if #entries == 0 then
    table.insert(rows, "Nessun risultato.")
    return table.concat(rows, "\n") .. "\n"
  end

  for _, entry in ipairs(entries) do
    table.insert(
      rows,
      string.format(
        "[[%s|%02d/%02d/%04d]] - %s",
        entry.path,
        entry.day,
        entry.month,
        entry.year,
        entry.title
      )
    )
  end

  return table.concat(rows, "\n") .. "\n"
end


function indiceDiario.renderVirtual(heading, entries)
  local rows = {
    "# " .. heading,
    "",
    tostring(#entries) .. (#entries == 1 and " pagina trovata." or " pagine trovate."),
    ""
  }

  if #entries == 0 then
    table.insert(rows, "Nessun risultato.")
    return table.concat(rows, "\n") .. "\n"
  end

  for _, entry in ipairs(entries) do
    table.insert(rows, string.format(
      "## %02d/%02d/%04d — [[%s|%s]]",
      entry.day, entry.month, entry.year, entry.path, entry.title
    ))

    local description = type(entry.description) == "string"
      and entry.description:match("^%s*(.-)%s*$") or ""
    table.insert(rows, description ~= "" and description or "Descrizione non presente")

    local meta = {}
    local tags = indiceDiarioTagLinks(entry.tags)
    if tags ~= "" then table.insert(meta, tags) end

    local places = indiceDiarioWikiList(entry.luoghi)
    if places ~= "" then
      table.insert(meta, "[[Mappa/" .. entry.path .. "|🗺️]] " .. places)
    end

    if luoghi.hasString(entry.Viaggio) then
      table.insert(meta, "🧳 " .. entry.Viaggio)
    end

    if #meta > 0 then
      table.insert(rows, table.concat(meta, "   "))
    end
    table.insert(rows, "")
  end

  return table.concat(rows, "\n")
end


-- ============================================================
-- RICERCA LUOGHI
-- ============================================================

local function cercaLuoghiParentLabel(pageName, labelsByPath)
  local relative = pageName:gsub("^luoghi/", "")
  local parts = {}
  for part in relative:gmatch("[^/]+") do
    table.insert(parts, part)
  end

  if #parts <= 1 then
    return ""
  end

  local labels = {}
  local path = "luoghi"
  for i = 1, #parts - 1 do
    path = path .. "/" .. parts[i]
    table.insert(labels, labelsByPath[path] or parts[i])
  end

  return table.concat(labels, " › ")
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
  for _, p in ipairs(pages or {}) do
    labelsByPath[p.name] = luoghi.label(p, luoghi.basename(p.name))
  end

  local entries = {}
  for _, p in ipairs(pages or {}) do
    local entry = {
      path = p.name,
      title = labelsByPath[p.name],
      displayName = p.displayName,
      aliases = p.aliases,
      pathLabel = cercaLuoghiParentLabel(p.name, labelsByPath)
    }
    entry.indexedSearchText = string.lower(table.concat({
      indiceDiario.valueText(p.name),
      indiceDiario.valueText(p.title),
      indiceDiario.valueText(p.displayName),
      indiceDiario.valueText(p.aliases),
      indiceDiario.valueText(entry.title),
      indiceDiario.valueText(entry.pathLabel)
    }, " "))
    table.insert(entries, entry)
  end

  return entries
end


function cercaLuoghi.filter(entries, value)
  return indiceDiario.filter(entries, value)
end


-- ============================================================
-- RIEPILOGO ANNUALE
-- ============================================================

local function indiceDiarioHasTag(value, wanted)
  for _, tag in ipairs(indiceDiario.list(value)) do
    if type(tag) == "string" then
      if type(value) == "string" then
        for one in value:gmatch("%S+") do
          if one == wanted then return true end
        end
        return false
      end
      if tag == wanted then return true end
    end
  end
  return false
end


local function indiceDiarioCountryPath(target)
  if type(target) ~= "string" then return nil end
  local country = target:match("^luoghi/([^/]+)")
  return country and ("luoghi/" .. country) or nil
end


local function indiceDiarioAnnualDayLink(p)
  local label = luoghi.hasString(p.displayName)
    and p.displayName
    or luoghi.basename(p.name)
  return string.format("[[%s|%s]]", p.name, label)
end


function indiceDiario.yearEntries(year)
  local y = tonumber(year)
  if not y then return {} end

  local prefix = string.format("%04d-", y)
  local rows = query[[
    from p = index.subPages("Diario")
    where p.date and string.startsWith(p.date, prefix)
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

  return rows or {}
end


function indiceDiario.yearSummaryPage(year)
  local y = tonumber(year)
  if not y then return nil end

  local pageName = "riepiloghi/" .. string.format("%04d", y)
  local pages = query[[
    from p = index.pages()
    where p.name == pageName
    select {name = p.name}
    limit 1
  ]]

  return pages and pages[1] and pages[1].name == pageName
    and pageName or nil
end


function indiceDiario.openYear(year)
  local y = tonumber(year)
  if not y then return end

  editor.open(
    "riepiloghi/" .. string.format("%04d", y)
  )
end


function indiceDiario.renderYear(year)
  local y = tonumber(year)
  if not y then return "## Riepilogo\n\nAnno non valido.\n" end

  local entries = indiceDiario.yearEntries(y)
  local cfg = indiceDiarioConfig()

  local monthCounts = {}
  local rememberRows = {}
  local recordRows = {}
  local seenViaggi = {}
  local viaggi = {}
  local countriesByPath = {}
  local tagCounts = {}

  for _, p in ipairs(entries) do
    local month = tonumber(tostring(p.date or ""):sub(6, 7))
    if month then
      monthCounts[month] = (monthCounts[month] or 0) + 1
    end

    local isRemember = false
    local isRecord = false

    for _, rawTag in ipairs(indiceDiario.list(p.tags)) do
      if type(rawTag) == "string" then
        for tag in rawTag:gmatch("%S+") do
          tag = tag:gsub("^#", "")
          if tag ~= "" then
            tagCounts[tag] = (tagCounts[tag] or 0) + 1
            if tag == "ricorda" then isRemember = true end
            if tag == "record" then isRecord = true end
          end
        end
      end
    end

    if isRemember or isRecord then
      local row = "- " .. date.format(p.date) .. " — " .. indiceDiarioAnnualDayLink(p)
      if isRemember then table.insert(rememberRows, row) end
      if isRecord then table.insert(recordRows, row) end
    end

    local viaggio = luoghi.wikilinkTarget(p.Viaggio)
    if viaggio and not seenViaggi[viaggio] then
      seenViaggi[viaggio] = true
      table.insert(viaggi, p.Viaggio)
    end

    for _, link in ipairs(indiceDiario.list(p.luoghi)) do
      local target = luoghi.wikilinkTarget(link)
      local country = indiceDiarioCountryPath(target)
      if country and not countriesByPath[country] then
        countriesByPath[country] = {path = country, firstDate = p.date}
      end
    end
  end

  local rows = {
    "## Riepilogo " .. tostring(y), "",
    tostring(#entries) .. (#entries == 1 and " giornata" or " giornate"), "",
    "### Mesi"
  }

  local monthFound = 0
  for month = 1, 12 do
    local count = monthCounts[month]
    if count and count > 0 then
      table.insert(rows, string.format(
        "- [[Diario:%04d:%02d|%s]] — %d %s",
        y,
        month,
        cfg.MONTHS[month] or string.format("%02d", month),
        count,
        count == 1 and "giornata" or "giornate"
      ))
      monthFound = monthFound + 1
    end
  end
  if monthFound == 0 then table.insert(rows, "Nessuna giornata registrata.") end
  table.insert(rows, "")

  local function addRowsSection(level, title, values, emptyText)
    if rows[#rows] ~= "" then table.insert(rows, "") end
    table.insert(rows, level .. " " .. title)
    if level ~= "###" then table.insert(rows, "") end
    if #values == 0 then
      table.insert(rows, emptyText)
    else
      for _, value in ipairs(values) do table.insert(rows, value) end
    end
  end

  addRowsSection("##", "Date da ricordare", rememberRows, "Nessuna voce registrata.")

  if rows[#rows] ~= "" then table.insert(rows, "") end
  table.insert(rows, "### Viaggi")
  if #viaggi == 0 then
    table.insert(rows, "Nessun viaggio registrato.")
  else
    for _, viaggio in ipairs(viaggi) do table.insert(rows, "- " .. viaggio) end
  end

  local countries = {}
  for _, item in pairs(countriesByPath) do table.insert(countries, item) end
  table.sort(countries, function(a, b)
    return a.firstDate == b.firstDate and a.path < b.path or a.firstDate < b.firstDate
  end)

  local byName = {}
  if #countries > 0 then
    local paths = {}
    for _, c in ipairs(countries) do table.insert(paths, c.path) end
    for _, p in ipairs(luoghi.resolvePages(paths)) do byName[p.name] = p end
  end

  if rows[#rows] ~= "" then table.insert(rows, "") end
  table.insert(rows, "### Luoghi visitati")
  if #countries == 0 then
    table.insert(rows, "Nessun luogo registrato.")
  else
    local countryLinks = {}
    for _, c in ipairs(countries) do
      local label = luoghi.label(byName[c.path], luoghi.basename(c.path))
      table.insert(countryLinks, string.format(
        "[[diario:luogo:%04d:%s|%s]]", y, c.path, label
      ))
    end
    table.insert(rows, table.concat(countryLinks, ", "))
  end

  local tags = {}
  for tag, count in pairs(tagCounts) do
    table.insert(tags, {tag = tag, count = count})
  end
  table.sort(tags, function(a, b)
    if a.count ~= b.count then return a.count > b.count end
    return a.tag < b.tag
  end)

  if rows[#rows] ~= "" then table.insert(rows, "") end
  table.insert(rows, "### Tag più usati")
  if #tags == 0 then
    table.insert(rows, "Nessun tag registrato.")
  else
    local tagLinks = {}
    local limit = math.min(15, #tags)
    for i = 1, limit do
      local item = tags[i]
      table.insert(tagLinks, string.format(
        "[[tag:%s|#%s]] — %d",
        item.tag,
        item.tag,
        item.count
      ))
    end
    table.insert(rows, table.concat(tagLinks, ", "))
  end

  addRowsSection("###", "Record", recordRows, "Nessuna voce registrata.")


  return table.concat(rows, "\n") .. "\n"
end


function widgets.RiepilogoAnno()
  local current = editor.getCurrentPage()
  local year = type(current) == "string"
    and current:match("^riepiloghi/(%d%d%d%d)$") or nil

  if not year then
    return widget.markdownBlock(
      "_RiepilogoAnno è utilizzabile nelle pagine `riepiloghi/YYYY`._"
    )
  end

  return widget.markdownBlock(indiceDiario.renderYear(year))
end


function indiceDiario.renderYearPlace(year, pageName)
  local y = tonumber(year)
  if not y
    or type(pageName) ~= "string"
    or not string.startsWith(pageName, "luoghi/")
  then
    return "# Luogo visitato\n\nParametri non validi.\n"
  end

  local filtered = {}
  for _, p in ipairs(indiceDiario.yearEntries(y)) do
    if luoghi.contieneLuogo(p.luoghi, pageName) then
      table.insert(filtered, p)
    end
  end

  local currentPlace = luoghi.getPage(pageName)
  local rows = {
    "# " .. luoghi.label(currentPlace, luoghi.basename(pageName)) .. " — " .. tostring(y),
    "", "## Viaggi", ""
  }

  local seenViaggi, countViaggi = {}, 0
  for _, p in ipairs(filtered) do
    local target = luoghi.wikilinkTarget(p.Viaggio)
    if target and not seenViaggi[target] then
      table.insert(rows, "- " .. p.Viaggio)
      seenViaggi[target] = true
      countViaggi = countViaggi + 1
    end
  end
  if countViaggi == 0 then table.insert(rows, "Nessun viaggio registrato.") end

  table.insert(rows, "")
  table.insert(rows, "## Luoghi visitati")
  table.insert(rows, "")

  local places, seen = {}, {}
  local prefix = pageName .. "/"
  for _, p in ipairs(filtered) do
    for _, link in ipairs(indiceDiario.list(p.luoghi)) do
      local target = luoghi.wikilinkTarget(link)
      if target
        and (target == pageName or string.startsWith(target, prefix))
        and not seen[target]
      then
        table.insert(places, {
          target = target,
          link = link,
          firstDate = p.date,
          parent = luoghi.parentName(target) or target
        })
        seen[target] = true
      end
    end
  end
  table.sort(places, function(a, b)
    if a.parent ~= b.parent then return a.parent < b.parent end
    if a.firstDate ~= b.firstDate then return a.firstDate < b.firstDate end
    return a.target < b.target
  end)

  if #places == 0 then
    table.insert(rows, "Nessun luogo registrato.")
  else
    for _, place in ipairs(places) do
      table.insert(rows, "- " .. place.link)
    end
  end

  return table.concat(rows, "\n") .. "\n"
end


-- ============================================================
-- VIRTUAL PAGE ATTIVE
-- ============================================================

virtualPage.define {
  pattern = "^diario:luogo:(.+)$",
  run = function(spec)
    local year, pageName = spec:match("^(%d%d%d%d):(luoghi/.+)$")
    return indiceDiario.renderYearPlace(year, pageName)
  end
}


function indiceDiario.renderMonth(period)
  local year, month = tostring(period or ""):match("^(%d%d%d%d)%-(%d%d)$")
  local y, m = tonumber(year), tonumber(month)
  if not y or not m or m < 1 or m > 12 then
    return "# Diario\n\nPeriodo non valido.\n"
  end

  local cfg = indiceDiarioConfig()
  local entries = indiceDiario.month(indiceDiario.entries(), y, m)
  local monthName = cfg.MONTHS[m] or string.format("%02d", m)

  return indiceDiario.renderPeriod(
    "Diario — " .. monthName .. " " .. tostring(y),
    entries
  )
end


virtualPage.define {
  pattern = "^Diario:(.+)$",
  run = function(spec)
    local year = spec:match("^(%d%d%d%d)$")
    if year then
      return string.format(
        "## Riepilogo %s\n\n[[riepiloghi/%s|Apri il riepilogo annuale %s]]\n",
        year, year, year
      )
    end

    local y, m = spec:match("^(%d%d%d%d):(%d%d)$")
    if y and m then return indiceDiario.renderMonth(y .. "-" .. m) end

    return "# Diario\n\nPeriodo non valido.\n"
  end
}


virtualPage.define {
  pattern = "^diario:ricerca:(.+)$",
  run = function(searchText)
    return indiceDiario.renderVirtual(
      "Ricerca Diario — " .. searchText,
      indiceDiario.filter(indiceDiario.entries(), searchText)
    )
  end
}


-- ============================================================
-- INDICE INLINE: DOM necessario per filtro live e lazy rendering
-- ============================================================

function widgets.IndiceDiario()
  local cfg = indiceDiarioConfig()
  local allEntries = nil

  local function entryTags(value)
    local result = {}
    if type(value) == "string" then
      for tag in value:gmatch("%S+") do table.insert(result, tag) end
    elseif type(value) == "table" then
      for _, tag in ipairs(value) do
        if type(tag) == "string" and tag ~= "" then table.insert(result, tag) end
      end
    end
    return result
  end

  local function buildTagsNode(entry)
    local tags = entryTags(entry.tags)
    if #tags == 0 then return nil end

    local row = dom.div {class = "id-meta-row id-tags"}
    for i, rawTag in ipairs(tags) do
      local tag = rawTag:gsub("^#", "")
      if tag ~= "" then
        if i > 1 then row.appendChild(dom.span {__rawText = " "}) end
        row.appendChild(dom.a {
          class = "id-meta-link",
          onclick = function() editor.open("tag:" .. tag) end,
          __rawText = "#" .. tag
        })
      end
    end
    return row
  end

  local function buildLuoghiNode(entry)
    local values = indiceDiario.list(entry.luoghi)
    if #values == 0 then return nil end

    local row = dom.div {class = "id-meta-row id-luoghi"}
    row.appendChild(dom.a {
      class = "id-meta-link",
      onclick = function() editor.open("Mappa/" .. entry.path) end,
      __rawText = "🗺️ "
    })

    local added = 0
    for _, value in ipairs(values) do
      local target = luoghi.wikilinkTarget(value)
      local label = target and luoghi.wikilinkLabel(value) or tostring(value)

      if added > 0 then row.appendChild(dom.span {__rawText = " · "}) end
      if target then
        row.appendChild(dom.a {
          class = "id-meta-link",
          onclick = function() editor.open(target) end,
          __rawText = label
        })
      else
        row.appendChild(dom.span {__rawText = label})
      end
      added = added + 1
    end

    return added > 0 and row or nil
  end

  local function buildViaggioNode(entry)
    local target = luoghi.wikilinkTarget(entry.Viaggio)
    if not target then return nil end

    return dom.div {
      class = "id-meta-row id-viaggio",
      dom.a {
        class = "id-meta-link",
        onclick = function() editor.open(target) end,
        __rawText = "🧳 " .. (luoghi.wikilinkLabel(entry.Viaggio) or luoghi.basename(target))
      }
    }
  end

  local function buildEntryNode(entry)
    local dayName = indiceDiarioWeekday(
      entry.year,
      entry.month,
      entry.day,
      cfg.DAYS
    )

    local dateLabel = string.format(
      "%02d/%02d/%04d · %s",
      entry.day,
      entry.month,
      entry.year,
      dayName
    )

    local row = dom.div {
      class = "id-entry",
      dom.div {
        class = "id-date",
        dom.a {
          class = "id-date-link",
          onclick = function() editor.open(entry.path) end,
          __rawText = dateLabel
        }
      },
      dom.div {
        class = "id-title",
        dom.a {
          class = "id-title-link",
          onclick = function() editor.open(entry.path) end,
          dom.strong {__rawText = entry.title}
        }
      }
    }

    if cfg.SNIPPETS then
      local description = type(entry.description) == "string"
        and entry.description:match("^%s*(.-)%s*$") or ""
      row.appendChild(dom.div {
        class = "id-snippet",
        __rawText = description ~= "" and description or "Descrizione non presente"
      })
    end

    local tagsNode = buildTagsNode(entry)
    if tagsNode then row.appendChild(tagsNode) end

    local luoghiNode = buildLuoghiNode(entry)
    if luoghiNode then row.appendChild(luoghiNode) end

    local viaggioNode = buildViaggioNode(entry)
    if viaggioNode then row.appendChild(viaggioNode) end

    return row
  end

  local root = dom.div {class = "id-index"}
  local filter = dom.input {
    class = "id-filter", type = "search", disabled = true,
    placeholder = "Filtra il Diario…"
  }
  local openSearch = dom.button {
    class = "id-open-search", type = "button", disabled = true,
    __rawText = "Apri risultati"
  }
  local loading = dom.div {class = "id-loading", __rawText = "Lettura pagine Diario in corso…"}
  local list = dom.div {class = "id-list"}

  root.appendChild(dom.div {class = "id-filter-bar", filter, openSearch})
  root.appendChild(loading)
  root.appendChild(list)

  local currentFilter = ""
  local activeEntries = {}
  local renderedCount = 0
  local lastMonthKey = nil

  local function appendMonth(entry)
    local key = string.format("%04d-%02d", entry.year, entry.month)
    if key == lastMonthKey then return end

    local monthName = cfg.MONTHS[entry.month] or string.format("%02d", entry.month)
    list.appendChild(dom.div {
      class = "id-month",
      dom.a {
        class = "id-month-link",
        onclick = function() editor.open(string.format("Diario:%04d:%02d", entry.year, entry.month)) end,
        __rawText = monthName
      },
      dom.span {__rawText = " "},
      dom.a {
        class = "id-year-link",
        onclick = function() indiceDiario.openYear(entry.year) end,
        __rawText = tostring(entry.year)
      }
    })
    lastMonthKey = key
  end

  local function renderNextBatch()
    if renderedCount >= #activeEntries then return end
    local last = math.min(renderedCount + cfg.BATCH, #activeEntries)
    for i = renderedCount + 1, last do
      appendMonth(activeEntries[i])
      list.appendChild(buildEntryNode(activeEntries[i]))
    end
    renderedCount = last
  end

  local function resetList(entries)
    activeEntries = entries
    renderedCount = 0
    lastMonthKey = nil
    list.replaceChildren()

    if #entries == 0 then
      list.appendChild(dom.div {class = "id-empty", "Nessun risultato."})
      return
    end

    renderNextBatch()
    list.scrollTop = 0
  end

  list.addEventListener("scroll", function()
    if list.scrollHeight - list.scrollTop - list.clientHeight < 180 then
      renderNextBatch()
    end
  end)

  local filterTimer = nil
  filter.addEventListener("input", function(e)
    local value = tostring(e.target.value or "")
    currentFilter = value
    openSearch.disabled = value:match("^%s*$") ~= nil

    if filterTimer then js.window.clearTimeout(filterTimer) end
    filterTimer = js.window.setTimeout(function()
      local trimmed = value:match("^%s*(.-)%s*$") or ""
      resetList(trimmed == "" and allEntries or indiceDiario.filter(allEntries, trimmed))
    end, 280)
  end)

  openSearch.addEventListener("click", function()
    local searchText = currentFilter:match("^%s*(.-)%s*$")
    if searchText and searchText ~= "" then
      editor.open("diario:ricerca:" .. searchText)
    end
  end)

  local sourcePage = editor.getCurrentPage()

  js.window.setTimeout(function()
    if editor.getCurrentPage() ~= sourcePage then
      return
    end

    allEntries = indiceDiario.entries() or {}
    loading.replaceChildren()

    if #allEntries == 0 then
      list.replaceChildren(dom.div {class = "id-empty", "Nessuna pagina del Diario trovata."})
      return
    end

    filter.disabled = false
    filter.placeholder = "Filtra " .. #allEntries .. " pagine…"

    resetList(allEntries)
  end, 20)

  return widget.htmlBlock(root)
end


-- ============================================================
-- CERCA LUOGHI: DOM necessario per ricerca live
-- ============================================================

function widgets.CercaLuoghi()
  local root = dom.div {class = "cl-index"}
  local filter = dom.input {class = "cl-filter", type = "search", placeholder = "Cerca un luogo…"}
  local status = dom.div {class = "cl-status"}
  local list = dom.div {class = "cl-list"}

  root.appendChild(filter)
  root.appendChild(status)
  root.appendChild(list)

  local allEntries = nil
  local activeEntries = {}
  local renderedCount = 0
  local batchSize = 25

  local function ensureEntries()
    if not allEntries then allEntries = cercaLuoghi.entries() end
    return allEntries
  end

  local function buildResultNode(entry)
    local row = dom.div {
      class = "cl-result",
      onclick = function() editor.open(entry.path) end,
      dom.div {class = "cl-title", __rawText = entry.title}
    }
    if entry.pathLabel and entry.pathLabel ~= "" then
      row.appendChild(dom.div {class = "cl-path", __rawText = entry.pathLabel})
    end
    return row
  end

  local function renderNextBatch()
    local last = math.min(renderedCount + batchSize, #activeEntries)
    for i = renderedCount + 1, last do
      list.appendChild(buildResultNode(activeEntries[i]))
    end
    renderedCount = last
  end

  local function resetResults(entries)
    activeEntries = entries
    renderedCount = 0
    list.replaceChildren()
    status.textContent = #entries == 0
      and "Nessun luogo trovato."
      or tostring(#entries) .. (#entries == 1 and " luogo trovato" or " luoghi trovati")
    if #entries > 0 then renderNextBatch() end
  end

  local function clearResults()
    activeEntries = {}
    renderedCount = 0
    list.replaceChildren()
    status.textContent = ""
  end

  list.addEventListener("scroll", function()
    if list.scrollHeight - list.scrollTop - list.clientHeight < 160 then
      renderNextBatch()
    end
  end)

  local filterTimer = nil
  filter.addEventListener("input", function(e)
    local value = tostring(e.target.value or "")
    if value:match("^%s*$") then
      if filterTimer then js.window.clearTimeout(filterTimer) end
      filterTimer = nil
      clearResults()
      return
    end

    if filterTimer then js.window.clearTimeout(filterTimer) end
    filterTimer = js.window.setTimeout(function()
      local trimmed = value:match("^%s*(.-)%s*$") or ""
      resetResults(cercaLuoghi.filter(ensureEntries(), trimmed))
    end, 250)
  end)

  return widget.htmlBlock(root)
end
```

### Stile Indice

```space-style
:root {
  --id-border: var(--modal-border-color);
  --id-bg: var(--top-background-color);
  --id-text: var(--root-color);
  --id-muted: color-mix(in srgb, var(--root-color) 65%, transparent);
  --id-accent: var(--ui-accent-color);
  --id-accent-text: var(--ui-accent-contrast-color, white);
  --id-card: color-mix(in srgb, var(--top-background-color) 92%, var(--root-color) 8%);
  --id-hover: color-mix(in srgb, var(--top-background-color) 86%, var(--root-color) 14%);
}

.id-index, .cl-index {display:flex; flex-direction:column; gap:8px; font-size:inherit;}
.id-loading {padding:8px 2px; color:var(--id-muted); font-size:.86em; font-style:italic;}
.id-loading:empty, .cl-list:empty {display:none;}
.id-filter-bar {display:grid; grid-template-columns:minmax(0,1fr) auto; gap:7px;}
.id-filter, .cl-filter {width:100%; min-width:0; box-sizing:border-box; padding:7px 9px; border:1px solid var(--id-border); border-radius:7px; background:var(--id-bg); color:var(--id-text); font:inherit; outline:none;}
.id-filter:focus, .cl-filter:focus {border-color:var(--id-accent);}
.id-open-search {box-sizing:border-box; padding:7px 10px; border:1px solid var(--id-border); border-radius:7px; background:var(--id-card); color:var(--id-text); font:inherit; cursor:pointer;}
.id-open-search:hover:not(:disabled), .id-entry:hover, .cl-result:hover {background:var(--id-hover);}
.id-open-search:disabled {opacity:.45; cursor:default;}
.id-list, .cl-list {display:flex; flex-direction:column; gap:8px; max-height:min(60vh,560px); overflow-y:auto; overflow-x:hidden; padding-right:3px; scrollbar-gutter:stable;}
.id-list::-webkit-scrollbar, .cl-list::-webkit-scrollbar {width:5px;}
.id-list::-webkit-scrollbar-track, .cl-list::-webkit-scrollbar-track {background:transparent;}
.id-list::-webkit-scrollbar-thumb, .cl-list::-webkit-scrollbar-thumb {background:var(--id-border); border-radius:3px;}
.id-empty {padding:18px 8px; color:var(--id-muted); text-align:center;}
.id-month {margin-top:5px; padding:3px 1px; color:var(--id-muted); font-weight:700; letter-spacing:.08em; text-transform:uppercase;}
.id-month-link, .id-year-link, .id-date-link, .id-title-link, .id-meta-link {color:inherit; text-decoration:none; cursor:pointer;}
.id-month-link:hover, .id-year-link:hover, .id-date-link:hover, .id-title-link:hover, .id-meta-link:hover {text-decoration:underline;}
.id-entry {display:flex; flex-direction:column; gap:4px; align-items:stretch; min-width:0; padding:8px 9px; border:1px solid var(--id-border); border-radius:8px; background:var(--id-card);}
.id-date {min-width:0; color:var(--id-muted); font-size:.78em; font-weight:700; line-height:1.25; overflow-wrap:anywhere;}
.id-title {min-width:0; font-weight:700; line-height:1.35; overflow-wrap:anywhere;}
.id-snippet {min-width:0; color:var(--id-muted); line-height:1.4; overflow-wrap:anywhere;}
.id-meta-row {min-width:0; color:var(--id-muted); font-size:.78em; line-height:1.3; overflow-wrap:anywhere;}
.id-tags {margin-top:2px;}
.cl-status {min-height:1.2em; color:var(--id-muted); font-size:.8em;}
.cl-result {min-width:0; padding:7px 9px; border:1px solid var(--id-border); border-radius:7px; background:var(--id-card); cursor:pointer;}
.cl-title {min-width:0; font-weight:700; line-height:1.3; overflow-wrap:anywhere;}
.cl-path {margin-top:2px; color:var(--id-muted); font-size:.78em; line-height:1.3; overflow-wrap:anywhere;}

@media (max-width:520px) {
  .id-filter-bar {grid-template-columns:1fr;}
  .id-open-search {width:100%;}
  .id-entry {gap:5px;}
  .id-snippet {display:-webkit-box; -webkit-box-orient:vertical; -webkit-line-clamp:3; overflow:hidden;}
  .id-date, .id-meta-row, .cl-path {font-size:.8em;}
  .cl-result {padding:8px 9px;}
}
```

## Implementazione — Riepiloghi

```space-lua
-- priority: 20

widgets = widgets or {}
riepiloghi = riepiloghi or {}


-- ============================================================
-- CONFIGURAZIONE
-- ============================================================

local TERMINAL_TYPES = {
  ["comune"] = true,
  ["città"] = true,
  ["citta"] = true,
  ["località"] = true,
  ["localita"] = true,
  ["attrazione"] = true,
}


-- ============================================================
-- FUNZIONI GENERALI
-- ============================================================

local function list(value)
  if type(value) == "table" then
    return value
  end

  if type(value) == "string" and value ~= "" then
    return { value }
  end

  return {}
end

local function wikilinkTarget(value)
  if type(value) ~= "string" then
    return nil
  end

  return value:match("^%[%[([^]|]+)")
end

local function wikilinkLabel(value)
  if type(value) ~= "string" then
    return nil
  end

  local target, label =
    value:match("^%[%[([^]|]+)|([^]]+)%]%]$")

  if target then
    return label
  end

  target = wikilinkTarget(value)
  return target and target:match("([^/]+)$") or nil
end

local function basename(name)
  return name:match("([^/]+)$") or name
end

local function pageLabel(p)
  if p then
    if type(p.displayName) == "string" and p.displayName ~= "" then
      return p.displayName
    end

    if type(p.title) == "string" and p.title ~= "" then
      return p.title
    end

    if type(p.aliases) == "table"
      and type(p.aliases[1]) == "string"
      and p.aliases[1] ~= ""
    then
      return p.aliases[1]
    end
  end

  return p and basename(p.name) or ""
end

local function pageLink(name, label)
  return string.format(
    "[[%s|%s]]",
    name,
    label or basename(name)
  )
end

local function dateLabel(value)
  local y, m, d =
    tostring(value or ""):match(
      "^(%d%d%d%d)%-(%d%d)%-(%d%d)"
    )

  if not y then
    return tostring(value or "")
  end

  return d .. "/" .. m .. "/" .. y
end

local function clampN(n, defaultValue)
  n = math.floor(tonumber(n) or defaultValue)
  return math.max(1, n)
end

local function randomSample(values, n)
  n = math.min(clampN(n, 1), #values)

  local copy = {}
  for _, value in ipairs(values) do
    table.insert(copy, value)
  end

  for i = 1, n do
    local j = math.random(i, #copy)
    copy[i], copy[j] = copy[j], copy[i]
  end

  local result = {}
  for i = 1, n do
    table.insert(result, copy[i])
  end

  return result
end

local function ensureRandomSeed()
  if riepiloghi._randomSeeded then
    return
  end

  math.randomseed(os.time())
  riepiloghi._randomSeeded = true
end


-- ============================================================
-- DATI DIARIO
-- ============================================================

function riepiloghi.diarioEntries()
  if not indiceDiario
    or type(indiceDiario.entries) ~= "function"
  then
    return {}
  end

  return indiceDiario.entries()
end


-- ============================================================
-- DATI LUOGHI
-- ============================================================

function riepiloghi.placeIndex()
  local pages = query[[
    from p = index.subPages("luoghi")
    select {
      name = p.name,
      title = p.title,
      displayName = p.displayName,
      aliases = p.aliases,
      tipoAmministrativo = p.tipoAmministrativo
    }
  ]]

  local byName = {}
  local hasChildren = {}

  for _, p in ipairs(pages) do
    byName[p.name] = p

    local parent =
      p.name:match("^(.*)/[^/]+$")

    if parent then
      hasChildren[parent] = true
    end
  end

  return byName, hasChildren
end

local function isTerminalPlace(p, hasChildren)
  if not p then
    return false
  end

  if not hasChildren[p.name] then
    return true
  end

  local tipo =
    type(p.tipoAmministrativo) == "string"
    and string.lower(p.tipoAmministrativo)
    or ""

  return TERMINAL_TYPES[tipo] == true
end

function riepiloghi.placeStats()
  local entries = riepiloghi.diarioEntries()
  local byName, hasChildren = riepiloghi.placeIndex()
  local stats = {}

  for _, entry in ipairs(entries) do
    local seenDay = {}

    for _, raw in ipairs(list(entry.luoghi)) do
      local target = wikilinkTarget(raw)

      if target
        and byName[target]
        and isTerminalPlace(byName[target], hasChildren)
        and not seenDay[target]
      then
        local item = stats[target]

        if not item then
          item = {
            name = target,
            page = byName[target],
            count = 0,
            firstDate = entry.date,
            lastDate = entry.date,
          }
          stats[target] = item
        end

        item.count = item.count + 1

        if entry.date < item.firstDate then
          item.firstDate = entry.date
        end

        if entry.date > item.lastDate then
          item.lastDate = entry.date
        end

        seenDay[target] = true
      end
    end
  end

  local result = {}
  for _, item in pairs(stats) do
    table.insert(result, item)
  end

  return result
end


-- ============================================================
-- DATI VIAGGI
-- ============================================================

function riepiloghi.tripStats()
  local stats = {}

  for _, entry in ipairs(riepiloghi.diarioEntries()) do
    local target = wikilinkTarget(entry.Viaggio)

    if target then
      local item = stats[target]

      if not item then
        item = {
          name = target,
          label = wikilinkLabel(entry.Viaggio) or basename(target),
          count = 0,
          firstDate = entry.date,
          lastDate = entry.date,
          places = {},
        }
        stats[target] = item
      end

      item.count = item.count + 1

      if entry.date < item.firstDate then
        item.firstDate = entry.date
      end

      if entry.date > item.lastDate then
        item.lastDate = entry.date
      end

      for _, raw in ipairs(list(entry.luoghi)) do
        local place = wikilinkTarget(raw)
        if place then
          item.places[place] = true
        end
      end
    end
  end

  local result = {}

  for _, item in pairs(stats) do
    local count = 0
    for _ in pairs(item.places) do
      count = count + 1
    end
    item.placeCount = count
    table.insert(result, item)
  end

  return result
end


-- ============================================================
-- OGGI
-- ============================================================

function riepiloghi.renderAccaddeOggi()
  local now = os.date("*t")
  local rows = { "## Accadde oggi", "" }
  local found = 0

  for _, entry in ipairs(riepiloghi.diarioEntries()) do
    if entry.month == now.month
      and entry.day == now.day
      and entry.year < now.year
    then
      table.insert(
        rows,
        string.format(
          "- **%d** — %s",
          entry.year,
          pageLink(entry.path, entry.title)
        )
      )
      found = found + 1
    end
  end

  if found == 0 then
    table.insert(rows, "Nessun ricordo registrato per oggi.")
  end

  return table.concat(rows, "\n")
end

function widgets.AccaddeOggi()
  return widget.markdownBlock(
    riepiloghi.renderAccaddeOggi()
  )
end


-- ============================================================
-- RICORDI CASUALI
-- ============================================================

function riepiloghi.renderRicordiCasuali(n)
  ensureRandomSeed()
  n = clampN(n, 3)

  local entries = riepiloghi.diarioEntries()
  local places = riepiloghi.placeStats()

  local rows = {
    "## Ricordi a caso",
    ""
  }

  for _, entry in ipairs(randomSample(entries, n)) do
    table.insert(
      rows,
      string.format(
        "- %s — %s",
        dateLabel(entry.date),
        pageLink(entry.path, entry.title)
      )
    )
  end

  table.insert(rows, "")
  table.insert(rows, "## Luoghi in cui siamo stati")
  table.insert(rows, "")

  for _, item in ipairs(randomSample(places, n)) do
    table.insert(
      rows,
      "- " .. pageLink(item.name, pageLabel(item.page))
    )
  end

  return table.concat(rows, "\n")
end

function widgets.RicordiCasuali(n)
  return widget.markdownBlock(
    riepiloghi.renderRicordiCasuali(n)
  )
end


-- ============================================================
-- LUOGHI
-- ============================================================

function riepiloghi.renderLuoghiPiuVisitati(n)
  n = clampN(n, 10)
  local values = riepiloghi.placeStats()

  table.sort(values, function(a, b)
    if a.count ~= b.count then
      return a.count > b.count
    end
    return pageLabel(a.page) < pageLabel(b.page)
  end)

  local rows = { "## Luoghi più visitati", "" }

  for i = 1, math.min(n, #values) do
    local item = values[i]
    table.insert(
      rows,
      string.format(
        "- %s — **%d** giorn%s",
        pageLink(item.name, pageLabel(item.page)),
        item.count,
        item.count == 1 and "o" or "i"
      )
    )
  end

  if #values == 0 then
    table.insert(rows, "Nessun luogo registrato.")
  end

  return table.concat(rows, "\n")
end

function widgets.LuoghiPiuVisitati(n)
  return widget.markdownBlock(
    riepiloghi.renderLuoghiPiuVisitati(n)
  )
end

function riepiloghi.renderLuoghiDimenticati(n)
  n = clampN(n, 10)
  local values = riepiloghi.placeStats()

  table.sort(values, function(a, b)
    if a.lastDate ~= b.lastDate then
      return a.lastDate < b.lastDate
    end
    return pageLabel(a.page) < pageLabel(b.page)
  end)

  local rows = { "## È da molto che non torniamo qui", "" }

  for i = 1, math.min(n, #values) do
    local item = values[i]
    table.insert(
      rows,
      string.format(
        "- %s — ultima visita %s",
        pageLink(item.name, pageLabel(item.page)),
        dateLabel(item.lastDate)
      )
    )
  end

  if #values == 0 then
    table.insert(rows, "Nessun luogo registrato.")
  end

  return table.concat(rows, "\n")
end

function widgets.LuoghiDimenticati(n)
  return widget.markdownBlock(
    riepiloghi.renderLuoghiDimenticati(n)
  )
end

function riepiloghi.renderLuoghiUnaVolta(n)
  n = clampN(n, 10)
  local values = {}

  for _, item in ipairs(riepiloghi.placeStats()) do
    if item.count == 1 then
      table.insert(values, item)
    end
  end

  table.sort(values, function(a, b)
    if a.firstDate ~= b.firstDate then
      return a.firstDate > b.firstDate
    end
    return pageLabel(a.page) < pageLabel(b.page)
  end)

  local rows = { "## Una volta soltanto", "" }

  for i = 1, math.min(n, #values) do
    local item = values[i]
    table.insert(
      rows,
      string.format(
        "- %s — %s",
        pageLink(item.name, pageLabel(item.page)),
        dateLabel(item.firstDate)
      )
    )
  end

  if #values == 0 then
    table.insert(rows, "Nessun luogo visitato una sola volta.")
  end

  return table.concat(rows, "\n")
end

function widgets.LuoghiUnaVolta(n)
  return widget.markdownBlock(
    riepiloghi.renderLuoghiUnaVolta(n)
  )
end

function widgets.RiepilogoLuoghi()
  return widget.markdownBlock(
    table.concat({
      riepiloghi.renderLuoghiPiuVisitati(10),
      "",
      riepiloghi.renderLuoghiDimenticati(10),
      "",
      riepiloghi.renderLuoghiUnaVolta(10),
    }, "\n")
  )
end


-- ============================================================
-- VIAGGI
-- ============================================================

function riepiloghi.renderViaggiCasuali(n)
  ensureRandomSeed()
  n = clampN(n, 1)
  local values = riepiloghi.tripStats()
  local rows = { "## Un viaggio da ricordare", "" }

  for _, item in ipairs(randomSample(values, n)) do
    table.insert(
      rows,
      "- " .. pageLink(item.name, item.label)
    )
  end

  if #values == 0 then
    table.insert(rows, "Nessun viaggio registrato.")
  end

  return table.concat(rows, "\n")
end

function widgets.ViaggiCasuali(n)
  return widget.markdownBlock(
    riepiloghi.renderViaggiCasuali(n)
  )
end

function riepiloghi.renderViaggiPiuLunghi(n)
  n = clampN(n, 10)
  local values = riepiloghi.tripStats()

  table.sort(values, function(a, b)
    if a.count ~= b.count then
      return a.count > b.count
    end
    return a.firstDate > b.firstDate
  end)

  local rows = { "## Viaggi con più giornate", "" }

  for i = 1, math.min(n, #values) do
    local item = values[i]
    table.insert(
      rows,
      string.format(
        "- %s — **%d** giorn%s",
        pageLink(item.name, item.label),
        item.count,
        item.count == 1 and "o" or "i"
      )
    )
  end

  if #values == 0 then
    table.insert(rows, "Nessun viaggio registrato.")
  end

  return table.concat(rows, "\n")
end

function widgets.ViaggiPiuLunghi(n)
  return widget.markdownBlock(
    riepiloghi.renderViaggiPiuLunghi(n)
  )
end

function riepiloghi.renderViaggiPiuLuoghi(n)
  n = clampN(n, 10)
  local values = riepiloghi.tripStats()

  table.sort(values, function(a, b)
    if a.placeCount ~= b.placeCount then
      return a.placeCount > b.placeCount
    end
    return a.firstDate > b.firstDate
  end)

  local rows = { "## Viaggi con più luoghi", "" }

  for i = 1, math.min(n, #values) do
    local item = values[i]
    table.insert(
      rows,
      string.format(
        "- %s — **%d** luogh%s",
        pageLink(item.name, item.label),
        item.placeCount,
        item.placeCount == 1 and "o" or "i"
      )
    )
  end

  if #values == 0 then
    table.insert(rows, "Nessun viaggio registrato.")
  end

  return table.concat(rows, "\n")
end

function widgets.ViaggiPiuLuoghi(n)
  return widget.markdownBlock(
    riepiloghi.renderViaggiPiuLuoghi(n)
  )
end

function widgets.RiepilogoViaggi()
  return widget.markdownBlock(
    table.concat({
      riepiloghi.renderViaggiCasuali(1),
      "",
      riepiloghi.renderViaggiPiuLunghi(10),
      "",
      riepiloghi.renderViaggiPiuLuoghi(10),
    }, "\n")
  )
end


-- ============================================================
-- STATISTICHE
-- ============================================================

function riepiloghi.renderStatistiche()
  local entries = riepiloghi.diarioEntries()
  local trips = riepiloghi.tripStats()
  local places = riepiloghi.placeStats()
  local states = {}
  local tags = {}
  local minYear = nil
  local maxYear = nil

  for _, entry in ipairs(entries) do
    minYear = not minYear and entry.year or math.min(minYear, entry.year)
    maxYear = not maxYear and entry.year or math.max(maxYear, entry.year)

    for _, raw in ipairs(list(entry.luoghi)) do
      local target = wikilinkTarget(raw)
      local state = target and target:match("^luoghi/([^/]+)")
      if state then
        states[state] = true
      end
    end

    for _, tag in ipairs(list(entry.tags)) do
      if type(tag) == "string" and tag ~= "" then
        tag = tag:gsub("^#", "")
        tags[tag] = (tags[tag] or 0) + 1
      end
    end
  end

  local stateCount = 0
  for _ in pairs(states) do
    stateCount = stateCount + 1
  end

  local tagValues = {}
  for name, count in pairs(tags) do
    table.insert(tagValues, { name = name, count = count })
  end

  table.sort(tagValues, function(a, b)
    if a.count ~= b.count then
      return a.count > b.count
    end
    return a.name < b.name
  end)

  local rows = {
    "## Statistiche generali",
    "",
    string.format("- **%d** pagine Diario", #entries),
    string.format("- **%d** viaggi", #trips),
    string.format("- **%d** luoghi terminali visitati", #places),
    string.format("- **%d** Stati visitati", stateCount),
  }

  if minYear and maxYear then
    table.insert(
      rows,
      string.format("- periodo coperto: **%d–%d**", minYear, maxYear)
    )
  end

  table.insert(rows, "")
  table.insert(rows, "## Tag più frequenti")
  table.insert(rows, "")

  for i = 1, math.min(10, #tagValues) do
    local item = tagValues[i]
    table.insert(
      rows,
      string.format(
        "- [[tag:%s|#%s]] — **%d**",
        item.name,
        item.name,
        item.count
      )
    )
  end

  if #tagValues == 0 then
    table.insert(rows, "Nessun tag registrato.")
  end

  return table.concat(rows, "\n")
end

function widgets.StatisticheDiario()
  return widget.markdownBlock(
    riepiloghi.renderStatistiche()
  )
end
```

## Implementazione — PhotoGallery

```space-lua
photoGalleryConfig = photoGalleryConfig or {
  webUiBase =
    "https://sb2.fm-nas.net/gallery/",

  apiBase =
    "https://sb2.fm-nas.net/api/gallery/",

  apiBatch =
    "https://sb2.fm-nas.net/api/gallery",

  batchMax =
    64,

  diaryPagePrefix =
    "Diario/",
}


photoGalleryConfig.apiBatch =
  photoGalleryConfig.apiBatch
  or "https://sb2.fm-nas.net/api/gallery"

photoGalleryConfig.batchMax =
  photoGalleryConfig.batchMax
  or 64


photoGalleryInfoCache =
  photoGalleryInfoCache
  or {}



function photoGalleryFullUrl(date)
  local iso = luoghi.isoDate(date)

  if not iso then
    return nil
  end

  return
    photoGalleryConfig.webUiBase
    .. iso
end


function photoGalleryInfo(date)
  date = luoghi.isoDate(date)

  if not date then
    return nil
  end

  if photoGalleryInfoCache[date] ~= nil then
    local cached =
      photoGalleryInfoCache[date]

    return cached ~= false
      and cached
      or nil
  end

  local response =
    net.proxyFetch(
      photoGalleryConfig.apiBase
        .. date
    )

  if not response
    or not response.ok
    or type(response.body) ~= "table"
  then
    photoGalleryInfoCache[date] = false
    return nil
  end

  photoGalleryInfoCache[date] =
    response.body

  return response.body
end


function photoGalleryHasPhotos(date)
  local info =
    photoGalleryInfo(date)

  if not info then
    return false
  end

  return info.exists ~= false
    and tonumber(info.count or 0) > 0
end


function photoGalleryAvailability(dates)
  local result = {}
  local pending = {}
  local seen = {}

  for _, value in ipairs(dates or {}) do
    local date = luoghi.isoDate(value)

    if date and not seen[date] then
      seen[date] = true

      local cached = photoGalleryInfoCache[date]

      if cached ~= nil then
        result[date] =
          cached ~= false
          and cached.exists ~= false
          and tonumber(cached.count or 0) > 0
      else
        table.insert(pending, date)
      end
    end
  end

  local batchMax = tonumber(photoGalleryConfig.batchMax) or 64

  for first = 1, #pending, batchMax do
    local batch = {}

    for i = first, math.min(first + batchMax - 1, #pending) do
      table.insert(batch, pending[i])
    end

    local response = net.proxyFetch(
      photoGalleryConfig.apiBatch
        .. "?dates="
        .. table.concat(batch, ",")
    )

    if response
      and response.ok
      and type(response.body) == "table"
    then
      for _, date in ipairs(batch) do
        local info = response.body[date]

        if type(info) == "table" then
          photoGalleryInfoCache[date] = info
          result[date] =
            info.exists ~= false
            and tonumber(info.count or 0) > 0
        end
      end
    end
  end

  return result
end
```

## Implementazione — Mappe Luoghi e GPX

```space-lua
-- ============================================================
-- DATI LUOGHI
-- ============================================================

local function luogoCoordinate(page)
  local c =
    page and page.coordinate

  if type(c) ~= "table" then
    return nil
  end

  local lat = tonumber(c[1])
  local lon = tonumber(c[2])

  if not lat or not lon then
    return nil
  end

  if lat < -90 or lat > 90
    or lon < -180 or lon > 180
  then
    return nil
  end

  return {
    lat = lat,
    lon = lon
  }
end


local function luogoMarker(page)
  local c =
    luogoCoordinate(page)

  if not c then
    return nil
  end

  return {
    name = tostring(page.name or ""),
    label = luoghi.label(
      page,
      luoghi.basename(
        tostring(page.name or "")
      )
    ),
    lat = c.lat,
    lon = c.lon,
    tipoAmministrativo =
      tostring(
        page.tipoAmministrativo
        or ""
      )
  }
end


local function uniquePages(pages)
  local result = {}
  local seen = {}

  for _, page in ipairs(pages or {}) do
    if page
      and page.name
      and not seen[page.name]
    then
      table.insert(result, page)
      seen[page.name] = true
    end
  end

  return result
end



local function collectLuoghiPages(options)
  options = options or {}

  if options.pages then
    return uniquePages(options.pages)
  end

  if options.luoghi ~= nil then
    return luoghi.resolvePages(
      options.luoghi
    )
  end

  local pageName =
    options.pageName
    or editor.getCurrentPage()

  if not pageName then
    return {}
  end

  if string.startsWith(
    pageName,
    "Viaggi/"
  ) then
    return luoghi.resolvePages(
      widgets.infoViaggioLuoghiList(
        pageName,
        viaggiDiarioInfo(pageName)
      )
    )
  end

  if string.startsWith(
    pageName,
    "luoghi/"
  ) then
    local result = {}

    local current =
      luoghi.getPage(pageName)

    if current then
      table.insert(result, current)
    end

    if options.children ~= false then
      for _, child in ipairs(
        luoghi.children(pageName)
        or {}
      ) do
        table.insert(result, child)
      end
    end

    return uniquePages(result)
  end

  local current =
    luoghi.getPage(pageName)

  if not current then
    return {}
  end

  return luoghi.resolvePages(
    current.luoghi
  )
end


local function pagesToMarkers(pages)
  local result = {}

  for _, page in ipairs(
    uniquePages(pages)
  ) do
    local marker =
      luogoMarker(page)

    if marker then
      table.insert(result, marker)
    end
  end

  return result
end


local function jsString(value)
  return string.format(
    "%q",
    tostring(value or "")
  )
end


local function markersToJavascript(markers)
  local rows = {}

  for _, marker in ipairs(markers) do
    table.insert(
      rows,
      "{"
        .. "name:" .. jsString(marker.name) .. ","
        .. "label:" .. jsString(marker.label) .. ","
        .. "lat:" .. tostring(marker.lat) .. ","
        .. "lon:" .. tostring(marker.lon) .. ","
        .. "tipoAmministrativo:"
        .. jsString(marker.tipoAmministrativo)
        .. "}"
    )
  end

  return "["
    .. table.concat(rows, ",")
    .. "]"
end


local function spaceLuaString(value)
  local text = tostring(value or "")

  text = string.gsub(text, "\\", "\\\\")
  text = string.gsub(text, '"', '\\"')

  return '"' .. text .. '"'
end


-- ============================================================
-- MAPPA LUOGHI STANDALONE
-- ============================================================

function luoghiMap(options)
  options = options or {}

  local pages =
    collectLuoghiPages(options)

  local markers =
    pagesToMarkers(pages)

  if #markers == 0 then
    if options.silentEmpty then
      return nil
    end

    return widget.markdownBlock(
      "_Nessun luogo con coordinate disponibile._"
    )
  end

  local markerData =
    markersToJavascript(markers)

  return widget.sandbox {
    html = [[
      <div style="width:100%;">
        <div id="luoghi-map"
          style="width:100%;height:520px;border-radius:6px;overflow:hidden;">
        </div>
      </div>
    ]],

    script = string.format([==[
      const places = %s;
      let map = null;
      let group = null;

      function escapeHtml(value) {
        return String(value || "")
          .replaceAll("&", "&amp;")
          .replaceAll("<", "&lt;")
          .replaceAll(">", "&gt;")
          .replaceAll('"', "&quot;")
          .replaceAll("'", "&#039;");
      }

      function syncHeight() {
        const body = document.body;
        const html = document.documentElement;

        globalThis.parent.postMessage(
          {
            type: "setHeight",
            height: Math.max(
              body.offsetHeight,
              html.offsetHeight
            )
          },
          "*"
        );
      }

      function normalizedPlaceType(value) {
        return String(value || "")
          .trim()
          .toLowerCase()
          .normalize("NFD")
          .replace(/[\u0300-\u036f]/g, "")
          .replace(/[_-]+/g, " ");
      }

      function placeTypeClass(place) {
        const value =
          normalizedPlaceType(
            place.tipoAmministrativo
          );

        if (value === "comune") {
          return "luogo-comune";
        }

        if (value === "localita") {
          return "luogo-localita";
        }

        if (value === "attrazione") {
          return "luogo-attrazione";
        }

        if (
          value === "area geografica"
          || value === "area"
        ) {
          return "luogo-area-geografica";
        }

        return "luogo-altro";
      }

      function labelIcon(place) {
        return L.divIcon({
          className: "",
          html:
            '<div class="luogo-label-marker '
            + placeTypeClass(place)
            + '">'
            + escapeHtml(place.label)
            + '</div>',
          iconSize: null,
          iconAnchor: [0, 0]
        });
      }

      function fitMap() {
        if (!map || !group) return;

        if (places.length == 1) {
          map.setView(
            [places[0].lat, places[0].lon],
            13
          );
        } else {
          map.fitBounds(
            group.getBounds(),
            {
              padding: [30, 30],
              maxZoom: 14
            }
          );
        }
      }

      async function main() {
        await loadJsByUrl(
          "https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"
        );

        const css =
          document.createElement("link");

        css.rel = "stylesheet";
        css.href =
          "https://unpkg.com/leaflet@1.9.4/dist/leaflet.css";

        document.head.appendChild(css);

        const style =
          document.createElement("style");

        style.textContent = `
          .luogo-label-marker {
            display:inline-block;
            transform:translate(-50%%,-110%%);
            padding:3px 6px;
            border:1px solid;
            border-radius:4px;
            line-height:1.15;
            white-space:nowrap;
            box-shadow:0 1px 2px rgba(0,0,0,.15);
            font-family:sans-serif;
          }

          .luogo-comune {
            color:#174A7E;
            background:#EAF3FB;
            border-color:#6FA4CF;
            font-size:13px;
            font-weight:700;
          }

          .luogo-localita {
            color:#2F6B3A;
            background:#EDF7EF;
            border-color:#7EB98A;
            font-size:12px;
            font-weight:600;
          }

          .luogo-attrazione {
            color:#7A4B00;
            background:#FFF4D8;
            border-color:#D6A94D;
            font-size:11px;
            font-weight:600;
          }

          .luogo-area-geografica {
            color:#65458A;
            background:#F3ECFA;
            border-color:#A98BC5;
            font-size:13px;
            font-weight:700;
          }

          .luogo-altro {
            color:#444;
            background:#F5F5F5;
            border-color:#AAA;
            font-size:12px;
            font-weight:600;
          }
        `;

        document.head.appendChild(style);

        map = L.map("luoghi-map");

        L.tileLayer(
          "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
          {
            maxZoom: 19,
            attribution:
              "&copy; OpenStreetMap contributors"
          }
        ).addTo(map);

        group =
          L.featureGroup()
            .addTo(map);

        for (const place of places) {
          const marker =
            L.marker(
              [place.lat, place.lon],
              {icon: labelIcon(place)}
            )
            .addTo(group);

          const popup =
            document.createElement("div");

          const title =
            document.createElement("strong");

          title.textContent = place.label;
          popup.appendChild(title);

          if (place.tipoAmministrativo) {
            const type =
              document.createElement("div");

            type.textContent =
              place.tipoAmministrativo;

            popup.appendChild(type);
          }

          const button =
            document.createElement("button");

          button.type = "button";
          button.textContent = "Apri pagina";
          button.style.marginTop = "8px";

          button.addEventListener(
            "click",
            async () => {
              await syscall(
                "editor.navigate",
                place.name
              );
            }
          );

          popup.appendChild(button);
          marker.bindPopup(popup);
        }

        fitMap();

        setTimeout(
          () => {
            map.invalidateSize();
            fitMap();
            syncHeight();
          },
          50
        );
      }

      main().catch(error => {
        const target =
          document.getElementById("luoghi-map");

        if (target) {
          target.innerHTML =
            '<div style="padding:12px;font-family:sans-serif;">Errore mappa: '
            + String(error.message || error)
            + '</div>';
        }

        syncHeight();
      });
    ]==], markerData),

    markdown = "Mappa dei luoghi"
  }
end


virtualPage.define {
  pattern = "Mappa/(.+)",

  run = function(sourcePage)
    if not sourcePage
      or sourcePage == ""
    then
      return "_Pagina sorgente non valida._"
    end

    return "${luoghiMap({pageName = "
      .. spaceLuaString(sourcePage)
      .. "})}"
  end
}


-- ============================================================
-- GPX: LOOKUP
-- ============================================================

local function gpxPageDate(pageName)
  if not pageName
    or not string.startsWith(
      pageName,
      "Diario/"
    )
  then
    return nil
  end

  local rows = query[[
    from p = index.pages()
    where p.name == pageName
      and p.date
    select {
      date = p.date
    }
    limit 1
  ]]

  if not rows
    or not rows[1]
  then
    return nil
  end

  return luoghi.isoDate(
    rows[1].date
  )
end


function gpxPathForDate(value)
  local iso =
    luoghi.isoDate(value)

  if not iso then
    return nil
  end

  local compactPrefix =
    "media/" .. string.gsub(iso, "-", "")

  local isoPrefix =
    "media/" .. iso

  local docs = query[[
    from doc = index.documents()
    where
      (
        string.startsWith(
          doc.name,
          compactPrefix
        )
        or
        string.startsWith(
          doc.name,
          isoPrefix
        )
      )
      and string.lower(
        doc.extension or ""
      ) == "gpx"
    order by doc.name
    select {
      name = doc.name
    }
  ]]

  if docs and docs[1] then
    return docs[1].name
  end

  return nil
end


function gpxPathForPage(pageName)
  pageName =
    pageName
    or editor.getCurrentPage()

  local d =
    gpxPageDate(pageName)

  if not d then
    return nil
  end

  return gpxPathForDate(d.iso)
end


function gpxCatalogByDate()
  local docs = query[[
    from doc = index.documents()
    where
      string.startsWith(
        doc.name,
        "media/"
      )
      and string.lower(
        doc.extension or ""
      ) == "gpx"
    order by doc.name
    select {
      name = doc.name
    }
  ]]

  local byDate = {}

  for _, doc in ipairs(docs or {}) do
    local iso =
      string.match(
        doc.name,
        "^media/(%d%d%d%d%-%d%d%-%d%d)"
      )

    if not iso then
      local y, m, d =
        string.match(
          doc.name,
          "^media/(%d%d%d%d)(%d%d)(%d%d)"
        )

      if y then
        iso =
          y .. "-" .. m .. "-" .. d
      end
    end

    if iso
      and not byDate[iso]
    then
      byDate[iso] = doc.name
    end
  end

  return byDate
end




-- ============================================================
-- GPX: LETTURA E RENDERING
-- ============================================================

local function gpxRead(path)
  if not path or path == "" then
    return nil
  end

  local ok, data =
    pcall(
      space.readDocument,
      path
    )

  if not ok or not data then
    return nil
  end

  return data
end


function gpxMap(path, options)
  if type(path) == "table"
    and options == nil
  then
    options = path
    path = nil
  end

  options = options or {}

  if not path then
    path =
      gpxPathForPage(
        options.pageName
        or editor.getCurrentPage()
      )
  end

  if not path then
    if options.silentEmpty then
      return nil
    end

    return widget.markdownBlock(
      "_Nessun file GPX associato alla pagina._"
    )
  end

  local data =
    gpxRead(path)

  if not data then
    if options.silentEmpty then
      return nil
    end

    return widget.markdownBlock(
      "_Impossibile leggere `"
        .. tostring(path)
        .. "`._"
    )
  end

  local gpxBase64 =
    encoding.base64Encode(data)

  return widget.sandbox {
    html = [[
      <div class="gpx-shell">
        <div id="gpx-controls" class="gpx-controls">
          <div class="gpx-presets" role="group" aria-label="Filtro orario">
            <button type="button" data-preset="all" class="active">Tutta</button>
            <button type="button" data-preset="morning">Mattina</button>
            <button type="button" data-preset="afternoon">Pomeriggio</button>
            <button type="button" data-preset="evening">Sera</button>
          </div>
          <div class="gpx-range-wrap">
            <div class="gpx-range-labels">
              <span>Intervallo</span>
              <strong id="gpx-range-value">00:00 – 24:00</strong>
            </div>
            <div class="gpx-dual-range" aria-label="Intervallo orario personalizzato">
              <div id="gpx-range-track" class="gpx-range-track"></div>
              <input id="gpx-range-start" type="range" min="0" max="1440" step="15" value="0" aria-label="Ora iniziale">
              <input id="gpx-range-end" type="range" min="0" max="1440" step="15" value="1440" aria-label="Ora finale">
            </div>
          </div>
        </div>
        <div id="gpx-status" class="gpx-status"></div>
        <div id="gpx-stats" class="gpx-stats"></div>
        <div id="gpx-map" class="gpx-map"></div>
      </div>
    ]],

    script = [[
      const gpxBase64 = "]] ..
      gpxBase64 ..
      [[";

      const TIME_ZONE = "Europe/Rome";
      const GAP_LIMIT_MS = 15 * 60 * 1000;
      const COLORS = {
        morning: "#D1495B",
        afternoon: "#2F6FB0",
        evening: "#7A4FA3",
        custom: "#2A7F7F",
        untimed: "#777777"
      };

      let map = null;
      let trackLayer = null;
      let endpointLayer = null;
      let parsedTrack = null;
      let currentMode = "all";

      const localTimeFormatter =
        new Intl.DateTimeFormat(
          "en-GB",
          {
            timeZone: TIME_ZONE,
            hourCycle: "h23",
            hour: "2-digit",
            minute: "2-digit",
            second: "2-digit"
          }
        );

      function syncHeight() {
        const body = document.body;
        const html = document.documentElement;

        globalThis.parent.postMessage(
          {
            type: "setHeight",
            height: Math.max(
              body.offsetHeight,
              html.offsetHeight
            )
          },
          "*"
        );
      }

      function decodeBase64Utf8(value) {
        const binary = atob(value);
        const bytes =
          new Uint8Array(binary.length);

        for (
          let i = 0;
          i < binary.length;
          i++
        ) {
          bytes[i] =
            binary.charCodeAt(i);
        }

        return new TextDecoder(
          "utf-8"
        ).decode(bytes);
      }

      function escapeHtml(value) {
        return String(value || "")
          .replaceAll("&", "&amp;")
          .replaceAll("<", "&lt;")
          .replaceAll(">", "&gt;")
          .replaceAll('"', "&quot;")
          .replaceAll("'", "&#039;");
      }

      function haversine(a, b) {
        const R = 6371000;
        const rad =
          value => value * Math.PI / 180;

        const dLat = rad(b.lat - a.lat);
        const dLon = rad(b.lon - a.lon);
        const lat1 = rad(a.lat);
        const lat2 = rad(b.lat);

        const h =
          Math.sin(dLat / 2) ** 2
          + Math.cos(lat1)
          * Math.cos(lat2)
          * Math.sin(dLon / 2) ** 2;

        return 2 * R
          * Math.asin(
            Math.min(
              1,
              Math.sqrt(h)
            )
          );
      }

      function localMinutes(date) {
        if (!(date instanceof Date)
          || !Number.isFinite(date.getTime())
        ) {
          return null;
        }

        const parts = {};

        for (const part of
          localTimeFormatter.formatToParts(date)
        ) {
          if (
            part.type === "hour"
            || part.type === "minute"
            || part.type === "second"
          ) {
            parts[part.type] =
              Number(part.value);
          }
        }

        if (
          !Number.isFinite(parts.hour)
          || !Number.isFinite(parts.minute)
        ) {
          return null;
        }

        return (
          parts.hour * 60
          + parts.minute
          + (Number(parts.second) || 0) / 60
        );
      }

      function bandForMinute(minute) {
        if (minute === null) {
          return "untimed";
        }

        if (minute >= 480 && minute < 780) {
          return "morning";
        }

        if (minute >= 780 && minute < 1230) {
          return "afternoon";
        }

        return "evening";
      }

      function parseTrack(xml) {
        const segmentNodes =
          Array.from(
            xml.querySelectorAll("trkseg")
          );

        const segments = [];
        let validPoints = 0;
        let timedPoints = 0;
        let elevationPoints = 0;
        let invalidCoordinatePoints = 0;
        let invalidTimePoints = 0;
        let sourceIndex = 0;

        const effectiveSegments =
          segmentNodes.length > 0
            ? segmentNodes
            : [xml];

        effectiveSegments.forEach(
          (segmentNode, segmentId) => {
            const points = [];
            const nodes =
              Array.from(
                segmentNode.querySelectorAll("trkpt")
              );

            for (const node of nodes) {
              const index = sourceIndex++;
              const lat =
                Number(
                  node.getAttribute("lat")
                );
              const lon =
                Number(
                  node.getAttribute("lon")
                );

              if (
                !Number.isFinite(lat)
                || !Number.isFinite(lon)
                || lat < -90
                || lat > 90
                || lon < -180
                || lon > 180
              ) {
                invalidCoordinatePoints++;
                continue;
              }

              const eleNode =
                node.querySelector("ele");
              const eleRaw =
                eleNode
                  ? Number(eleNode.textContent)
                  : null;
              const ele =
                Number.isFinite(eleRaw)
                  ? eleRaw
                  : null;

              const timeNode =
                node.querySelector("time");
              let time = null;
              let timeMs = null;
              let minute = null;

              if (
                timeNode
                && String(timeNode.textContent || "").trim()
              ) {
                const candidate =
                  new Date(
                    String(timeNode.textContent).trim()
                  );

                if (Number.isFinite(candidate.getTime())) {
                  time = candidate;
                  timeMs = candidate.getTime();
                  minute = localMinutes(candidate);
                  timedPoints++;
                } else {
                  invalidTimePoints++;
                }
              }

              if (ele !== null) {
                elevationPoints++;
              }

              validPoints++;

              points.push({
                lat,
                lon,
                ele,
                time,
                timeMs,
                minute,
                band: bandForMinute(minute),
                segmentId,
                sourceIndex: index
              });
            }

            if (points.length > 0) {
              segments.push(points);
            }
          }
        );

        return {
          segments,
          validPoints,
          timedPoints,
          elevationPoints,
          invalidCoordinatePoints,
          invalidTimePoints
        };
      }

      function isContinuous(a, b) {
        if (!a || !b) return false;

        if (a.segmentId !== b.segmentId) {
          return false;
        }

        if (b.sourceIndex !== a.sourceIndex + 1) {
          return false;
        }

        if (
          a.timeMs !== null
          && b.timeMs !== null
        ) {
          const dt = b.timeMs - a.timeMs;

          if (dt < 0 || dt > GAP_LIMIT_MS) {
            return false;
          }
        }

        return true;
      }

      function minutesInRange(
        minute,
        startMinute,
        endMinute
      ) {
        if (minute === null) return false;

        if (startMinute === 0
          && endMinute === 1440
        ) {
          return true;
        }

        if (startMinute < endMinute) {
          return minute >= startMinute
            && minute < endMinute;
        }

        if (startMinute > endMinute) {
          return minute >= startMinute
            || minute < endMinute;
        }

        return false;
      }

      function presetRange(name) {
        if (name === "morning") {
          return [480, 780];
        }

        if (name === "afternoon") {
          return [780, 1230];
        }

        if (name === "evening") {
          return [1230, 480];
        }

        return [0, 1440];
      }

      function buildVisibleSegments(
        mode,
        startMinute,
        endMinute
      ) {
        const visible = [];

        for (const sourceSegment of
          parsedTrack.segments
        ) {
          let current = [];
          let currentBand = null;
          let previousAccepted = null;

          function flush() {
            if (current.length > 0) {
              visible.push({
                points: current,
                band:
                  mode === "all"
                    ? currentBand
                    : mode === "custom"
                      ? "custom"
                      : mode
              });
            }

            current = [];
            currentBand = null;
            previousAccepted = null;
          }

          for (const point of sourceSegment) {
            let accepted = false;

            if (mode === "all") {
              accepted = true;
            } else {
              accepted =
                minutesInRange(
                  point.minute,
                  startMinute,
                  endMinute
                );
            }

            if (!accepted) {
              flush();
              continue;
            }

            const pointBand =
              mode === "all"
                ? point.band
                : mode;

            if (current.length > 0) {
              const continuous =
                isContinuous(
                  previousAccepted,
                  point
                );
              const bandChanged =
                mode === "all"
                && pointBand !== currentBand;

              if (!continuous) {
                flush();
              } else if (bandChanged) {
                const bridgePoint =
                  previousAccepted;
                flush();
                current = [bridgePoint];
                currentBand = pointBand;
                previousAccepted = bridgePoint;
              }
            }

            if (current.length === 0) {
              currentBand = pointBand;
            }

            current.push(point);
            previousAccepted = point;
          }

          flush();
        }

        return visible;
      }

      function segmentDistance(points) {
        let total = 0;

        for (
          let i = 1;
          i < points.length;
          i++
        ) {
          total +=
            haversine(
              points[i - 1],
              points[i]
            );
        }

        return total;
      }

      function distanceMeters(segments) {
        return segments.reduce(
          (total, segment) =>
            total
            + segmentDistance(segment.points),
          0
        );
      }

      function elevationStats(segments) {
        let ascent = 0;
        let descent = 0;
        let usablePairs = 0;
        let elevationCount = 0;
        let pointCount = 0;

        for (const segment of segments) {
          const points = segment.points;
          pointCount += points.length;
          elevationCount +=
            points.filter(
              point => point.ele !== null
            ).length;

          if (points.length < 2) {
            continue;
          }

          const samples = [];
          let bucketDistance = 0;
          let bucketElevations = [];

          function closeBucket() {
            if (bucketElevations.length === 0) {
              bucketDistance = 0;
              return;
            }

            const sum =
              bucketElevations.reduce(
                (a, b) => a + b,
                0
              );

            samples.push(
              sum / bucketElevations.length
            );

            bucketDistance = 0;
            bucketElevations = [];
          }

          if (points[0].ele !== null) {
            bucketElevations.push(points[0].ele);
          }

          for (
            let i = 1;
            i < points.length;
            i++
          ) {
            bucketDistance +=
              haversine(
                points[i - 1],
                points[i]
              );

            if (points[i].ele !== null) {
              bucketElevations.push(points[i].ele);
            }

            if (bucketDistance >= 150) {
              closeBucket();
            }
          }

          closeBucket();

          for (
            let i = 1;
            i < samples.length;
            i++
          ) {
            const delta =
              samples[i] - samples[i - 1];

            if (Math.abs(delta) <= 5) {
              continue;
            }

            usablePairs++;

            if (delta > 0) {
              ascent += delta;
            } else {
              descent += -delta;
            }
          }
        }

        if (
          pointCount < 2
          || elevationCount / pointCount < 0.90
          || usablePairs === 0
        ) {
          return null;
        }

        return {ascent, descent};
      }

      function timedExtent(segments) {
        const timed = [];

        for (const segment of segments) {
          for (const point of segment.points) {
            if (point.timeMs !== null) {
              timed.push(point);
            }
          }
        }

        if (timed.length === 0) {
          return null;
        }

        timed.sort(
          (a, b) => a.timeMs - b.timeMs
        );

        return {
          first: timed[0],
          last: timed[timed.length - 1],
          durationMs:
            timed.length >= 2
              ? timed[timed.length - 1].timeMs
                - timed[0].timeMs
              : null
        };
      }

      function formatDuration(ms) {
        if (!Number.isFinite(ms) || ms < 0) {
          return null;
        }

        const totalMinutes =
          Math.round(ms / 60000);
        const hours =
          Math.floor(totalMinutes / 60);
        const minutes =
          totalMinutes % 60;

        if (hours > 0) {
          return hours + " h "
            + String(minutes).padStart(2, "0")
            + " min";
        }

        return minutes + " min";
      }

      function formatMinute(value) {
        if (value === 1440) {
          return "24:00";
        }

        const normalized =
          ((value % 1440) + 1440) % 1440;
        const hour =
          Math.floor(normalized / 60);
        const minute =
          normalized % 60;

        return String(hour).padStart(2, "0")
          + ":"
          + String(minute).padStart(2, "0");
      }

      function updateRangeLabel(
        startMinute,
        endMinute
      ) {
        const target =
          document.getElementById(
            "gpx-range-value"
          );

        target.textContent =
          formatMinute(startMinute)
          + " – "
          + formatMinute(endMinute);
      }

      function renderStats(segments) {
        const target =
          document.getElementById(
            "gpx-stats"
          );

        const allPoints =
          segments.reduce(
            (acc, segment) =>
              acc.concat(segment.points),
            []
          );

        if (allPoints.length === 0) {
          target.textContent =
            "Nessun punto nella fascia selezionata.";
          return;
        }

        const distance =
          distanceMeters(segments);
        const elevation =
          elevationStats(segments);
        const extent =
          timedExtent(segments);

        const parts = [
          "Distanza: "
          + (distance / 1000).toFixed(1)
          + " km"
        ];

        if (elevation) {
          parts.push(
            "Dislivello: +"
            + Math.round(elevation.ascent)
            + " m / -"
            + Math.round(elevation.descent)
            + " m"
          );
        } else {
          parts.push("Dislivello: —");
        }

        if (
          extent
          && extent.durationMs !== null
        ) {
          const duration =
            formatDuration(extent.durationMs);

          if (duration) {
            parts.push("Durata: " + duration);
          }
        }

        target.textContent =
          parts.join(" · ");
      }

      function renderStatus() {
        const target =
          document.getElementById("gpx-status");
        const messages = [];

        if (parsedTrack.timedPoints === 0) {
          messages.push(
            "Traccia senza timestamp: filtri orari disabilitati."
          );
        } else if (
          parsedTrack.timedPoints
          < parsedTrack.validPoints
        ) {
          messages.push(
            String(
              parsedTrack.validPoints
              - parsedTrack.timedPoints
            )
            + " punti senza orario sono esclusi dai filtri."
          );
        }

        if (parsedTrack.invalidCoordinatePoints > 0) {
          messages.push(
            parsedTrack.invalidCoordinatePoints
            + " punti con coordinate non valide ignorati."
          );
        }

        if (parsedTrack.invalidTimePoints > 0) {
          messages.push(
            parsedTrack.invalidTimePoints
            + " timestamp non validi ignorati."
          );
        }

        target.textContent =
          messages.join(" ");
        target.style.display =
          messages.length > 0
            ? "block"
            : "none";
      }

      function clearMapLayers() {
        if (trackLayer) {
          trackLayer.remove();
          trackLayer = null;
        }

        if (endpointLayer) {
          endpointLayer.remove();
          endpointLayer = null;
        }
      }

      function drawSegments(segments) {
        clearMapLayers();

        trackLayer =
          L.featureGroup().addTo(map);
        endpointLayer =
          L.featureGroup().addTo(map);

        for (const segment of segments) {
          if (segment.points.length < 2) {
            continue;
          }

          L.polyline(
            segment.points.map(
              point => [point.lat, point.lon]
            ),
            {
              color:
                COLORS[segment.band]
                || COLORS.custom,
              weight: 4,
              opacity: 0.88
            }
          ).addTo(trackLayer);
        }

        const extent =
          timedExtent(segments);
        const allPoints =
          segments.reduce(
            (acc, segment) =>
              acc.concat(segment.points),
            []
          );

        const start =
          extent
            ? extent.first
            : allPoints[0];
        const finish =
          extent
            ? extent.last
            : allPoints[allPoints.length - 1];

        function endpointIcon(kind) {
          const isStart = kind === "start";
          const symbol = isStart ? "▶" : "■";
          const background = isStart
            ? "#238636"
            : "#B42318";

          return L.divIcon({
            className: "gpx-endpoint-icon",
            html:
              '<div style="'
              + 'width:24px;height:24px;'
              + 'border-radius:50%;'
              + 'display:flex;'
              + 'align-items:center;'
              + 'justify-content:center;'
              + 'background:' + background + ';'
              + 'color:#fff;'
              + 'border:2px solid #fff;'
              + 'box-shadow:0 1px 4px rgba(0,0,0,.55);'
              + 'font-size:' + (isStart ? '12px' : '10px') + ';'
              + 'font-weight:700;'
              + 'line-height:1;'
              + '">'
              + symbol
              + '</div>',
            iconSize: [24, 24],
            iconAnchor: [12, 12]
          });
        }

        if (start) {
          L.marker(
            [start.lat, start.lon],
            { icon: endpointIcon("start") }
          )
            .bindTooltip("Partenza")
            .addTo(endpointLayer);
        }

        if (
          finish
          && (
            !start
            || finish !== start
          )
        ) {
          L.marker(
            [finish.lat, finish.lon],
            { icon: endpointIcon("finish") }
          )
            .bindTooltip("Arrivo")
            .addTo(endpointLayer);
        }

        const bounds =
          L.latLngBounds([]);

        for (const point of allPoints) {
          bounds.extend([point.lat, point.lon]);
        }

        if (bounds.isValid()) {
          map.fitBounds(
            bounds,
            {
              padding: [25, 25],
              maxZoom: 16
            }
          );
        }
      }

      function setActivePreset(name) {
        document
          .querySelectorAll(
            "[data-preset]"
          )
          .forEach(button => {
            button.classList.toggle(
              "active",
              button.dataset.preset === name
            );
          });
      }

      function renderSelection(
        mode,
        startMinute,
        endMinute
      ) {
        currentMode = mode;

        const visible =
          buildVisibleSegments(
            mode,
            startMinute,
            endMinute
          );

        drawSegments(visible);
        renderStats(visible);
        updateRangeLabel(
          startMinute,
          endMinute
        );
        updateTrackFill(
          startMinute,
          endMinute
        );

        setTimeout(
          () => {
            map.invalidateSize();
            syncHeight();
          },
          25
        );
      }

      function updateTrackFill(
        startMinute,
        endMinute
      ) {
        const track =
          document.getElementById(
            "gpx-range-track"
          );
        const startPct =
          startMinute / 1440 * 100;
        const endPct =
          endMinute / 1440 * 100;

        if (startMinute <= endMinute) {
          track.style.background =
            "linear-gradient(to right, #ddd 0%, #ddd "
            + startPct
            + "%, #777 "
            + startPct
            + "%, #777 "
            + endPct
            + "%, #ddd "
            + endPct
            + "%, #ddd 100%)";
        } else {
          track.style.background =
            "linear-gradient(to right, #777 0%, #777 "
            + endPct
            + "%, #ddd "
            + endPct
            + "%, #ddd "
            + startPct
            + "%, #777 "
            + startPct
            + "%, #777 100%)";
        }
      }

      function applyPreset(name) {
        const [startMinute, endMinute] =
          presetRange(name);
        const startInput =
          document.getElementById(
            "gpx-range-start"
          );
        const endInput =
          document.getElementById(
            "gpx-range-end"
          );

        startInput.value =
          String(startMinute);
        endInput.value =
          String(endMinute);
        setActivePreset(name);

        renderSelection(
          name,
          startMinute,
          endMinute
        );
      }

      function bindControls() {
        const controls =
          document.getElementById(
            "gpx-controls"
          );
        const startInput =
          document.getElementById(
            "gpx-range-start"
          );
        const endInput =
          document.getElementById(
            "gpx-range-end"
          );

        if (parsedTrack.timedPoints === 0) {
          controls.classList.add("disabled");
          controls
            .querySelectorAll("button,input")
            .forEach(control => {
              control.disabled = true;
            });
          return;
        }

        controls
          .querySelectorAll("[data-preset]")
          .forEach(button => {
            button.addEventListener(
              "click",
              () => applyPreset(
                button.dataset.preset
              )
            );
          });

        function customSelection() {
          const startMinute =
            Number(startInput.value);
          const endMinute =
            Number(endInput.value);

          setActivePreset(null);

          renderSelection(
            "custom",
            startMinute,
            endMinute
          );
        }

        startInput.addEventListener(
          "input",
          customSelection
        );
        endInput.addEventListener(
          "input",
          customSelection
        );
      }

      function installStyles() {
        const style =
          document.createElement("style");

        style.textContent = `
          html,body { margin:0; padding:0; }
          .gpx-shell {
            width:100%;
            font-family:sans-serif;
          }
          .gpx-controls {
            margin:0 0 8px 0;
          }
          .gpx-controls.disabled {
            opacity:.55;
          }
          .gpx-presets {
            display:flex;
            flex-wrap:wrap;
            gap:6px;
            margin-bottom:8px;
          }
          .gpx-presets button {
            border:1px solid #aaa;
            border-radius:5px;
            background:#fff;
            padding:4px 10px;
            cursor:pointer;
            font:600 12px sans-serif;
          }
          .gpx-presets button.active {
            background:#e8e8e8;
            border-color:#666;
          }
          .gpx-presets button[data-preset=morning] {
            border-left:4px solid #E69F00;
          }
          .gpx-presets button[data-preset=afternoon] {
            border-left:4px solid #2F6FB0;
          }
          .gpx-presets button[data-preset=evening] {
            border-left:4px solid #7A4FA3;
          }
          .gpx-range-wrap {
            max-width:720px;
          }
          .gpx-range-labels {
            display:flex;
            justify-content:space-between;
            align-items:center;
            gap:12px;
            margin-bottom:5px;
            font-size:12px;
          }
          .gpx-dual-range {
            position:relative;
            height:28px;
          }
          .gpx-range-track {
            position:absolute;
            left:0;
            right:0;
            top:12px;
            height:4px;
            border-radius:3px;
            background:#777;
          }
          .gpx-dual-range input[type=range] {
            position:absolute;
            left:0;
            top:0;
            width:100%;
            height:28px;
            margin:0;
            background:transparent;
            pointer-events:none;
            -webkit-appearance:none;
            appearance:none;
          }
          .gpx-dual-range input[type=range]::-webkit-slider-runnable-track {
            height:4px;
            background:transparent;
          }
          .gpx-dual-range input[type=range]::-moz-range-track {
            height:4px;
            background:transparent;
          }
          .gpx-dual-range input[type=range]::-webkit-slider-thumb {
            width:16px;
            height:16px;
            margin-top:-6px;
            border:1px solid #555;
            border-radius:50%;
            background:#fff;
            pointer-events:auto;
            -webkit-appearance:none;
            cursor:pointer;
          }
          .gpx-dual-range input[type=range]::-moz-range-thumb {
            width:16px;
            height:16px;
            border:1px solid #555;
            border-radius:50%;
            background:#fff;
            pointer-events:auto;
            cursor:pointer;
          }
          .gpx-status {
            display:none;
            margin:0 0 7px 0;
            padding:6px 8px;
            border:1px solid #d2b36b;
            border-radius:5px;
            background:#fff8df;
            font-size:12px;
          }
          .gpx-stats {
            margin:0 0 8px 0;
            font-size:.95em;
          }
          .gpx-map {
            width:100%;
            height:520px;
            border-radius:6px;
            overflow:hidden;
          }
        `;

        document.head.appendChild(style);
      }

      function showFatalError(message) {
        const controls =
          document.getElementById(
            "gpx-controls"
          );
        const stats =
          document.getElementById(
            "gpx-stats"
          );
        const target =
          document.getElementById(
            "gpx-map"
          );

        if (controls) {
          controls.style.display = "none";
        }
        if (stats) {
          stats.style.display = "none";
        }
        if (target) {
          target.innerHTML =
            '<div style="padding:12px;font-family:sans-serif;">⚠ '
            + escapeHtml(message)
            + '</div>';
        }
      }

      async function main() {
        installStyles();

        await loadJsByUrl(
          "https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"
        );

        const css =
          document.createElement("link");

        css.rel = "stylesheet";
        css.href =
          "https://unpkg.com/leaflet@1.9.4/dist/leaflet.css";

        document.head.appendChild(css);

        const xmlText =
          decodeBase64Utf8(gpxBase64);
        const xml =
          new DOMParser()
            .parseFromString(
              xmlText,
              "application/xml"
            );

        if (xml.querySelector("parsererror")) {
          throw new Error("GPX XML non valido");
        }

        parsedTrack = parseTrack(xml);

        if (parsedTrack.validPoints === 0) {
          throw new Error(
            "Nessun punto traccia valido nel GPX"
          );
        }

        map = L.map("gpx-map");

        L.tileLayer(
          "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
          {
            maxZoom: 19,
            attribution:
              "&copy; OpenStreetMap contributors"
          }
        ).addTo(map);

        renderStatus();
        bindControls();
        applyPreset("all");

        setTimeout(
          () => {
            map.invalidateSize();
            syncHeight();
          },
          50
        );
      }

      main().catch(error => {
        showFatalError(
          "Impossibile leggere la traccia GPX: "
          + String(error.message || error)
        );
        syncHeight();
      });
    ]],

    markdown = "Mappa del percorso GPX"
  }
end


virtualPage.define {
  pattern = "GPX/(.+)",

  run = function(isoDate)
    if not string.match(
      isoDate,
      "^%d%d%d%d%-%d%d%-%d%d$"
    ) then
      return "_Data non valida._"
    end

    local path =
      gpxPathForDate(isoDate)

    if not path then
      return "_Nessun file GPX associato._"
    end

    return "${gpxMap("
      .. spaceLuaString(path)
      .. ")}"
  end
}
```
