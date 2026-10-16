---
name: "Library/MG/investimenti"
tags: meta/library
version: "0.17"
versionDate: 2026-10-16
pageDecoration.prefix: "💸 "
share.uri: "github:marco10x15/silverbullet-libraries/investimenti.md"
---

# 💸 Investimenti

**Versione 0.17 — 16.10.2026** · Libreria per SilverBullet v2.11+.

Gestione degli strumenti finanziari in Markdown: movimenti, quotazioni storiche, valorizzazioni in EUR, andamento e indicatori. Le pagine Markdown sono la fonte primaria; le sole scritture automatiche riguardano `Quotazioni/...` mediante `quotazioni.crea()` e `quotazioni.aggiorna()`.

> **Prima di sostituire la pagina**: conservare un backup della libreria e dello Space. Questa versione mantiene lo schema delle pagine e le funzioni pubbliche della 0.15. Il codice è stato verificato strutturalmente ma non eseguito in SilverBullet: provarlo prima su Space di test.

## Novità 0.16

- Aggiornamento conservativo della tabella `Data | Prezzo`: le note successive e precedenti non vengono eliminate.
- Blocco della scrittura su dati storici illeggibili, date o prezzi non validi, ISIN duplicati e valuta Yahoo diversa dalla valuta dichiarata.
- Otto decimali per i nuovi prezzi Yahoo; i prezzi storici già memorizzati conservano la precisione originale.
- Nuova `investimenti.verifica()` in sola lettura per anomalie di struttura, pagine e ISIN.
- Nessuna migrazione obbligatoria delle tabelle `Quotazioni/...` già presenti in 0.15.

## Struttura dello Space

- `Strumenti/<id>`: frontmatter con `displayName`, `isin`, `ticker`, `valuta`, `tipo`, `KID` e tabella dei movimenti. `ticker` è facoltativo per quotazioni manuali.
- `Quotazioni/<id>`: frontmatter con `isin`, tabella `Data | Prezzo` e grafici opzionali. Le quotazioni sono collegate mediante l'ISIN.
- `Conti/<nome>`: attributo `strumento` con link `[[Strumenti/<id>]]`; conti e portafogli possono essere visualizzati individualmente.
- `Index`: cruscotto generale; la pagina può essere personalizzata con `config.set("investimenti.paginaIndice", "...")`.

La libreria supporta prezzi in EUR e USD; i prezzi in USD vengono convertiti mediante l'ultimo `EURUSD` disponibile. La performance 1M/YTD/1A è una variazione della quotazione, non un rendimento ponderato sui flussi di cassa. Il P/L è calcolato come valore + incassato − acquistato. Non include automaticamente tutte le imposte, i costi di conto e le particolarità delle obbligazioni.

## API pubbliche

| Funzione | Uso |
|---|---|
| `investimenti.strumenti()` | Elenco strumenti e frontmatter |
| `investimenti.elenco(pagina)` | Strumenti contenuti nel conto o nella pagina |
| `investimenti.movimentiDati(ids)` | Quote nette, acquisti e incassi |
| `investimenti.quotazioniPerStrumento(strumenti, ids)` | Serie storiche per ID |
| `investimenti.ultimiPrezzi(strumenti, per)` | Ultimo prezzo disponibile |
| `investimenti.valorizzazioneDati(ids)` | Dati numerici e totali |
| `investimenti.valorizzazione(pagina)` | Valorizzazione dettagliata |
| `investimenti.valore(pagina)` | Tabella compatta con performance |
| `investimenti.movimenti(pagina)` | Movimenti aggregati |
| `investimenti.performance(ids)` | Variazione quotazioni 1M/YTD/1A |
| `investimenti.serie(id)` / `investimenti.acquisti(id)` | Dati per grafici |
| `investimenti.grafico(pagina)` | Prezzi, MM200, Bollinger, RSI, stop/TP/trailing |
| `investimenti.torta(pagina)` / `investimenti.tortaRischio(pagina)` | Distribuzioni per strumento e KID |
| `investimenti.allocazione(pagina)` | Entrambe le distribuzioni |
| `investimenti.barraRischio(n)` / `investimenti.rischio(pagina)` | Indicatore KID per singolo strumento e aggregato |
| `investimenti.barrePL(pagina)` | Grafico P/L per strumento |
| `investimenti.scheda(pagina)` / `investimenti.elencoStrumenti()` / `investimenti.elencoConti()` | Viste anagrafiche |
| `investimenti.vista(pagina)` | Widget automatici per pagina |
| `investimenti.verifica()` | Anomalie struttura, ISIN e tabelle (v0.16) |
| `quotazioni.yahoo(simbolo, range)` | Storico Yahoo Finance |
| `quotazioni.cerca(testo)` / `quotazioni.verifica(simbolo, range)` / `quotazioni.test()` | Ricerca e diagnostica Yahoo |
| `quotazioni.crea()` / `quotazioni.aggiorna(range, ids, pagine)` | Creazione e aggiornamento pagine quotazioni |

Le funzioni che accettano `pagina` usano per impostazione predefinita la pagina aperta nell'editor; `ids` è un elenco di identificatori senza prefisso `Strumenti/`. Richiede facoltativamente `DateFormat` per mostrare le date nel formato `DD.MM.YYYY`.

## Quotazioni: aggiornamento e integrità

`quotazioni.aggiorna()` scarica per default un mese di dati; `quotazioni.aggiorna("5y")` richiede cinque anni. Prima della scrittura il parser `quotazioni.blocco()` verifica la tabella corrente, poi valida tutti i punti scaricati. Se un controllo fallisce, la pagina non viene modificata e il risultato riporta `errore`.

La tabella riconosciuta deve avere esattamente due colonne `Data | Prezzo`, date ISO e prezzi positivi separati dal punto. Il contenuto prima e dopo la tabella viene mantenuto; se esistono più intestazioni omonime, l'aggiornamento viene bloccato. Yahoo Finance è una fonte non ufficiale: non si garantiscono disponibilità, puntualità o correttezza dei dati.

`investimenti.verifica()` restituisce una tabella con `pagina`, `problema`, `dettaglio`, senza modificare dati. Controlli inclusi: ISIN assenti/duplicati, pagine quotazioni prive di tabella valida, strumenti con ticker senza pagina collegata.

**Limiti:** le quotazioni obsolete e l'allineamento storico del cambio USD/EUR non vengono ancora bloccati dal motore patrimoniale. L'associazione movimento-conto, le cedole obbligazionarie e il costo medio delle quote residue sono previsti per una versione successiva.

## Esempi di pagine

### Strumento

