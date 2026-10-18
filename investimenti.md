---
name: "Library/MG/investimenti"
tags: meta/library
version: "0.20"
versionDate: 2026-10-18
pageDecoration.prefix: "💸 "
share.uri: "github:marco10x15/silverbullet-libraries/investimenti.md"
---

# Investimenti

**Versione 0.19.8 — Dashboard patrimoniale, classificazione snapshot e indicatori essenziali, SilverBullet v2.11+.** Libreria completa, da sostituire integralmente a `Library/MG/investimenti`.

## Versione 0.20 — Pulizia del codice e API stabili

La libreria legge strumenti, conti, quotazioni e movimenti direttamente dallo Space SilverBullet. Non gestisce liquidità né spese del conto corrente. Gli snapshot si inseriscono come righe informative `Snapshot` nella tabella `Movimenti`. La dashboard rappresenta il costo residuo, il controvalore e il P/L latente; il grafico patrimoniale usa quantità storiche e rendiconti disponibili.

**Modifiche 0.20:** rimossi parametri `stopLoss`, `takeProfit`, `trailing`, `segnali`; rimossa la compatibilità frontmatter `valoreManuale`/`dataValore`; rimosso il parametro inutilizzato `periodo` dalla dashboard; eliminate `investimenti.nomi()` e `investimenti.flussiEsterni019()`; allineate le diagnostiche alla copertura snapshot; rimossi i suffissi numerici dalle funzioni di supporto. Per l'utente restano i metodi `dashboard`, `grafico`, `graficoPatrimonio`, `scheda`, `risultato`, `rendimento`, `copertura` e la diagnostica.

**Migrazione:** se esistono strumenti con `valoreManuale` e `dataValore` e senza righe `Snapshot`, inserire una riga `Snapshot` nella tabella `Movimenti` prima di installare la 0.20. Le chiamate `dashboard(pagina, tipo, periodo)` vanno convertite in `dashboard(pagina, tipo)`; il periodo rimane un argomento valido per `graficoPatrimonio`. I nomi interni con suffissi `018` e `019` non sono più esposti. I movimenti e le quotazioni non vengono modificati automaticamente.

## Esempi

