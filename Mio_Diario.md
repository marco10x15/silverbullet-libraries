---
name: "Library/MG/Mio_Diario"
tags: meta/library
pageDecoration.prefix: "📃 "
share.uri: "github:marco10x15/silverbullet-libraries/Mio_Diario.md"
---
# Struttura e query utilizzate per il mio diario

Documentazione, funzioni di navigazione pagine, gestione dei metadati.

# Virtual Page

## Top Widget - wTopDiario()
### Descrizione
Inserisce per ogni pagina Diario una intestazione con:
- navigazione pagina precedente e successiva
- link a al Viaggio, se la pagina apaprtiene a un Viaggio
- elenco ei Tag della pagina del Diario


### Esempio
`${wTopDiario("Diario/2026-05-08")}`

### Configurazione
Per attivare il Widget occorre:
```lua
config.set("std.widgets.wTopDiario.enabled", true)
```

```space-lua
-- priority: -1
if config.get("std.widgets.wTopDiario.enabled", true) then
  event.listen {
    name = "hooks:renderTopWidgets",
    run = function(e)
        return wTopDiario()
    end
  }
end
```

### Funzioni utilizzate

* space.readPage(pageName)
* index.extractFrontmatter(content)

## Info Luoghi
Questo widget presenta l'elenco dei Viaggi e dei Giorni in cui siamo stati nel luogo di cui la pagina "luoghi" è inserita.

### Esempio di Render
Per il luogo **[[luoghi/ITA/34/Venezia|Venezia]]**