```markdown
---
displayName: "NEF Ethical Euro Equity R Acc"
isin: "LU0102238655"
ticker: "0P00000I8A.F"
tipo: Fondo
valuta: EUR
KID: 4
---

| Data | Tipo | Quantità | Prezzo | Valore | Commissioni | Tasse |
|---|---|---:|---:|---:|---:|---:|
| 2025-04-01 | Acquisto | 3.983 | 25.11 | 100 | 0 | 0 |
```

### Quotazioni

```markdown
---
isin: "LU0102238655"
---

| Data | Prezzo |
|---|---|
| 2026-10-15 | 25.112345 |

## Note manuali
Questo contenuto viene conservato durante l'aggiornamento.
```

### Conto

```yaml
---
displayName: NEF - Famiglia
strumento:
  - "[[Strumenti/LU0102238655]]"
---
```

## Comandi

- **Quotazioni: Crea pagine e scarica storico** — crea pagine nuove e richiede cinque anni da Yahoo.
- **Quotazioni: Aggiorna** — aggiorna l'ultimo mese delle pagine automatiche.

## Implementazione Space Lua

I blocchi seguenti costituiscono l'intera implementazione. Non rimuoverli se si sostituisce la pagina `Library/MG/investimenti`.

### Strumenti e collegamenti

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

--- Tabella id -> displayName da un elenco di strumenti.
function investimenti.nomi(strumenti)
  local out = {}
  for _, s in ipairs(strumenti) do out[s.id] = s.nome end
  return out
end

--- Strumenti: pagine sotto Strumenti/, ordinate per id.
function investimenti.strumenti()
  local out = {}
  for _, p in ipairs(query[[
    from p = index.subPages("Strumenti")
    select {name = p.name, nome = p.displayName, isin = p.isin, ticker = p.ticker,
      valuta = p.valuta, tipo = p.tipo, rischio = p.KID,
      descrizione = p.description, tags = p.tags,
      stopLoss = p.stopLoss, takeProfit = p.takeProfit, trailing = p.trailing, segnali = p.segnali}
  ]]) do
    table.insert(out, {id = string.sub(p.name, 11),  -- "Strumenti/" = 10 caratteri
      nome = str(p.nome) or p.name, isin = str(p.isin), ticker = str(p.ticker),
      valuta = str(p.valuta) or "EUR", tipo = str(p.tipo),
      rischio = type(p.rischio) == "number" and p.rischio or nil,  -- p.rischio = attributo KID
      descrizione = str(p.descrizione), tags = p.tags,
      stopLoss = tonumber(p.stopLoss), takeProfit = tonumber(p.takeProfit),
      trailing = tonumber(p.trailing), segnali = p.segnali ~= false})
  end
  table.sort(out, function(a, b) return a.id < b.id end)
  return out
end

--- Soglie percentuali di uno strumento: stop loss, take profit, trailing stop.
--- Frontmatter dello strumento, poi config (investimenti.stopLoss/takeProfit/trailing), poi 10/25/8.
function investimenti.soglie(s)
  return s.stopLoss or config.get("investimenti.stopLoss") or 10,
    s.takeProfit or config.get("investimenti.takeProfit") or 25,
    s.trailing or config.get("investimenti.trailing") or 8
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

### Accesso Yahoo Finance

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

### Creazione e aggiornamento conservativo

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

### Verifica in sola lettura

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

### Comando: crea quotazioni

```space-lua
command.define {
  name = "Quotazioni: Crea pagine e scarica storico",
  run = function()
    local ids, pagine = quotazioni.crea()
    quotazioni.aggiorna("5y", ids, pagine)
  end
}
```

### Comando: aggiorna quotazioni

```space-lua
command.define {
  name = "Quotazioni: Aggiorna",
  run = function()
    quotazioni.aggiorna("1mo")
  end
}
```

### v0.17 — Conti, posizioni e portafogli

La pagina dello **strumento** conserva i movimenti, ma uno stesso ISIN può avere più pagine strumento e più intestatari. La chiave contabile è **conto + id strumento**, non l'ISIN. Per le pagine dell'archivio di esempio il conto è inferito dai riferimenti nei frontmatter `Conti/...`: le pagine con tag `Portafoglio` sono aggregatori e non duplicano i movimenti. In mancanza di un conto fisico, una pagina `Portafoglio` può essere il contenitore esclusivo (es. Samuele).

Nelle tabelle dei movimenti è facoltativa la colonna `Conto`, con nome esatto della pagina, es. `Conti/NEF - Famiglia` o il solo nome `NEF - Famiglia`. È necessaria per distinguere acquisti del medesimo strumento presso conti fisici diversi. Se esistono più conti fisici candidati e manca `Conto`, la riga non è assegnata a nessun conto specifico: resta visibile nell'aggregato globale ed è segnalata da `investimenti.verificaConti()`. Non vengono ripartite quote in modo arbitrario.

`investimenti.posizioni(pagina)` produce righe `{conto, id, quote, acquistato, incassato, qAcq, primo}`; `investimenti.verificaConti()` segnala i riferimenti e le assegnazioni ambigue. La somma per portafoglio utilizza gli id di strumenti associati, senza sommare i totali dei conti figli. Restano invariati i frontmatter e le tabelle esistenti.

### Movimenti e ultimi prezzi

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
function investimenti.movimentiDati(ids, pagina)
  local per, elenco, ammessi = {}, {}, nil
  if ids then ammessi = {}; for _, id in ipairs(ids) do ammessi[id] = true end end
  local conto = type(pagina) == "string" and string.startsWith(pagina, "Conti/") and pagina or nil
  -- Il portafoglio include le sole posizioni dei propri strumenti, anche in conti fisici diversi.
  local portafoglio = false
  if conto then
    for _, p in ipairs(query[[
      from p = index.subPages("Conti")
      select {name = p.name, tags = p.tags}
    ]]) do
      if p.name == conto then
        if type(p.tags) == "string" then portafoglio = string.find(p.tags, "Portafoglio", 1, true) ~= nil end
        if type(p.tags) == "table" then
          for _, t in ipairs(p.tags) do if t == "Portafoglio" then portafoglio = true end end
        end
      end
    end
  end
  for _, r in ipairs(investimenti.ledger(ids)) do
    if (not ammessi or ammessi[r.id]) and
      (not conto or (portafoglio and r.conto ~= nil) or r.conto == conto) then
      local m = per[r.id]
      if not m then
        m = {id = r.id, quote = 0, acquistato = 0, incassato = 0, qAcq = 0}
        per[r.id] = m
        table.insert(elenco, r.id)
      end
      local q, v = tonumber(r.quote), tonumber(r.valore)
      if r.tipo == "Acquisto" and q and v then
        m.quote, m.acquistato, m.qAcq = m.quote + q, m.acquistato + v, m.qAcq + q
        if type(r.data) == "string" and (not m.primo or r.data < m.primo) then m.primo = r.data end
      elseif r.tipo == "Vendita" and q and v then
        m.quote, m.incassato = m.quote - q, m.incassato + v
      elseif r.tipo == "Dividendo" and v then m.incassato = m.incassato + v end
    end
  end
  table.sort(elenco)
  local out = {}
  for _, id in ipairs(elenco) do table.insert(out, per[id]) end
  return out
