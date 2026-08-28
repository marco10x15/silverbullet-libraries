---
name: "Library/MG/Mio_Diario"
tags: meta/library
description: "Utility per gestire, navigare e aggregare il Diario personale in SilverBullet."
version: "1.06"
versionDate: 2026-08-28
pageDecoration.prefix: "📔 "
---

# 📔 Mio Diario

**Mio Diario** è la raccolta di utility per gestire, navigare e aggregare il Mio Diario con SilverBullet.

**Versione:** 1.06 — 28.08.2026

La libreria mantiene le pagine Markdown come fonte primaria dei dati e costruisce dinamicamente navigazione, relazioni geografiche, viaggi e riepiloghi tramite gli indici di SilverBullet.

La struttura principale dello Space utilizzata dalla libreria è:

* `Diario/`
  * pagine giornaliere del Diario;
* `luoghi/`
  * `luoghi/Stato/Suddivisione primaria/Comune/Località/...`;
* `Viaggi/`
  * pagine descrittive dei viaggi;
* `riepiloghi/`
  * pagine destinate ai riepiloghi periodici e annuali.

## Struttura dati attesa

### Pagine Diario

Le pagine `Diario/...` utilizzano il frontmatter come fonte dei dati strutturati.

```yaml
---
date: 2026-08-26
displayName: Titolo della giornata
description: Una breve descrizione opzionale
Viaggio: "[[Viaggi/Nome viaggio]]"
luoghi:
  - "[[luoghi/ITA/21/Torino]]"
  - "[[luoghi/ITA/21/Torino/Mole Antonelliana]]"
galleriafoto:
  - "[[...]]"
tags:
  - esempio
---
```

Campi utilizzati dalla libreria:

* `date` — data della pagina Diario;
* `displayName` — titolo canonico indicizzato della pagina Diario; normalmente coincide con il primo H1;
* `description` — descrizione opzionale;
* `Viaggio` — wikilink opzionale a una singola pagina `Viaggi/...`;
* `luoghi` — lista opzionale di wikilink a pagine `luoghi/...`;
* `galleriafoto` — lista opzionale di riferimenti alle gallerie fotografiche;
* `tags` — gestiti direttamente dal rendering del frontmatter di SilverBullet.

L'attributo personalizzato `title` non è più richiesto nelle pagine `Diario/`.

### Pagine Luoghi

La gerarchia geografica principale è ricavata direttamente dal path:

```text
luoghi/
  Stato/
    Suddivisione primaria/
      Comune/
        Località/
          Sotto-località/...
```

Il path costituisce la relazione geografica primaria e non viene duplicato in attributi `parent`.

`divisioneIntermedia` conserva il codice ISO 3166-2 di un livello amministrativo non materializzato nella gerarchia fisica delle pagine. La relativa pagina viene costruita dinamicamente come Virtual Page `geo:divisione:<codice>`.

## Main Features

* **Navigazione del Diario** — inserisce in testa a ogni pagina Diario titolo, navigazione precedente/successiva e, quando presenti, descrizione, viaggio, luoghi e galleria fotografica.
* **Navigazione geografica** — trasforma `luoghi.md` nell'ingresso della gerarchia geografica, raggruppando gli Stati visitati per continente e mostrando progressivamente suddivisioni, Comuni e Località.
* **Divisioni amministrative virtuali** — ricostruisce Province, Dipartimenti e altre divisioni intermedie tramite `divisioneIntermedia` e Virtual Pages `geo:divisione:*`.
* **Elenco Luoghi** — mostra in un widget dedicato la struttura geografica del luogo corrente: padre, eventuale divisione intermedia e figli diretti.
* **Siamo stati qui** — mostra in widget separati i Viaggi e le giornate del Diario direttamente associate al luogo corrente o ai suoi discendenti.
* **Info Viaggio** — nelle pagine `Viaggi/...` ricostruisce cronologicamente i luoghi visitati e le relative pagine Diario.
* **Aggregazione gerarchica dei luoghi** — una giornata associata a una sotto-località viene automaticamente considerata anche appartenente ai suoi luoghi antenati.
* **Integrazione ISO 3166-2** — utilizza il dataset `iso-codes` per risolvere dinamicamente nome e tipo delle divisioni intermedie, mantenendo nel Markdown soltanto il codice.

## Config Example & Defaults

