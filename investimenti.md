---
name: "Library/MG/investimenti"
tags: meta/library
version: "0.23"
versionDate: 2026-10-20
pageDecoration.prefix: "💸 "
share.uri: "github:marco10x15/silverbullet-libraries/investimenti.md"
---

# Investimenti

**Versione 0.23 — Dashboard patrimoniale, snapshot e indicatori essenziali, SilverBullet v2.11+.** Libreria completa, da sostituire integralmente a `Library/MG/investimenti`. Documentazione: pagine `manuale/`.

La libreria legge strumenti, conti, quotazioni e movimenti direttamente dallo Space. Non gestisce liquidità né spese del conto corrente. Gli snapshot sono righe `Snapshot` nella tabella `Movimenti`. La dashboard mostra costo residuo, controvalore e P/L latente; i grafici patrimoniali usano le quantità storiche e i rendiconti disponibili.

**Modifiche 0.23 (schermate nel corpo della pagina):** `scheda` mostra titolo, descrizione, url, orizzonte (`orizzonte.min`/`max` in anni e/o `orizzonte.scade`) e conti, leggendo solo la pagina dello strumento; `dashboard` omette copertura ed elenco quando c'è una sola posizione completa; nuova `serieMensile` (valore di fine mese per strumento, un solo contesto); nuove `graficiConto` (torte valore e rischio KID con «Oggi», barre mensili impilate con 6M/YTD/1A/5A/Tutto) e `graficoPosizione` (linea del valore della posizione); il periodo si sceglie nel browser. Rimossi `seriePatrimonio`, `graficoPatrimonio` e il campionamento a 48 punti.

**Migrazione dalla 0.22:** sostituire `graficoPatrimonio(...)` con `graficiConto(editor.getCurrentPage())` nelle pagine `Conti/` e Index e con `graficoPosizione(editor.getCurrentPage())` nelle pagine `Strumenti/`. `seriePatrimonio(pagina, periodo, tipo)` diventa `serieMensile(pagina, tipo)`, con struttura diversa. Nel frontmatter degli strumenti si possono aggiungere `url` (lista di link Markdown) e `orizzonte`.

## Esempi

```lua
investimenti.dashboard("Index")
investimenti.dashboard("Conti/NEF - Famiglia", "Fondo")
investimenti.scheda("Strumenti/LU0140696401")
investimenti.grafico("Strumenti/LU0140696401")
investimenti.graficiConto("Conti/NEF - Famiglia")
investimenti.graficoPosizione("Strumenti/LU0140696401")
investimenti.rendimentoDati("Strumenti/LU0140696401", "xirr")
investimenti.diagnosticaVista("Conti/NEF - Famiglia")
```

## Codice della libreria

```space-lua
investimenti = investimenti or {}

-- Un attributo assente in una select supera il controllo "if x" pur avendo type "nil":
-- str() lo riporta a un vero nil, così "if x" funziona.
local function str(x) if type(x) == "string" then return x end end

-- Helper condivisi tra i blocchi (ogni blocco ha i propri local): usati solo a chiamata avvenuta.
investimenti.chartjs = "https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.min.js"

--- Testo JavaScript per un numero o una stringa (escape di \\, virgolette e a capo).
function investimenti.js(x)
  if type(x) == "number" then return tostring(x) end
  if type(x) ~= "string" then return '""' end
  local t = string.gsub(x, "\\", "\\\\")
  t = string.gsub(t, '"', '\\"')
  t = string.gsub(t, "\n", " ")
  return '"' .. t .. '"'
end

--- Numero da una cella di tabella o da un attributo: accetta numeri e testo con virgola decimale
--- ("14942,92", "1.234,56"). Restituisce nil se il valore è assente o non è un numero decimale.
function investimenti.numero(x)
  if type(x) == "number" then return x end
  if type(x) ~= "string" then return nil end
  local t = string.gsub(x, "%s", "")
  if string.find(t, ",", 1, true) then
    t = string.gsub(t, "%.", "")
    t = string.gsub(t, ",", ".")
  end
  if not string.match(t, "^%-?%d*%.?%d+$") and not string.match(t, "^%-?%d+%.$") then return nil end
  return tonumber(t)
end

--- Strumenti: pagine sotto Strumenti/, ordinate per id. Include i metadati di valorizzazione
--- (metodoValorizzazione, frequenzaPrezzoGiorni), letti nella stessa scansione.
function investimenti.strumenti()
  local out = {}
  for _, p in ipairs(query[[
    from p = index.subPages("Strumenti")
    select {name = p.name, nome = p.displayName, isin = p.isin, ticker = p.ticker,
      valuta = p.valuta, tipo = p.tipo, rischio = p.KID,
      metodo = p.metodoValorizzazione, frequenza = p.frequenzaPrezzoGiorni}
  ]]) do
    table.insert(out, {id = string.sub(p.name, 11),  -- "Strumenti/" = 10 caratteri
      nome = str(p.nome) or p.name, isin = str(p.isin), ticker = str(p.ticker),
      valuta = str(p.valuta) or "EUR", tipo = str(p.tipo),
      rischio = type(p.rischio) == "number" and p.rischio or nil,  -- p.rischio = attributo KID
      metodoDichiarato = str(p.metodo), frequenza = investimenti.numero(p.frequenza)})
  end
  table.sort(out, function(a, b) return a.id < b.id end)
  return out
end

-- Formattazione SOLO per la presentazione: nessuna modifica ai numeri dei calcoli.
-- I prezzi e le quantità nei dati restano numeri con punto decimale.
function investimenti.formatoNumero(x, decimali)
  if type(x) ~= "number" or x ~= x or x == math.huge or x == -math.huge then return "n.d." end
  local d = decimali or 2
  local t = string.format("%." .. d .. "f", x)
  local segno, intero, frazione = string.match(t, "^([%-]?)(%d+)%.(%d+)$")
  if not intero then return t end
  local gruppi = {}
  while #intero > 3 do
    table.insert(gruppi, 1, string.sub(intero, -3))
    intero = string.sub(intero, 1, -4)
  end
  table.insert(gruppi, 1, intero)
  local z = segno .. table.concat(gruppi, " ")
  if d > 0 then return z .. "," .. frazione end
  return z
end

-- Scheda dello strumento: solo metadati della pagina e relazioni con i conti (nessun contesto).
local function dataIt(d) return string.sub(d, 9, 10) .. "." .. string.sub(d, 6, 7) .. "." .. string.sub(d, 1, 4) end

-- Mesi interi tra due date AAAA-MM-GG (negativo se la seconda precede la prima).
local function mesiTra(da, a)
  local y1, m1, d1 = string.match(da, "^(%d+)%-(%d+)%-(%d+)")
  local y2, m2, d2 = string.match(a, "^(%d+)%-(%d+)%-(%d+)")
  if not (y1 and y2) then return nil end
  local n = (tonumber(y2) - tonumber(y1)) * 12 + (tonumber(m2) - tonumber(m1))
  if tonumber(d2) < tonumber(d1) then n = n - 1 end
  return n
end

local function durata(mesi)
  local a, m = math.floor(mesi / 12), mesi % 12
  local t = {}
  if a > 0 then table.insert(t, a .. "a") end
  if m > 0 then table.insert(t, m .. "m") end
  return #t > 0 and table.concat(t, " ") or "meno di un mese"
end

local function anni(x) return x == math.floor(x) and string.format("%d", x) or tostring(x) end

function investimenti.scheda(pagina)
  pagina = pagina or editor.getCurrentPage()
  if type(pagina) ~= "string" or not string.startsWith(pagina, "Strumenti/") then return nil end
  local id = string.sub(pagina, 11)
  local p = query[[
    from p = index.subPages("Strumenti")
    where p.name == pagina
    select {nome = p.displayName, desc = p.description, tipo = p.tipo, valuta = p.valuta, kid = p.KID,
      url = p.url, orizzonte = p.orizzonte, omin = p["orizzonte.min"], omax = p["orizzonte.max"],
      oscade = p["orizzonte.scade"]}
  ]][1]
  if not p then return widget.markdown("_Strumento non trovato._") end

  -- Emoji Unicode: nessun tag HTML/CSS, per il rendering Markdown di SilverBullet.
  -- Il livello numerico KID resta sempre visibile; 2 e 3 condividono il verde.
  local colori = {"⚪", "🟢", "🟢", "🟠", "🟤", "🔴", "🟣"}
  local kid = type(p.kid) == "number" and p.kid or nil
  local rischio = "Rischio KID n.d."
  if kid and kid == math.floor(kid) and kid >= 1 and kid <= 7 then
    rischio = "Rischio KID " .. tostring(kid) .. "/7 " .. colori[kid]
  end

  local conti = investimenti.conti()
  local function links(pagine)
    local a = {}
    for _, nome in ipairs(pagine or {}) do table.insert(a, "[[" .. nome .. "]]") end
    table.sort(a)
    return #a > 0 and table.concat(a, ", ") or "nessuno"
  end

  -- url: lista (o stringa) di link Markdown già pronti nel frontmatter.
  local urls = {}
  if type(p.url) == "table" then
    for _, u in ipairs(p.url) do if type(u) == "string" then table.insert(urls, u) end end
  elseif type(p.url) == "string" then
    table.insert(urls, p.url)
  end

  -- Orizzonte: orizzonte.min/max (anni) e/o orizzonte.scade (AAAA-MM-GG); forma annidata o con il punto.
  local o = type(p.orizzonte) == "table" and p.orizzonte or {}
  local omin = investimenti.numero(o.min) or investimenti.numero(p.omin)
  local omax = investimenti.numero(o.max) or investimenti.numero(p.omax)
  local scade = str(o.scade) or str(p.oscade)
  local orizz = {}
  if omin and omax then table.insert(orizz, "da " .. anni(omin) .. " a " .. anni(omax) .. " anni")
  elseif omin then table.insert(orizz, "almeno " .. anni(omin) .. " anni")
  elseif omax then table.insert(orizz, "fino a " .. anni(omax) .. " anni") end
  if scade and string.match(scade, "^%d%d%d%d%-%d%d%-%d%d") then
    scade = string.sub(scade, 1, 10)
    local oggi = os.date("!%Y-%m-%d")
    if scade == oggi then table.insert(orizz, "scade oggi")
    elseif scade > oggi then
      table.insert(orizz, "scade il " .. dataIt(scade) .. ", tra " .. durata(mesiTra(oggi, scade)))
    else
      table.insert(orizz, "scaduto il " .. dataIt(scade) .. ", da " .. durata(mesiTra(scade, oggi)))
    end
  end

  local nome = str(p.nome) or id
  local riga1 = "**" .. (str(p.tipo) or "Strumento") .. " · " .. (str(p.valuta) or "EUR") ..
    " · " .. rischio .. "**"
  for _, u in ipairs(urls) do riga1 = riga1 .. " · " .. u end
  if #orizz > 0 then riga1 = riga1 .. " · Orizzonte: " .. table.concat(orizz, ", ") end
  local riga2 = "**Conti:** " .. links(conti.fisici[id]) .. " · **Portafogli:** " .. links(conti.portafogli[id])
  local blocchi = {"# " .. nome}
  local desc = str(p.desc)
  if desc and desc ~= nome then table.insert(blocchi, desc) end
  table.insert(blocchi, riga1 .. "  \n" .. riga2)
  return widget.markdown(table.concat(blocchi, "\n\n"))
end

--- Pagine Quotazioni/...: restituisce isin -> pagina, pagina -> isin, isin -> true per gli isin
--- presenti in più pagine (ambigui: non vengono usati per quotazioni né aggiornamenti) e la lista
--- delle pagine senza isin.
--- Il collegamento tra Strumenti/<id> e la sua pagina quotazioni è l'attributo isin;
--- più strumenti con lo stesso isin condividono la stessa pagina (NAV condiviso).
function investimenti.pagineQuotazioni()
  local perIsin, perPagina, duplicati, senzaIsin = {}, {}, {}, {}
  for _, p in ipairs(query[[
    from p = index.subPages("Quotazioni")
    select {name = p.name, isin = p.isin}
  ]]) do
    local i = str(p.isin)
    if i then
      if perIsin[i] then duplicati[i] = true end
      perIsin[i], perPagina[p.name] = p.name, i
    else table.insert(senzaIsin, p.name) end
  end
  return perIsin, perPagina, duplicati, senzaIsin
end

--- Id degli strumenti di una pagina: Conti/ -> attributo strumento; Strumenti/ -> la pagina;
--- altrove nil (tutti gli strumenti). Senza argomento usa la pagina aperta.
--- conti: risultato di investimenti.conti(), se già disponibile.
function investimenti.elenco(pagina, conti)
  pagina = pagina or editor.getCurrentPage()
  if type(pagina) ~= "string" then return nil end
  if string.startsWith(pagina, "Strumenti/") then return {string.sub(pagina, 11)} end
  if not string.startsWith(pagina, "Conti/") then return nil end
  return (conti or investimenti.conti()).ids[pagina] or {}
end
```