end

--- Posizioni contabili distinte per conto. Con pagina valorizza il solo ambito richiesto.
function investimenti.posizioni(pagina)
  local ids = investimenti.elenco(pagina)
  local per, chiavi = {}, {}
  local portafoglio = false
  if type(pagina) == "string" and string.startsWith(pagina, "Conti/") then
    for _, p in ipairs(query[[
      from p = index.subPages("Conti")
      select {name = p.name, tags = p.tags}
    ]]) do
      if p.name == pagina then
        if type(p.tags) == "string" and string.find(p.tags, "Portafoglio", 1, true) then portafoglio = true end
        if type(p.tags) == "table" then
          for _, t in ipairs(p.tags) do if t == "Portafoglio" then portafoglio = true end end
        end
      end
    end
  end
  for _, r in ipairs(investimenti.ledger(ids)) do
    local include = not pagina or not string.startsWith(pagina, "Conti/")
    if not include then
      for _, id in ipairs(ids or {}) do
        if id == r.id and r.conto then
          include = pagina == r.conto
          if not include and portafoglio then include = true end
          break
        end
      end
    end
    if include then
      local conto = r.conto or "NON_ASSEGNATO"
      local key = conto .. "|" .. r.id
      if not per[key] then
        per[key] = {conto = conto, id = r.id, quote = 0, acquistato = 0,
          incassato = 0, qAcq = 0}
        table.insert(chiavi, key)
      end
      local m, q, v = per[key], tonumber(r.quote), tonumber(r.valore)
      if r.tipo == "Acquisto" and q and v then
        m.quote, m.acquistato, m.qAcq = m.quote + q, m.acquistato + v, m.qAcq + q
        if type(r.data) == "string" and (not m.primo or r.data < m.primo) then m.primo = r.data end
      elseif r.tipo == "Vendita" and q and v then m.quote = m.quote - q; m.incassato = m.incassato + v
      elseif r.tipo == "Dividendo" and v then m.incassato = m.incassato + v end
    end
  end
  table.sort(chiavi)
  local out = {}
  for _, key in ipairs(chiavi) do table.insert(out, per[key]) end
  return out
end

--- Controlli non distruttivi sulle assegnazioni e sui valori dei movimenti.
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
function investimenti.ultimiPrezzi(strumenti, per)
  strumenti = strumenti or investimenti.strumenti()
  per = per or investimenti.quotazioniPerStrumento(strumenti)
  local out = {}
  for _, s in ipairs(strumenti) do
    local ultimo
    for _, r in ipairs(per[s.id] or {}) do
      if not ultimo or r[1] > ultimo[1] then ultimo = r end
    end
    if ultimo then table.insert(out, {id = s.id, data = ultimo[1], prezzo = ultimo[2], valuta = s.valuta}) end
  end
  return out
end
```

### Valorizzazione numerica

```space-lua
investimenti = investimenti or {}

--- Righe {id, nome, quote, prezzo, data, acquistato, incassato, valore, pl, pct, segnale}, totale {a, i, v, pl}
--- e numero di strumenti senza prezzo. Prezzi in USD convertiti con lo strumento EURUSD.
function investimenti.valorizzazioneDati(ids, pagina)
  local strumenti = investimenti.strumenti()
  local nome = investimenti.nomi(strumenti)

  local per = investimenti.quotazioniPerStrumento(strumenti)
  local px, fx = {}, nil
  for _, p in ipairs(investimenti.ultimiPrezzi(strumenti, per)) do
    px[p.id] = p
    if p.id == "EURUSD" then fx = p.prezzo end
  end

  local strumento = {}
  for _, s in ipairs(strumenti) do strumento[s.id] = s end

  local righe, tot, nd = {}, {a = 0, i = 0, v = 0, pl = 0}, 0
  for _, m in ipairs(investimenti.movimentiDati(ids, pagina)) do
    local p = px[m.id]
    local prezzo = p and p.prezzo
    if prezzo and p.valuta == "USD" then prezzo = fx and prezzo / fx end
    local valore
    if math.abs(m.quote) < 1e-9 then valore = 0       -- posizione chiusa
    elseif prezzo then valore = m.quote * prezzo end  -- senza prezzo: n.d.
    local pl, pct
    if valore then
      pl = valore + m.incassato - m.acquistato
      if m.acquistato > 0 then pct = pl / m.acquistato * 100 end
      tot.a, tot.i, tot.v = tot.a + m.acquistato, tot.i + m.incassato, tot.v + valore
      tot.pl = tot.pl + pl
    else
      nd = nd + 1
    end
    -- segnale: stop loss e take profit rispetto al prezzo medio di carico (in EUR); trailing
    -- rispetto al massimo dal primo acquisto. Solo per posizioni aperte con acquisti.
    local segnale
    local s = strumento[m.id]
    if prezzo and s and s.segnali and m.quote > 1e-9 and m.qAcq > 0 and m.acquistato > 0 then
      local sl, tp, tr = investimenti.soglie(s)
      local variazione = (prezzo / (m.acquistato / m.qAcq) - 1) * 100
      local massimo
      for _, r in ipairs(per[m.id] or {}) do
        if r[1] >= (m.primo or "") and (not massimo or r[2] > massimo) then massimo = r[2] end
      end
      if variazione <= -sl then segnale = "🔴 Stop loss"
      elseif massimo and p.prezzo <= massimo * (1 - tr / 100) then segnale = "🟠 Trailing"
      elseif variazione >= tp then segnale = "🟢 Take profit" end
    end
    table.insert(righe, {id = m.id, nome = nome[m.id] or m.id, quote = m.quote, prezzo = prezzo,
      data = p and p.data, acquistato = m.acquistato, incassato = m.incassato,
      valore = valore, pl = pl, pct = pct, segnale = segnale})
  end
  return righe, tot, nd
end
```

### Tabelle e viste

```space-lua
investimenti = investimenti or {}