```space-lua
config.set("std.widgets.linkedInfoLuoghi.enabled", true)
config.set("std.widgets.siamoStatiQui.enabled", true)
config.set("std.widgets.infoViaggio.enabled", true)
config.set("std.widgets.wTopDiario.enabled", true)
```

La sorgente ISO 3166-2 può essere sovrascritta, ma normalmente non è necessario:

```space-lua
config.set(
  "luoghi.iso3166_2.url",
  "https://salsa.debian.org/iso-codes-team/iso-codes/-/raw/main/data/iso_3166-2.json"
)
```

> **warning** La libreria considera normalizzate le pagine `Diario/...`: `displayName` è il titolo indicizzato, `Viaggio` deve essere un wikilink nel frontmatter e `luoghi` una lista di wikilink.

> **warning** Le funzioni `page.nome()`, `page.prec()` e `page.succ()` devono essere disponibili nello Space.

> **warning** La risoluzione descrittiva di `divisioneIntermedia` utilizza una risorsa Internet esterna. Se il dataset ISO non è raggiungibile, la libreria continua a funzionare utilizzando il codice ISO come fallback.

> **warning** La correttezza delle aggregazioni geografiche dipende dalla coerenza dei path `luoghi/...` e degli attributi `divisioneIntermedia` delle pagine Markdown.

> **warning** Il widget **Siamo stati qui** usa direttamente l'attributo indicizzato `luoghi` delle pagine Diario. Una visita registrata soltanto sul Comune viene mostrata sul Comune, ma non viene attribuita automaticamente a una sua Località. Una visita registrata sulla Località viene invece inclusa anche nelle pagine dei suoi antenati.

## Implementazione

