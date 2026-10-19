---
name: "Library/MG/investimenti"
tags: meta/library
version: "0.22"
versionDate: 2026-10-19
pageDecoration.prefix: "💸 "
share.uri: "github:marco10x15/silverbullet-libraries/investimenti.md"
share.hash: a5553b26
share.mode: pull
---

# Investimenti

**Versione 0.22 — Dashboard patrimoniale, snapshot e indicatori essenziali, SilverBullet v2.11+.** Libreria completa, da sostituire integralmente a `Library/MG/investimenti`. Documentazione: pagine `manuale/`.

La libreria legge strumenti, conti, quotazioni e movimenti direttamente dallo Space. Non gestisce liquidità né spese del conto corrente. Gli snapshot sono righe `Snapshot` nella tabella `Movimenti`. La dashboard mostra costo residuo, controvalore e P/L latente; il grafico patrimoniale usa le quantità storiche e i rendiconti disponibili.

**Modifiche 0.22:** strumenti valorizzati a rendiconto (metodo `snapshot`/`manuale`): `rendimentoDati()` calcola lo XIRR alla data dell'ultimo rendiconto (prima restituiva `VALORE_INDICATIVO`); `grafico()` su `Strumenti/`, se mancano le quotazioni, mostra il nuovo `graficoSnapshot()` (valore dei rendiconti e versato netto). Comportamento invariato per gli strumenti con quotazioni.

**Migrazione dalla 0.21:** nessuna modifica ai dati né alle chiamate esistenti.

**Modifiche 0.21:** contesto unico (`investimenti.contesto()`): strumenti, conti, movimenti e quotazioni sono letti una sola volta per widget e riusati da tutte le funzioni, anche per i punti di `graficoPatrimonio`; `investimenti.numero()` accetta la virgola decimale (`14942,92`); XIRR corretto per rendimento 0 %; soglia di età applicata anche agli snapshot; il ledger scarta le tabelle senza colonna `Tipo`; nuove diagnostiche `SNAPSHOT_NON_VALIDO`, `REDDITO_NON_VALIDO`, `DATA_NON_VALIDA`; `verifica()` ammette più strumenti con lo stesso isin (NAV condiviso); valute diverse da EUR/USD danno `CAMBIO_MANCANTE`; rimossi `metadati`, `verificaPatrimonio`, `contiStrumento` e il codice non usato. Il metodo di valorizzazione dichiarato è ora un campo di `strumenti()`.

**Migrazione dalla 0.20:** nessuna modifica ai dati. Le funzioni accettano un ultimo argomento facoltativo `ctx` (risultato di `investimenti.contesto()`); le chiamate esistenti restano valide. `investimenti.metodo(s)` non riceve più i metadati.

## Esempi