```space-lua
quotazioni = quotazioni or {}

--- Serie giornaliera di un simbolo Yahoo Finance (endpoint non ufficiale).
--- Restituisce {valuta, righe = {{data, prezzo}, ...}} oppure {errore}.
function quotazioni.yahoo(simbolo, range)
  if type(simbolo) ~= "string" then return {errore = "simbolo mancante"} end
  local r = net.proxyFetch(
    "https://query1.finance.yahoo.com/v8/finance/chart/" .. simbolo ..
    "?range=" .. (range or "5d") .. "&interval=1d")
  local c = r.ok and r.body and r.body.chart
  if not c then return {errore = "HTTP " .. tostring(r.status)} end
  if not c.result then
    return {errore = c.error and c.error.description or "nessun dato"}
  end
  local res = c.result[1]
  local righe = {}
  for i, ts in ipairs(res.timestamp or {}) do
    local p = res.indicators.quote[1].close[i]
    -- date in UTC; salta i giorni senza chiusura
    if p then table.insert(righe, {data = os.date("!%Y-%m-%d", ts), prezzo = p}) end
  end
  return {valuta = res.meta.currency, righe = righe}
end

--- Prova: ultimo prezzo disponibile per ogni strumento con ticker.
function quotazioni.test()
  local out = {}
  for _, s in ipairs(investimenti.strumenti()) do
    if s.ticker then
      local q = quotazioni.yahoo(s.ticker, "5d")
      local ultima = q.righe and q.righe[#q.righe]
      table.insert(out, {
        strumento = s.id, ticker = s.ticker, valuta = q.valuta,
        data = ultima and ultima.data, prezzo = ultima and ultima.prezzo,
        punti = q.righe and #q.righe, errore = q.errore})
    end
  end
  return out
end

--- Cerca il ticker Yahoo da ISIN o nome (restituisce i risultati grezzi).
function quotazioni.cerca(testo)
  local r = net.proxyFetch(
    "https://query1.finance.yahoo.com/v1/finance/search?q=" .. testo ..
    "&quotesCount=5&newsCount=0")
  return r.ok and r.body and r.body.quotes or {errore = "HTTP " .. tostring(r.status)}
end

--- Riepilogo di una serie (valuta, punti, primo e ultimo dato) per verificare un ticker.
function quotazioni.verifica(simbolo, range)
  local q = quotazioni.yahoo(simbolo, range or "5y")
  local r = q.righe or {}
  return {{simbolo = simbolo, valuta = q.valuta, punti = #r,
    prima = r[1] and r[1].data, ultima = r[#r] and r[#r].data,
    prezzo = r[#r] and r[#r].prezzo, errore = q.errore}}
end
```

```space-lua
quotazioni = quotazioni or {}

--- Crea la pagina Quotazioni/<id> (con displayName e isin nel frontmatter, il grafico è un widget
--- automatico) per ogni strumento con ticker e isin che non ha ancora una pagina con quell'isin.
--- Restituisce gli id creati e una tabella id -> pagina (da passare ad aggiorna, perché l'indice
--- potrebbe non conoscere ancora le pagine nuove).
function quotazioni.crea()
  local perIsin, _, duplicati = investimenti.pagineQuotazioni()
  local create, pagine = {}, {}
  for _, s in ipairs(investimenti.strumenti()) do
    local nome = "Quotazioni/" .. s.id
    if s.ticker and s.isin and not perIsin[s.isin] and not duplicati[s.isin]
      and not space.pageExists(nome) then
      space.writePage(nome, "---\ndisplayName: " .. investimenti.js(s.nome) .. "\nisin: " ..
        investimenti.js(s.isin) .. "\n---\n| Data | Prezzo |\n|---|---|\n")
      table.insert(create, s.id)
      pagine[s.id] = nome
    end
  end
  return create, pagine
end

--- Legge la sola tabella "| Data | Prezzo |" di una pagina Quotazioni: restituisce
--- {prima, dopo, valori = data -> testo prezzo} oppure nil e il motivo.
function quotazioni.blocco(testo)
  if type(testo) ~= "string" then return nil, "pagina non leggibile" end
  local i, j = string.find(testo, "| Data | Prezzo |", 1, true)
  if not i or string.find(testo, "| Data | Prezzo |", j + 1, true) then
    return nil, "intestazione assente o duplicata"
  end
  local prima, resto = string.sub(testo, 1, i - 1), string.sub(testo, i)
  local righe, pos = {}, 1
  while pos <= #resto do
    local e = string.find(resto, "\n", pos, true)
    local riga = e and string.sub(resto, pos, e - 1) or string.sub(resto, pos)
    if #righe > 0 and not string.match(riga, "^%s*|") then break end
    table.insert(righe, riga)
    pos = e and e + 1 or #resto + 1
  end
  if #righe < 2 or not string.match(righe[2], "^|[%s%-%:|]+|%s*$") then
    return nil, "separatore tabella non valido"
  end
  local m = {}
  for n = 3, #righe do
    local d, v = string.match(righe[n], "^|%s*(%d%d%d%d%-%d%d%-%d%d)%s*|%s*([%d%.]+)%s*|%s*$")
    if not d or not tonumber(v) or tonumber(v) <= 0 or m[d] then
      return nil, "riga storica non valida o duplicata: " .. righe[n]
    end
    m[d] = v
  end
  return {prima = prima, dopo = string.sub(resto, pos), valori = m}
end

--- Scarica e unisce le quotazioni (per data) nelle pagine Quotazioni con lo stesso isin dello strumento.
--- range: periodo Yahoo (default "1mo"); ids: elenco di id strumento (default: strumenti con ticker);
--- pagine: tabella id -> pagina per le pagine appena create (vedi crea).
--- Aggiornamento conservativo: valida tutto prima di scrivere.
function quotazioni.aggiorna(range, ids, pagine)
  range, pagine = range or "1mo", pagine or {}
  local perIsin, _, duplicati = investimenti.pagineQuotazioni()
  local ticker, valuta, tutti = {}, {}, {}
  for _, s in ipairs(investimenti.strumenti()) do
    if s.ticker then
      ticker[s.id], valuta[s.id] = s.ticker, s.valuta
      table.insert(tutti, s.id)
      pagine[s.id] = pagine[s.id] or (s.isin and not duplicati[s.isin] and perIsin[s.isin]) or nil
    end
  end
  ids = ids or tutti
  local esiti, errori = {}, 0
  for _, id in ipairs(ids) do
    local nome, errore, nuovi = pagine[id], nil, nil
    if not ticker[id] then errore = "ticker mancante"
    elseif not nome then errore = "pagina mancante o ISIN duplicato"
    elseif not space.pageExists(nome) then errore = "pagina inesistente"
    else
      local testo = space.readPage(nome)
      local blocco, e = quotazioni.blocco(testo)
      if not blocco then errore = e
      else
        local q = quotazioni.yahoo(ticker[id], range)
        if q.errore then errore = q.errore
        elseif not q.valuta or string.upper(q.valuta) ~= string.upper(valuta[id] or "EUR") then
          errore = "valuta Yahoo non coerente: " .. tostring(q.valuta)
        elseif not q.righe or #q.righe == 0 then errore = "nessuna quotazione"
        else
          local valori, valido = {}, true
          for _, r in ipairs(q.righe) do
            if type(r.data) ~= "string" or not string.match(r.data, "^%d%d%d%d%-%d%d%-%d%d$")
              or type(r.prezzo) ~= "number" or r.prezzo <= 0 or r.prezzo ~= r.prezzo
              or r.prezzo == math.huge or r.prezzo == -math.huge then
              valido = false
              break
            end
            valori[r.data] = string.format("%.6f", r.prezzo)
          end
          if not valido then errore = "serie scaricata non valida"
          else
            nuovi = #q.righe
            for d, v in pairs(valori) do blocco.valori[d] = v end
            local date, righe = {}, {"| Data | Prezzo |", "|---|---|"}
            for d in pairs(blocco.valori) do table.insert(date, d) end
            table.sort(date, function(a, b) return a > b end)
            for _, d in ipairs(date) do
              table.insert(righe, "| " .. d .. " | " .. blocco.valori[d] .. " |")
            end
            local aggiornato = blocco.prima .. table.concat(righe, "\n") .. "\n" .. blocco.dopo
            if aggiornato ~= testo then space.writePage(nome, aggiornato) end
          end
        end
      end
    end
    if errore then errori = errori + 1 end
    table.insert(esiti, {id = id, pagina = nome, righe = nuovi, errore = errore})
  end
  editor.flashNotification(#esiti .. " strumenti, " .. errori .. " errori",
    errori > 0 and "warning" or "info")
  return esiti
end
```