-- Separa le migliaia con uno spazio nella parte intera ("15000.00" -> "15 000.00").
local function migliaia(t)
  local segno, int, resto = string.match(t, "^(-?)(%d+)(.*)$")
  if not int then return t end
  local n = 1
  while n > 0 do int, n = string.gsub(int, "^(%d+)(%d%d%d)", "%1 %2") end
  return segno .. int .. resto
end

-- Due decimali con migliaia separate, "n.d." se manca.
local function f2(x) return x and migliaia(string.format("%.2f", x)) or "n.d." end

-- Quantità: fino a 8 decimali, senza zeri finali.
local function fq(x)
  local s = string.format("%.8f", x)
  s = string.gsub(s, "0+$", "")
  s = string.gsub(s, "%.$", "")
  return migliaia(s)
end

-- Data ISO come DD.MM.YYYY con date.format (libreria DateFormat); senza, resta ISO.
local function data(d)
  if type(d) ~= "string" then return "n.d." end
  if type(date) == "table" and type(date.format) == "function" then
    return date.format(d, "DD.MM.YYYY")
  end
  return d
end



-- Stile della tabella nel widget isolato: sotto 600 px ogni riga diventa una scheda
-- (etichetta e valore per ogni campo); .v = elenco verticale, senza modalità scheda.
local CSS_TABELLA = [[
body{margin:0;font:18px system-ui,sans-serif}
td{white-space:nowrap}
@media (prefers-color-scheme: dark){body{color:#ddd}}
table{border-collapse:collapse;width:100%}
th,td{padding:2px 10px}
th{border-bottom:1px solid currentColor}
tr.t td{font-weight:bold;border-top:1px solid currentColor}
a{cursor:pointer;text-decoration:underline}
.l{text-align:left}.r{text-align:right}.c{text-align:center}
table.v td:first-child{font-weight:bold}
@media (max-width:600px){
 table.m thead{display:none}
 table.m tr{display:block;padding:6px 0;border-bottom:1px solid rgba(128,128,128,.4)}
 table.m td{display:flex;justify-content:space-between;gap:12px;text-align:right}
 table.m td::before{content:attr(data-l);font-weight:bold;text-align:left}
 table.m td:first-child{font-weight:bold}
 table.m td:first-child::before{content:none}
}
]]

-- Script della tabella: DATI = {vert, cols: [{t, a}], rows: [{tot, c: [testo | {n, p}]}]}.
local JS_TABELLA = [[
const t = document.createElement("table");
t.className = DATI.vert ? "v" : "m";
if (!DATI.vert) {
  const tr = document.createElement("tr");
  DATI.cols.forEach(function (c) {
    const th = document.createElement("th");
    th.className = c.a;
    th.textContent = c.t;
    tr.appendChild(th);
  });
  const th = document.createElement("thead");
  th.appendChild(tr);
  t.appendChild(th);
}
const tb = document.createElement("tbody");
DATI.rows.forEach(function (r) {
  const tr = document.createElement("tr");
  if (r.tot) tr.className = "t";
  r.c.forEach(function (v, i) {
    const td = document.createElement("td");
    td.className = DATI.cols[i].a;
    td.setAttribute("data-l", DATI.cols[i].t);
    if (typeof v === "object") {
      const a = document.createElement("a");
      a.textContent = v.n;
      a.onclick = function () { syscall("editor.navigate", v.p); };
      td.appendChild(a);
    } else {
      td.textContent = v;
    }
    tr.appendChild(td);
  });
  tb.appendChild(tr);
});
t.appendChild(tb);
document.getElementById("t").appendChild(t);
]]

-- Tabella in un widget isolato. colonne: {titolo, chiave, allinea (left|right|center)};
-- righe: {chiave = testo | {id, nome} | {pagina, nome}, totale = bool}; verticale = elenco
-- campo/valore senza intestazione né modalità scheda. Il contenuto Markdown serve al pulsante Copia.
local function tabella(colonne, righe, verticale)
  local sigla = {left = "l", right = "r", center = "c"}
  local cols, rows, md = {}, {}, {}
  for _, c in ipairs(colonne) do
    table.insert(cols, '{"t":' .. investimenti.js(c.titolo) .. ',"a":"' .. (sigla[c.allinea] or "l") .. '"}')
  end
  local testa, riga = {}, {}
  for _, c in ipairs(colonne) do table.insert(testa, c.titolo); table.insert(riga, "---") end
  table.insert(md, "| " .. table.concat(testa, " | ") .. " |")
  table.insert(md, "|" .. table.concat(riga, "|") .. "|")
  for _, r in ipairs(righe) do
    local celle, testo = {}, {}
    for _, c in ipairs(colonne) do
      local v = r[c.chiave]
      if type(v) == "table" then
        table.insert(celle, '{"n":' .. investimenti.js(v.nome) .. ',"p":' .. investimenti.js(v.pagina or ("Strumenti/" .. v.id)) .. '}')
        table.insert(testo, v.nome)
      else
        table.insert(celle, investimenti.js(v or ""))
        table.insert(testo, v or "")
      end
    end
    table.insert(rows, '{"tot":' .. (r.totale and "true" or "false") .. ',"c":[' .. table.concat(celle, ",") .. "]}")
    table.insert(md, "| " .. table.concat(testo, " | ") .. " |")
  end
  local dati = "const DATI = {vert: " .. (verticale and "true" or "false") .. ', cols: [' ..
    table.concat(cols, ",") .. "], rows: [" .. table.concat(rows, ",") .. "]};\n"
  return widget.sandbox {
    html = "<style>" .. CSS_TABELLA .. '</style><div id="t"></div>',
    script = dati .. JS_TABELLA,
    markdown = table.concat(md, "\n"),
  }
end

--- Movimenti aggregati per strumento, come tabella. pagina: vedi investimenti.elenco.
function investimenti.movimenti(pagina)
  local ids = investimenti.elenco(pagina)
  if ids and #ids == 0 then return widget.markdownBlock("_Nessuno strumento._") end
  local nome = investimenti.nomi(investimenti.strumenti())
  local righe = {}
  for _, m in ipairs(investimenti.movimentiDati(ids, pagina)) do
    table.insert(righe, {id = {id = m.id, nome = nome[m.id] or m.id}, quote = fq(m.quote),
      acquistato = f2(m.acquistato), incassato = f2(m.incassato)})
  end
  return tabella({
    {titolo = "id", chiave = "id", allinea = "left"},
    {titolo = "quote", chiave = "quote", allinea = "right"},
    {titolo = "acquistato", chiave = "acquistato", allinea = "right"},
    {titolo = "incassato", chiave = "incassato", allinea = "right"},
  }, righe)
end

--- Valorizzazione in EUR, come tabella. P/L = valore + incassato - acquistato,
--- al lordo di imposte e commissioni di conto. pagina: vedi investimenti.elenco.
function investimenti.valorizzazione(pagina)
  local ids = investimenti.elenco(pagina)
  if ids and #ids == 0 then return widget.markdownBlock("_Nessuno strumento._") end
  local dati, tot, nd = investimenti.valorizzazioneDati(ids, pagina)
  local righe = {}
  for _, d in ipairs(dati) do
    table.insert(righe, {strumento = {id = d.id, nome = d.nome}, quote = fq(d.quote),
      prezzo_eur = f2(d.prezzo), data_prezzo = d.prezzo and data(d.data) or "n.d.",
      acquistato = f2(d.acquistato), incassato = f2(d.incassato), valore = f2(d.valore),
      pl = f2(d.pl), pct = d.pct and (f2(d.pct) .. " %") or "n.d.", segnale = d.segnale or ""})
  end
  if #dati > 1 then table.insert(righe, {totale = true,
    strumento = "TOTALE" .. (nd > 0 and " (esclusi " .. nd .. " senza prezzo)" or ""),
    acquistato = f2(tot.a), incassato = f2(tot.i), valore = f2(tot.v), pl = f2(tot.pl)}) end
  return tabella({
    {titolo = "strumento", chiave = "strumento", allinea = "left"},
    {titolo = "quote", chiave = "quote", allinea = "right"},
    {titolo = "prezzo_eur", chiave = "prezzo_eur", allinea = "right"},
    {titolo = "data_prezzo", chiave = "data_prezzo", allinea = "center"},
    {titolo = "acquistato", chiave = "acquistato", allinea = "right"},
    {titolo = "incassato", chiave = "incassato", allinea = "right"},
    {titolo = "valore", chiave = "valore", allinea = "right"},
    {titolo = "pl", chiave = "pl", allinea = "right"},
    {titolo = "%", chiave = "pct", allinea = "center"},
    {titolo = "segnale", chiave = "segnale", allinea = "left"},
  }, righe)