```space-lua
luoghi = luoghi or {}
widgets = widgets or {}


-- ============================================================
-- FUNZIONI GENERALI
-- ============================================================

function luoghi.hasString(value)
  return type(value) == "string" and value ~= ""
end

function luoghi.hasList(value)
  return type(value) == "table" and #value > 0
end

function luoghi.basename(pageName)
  return string.match(pageName, "([^/]+)$") or pageName
end

function luoghi.parentName(pageName)
  return string.match(pageName, "^(.*)/[^/]+$")
end

-- Etichetta di una pagina Luogo.
-- La semantica dei Luoghi resta distinta da quella del Diario.
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
      tags = p.tags,
      wikipedia = p.wikipedia
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
      string.sub(p.name, #pageName + 2),
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
      tags = p.tags,
      wikipedia = p.wikipedia
    }
  ]]
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

-- Verifica se un elenco di wikilink contiene il luogo indicato
-- oppure uno dei suoi discendenti.
function luoghi.contieneLuogo(
  valori,
  pageName
)
  if not luoghi.hasList(valori)
    or not luoghi.hasString(pageName)
  then
    return false
  end

  local prefix = pageName .. "/"

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

-- Etichetta canonica di una pagina Diario.
-- displayName è la sola sorgente titolo indicizzata; il basename
-- resta il fallback per pagine anomale o non ancora normalizzate.
function luoghi.diarioLabel(info)
  if luoghi.hasString(info.displayName) then
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

  if not children or #children == 0 then
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

  local text = "## Paesi visitati\n"

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

  local response = net.proxyFetch(
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
    return prefisso .. divisione.name
  end

  return divisione.name or code
end

function luoghi.divisioneLink(code)
  return string.format(
    "[[geo:divisione:%s|%s]]",
    code,
    luoghi.divisioneLabel(code)
  )
end


-- ============================================================
-- VIRTUAL PAGE DIVISIONE INTERMEDIA
-- ============================================================

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

  for _, p in ipairs(pages) do
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

    for _, parent in ipairs(parents) do
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
-- PRESENTAZIONE E NAVIGAZIONE PAGINE LUOGHI
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
  elseif luoghi.hasList(p.tags) then
    for _, tag in ipairs(p.tags) do
      if luoghi.hasString(tag) then
        table.insert(
          elementi,
          "#" .. tag
        )
      end
    end
  end

  if luoghi.hasString(p.wikipedia) then
    table.insert(
      elementi,
      "[wikipedia]("
        .. p.wikipedia
        .. ")"
    )
  end

  return table.concat(
    elementi,
    " "
  )
end

function luoghi.renderStato(p)
  local text =
    luoghi.renderTipo(p)

  if luoghi.hasString(p.continente) then
    text = text
      .. "\n\nSi trova in **"
      .. p.continente
      .. "**."
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

    text = text
      .. "\n\nIn cui ci sono "
      .. #children
      .. " suddivisioni: "
      .. table.concat(links, ", ")
  end

  return text
end

function luoghi.renderSuddivisione(p)
  local text =
    luoghi.renderTipo(p)

  local children =
    luoghi.children(p.name)

  local divisioni = {}
  local divisioniSeen = {}
  local altriLuoghi = {}

  for _, child in ipairs(children) do
    if luoghi.hasString(
      child.divisioneIntermedia
    ) then
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

  if #divisioni > 0 then
    local links = {}

    for _, code in ipairs(divisioni) do
      table.insert(
        links,
        luoghi.divisioneLink(code)
      )
    end

    text = text
      .. "\n\nIn cui sono rappresentate "
      .. #divisioni
      .. " divisioni intermedie: "
      .. table.concat(links, ", ")
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

    if #divisioni > 0 then
      text = text
        .. "\n\nAltri luoghi direttamente nella suddivisione: "
    else
      text = text
        .. "\n\nIn cui ci sono "
        .. #altriLuoghi
        .. " luoghi: "
    end

    text = text
      .. table.concat(links, ", ")
  end

  return text
end

function luoghi.renderLuogo(p)
  local text =
    luoghi.renderTipo(p)

  local parentName =
    luoghi.parentName(p.name)

  local parent = nil

  if parentName then
    parent =
      luoghi.getPage(parentName)
  end

  local frase =
    luoghi.pageLink(p)

  if parent then
    frase = frase
      .. " che si trova in "
      .. luoghi.pageLink(parent)
  end

  if luoghi.hasString(
    p.divisioneIntermedia
  ) then
    frase = frase
      .. ", appartiene a "
      .. luoghi.divisioneLink(
        p.divisioneIntermedia
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

    frase = frase
      .. " e include: "
      .. table.concat(links, ", ")
  end

  if text ~= "" then
    text = text .. "\n\n"
  end

  return text .. frase
end

function luoghi.withBreadcrumb(
  pageName,
  text
)
  if not page
    or not page.breadcrumb
  then
    return text
  end

  local trail =
    page.breadcrumb(
      pageName,
      false
    )

  if not luoghi.hasString(trail) then
    return text
  end

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
    text =
      luoghi.renderStato(p)
  elseif #parts == 3 then
    text =
      luoghi.renderSuddivisione(p)
  else
    text =
      luoghi.renderLuogo(p)
  end

  return luoghi.withBreadcrumb(
    pageName,
    text
  )
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
  local seenViaggi = {}

  for _, info in ipairs(diarioInfo) do
    local viaggioTarget =
      luoghi.wikilinkTarget(
        info.Viaggio
      )

    if viaggioTarget
      and not seenViaggi[viaggioTarget]
    then
      local link =
        luoghi.viaggioLink(
          info.Viaggio
        )

      if link then
        table.insert(
          viaggi,
          link
        )
      end

      seenViaggi[viaggioTarget] = true
    end
  end

  if #viaggi == 0 then
    return ""
  end

  return "## 🧭 In questi viaggi\n\n"
    .. table.concat(
      viaggi,
      "\n"
    )
end

function luoghi.renderGiorniPerLuogo(
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

  local giorni = {}

  for _, info in ipairs(diarioInfo) do
    local titolo =
      luoghi.diarioLabel(info)

    local giorno =
      date.format(info.date)

    table.insert(
      giorni,
      string.format(
        "**[[%s|%s]]** %s",
        info.page,
        giorno,
        titolo
      )
    )
  end

  if #giorni == 0 then
    return ""
  end

  return "## In questi giorni\n\n"
    .. table.concat(
      giorni,
      "\n"
    )
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
    string.split(
      pageName,
      "/"
    )

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

  return luoghi.renderGiorniPerLuogo(
    pageName,
    diarioInfo
  )
end

-- Wrapper mantenuto per compatibilità con gli usi precedenti.
function widgets.siamoStatiQui(pageName)
  pageName =
    pageName
    or editor.getCurrentPage()

  if not luoghi.siamoStatiQuiEnabled(
    pageName
  ) then
    return ""
  end

  local diarioInfo =
    luoghi.diarioPerLuogo(pageName)

  local sezioni = {}

  local viaggiText =
    luoghi.renderViaggiPerLuogo(
      pageName,
      diarioInfo
    )

  if viaggiText ~= "" then
    table.insert(
      sezioni,
      viaggiText
    )
  end

  local giorniText =
    luoghi.renderGiorniPerLuogo(
      pageName,
      diarioInfo
    )

  if giorniText ~= "" then
    table.insert(
      sezioni,
      giorniText
    )
  end

  return table.concat(
    sezioni,
    "\n\n"
  )
end


-- ============================================================
-- INFO VIAGGIO
-- ============================================================

function contieneViaggio(
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
      and contieneViaggio(
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

function widgets.infoViaggioLuoghi(
  pageName,
  diarioInfo
)
  diarioInfo =
    diarioInfo
    or viaggiDiarioInfo(pageName)

  if not diarioInfo
    or #diarioInfo == 0
  then
    return ""
  end

  local luoghiVisitati = {}
  local seenLuoghi = {}

  for _, info in ipairs(diarioInfo) do
    if luoghi.hasList(info.luoghi) then
      for _, luogo in ipairs(
        info.luoghi
      ) do
        local target =
          luoghi.wikilinkTarget(luogo)
          or luogo

        if not seenLuoghi[target] then
          table.insert(
            luoghiVisitati,
            luogo
          )

          seenLuoghi[target] = true
        end
      end
    end
  end

  local text =
    "## Luoghi visitati\n"

  if #luoghiVisitati > 0 then
    text = text
      .. table.concat(
        luoghiVisitati,
        ", "
      )
      .. "\n"
  else
    text = text
      .. "Nessun luogo registrato.\n"
  end

  return text
end

function widgets.infoViaggioDiario(
  pageName,
  diarioInfo
)
  diarioInfo =
    diarioInfo
    or viaggiDiarioInfo(pageName)

  if not diarioInfo
    or #diarioInfo == 0
  then
    return ""
  end

  local giorni = {}

  for _, info in ipairs(diarioInfo) do
    local giorno =
      date.format(info.date)

    local titolo =
      luoghi.diarioLabel(info)

    local riga =
      string.format(
        "[[%s|%s]] %s",
        info.page,
        giorno,
        titolo
      )

    if luoghi.hasList(info.luoghi) then
      riga = riga
        .. " ("
        .. table.concat(
          info.luoghi,
          ", "
        )
        .. ")"
    end

    table.insert(
      giorni,
      riga
    )
  end

  return "## "
    .. #diarioInfo
    .. " pagine del diario per questo viaggio\n\n"
    .. table.concat(
      giorni,
      "\n"
    )
    .. "\n"
end

function widgets.infoViaggio(pageName)
  pageName =
    pageName
    or editor.getCurrentPage()

  local diarioInfo =
    viaggiDiarioInfo(pageName)

  if not diarioInfo
    or #diarioInfo == 0
  then
    return ""
  end

  local sezioni = {}

  local luoghiText =
    widgets.infoViaggioLuoghi(
      pageName,
      diarioInfo
    )

  if luoghiText ~= "" then
    table.insert(
      sezioni,
      luoghiText
    )
  end

  local diarioText =
    widgets.infoViaggioDiario(
      pageName,
      diarioInfo
    )

  if diarioText ~= "" then
    table.insert(
      sezioni,
      diarioText
    )
  end

  return table.concat(
    sezioni,
    "\n"
  )
end


-- ============================================================
-- ATTIVAZIONE WIDGET
-- ============================================================

if config.get(
  "std.widgets.linkedInfoLuoghi.enabled"
) then
  event.listen {
    name = "hooks:renderBottomWidgets",

    run = function(e)
      local pageName =
        editor.getCurrentPage()

      if pageName ~= "luoghi"
        and not string.startsWith(
          pageName,
          "luoghi/"
        )
      then
        return
      end

      local text =
        widgets.linkedInfoLuoghi(
          pageName
        )

      if text ~= "" then
        return widget.markdown(text)
      end
    end
  }
end

-- I due widget Siamo stati qui condividono la stessa query.
if config.get(
  "std.widgets.siamoStatiQui.enabled"
) then
  event.listen {
    name = "hooks:renderBottomWidgets",

    run = function(e)
      local pageName =
        editor.getCurrentPage()

      if not luoghi.siamoStatiQuiEnabled(
        pageName
      ) then
        return
      end

      local diarioInfo =
        luoghi.diarioPerLuogo(
          pageName
        )

      luoghi._siamoStatiQuiPending = {
        pageName = pageName,
        diarioInfo = diarioInfo
      }

      local text =
        widgets.siamoStatiQuiViaggi(
          pageName,
          diarioInfo
        )

      if text ~= "" then
        return widget.markdown(text)
      end
    end
  }

  event.listen {
    name = "hooks:renderBottomWidgets",

    run = function(e)
      local pageName =
        editor.getCurrentPage()

      if not luoghi.siamoStatiQuiEnabled(
        pageName
      ) then
        return
      end

      local pending =
        luoghi._siamoStatiQuiPending

      local diarioInfo = nil

      if pending
        and pending.pageName == pageName
      then
        diarioInfo =
          pending.diarioInfo
      else
        diarioInfo =
          luoghi.diarioPerLuogo(
            pageName
          )
      end

      luoghi._siamoStatiQuiPending = nil

      local text =
        widgets.siamoStatiQuiGiorni(
          pageName,
          diarioInfo
        )

      if text ~= "" then
        return widget.markdown(text)
      end
    end
  }
end

if config.get(
  "std.widgets.infoViaggio.enabled"
) then
  event.listen {
    name = "hooks:renderBottomWidgets",

    run = function(e)
      local pageName =
        editor.getCurrentPage()

      if not string.startsWith(
        pageName,
        "Viaggi/"
      ) then
        return
      end

      local text =
        widgets.infoViaggio(
          pageName
        )

      if text ~= "" then
        return widget.markdown(text)
      end
    end
  }
end


-- ============================================================
-- WIDGET SUPERIORE PAGINA DIARIO
-- ============================================================

if config.get(
  "std.widgets.wTopDiario.enabled",
  true
) then
  event.listen {
    name = "hooks:renderTopWidgets",

    run = function(e)
      return wTopDiario()
    end
  }
end

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

  local pages = query[[
    from p = index.pages()
    where p.name == pageName
    select {
      displayName = p.displayName,
      description = p.description,
      Viaggio = p.Viaggio,
      luoghi = p.luoghi,
      galleriafoto = p.galleriafoto
    }
    limit 1
  ]]

  local p = pages[1] or {}

  local titolo =
    luoghi.hasString(p.displayName)
    and p.displayName
    or page.nome(pageName)

  local righe = {
    string.format(
      "# [[%s|⬅️]]%s[[%s|➡️]]",
      page.prec(pageName),
      titolo,
      page.succ(pageName)
    )
  }

  if luoghi.hasString(p.description) then
    table.insert(
      righe,
      "📒 " .. p.description
    )
  end

  if luoghi.hasString(p.Viaggio) then
    table.insert(
      righe,
      "🧭 " .. p.Viaggio
    )
  end

  if luoghi.hasList(p.luoghi) then
    table.insert(
      righe,
      "📍 "
        .. table.concat(
          p.luoghi,
          ", "
        )
    )
  end

  if luoghi.hasList(p.galleriafoto) then
    table.insert(
      righe,
      "📷 "
        .. table.concat(
          p.galleriafoto,
          ", "
        )
    )
  end

  return table.concat(
    righe,
    "\n"
  ) .. "\n"
end
```