```space-lua
investimenti = investimenti or {}

--- Anomalie del collegamento tra strumenti e pagine Quotazioni: isin mancante o presente in più
--- pagine (in tal caso nessuna delle pagine è usata). Più strumenti con lo stesso isin sono ammessi
--- (NAV condiviso). Prezzi mancanti e tabelle illeggibili emergono dalla copertura e da aggiorna().
function investimenti.verifica()
  local _, perPagina, duplicati, senzaIsin = investimenti.pagineQuotazioni()
  local anomalie = {}
  for pagina, isin in pairs(perPagina) do
    if duplicati[isin] then table.insert(anomalie, {pagina = pagina, problema = "ISIN_DUPLICATO", dettaglio = isin}) end
  end
  for _, pagina in ipairs(senzaIsin) do
    table.insert(anomalie, {pagina = pagina, problema = "ISIN_MANCANTE", dettaglio = "Frontmatter senza isin"})
  end
  table.sort(anomalie, function(a, b) return a.pagina < b.pagina end)
  return anomalie
end
```

```space-lua
command.define {
  name = "Quotazioni: Crea pagine e scarica storico",
  run = function()
    local ids, pagine = quotazioni.crea()
    quotazioni.aggiorna("5y", ids, pagine)
  end
}
```

```space-lua
command.define {
  name = "Quotazioni: Aggiorna",
  run = function()
    quotazioni.aggiorna("1mo")
  end
}
```

```space-lua
investimenti = investimenti or {}

local function str(x) if type(x) == "string" then return x end end

local function haTag(tags, nome)
  if type(tags) == "string" then return string.find(tags, nome, 1, true) ~= nil end
  if type(tags) == "table" then
    for _, t in ipairs(tags) do if t == nome then return true end end
  end
  return false
end

--- Conti: una sola lettura di Conti/. Restituisce
--- ids[conto] = id degli strumenti dell'attributo `strumento` ("[[Strumenti/X|alias]]" -> "X"),
--- aggregati[conto] = true per i portafogli logici (tag Portafoglio),
--- fisici[id] / portafogli[id] = conti che contengono lo strumento.
function investimenti.conti()
  local c = {ids = {}, aggregati = {}, fisici = {}, portafogli = {}}
  for _, p in ipairs(query[[
    from p = index.subPages("Conti")
    select {name = p.name, tags = p.tags, strumento = p.strumento}
  ]]) do
    local aggregato = haTag(p.tags, "Portafoglio")
    local lista, ids = p.strumento, {}
    if type(lista) == "string" then lista = {lista} end
    if type(lista) == "table" then
      for _, v in ipairs(lista) do
        local t = type(v) == "string" and string.match((string.gsub(v, "[%[%]]", "")), "^[^|]+")
        if t and string.startsWith(t, "Strumenti/") then table.insert(ids, string.sub(t, 11)) end
      end
    end
    c.ids[p.name], c.aggregati[p.name] = ids, aggregato
    local dest = aggregato and c.portafogli or c.fisici
    for _, id in ipairs(ids) do
      dest[id] = dest[id] or {}
      table.insert(dest[id], p.name)
    end
  end
  return c
end

--- Risolve solo il conto fisico univoco; portafoglio unico come ripiego.
--- Se l'assegnazione esplicita manca dall'archivio, viene mantenuta e diagnosticata.
function investimenti.risolviConto(id, esplicito, fisici, portfolio)
  if type(esplicito) == "string" and esplicito ~= "" then
    if string.startsWith(esplicito, "Conti/") then return esplicito end
    return "Conti/" .. esplicito
  end
  local possibili = fisici[id] or {}
  if #possibili == 1 then return possibili[1] end
  if #possibili > 1 then return nil end
  local p = portfolio[id] or {}
  if #p == 1 then return p[1] end
  return nil
end

-- A parità di data: acquisti, altri movimenti, vendite (una vendita non precede mai l'acquisto).
local ORDINE = {Acquisto = 1, Vendita = 3}

-- Validità di una riga di movimento, controllata una sola volta: nil se valida.
local function erroreRiga(tipo, data, q, v)
  if not data then return "DATA_NON_VALIDA" end
  if tipo == "Acquisto" then
    if not q or q <= 0 or not v or v <= 0 then return "MOVIMENTO_NON_VALIDO" end
  elseif tipo == "Vendita" then
    if not q or q <= 0 or not v or v < 0 then return "MOVIMENTO_NON_VALIDO" end
  elseif tipo == "Dividendo" or tipo == "Cedola" then
    if not v or v < 0 then return "REDDITO_NON_VALIDO" end
  elseif tipo == "Snapshot" then
    if not v or v < 0 then return "SNAPSHOT_NON_VALIDO" end
  end
end

-- "acquisto", "ACQUISTO" -> "Acquisto": il tipo non dipende da maiuscole e minuscole.
local function canonico(t) return string.upper(string.sub(t, 1, 1)) .. string.lower(string.sub(t, 2)) end

--- Contesto: tutto ciò che serve ai calcoli, letto UNA volta (3 scansioni di pagine, 1 di tabelle)
--- e passato come ultimo argomento facoltativo alle funzioni. Campi:
---   strumenti (lista ordinata) e perId (id -> strumento, con metodoDichiarato/frequenza/quoteReali);
---   conti (vedi investimenti.conti);
---   serie: id -> lista {data, prezzo} ordinata, dalla pagina Quotazioni con lo stesso isin
---          (pagine con isin duplicato escluse);
---   ledger: solo le righe di movimento (tabelle di Strumenti/ con colonna Tipo), con il conto
---          risolto, i valori numerici (virgola decimale ammessa), la data (nil se non nel
---          formato AAAA-MM-GG) e `errore` (nil se la riga è valida: unica validazione), in
---          ordine cronologico.
--- Un contesto è un'istantanea: va ricreato a ogni calcolo/widget, non conservato.
function investimenti.contesto()
  local c = {conti = investimenti.conti(), strumenti = investimenti.strumenti(),
    perId = {}, serie = {}, ledger = {}}
  local _, perPagina, duplicati = investimenti.pagineQuotazioni()
  local idDi = {}  -- isin -> id strumenti: PIR Marco e Franca possono condividere un NAV.
  for _, s in ipairs(c.strumenti) do
    c.perId[s.id] = s
    if s.isin and not duplicati[s.isin] then
      idDi[s.isin] = idDi[s.isin] or {}
      table.insert(idDi[s.isin], s.id)
    end
  end
  for _, r in ipairs(query[[
    from t = index.tables()
    where string.startsWith(t.page, "Strumenti/") or string.startsWith(t.page, "Quotazioni/")
    select {page = t.page, ref = t.ref, giorno = t.data, tipo = t.tipo, quote = t.quantit_,
      valore = t.valore_eur or t.valore, conto = t.conto, px = t.prezzo}
  ]]) do
    if string.startsWith(r.page, "Quotazioni/") then
      local pr = investimenti.numero(r.px)
      local ids = idDi[perPagina[r.page]]
      if ids and type(r.giorno) == "string" and pr and pr > 0 then
        for _, id in ipairs(ids) do
          c.serie[id] = c.serie[id] or {}
          table.insert(c.serie[id], {r.giorno, pr})
        end
      end
    elseif type(r.tipo) == "string" then  -- le tabelle senza Tipo non sono movimenti
      local id = string.sub(r.page, 11)
      local tipo = canonico(r.tipo)
      local data = type(r.giorno) == "string" and string.match(r.giorno, "^%d%d%d%d%-%d%d%-%d%d$") and r.giorno or nil
      local q, v = investimenti.numero(r.quote), investimenti.numero(r.valore)
      table.insert(c.ledger, {id = id,
        conto = investimenti.risolviConto(id, r.conto, c.conti.fisici, c.conti.portafogli),
        tipo = tipo, dataGrezza = tostring(r.giorno), data = data, quote = q, valore = v,
        errore = erroreRiga(tipo, data, q, v), ref = r.ref})
    end
  end
  for _, lista in pairs(c.serie) do table.sort(lista, function(a, b) return a[1] < b[1] end) end
  table.sort(c.ledger, function(a, b)
    if a.dataGrezza ~= b.dataGrezza then return a.dataGrezza < b.dataGrezza end
    local x, y = ORDINE[a.tipo] or 2, ORDINE[b.tipo] or 2
    if x ~= y then return x < y end
    return tostring(a.ref or "") < tostring(b.ref or "")
  end)
  return c
end

--- Perimetro di una pagina: ammessi (id -> true; nil = tutti gli strumenti),
--- conto (la pagina se è un Conti/..., altrimenti nil) e aggregato (true se è un portafoglio).
function investimenti.perimetro(pagina, ctx)
  local ids = investimenti.elenco(pagina, ctx.conti)
  local ammessi
  if ids then
    ammessi = {}
    for _, id in ipairs(ids) do ammessi[id] = true end
  end
  local conto = type(pagina) == "string" and string.startsWith(pagina, "Conti/") and pagina or nil
  return ammessi, conto, conto ~= nil and ctx.conti.aggregati[conto] == true
end

--- Anomalie contabili: strumenti inesistenti nei conti, movimenti non assegnati a un conto e
--- righe non valide (data, acquisti, vendite, redditi, snapshot). ctx facoltativo.
function investimenti.verificaConti(ctx)
  ctx = ctx or investimenti.contesto()
  local out = {}
  local function errore(id, tipo, dettaglio)
    table.insert(out, {pagina = "Strumenti/" .. id, id = id, problema = tipo, dettaglio = dettaglio})
  end
  for _, lista in ipairs({ctx.conti.fisici, ctx.conti.portafogli}) do
    for id, pp in pairs(lista) do
      if not ctx.perId[id] then errore(id, "STRUMENTO_INESISTENTE", table.concat(pp, ", ")) end
    end
  end
  for _, r in ipairs(ctx.ledger) do
    if r.errore then errore(r.id, r.errore, r.dataGrezza) end
    if not r.conto then errore(r.id, "CONTO_NON_ASSEGNATO", r.dataGrezza) end
  end
  return out
end
```