end

--- Valorizzazione compatta: investimento, acquistato, attuale, P/L, %, performance 1M/YTD/1A.
--- pagina: vedi investimenti.elenco.
function investimenti.valore(pagina)
  local ids = investimenti.elenco(pagina)
  if ids and #ids == 0 then return widget.markdownBlock("_Nessuno strumento._") end
  local dati, tot, nd = investimenti.valorizzazioneDati(ids, pagina)
  local perf = investimenti.performance(ids)
  local function fp(x) return x and string.format("%+.2f %%", x) or "n.d." end
  local righe = {}
  for _, d in ipairs(dati) do
    local f = perf[d.id] or {}
    table.insert(righe, {nome = {id = d.id, nome = d.nome}, acquistato = f2(d.acquistato),
      attuale = f2(d.valore), pl = f2(d.pl), pct = d.pct and (f2(d.pct) .. " %") or "n.d.",
      m1 = fp(f.m1), ytd = fp(f.ytd), a1 = fp(f.a1), segnale = d.segnale or ""})
  end
  if #dati > 1 then table.insert(righe, {totale = true,
    nome = "TOTALE" .. (nd > 0 and " (esclusi " .. nd .. " senza prezzo)" or ""),
    acquistato = f2(tot.a), attuale = f2(tot.v), pl = f2(tot.pl)}) end
  return tabella({
    {titolo = "Investimento", chiave = "nome", allinea = "left"},
    {titolo = "Acquistato", chiave = "acquistato", allinea = "right"},
    {titolo = "Attuale", chiave = "attuale", allinea = "right"},
    {titolo = "P/L", chiave = "pl", allinea = "right"},
    {titolo = "%", chiave = "pct", allinea = "center"},
    {titolo = "1M", chiave = "m1", allinea = "right"},
    {titolo = "YTD", chiave = "ytd", allinea = "right"},
    {titolo = "1A", chiave = "a1", allinea = "right"},
    {titolo = "Segnale", chiave = "segnale", allinea = "left"},
  }, righe)
end

-- tags (stringa o lista) -> lista di stringhe
local function listaTags(t)
  if type(t) == "string" then return {t} end
  local out = {}
  if type(t) == "table" then
    for _, x in ipairs(t) do if type(x) == "string" then table.insert(out, x) end end
  end
  return out
end

--- Scheda di uno strumento dal frontmatter, su una riga: tipo, valuta, rischio e link ai
--- conti/portafogli che lo contengono (attributo `strumento` delle pagine Conti/...).
function investimenti.scheda(pagina)
  pagina = pagina or editor.getCurrentPage()
  if type(pagina) ~= "string" or not string.startsWith(pagina, "Strumenti/") then return nil end
  local id = string.sub(pagina, 11)
  for _, s in ipairs(investimenti.strumenti()) do
    if s.id == id then
      local testo = (s.tipo or "n.d.") .. " · " .. s.valuta ..
        " · Rischio: " .. investimenti.barraRischio(s.rischio)
      local link = {}
      for _, c in ipairs(query[[
        from p = index.subPages("Conti")
        select {name = p.name, nome = p.displayName}
      ]]) do
        for _, x in ipairs(investimenti.elenco(c.name) or {}) do
          if x == id then
            table.insert(link, "[[" .. c.name .. "|" .. (type(c.nome) == "string" and c.nome or c.name) .. "]]")
            break
          end
        end
      end
      if #link > 0 then testo = testo .. " · " .. table.concat(link, ", ") end
      return widget.markdownBlock(testo)
    end
  end
end

--- Elenco di tutti gli strumenti con tipo, valuta, ultimo prezzo, data e rischio.
function investimenti.elencoStrumenti()
  local strumenti = investimenti.strumenti()
  local px = {}
  for _, p in ipairs(investimenti.ultimiPrezzi(strumenti)) do px[p.id] = p end
  local righe = {}
  for _, s in ipairs(strumenti) do
    local p = px[s.id]
    table.insert(righe, {nome = {id = s.id, nome = s.nome}, tipo = s.tipo or "",
      valuta = s.valuta, prezzo = p and string.format("%.4f", p.prezzo) or "n.d.",
      data = p and p.data or "n.d.", rischio = investimenti.barraRischio(s.rischio)})
  end
  return tabella({
    {titolo = "Strumento", chiave = "nome", allinea = "left"},
    {titolo = "Tipo", chiave = "tipo", allinea = "left"},
    {titolo = "Valuta", chiave = "valuta", allinea = "center"},
    {titolo = "Ultimo prezzo", chiave = "prezzo", allinea = "right"},
    {titolo = "Data", chiave = "data", allinea = "center"},
    {titolo = "Rischio", chiave = "rischio", allinea = "left"},
  }, righe)