## Modifiche versione 1.06

* `displayName` diventa il titolo canonico indicizzato delle pagine `Diario/`.
* Rimosso `p.title` da tutte le query della libreria rivolte al Diario.
* `luoghi.diarioLabel()` usa `displayName` con fallback al nome pagina.
* `wTopDiario()` usa direttamente `p.displayName`, con fallback a `page.nome()`.
* `Siamo stati qui` e `Info Viaggio` non dipendono più dall'attributo `title` né da `page.title()`.
* La semantica di `title` nelle pagine `luoghi/` resta invariata.
* Nessuna scansione aggiuntiva del Markdown e nessun nuovo stato persistente.

## Modifiche versione 1.05

* `Siamo stati qui` viene **diviso in due bottom widget distinti**:
  * `🧭 In questi viaggi`;
  * `In questi giorni`.
* `luoghi.diarioPerLuogo()` non viene ricalcolata due volte: il primo listener salva temporaneamente il dataset in memoria e il secondo lo consuma.
* Nessuna cache persistente viene introdotta: il dato temporaneo serve soltanto durante il rendering corrente.
* `widgets.siamoStatiQui()` rimane disponibile come wrapper di compatibilità e continua a produrre entrambe le sezioni insieme quando viene richiamato manualmente.

## Modifiche versione 1.04