***
<!--#lua widgets.linkedInfoLuoghi("luoghi/ITA/34/Venezia") -->
`comune` [wikipedia](https://it.wikipedia.org/wiki/Venezia)

[[luoghi/ITA/34/Venezia|Venezia]] che si trova in [[luoghi/ITA/34|Veneto]], appartiene a [[geo:divisione:IT-VE|Città metropolitana di Venezia]] e include: [[luoghi/ITA/34/Venezia/Burano|Burano]], [[luoghi/ITA/34/Venezia/Murano|Murano]]

<br>

## Siamo stati qui:

🧭 **In questi viaggi:**
[[Viaggi/Tre giorni a Venezia|Tre giorni a Venezia]]

**In questi giorni:**
**[[Diario/2019-04-25|25.04.2019]]** Venezia, Murano e Burano.
**[[Diario/2019-04-24|24.04.2019]]** Saliamo sul campanile.
**[[Diario/2019-04-23|23.04.2019]]** Primo giorno del viaggio a Venezia.
<!--/lua-->
***

### Attivazione
```space-lua
config.set("std.widgets.linkedInfoLuoghi.enabled", true)
```

### Implementazione 
```space-lua
luoghi = luoghi or {}
widgets = widgets or {}


-- ============================================================
-- VALORI E FUNZIONI GENERALI
-- ============================================================


-- Verifica che un valore restituito dall'indice sia
-- effettivamente una stringa utilizzabile.
--
-- Evita che SQL NULL o altri valori non stringa vengano
-- interpretati come testo valido nel rendering.
function luoghi.hasString(value)
  return type(value) == "string" and value ~= ""
end


-- Verifica che un valore sia una lista Lua non vuota.
--
-- È utilizzata soprattutto per aliases, tags e luoghi.
function luoghi.hasList(value)
  return type(value) == "table" and #value > 0
end


-- Estrae l'ultimo segmento del path.
--
-- Esempio:
-- luoghi/FRA/BFC/Besançon -> Besançon
--
-- È anche il fallback finale quando non esistono metadati
-- utilizzabili per l'etichetta.
function luoghi.basename(pageName)
  return string.match(pageName, "([^/]+)$") or pageName
end


-- Restituisce il path della pagina madre.
--
-- La gerarchia geografica continua quindi a essere derivata
-- dal Markdown e non richiede attributi parent ridondanti.
function luoghi.parentName(pageName)
  return string.match(pageName, "^(.*)/[^/]+$")
end


-- Determina l'etichetta visualizzata di una pagina.
--
-- displayName è prioritario perché è il campo esplicitamente
-- previsto dal modello dati per la presentazione e nei test
-- correnti risulta affidabile.
--
-- Priorità:
-- displayName -> title -> aliases[1] -> fallback.
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


-- Costruisce il wikilink verso una pagina luogo reale.
--
-- Il target rimane sempre p.name; l'etichetta è invece
-- ricavata dai metadati tramite luoghi.label().
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


-- Recupera una singola pagina dall'indice con i soli attributi
-- necessari alla libreria.
--
-- Ogni chiamata produce una query; va quindi mantenuta fuori
-- dai loop più ampi quando possibile.
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


-- Restituisce esclusivamente i figli diretti.
--
-- index.subPages() restringe la ricerca al ramo richiesto;
-- il filtro successivo elimina nipoti e livelli inferiori.
--
-- continente è incluso per permettere alla radice luoghi.md
-- di raggruppare gli Stati.
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


-- ============================================================
-- RADICE LUOGHI
-- ============================================================


-- Costruisce la parte dinamica della pagina reale luoghi.md.
--
-- Ogni figlio diretto di luoghi/ rappresenta uno Stato.
-- Gli Stati vengono raggruppati per continente senza interrogare
-- Diario o Viaggi, quindi questa vista rimane economica.
--
-- Stati con continente assente o non riconosciuto vengono
-- riportati in "Da verificare" invece di essere nascosti.
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

  for _, continente in ipairs(ordineContinenti) do
    local stati = gruppi[continente]

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

    for _, stato in ipairs(nonClassificati) do
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


-- Scarica il dataset ISO 3166-2 e lo indicizza per codice.
--
-- Il risultato viene mantenuto in memoria durante la sessione
-- per evitare fetch ripetuti.
--
-- Se la fonte non è disponibile, le funzioni successive
-- degradano mostrando semplicemente il codice ISO.
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
    headers = {
      Accept = "application/json"
    }
  })

  if not response or not response.ok then
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

  for _, item in ipairs(body["3166-2"]) do
    if item.code then
      byCode[item.code] = item
    end
  end

  luoghi._isoByCode = byCode

  return byCode
end


-- Restituisce il record ISO relativo al codice richiesto.
--
-- Isola il resto della libreria dalla struttura interna
-- dell'indice ISO.
function luoghi.isoDivisione(code)
  local indexIso = luoghi.isoIndex()

  if not indexIso then
    return nil
  end

  return indexIso[code]
end


-- Costruisce il nome visualizzato di una divisione intermedia.
--
-- Alcuni tipi ISO vengono tradotti in un prefisso italiano.
-- Se il tipo non è conosciuto viene utilizzato il nome ISO;
-- se il fetch fallisce viene mostrato il codice.
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


-- Costruisce il wikilink verso la Virtual Page associata
-- a una divisione intermedia.
function luoghi.divisioneLink(code)
  return string.format(
    "[[geo:divisione:%s|%s]]",
    code,
    luoghi.divisioneLabel(code)
  )
end


-- ============================================================
-- VIRTUAL PAGE geo:divisione:*
-- ============================================================


-- Costruisce una Virtual Page di divisione intermedia.
--
-- Le pagine Markdown che dichiarano divisioneIntermedia == code
-- sono la fonte primaria dell'appartenenza geografica.
-- ISO viene usato solo per arricchire il nome visualizzato.
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

  local text = "# " .. label .. "\n\n"
    .. "`" .. code .. "`"

  if divisione
    and luoghi.hasString(divisione.type)
  then
    text = text
      .. " — "
      .. divisione.type
  end

  local parentSeen = {}
  local parentLinks = {}

  for _, p in ipairs(pages) do
    local parentName =
      luoghi.parentName(p.name)

    if parentName
      and not parentSeen[parentName]
    then
      local parent =
        luoghi.getPage(parentName)

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


-- Registra la Virtual Page parametrica.
--
-- Tutta la logica rimane in renderDivisione(); la definizione
-- della Virtual Page resta quindi minimale.
virtualPage.define {
  pattern = "geo:divisione:(.+)",

  run = function(code)
    return luoghi.renderDivisione(code)
  end
}


-- ============================================================
-- WIKILINK E RELAZIONI DIARIO
-- ============================================================


-- Estrae il target da un wikilink SilverBullet.
--
-- [[luoghi/FRA/BFC/Besançon|Besançon]]
-- diventa:
-- luoghi/FRA/BFC/Besançon
--
-- Se riceve una normale stringa non wikilink restituisce nil.
function luoghi.wikilinkTarget(link)
  if not luoghi.hasString(link) then
    return nil
  end

  return string.match(
    link,
    "^%[%[([^]|]+)"
  )
end


-- Verifica se una pagina Diario è associata al luogo richiesto.
--
-- Il confronto viene effettuato sul TARGET del wikilink,
-- non sulla sua rappresentazione completa.
--
-- In questo modo sono equivalenti:
-- [[luoghi/X]]
-- [[luoghi/X|alias]]
--
-- e un sottoluogo viene automaticamente attribuito anche
-- a tutti i suoi antenati geografici.
function contieneLuogo(luoghiPagina, pageName)
  if type(luoghiPagina) ~= "table" then
    return false
  end

  for _, luogo in ipairs(luoghiPagina) do
    if type(luogo) == "string" then
      local target =
        luoghi.wikilinkTarget(luogo)

      -- Compatibilità anche con eventuali target raw.
      if not target then
        target = luogo
      end

      if target == pageName
        or string.startsWith(
          target,
          pageName .. "/"
        )
      then
        return true
      end
    end
  end

  return false
end


-- Costruisce un wikilink normalizzato verso un Viaggio.
--
-- L'alias eventualmente presente nel dato viene scartato e
-- viene visualizzato il nome finale della pagina.
--
-- Se il campo Viaggio viene oscurato da sintassi legacy
-- indicizzata da SilverBullet, questa funzione non tenta
-- deliberatamente di reinterpretarlo.
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


-- ============================================================
-- DIARIO E VIAGGI
-- ============================================================


-- Cerca le pagine Diario associate al luogo corrente.
--
-- La query non usa più GROUP BY: ogni pagina viene restituita
-- direttamente e l'eventuale deduplicazione viene fatta in Lua.
-- Questo rende più semplice il comportamento e preserva meglio
-- gli attributi delle singole pagine.
--
-- I risultati sono ordinati per data decrescente.
function luoghi.renderDiario(pageName)
  local diarioInfo = query[[
    from p = index.pages()
    where string.startsWith(p.name, "Diario/")
      and p.date
      and p.luoghi
      and contieneLuogo(p.luoghi, pageName)
    order by p.date desc, p.name
    select {
      page = p.name,
      date = p.date,
      Viaggio = p.Viaggio
    }
    limit 500
  ]]

  if not diarioInfo
    or #diarioInfo == 0
  then
    return ""
  end

  local viaggi = {}
  local giorni = {}

  local seenViaggi = {}
  local seenPagine = {}

  for _, info in ipairs(diarioInfo) do

    -- Evita eventuali duplicazioni dello stesso oggetto pagina.
    if not seenPagine[info.page] then
      seenPagine[info.page] = true

      local viaggioTarget =
        luoghi.wikilinkTarget(info.Viaggio)

      if viaggioTarget
        and not seenViaggi[viaggioTarget]
      then
        local link =
          luoghi.viaggioLink(info.Viaggio)

        if link then
          table.insert(
            viaggi,
            link
          )
        end

        seenViaggi[viaggioTarget] = true
      end

      local titolo =
        page.title(info.page)

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
  end

  local text =
    "\n\n<br>\n\n## Siamo stati qui:\n"

  if #viaggi > 0 then
    text = text
      .. "\n🧭 **In questi viaggi:**\n"
      .. table.concat(viaggi, "\n")
      .. "\n"
  end

  if #giorni > 0 then
    text = text
      .. "\n**In questi giorni:**\n"
      .. table.concat(giorni, "\n")
      .. "\n"
  end

  return text
end


-- ============================================================
-- TIPO / TAGS / WIKIPEDIA
-- ============================================================


-- Produce la descrizione sintetica della pagina.
--
-- Le entità amministrative usano tipoAmministrativo;
-- le località non amministrative usano invece tags.
-- Wikipedia viene aggiunta quando disponibile.
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


-- ============================================================
-- STATO
-- ============================================================


-- Renderizza uno Stato.
--
-- Mostra tipo amministrativo, Wikipedia, continente e
-- suddivisioni primarie presenti nello Space.
--
-- Non interroga il Diario.
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

  if children
    and #children > 0
  then
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


-- ============================================================
-- SUDDIVISIONE PRIMARIA
-- ============================================================


-- Renderizza una suddivisione primaria.
--
-- I figli dotati di divisioneIntermedia vengono aggregati in
-- Virtual Pages; quelli privi dell'attributo vengono mostrati
-- direttamente come pagine luogo.
--
-- La qualità di questa distinzione dipende dalla correttezza
-- del frontmatter Markdown.
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

    for _, child in ipairs(altriLuoghi) do
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


-- ============================================================
-- COMUNE / CITTÀ / LOCALITÀ
-- ============================================================


-- Renderizza Comune/Città, Località e livelli inferiori.
--
-- Mostra la pagina corrente, il padre immediato, l'eventuale
-- divisione intermedia e i soli figli diretti.
--
-- La stessa logica può proseguire a profondità arbitraria
-- senza introdurre nuovi livelli amministrativi nel codice.
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

  if children
    and #children > 0
  then
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


-- ============================================================
-- WIDGET PRINCIPALE
-- ============================================================


-- Dispatcher principale.
--
-- La profondità determina il comportamento:
--
-- luoghi
--   -> elenco Stati per continente
--
-- luoghi/FRA
--   -> Stato
--
-- luoghi/FRA/BFC
--   -> suddivisione primaria
--
-- luoghi/FRA/BFC/Besançon e livelli inferiori
--   -> luogo + Diario/Viaggi
--
-- Le query più costose sul Diario vengono quindi eseguite
-- soltanto dal livello Comune/Città in giù.
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

  if #parts == 2 then
    return luoghi.renderStato(p)
  end

  if #parts == 3 then
    return luoghi.renderSuddivisione(p)
  end

  return luoghi.renderLuogo(p)
    .. luoghi.renderDiario(pageName)
end


-- ============================================================
-- ATTIVAZIONE
-- ============================================================


-- Attiva il bottom widget sia sulla pagina radice luoghi.md
-- sia su tutte le pagine appartenenti alla gerarchia luoghi/.
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
```

***
***

## Info Viaggio 
Restituisce le informazioni del viaggio prendendole dalle pagine Diario.

### Esempio di Render


### Implementazione

```space-lua
-- Verifica se l'attributo Viaggio punta alla pagina corrente.
function contieneViaggio(viaggio, pageName)
  if not viaggio then
    return false
  end

  return viaggio == "[[" .. pageName .. "]]"
    or string.startsWith(viaggio, "[[" .. pageName .. "|")
end


-- Estrae il target di un wikilink per deduplicare anche link con alias.
function wikilinkTarget(link)
  if not link then
    return nil
  end

  return string.match(link, "^%[%[([^]|]+)")
end


function widgets.infoViaggio(pageName)
  pageName = pageName or editor.getCurrentPage()

  if not string.startsWith(pageName, "Viaggi/") then
    return ""
  end

  local diarioInfo = query[[
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

  if not diarioInfo or #diarioInfo == 0 then
    return ""
  end

  local luoghi = {}
  local seenLuoghi = {}
  local giorni = {}

  for _, info in ipairs(diarioInfo) do
    local luoghiGiorno = {}

    if info.luoghi then
      for _, luogo in ipairs(info.luoghi) do
        local target = wikilinkTarget(luogo) or luogo

        -- Elenco complessivo: una sola volta,
        -- nell'ordine della prima visita.
        if not seenLuoghi[target] then
          table.insert(luoghi, luogo)
          seenLuoghi[target] = true
        end

        -- Elenco relativo alla singola giornata.
        table.insert(luoghiGiorno, luogo)
      end
    end

    local giorno = date.format(info.date, "DD.MM.YYYY")
    local title = info.title or ""
    local riga = string.format(
      "[[%s|%s]] %s",
      info.page,
      giorno,
      title
    )

    if #luoghiGiorno > 0 then
      riga = riga
        .. " ("
        .. table.concat(luoghiGiorno, ", ")
        .. ")"
    end

    table.insert(giorni, riga)
  end

  local text = "\n## Luoghi visitati\n"

  if #luoghi > 0 then
    text = text
      .. table.concat(luoghi, ", ")
      .. "\n"
  else
    text = text .. "Nessun luogo registrato.\n"
  end

  text = text
    .. "\n## "
    .. #diarioInfo
    .. " pagine del diario per questo viaggio\n"
    .. table.concat(giorni, "\n")
    .. "\n"

  return text
end
```

### Attivazione 

```space-lua
config.set("std.widgets.infoViaggio.enabled", true)
```

### Render
```space-lua
-- priority: -1
if config.get("std.widgets.infoViaggio.enabled") then
  event.listen {
    name = "hooks:renderBottomWidgets",
    run = function(e)
      if string.startsWith(editor.getCurrentPage(), "Viaggi/") then
        return widgets.infoViaggio()
      end
    end
  }
end
```