end

--- Elenco di conti e portafogli (sottopagine di Conti/): tipo dal tag, numero di strumenti,
--- acquistato, attuale, P/L e %. La valorizzazione si calcola una sola volta per tutti.
function investimenti.elencoConti()
  local righe = {}
  for _, p in ipairs(query[[
    from p = index.subPages("Conti")
    select {name = p.name, nome = p.displayName, tags = p.tags}
  ]]) do
    local ids = investimenti.elenco(p.name) or {}
    local _, tot = investimenti.valorizzazioneDati(ids, p.name)
    local a, i, v = tot.a, tot.i, tot.v
    local pl = v + i - a
    local tipo = "Conto"
    for _, t in ipairs(listaTags(p.tags)) do if t == "Portafoglio" then tipo = "Portafoglio" end end
    table.insert(righe, {nome = {pagina = p.name, nome = type(p.nome) == "string" and p.nome or p.name},
      tipo = tipo, n = tostring(#ids), acquistato = f2(a), attuale = f2(v), pl = f2(pl),
      pct = a > 0 and (f2(pl / a * 100) .. " %") or "n.d."})
  end
  table.sort(righe, function(x, y) return x.nome.nome < y.nome.nome end)
  return tabella({
    {titolo = "Conto/Portafoglio", chiave = "nome", allinea = "left"},
    {titolo = "Tipo", chiave = "tipo", allinea = "left"},
    {titolo = "Strumenti", chiave = "n", allinea = "center"},
    {titolo = "Acquistato", chiave = "acquistato", allinea = "right"},
    {titolo = "Attuale", chiave = "attuale", allinea = "right"},
    {titolo = "P/L", chiave = "pl", allinea = "right"},
    {titolo = "%", chiave = "pct", allinea = "center"},
  }, righe)
end
```

### Grafici e allocazione

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

-- Script del grafico lineare. DATI = {titolo, serie, acquisti, pmc, stop, tp, trailing}:
-- serie e acquisti sono liste di [data, prezzo]; pmc/stop/tp sono null senza acquisti.
-- Gli indicatori (finestre in osservazioni, cioè in quotazioni) sono calcolati sull'intera serie
-- e poi ritagliati sul periodo scelto, così all'inizio del periodo sono già completi.
local JS_LINEA = [[
const RANGE = [{n:"7G",d:7},{n:"1M",d:30},{n:"3M",d:91},{n:"6M",d:182},{n:"YTD",d:-1},{n:"1A",d:365},{n:"5A",d:1826},{n:"Tutto",d:0}];
const dark = window.matchMedia && window.matchMedia("(prefers-color-scheme: dark)").matches;
const txt = dark ? "#ddd" : "#333", grid = "rgba(128,128,128,.25)";
const S = DATI.serie, P = S.map(function (p) { return p[1]; });
const ultimo = S[S.length - 1][0];
const primoAcq = DATI.acquisti.length ? DATI.acquisti[0][0] : null;
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
// trailing stop: massimo dal primo acquisto meno la percentuale; non scende mai
let massimo = null;
const TR = S.map(function (p) {
  if (primoAcq === null || p[0] < primoAcq) return null;
  massimo = massimo === null ? p[1] : Math.max(massimo, p[1]);
  return massimo * (1 - DATI.trailing / 100);
});
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
  if (attivi.sl) {
    ds.push(linea("Prezzo medio di carico", fisso(DATI.pmc), "#7d8597", [2, 3]));
    ds.push(linea("Stop loss −" + DATI.stopPct + "%", fisso(DATI.stop), "#e4572e", [6, 4]));
    ds.push(linea("Take profit +" + DATI.tpPct + "%", fisso(DATI.tp), "#2a9d4a", [6, 4]));
  }
  if (attivi.tr) ds.push(linea("Trailing stop −" + DATI.trailing + "%", scegli(TR), "#c1666b"));
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
if (DATI.pmc !== null) {
  indicatori.push({k: "sl", n: "Stop · TP"});
  indicatori.push({k: "tr", n: "Trailing"});
}
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

  -- prezzo medio di carico (pesato sulle quote) e livelli di stop loss / take profit
  local pmc, stop, tp
  local sl, tpp, tr = 10, 25, 8
  for _, s in ipairs(investimenti.strumenti()) do
    if s.id == id then sl, tpp, tr = investimenti.soglie(s) end
  end
  local q, v = 0, 0
  for _, a in ipairs(acq) do q, v = q + a[3], v + a[3] * a[2] end
  if q > 0 then pmc = v / q; stop = pmc * (1 - sl / 100); tp = pmc * (1 + tpp / 100) end

  local function coppie(lista)
    local p = {}
    for _, c in ipairs(lista) do table.insert(p, "[" .. investimenti.js(c[1]) .. "," .. c[2] .. "]") end
    return "[" .. table.concat(p, ",") .. "]"
  end
  local nome = id or quot
  for _, s in ipairs(investimenti.strumenti()) do if s.id == id then nome = s.nome end end
  local dati = "const DATI = {titolo: " .. investimenti.js(nome) .. ", serie: " .. coppie(serie) ..
    ", acquisti: " .. coppie(acq) .. ", pmc: " .. (pmc or "null") .. ", stop: " .. (stop or "null") ..
    ", tp: " .. (tp or "null") .. ", stopPct: " .. sl .. ", tpPct: " .. tpp .. ", trailing: " .. tr .. "};\n"
  return widget.sandbox {
    html = '<div id="b"></div><div id="b2"></div><div style="position:relative;height:300px">' ..
      '<canvas id="c"></canvas></div><div id="rw" style="display:none;position:relative;height:140px;' ..
      'margin-top:6px"><canvas id="r"></canvas></div>',
    script = 'loadJsByUrl("' .. investimenti.chartjs .. '").then(function () {\n' .. dati ..
      JS_LINEA .. "\n});",
    markdown = "Grafico " .. nome,
  }
end

-- Script delle ciambelle: GRAFICI = lista di {id, titolo, etichette, valori, colori}.
local JS_TORTA = [[
const dark = window.matchMedia && window.matchMedia("(prefers-color-scheme: dark)").matches;
const col = dark ? "#ddd" : "#333";
GRAFICI.forEach(function (g) {
  const tot = g.valori.reduce(function (a, b) { return a + b; }, 0);
  new Chart(document.getElementById(g.id), {type: "doughnut",
    data: {labels: g.etichette, datasets: [{data: g.valori, backgroundColor: g.colori,
      borderColor: dark ? "#1e1e1e" : "#fff"}]},
    options: {responsive: true, maintainAspectRatio: false, animation: false,
      plugins: {legend: {position: "bottom", labels: {color: col}},
        title: {display: true, text: g.titolo, color: col},
        tooltip: {callbacks: {label: function (c) {
          return c.label + ": " + c.parsed.toFixed(2) + " € (" + (c.parsed / tot * 100).toFixed(1) + " %)";
        }}}}}});
});
]]

local COLORI = {"#2f6fdb", "#d9822b", "#2a9d8f", "#9b5de5", "#e4572e", "#76b041", "#f2c14e", "#7d8597", "#c1666b", "#4ecdc4"}
local COLORI_RISCHIO = {"#2a9d4a", "#76b041", "#c8c63a", "#f2c14e", "#e8923a", "#e4572e", "#b3202a"}

-- Un solo widget con una o più ciambelle: griglia CSS auto-fit, quindi affiancate su schermo
-- largo e impilate su schermo stretto. specs: lista di {titolo, etichette, valori, colori}.
local function torte(specs)
  local celle, grafici = {}, {}
  for i, sp in ipairs(specs) do
    local e, v, c = {}, {}, {}
    for k = 1, #sp.etichette do
      table.insert(e, investimenti.js(sp.etichette[k]))
      table.insert(v, string.format("%.2f", sp.valori[k]))
      table.insert(c, investimenti.js(sp.colori[k]))
    end
    table.insert(grafici, "{id: \"g" .. i .. "\", titolo: " .. investimenti.js(sp.titolo) .. ", etichette: [" ..
      table.concat(e, ",") .. "], valori: [" .. table.concat(v, ",") .. "], colori: [" ..
      table.concat(c, ",") .. "]}")
    table.insert(celle, '<div style="position:relative;height:280px"><canvas id="g' .. i .. '"></canvas></div>')
  end
  local titoli = {}
  for _, sp in ipairs(specs) do table.insert(titoli, sp.titolo) end
  return widget.sandbox {
    html = '<div style="display:grid;grid-template-columns:repeat(auto-fit,minmax(280px,1fr));gap:12px">' ..
      table.concat(celle) .. "</div>",
    script = 'loadJsByUrl("' .. investimenti.chartjs .. '").then(function () {\nconst GRAFICI = [' ..
      table.concat(grafici, ",") .. "];\n" .. JS_TORTA .. "\n});",
    markdown = table.concat(titoli, " / "),
  }
end

-- Dati della ciambella per strumento, o nil se non c'è nulla da mostrare.
local function specValore(ids, pagina)
  local e, v, c = {}, {}, {}
  for _, d in ipairs(investimenti.valorizzazioneDati(ids, pagina)) do
    if d.valore and d.valore > 0 then
      table.insert(e, d.nome)
      table.insert(v, d.valore)
      table.insert(c, COLORI[(#e - 1) % #COLORI + 1])
    end
  end
  if #v > 0 then return {titolo = "Valore per strumento (EUR)", etichette = e, valori = v, colori = c} end
end

-- Dati della ciambella per classe di rischio (attributo KID), o nil.
local function specRischio(ids, pagina)
  local rischio = {}
  for _, s in ipairs(investimenti.strumenti()) do rischio[s.id] = s.rischio end
  local per = {}   -- classe (0 = non indicata) -> valore
  for _, d in ipairs(investimenti.valorizzazioneDati(ids, pagina)) do
    if d.valore and d.valore > 0 then
      local k = rischio[d.id] or 0
      per[k] = (per[k] or 0) + d.valore
    end
  end
  local e, v, c = {}, {}, {}
  for k = 0, 7 do
    if per[k] then
      table.insert(e, k == 0 and "n.d." or ("Rischio " .. k))
      table.insert(v, per[k])
      table.insert(c, k == 0 and "#7d8597" or COLORI_RISCHIO[k])
    end
  end
  if #v > 0 then return {titolo = "Valore per classe di rischio (KID, 1-7)", etichette = e, valori = v, colori = c} end
end

--- Ciambella della ripartizione del valore attuale per strumento. pagina: vedi investimenti.elenco.
function investimenti.torta(pagina)
  local ids = investimenti.elenco(pagina)
  if ids and #ids == 0 then return widget.markdownBlock("_Nessuno strumento._") end
  local sp = specValore(ids, pagina)
  if not sp then return widget.markdownBlock("_Nessun valore da mostrare._") end
  return torte({sp})
end

--- Ciambella del valore attuale per classe di rischio (attributo KID 1-7 degli strumenti).
function investimenti.tortaRischio(pagina)
  local ids = investimenti.elenco(pagina)
  if ids and #ids == 0 then return widget.markdownBlock("_Nessuno strumento._") end
  local sp = specRischio(ids, pagina)
  if not sp then return widget.markdownBlock("_Nessun valore da mostrare._") end
  return torte({sp})
end

--- Le due ciambelle (valore per strumento, valore per rischio) in un solo widget:
--- affiancate su schermo largo, impilate su schermo stretto. pagina: vedi investimenti.elenco.
function investimenti.allocazione(pagina)
  local ids = investimenti.elenco(pagina)
  if ids and #ids == 0 then return widget.markdownBlock("_Nessuno strumento._") end
  local specs = {}
  for _, sp in ipairs({specValore(ids, pagina) or false, specRischio(ids, pagina) or false}) do
    if sp then table.insert(specs, sp) end
  end
  if #specs == 0 then return widget.markdownBlock("_Nessun valore da mostrare._") end
  return torte(specs)
end

--- Barra di rischio in emoji: n quadrati colorati su 7 (verde, giallo, arancio, rosso) e n/7.
function investimenti.barraRischio(n)
  if type(n) ~= "number" then return "n.d." end
  n = math.max(0, math.min(7, math.floor(n + 0.5)))
  local col = {"🟩", "🟩", "🟨", "🟨", "🟧", "🟧", "🟥"}
  local out = {}
  for i = 1, 7 do table.insert(out, i <= n and col[i] or "⬜") end
  return table.concat(out) .. " " .. n .. "/7"
end

--- Rischio come testo: barra emoji dello strumento, o media ponderata sul valore per conti/portafogli.
function investimenti.rischio(pagina)
  local ids = investimenti.elenco(pagina)
  if not ids or #ids == 0 then return "n.d." end
  local rischio = {}
  for _, s in ipairs(investimenti.strumenti()) do rischio[s.id] = s.rischio end
  if #ids == 1 then return investimenti.barraRischio(rischio[ids[1]]) end
  local somma, peso = 0, 0
  for _, d in ipairs(investimenti.valorizzazioneDati(ids, pagina)) do
    local r = rischio[d.id]
    if r and d.valore and d.valore > 0 then somma, peso = somma + r * d.valore, peso + d.valore end
  end
  if peso == 0 then return "n.d." end
  return investimenti.barraRischio(somma / peso)
end
```

### Performance e widget automatici

```space-lua
investimenti = investimenti or {}

-- Ultimo punto {data, prezzo} con data <= limite (serie ordinata), o nil.
local function finoA(serie, limite)
  local best
  for _, p in ipairs(serie) do
    if p[1] > limite then break end
    best = p
  end
  return best
end

-- Data ISO meno n giorni.
local function menoGiorni(d, n)
  local y, m, g = string.match(d, "^(%d+)-(%d+)-(%d+)")
  local t = os.time({year = tonumber(y), month = tonumber(m), day = tonumber(g), hour = 12})
  return os.date("!%Y-%m-%d", t - n * 86400)
end

--- Variazione % a 1M, YTD e 1A per strumento: tabella id -> {m1, ytd, a1} (nil se manca lo storico).
--- ids: lista di id; nil = tutti gli strumenti con quotazioni.
function investimenti.performance(ids)
  local lista  -- ids richiesti più EURUSD (serve per le conversioni)
  if ids then
    lista = {"EURUSD"}
    for _, id in ipairs(ids) do table.insert(lista, id) end
  end
  local strumenti = investimenti.strumenti()
  local per = investimenti.quotazioniPerStrumento(strumenti, lista)
  for _, s in pairs(per) do table.sort(s, function(a, b) return a[1] < b[1] end) end

  local usd = {}
  for _, s in ipairs(strumenti) do usd[s.id] = s.valuta == "USD" end
  local fx = per.EURUSD

  -- prezzo in EUR di un punto della serie
  local function eur(p, id)
    if not usd[id] then return p[2] end
    local f = fx and finoA(fx, p[1])
    return f and p[2] / f[2]
  end

  local out = {}
  for id, s in pairs(per) do
    if id ~= "EURUSD" and #s >= 2 then
      local ultimo = s[#s]
      local v1 = eur(ultimo, id)
      local function var(limite)
        local ref = finoA(s, limite)
        local v0 = ref and eur(ref, id)
        if v1 and v0 and v0 ~= 0 then return (v1 / v0 - 1) * 100 end
      end
      out[id] = {
        m1 = var(menoGiorni(ultimo[1], 30)),
        ytd = var(tostring(tonumber(string.sub(ultimo[1], 1, 4)) - 1) .. "-12-31"),
        a1 = var(menoGiorni(ultimo[1], 365)),
      }
    end
  end
  return out
end

-- Script delle barre di P/L: DATI = {etichette, valori}.
local JS_BARRE = [[
const dark = window.matchMedia && window.matchMedia("(prefers-color-scheme: dark)").matches;
const txt = dark ? "#ddd" : "#333", grid = "rgba(128,128,128,.25)";
new Chart(document.getElementById("c"), {type: "bar",
  data: {labels: DATI.etichette, datasets: [{data: DATI.valori,
    backgroundColor: DATI.valori.map(function (v) { return v >= 0 ? "#2a9d4a" : "#d6453d"; })}]},
  options: {indexAxis: "y", responsive: true, maintainAspectRatio: false, animation: false,
    plugins: {legend: {display: false},
      title: {display: true, text: "Profitto/perdita per strumento (EUR)", color: txt},
      tooltip: {callbacks: {label: function (c) { return c.parsed.x.toFixed(2) + " €"; }}}},
    scales: {x: {ticks: {color: txt}, grid: {color: grid}}, y: {ticks: {color: txt}, grid: {display: false}}}}});
]]

--- Barre orizzontali del P/L per strumento (verde guadagno, rosso perdita), dal maggiore al minore.
--- pagina: vedi investimenti.elenco.
function investimenti.barrePL(pagina)
  local ids = investimenti.elenco(pagina)
  if ids and #ids == 0 then return widget.markdownBlock("_Nessuno strumento._") end
  local righe = {}
  for _, d in ipairs(investimenti.valorizzazioneDati(ids, pagina)) do
    if d.pl then table.insert(righe, d) end
  end
  if #righe == 0 then return widget.markdownBlock("_Nessun P/L da mostrare._") end
  table.sort(righe, function(a, b) return a.pl > b.pl end)
  local e, v = {}, {}
  for _, d in ipairs(righe) do
    table.insert(e, '"' .. string.gsub(d.nome, '"', "'") .. '"')
    table.insert(v, string.format("%.2f", d.pl))
  end
  local dati = "const DATI = {etichette: [" .. table.concat(e, ",") .. "], valori: [" ..
    table.concat(v, ",") .. "]};\n"
  return widget.sandbox {
    html = '<div style="position:relative;height:' .. (60 + 28 * #righe) .. 'px"><canvas id="c"></canvas></div>',
    script = 'loadJsByUrl("' .. investimenti.chartjs .. '").then(function () {\n' .. dati .. JS_BARRE .. "\n});",
    markdown = "P/L per strumento",
  }
end

--- Widget da mostrare in testa a una pagina: lista di funzioni (nessun calcolo finché non
--- vengono chiamate), nell'ordine di visualizzazione. Lista vuota per le pagine non gestite.
--- Nome della pagina indice: investimenti.paginaIndice (predefinito "Index").
function investimenti.vista(pagina)
  pagina = pagina or editor.getCurrentPage()
  if type(pagina) ~= "string" then return {} end
  local function v(f, p) return function() return f(p) end end
  if pagina == (config.get("investimenti.paginaIndice") or "Index") then
    return {v(investimenti.valore), v(investimenti.barrePL), v(investimenti.allocazione)}
  elseif pagina == "Strumenti" then return {v(investimenti.elencoStrumenti)}
  elseif pagina == "Conti" then return {v(investimenti.elencoConti)}
  elseif string.startsWith(pagina, "Conti/") then
    return {
      function() return widget.markdown("Rischio (media ponderata): " .. investimenti.rischio(pagina)) end,
      v(investimenti.valorizzazione, pagina), v(investimenti.allocazione, pagina)}
  elseif string.startsWith(pagina, "Strumenti/") then
    return {v(investimenti.scheda, pagina), v(investimenti.valorizzazione, pagina), v(investimenti.grafico, pagina)}
  elseif string.startsWith(pagina, "Quotazioni/") then return {v(investimenti.grafico, pagina)}
  end
  return {}
end

-- hooks:renderTopWidgets (config.set("investimenti.widgetAutomatici", true)).
-- Un listener per posizione: il listener i mostra l'i-esimo widget di investimenti.vista.
for i = 1, 4 do
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

### Comando diagnostico v0.17

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