* `Siamo stati qui` viene esteso anche al **primo livello sotto `luoghi/`**, quindi alle pagine Stato come `luoghi/ITA`.
* La pagina radice `luoghi` rimane esclusa.
* La query resta invariata: `index.subPages("Diario")` + filtro sull'attributo indicizzato `p.luoghi`.
* L'unico impatto atteso è l'aumento del numero di giornate e Viaggi da renderizzare per Stati molto visitati; non vengono introdotte query aggiuntive.

## Modifiche versione 1.03

* `Siamo stati qui` viene esteso dal livello Comune/Città al livello **suddivisione primaria**, quindi anche pagine come `luoghi/ITA/55` mostrano Viaggi e giornate relativi all'intero ramo geografico.
* La query resta basata su `index.subPages("Diario")` e sull'attributo indicizzato `p.luoghi`: il costo di selezione è sostanzialmente lo stesso; può aumentare invece il numero di risultati da renderizzare sulle suddivisioni molto frequentate.
* Il breadcrumb viene inserito direttamente nella scheda geografica `linkedInfoLuoghi`.
* La scheda usa `page.breadcrumb(pageName, false)` fornito da `PageNavigation.md` (versione 1.02); se la funzione non è disponibile continua a funzionare senza breadcrumb.
* Il breadcrumb non viene aggiunto alla pagina radice `luoghi`.