```space-lua
investimenti = investimenti or {}

--- Quotazioni di una pagina Quotazioni/...: lista di {data, prezzo}, dalla più vecchia.
function investimenti.serie(pagina)
  local out = {}
  for _, r in ipairs(query[[
    from t = index.tables()
    where t.page == pagina
    select {ref = t.ref, giorno = t.data, px = t.prezzo}
  ]]) do
    local pr = investimenti.numero(r.px)
    if type(r.giorno) == "string" and pr then table.insert(out, {r.giorno, pr}) end
  end
  table.sort(out, function(a, b) return a[1] < b[1] end)
  return out
end

--- Acquisti di uno strumento: lista di {data, prezzo unitario in valuta strumento}.
function investimenti.acquisti(id)
  local out = {}
  for _, r in ipairs(query[[
    from t = index.tables()
    where t.page == "Strumenti/" .. id and t.tipo == "Acquisto"
    select {ref = t.ref, giorno = t.data, px = t.prezzo}
  ]]) do
    local pr = investimenti.numero(r.px)
    if type(r.giorno) == "string" and pr then table.insert(out, {r.giorno, pr}) end
  end
  table.sort(out, function(a, b) return a[1] < b[1] end)
  return out
end

-- Script del grafico lineare: prezzi, acquisti, MM200, Bollinger e RSI.
-- serie e acquisti sono liste di [data, prezzo].
-- Gli indicatori (finestre in osservazioni, cioè in quotazioni) sono calcolati sull'intera serie
-- e poi ritagliati sul periodo scelto, così all'inizio del periodo sono già completi.
local JS_LINEA = [[
const RANGE = [{n:"7G",d:7},{n:"1M",d:30},{n:"3M",d:91},{n:"6M",d:182},{n:"YTD",d:-1},{n:"1A",d:365},{n:"5A",d:1826},{n:"Tutto",d:0}];
const dark = window.matchMedia && window.matchMedia("(prefers-color-scheme: dark)").matches;
const txt = dark ? "#ddd" : "#333", grid = "rgba(128,128,128,.25)";
const S = DATI.serie, P = S.map(function (p) { return p[1]; });
const ultimo = S[S.length - 1][0];
function mediaMobile(n) {
  return P.map(function (v, i) {
    if (i < n - 1) return null;
    let s = 0;
    for (let k = i - n + 1; k <= i; k++) s += P[k];
    return s / n;
  });
}
function scarto(n, m) {
  return P.map(function (v, i) {
    if (i < n - 1) return null;
    let s = 0;
    for (let k = i - n + 1; k <= i; k++) s += (P[k] - m[i]) * (P[k] - m[i]);
    return Math.sqrt(s / n);
  });
}
function rsi(n) {
  const out = P.map(function () { return null; });
  if (P.length <= n) return out;
  let g = 0, l = 0;
  for (let i = 1; i <= n; i++) {
    const d = P[i] - P[i - 1];
    if (d > 0) g += d; else l -= d;
  }
  g /= n; l /= n;
  out[n] = l === 0 ? 100 : 100 - 100 / (1 + g / l);
  for (let i = n + 1; i < P.length; i++) {
    const d = P[i] - P[i - 1];
    g = (g * (n - 1) + (d > 0 ? d : 0)) / n;
    l = (l * (n - 1) + (d < 0 ? -d : 0)) / n;
    out[i] = l === 0 ? 100 : 100 - 100 / (1 + g / l);
  }
  return out;
}
const MM = mediaMobile(200);
const BM = mediaMobile(20), BS = scarto(20, BM);
const BU = BM.map(function (m, i) { return m === null ? null : m + 2 * BS[i]; });
const BL = BM.map(function (m, i) { return m === null ? null : m - 2 * BS[i]; });
const RS = rsi(14);
function da(r) {
  if (r.d === 0) return "0000-00-00";
  if (r.d < 0) return ultimo.slice(0, 4) + "-01-01";
  const d = new Date(ultimo + "T00:00:00Z");
  d.setUTCDate(d.getUTCDate() - r.d);
  return d.toISOString().slice(0, 10);
}
const attivi = {};
let periodo = RANGE[3], grafico = null, graficoRsi = null;
function linea(nome, dati, colore, tratto) {
  return {label: nome, data: dati, borderColor: colore, borderWidth: 1.2, pointRadius: 0,
    tension: 0, borderDash: tratto || []};
}
function asseY() {
  return {ticks: {color: txt}, grid: {color: grid}, afterFit: function (a) { a.width = 56; }};
}
function asseX(visibile) {
  return {ticks: {color: txt, maxTicksLimit: 8}, grid: {color: grid}, display: visibile};
}
function disegna() {
  const da0 = da(periodo);
  const ix = [];
  S.forEach(function (p, i) { if (p[0] >= da0) ix.push(i); });
  const lab = ix.map(function (i) { return S[i][0]; });
  const scegli = function (a) { return ix.map(function (i) { return a[i]; }); };
  const fisso = function (v) { return lab.map(function () { return v; }); };
  const buy = lab.map(function () { return null; });
  DATI.acquisti.forEach(function (a) {
    const i = lab.findIndex(function (d) { return d >= a[0]; });
    if (i >= 0 && a[0] >= da0 && a[0] >= lab[0]) buy[i] = a[1];
  });
  const ds = [{label: DATI.titolo, data: scegli(P), borderColor: "#2f6fdb", borderWidth: 1.5,
    pointRadius: 0, tension: 0}];
  if (DATI.acquisti.length) ds.push({label: "Acquisti", data: buy, showLine: false,
    pointStyle: "triangle", pointRadius: 7, backgroundColor: "#d9822b", borderColor: "#d9822b"});
  if (attivi.mm) ds.push(linea("MM200", scegli(MM), "#9b5de5"));
  if (attivi.bb) {
    ds.push(linea("Bollinger +2σ", scegli(BU), "#7d8597", [4, 3]));
    ds.push(linea("Bollinger −2σ", scegli(BL), "#7d8597", [4, 3]));
  }
  if (grafico) grafico.destroy();
  grafico = new Chart(document.getElementById("c"), {type: "line",
    data: {labels: lab, datasets: ds},
    options: {responsive: true, maintainAspectRatio: false, animation: false,
      interaction: {mode: "nearest", axis: "x", intersect: false},
      plugins: {legend: {display: ds.length > 1, labels: {color: txt}}},
      scales: {x: asseX(true), y: asseY()}}});
  const box = document.getElementById("rw");
  box.style.display = attivi.rsi ? "block" : "none";
  if (graficoRsi) { graficoRsi.destroy(); graficoRsi = null; }
  if (attivi.rsi) {
    graficoRsi = new Chart(document.getElementById("r"), {type: "line",
      data: {labels: lab, datasets: [linea("RSI 14", scegli(RS), "#2f6fdb"),
        linea("70", fisso(70), "#e4572e", [4, 3]), linea("30", fisso(30), "#2a9d4a", [4, 3])]},
      options: {responsive: true, maintainAspectRatio: false, animation: false,
        plugins: {legend: {display: false}, title: {display: true, text: "RSI 14", color: txt}},
        scales: {x: asseX(false), y: Object.assign(asseY(), {min: 0, max: 100})}}});
  }
}
const bottoni = [];
function stile() {
  bottoni.forEach(function (b) {
    b.el.style.fontWeight = b.attivo() ? "bold" : "normal";
    b.el.style.background = b.attivo() ? "rgba(47,111,219,.25)" : "";
  });
}
function bottone(riga, nome, attivo, azione) {
  const el = document.createElement("button");
  el.textContent = nome;
  el.style.cssText = "margin:0 4px 6px 0;padding:2px 10px;cursor:pointer";
  el.onclick = function () { azione(); stile(); disegna(); };
  bottoni.push({el: el, attivo: attivo});
  riga.appendChild(el);
}
const barra = document.getElementById("b");
RANGE.forEach(function (r) {
  bottone(barra, r.n, function () { return periodo === r; }, function () { periodo = r; });
});
const indicatori = [{k: "mm", n: "MM200"}, {k: "bb", n: "Bollinger"}, {k: "rsi", n: "RSI"}];
const barra2 = document.getElementById("b2");
indicatori.forEach(function (x) {
  bottone(barra2, x.n, function () { return !!attivi[x.k]; }, function () { attivi[x.k] = !attivi[x.k]; });
});
stile();
disegna();
]]

--- Grafico lineare delle quotazioni con pulsanti 7G/1M/3M/6M/YTD/1A/5A/Tutto.
--- Su Strumenti/ aggiunge i punti di acquisto. pagina: Quotazioni/... o Strumenti/<id>.
function investimenti.grafico(pagina)
  pagina = pagina or editor.getCurrentPage()
  if type(pagina) ~= "string" then return nil end
  local perIsin, perPagina, duplicati = investimenti.pagineQuotazioni()
  local id, quot, conAcquisti, nome
  if string.startsWith(pagina, "Quotazioni/") then
    quot = pagina
    for _, s in ipairs(investimenti.strumenti()) do
      if s.isin and s.isin == perPagina[pagina] then id, nome = s.id, s.nome end
    end
  elseif string.startsWith(pagina, "Strumenti/") then
    id, conAcquisti = string.sub(pagina, 11), true
    for _, s in ipairs(investimenti.strumenti()) do
      if s.id == id then
        nome = s.nome
        quot = s.isin and not duplicati[s.isin] and perIsin[s.isin] or nil
      end
    end
  else return widget.markdownBlock("_Il grafico vale per pagine Quotazioni/ e Strumenti/._") end
  if not quot then
    return widget.markdownBlock("_Nessuna pagina Quotazioni univoca con l'isin dello strumento._")
  end

  local serie = investimenti.serie(quot)
  if #serie < 2 then return widget.markdownBlock("_Quotazioni insufficienti per il grafico._") end
  local acq = conAcquisti and investimenti.acquisti(id) or {}

  local function coppie(lista)
    local p = {}
    for _, c in ipairs(lista) do table.insert(p, "[" .. investimenti.js(c[1]) .. "," .. c[2] .. "]") end
    return "[" .. table.concat(p, ",") .. "]"
  end
  nome = nome or id or quot
  local dati = "const DATI = {titolo: " .. investimenti.js(nome) .. ", serie: " .. coppie(serie) ..
    ", acquisti: " .. coppie(acq) .. "};\n"
  return widget.sandbox {
    html = '<div id="b"></div><div id="b2"></div><div style="position:relative;height:300px">' ..
      '<canvas id="c"></canvas></div><div id="rw" style="display:none;position:relative;height:140px;' ..
      'margin-top:6px"><canvas id="r"></canvas></div>',
    script = 'loadJsByUrl("' .. investimenti.chartjs .. '").then(function () {\n' .. dati ..
      JS_LINEA .. "\n});",
    markdown = "Grafico " .. nome,
  }
end
```