```lua
investimenti.dashboard("Index")
investimenti.dashboard("Conti/NEF - Famiglia", "Fondo")
investimenti.scheda("Strumenti/LU0140696401")
investimenti.grafico("Strumenti/LU0140696401")
investimenti.graficoPatrimonio("Conti/NEF - Famiglia", "YTD")
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

--- Strumenti: pagine sotto Strumenti/, ordinate per id.
function investimenti.strumenti()
  local out = {}
  for _, p in ipairs(query[[
    from p = index.subPages("Strumenti")
    select {name = p.name, nome = p.displayName, isin = p.isin, ticker = p.ticker,
      valuta = p.valuta, tipo = p.tipo, rischio = p.KID,
      descrizione = p.description, tags = p.tags}
  ]]) do
    table.insert(out, {id = string.sub(p.name, 11),  -- "Strumenti/" = 10 caratteri
      nome = str(p.nome) or p.name, isin = str(p.isin), ticker = str(p.ticker),
      valuta = str(p.valuta) or "EUR", tipo = str(p.tipo),
      rischio = type(p.rischio) == "number" and p.rischio or nil,  -- p.rischio = attributo KID
      descrizione = str(p.descrizione), tags = p.tags})
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

-- La scheda usa solo metadati e relazioni già presenti, senza il motore legacy.
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
  local kid = tonumber(s.rischio)
  local rischio = "Rischio KID n.d."
  if kid and kid == math.floor(kid) and kid >= 1 and kid <= 7 then
    rischio = "Rischio KID " .. tostring(kid) .. "/7 " .. colori[kid]
  end

  local fisici, portafogli = investimenti.contiStrumento()
  local function links(pagine)
    local a = {}
    for _, nome in ipairs(pagine or {}) do table.insert(a, "[[" .. nome .. "]]") end
    table.sort(a)
    return #a > 0 and table.concat(a, ", ") or "nessuno"
  end
  -- Due righe compatte: nessun paragrafo vuoto; link wiki attivi anche nella seconda riga.
  local riga1 = "**" .. (s.tipo or "Strumento") .. " · " .. (s.valuta or "EUR") ..
    " · " .. rischio .. "**"
  local riga2 = "**Conti:** " .. links(fisici[id]) .. " · **Portafogli:** " .. links(portafogli[id])
  return widget.markdown(riga1 .. "  \n" .. riga2)
end

--- Pagine Quotazioni/...: restituisce due tabelle, isin -> pagina e pagina -> isin.
--- Il collegamento tra Strumenti/<id> e la sua pagina quotazioni è l'attributo isin.
function investimenti.pagineQuotazioni()
  local perIsin, perPagina = {}, {}
  for _, p in ipairs(query[[
    from p = index.subPages("Quotazioni")
    select {name = p.name, isin = p.isin}
  ]]) do
    local i = str(p.isin)
    if i then perIsin[i], perPagina[p.name] = p.name, i end
  end
  return perIsin, perPagina
end

--- Id degli strumenti di una pagina: Conti/ -> attributo strumento; Strumenti/ -> la pagina;
--- altrove nil (tutti gli strumenti). Senza argomento usa la pagina aperta.
function investimenti.elenco(pagina)
  pagina = pagina or editor.getCurrentPage()
  if type(pagina) ~= "string" then return nil end
  if string.startsWith(pagina, "Strumenti/") then return {string.sub(pagina, 11)} end
  if not string.startsWith(pagina, "Conti/") then return nil end

  local ids = {}
  for _, p in ipairs(query[[
    from p = index.subPages("Conti")
    select {name = p.name, strumento = p.strumento}
  ]]) do
    if p.name == pagina then
      local lista = p.strumento
      if type(lista) == "string" then lista = {lista} end
      if type(lista) == "table" then
        for _, v in ipairs(lista) do
          -- "[[Strumenti/X|alias]]" -> "Strumenti/X"
          local t = type(v) == "string" and string.match((string.gsub(v, "[%[%]]", "")), "^[^|]+")
          if t and string.startsWith(t, "Strumenti/") then table.insert(ids, string.sub(t, 11)) end
        end
      end
    end
  end
  return ids
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

--- Crea la pagina Quotazioni/<id> (con l'isin nel frontmatter) per ogni strumento con ticker e isin
--- che non ha ancora una pagina con quell'isin. Restituisce gli id creati e una tabella id -> pagina
--- (da passare ad aggiorna, perché l'indice potrebbe non conoscere ancora le pagine nuove).
function quotazioni.crea()
  local perIsin = investimenti.pagineQuotazioni()
  local create, pagine = {}, {}
  for _, s in ipairs(investimenti.strumenti()) do
    local nome = "Quotazioni/" .. s.id
    if s.ticker and s.isin and not perIsin[s.isin] and not space.pageExists(nome) then
      space.writePage(nome, '---\nisin: "' .. s.isin .. '"\n---\n# ' .. s.nome ..
        "\n\n${investimenti.grafico()}\n\n| Data | Prezzo |\n|---|---|\n")
      table.insert(create, s.id)
      pagine[s.id] = nome
    end
  end
  return create, pagine
end

--- Scarica e unisce le quotazioni (per data) nelle pagine Quotazioni con lo stesso isin dello strumento.
--- range: periodo Yahoo (default "1mo"); ids: elenco di id strumento (default: strumenti con ticker);
--- pagine: tabella id -> pagina per le pagine appena create (vedi crea).
-- Parse conservativo della sola tabella Data | Prezzo.
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

-- Aggiornamento conservativo: valida tutto prima di scrivere.
function quotazioni.aggiorna(range, ids, pagine)
  range, pagine = range or "1mo", pagine or {}
  local perIsin, duplicati, ticker, valuta, tutti = {}, {}, {}, {}, {}
  for _, p in ipairs(query[[
    from p = index.subPages("Quotazioni")
    select {name = p.name, isin = p.isin}
  ]]) do
    if type(p.isin) == "string" then
      if perIsin[p.isin] then duplicati[p.isin] = true end
      perIsin[p.isin] = p.name
    end
  end
  for _, s in ipairs(investimenti.strumenti()) do
    if s.ticker then
      ticker[s.id], valuta[s.id] = s.ticker, s.valuta
      table.insert(tutti, s.id)
      pagine[s.id] = pagine[s.id] or (s.isin and perIsin[s.isin])
      if s.isin and duplicati[s.isin] then pagine[s.id] = nil end
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
            valori[r.data] = string.format("%.8f", r.prezzo)
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

function investimenti.verifica()
  local anomalie, perIsin, perStrumento = {}, {}, {}
  local function avvisa(pagina, problema, dettaglio)
    table.insert(anomalie, {pagina = pagina, problema = problema, dettaglio = dettaglio})
  end
  for _, p in ipairs(query[[
    from p = index.subPages("Quotazioni")
    select {name = p.name, isin = p.isin}
  ]]) do
    if type(p.isin) ~= "string" or p.isin == "" then
      avvisa(p.name, "ISIN_MANCANTE", "Frontmatter senza isin")
    else
      if perIsin[p.isin] then avvisa(p.name, "ISIN_DUPLICATO", perIsin[p.isin]) end
      perIsin[p.isin] = p.name
    end
    local blocco, errore = quotazioni.blocco(space.readPage(p.name))
    if not blocco then avvisa(p.name, "TABELLA_NON_VALIDA", errore) end
  end
  for _, s in ipairs(investimenti.strumenti()) do
    if s.ticker and (not s.isin or not perIsin[s.isin]) then
      avvisa("Strumenti/" .. s.id, "QUOTAZIONI_MANCANTI", "Ticker senza pagina quotazioni collegata")
    end
    if s.isin then
      if perStrumento[s.isin] then
        avvisa("Strumenti/" .. s.id, "STRUMENTI_ISIN_DUPLICATO", perStrumento[s.isin])
      end
      perStrumento[s.isin] = s.id
    end
  end
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

local function num(x) return tonumber(x or 0) or 0 end

--- Elenco dei conti diretti dello strumento, distinguendo i portafogli logici.
--- Usa la proprietà `strumento` delle pagine Conti già presenti nella v0.16.
function investimenti.contiStrumento()
  local fisici, portfolio, noti = {}, {}, {}
  for _, p in ipairs(query[[
    from p = index.subPages("Conti")
    select {name = p.name, tags = p.tags}
  ]]) do
    noti[p.name] = true
    local aggregato = false
    if type(p.tags) == "string" then aggregato = string.find(p.tags, "Portafoglio", 1, true) ~= nil end
    if type(p.tags) == "table" then
      for _, t in ipairs(p.tags) do if t == "Portafoglio" then aggregato = true end end
    end
    for _, id in ipairs(investimenti.elenco(p.name) or {}) do
      local dest = aggregato and portfolio or fisici
      dest[id] = dest[id] or {}
      table.insert(dest[id], p.name)
    end
  end
  return fisici, portfolio, noti
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

--- Ledger grezzo con assegnazione per conto. Le righe non assegnate restano nell'aggregato globale.
function investimenti.ledger(ids)
  local ok, fisici, portfolio = nil, nil, nil
  fisici, portfolio = investimenti.contiStrumento()
  if ids then ok = {}; for _, id in ipairs(ids) do ok[id] = true end end
  local out = {}
  for _, r in ipairs(query[[
    from t = index.tables()
    where string.startsWith(t.page, "Strumenti/")
    select {page = t.page, ref = t.ref, giorno = t.data, tipo = t.tipo,
      quote = t.quantit_, valore = t.valore_eur or t.valore, conto = t.conto}
  ]]) do
    local id = string.sub(r.page, 11)
    if not ok or ok[id] then
      table.insert(out, {id = id, conto = investimenti.risolviConto(id, r.conto, fisici, portfolio),
        contoEsplicito = r.conto, data = r.giorno, tipo = r.tipo, quote = r.quote,
        valore = r.valore, ref = r.ref})
    end
  end
  return out
end

--- Quote, acquistato e incassato (EUR) per ogni strumento; ids opzionale per filtrare.
function investimenti.verificaConti()
  local out, fisici, portfolio, noti = {}, investimenti.contiStrumento()
  local function errore(p, tipo, dettaglio)
    table.insert(out, {pagina = p, problema = tipo, dettaglio = dettaglio})
  end
  local strumenti = {}
  for _, s in ipairs(investimenti.strumenti()) do strumenti[s.id] = true end
  for id, pp in pairs(fisici) do
    if not strumenti[id] then errore(id, "STRUMENTO_INESISTENTE", table.concat(pp, ", ")) end
  end
  for id, pp in pairs(portfolio) do
    if not strumenti[id] then errore(id, "STRUMENTO_INESISTENTE", table.concat(pp, ", ")) end
  end
  for _, r in ipairs(investimenti.ledger()) do
    if r.contoEsplicito and r.conto and not noti[r.conto] then
      errore("Strumenti/" .. r.id, "CONTO_INESISTENTE", r.conto)
    end
    if not r.conto then
      errore("Strumenti/" .. r.id, "CONTO_NON_ASSEGNATO", tostring(r.data))
    end
    if (r.tipo == "Acquisto" or r.tipo == "Vendita") and
      (not tonumber(r.quote) or not tonumber(r.valore)) then
      errore("Strumenti/" .. r.id, "MOVIMENTO_NON_VALIDO", tostring(r.data))
    end
  end
  return out
end

--- Quotazioni raggruppate per strumento: tabella id -> lista {data, prezzo} (non ordinata).
--- Lo strumento è collegato alla pagina Quotazioni dall'attributo isin. ids opzionale (filtro);
--- strumenti: risultato di investimenti.strumenti(), se già disponibile.
function investimenti.quotazioniPerStrumento(strumenti, ids)
  local ammessi
  if ids then
    ammessi = {}
    for _, id in ipairs(ids) do ammessi[id] = true end
  end
  local perIsin, perPagina = investimenti.pagineQuotazioni()
  local idDi = {}  -- isin -> lista id strumenti: PIR Marco e Franca condividono un NAV.
  for _, s in ipairs(strumenti or investimenti.strumenti()) do
    if s.isin and (not ammessi or ammessi[s.id]) then
      idDi[s.isin] = idDi[s.isin] or {}
      table.insert(idDi[s.isin], s.id)
    end
  end

  local per = {}
  for _, r in ipairs(query[[
    from t = index.tables()
    where string.startsWith(t.page, "Quotazioni/")
    select {pagina = t.page, ref = t.ref, giorno = t.data, px = t.prezzo}
  ]]) do
    local collegati = idDi[perPagina[r.pagina]] or {}
    local pr = tonumber(r.px)
    if type(r.giorno) == "string" and pr then
      for _, id in ipairs(collegati) do
        per[id] = per[id] or {}
        table.insert(per[id], {r.giorno, pr})
      end
    end
  end
  return per
end

--- Ultimo prezzo di ogni strumento con quotazioni: lista {id, data, prezzo, valuta}.
--- strumenti: risultato di investimenti.strumenti(); per: risultato di quotazioniPerStrumento,
--- se già disponibili (evita riletture).
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
    local pr = tonumber(r.px)
    if type(r.giorno) == "string" and pr then table.insert(out, {r.giorno, pr}) end
  end
  table.sort(out, function(a, b) return a[1] < b[1] end)
  return out
end

--- Acquisti di uno strumento: lista di {data, prezzo unitario in valuta strumento, quote}.
function investimenti.acquisti(id)
  local out = {}
  for _, r in ipairs(query[[
    from t = index.tables()
    where t.page == "Strumenti/" .. id and t.tipo == "Acquisto"
    select {ref = t.ref, giorno = t.data, px = t.prezzo, qt = t.quantit_}
  ]]) do
    local pr = tonumber(r.px)
    if type(r.giorno) == "string" and pr then table.insert(out, {r.giorno, pr, tonumber(r.qt) or 0}) end
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
  local perIsin, perPagina = investimenti.pagineQuotazioni()
  local id, quot, conAcquisti
  if string.startsWith(pagina, "Quotazioni/") then
    quot = pagina
    for _, s in ipairs(investimenti.strumenti()) do
      if s.isin and s.isin == perPagina[pagina] then id = s.id end
    end
  elseif string.startsWith(pagina, "Strumenti/") then
    id, conAcquisti = string.sub(pagina, 11), true
    for _, s in ipairs(investimenti.strumenti()) do
      if s.id == id then quot = s.isin and perIsin[s.isin] end
    end
  else return widget.markdownBlock("_Il grafico vale per pagine Quotazioni/ e Strumenti/._") end
  if not quot then return widget.markdownBlock("_Nessuna pagina Quotazioni con l'isin dello strumento._") end

  local serie = investimenti.serie(quot)
  if #serie < 2 then return widget.markdownBlock("_Quotazioni insufficienti per il grafico._") end
  local acq = conAcquisti and investimenti.acquisti(id) or {}

  local function coppie(lista)
    local p = {}
    for _, c in ipairs(lista) do table.insert(p, "[" .. investimenti.js(c[1]) .. "," .. c[2] .. "]") end
    return "[" .. table.concat(p, ",") .. "]"
  end
  local nome = id or quot
  for _, s in ipairs(investimenti.strumenti()) do if s.id == id then nome = s.nome end end
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

-- Metadati correnti di valorizzazione degli strumenti.
function investimenti.metadati()
  local out = {}
  for _, p in ipairs(query[[
    from p = index.subPages("Strumenti")
    select {name = p.name, metodo = p.metodoValorizzazione,
      frequenza = p.frequenzaPrezzoGiorni, quoteReali = p.quoteReali}
  ]]) do
    out[string.sub(p.name, 11)] = {metodo = p.metodo,
      frequenza = tonumber(p.frequenza), quoteReali = p.quoteReali}
  end
  return out
end

function investimenti.metodo(s, meta)
  if type(s) ~= "table" then return "manuale" end
  local m = meta or {}
  local dichiarato = type(m.metodo) == "string" and m.metodo
  local ammessi = {mercato = true, nav = true, obbligazione = true,
    contrattuale = true, snapshot = true, manuale = true, nominale = true, costo = true}
  if dichiarato and ammessi[dichiarato] then return dichiarato end
  if s.tipo == "Cambio" then return "mercato" end
  if s.tipo == "PIR" then return "snapshot" end
  if s.tipo == "Fondo" then return "nav" end
  if s.tipo == "Cripto" then return "mercato" end
  if s.tipo == "Obbligazione" then
    if string.startsWith(s.id, "CD-") then return "contrattuale" end
    return "obbligazione"
  end
  return "mercato"
end

-- Restituisce il prezzo noto non successivo alla data richiesta.
local function prezzoPrima(serie, data)
  local scelta
  for _, r in ipairs(serie or {}) do
    if type(r[1]) == "string" and r[1] <= data and type(r[2]) == "number"
       and r[2] > 0 and (not scelta or r[1] > scelta[1]) then scelta = r end
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

function investimenti.prezzi(data)
  data = data or os.date("!%Y-%m-%d")
  local strumenti = investimenti.strumenti()
  local per = investimenti.quotazioniPerStrumento(strumenti)
  local px = {}
  for _, s in ipairs(strumenti) do
    local r = prezzoPrima(per[s.id], data)
    if r then
      local eur = r[2]
      local dtCambio
      if s.valuta == "USD" and s.id ~= "EURUSD" then
        local cambioData = prezzoPrima(per.EURUSD, r[1])
        if cambioData and cambioData[2] > 0 then eur, dtCambio = eur / cambioData[2], cambioData[1]
        else eur = nil end
      end
      px[s.id] = {data = r[1], prezzo = r[2], eur = eur, dataCambio = dtCambio}
    end
  end
  return px
end

-- Posizioni e lotti PMC, calcolati cronologicamente per conto + strumento.
function investimenti.lotti(pagina, data)
  data = data or os.date("!%Y-%m-%d")
  local idOk = investimenti.elenco(pagina)
  local ammessi
  if idOk then ammessi = {}; for _, id in ipairs(idOk) do ammessi[id] = true end end
  local conto = type(pagina) == "string" and string.startsWith(pagina, "Conti/") and pagina
  local portafoglio = false
  if conto then
    for _, p in ipairs(query[[
      from p = index.subPages("Conti")
      select {name = p.name, tags = p.tags}
    ]]) do
      if p.name == conto then
        if type(p.tags) == "string" then portafoglio = string.find(p.tags, "Portafoglio", 1, true) ~= nil end
        if type(p.tags) == "table" then
          for _, tag in ipairs(p.tags) do if tag == "Portafoglio" then portafoglio = true end end
        end
      end
    end
  end
  local righe = investimenti.ledger(idOk)
  table.sort(righe, function(a,b)
    if tostring(a.data) ~= tostring(b.data) then return tostring(a.data) < tostring(b.data) end
    return tostring(a.ref or "") < tostring(b.ref or "")
  end)
  local per, chiavi = {}, {}
  for _, r in ipairs(righe) do
    local include = (not ammessi or ammessi[r.id]) and type(r.data) == "string" and r.data <= data
    if include and conto then
      include = r.conto ~= nil and (r.conto == conto or portafoglio)
    end
    if include then
      local key = (r.conto or "NON_ASSEGNATO") .. "|" .. r.id
      if not per[key] then
        per[key] = {id = r.id, conto = r.conto, quote = 0, costo = 0,
          realizzato = 0, redditi = 0, acquisti = 0, vendite = 0,
          errori = {}, flussi = {}, primo = nil}
        table.insert(chiavi, key)
      end
      local v, q, p = tonumber(r.valore), tonumber(r.quote), per[key]
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

-- Snapshot informativi: un controvalore in EUR per posizione e data.
-- Non contribuiscono ai lotti o ai flussi finanziari. Il conto viene risolto
-- dal ledger esistente, così gli snapshot rimangono nella stessa posizione.
function investimenti.snapshotDati(pagina, data)
  data = data or os.date("!%Y-%m-%d")
  local ids = investimenti.elenco(pagina)
  local conto = type(pagina) == "string" and string.startsWith(pagina,"Conti/") and pagina
  local aggregato = false
  if conto then
    for _, x in ipairs(query[[
      from p = index.subPages("Conti")
      select {name = p.name, tags = p.tags}
    ]]) do
      if x.name == conto then
        local tags = x.tags
        if type(tags) == "string" then aggregato = string.find(tags,"Portafoglio",1,true) ~= nil end
        if type(tags) == "table" then
          for _, t in ipairs(tags) do if t == "Portafoglio" then aggregato = true end end
        end
      end
    end
  end
  local out = {}
  for _, r in ipairs(investimenti.ledger(ids)) do
    if type(r.tipo) == "string" and string.lower(r.tipo) == "snapshot" and
      type(r.data) == "string" and string.match(r.data,"^%d%d%d%d%-%d%d%-%d%d$") and r.data <= data and
      (not conto or r.conto == conto or aggregato) then
      local v = tonumber(r.valore)
      if v and v >= 0 then
        local key = (r.conto or "NON_ASSEGNATO") .. "|" .. r.id
        out[key] = out[key] or {}
        table.insert(out[key], {data=r.data, valore=v})
      end
    end
  end
  for _, serie in pairs(out) do
    table.sort(serie,function(a,b) return a.data < b.data end)
  end
  return out
end

-- Apporti e rimborsi posteriori all'ultimo rendiconto: una PROXY
-- contabile, non una quotazione o una variazione certificata di mercato.
function investimenti.movimentiDopoSnapshot(id, conto, dataSnapshot, dataFine, ledger)
  local netto = 0
  for _, r in ipairs(ledger or investimenti.ledger({id})) do
    local v = tonumber(r.valore)
    if r.id == id and (r.conto or "NON_ASSEGNATO") == (conto or "NON_ASSEGNATO") and
      type(r.data) == "string" and r.data > dataSnapshot and r.data <= dataFine and v then
      if r.tipo == "Acquisto" then netto = netto + v
      elseif r.tipo == "Vendita" then netto = netto - v end
    end
  end
  return netto
end

-- Valorizzazione datata: NAV e mercato su quotazioni; contrattuale/nominale su
-- capitale effettivo; snapshot/manuale solo con valore di posizione esplicito.
-- I PIR dell'esempio: l'unità sintetica di 1 EUR NON è un numero di quote NAV.
function investimenti.valoreAllaData(pagina, data)
  data = data or os.date("!%Y-%m-%d")
  local metadati = investimenti.metadati()
  local strumenti = {}; for _, s in ipairs(investimenti.strumenti()) do strumenti[s.id] = s end
  local prezzi = investimenti.prezzi(data)
  local snapshots = investimenti.snapshotDati(pagina, data)
  local operazioni = investimenti.ledger(investimenti.elenco(pagina))
  local righe, totale = {}, {valore = 0, note = 0, mancanti = 0, stimati = 0, posizioni = 0}
  for _, p in ipairs(investimenti.lotti(pagina, data)) do
    local s = strumenti[p.id] or {id = p.id, tipo = ""}
    local meta = metadati[p.id] or {}
    local metodo = investimenti.metodo(s, meta)
    local px = prezzi[p.id]
    local valore, stato, dt, fonte
    if p.quote == 0 then valore, stato = 0, "CHIUSA"
    elseif #p.errori > 0 then stato = "MOVIMENTI_INVALIDI"
    elseif (metodo == "mercato" or metodo == "nav" or metodo == "obbligazione") then
      if metodo == "nav" and s.tipo == "PIR" and meta.quoteReali ~= true then
        stato = "QUOTE_NAV_NON_VERIFICATE"
      elseif px and px.eur then
        valore, dt, fonte = p.quote * px.eur, px.data, "quotazione"
        stato = "PREZZO_DISPONIBILE"
      else stato = px and "CAMBIO_MANCANTE" or "PREZZO_MANCANTE" end
    elseif metodo == "snapshot" or metodo == "manuale" then
      local key = (p.conto or "NON_ASSEGNATO") .. "|" .. p.id
      local serie = snapshots[key] or {}
      local ultimo = serie[#serie]
      if ultimo then
        local flussi = investimenti.movimentiDopoSnapshot(p.id,p.conto,ultimo.data,data,operazioni)
        valore, dt = ultimo.valore + flussi, ultimo.data
        if math.abs(flussi) > 0.000001 then
          stato, fonte = "SNAPSHOT_CON_APPORTI", "snapshot_piu_movimenti_indicativo"
        else
          stato, fonte = "SNAPSHOT_DISPONIBILE", fonte or "snapshot_movimenti"
        end
      elseif px and px.eur and meta.quoteReali == true then
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
      if stato ~= "PREZZO_DISPONIBILE" and stato ~= "SNAPSHOT_DISPONIBILE" and stato ~= "CHIUSA" then totale.stimati = totale.stimati + 1 end
    else totale.mancanti = totale.mancanti + 1 end
    totale.posizioni = totale.posizioni + 1
    table.insert(righe, {id = p.id, conto = p.conto, metodo = metodo, quote = p.quote,
      valore = valore, stato = stato, fonte = fonte, dataPrezzo = dt,
      prezzo = px and px.prezzo, dataCambio = px and px.dataCambio, costo = p.costo})
  end
  return righe, totale
end

function investimenti.risultatoDati(pagina, data)
  local valori = investimenti.valoreAllaData(pagina, data)
  local per = {}
  for _, r in ipairs(valori) do per[(r.conto or "NON_ASSEGNATO") .. "|" .. r.id] = r end
  local out, tot = {}, {realizzato = 0, latente = 0, redditi = 0, complessivo = 0, indisponibili = 0}
  for _, p in ipairs(investimenti.lotti(pagina, data)) do
    local v = per[(p.conto or "NON_ASSEGNATO") .. "|" .. p.id]
    local latente = v and v.valore and (v.valore - p.costo) or nil
    local risultato = latente and (p.realizzato + latente + p.redditi) or nil
    tot.realizzato, tot.redditi = tot.realizzato + p.realizzato, tot.redditi + p.redditi
    if latente then tot.latente, tot.complessivo = tot.latente + latente, tot.complessivo + risultato
    else tot.indisponibili = tot.indisponibili + 1 end
    table.insert(out, {id = p.id, conto = p.conto, costo = p.costo,
      realizzato = p.realizzato, latente = latente, redditi = p.redditi,
      risultato = risultato, stato = v and v.stato, errori = p.errori})
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

local function xirr(flussi)
  if #flussi < 2 then return nil, "FLUSSI_INSUFFICIENTI" end
  local d0 = flussi[1].data
  local neg, pos = false, false
  for _, f in ipairs(flussi) do
    if f.valore < 0 then neg = true end
    if f.valore > 0 then pos = true end
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
  -- La ricerca nel dominio (-1, +10000) individua solo un cambio di segno;
  -- multipli IRR non sono certificabili, vengono rifiutati.
  local intervalli = {-0.9999, -0.99, -0.9, -0.5, 0, 0.05, 0.1, 0.25, 0.5, 1, 2, 5, 10, 100, 10000}
  local a, b, cambi = nil, nil, 0
  for i = 2, #intervalli do
    local x, y = intervalli[i-1], intervalli[i]
    local fx, fy = npv(x), npv(y)
    if fx and fy and fx * fy <= 0 then a, b, cambi = x, y, cambi + 1 end
  end
  if cambi ~= 1 then return nil, cambi == 0 and "NESSUNA_RADICE" or "RADICI_MULTIPLE" end
  local fa = npv(a)
  for _ = 1, 100 do
    local m = (a + b) / 2
    local fm = npv(m)
    if not fm then return nil, "DATE_NON_VALIDE" end
    if math.abs(fm) < 1e-8 then return m * 100 end
    if fa * fm <= 0 then b = m else a, fa = m, fm end
  end
  return ((a + b) / 2) * 100
end

function investimenti.rendimentoDati(pagina, metodo)
  pagina = pagina or editor.getCurrentPage()
  metodo = metodo or "xirr"
  if metodo ~= "xirr" and metodo ~= "twr" then return {errore = "METODO_SCONOSCIUTO"} end
  local id = type(pagina) == "string" and string.startsWith(pagina, "Strumenti/") and string.sub(pagina, 11)
  if not id then return {errore = "PERIMETRO_NON_RICONCILIATO", dettaglio = "Richiesto singolo strumento: mancano liquidita e flussi esterni del conto"} end
  local parti = investimenti.lotti(pagina)
  if #parti ~= 1 then return {errore = "POSIZIONI_MULTIPLE", dettaglio = "Serve una posizione univoca per conto"} end
  local p = parti[1]
  if #p.errori > 0 then return {errore = "MOVIMENTI_INVALIDI"} end
  local data = os.date("!%Y-%m-%d")
  local valori = investimenti.valoreAllaData(pagina, data)
  local v = valori[1]
  if not v or v.valore == nil then return {errore = v and v.stato or "VALORE_MANCANTE"} end
  if v.stato ~= "PREZZO_DISPONIBILE" and v.stato ~= "CHIUSA" then
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
  table.sort(flussi, function(a,b) return a.data < b.data end)
  local rendimento, errore = xirr(flussi)
  return {metodo = metodo, percentuale = rendimento, errore = errore,
    nota = "XIRR annualizzato; flussi della singola posizione"}
end

function investimenti.rendimento(pagina, metodo)
  local r = investimenti.rendimentoDati(pagina, metodo)
  if r.errore then return widget.markdown("**Rendimento n.d.**: " .. r.errore) end
  return widget.markdown("**" .. string.upper(r.metodo) .. "**: " ..
    investimenti.formatoNumero(r.percentuale, 2) .. " %" .. " — " .. r.nota)
end

function investimenti.coperturaDati(pagina, data)
  data = data or os.date("!%Y-%m-%d")
  local righe, tot = investimenti.valoreAllaData(pagina, data)
  local meta = investimenti.metadati()
  local report = {aggiornati = 0, obsoleti = 0, mancanti = tot.mancanti,
    stimati = tot.stimati, totalePosizioni = tot.posizioni, valoreParziale = tot.valore}
  for _, r in ipairs(righe) do
    local m = meta[r.id] or {}
    local limite = m.frequenza or
      ((r.metodo == "snapshot" or r.metodo == "contrattuale") and 365 or 7)
    local gg = r.dataPrezzo and giorniTraDate(r.dataPrezzo, data)
    r.etaGiorni = gg
    r.sogliaGiorni = limite
    if r.valore ~= nil and r.stato == "SNAPSHOT_DISPONIBILE" then
      r.copertura = "SNAPSHOT_UFFICIALE"; report.aggiornati = report.aggiornati + 1
    elseif r.valore ~= nil and r.stato == "PREZZO_DISPONIBILE" then
      if gg and gg <= limite and gg >= 0 then
        r.copertura = "AGGIORNATO"; report.aggiornati = report.aggiornati + 1
      else r.copertura = "OBSOLETO"; report.obsoleti = report.obsoleti + 1 end
    elseif r.valore ~= nil then r.copertura = "INDICATIVO"
    else r.copertura = "MANCANTE" end
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

function investimenti.verificaPatrimonio()
  local righe = investimenti.coperturaDati("Index")
  local out = {}
  for _, r in ipairs(righe) do
    if r.copertura ~= "AGGIORNATO" and r.copertura ~= "SNAPSHOT_UFFICIALE" and r.stato ~= "CHIUSA" then
      table.insert(out, {pagina = "Strumenti/" .. r.id, conto = r.conto,
        problema = r.copertura, dettaglio = r.stato, data = r.dataPrezzo})
    end
  end
  return out
end
```