## Ottimizzazioni versione 1.02

Questa revisione completa la seconda fase di pulizia delle query.

* `renderDivisione()` non esegue più `luoghi.getPage()` per ogni parent: raccoglie prima i parent distinti e li risolve con una sola query su `index.subPages("luoghi")`.
* `wTopDiario()` usa prima `p.title` e `p.displayName`, già restituiti dall'indice; `page.title()` rimane soltanto come fallback per le pagine prive di entrambi.
* `wTopDiario()` passa esplicitamente `pageName` a `page.prec()` e `page.succ()`, rendendo la navigazione indipendente dalla pagina corrente implicita e compatibile con `PageNavigation.md` (versione 1.01).
* Non vengono introdotte cache persistenti, nuovi attributi o letture aggiuntive del Markdown.

## Ottimizzazioni versione 1.01

Questa revisione interviene sui percorsi più costosi senza modificare il modello Markdown.

* `Siamo stati qui` non usa più `index.relations("luoghi")` né una seconda query sulle pagine: parte direttamente da `index.subPages("Diario")` e filtra `p.luoghi`.
* `Info Viaggio` usa `index.subPages("Diario")` ed esegue una sola query condivisa dalle sezioni **Luoghi visitati** e **Pagine Diario**.
* Rimossi i lookup `page.title()` dai loop: i titoli usano `p.title` → `p.displayName` → nome pagina.
* `renderDivisione()` usa `index.subPages("luoghi")`.
* Rimossi i `limit 500` dalle query di produzione interessate.

Non vengono introdotti nuovi attributi persistenti né letture del Markdown.

## Navigazione con tastiera

### Compatibilità browser

In Microsoft Edge per Windows:

* `Alt + ←` = **browser indietro**;
* `Alt + →` = **browser avanti**.

Gli shortcut già riservati dal browser non sono sempre sovrascrivibili in modo affidabile.

```space-lua
command.define {
  name = "Diario: Pagina precedente",
  key = "Alt-ArrowLeft",

  run = function()
    local pageName =
      editor.getCurrentPage()

    if string.startsWith(
      pageName,
      "Diario/"
    ) then
      local target =
        page.prec()

      if target
        and target ~= ""
      then
        editor.open(target)
      end

      return
    end

    editor.goHistory(-1)
  end
}

command.define {
  name = "Diario: Pagina successiva",
  key = "Alt-ArrowRight",

  run = function()
    local pageName =
      editor.getCurrentPage()

    if string.startsWith(
      pageName,
      "Diario/"
    ) then
      local target =
        page.succ()

      if target
        and target ~= ""
      then
        editor.open(target)
      end

      return
    end

    editor.goHistory(1)
  end
}
```

## Verifica diagnostica dei luoghi nel Diario

Se una pagina Luogo non visualizza **Siamo stati qui**, verificare che le pagine `Diario/...` interessate abbiano l'attributo frontmatter `luoghi` indicizzato con il path atteso.

La ricerca utilizza direttamente `p.luoghi` da `index.subPages("Diario")`.

## Space Style (Optional Theming Overrides)

Non sono previste personalizzazioni CSS specifiche. La sezione resta disponibile per eventuali miglioramenti successivi dell'interfaccia.