```space-lua
command.define {
  name = "Investimenti: Verifica conti e movimenti",
  run = function()
    local problemi = investimenti.verificaConti()
    editor.flashNotification(#problemi .. " anomalie contabili",
      #problemi > 0 and "warning" or "info")
    return problemi
  end
}
```

```space-lua
investimenti = investimenti or {}

local function oggi() return os.date("!%Y-%m-%d") end

-- Nomi dei metodi: i quattro canonici e i sinonimi delle versioni precedenti.
local METODI = {prezzo = "prezzo", mercato = "prezzo", nav = "prezzo", obbligazione = "prezzo",
  snapshot = "snapshot", manuale = "snapshot",
  contrattuale = "contrattuale", nominale = "contrattuale", costo = "costo"}

--- Metodo di valorizzazione di uno strumento di strumenti(): l'attributo metodoValorizzazione
--- se valido, altrimenti PIR -> snapshot e tutto il resto -> prezzo.
--- prezzo: quote x ultimo prezzo; snapshot: ultimo rendiconto + apporti successivi;
--- contrattuale: prezzo manuale o costo; costo: costo residuo.
function investimenti.metodo(s)
  if type(s) ~= "table" then return "snapshot" end
  return METODI[s.metodoDichiarato] or (s.tipo == "PIR" and "snapshot" or "prezzo")
end

-- Ultimo prezzo non successivo alla data: ricerca binaria sulla serie ordinata per data.
local function prezzoPrima(serie, data)
  local lo, hi, scelta = 1, serie and #serie or 0, nil
  while lo <= hi do
    local m = math.floor((lo + hi) / 2)
    if serie[m][1] <= data then scelta, lo = serie[m], m + 1 else hi = m - 1 end
  end
  return scelta
end

local function giorniTraDate(a, b)
  local function giorno(x)
    if type(x) ~= "string" then return nil end
    local y, m, d = string.match(x, "^(%d%d%d%d)%-(%d%d)%-(%d%d)$")
    if not y then return nil end
    -- Giorni civili gregoriani: nessuna dipendenza da os.time/timezone.
    y, m, d = tonumber(y), tonumber(m), tonumber(d)
    if m < 1 or m > 12 or d < 1 or d > 31 then return nil end
    if m <= 2 then y = y - 1 end
    local era = math.floor(y / 400)
    local yo = y - era * 400
    local mp = m + (m > 2 and -3 or 9)
    local doy = math.floor((153 * mp + 2) / 5) + d - 1
    return era * 146097 + yo * 365 + math.floor(yo / 4) - math.floor(yo / 100) + doy
  end
  local x, y = giorno(a), giorno(b)
  return x and y and (y - x) or nil
end

--- Ultimo prezzo di ogni strumento alla data, convertito in EUR: id -> {data, prezzo, eur}.
--- Valute diverse da EUR e USD non sono convertibili: eur = nil (stato CAMBIO_MANCANTE).
function investimenti.prezzi(data, ctx)
  data = data or oggi()
  ctx = ctx or investimenti.contesto()
  local px = {}
  for _, s in ipairs(ctx.strumenti) do
    local r = prezzoPrima(ctx.serie[s.id], data)
    if r then
      local eur = r[2]
      if s.valuta == "USD" and s.id ~= "EURUSD" then
        local cambio = prezzoPrima(ctx.serie.EURUSD, r[1])
        eur = cambio and eur / cambio[2] or nil
      elseif s.valuta ~= "EUR" and s.valuta ~= "USD" then eur = nil end
      px[s.id] = {data = r[1], prezzo = r[2], eur = eur}
    end
  end
  return px
end

--- Posizioni PMC per conto + strumento, calcolate cronologicamente sul ledger del contesto.
--- Ogni posizione porta anche l'ultimo snapshot valido (snapshot = {data, valore}) e gli apporti
--- netti (acquisti meno vendite) successivi alla data dello snapshot: PROXY contabile,
--- non una variazione di mercato certificata.
function investimenti.lotti(pagina, data, ctx)
  data = data or oggi()
  ctx = ctx or investimenti.contesto()
  local ammessi, conto, aggregato = investimenti.perimetro(pagina, ctx)
  local per, chiavi = {}, {}
  for _, r in ipairs(ctx.ledger) do
    local include = (not ammessi or ammessi[r.id]) and r.data and r.data <= data
    if include and conto then
      include = r.conto ~= nil and (r.conto == conto or aggregato)
    end
    if include then
      local key = (r.conto or "NON_ASSEGNATO") .. "|" .. r.id
      local p = per[key]
      if not p then
        p = {id = r.id, conto = r.conto, quote = 0, costo = 0, realizzato = 0, redditi = 0,
          errori = {}, flussi = {}, snapshot = nil, apporti = 0}
        per[key] = p
        table.insert(chiavi, key)
      end
      local v, q = r.valore, r.quote
      if r.tipo == "Snapshot" then
        -- Un snapshot non valido non è usato e non invalida la posizione.
        if not r.errore then p.snapshot, p.apporti = {data = r.data, valore = v}, 0 end
      elseif r.errore then
        table.insert(p.errori, r.errore .. " " .. r.data)
      elseif r.tipo == "Acquisto" then
        p.quote, p.costo = p.quote + q, p.costo + v
        table.insert(p.flussi, {data = r.data, valore = -v})
        if p.snapshot and r.data > p.snapshot.data then p.apporti = p.apporti + v end
      elseif r.tipo == "Vendita" then
        if q > p.quote + 1e-7 then
          table.insert(p.errori, "VENDITA_ECCEDENTE " .. r.data)
        else
          local costoVenduto = p.quote > 0 and p.costo * q / p.quote or 0
          p.realizzato = p.realizzato + v - costoVenduto
          p.quote, p.costo = p.quote - q, p.costo - costoVenduto
          if math.abs(p.quote) < 1e-7 then p.quote, p.costo = 0, 0 end
          table.insert(p.flussi, {data = r.data, valore = v})
          if p.snapshot and r.data > p.snapshot.data then p.apporti = p.apporti - v end
        end
      elseif r.tipo == "Dividendo" or r.tipo == "Cedola" then
        p.redditi = p.redditi + v
        table.insert(p.flussi, {data = r.data, valore = v})
      end
    end
  end
  table.sort(chiavi)
  local out = {}
  for _, k in ipairs(chiavi) do table.insert(out, per[k]) end
  return out
end

--- Valorizza le posizioni del perimetro alla data (oggi se omessa). Ogni riga:
--- {id, conto, metodo, quote, costo, valore, stato, copertura, dataPrezzo, lotto}.
--- stato: come è stato ottenuto il valore; copertura: quanto è affidabile e recente
--- (AGGIORNATO, SNAPSHOT_UFFICIALE, OBSOLETO, INDICATIVO, MANCANTE, CHIUSA).
--- Soglia di età: frequenzaPrezzoGiorni dello strumento, altrimenti 365 giorni per snapshot
--- e contrattuali, 7 per gli altri. Vale per prezzi e snapshot.
function investimenti.valoreAllaData(pagina, data, ctx)
  data = data or oggi()
  ctx = ctx or investimenti.contesto()
  local prezzi = investimenti.prezzi(data, ctx)
  local righe = {}
  for _, p in ipairs(investimenti.lotti(pagina, data, ctx)) do
    local s = ctx.perId[p.id] or {id = p.id, tipo = ""}
    local metodo = investimenti.metodo(s)
    local px = prezzi[p.id]
    local valore, stato, dt
    if p.quote == 0 then valore, stato = 0, "CHIUSA"
    elseif #p.errori > 0 then stato = "MOVIMENTI_INVALIDI"
    elseif metodo == "prezzo" then
      if px and px.eur then valore, dt, stato = p.quote * px.eur, px.data, "PREZZO_DISPONIBILE"
      else stato = px and "CAMBIO_MANCANTE" or "PREZZO_MANCANTE" end
    elseif metodo == "snapshot" then
      if p.snapshot then
        valore, dt = p.snapshot.valore + p.apporti, p.snapshot.data
        stato = math.abs(p.apporti) > 0.000001 and "SNAPSHOT_CON_APPORTI" or "SNAPSHOT_DISPONIBILE"
      else stato = "SNAPSHOT_MANCANTE" end
    elseif metodo == "contrattuale" then
      -- Prezzo manuale datato se esiste, altrimenti costo: mai un prezzo di mercato.
      if px and px.eur then valore, dt, stato = p.quote * px.eur, px.data, "PREZZO_MANUALE_CONTRATTUALE"
      else valore, stato = p.costo, "COSTO_PROXY_CONTRATTUALE" end
    else valore, stato = p.costo, "COSTO_NON_MERCATO" end

    local copertura
    if stato == "CHIUSA" then copertura = "CHIUSA"
    elseif valore == nil then copertura = "MANCANTE"
    elseif stato == "PREZZO_DISPONIBILE" or stato == "SNAPSHOT_DISPONIBILE" then
      local limite = s.frequenza or ((metodo == "snapshot" or metodo == "contrattuale") and 365 or 7)
      local eta = giorniTraDate(dt, data)
      if eta and eta >= 0 and eta <= limite then
        copertura = stato == "PREZZO_DISPONIBILE" and "AGGIORNATO" or "SNAPSHOT_UFFICIALE"
      else copertura = "OBSOLETO" end
    else copertura = "INDICATIVO" end

    table.insert(righe, {id = p.id, conto = p.conto, metodo = metodo, quote = p.quote,
      costo = p.costo, valore = valore, stato = stato, copertura = copertura,
      dataPrezzo = dt, lotto = p})
  end
  return righe
end

function investimenti.risultatoDati(pagina, data, ctx)
  local out, tot = {}, {realizzato = 0, latente = 0, redditi = 0, complessivo = 0, indisponibili = 0}
  for _, v in ipairs(investimenti.valoreAllaData(pagina, data, ctx)) do
    local p = v.lotto
    local latente = v.valore and (v.valore - p.costo) or nil
    local risultato = latente and (p.realizzato + latente + p.redditi) or nil
    tot.realizzato, tot.redditi = tot.realizzato + p.realizzato, tot.redditi + p.redditi
    if latente then tot.latente, tot.complessivo = tot.latente + latente, tot.complessivo + risultato
    else tot.indisponibili = tot.indisponibili + 1 end
    table.insert(out, {id = p.id, conto = p.conto, costo = p.costo,
      realizzato = p.realizzato, latente = latente, redditi = p.redditi,
      risultato = risultato, stato = v.stato, errori = p.errori})
  end
  tot.completo = tot.indisponibili == 0
  return out, tot
end

function investimenti.risultato(pagina)
  local _, t = investimenti.risultatoDati(pagina)
  local testo = ": realizzato " .. investimenti.formatoNumero(t.realizzato, 2) ..
    "; redditi " .. investimenti.formatoNumero(t.redditi, 2) ..
    "; latente " .. investimenti.formatoNumero(t.latente, 2)
  -- Un elemento inline evita il margine verticale del paragrafo Markdown.
  -- I contenitori dei widget, gestiti da SilverBullet, non sono modificati.
  if t.completo then
    return widget.html(dom.span {
      dom.strong {"Risultato economico (EUR)"},
      testo .. "; totale " .. investimenti.formatoNumero(t.complessivo, 2)
    })
  end
  return widget.html(dom.span {
    dom.strong {"Risultato economico (EUR)"}, testo .. "; ",
    dom.strong {"totale non disponibile: posizioni senza valore " .. t.indisponibili}
  })
end

-- XIRR (base 365 giorni) su flussi ordinati {data, valore}; restituisce la percentuale o nil e il motivo.
-- La radice è cercata su intervalli fissi di tasso: serve esattamente una radice, altrimenti il
-- risultato non è certificabile (NESSUNA_RADICE / RADICI_MULTIPLE). Un NPV nullo (entro la
-- tolleranza) in un nodo è una radice esatta: il rendimento 0 % non è un caso di radici multiple.
local function xirr(flussi)
  if #flussi < 2 then return nil, "FLUSSI_INSUFFICIENTI" end
  local d0, scala = flussi[1].data, 0
  local neg, pos = false, false
  for _, f in ipairs(flussi) do
    if f.valore < 0 then neg = true end
    if f.valore > 0 then pos = true end
    scala = scala + math.abs(f.valore)
  end
  if not neg or not pos then return nil, "SEGNI_INSUFFICIENTI" end
  local function npv(r)
    local somma = 0
    for _, f in ipairs(flussi) do
      local delta = giorniTraDate(d0, f.data)
      if not delta then return nil end
      somma = somma + f.valore / ((1 + r) ^ (delta / 365))
    end
    return somma
  end
  local eps = scala * 1e-9
  local nodi = {-0.9999, -0.99, -0.9, -0.5, 0, 0.05, 0.1, 0.25, 0.5, 1, 2, 5, 10, 100, 10000}
  local f, segno = {}, {}
  for i, r in ipairs(nodi) do
    f[i] = npv(r)
    if not f[i] then return nil, "DATE_NON_VALIDE" end
    segno[i] = f[i] > eps and 1 or (f[i] < -eps and -1 or 0)
  end
  local radici, esatta, a = 0, nil, nil
  for i = 1, #nodi do
    if segno[i] == 0 then radici, esatta = radici + 1, nodi[i]
    elseif i > 1 and segno[i - 1] * segno[i] < 0 then radici, a = radici + 1, i - 1 end
  end
  if radici ~= 1 then return nil, radici == 0 and "NESSUNA_RADICE" or "RADICI_MULTIPLE" end
  if esatta then return esatta * 100 end
  local x, y, fx = nodi[a], nodi[a + 1], f[a]
  for _ = 1, 200 do
    local m = (x + y) / 2
    local fm = npv(m)
    if math.abs(fm) < 1e-8 then return m * 100 end
    if fx * fm <= 0 then y = m else x, fx = m, fm end
  end
  return ((x + y) / 2) * 100
end

--- Rendimento XIRR della singola posizione di uno strumento (pagina Strumenti/<id>).
--- Restituisce {metodo, percentuale, nota} oppure {errore, dettaglio}.
function investimenti.rendimentoDati(pagina)
  pagina = pagina or editor.getCurrentPage()
  local id = type(pagina) == "string" and string.startsWith(pagina, "Strumenti/") and string.sub(pagina, 11)
  if not id then return {errore = "PERIMETRO_NON_RICONCILIATO", dettaglio = "Richiesto singolo strumento: mancano liquidita e flussi esterni del conto"} end
  local data = oggi()
  local valori = investimenti.valoreAllaData(pagina, data)
  if #valori ~= 1 then return {errore = "POSIZIONI_MULTIPLE", dettaglio = "Serve una posizione univoca per conto"} end
  local v = valori[1]
  local p = v.lotto
  if #p.errori > 0 then return {errore = "MOVIMENTI_INVALIDI", dettaglio = table.concat(p.errori, "; ")} end
  if v.valore == nil then return {errore = v.stato} end
  if v.stato ~= "PREZZO_DISPONIBILE" and v.stato ~= "CHIUSA" then
    return {errore = "VALORE_INDICATIVO", dettaglio = v.stato}
  end
  local flussi = {}
  for _, f in ipairs(p.flussi) do table.insert(flussi, {data = f.data, valore = f.valore}) end
  if p.quote ~= 0 then table.insert(flussi, {data = data, valore = v.valore}) end
  table.sort(flussi, function(a, b) return a.data < b.data end)
  local rendimento, errore = xirr(flussi)
  return {metodo = "xirr", percentuale = rendimento, errore = errore,
    nota = "XIRR annualizzato; flussi della singola posizione"}
end

function investimenti.rendimento(pagina)
  local r = investimenti.rendimentoDati(pagina)
  if r.errore then return widget.markdown("**Rendimento n.d.**: " .. r.errore) end
  return widget.markdown("**XIRR**: " .. investimenti.formatoNumero(r.percentuale, 2) .. " % — " .. r.nota)
end
```