```space-lua
-- Dashboard: nessuna scrittura, lettura del motore patrimoniale corrente.
investimenti = investimenti or {}

local function ambito(pagina, tipo)
  pagina = pagina or "Index"
  local ids = investimenti.elenco(pagina)
  local ammessi = {}
  if ids then for _, id in ipairs(ids) do ammessi[id] = true end end
  local nomi = {}
  for _, s in ipairs(investimenti.strumenti()) do
    nomi[s.id] = s
    if not ids then ammessi[s.id] = true end
    if tipo and tipo ~= "Tutti" and s.tipo ~= tipo then ammessi[s.id] = nil end
  end
  return pagina, ammessi, nomi
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
function investimenti.dashboardDati(pagina, tipo)
  local scope, ammessi, nomi = ambito(pagina, tipo)
  local attuali = investimenti.valoreAllaData(scope)
  local conti = investimenti.lotti(scope)
  local per = {}
  for _, p in ipairs(conti) do per[(p.conto or "") .. "|" .. p.id] = p end
  local rows, t = {}, {costo = 0, valore = 0, pl = 0, conteggio = 0, mancanti = 0,
    indicativi = 0, ufficiali = 0, chiuse = 0, kidSomma = 0, kidPeso = 0, kidSenza = 0}
  for _, r in ipairs(attuali) do
    if ammessi[r.id] then
      local p = per[(r.conto or "") .. "|" .. r.id]
      local q = p and p.quote or r.quote
      if q and q > 0.0000001 then
        t.conteggio = t.conteggio + 1
        if r.valore == nil then t.mancanti = t.mancanti + 1
        else
          if r.stato == "SNAPSHOT_DISPONIBILE" then
            t.ufficiali = t.ufficiali + 1
          elseif r.stato ~= "PREZZO_DISPONIBILE" then
            t.indicativi = t.indicativi + 1
          end
          local kid = tonumber(nomi[r.id] and nomi[r.id].rischio)
          if kid and kid >= 1 and kid <= 7 and r.valore > 0 then
            t.kidSomma, t.kidPeso = t.kidSomma + kid * r.valore, t.kidPeso + r.valore
          else t.kidSenza = t.kidSenza + 1 end
          local costo = p and p.costo or 0
          t.costo, t.valore, t.pl = t.costo + costo, t.valore + r.valore, t.pl + r.valore - costo
          table.insert(rows, {id=r.id, nome=(nomi[r.id] and nomi[r.id].nome) or r.id,
            conto=r.conto, tipo=nomi[r.id] and nomi[r.id].tipo, costo=costo,
            valore=r.valore, pl=r.valore-costo,
            dataPrezzo=r.dataPrezzo, stato=r.stato})
        end
      else t.chiuse = t.chiuse + 1 end
    end
  end
  table.sort(rows, function(a,b) return a.nome < b.nome end)
  t.completo = t.mancanti == 0 and t.indicativi == 0
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
      tot.indicativi .. " indicative, " .. tot.mancanti .. " mancanti" ..
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
function investimenti.switch(pagina)
  local _, ids = ambito(pagina, nil)
  local out, gruppi = {}, {}
  for _, r in ipairs(query[[
    from t = index.tables()
    where string.startsWith(t.page, "Strumenti/")
    select {pagina = t.page, giorno = t.data, tipo = t.tipo,
      causale = t.causale, riferimento = t.riferimento,
      valore = t.valore_eur or t.valore}
  ]]) do
    local id = string.sub(r.pagina, 11)
    if ids[id] and type(r.causale) == "string" and
       string.lower(r.causale) == "switch" then
      local x = {id=id, data=r.giorno, tipo=r.tipo, riferimento=r.riferimento, valore=tonumber(r.valore)}
      table.insert(out,x)
      if type(r.riferimento) == "string" and r.riferimento ~= "" then
        if not gruppi[r.riferimento] then gruppi[r.riferimento] = {} end
        table.insert(gruppi[r.riferimento],x)
      end
    end
  end
  local problemi = {}
  for _, r in ipairs(out) do
    if type(r.riferimento) ~= "string" or r.riferimento == "" then
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

function investimenti.diagnosticaDati(pagina)
  local out = {}
  local scope, ids = ambito(pagina,nil)
  for _, r in ipairs(investimenti.coperturaDati(scope)) do
    if ids[r.id] and r.copertura ~= "AGGIORNATO" and r.copertura ~= "SNAPSHOT_UFFICIALE" and r.stato ~= "CHIUSA" then
      table.insert(out,{strumento=r.id, problema=r.copertura, dettaglio=r.stato})
    end
  end
  local _, problemi = investimenti.switch(scope)
  for _, p in ipairs(problemi) do
    table.insert(out,{strumento=p.id or p.riferimento,
      problema=p.problema,dettaglio=p.data or p.dettaglio})
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

--- Campionamento della serie patrimoniale basato sulle date di quotazione.
--- Ogni punto ricalcola le quantità storiche: non usa le quote detenute oggi.
function investimenti.seriePatrimonio(pagina, periodo, tipo)
  local scope, ids, nomi = ambito(pagina, tipo)
  local fine = os.date("!%Y-%m-%d")
  local inizio = inizioPeriodo(periodo or "1A", fine)
  local serie, date = investimenti.quotazioniPerStrumento(investimenti.strumenti()) , {}
  -- Oltre ai NAV, i rendiconti Snapshot definiscono date reali del patrimonio.
  for _, r in ipairs(investimenti.ledger()) do
    if ids[r.id] and type(r.tipo) == "string" and string.lower(r.tipo) == "snapshot" and
      type(r.data) == "string" and r.data <= fine and (not inizio or r.data >= inizio) and
      tonumber(r.valore) and tonumber(r.valore) >= 0 then
      date[r.data] = true
    end
  end
  for id in pairs(ids) do
    for _, r in ipairs(serie[id] or {}) do
      if type(r[1]) == "string" and r[1] <= fine and (not inizio or r[1] >= inizio) then
        date[r[1]] = true
      end
    end
  end
  local giorni = {}
  for d in pairs(date) do table.insert(giorni,d) end
  table.sort(giorni)
  local campioni, salta = {}, math.max(1, math.ceil(#giorni/48))
  for i,d in ipairs(giorni) do
    if i == 1 or i == #giorni or i % salta == 0 then
      local totale, incompleto, attivi = 0,0,0
      for _, r in ipairs(investimenti.valoreAllaData(scope,d)) do
        if ids[r.id] and r.quote and r.quote > 0.0000001 then
          attivi=attivi+1
          if r.valore ~= nil then totale=totale+r.valore else incompleto=incompleto+1 end
        end
      end
      table.insert(campioni,{data=d, valore=(incompleto==0 and attivi>0) and totale or nil,
        mancanti=incompleto, attivi=attivi})
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
--- Widget automatici moderni, solo se configurati (API pubblica SilverBullet v2).
function investimenti.vista(pagina)
  pagina = pagina or editor.getCurrentPage()
  if type(pagina) ~= "string" then return {} end
  local function v(f, p) return function() return f(p) end end
  if pagina == (config.get("investimenti.paginaIndice") or "Index") then
    return {v(investimenti.dashboard, pagina), v(investimenti.diagnosticaVista, pagina)}
  elseif string.startsWith(pagina, "Conti/") then
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
