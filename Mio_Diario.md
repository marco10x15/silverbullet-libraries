---
name: "Library/MG/Mio_Diario"
tags: meta/library
description: "Utility per gestire, navigare e aggregare il Diario personale in SilverBullet."
version: "1.01 Beta"
versionDate: 2026-08-26
pageDecoration.prefix: "📔 "
share.uri: "github:marco10x15/silverbullet-libraries/Mio_Diario.md"
---

# 📔 Mio Diario

**Mio Diario** è la raccolta di utility per gestire, navigare e aggregare il Mio Diario con SilverBullet.

**Versione:** 1.01 Beta — 26.08.2026

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
* `description` — descrizione opzionale;
* `Viaggio` — wikilink opzionale a una singola pagina `Viaggi/...`;
* `luoghi` — lista opzionale di wikilink a pagine `luoghi/...`;
* `galleriafoto` — lista opzionale di riferimenti alle gallerie fotografiche;
* `tags` — gestiti direttamente dal rendering del frontmatter di SilverBullet.

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
* **Siamo stati qui** — mostra in un widget separato i Viaggi e le giornate del Diario direttamente associate al luogo corrente o ai suoi discendenti.
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

> **warning** La libreria considera normalizzate le pagine `Diario/...`: `Viaggio` deve essere un wikilink nel frontmatter e `luoghi` una lista di wikilink.

> **warning** Le funzioni `page.title()`, `page.nome()`, `page.prec()` e `page.succ()` devono essere disponibili nello Space.

> **warning** La risoluzione descrittiva di `divisioneIntermedia` utilizza una risorsa Internet esterna. Se il dataset ISO non è raggiungibile, la libreria continua a funzionare utilizzando il codice ISO come fallback.

> **warning** La correttezza delle aggregazioni geografiche dipende dalla coerenza dei path `luoghi/...` e degli attributi `divisioneIntermedia` delle pagine Markdown.

> **warning** Il widget **Siamo stati qui** usa le `relation` indicizzate da SilverBullet con `kind = "luoghi"`. Una visita registrata soltanto sul Comune viene mostrata sul Comune, ma non viene attribuita automaticamente a una sua Località. Una visita registrata sulla Località viene invece inclusa anche nelle pagine dei suoi antenati.

## Implementazione