```lua
investimenti.dashboard("Index")
investimenti.dashboard("Conti/NEF - Famiglia", "Fondo")
investimenti.scheda("Strumenti/LU0140696401")
investimenti.grafico("Strumenti/LU0140696401")
investimenti.graficoPatrimonio("Conti/NEF - Famiglia", "YTD")
investimenti.rendimentoDati("Strumenti/LU0140696401", "xirr")
investimenti.rendimento("Strumenti/<id>")
investimenti.graficoSnapshot("Strumenti/<id>")
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
--- (metodoValorizzazione, frequenzaPrezzoGiorni, quoteReali), letti nella stessa scansione.
function investimenti.strumenti()
  local out = {}
  for _, p in ipairs(query[[
    from p = index.subPages("Strumenti")
    select {name = p.name, nome = p.displayName, isin = p.isin, ticker = p.ticker,
      valuta = p.valuta, tipo = p.tipo, rischio = p.KID,
      descrizione = p.description, tags = p.tags,
      metodo = p.metodoValorizzazione, frequenza = p.frequenzaPrezzoGiorni,
      quoteReali = p.quoteReali}
  ]]) do
    table.insert(out, {id = string.sub(p.name, 11),  -- "Strumenti/" = 10 caratteri
      nome = str(p.nome) or p.name, isin = str(p.isin), ticker = str(p.ticker),
      valuta = str(p.valuta) or "EUR", tipo = str(p.tipo),
      rischio = type(p.rischio) == "number" and p.rischio or nil,  -- p.rischio = attributo KID
      descrizione = str(p.descrizione), tags = p.tags,
      metodoDichiarato = str(p.metodo), frequenza = investimenti.numero(p.frequenza),
      quoteReali = p.quoteReali == true})
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

-- La scheda usa solo metadati e relazioni già presenti (nessun calcolo patrimoniale).
function investimenti.scheda(pagina)
  pagina = pagina or editor.getCurrentPage()
  if type(pagina) ~= "string" or not string.startsWith(pagina, "Strumenti/") then return nil end
  local id = string.sub(pagina, 11)
  local s
  for _, x in ipairs(investimenti.strumenti()) do
    if x.id == id then s = x; break end
  end
  if not s then return widget.markdown("_Strumento non trovato._") end

  -- Emoji Unicode: nessun tag HTML/CSS, per il rendering Markdown di SilverBullet.
  -- Il livello numerico KID resta sempre visibile; 2 e 3 condividono il verde.
  local colori = {"⚪", "🟢", "🟢", "🟠", "🟤", "🔴", "🟣"}
  local kid = s.rischio
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
  -- Due righe compatte: nessun paragrafo vuoto; link wiki attivi anche nella seconda riga.
  local riga1 = "**" .. (s.tipo or "Strumento") .. " · " .. (s.valuta or "EUR") ..
    " · " .. rischio .. "**"
  local riga2 = "**Conti:** " .. links(conti.fisici[id]) .. " · **Portafogli:** " .. links(conti.portafogli[id])
  return widget.markdown(riga1 .. "  \n" .. riga2)
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
quotazioni = quotazioni or {}

--- Anomalie delle pagine Quotazioni e del loro collegamento agli strumenti (isin).
--- Più strumenti con lo stesso isin sono ammessi (NAV condiviso). La tabella Data | Prezzo
--- è obbligatoria solo per le pagine collegate a uno strumento con ticker (aggiornabile da Yahoo).
function investimenti.verifica()
  local anomalie = {}
  local function avvisa(pagina, problema, dettaglio)
    table.insert(anomalie, {pagina = pagina, problema = problema, dettaglio = dettaglio})
  end
  local perIsin, perPagina, duplicati, senzaIsin = investimenti.pagineQuotazioni()
  local conTicker = {}  -- isin -> true se almeno uno strumento collegato ha un ticker
  local strumenti = investimenti.strumenti()
  for _, s in ipairs(strumenti) do
    if s.isin and s.ticker then conTicker[s.isin] = true end
  end
  for pagina, isin in pairs(perPagina) do
    if duplicati[isin] then avvisa(pagina, "ISIN_DUPLICATO", isin) end
    if conTicker[isin] then
      local _, errore = quotazioni.blocco(space.readPage(pagina))
      if errore then avvisa(pagina, "TABELLA_NON_VALIDA", errore) end
    end
  end
  for _, nome in ipairs(senzaIsin) do avvisa(nome, "ISIN_MANCANTE", "Frontmatter senza isin") end
  for _, s in ipairs(strumenti) do
    if s.ticker and (not s.isin or not perIsin[s.isin]) then
      avvisa("Strumenti/" .. s.id, "QUOTAZIONI_MANCANTI", "Ticker senza pagina quotazioni collegata")
    end
  end
  table.sort(anomalie, function(a, b)
    if a.pagina ~= b.pagina then return a.pagina < b.pagina end
    return a.problema < b.problema
  end)
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
--- aggregati[conto] = true per i portafogli logici (tag Portafoglio), noti[conto] = true,
--- fisici[id] / portafogli[id] = conti che contengono lo strumento.
function investimenti.conti()
  local c = {ids = {}, aggregati = {}, noti = {}, fisici = {}, portafogli = {}}
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
    c.noti[p.name], c.ids[p.name], c.aggregati[p.name] = true, ids, aggregato
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

-- "acquisto", "ACQUISTO" -> "Acquisto": il tipo non dipende da maiuscole e minuscole.
local function canonico(t) return string.upper(string.sub(t, 1, 1)) .. string.lower(string.sub(t, 2)) end

--- Contesto: tutto ciò che serve ai calcoli, letto UNA volta (2 scansioni di pagine, 1 di tabelle)
--- e passato come ultimo argomento facoltativo alle funzioni. Campi:
---   strumenti (lista ordinata) e perId (id -> strumento, con metodoDichiarato/frequenza/quoteReali);
---   conti (vedi investimenti.conti);
---   serie: id -> lista {data, prezzo} ordinata, dalla pagina Quotazioni con lo stesso isin
---          (pagine con isin duplicato escluse);
---   ledger: solo le righe di movimento (tabelle di Strumenti/ con colonna Tipo), con il conto
---          risolto, i valori numerici (virgola decimale ammessa) e la data (nil se non nel
---          formato AAAA-MM-GG), in ordine cronologico.
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
      valore = t.valore_eur or t.valore, conto = t.conto, px = t.prezzo,
      causale = t.causale, riferimento = t.riferimento}
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
      table.insert(c.ledger, {id = id,
        conto = investimenti.risolviConto(id, r.conto, c.conti.fisici, c.conti.portafogli),
        contoEsplicito = str(r.conto), tipo = canonico(r.tipo), dataGrezza = tostring(r.giorno),
        data = type(r.giorno) == "string" and string.match(r.giorno, "^%d%d%d%d%-%d%d%-%d%d$") and r.giorno or nil,
        quote = investimenti.numero(r.quote), valore = investimenti.numero(r.valore),
        causale = str(r.causale), riferimento = str(r.riferimento), ref = r.ref})
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

--- Anomalie contabili: strumenti/conti inesistenti, conto non assegnato, date, importi e quantità
--- non validi nei movimenti (acquisti, vendite, redditi, snapshot). ctx facoltativo.
function investimenti.verificaConti(ctx)
  ctx = ctx or investimenti.contesto()
  local out, c = {}, ctx.conti
  local function errore(id, tipo, dettaglio)
    table.insert(out, {pagina = "Strumenti/" .. id, id = id, problema = tipo, dettaglio = dettaglio})
  end
  for _, lista in ipairs({c.fisici, c.portafogli}) do
    for id, pp in pairs(lista) do
      if not ctx.perId[id] then errore(id, "STRUMENTO_INESISTENTE", table.concat(pp, ", ")) end
    end
  end
  for _, r in ipairs(ctx.ledger) do
    local giorno, tipo = r.dataGrezza, r.tipo
    if r.contoEsplicito and not c.noti[r.conto] then errore(r.id, "CONTO_INESISTENTE", r.conto) end
    if not r.conto then errore(r.id, "CONTO_NON_ASSEGNATO", giorno) end
    if not r.data then
      errore(r.id, "DATA_NON_VALIDA", giorno)
    elseif (tipo == "Acquisto" or tipo == "Vendita") and (not r.quote or not r.valore) then
      errore(r.id, "MOVIMENTO_NON_VALIDO", giorno)
    elseif tipo == "Snapshot" and (not r.valore or r.valore < 0) then
      errore(r.id, "SNAPSHOT_NON_VALIDO", giorno)
    elseif (tipo == "Dividendo" or tipo == "Cedola") and (not r.valore or r.valore < 0) then
      errore(r.id, "REDDITO_NON_VALIDO", giorno)
    end
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
  local id, quot, conAcquisti, nome, strumento
  if string.startsWith(pagina, "Quotazioni/") then
    quot = pagina
    for _, s in ipairs(investimenti.strumenti()) do
      if s.isin and s.isin == perPagina[pagina] then id, nome = s.id, s.nome end
    end
  elseif string.startsWith(pagina, "Strumenti/") then
    id, conAcquisti = string.sub(pagina, 11), true
    for _, s in ipairs(investimenti.strumenti()) do
      if s.id == id then
        nome, strumento = s.nome, s
        quot = s.isin and not duplicati[s.isin] and perIsin[s.isin] or nil
      end
    end
  else return widget.markdownBlock("_Il grafico vale per pagine Quotazioni/ e Strumenti/._") end

  -- Senza quotazioni, uno strumento a rendiconto usa il grafico dei rendiconti.
  local function senzaQuotazioni(messaggio)
    local m = strumento and investimenti.metodo(strumento)
    if m == "snapshot" or m == "manuale" then return investimenti.graficoSnapshot(pagina) end
    return widget.markdownBlock(messaggio)
  end
  if not quot then
    return senzaQuotazioni("_Nessuna pagina Quotazioni univoca con l'isin dello strumento._")
  end

  local serie = investimenti.serie(quot)
  if #serie < 2 then return senzaQuotazioni("_Quotazioni insufficienti per il grafico._") end
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

--- Metodo di valorizzazione: quello dichiarato nello strumento (attributo metodoValorizzazione)
--- se valido, altrimenti dedotto dal tipo.
function investimenti.metodo(s)
  if type(s) ~= "table" then return "manuale" end
  local ammessi = {mercato = true, nav = true, obbligazione = true,
    contrattuale = true, snapshot = true, manuale = true, nominale = true, costo = true}
  if s.metodoDichiarato and ammessi[s.metodoDichiarato] then return s.metodoDichiarato end
  if s.tipo == "PIR" then return "snapshot" end
  if s.tipo == "Fondo" then return "nav" end
  if s.tipo == "Obbligazione" then
    if string.startsWith(s.id, "CD-") then return "contrattuale" end
    return "obbligazione"
  end
  return "mercato"  -- Cambio, Cripto e tutto il resto
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

--- Ultimo prezzo di ogni strumento alla data, convertito in EUR: id -> {data, prezzo, eur, dataCambio}.
--- Valute diverse da EUR e USD non sono convertibili: eur = nil (stato CAMBIO_MANCANTE).
function investimenti.prezzi(data, ctx)
  data = data or oggi()
  ctx = ctx or investimenti.contesto()
  local px = {}
  for _, s in ipairs(ctx.strumenti) do
    local r = prezzoPrima(ctx.serie[s.id], data)
    if r then
      local eur, dtCambio = r[2], nil
      if s.valuta == "USD" and s.id ~= "EURUSD" then
        local cambio = prezzoPrima(ctx.serie.EURUSD, r[1])
        if cambio then eur, dtCambio = eur / cambio[2], cambio[1] else eur = nil end
      elseif s.valuta ~= "EUR" and s.valuta ~= "USD" then eur = nil end
      px[s.id] = {data = r[1], prezzo = r[2], eur = eur, dataCambio = dtCambio}
    end
  end
  return px
end

-- Posizioni e lotti PMC, calcolati cronologicamente per conto + strumento.
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
      if not per[key] then
        per[key] = {id = r.id, conto = r.conto, quote = 0, costo = 0,
          realizzato = 0, redditi = 0, acquisti = 0, vendite = 0,
          errori = {}, flussi = {}, primo = nil}
        table.insert(chiavi, key)
      end
      local v, q, p = r.valore, r.quote, per[key]
      if r.tipo == "Acquisto" then
        if not q or q <= 0 or not v or v <= 0 then
          table.insert(p.errori, "ACQUISTO_NON_VALIDO " .. r.data)
        else
          p.quote, p.costo, p.acquisti = p.quote + q, p.costo + v, p.acquisti + v
          if not p.primo then p.primo = r.data end
          table.insert(p.flussi, {data = r.data, valore = -v})
        end
      elseif r.tipo == "Vendita" then
        if not q or q <= 0 or not v or v < 0 or q > p.quote + 1e-7 then
          table.insert(p.errori, "VENDITA_NON_VALIDA " .. r.data)
        else
          local costoVenduto = p.quote > 0 and p.costo * q / p.quote or 0
          p.realizzato = p.realizzato + v - costoVenduto
          p.quote, p.costo, p.vendite = p.quote - q, p.costo - costoVenduto, p.vendite + v
          if math.abs(p.quote) < 1e-7 then p.quote, p.costo = 0, 0 end
          table.insert(p.flussi, {data = r.data, valore = v})
        end
      elseif r.tipo == "Dividendo" or r.tipo == "Cedola" then
        if v and v >= 0 then
          p.redditi = p.redditi + v
          table.insert(p.flussi, {data = r.data, valore = v})
        else table.insert(p.errori, "REDDITO_NON_VALIDO " .. r.data) end
      end
    end
  end
  table.sort(chiavi)
  local out = {}
  for _, k in ipairs(chiavi) do table.insert(out, per[k]) end
  return out
end

-- Snapshot informativi: un controvalore in EUR per posizione e data (chiave "conto|id").
-- Non contribuiscono ai lotti o ai flussi finanziari. Il conto viene risolto
-- dal ledger esistente, così gli snapshot rimangono nella stessa posizione.
-- Le serie sono in ordine cronologico (il ledger è già ordinato).
function investimenti.snapshotDati(pagina, data, ctx)
  data = data or oggi()
  ctx = ctx or investimenti.contesto()
  local ammessi, conto, aggregato = investimenti.perimetro(pagina, ctx)
  local out = {}
  for _, r in ipairs(ctx.ledger) do
    if r.tipo == "Snapshot" and (not ammessi or ammessi[r.id]) and r.valore and r.valore >= 0 and
      r.data and r.data <= data and (not conto or r.conto == conto or aggregato) then
      local key = (r.conto or "NON_ASSEGNATO") .. "|" .. r.id
      out[key] = out[key] or {}
      table.insert(out[key], {data = r.data, valore = r.valore})
    end
  end
  return out
end

-- Apporti e rimborsi posteriori all'ultimo rendiconto: una PROXY
-- contabile, non una quotazione o una variazione certificata di mercato.
-- ledger: ctx.ledger.
function investimenti.movimentiDopoSnapshot(id, conto, dataSnapshot, dataFine, ledger)
  local netto = 0
  for _, r in ipairs(ledger) do
    if r.id == id and (r.conto or "NON_ASSEGNATO") == (conto or "NON_ASSEGNATO") and
      r.data and r.data > dataSnapshot and r.data <= dataFine and r.valore then
      if r.tipo == "Acquisto" then netto = netto + r.valore
      elseif r.tipo == "Vendita" then netto = netto - r.valore end
    end
  end
  return netto
end

-- Valorizzazione datata: NAV e mercato su quotazioni; contrattuale/nominale su
-- capitale effettivo; snapshot/manuale solo con valore di posizione esplicito.
-- I PIR dell'esempio: l'unità sintetica di 1 EUR NON è un numero di quote NAV.
-- Ogni riga porta con sé il lotto (campo lotto) da cui deriva.
function investimenti.valoreAllaData(pagina, data, ctx)
  data = data or oggi()
  ctx = ctx or investimenti.contesto()
  local prezzi = investimenti.prezzi(data, ctx)
  local snapshots = investimenti.snapshotDati(pagina, data, ctx)
  local righe, totale = {}, {valore = 0, note = 0, mancanti = 0, stimati = 0, posizioni = 0}
  for _, p in ipairs(investimenti.lotti(pagina, data, ctx)) do
    local s = ctx.perId[p.id] or {id = p.id, tipo = ""}
    local metodo = investimenti.metodo(s)
    local px = prezzi[p.id]
    local valore, stato, dt, fonte
    if p.quote == 0 then valore, stato = 0, "CHIUSA"
    elseif #p.errori > 0 then stato = "MOVIMENTI_INVALIDI"
    elseif (metodo == "mercato" or metodo == "nav" or metodo == "obbligazione") then
      if metodo == "nav" and s.tipo == "PIR" and not s.quoteReali then
        stato = "QUOTE_NAV_NON_VERIFICATE"
      elseif px and px.eur then
        valore, dt, fonte = p.quote * px.eur, px.data, "quotazione"
        stato = "PREZZO_DISPONIBILE"
      else stato = px and "CAMBIO_MANCANTE" or "PREZZO_MANCANTE" end
    elseif metodo == "snapshot" or metodo == "manuale" then
      local serie = snapshots[(p.conto or "NON_ASSEGNATO") .. "|" .. p.id] or {}
      local ultimo = serie[#serie]
      if ultimo then
        local flussi = investimenti.movimentiDopoSnapshot(p.id, p.conto, ultimo.data, data, ctx.ledger)
        valore, dt = ultimo.valore + flussi, ultimo.data
        if math.abs(flussi) > 0.000001 then
          stato, fonte = "SNAPSHOT_CON_APPORTI", "snapshot_piu_movimenti_indicativo"
        else
          stato, fonte = "SNAPSHOT_DISPONIBILE", "snapshot_movimenti"
        end
      elseif px and px.eur and s.quoteReali then
        valore, dt, fonte = p.quote * px.eur, px.data, "nav_confermato"
        stato = "PREZZO_DISPONIBILE"
      else stato = "SNAPSHOT_MANCANTE" end
    elseif metodo == "nominale" or metodo == "contrattuale" then
      -- Se esiste una quotazione manuale datata, mostra il nominale comunicato;
      -- il prezzo resta indicativo, quindi non abilita XIRR come dato realizzato.
      if px and px.eur then
        valore, dt, stato, fonte = p.quote * px.eur, px.data,
          "PREZZO_MANUALE_CONTRATTUALE", "quotazione_manualizzata"
      else
        valore, stato, fonte = p.costo, "COSTO_PROXY_CONTRATTUALE", "costo_residuo"
      end
    elseif metodo == "costo" then
      valore, stato, fonte = p.costo, "COSTO_NON_MERCATO", "costo_residuo"
    end
    if valore ~= nil then
      totale.valore = totale.valore + valore
      totale.note = totale.note + 1
      if stato ~= "PREZZO_DISPONIBILE" and stato ~= "SNAPSHOT_DISPONIBILE" and stato ~= "CHIUSA" then
        totale.stimati = totale.stimati + 1
      end
    else totale.mancanti = totale.mancanti + 1 end
    totale.posizioni = totale.posizioni + 1
    table.insert(righe, {id = p.id, conto = p.conto, metodo = metodo, quote = p.quote,
      valore = valore, stato = stato, fonte = fonte, dataPrezzo = dt,
      prezzo = px and px.prezzo, dataCambio = px and px.dataCambio, costo = p.costo, lotto = p})
  end
  return righe, totale
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

function investimenti.rendimentoDati(pagina, metodo)
  pagina = pagina or editor.getCurrentPage()
  metodo = metodo or "xirr"
  if metodo ~= "xirr" and metodo ~= "twr" then return {errore = "METODO_SCONOSCIUTO"} end
  local id = type(pagina) == "string" and string.startsWith(pagina, "Strumenti/") and string.sub(pagina, 11)
  if not id then return {errore = "PERIMETRO_NON_RICONCILIATO", dettaglio = "Richiesto singolo strumento: mancano liquidita e flussi esterni del conto"} end
  local data, ctx = oggi(), investimenti.contesto()
  local valori = investimenti.valoreAllaData(pagina, data, ctx)
  if #valori ~= 1 then return {errore = "POSIZIONI_MULTIPLE", dettaglio = "Serve una posizione univoca per conto"} end
  local v = valori[1]
  -- Strumento a rendiconto: il valore è certificato solo alla data dell'ultimo snapshot,
  -- quindi il rendimento si calcola a quella data (flussi successivi esclusi).
  if (v.stato == "SNAPSHOT_DISPONIBILE" or v.stato == "SNAPSHOT_CON_APPORTI") and
    v.dataPrezzo and v.dataPrezzo < data then
    data = v.dataPrezzo
    valori = investimenti.valoreAllaData(pagina, data, ctx)
    if #valori ~= 1 then return {errore = "POSIZIONI_MULTIPLE", dettaglio = "Serve una posizione univoca per conto"} end
    v = valori[1]
  end
  local p = v.lotto
  if #p.errori > 0 then return {errore = "MOVIMENTI_INVALIDI"} end
  if v.valore == nil then return {errore = v.stato or "VALORE_MANCANTE"} end
  if v.stato ~= "PREZZO_DISPONIBILE" and v.stato ~= "SNAPSHOT_DISPONIBILE" and v.stato ~= "CHIUSA" then
    return {errore = "VALORE_INDICATIVO", dettaglio = v.stato}
  end
  if metodo == "twr" then
    -- Evita di dichiarare esatto un TWR con versamenti intermedi o dividendi.
    local acquisti, altri = 0, 0
    for _, f in ipairs(p.flussi) do
      if f.valore < 0 then acquisti = acquisti + 1 else altri = altri + 1 end
    end
    if acquisti ~= 1 or altri > 0 then return {errore = "SOTTOPERIODI_MANCANTI"} end
    if p.acquisti <= 0 then return {errore = "CAPITALE_MANCANTE"} end
    return {metodo = metodo, percentuale = (v.valore / p.acquisti - 1) * 100,
      nota = "Rendimento dal solo acquisto iniziale, non annualizzato"}
  end
  local flussi = {}
  for _, f in ipairs(p.flussi) do table.insert(flussi, {data = f.data, valore = f.valore}) end
  if p.quote ~= 0 then table.insert(flussi, {data = data, valore = v.valore}) end
  table.sort(flussi, function(a, b) return a.data < b.data end)
  local rendimento, errore = xirr(flussi)
  local nota = "XIRR annualizzato; flussi della singola posizione"
  if data ~= oggi() then nota = nota .. "; al rendiconto del " .. data end
  return {metodo = metodo, percentuale = rendimento, errore = errore, data = data, nota = nota}
end

function investimenti.rendimento(pagina, metodo)
  local r = investimenti.rendimentoDati(pagina, metodo)
  if r.errore then return widget.markdown("**Rendimento n.d.**: " .. r.errore) end
  return widget.markdown("**" .. string.upper(r.metodo) .. "**: " ..
    investimenti.formatoNumero(r.percentuale, 2) .. " %" .. " — " .. r.nota)
end

--- Valorizzazione con stato di copertura per riga: AGGIORNATO / SNAPSHOT_UFFICIALE (prezzo o
--- snapshot entro la soglia di età), OBSOLETO (oltre la soglia), INDICATIVO, MANCANTE.
--- Soglia in giorni: frequenzaPrezzoGiorni dello strumento, altrimenti 365 per snapshot e
--- contrattuali, 7 per gli altri.
function investimenti.coperturaDati(pagina, data, ctx)
  data = data or oggi()
  ctx = ctx or investimenti.contesto()
  local righe, tot = investimenti.valoreAllaData(pagina, data, ctx)
  local report = {aggiornati = 0, obsoleti = 0, mancanti = tot.mancanti,
    stimati = tot.stimati, totalePosizioni = tot.posizioni, valoreParziale = tot.valore}
  for _, r in ipairs(righe) do
    local s = ctx.perId[r.id] or {}
    local limite = s.frequenza or ((r.metodo == "snapshot" or r.metodo == "contrattuale") and 365 or 7)
    local gg = r.dataPrezzo and giorniTraDate(r.dataPrezzo, data)
    r.etaGiorni, r.sogliaGiorni = gg, limite
    if r.valore == nil then r.copertura = "MANCANTE"
    elseif r.stato == "SNAPSHOT_DISPONIBILE" or r.stato == "PREZZO_DISPONIBILE" then
      if gg and gg >= 0 and gg <= limite then
        r.copertura = r.stato == "PREZZO_DISPONIBILE" and "AGGIORNATO" or "SNAPSHOT_UFFICIALE"
        report.aggiornati = report.aggiornati + 1
      else r.copertura = "OBSOLETO"; report.obsoleti = report.obsoleti + 1 end
    else r.copertura = "INDICATIVO" end
  end
  report.completo = report.mancanti == 0 and report.obsoleti == 0 and report.stimati == 0
  return righe, report
end

function investimenti.copertura(pagina)
  local _, r = investimenti.coperturaDati(pagina)
  return widget.markdown("**Copertura patrimonio**: " .. r.totalePosizioni ..
    " posizioni; " .. r.aggiornati .. " aggiornate, " ..
    r.obsoleti .. " obsolete, " .. r.stimati .. " indicative, " ..
    r.mancanti .. " mancanti. **Valore parziale:** " ..
    investimenti.formatoNumero(r.valoreParziale, 2) .. " €" ..
    (r.completo and "" or " (NON rappresenta il patrimonio completo)"))
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

local function inizioPeriodo(periodo, fine)
  if periodo == nil or periodo == "Tutto" then return nil end
  if periodo == "YTD" then return string.sub(fine, 1, 4) .. "-01-01" end
  local giorni = ({["1M"] = 30, ["3M"] = 90, ["6M"] = 180, ["1A"] = 365})[periodo]
  if not giorni then return nil end
  local y,m,d = string.match(fine, "^(%d%d%d%d)%-(%d%d)%-(%d%d)$")
  if not y then return nil end
  return os.date("!%Y-%m-%d", os.time({year=tonumber(y), month=tonumber(m), day=tonumber(d), hour=12}) - giorni*86400)
end

--- Valori del perimetro selezionato (solo posizioni aperte e valorizzate).
--- Il riepilogo considera le posizioni aperte alla data di consultazione.
--- Copertura per posizione: ufficiali (snapshot entro soglia), indicativi, obsoleti, mancanti.
function investimenti.dashboardDati(pagina, tipo, ctx)
  ctx = ctx or investimenti.contesto()
  local scope, ammessi = ambito(pagina, tipo, ctx)
  local rows, t = {}, {costo = 0, valore = 0, pl = 0, conteggio = 0, mancanti = 0,
    indicativi = 0, ufficiali = 0, obsoleti = 0, kidSomma = 0, kidPeso = 0}
  for _, r in ipairs((investimenti.coperturaDati(scope, nil, ctx))) do
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
    "",
    "**Copertura:** " .. (tot.conteggio - tot.mancanti) .. "/" .. tot.conteggio ..
      " posizioni valorizzate; " .. tot.ufficiali .. " snapshot ufficiali, " ..
      tot.indicativi .. " indicative, " .. tot.obsoleti .. " obsolete, " .. tot.mancanti .. " mancanti" ..
      (tot.completo and "." or ". **Patrimonio parziale.**"),
    "| Strumento | Costo residuo | Attuale | P/L latente | % |",
    "|---|---:|---:|---:|---:|"
  }
  for _, r in ipairs(rows) do
    table.insert(out, "| [[Strumenti/" .. r.id .. "|" .. r.nome .. "]] | " ..
      eur(r.costo) .. " € | " .. eur(r.valore) .. " € | " .. eur(r.pl) ..
      " € | " .. pct(r.pl, r.costo) .. " % |")
  end
  if #rows == 0 then table.insert(out, "| Nessuna posizione valorizzata | | | | |") end
  return widget.markdown(table.concat(out, "\n"))
end

--- Metadati switch FACOLTATIVI: nessun abbinamento automatico dalla sola data.
function investimenti.switch(pagina, ctx)
  ctx = ctx or investimenti.contesto()
  local _, ids = ambito(pagina, nil, ctx)
  local out, gruppi = {}, {}
  for _, r in ipairs(ctx.ledger) do
    if ids[r.id] and r.causale and string.lower(r.causale) == "switch" then
      local x = {id = r.id, data = r.data, tipo = r.tipo, riferimento = r.riferimento, valore = r.valore}
      table.insert(out, x)
      if r.riferimento and r.riferimento ~= "" then
        gruppi[r.riferimento] = gruppi[r.riferimento] or {}
        table.insert(gruppi[r.riferimento], x)
      end
    end
  end
  local problemi = {}
  for _, r in ipairs(out) do
    if not r.riferimento or r.riferimento == "" then
      table.insert(problemi,{problema="SWITCH_SENZA_RIFERIMENTO",id=r.id,data=r.data})
    end
  end
  for ref, righe in pairs(gruppi) do
    local nA,nV = 0,0
    for _, r in ipairs(righe) do
      if r.tipo == "Acquisto" then nA=nA+1 elseif r.tipo == "Vendita" then nV=nV+1 end
    end
    if nA == 0 or nV == 0 then
      table.insert(problemi,{problema="SWITCH_NON_ABBINATO",riferimento=ref,
        dettaglio="acquisti=" .. nA .. ", vendite=" .. nV})
    end
  end
  return out, problemi
end

--- Segnalazioni del perimetro: copertura (obsoleti, indicativi, mancanti), anomalie dei movimenti
--- (verificaConti) e switch incompleti. Lista di {strumento, problema, dettaglio}.
function investimenti.diagnosticaDati(pagina, ctx)
  ctx = ctx or investimenti.contesto()
  local out = {}
  local scope, ids = ambito(pagina, nil, ctx)
  for _, r in ipairs((investimenti.coperturaDati(scope, nil, ctx))) do
    if r.copertura ~= "AGGIORNATO" and r.copertura ~= "SNAPSHOT_UFFICIALE" and r.stato ~= "CHIUSA" then
      table.insert(out, {strumento = r.id, problema = r.copertura, dettaglio = r.stato})
    end
  end
  for _, p in ipairs(investimenti.verificaConti(ctx)) do
    if ids[p.id] then
      table.insert(out, {strumento = p.id, problema = p.problema, dettaglio = p.dettaglio})
    end
  end
  local _, problemi = investimenti.switch(scope, ctx)
  for _, p in ipairs(problemi) do
    table.insert(out, {strumento = p.id or p.riferimento,
      problema = p.problema, dettaglio = p.data or p.dettaglio})
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

--- Campionamento della serie patrimoniale basato sulle date di quotazione e sugli snapshot.
--- Ogni punto ricalcola le quantità storiche (non usa le quote detenute oggi), ma tutti i punti
--- (al massimo 48) condividono lo stesso contesto: nessuna rilettura dello Space per data.
function investimenti.seriePatrimonio(pagina, periodo, tipo)
  local ctx = investimenti.contesto()
  local scope, ids = ambito(pagina, tipo, ctx)
  local fine = os.date("!%Y-%m-%d")
  local inizio = inizioPeriodo(periodo or "1A", fine)
  local date = {}
  local function aggiungi(d)
    if type(d) == "string" and d <= fine and (not inizio or d >= inizio) then date[d] = true end
  end
  -- Oltre ai NAV, i rendiconti Snapshot definiscono date reali del patrimonio.
  for _, r in ipairs(ctx.ledger) do
    if ids[r.id] and r.tipo == "Snapshot" and r.valore and r.valore >= 0 then aggiungi(r.data) end
  end
  for id in pairs(ids) do
    for _, r in ipairs(ctx.serie[id] or {}) do aggiungi(r[1]) end
  end
  local giorni = {}
  for d in pairs(date) do table.insert(giorni, d) end
  table.sort(giorni)
  local campioni, salta = {}, math.max(1, math.ceil(#giorni / 48))
  for i, d in ipairs(giorni) do
    if i == 1 or i == #giorni or i % salta == 0 then
      local totale, incompleto, attivi = 0, 0, 0
      for _, r in ipairs((investimenti.valoreAllaData(scope, d, ctx))) do
        if ids[r.id] and r.quote and r.quote > 0.0000001 then
          attivi = attivi + 1
          if r.valore ~= nil then totale = totale + r.valore else incompleto = incompleto + 1 end
        end
      end
      table.insert(campioni, {data = d, valore = (incompleto == 0 and attivi > 0) and totale or nil,
        mancanti = incompleto, attivi = attivi})
    end
  end
  return campioni
end

function investimenti.graficoPatrimonio(pagina, periodo, tipo)
  local serie = investimenti.seriePatrimonio(pagina, periodo, tipo)
  if #serie < 2 then return widget.markdown("_Dati insufficienti per il grafico patrimoniale._") end
  local etichette, valori, note = {}, {}, {}
  for _, r in ipairs(serie) do
    table.insert(etichette,investimenti.js(r.data))
    table.insert(valori,r.valore and string.format("%.6f",r.valore) or "null")
    table.insert(note,r.mancanti > 0 and "Copertura incompleta" or "")
  end
  local dati = "const labels=[" .. table.concat(etichette,",") .. "];const values=[" ..
    table.concat(valori,",") .. "];"
  local script = 'loadJsByUrl("' .. investimenti.chartjs .. '").then(function(){' .. dati ..
    'new Chart(document.getElementById("c"),{type:"line",data:{labels:labels,datasets:[{label:"Patrimonio EUR (quote storiche)",data:values,borderColor:"#487da3",spanGaps:false,pointRadius:0}]},options:{responsive:true,maintainAspectRatio:false,animation:false,scales:{y:{beginAtZero:false}}}});});'
  return widget.sandbox{html='<div style="height:300px;position:relative"><canvas id="c"></canvas></div>',
    script=script, markdown="Grafico patrimonio: " .. (pagina or "Index") ..
      ". Punti mancanti = serie non sufficientemente coperta."}
end

--- Grafico di uno strumento valorizzato a rendiconto (metodo snapshot/manuale): valore dei
--- rendiconti Snapshot e versato netto cumulato (acquisti − vendite), su tutti i conti.
function investimenti.graficoSnapshot(pagina)
  pagina = pagina or editor.getCurrentPage()
  if type(pagina) ~= "string" or not string.startsWith(pagina, "Strumenti/") then return nil end
  local id = string.sub(pagina, 11)
  local valore, versato, giorni, netto = {}, {}, {}, 0
  -- Il ledger è in ordine cronologico; a parità di data gli acquisti precedono lo snapshot.
  for _, r in ipairs(investimenti.contesto().ledger) do
    if r.id == id and r.data and r.valore and
      (r.tipo == "Snapshot" or r.tipo == "Acquisto" or r.tipo == "Vendita") then
      if r.tipo == "Snapshot" then valore[r.data] = (valore[r.data] or 0) + r.valore
      elseif r.tipo == "Acquisto" then netto = netto + r.valore
      else netto = netto - r.valore end
      if versato[r.data] == nil then table.insert(giorni, r.data) end
      versato[r.data] = netto
    end
  end
  if next(valore) == nil then return widget.markdownBlock("_Nessun rendiconto Snapshot per il grafico._") end
  local etichette, valori, versati = {}, {}, {}
  for _, d in ipairs(giorni) do
    table.insert(etichette, investimenti.js(d))
    table.insert(valori, valore[d] and string.format("%.2f", valore[d]) or "null")
    table.insert(versati, string.format("%.2f", versato[d]))
  end
  local dati = "const labels=[" .. table.concat(etichette, ",") .. "];const valori=[" ..
    table.concat(valori, ",") .. "];const versato=[" .. table.concat(versati, ",") .. "];"
  local script = 'loadJsByUrl("' .. investimenti.chartjs .. '").then(function(){' .. dati ..
    'new Chart(document.getElementById("c"),{type:"line",data:{labels:labels,datasets:[' ..
    '{label:"Valore (rendiconti)",data:valori,borderColor:"#487da3",spanGaps:true,pointRadius:3},' ..
    '{label:"Versato netto",data:versato,borderColor:"#d9822b",borderDash:[4,3],stepped:true,pointRadius:0}]},' ..
    'options:{responsive:true,maintainAspectRatio:false,animation:false,scales:{y:{beginAtZero:false}}}});});'
  return widget.sandbox{html = '<div style="height:300px;position:relative"><canvas id="c"></canvas></div>',
    script = script, markdown = "Grafico rendiconti: " .. pagina}
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