```space-lua
-- Dashboard: nessuna scrittura, lettura del motore patrimoniale corrente.
investimenti = investimenti or {}

-- Perimetro di pagina e tipo: restituisce la pagina e gli id ammessi (id -> true).
-- Pagina "Index" o qualsiasi pagina senza elenco proprio = tutti gli strumenti.
local function ambito(pagina, tipo, ctx)
  pagina = pagina or "Index"
  local ammessi = investimenti.perimetro(pagina, ctx)
  local out = {}
  if ammessi then
    for id in pairs(ammessi) do out[id] = true end
  else
    for _, s in ipairs(ctx.strumenti) do out[s.id] = true end
  end
  if tipo and tipo ~= "Tutti" then
    for id in pairs(out) do
      if not ctx.perId[id] or ctx.perId[id].tipo ~= tipo then out[id] = nil end
    end
  end
  return pagina, out
end

--- Valori del perimetro selezionato (solo posizioni aperte e valorizzate).
--- Il riepilogo considera le posizioni aperte alla data di consultazione.
--- Copertura: ufficiali (snapshot entro soglia), indicativi, obsoleti, mancanti.
function investimenti.dashboardDati(pagina, tipo, ctx)
  ctx = ctx or investimenti.contesto()
  local scope, ammessi = ambito(pagina, tipo, ctx)
  local rows, t = {}, {costo = 0, valore = 0, pl = 0, conteggio = 0, mancanti = 0,
    indicativi = 0, ufficiali = 0, obsoleti = 0, kidSomma = 0, kidPeso = 0}
  for _, r in ipairs(investimenti.valoreAllaData(scope, nil, ctx)) do
    if ammessi[r.id] and r.quote > 0.0000001 then
      t.conteggio = t.conteggio + 1
      if r.valore == nil then t.mancanti = t.mancanti + 1
      else
        local s = ctx.perId[r.id] or {}
        if r.copertura == "SNAPSHOT_UFFICIALE" then t.ufficiali = t.ufficiali + 1
        elseif r.copertura == "OBSOLETO" then t.obsoleti = t.obsoleti + 1
        elseif r.copertura ~= "AGGIORNATO" then t.indicativi = t.indicativi + 1 end
        local kid = s.rischio
        if kid and kid >= 1 and kid <= 7 and r.valore > 0 then
          t.kidSomma, t.kidPeso = t.kidSomma + kid * r.valore, t.kidPeso + r.valore
        end
        t.costo, t.valore, t.pl = t.costo + r.costo, t.valore + r.valore, t.pl + r.valore - r.costo
        table.insert(rows, {id = r.id, nome = s.nome or r.id, conto = r.conto, tipo = s.tipo,
          costo = r.costo, valore = r.valore, pl = r.valore - r.costo,
          dataPrezzo = r.dataPrezzo, stato = r.stato})
      end
    end
  end
  table.sort(rows, function(a, b) return a.nome < b.nome end)
  t.completo = t.mancanti == 0 and t.indicativi == 0 and t.obsoleti == 0
  t.kid = t.kidPeso > 0 and math.floor(t.kidSomma / t.kidPeso + 0.5) or nil
  return rows, t
end

--- Dashboard compatta: posizioni aperte valorizzate, costo residuo e P/L latente.
function investimenti.dashboard(pagina, tipo)
  local rows, tot = investimenti.dashboardDati(pagina, tipo)
  local function eur(n) return investimenti.formatoNumero(n, 2) end
  local function pct(v, base)
    if not base or base == 0 then return "n.d." end
    local z = 100 * v / base
    return (z > 0 and "+" or "") .. investimenti.formatoNumero(z, 2)
  end
  local ultima
  for _, r in ipairs(rows) do
    if type(r.dataPrezzo) == "string" and string.match(r.dataPrezzo, "^%d%d%d%d%-%d%d%-%d%d$") then
      if not ultima or r.dataPrezzo > ultima then ultima = r.dataPrezzo end
    end
  end
  local data = ultima and (string.sub(ultima, 9, 10) .. "." ..
    string.sub(ultima, 6, 7) .. "." .. string.sub(ultima, 1, 4)) or "n.d."
  local out = {
    "**Patrimonio:**",
    "| Investito | Attuale | **P/L latente** | % | In data | KID |",
    "|---:|---:|---:|---:|:---:|:---:|",
    "| " .. eur(tot.costo) .. " € | " .. eur(tot.valore) .. " € | **" ..
      eur(tot.pl) .. " €** | " .. pct(tot.pl, tot.costo) .. " % | " .. data ..
      " | " .. (tot.kid and tostring(tot.kid) or "n.d.") .. " |",
  }
  -- Con una sola posizione completa il totale coincide con l'elenco: copertura ed elenco sono omessi.
  if tot.conteggio ~= 1 or not tot.completo then
    table.insert(out, "")
    table.insert(out, "**Copertura:** " .. (tot.conteggio - tot.mancanti) .. "/" .. tot.conteggio ..
      " posizioni valorizzate; " .. tot.ufficiali .. " snapshot ufficiali, " ..
      tot.indicativi .. " indicative, " .. tot.obsoleti .. " obsolete, " .. tot.mancanti .. " mancanti" ..
      (tot.completo and "." or ". **Patrimonio parziale.**"))
  end
  if tot.conteggio ~= 1 then
    table.insert(out, "| Strumento | Costo residuo | Attuale | P/L latente | % |")
    table.insert(out, "|---|---:|---:|---:|---:|")
    for _, r in ipairs(rows) do
      table.insert(out, "| [[Strumenti/" .. r.id .. "|" .. r.nome .. "]] | " ..
        eur(r.costo) .. " € | " .. eur(r.valore) .. " € | " .. eur(r.pl) ..
        " € | " .. pct(r.pl, r.costo) .. " % |")
    end
    if #rows == 0 then table.insert(out, "| Nessuna posizione valorizzata | | | | |") end
  end
  return widget.markdown(table.concat(out, "\n"))
end

--- Segnalazioni del perimetro: posizioni non aggiornate (obsolete, indicative, mancanti, con
--- la causa) e anomalie contabili di verificaConti. Lista di {strumento, problema, dettaglio}.
function investimenti.diagnosticaDati(pagina, ctx)
  ctx = ctx or investimenti.contesto()
  local out = {}
  local scope, ids = ambito(pagina, nil, ctx)
  for _, r in ipairs(investimenti.valoreAllaData(scope, nil, ctx)) do
    if r.copertura ~= "AGGIORNATO" and r.copertura ~= "SNAPSHOT_UFFICIALE" and r.copertura ~= "CHIUSA" then
      table.insert(out, {strumento = r.id, problema = r.copertura,
        dettaglio = r.stato == "MOVIMENTI_INVALIDI" and table.concat(r.lotto.errori, "; ") or r.stato})
    end
  end
  for _, p in ipairs(investimenti.verificaConti(ctx)) do
    if ids[p.id] then
      table.insert(out, {strumento = p.id, problema = p.problema, dettaglio = p.dettaglio})
    end
  end
  return out
end

--- Resoconto diagnostico leggibile direttamente nelle pagine SilverBullet.
function investimenti.diagnosticaVista(pagina)
  local problemi = investimenti.diagnosticaDati(pagina)
  if #problemi == 0 then
    return widget.markdown("**Diagnostica:** nessuna anomalia rilevata dai controlli disponibili.")
  end
  local out = {"**Diagnostica:** " .. #problemi .. " segnalazioni",
    "| Strumento | Problema | Dettaglio |", "|---|---|---|"}
  for _, p in ipairs(problemi) do
    table.insert(out, "| " .. tostring(p.strumento or "") .. " | " ..
      tostring(p.problema or "") .. " | " .. tostring(p.dettaglio or "") .. " |")
  end
  return widget.markdown(table.concat(out, "\n"))
end

local function giorniMese(y, m)
  if m == 2 then return (y % 4 == 0 and (y % 100 ~= 0 or y % 400 == 0)) and 29 or 28 end
  return ({31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31})[m]
end

--- Valore di fine mese di ogni posizione del perimetro, dal primo movimento a oggi (l'ultimo punto
--- è oggi). Restituisce {date, righe = {{id, nome, kid, valori}}, mancanti}: valori[i] è nil se lo
--- strumento non è posseduto o non è valorizzabile nel mese i; mancanti[i] conta le posizioni
--- possedute senza valore. Un solo contesto per tutti i mesi; gli strumenti ceduti restano nei mesi
--- in cui avevano quote.
function investimenti.serieMensile(pagina, tipo, ctx)
  ctx = ctx or investimenti.contesto()
  local scope, ids = ambito(pagina, tipo, ctx)
  local fine = os.date("!%Y-%m-%d")
  local primo
  for _, r in ipairs(ctx.ledger) do
    if ids[r.id] and r.data and (not primo or r.data < primo) then primo = r.data end
  end
  local out = {date = {}, righe = {}, mancanti = {}}
  if not primo or primo > fine then return out end
  local y, m = tonumber(string.sub(primo, 1, 4)), tonumber(string.sub(primo, 6, 7))
  local ye, me = tonumber(string.sub(fine, 1, 4)), tonumber(string.sub(fine, 6, 7))
  local perId = {}
  while y < ye or (y == ye and m <= me) do
    local d = (y == ye and m == me) and fine or string.format("%04d-%02d-%02d", y, m, giorniMese(y, m))
    table.insert(out.date, d)
    local i, manca = #out.date, 0
    for _, r in ipairs((investimenti.valoreAllaData(scope, d, ctx))) do
      if ids[r.id] and r.quote and r.quote > 0.0000001 then
        local x = perId[r.id]
        if not x then
          local s = ctx.perId[r.id] or {}
          x = {id = r.id, nome = s.nome or r.id, kid = s.rischio, valori = {}}
          perId[r.id] = x
          table.insert(out.righe, x)
        end
        if r.valore ~= nil then x.valori[i] = (x.valori[i] or 0) + r.valore else manca = manca + 1 end
      end
    end
    out.mancanti[i] = manca
    m = m + 1
    if m > 12 then y, m = y + 1, 1 end
  end
  table.sort(out.righe, function(a, b) return a.nome < b.nome end)
  return out
end

-- Parte comune dei grafici con pulsanti di periodo (calcolati nel browser sulla serie completa).
local JS_PERIODI = [[
const dark = window.matchMedia && window.matchMedia("(prefers-color-scheme: dark)").matches;
const txt = dark ? "#ddd" : "#333", grid = "rgba(128,128,128,.25)";
const PAL = ["#2f6fdb", "#d9822b", "#2a9d4a", "#9b5de5", "#e4572e", "#1fa2b8", "#c2185b", "#7d8597", "#8d6e63", "#b5a300"];
function col(k) { return PAL[k % PAL.length]; }
const RANGE = [{n: "6M", m: 6}, {n: "YTD", m: -1}, {n: "1A", m: 12}, {n: "5A", m: 60}, {n: "Tutto", m: 0}];
const D = DATI.date, N = D.length;
function indici(r) {
  let da = 0;
  if (r.m === -1) {
    const inizio = D[N - 1].slice(0, 4) + "-01-01";
    da = D.findIndex(function (d) { return d >= inizio; });
    if (da < 0) da = 0;
  } else if (r.m > 0) da = Math.max(0, N - r.m);
  const ix = [];
  for (let i = da; i < N; i++) ix.push(i);
  return ix;
}
const bottoni = [];
function stile() {
  bottoni.forEach(function (b) {
    b.el.style.fontWeight = b.attivo() ? "bold" : "normal";
    b.el.style.background = b.attivo() ? "rgba(47,111,219,.25)" : "";
  });
}
function bottone(riga, nome, attivo, azione) {
  const el = document.createElement("button");
  el.textContent = nome;
  el.style.cssText = "margin:0 4px 6px 0;padding:2px 10px;cursor:pointer";
  el.onclick = function () { azione(); stile(); disegna(); };
  bottoni.push({el: el, attivo: attivo});
  riga.appendChild(el);
}
function assi(stacked) {
  return {x: {stacked: stacked, ticks: {color: txt, maxTicksLimit: 12}, grid: {color: grid}},
    y: {stacked: stacked, ticks: {color: txt}, grid: {color: grid}}};
}
]]

-- Torte (Oggi) e barre mensili impilate per strumento (periodi).
local JS_CONTO = [[
let periodo = null, grafici = [];
function torta(id, titolo, fette) {
  const tot = fette.reduce(function (a, f) { return a + f.valore; }, 0);
  grafici.push(new Chart(document.getElementById(id), {type: "pie",
    data: {labels: fette.map(function (f) { return f.nome + " " + (100 * f.valore / tot).toFixed(1) + " %"; }),
      datasets: [{data: fette.map(function (f) { return f.valore; }),
        backgroundColor: fette.map(function (f) { return col(f.k); })}]},
    options: {responsive: true, maintainAspectRatio: false, animation: false,
      plugins: {legend: {position: "bottom", labels: {color: txt, boxWidth: 12}},
        title: {display: true, text: titolo, color: txt}}}}));
}
function disegna() {
  grafici.forEach(function (g) { g.destroy(); });
  grafici = [];
  const oggi = periodo === null;
  document.getElementById("p").style.display = oggi ? "flex" : "none";
  document.getElementById("s").style.display = oggi ? "none" : "block";
  const nota = document.getElementById("n");
  if (oggi) {
    const u = N - 1, fv = [], fk = [];
    DATI.righe.forEach(function (r, k) {
      const v = r.v[u] || 0;
      if (v > 0) fv.push({nome: r.nome, valore: v, k: k});
      if (v > 0 && r.kid) fk.push({nome: r.nome, valore: v * r.kid, k: k});
    });
    torta("c1", "Valore per posizione", fv);
    torta("c2", "Rischio KID (KID × valore)", fk);
    nota.textContent = DATI.mancanti[u] ? DATI.mancanti[u] + " posizioni senza valore escluse." : "";
    return;
  }
  const ix = indici(periodo), ds = [];
  DATI.righe.forEach(function (r, k) {
    const dati = ix.map(function (i) { return r.v[i] || 0; });
    if (dati.some(function (v) { return v > 0; })) {
      ds.push({label: r.nome, data: dati, backgroundColor: col(k), stack: "s"});
    }
  });
  grafici.push(new Chart(document.getElementById("c3"), {type: "bar",
    data: {labels: ix.map(function (i) { return D[i].slice(0, 7) + (DATI.mancanti[i] ? "*" : ""); }), datasets: ds},
    options: {responsive: true, maintainAspectRatio: false, animation: false,
      interaction: {mode: "index", intersect: false},
      plugins: {legend: {labels: {color: txt, boxWidth: 12}}}, scales: assi(true)}}));
  nota.textContent = ix.some(function (i) { return DATI.mancanti[i]; })
    ? "* Mese con posizioni senza valore: il totale è parziale." : "";
}
const barra = document.getElementById("b");
bottone(barra, "Oggi", function () { return periodo === null; }, function () { periodo = null; });
RANGE.forEach(function (r) {
  bottone(barra, r.n, function () { return periodo === r; }, function () { periodo = r; });
});
stile();
disegna();
]]

-- Linea del valore della posizione a fine mese.
local JS_POSIZIONE = [[
let periodo = RANGE[2], g = null;
function disegna() {
  if (g) g.destroy();
  const ix = indici(periodo);
  g = new Chart(document.getElementById("c"), {type: "line",
    data: {labels: ix.map(function (i) { return D[i]; }),
      datasets: [{label: DATI.titolo, data: ix.map(function (i) { return DATI.v[i]; }),
        borderColor: "#2f6fdb", borderWidth: 1.5, pointRadius: 2, spanGaps: false, tension: 0}]},
    options: {responsive: true, maintainAspectRatio: false, animation: false,
      interaction: {mode: "nearest", axis: "x", intersect: false},
      plugins: {legend: {labels: {color: txt}}}, scales: assi(false)}});
}
const barra = document.getElementById("b");
RANGE.forEach(function (r) {
  bottone(barra, r.n, function () { return periodo === r; }, function () { periodo = r; });
});
stile();
disegna();
]]

local function listaJs(t, f)
  local a = {}
  for i, x in ipairs(t) do a[i] = f(x, i) end
  return "[" .. table.concat(a, ",") .. "]"
end

local function numeriJs(valori, n)
  local a = {}
  for i = 1, n do a[i] = valori[i] and string.format("%.2f", valori[i]) or "null" end
  return "[" .. table.concat(a, ",") .. "]"
end

--- Pagine Conti/, portafogli e Index: torte (valore e rischio KID) con il pulsante Oggi, barre
--- mensili impilate per strumento con 6M/YTD/1A/5A/Tutto. Il periodo è scelto nel browser.
function investimenti.graficiConto(pagina, tipo)
  pagina = pagina or editor.getCurrentPage()
  local s = investimenti.serieMensile(pagina, tipo)
  if #s.date < 2 or #s.righe == 0 then return widget.markdown("_Dati insufficienti per i grafici._") end
  local n = #s.date
  local dati = "const DATI = {date: " .. listaJs(s.date, investimenti.js) .. ", mancanti: " ..
    numeriJs(s.mancanti, n) .. ", righe: " .. listaJs(s.righe, function(r)
      return "{nome: " .. investimenti.js(r.nome) .. ", kid: " .. (r.kid and tostring(r.kid) or "null") ..
        ", v: " .. numeriJs(r.valori, n) .. "}"
    end) .. "};\n"
  return widget.sandbox {
    html = '<div id="b"></div><div id="p" style="display:flex;gap:12px"><div style="position:relative;' ..
      'height:320px;flex:1"><canvas id="c1"></canvas></div><div style="position:relative;height:320px;' ..
      'flex:1"><canvas id="c2"></canvas></div></div><div id="s" style="display:none;position:relative;' ..
      'height:320px"><canvas id="c3"></canvas></div><div id="n" style="font-size:.85em"></div>',
    script = 'loadJsByUrl("' .. investimenti.chartjs .. '").then(function () {\n' .. dati ..
      JS_PERIODI .. JS_CONTO .. "\n});",
    markdown = "Grafici patrimonio: " .. pagina,
  }
end

--- Pagine Strumenti/: valore della posizione a fine mese, con pulsanti 6M/YTD/1A/5A/Tutto.
function investimenti.graficoPosizione(pagina)
  pagina = pagina or editor.getCurrentPage()
  local s = investimenti.serieMensile(pagina)
  if #s.date < 2 or #s.righe == 0 then return widget.markdown("_Dati insufficienti per il grafico._") end
  local n, v = #s.date, {}
  for i = 1, n do
    for _, r in ipairs(s.righe) do
      if r.valori[i] then v[i] = (v[i] or 0) + r.valori[i] end
    end
  end
  local titolo = s.righe[1].nome
  local dati = "const DATI = {titolo: " .. investimenti.js("Posizione " .. titolo .. " (EUR, fine mese)") ..
    ", date: " .. listaJs(s.date, investimenti.js) .. ", v: " .. numeriJs(v, n) .. "};\n"
  return widget.sandbox {
    html = '<div id="b"></div><div style="position:relative;height:300px"><canvas id="c"></canvas></div>',
    script = 'loadJsByUrl("' .. investimenti.chartjs .. '").then(function () {\n' .. dati ..
      JS_PERIODI .. JS_POSIZIONE .. "\n});",
    markdown = "Grafico posizione " .. titolo,
  }
end

--- Widget automatici, solo se configurati (investimenti.widgetAutomatici = true): elenco di funzioni
--- da mostrare in alto nella pagina (hooks:renderTopWidgets).
function investimenti.vista(pagina)
  pagina = pagina or editor.getCurrentPage()
  if type(pagina) ~= "string" then return {} end
  local function v(f, p) return function() return f(p) end end
  if pagina == (config.get("investimenti.paginaIndice") or "Index") or string.startsWith(pagina, "Conti/") then
    return {v(investimenti.dashboard, pagina), v(investimenti.diagnosticaVista, pagina)}
  elseif string.startsWith(pagina, "Strumenti/") then
    return {v(investimenti.scheda, pagina), v(investimenti.risultato, pagina), v(investimenti.grafico, pagina)}
  elseif string.startsWith(pagina, "Quotazioni/") then
    return {v(investimenti.grafico, pagina)}
  end
  return {}
end

for i = 1, 3 do
  event.listen {
    name = "hooks:renderTopWidgets",
    run = function(e)
      if config.get("investimenti.widgetAutomatici") ~= true then return end
      local f = investimenti.vista()[i]
      if f then return f() end
    end
  }
end
```