```space-lua
luoghi = luoghi or {}
widgets = widgets or {}


-- ============================================================
-- FUNZIONI GENERALI
-- ============================================================

-- Verifica che un valore restituito dall'indice sia una
-- stringa non vuota. Evita che SQL NULL venga interpretato
-- come testo valido nel rendering.
function luoghi.hasString(value)
  return type(value) == "string" and value ~= ""
end

-- Verifica che un valore sia una lista Lua non vuota.
function luoghi.hasList(value)
  return type(value) == "table" and #value > 0
end

-- Estrae l'ultimo segmento di un path SilverBullet.
function luoghi.basename(pageName)
  return string.match(pageName, "([^/]+)$") or pageName
end

-- Restituisce il path della pagina madre.
function luoghi.parentName(pageName)
  return string.match(pageName, "^(.*)/[^/]+$")
end

-- Determina l'etichetta visualizzata di una pagina.
-- Priorità: displayName -> title -> aliases[1] -> fallback.
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

-- Costruisce un wikilink verso una pagina luogo reale.
function luoghi.pageLink(p)
  return string.format(
    "[[%s|%s]]",
    p.name,
    luoghi.label(p, luoghi.basename(p.name))
  )
end

-- Recupera dall'indice una singola pagina luogo e soltanto
-- gli attributi utilizzati dalla libreria.
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

-- Restituisce esclusivamente i figli diretti della pagina.
-- Usa index.subPages() per limitare la query al ramo richiesto.
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

-- Estrae il target da un wikilink SilverBullet.
function luoghi.wikilinkTarget(link)
  if not luoghi.hasString(link) then
    return nil
  end
  return string.match(link, "^%[%[([^]|]+)")
end


-- ============================================================
-- RADICE LUOGHI
-- ============================================================

-- Costruisce la parte dinamica della pagina reale luoghi.md.
-- Ogni figlio diretto rappresenta uno Stato e viene raggruppato
-- per continente. Gli Stati senza continente valido finiscono
-- nella sezione "Da verificare".
function luoghi.renderRoot()
  local children = luoghi.children("luoghi")
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
  for _, continente in ipairs(ordineContinenti) do
    gruppi[continente] = {}
  end

  local nonClassificati = {}

  for _, p in ipairs(children) do
    local item = {
      label = luoghi.label(p, luoghi.basename(p.name)),
      link = luoghi.pageLink(p)
    }

    if luoghi.hasString(p.continente)
      and gruppi[p.continente]
    then
      table.insert(gruppi[p.continente], item)
    else
      table.insert(nonClassificati, item)
    end
  end

  local function ordina(lista)
    table.sort(lista, function(a, b)
      return a.label < b.label
    end)
  end

  local text = "## Paesi visitati\n"

  for _, continente in ipairs(ordineContinenti) do
    local stati = gruppi[continente]
    if #stati > 0 then
      ordina(stati)
      local links = {}
      for _, stato in ipairs(stati) do
        table.insert(links, stato.link)
      end

      text = text
        .. "\n### " .. continente
        .. "\n\n"
        .. table.concat(links, ", ")
        .. "\n"
    end
  end

  if #nonClassificati > 0 then
    ordina(nonClassificati)
    local links = {}
    for _, stato in ipairs(nonClassificati) do
      table.insert(links, stato.link)
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

-- Scarica il dataset ISO 3166-2 e costruisce un indice locale
-- in memoria basato sul codice della suddivisione.
function luoghi.isoIndex()
  if luoghi._isoAttempted then
    return luoghi._isoByCode
  end

  luoghi._isoAttempted = true

  local url = config.get(
    "luoghi.iso3166_2.url",
    "https://salsa.debian.org/iso-codes-team/iso-codes/-/raw/main/data/iso_3166-2.json"
  )

  local response = net.proxyFetch(url, {
    headers = { Accept = "application/json" }
  })

  if not response or not response.ok then
    return nil
  end

  local body = response.body
  if type(body) == "string" then
    body = js.tolua(js.window.JSON.parse(body))
  end

  if type(body) ~= "table" or not body["3166-2"] then
    return nil
  end

  local byCode = {}
  for _, item in ipairs(body["3166-2"]) do
    if item.code then
      byCode[item.code] = item
    end
  end

  luoghi._isoByCode = byCode
  return byCode
end

-- Restituisce il record ISO relativo a un codice divisione.
function luoghi.isoDivisione(code)
  local indexIso = luoghi.isoIndex()
  if not indexIso then
    return nil
  end
  return indexIso[code]
end

-- Costruisce il nome visualizzato di una divisione intermedia.
function luoghi.divisioneLabel(code)
  local divisione = luoghi.isoDivisione(code)
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

  local prefisso = prefissi[divisione.type]
  if prefisso then
    return prefisso .. divisione.name
  end

  return divisione.name or code
end

-- Costruisce il wikilink verso una Virtual Page geo:divisione:*.
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

-- Costruisce una Virtual Page di divisione intermedia.
-- Le pagine Markdown con divisioneIntermedia == code sono la
-- fonte primaria; ISO viene usato soltanto per la descrizione.
function luoghi.renderDivisione(code)
  local divisione = luoghi.isoDivisione(code)
  local label = luoghi.divisioneLabel(code)

  local pages = query[[
    from p = index.pages()
    where string.startsWith(p.name, "luoghi/")
      and p.divisioneIntermedia == code
    order by p.name
    select {
      name = p.name,
      title = p.title,
      displayName = p.displayName,
      aliases = p.aliases
    }
  ]]

  local text = "# " .. label .. "\n\n`" .. code .. "`"

  if divisione and luoghi.hasString(divisione.type) then
    text = text .. " — " .. divisione.type
  end

  local parentSeen = {}
  local parentLinks = {}

  for _, p in ipairs(pages) do
    local parentName = luoghi.parentName(p.name)
    if parentName and not parentSeen[parentName] then
      local parent = luoghi.getPage(parentName)

      if parent then
        table.insert(parentLinks, luoghi.pageLink(parent))
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

      parentSeen[parentName] = true
    end
  end

  if #parentLinks > 0 then
    text = text
      .. "\n\nAppartiene a "
      .. table.concat(parentLinks, ", ")
      .. "."
  end

  if pages and #pages > 0 then
    local links = {}
    for _, p in ipairs(pages) do
      table.insert(links, luoghi.pageLink(p))
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

-- Registra le Virtual Pages delle divisioni intermedie.
virtualPage.define {
  pattern = "geo:divisione:(.+)",
  run = function(code)
    return luoghi.renderDivisione(code)
  end
}


-- ============================================================
-- PRESENTAZIONE E NAVIGAZIONE PAGINE LUOGHI
-- ============================================================

-- Produce la descrizione sintetica della natura del luogo.
-- Le entità amministrative usano tipoAmministrativo; le località
-- fisiche usano tags. Wikipedia viene aggiunta se disponibile.
function luoghi.renderTipo(p)
  local elementi = {}

  if luoghi.hasString(p.tipoAmministrativo) then
    table.insert(elementi, "`" .. p.tipoAmministrativo .. "`")
  elseif luoghi.hasList(p.tags) then
    for _, tag in ipairs(p.tags) do
      if luoghi.hasString(tag) then
        table.insert(elementi, "#" .. tag)
      end
    end
  end

  if luoghi.hasString(p.wikipedia) then
    table.insert(elementi, "[wikipedia](" .. p.wikipedia .. ")")
  end

  return table.concat(elementi, " ")
end

-- Renderizza una pagina Stato senza interrogare il Diario.
function luoghi.renderStato(p)
  local text = luoghi.renderTipo(p)

  if luoghi.hasString(p.continente) then
    text = text
      .. "\n\nSi trova in **"
      .. p.continente
      .. "**."
  end

  local children = luoghi.children(p.name)
  if children and #children > 0 then
    local links = {}
    for _, child in ipairs(children) do
      table.insert(links, luoghi.pageLink(child))
    end

    text = text
      .. "\n\nIn cui ci sono "
      .. #children
      .. " suddivisioni: "
      .. table.concat(links, ", ")
  end

  return text
end

-- Renderizza una suddivisione primaria.
-- I figli con divisioneIntermedia vengono raggruppati in
-- Virtual Pages; gli altri restano pagine luogo dirette.
function luoghi.renderSuddivisione(p)
  local text = luoghi.renderTipo(p)
  local children = luoghi.children(p.name)

  local divisioni = {}
  local divisioniSeen = {}
  local altriLuoghi = {}

  for _, child in ipairs(children) do
    if luoghi.hasString(child.divisioneIntermedia) then
      local code = child.divisioneIntermedia
      if not divisioniSeen[code] then
        table.insert(divisioni, code)
        divisioniSeen[code] = true
      end
    else
      table.insert(altriLuoghi, child)
    end
  end

  table.sort(divisioni)

  if #divisioni > 0 then
    local links = {}
    for _, code in ipairs(divisioni) do
      table.insert(links, luoghi.divisioneLink(code))
    end

    text = text
      .. "\n\nIn cui sono rappresentate "
      .. #divisioni
      .. " divisioni intermedie: "
      .. table.concat(links, ", ")
  end

  if #altriLuoghi > 0 then
    local links = {}
    for _, child in ipairs(altriLuoghi) do
      table.insert(links, luoghi.pageLink(child))
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

    text = text .. table.concat(links, ", ")
  end

  return text
end

-- Renderizza Comune/Città, Località e livelli inferiori.
-- Produce soltanto la parte geografica; Viaggi e giornate sono
-- demandati al widget separato "Siamo stati qui".
function luoghi.renderLuogo(p)
  local text = luoghi.renderTipo(p)

  local parentName = luoghi.parentName(p.name)
  local parent = nil
  if parentName then
    parent = luoghi.getPage(parentName)
  end

  local frase = luoghi.pageLink(p)

  if parent then
    frase = frase
      .. " che si trova in "
      .. luoghi.pageLink(parent)
  end

  if luoghi.hasString(p.divisioneIntermedia) then
    frase = frase
      .. ", appartiene a "
      .. luoghi.divisioneLink(p.divisioneIntermedia)
  end

  local children = luoghi.children(p.name)
  if children and #children > 0 then
    local links = {}
    for _, child in ipairs(children) do
      table.insert(links, luoghi.pageLink(child))
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

-- Dispatcher della sola navigazione geografica.
-- Non interroga il Diario.
function widgets.linkedInfoLuoghi(pageName)
  pageName = pageName or editor.getCurrentPage()

  if pageName == "luoghi" then
    return luoghi.renderRoot()
  end

  if not string.startsWith(pageName, "luoghi/") then
    return ""
  end

  local p = luoghi.getPage(pageName)
  if not p then
    return ""
  end

  local parts = string.split(pageName, "/")

  if #parts == 2 then
    return luoghi.renderStato(p)
  end

  if #parts == 3 then
    return luoghi.renderSuddivisione(p)
  end

  return luoghi.renderLuogo(p)
end


-- ============================================================
-- SIAMO STATI QUI
-- ============================================================

-- Individua le pagine Diario collegate al luogo corrente usando
-- direttamente le relation indicizzate con kind "luoghi".
-- Include il luogo esatto e i suoi discendenti.
function luoghi.diarioPerLuogo(pageName)
  local relazioni = query[[
    from r = index.relations("luoghi")
    where string.startsWith(r.page, "Diario/")
      and (
        r.to == pageName
        or string.startsWith(
          r.to,
          pageName .. "/"
        )
      )
    select {
      page = r.page
    }
  ]]

  if not relazioni or #relazioni == 0 then
    return {}
  end

  local pageNames = {}
  local seenPages = {}

  for _, relazione in ipairs(relazioni) do
    if luoghi.hasString(relazione.page)
      and not seenPages[relazione.page]
    then
      table.insert(pageNames, relazione.page)
      seenPages[relazione.page] = true
    end
  end

  return query[[
    from p = index.pages()
    where p.date
      and table.includes(pageNames, p.name)
    order by p.date desc, p.name
    select {
      page = p.name,
      date = p.date,
      Viaggio = p.Viaggio
    }
    limit 500
  ]]
end

-- Costruisce un wikilink normalizzato verso una pagina Viaggio.
function luoghi.viaggioLink(viaggio)
  local target = luoghi.wikilinkTarget(viaggio)
  if not target then
    return nil
  end

  return string.format(
    "[[%s|%s]]",
    target,
    luoghi.basename(target)
  )
end

-- Costruisce esclusivamente il contenuto "Siamo stati qui".
-- Le giornate sono ordinate dalla più recente alla più vecchia;
-- i Viaggi vengono deduplicati.
function luoghi.renderSiamoStatiQui(pageName)
  local diarioInfo = luoghi.diarioPerLuogo(pageName)

  if not diarioInfo or #diarioInfo == 0 then
    return ""
  end

  local viaggi = {}
  local giorni = {}
  local seenViaggi = {}

  for _, info in ipairs(diarioInfo) do
    local viaggioTarget = luoghi.wikilinkTarget(info.Viaggio)

    if viaggioTarget and not seenViaggi[viaggioTarget] then
      local link = luoghi.viaggioLink(info.Viaggio)
      if link then
        table.insert(viaggi, link)
      end
      seenViaggi[viaggioTarget] = true
    end

    local titolo = page.title(info.page)
    if not luoghi.hasString(titolo) then
      titolo = page.nome(info.page)
    end

    local giorno = date.format(info.date)

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

  local righe = { "## Siamo stati qui:" }

  if #viaggi > 0 then
    table.insert(righe, "\n🧭 **In questi viaggi:**")
    table.insert(righe, table.concat(viaggi, "\n"))
  end

  if #giorni > 0 then
    table.insert(righe, "\n**In questi giorni:**")
    table.insert(righe, table.concat(giorni, "\n"))
  end

  return table.concat(righe, "\n")
end

-- Espone "Siamo stati qui" come widget autonomo.
-- Viene usato soltanto da Comune/Città in giù.
function widgets.siamoStatiQui(pageName)
  pageName = pageName or editor.getCurrentPage()

  if not string.startsWith(pageName, "luoghi/") then
    return ""
  end

  local parts = string.split(pageName, "/")
  if #parts < 4 then
    return ""
  end

  return luoghi.renderSiamoStatiQui(pageName)
end


-- ============================================================
-- INFO VIAGGIO
-- ============================================================


-- Verifica che il campo Viaggio della pagina Diario punti
-- esattamente alla pagina Viaggi richiesta.
--
-- Il confronto utilizza il target del wikilink e ignora
-- l'eventuale alias.
function contieneViaggio(viaggio, pageName)
  return luoghi.wikilinkTarget(viaggio)
    == pageName
end


-- Recupera le pagine Diario appartenenti a un Viaggio.
--
-- La funzione centralizza la query comune ai due widget:
-- "Luoghi visitati" e "Pagine del diario".
--
-- I risultati sono ordinati cronologicamente.
function viaggiDiarioInfo(pageName)
  if not string.startsWith(
    pageName,
    "Viaggi/"
  ) then
    return {}
  end

  return query[[
    from p = index.pages()
    where string.startsWith(p.name, "Diario/")
      and p.date
      and p.Viaggio
      and contieneViaggio(p.Viaggio, pageName)
    order by p.date, p.name
    select {
      page = p.name,
      date = p.date,
      title = page.title(p.name),
      luoghi = p.luoghi
    }
    limit 500
  ]]
end


-- ============================================================
-- LUOGHI VISITATI NEL VIAGGIO
-- ============================================================


-- Costruisce esclusivamente il widget "Luoghi visitati".
--
-- I luoghi vengono raccolti dalle pagine Diario appartenenti
-- al Viaggio e deduplicati usando il target del wikilink.
--
-- L'ordine corrisponde alla prima comparsa del luogo nel
-- viaggio, quindi segue naturalmente la cronologia del Diario.
function widgets.infoViaggioLuoghi(pageName)
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

  local luoghiVisitati = {}
  local seenLuoghi = {}

  for _, info in ipairs(diarioInfo) do
    if luoghi.hasList(info.luoghi) then
      for _, luogo in ipairs(info.luoghi) do
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
      .. "\n"
      .. table.concat(
        luoghiVisitati,
        ", "
      )
      .. "\n"
  else
    text = text
      .. "\nNessun luogo registrato.\n"
  end

  return text
end


-- ============================================================
-- PAGINE DIARIO DEL VIAGGIO
-- ============================================================


-- Costruisce esclusivamente il widget con le pagine Diario
-- appartenenti al Viaggio.
--
-- Per ogni giornata visualizza:
-- - data come link alla pagina Diario;
-- - titolo della giornata;
-- - luoghi associati alla singola pagina.
--
-- date.format() viene usato senza imporre un pattern, lasciando
-- a SilverBullet la rappresentazione Human Readable della data.
function widgets.infoViaggioDiario(pageName)
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

  local giorni = {}

  for _, info in ipairs(diarioInfo) do
    local giorno =
      date.format(info.date)

    local titolo =
      luoghi.hasString(info.title)
      and info.title
      or page.nome(info.page)

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

-- ============================================================
-- ATTIVAZIONE WIDGET
-- ============================================================

-- Widget 1: struttura e navigazione geografica.
if config.get("std.widgets.linkedInfoLuoghi.enabled") then
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
        return widget.markdown(text)
      end
    end
  }
end

-- Widget 2: Viaggi e giornate associate al luogo.
-- È separato dal widget geografico per evitare problemi di
-- rendering e per mantenere distinte le due responsabilità.
if config.get("std.widgets.siamoStatiQui.enabled") then
  event.listen {
    name = "hooks:renderBottomWidgets",
    run = function(e)
      local pageName = editor.getCurrentPage()

      if not string.startsWith(pageName, "luoghi/") then
        return
      end

      local text = widgets.siamoStatiQui(pageName)
      if text ~= "" then
        return widget.markdown(text)
      end
    end
  }
end

-- ============================================================
-- ATTIVAZIONE INFO VIAGGIO
-- ============================================================

-- I due listener utilizzano la stessa opzione di configurazione
-- ma producono due bottom widget distinti.
--
-- Questo permette a SilverBullet di renderizzare separatamente:
-- 1. Luoghi visitati
-- 2. Pagine Diario del Viaggio
if config.get(
  "std.widgets.infoViaggio.enabled"
) then

  -- Luoghi visitati
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
        widgets.infoViaggioLuoghi(
          pageName
        )

      if text ~= "" then
        return widget.markdown(text)
      end
    end
  }

  -- Pagine Diario
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
        widgets.infoViaggioDiario(
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

-- Widget superiore delle pagine Diario.
if config.get("std.widgets.wTopDiario.enabled", true) then
  event.listen {
    name = "hooks:renderTopWidgets",
    run = function(e)
      return wTopDiario()
    end
  }
end

-- Costruisce il widget superiore delle pagine Diario.
--
-- Visualizza:
-- - navigazione alla pagina precedente e successiva;
-- - titolo della giornata;
-- - description;
-- - Viaggio;
-- - luoghi;
-- - galleria fotografica.
--
-- I dati strutturati vengono letti direttamente dall'indice.
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
      description = p.description,
      Viaggio = p.Viaggio,
      luoghi = p.luoghi,
      galleriafoto = p.galleriafoto
    }
    limit 1
  ]]

  local p = pages[1] or {}

  local titolo =
    page.title(pageName)

  if not luoghi.hasString(titolo) then
    titolo =
      page.nome(pageName)
  end

  local righe = {
    string.format(
      "# [[%s|⬅️]]%s[[%s|➡️]]",
      page.prec(),
      titolo,
      page.succ()
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


## Navigazione con tastiera

### Compatibilità browser

In Microsoft Edge per Windows:

*   `Alt + ←` = **browser indietro**
*   `Alt + →` = **browser avanti**.

E la documentazione SilverBullet avverte esplicitamente che gli shortcut già riservati dal browser **non sono sempre sovrascrivibili in modo affidabile**.

```space-lua
-- Naviga alla pagina Diario precedente.
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

    -- Mantiene il normale comportamento di navigazione
    -- nelle pagine che non appartengono al Diario.
    editor.goHistory(-1)
  end
}


-- Naviga alla pagina Diario successiva.
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

## Verifica diagnostica delle relazioni Luoghi

Se una pagina Luogo non visualizza **Siamo stati qui**, verificare prima che SilverBullet abbia indicizzato la relazione `luoghi` attesa.

Se la query non restituisce risultati, il widget non attribuisce alla Località una visita registrata soltanto sulla pagina madre.

## Space Style (Optional Theming Overrides)

Non sono previste personalizzazioni CSS nella **Versione 1 alpha**.

Questa sezione è riservata a eventuali miglioramenti successivi dell'interfaccia, in particolare per:

* navigazione ottimizzata su smartphone;
* spaziatura e disposizione dei link precedente/successivo;
* visualizzazione compatta di luoghi e Viaggi;
* gallerie fotografiche;
* eventuali widget cartografici.
