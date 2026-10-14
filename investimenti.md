---
name: "Library/MG/investimenti"
tags: meta/library
version: "0.01"
versionDate: 2026-10-14
pageDecoration.prefix: "💸 "
share.uri: "github:marco10x15/silverbullet-libraries/investimenti.md"
---

# 💸 Investimenti

**Versione:** 0.01 — 14.10.2026 (test)

Libreria per monitorare gli investimenti con pagine Markdown: movimenti per strumento, quotazioni storiche scaricate da Yahoo Finance e valorizzazione con profitto/perdita (P/L), per tutto il portafoglio o per singolo conto. Le pagine Markdown restano la fonte primaria; la libreria le scrive solo nelle pagine `Quotazioni/<id>` e solo con i comandi o con `quotazioni.crea`/`quotazioni.aggiorna`.

## Funzioni

### investimenti.strumenti()

Restituisce l'elenco degli strumenti, una riga per ogni pagina con tag `strumento`, ordinata per id. Ogni riga contiene `id` (nome della pagina senza il prefisso `Strumenti/`), `nome` (`displayName` della pagina o, se manca, il suo nome), `ticker` (ticker Yahoo, presente solo per gli strumenti con quotazione automatica), `valuta` (default `EUR`), `conto` e `tipo`. È la sorgente di tutte le altre funzioni: aggiungere uno strumento significa solo creare la sua pagina.

Non ha argomenti.

### investimenti.movimenti(conto)

Legge le tabelle delle pagine `Strumenti/<id>` e somma per ogni strumento le quote nette (acquisti meno vendite), l'importo acquistato e l'importo incassato (vendite e dividendi), in EUR. Restituisce una riga per strumento con `id`, `quote`, `acquistato` e `incassato`. Per gli strumenti in valuta estera usa la colonna `Valore EUR` al posto di `Valore`.

**conto** (stringa, opzionale): nome di un conto, uguale al campo `conto` del frontmatter degli strumenti (per esempio `"NEF - Famiglia"`). Se presente, sono considerati solo gli strumenti di quel conto; se omesso, tutti.

### investimenti.ultimiPrezzi()

Cerca nelle tabelle delle pagine `Quotazioni/<id>` la riga con la data più recente e restituisce, per ogni pagina, `id`, `data`, `prezzo` e `valuta` (presa dallo strumento, default `EUR`). Scarta le righe senza data o prezzo, quindi non dipende dall'ordine delle righe nella tabella.

Non ha argomenti.

### investimenti.valorizzazione(conto)

Combina `movimenti` e `ultimiPrezzi` in una tabella in EUR con quote, prezzo, data del prezzo, acquistato, incassato, valore, P/L e P/L percentuale per strumento, più una riga TOTALE. P/L = valore + incassato − acquistato, al lordo di imposte e commissioni di conto. I prezzi in USD sono convertiti dividendo per l'ultimo prezzo dello strumento `EURUSD` (cambio USD per 1 EUR). Una posizione chiusa vale 0; uno strumento senza prezzo mostra `n.d.` ed è escluso dal totale, che lo segnala.

**conto** (stringa, opzionale): come in `investimenti.movimenti`. Con un nome di conto la tabella riguarda solo quel conto; è la forma usata nelle pagine `Conti/<nome>`, per esempio `${investimenti.valorizzazione("NEF - Famiglia")}`.

### quotazioni.yahoo(simbolo, range)

Scarica da Yahoo Finance (endpoint non ufficiale, uso personale) la serie giornaliera di un simbolo. Restituisce `{valuta, righe}`, dove `righe` è l'elenco di `{data, prezzo}` in ordine cronologico, oppure `{errore}` se la richiesta fallisce o il simbolo non ha dati. I giorni senza chiusura sono saltati e le date sono in UTC.

**simbolo** (stringa): ticker Yahoo, per esempio `0P00000I8A.F` (fondo su Francoforte), `BTC-EUR` o `EURUSD=X`. Se manca o non è una stringa, la funzione restituisce `{errore = "simbolo mancante"}`.

**range** (stringa, opzionale): periodo richiesto a Yahoo, per esempio `5d`, `1mo`, `1y`, `5y`, `max`. Il default è `5d`.

### quotazioni.test()

Funzione di prova: per ogni strumento con `ticker` scarica gli ultimi 5 giorni e restituisce `strumento`, `ticker`, `valuta`, `data` e `prezzo` dell'ultima chiusura, il numero di punti e l'eventuale errore. Serve a verificare che i ticker rispondano.

Non ha argomenti.

### quotazioni.cerca(testo)

Cerca su Yahoo Finance un ticker partendo da un ISIN o da un nome e restituisce i risultati grezzi, oppure `{errore}`. Serve a trovare il ticker da scrivere nel frontmatter di uno strumento.

**testo** (stringa): ISIN o nome dello strumento da cercare, per esempio `"LU0210526801"`.

### quotazioni.verifica(simbolo, range)

Riassume una serie Yahoo in una sola riga con `valuta`, numero di `punti`, `prima` e `ultima` data, ultimo `prezzo` ed eventuale errore. Serve a controllare la copertura storica e la valuta di un ticker prima di usarlo.

**simbolo** (stringa): ticker Yahoo da verificare.

**range** (stringa, opzionale): periodo richiesto. Il default è `5y`.

### quotazioni.crea()

Crea la pagina `Quotazioni/<id>` per ogni strumento che ha un `ticker` e non ha ancora la pagina, con tag `quotazioni` e la sola tabella `Data | Prezzo`. Restituisce l'elenco degli id creati, da passare a `quotazioni.aggiorna` per scaricarne lo storico.

Non ha argomenti.

### quotazioni.aggiorna(range, ids)

Scarica le quotazioni degli strumenti con `ticker`, le unisce per data alle righe già presenti (le ultime righe provvisorie vengono corrette) e riscrive la tabella con le date più recenti in alto. Il testo sopra la tabella resta invariato, quello sotto viene riscritto. Mostra una notifica con il numero di pagine e di errori; è un errore anche una serie con meno di 2 punti. Restituisce l'esito per ogni pagina.

**range** (stringa, opzionale): periodo Yahoo da scaricare. Il default è `1mo`; per lo storico completo di una pagina nuova si usa `5y`.

**ids** (elenco di stringhe, opzionale): id degli strumenti da aggiornare, come restituiti da `quotazioni.crea`. Se omesso, sono aggiornati tutti gli strumenti con `ticker`.

## Comandi

- **Quotazioni: Crea pagine e scarica storico**: crea le pagine `Quotazioni/<id>` mancanti e ne scarica 5 anni di storico (`quotazioni.aggiorna("5y", quotazioni.crea())`). Da usare quando si aggiunge uno strumento con `ticker`.
- **Quotazioni: Aggiorna**: aggiorna l'ultimo mese di tutte le quotazioni automatiche (`quotazioni.aggiorna("1mo")`). È il comando periodico.

## Funzioni di servizio

- **`str(x)`** (locale, blocco "Elenco strumenti"): restituisce `x` se è una stringa, altrimenti un vero `nil`. Serve perché un attributo assente letto in una `select` supera il controllo `if` pur avendo tipo `nil`.
- **`num(x)`** (locale, blocco "Movimenti e ultimi prezzi"): converte `x` in numero, con 0 se manca o non è convertibile.

## Struttura delle pagine

La libreria presuppone tre gruppi di pagine, con questi prefissi fissi.

**`Strumenti/<id>`**: una pagina per strumento, con tag `strumento` e una tabella dei movimenti. L'id è l'ISIN o un codice (`BTC`, `PIR-F-I50`).

```
---
tags: strumento
displayName: "NEF Ethical Euro Equity R Acc"
description: "Fondo - NEF - Famiglia"
isin: "LU0102238655"
ticker: "0P00000I8A.F"
tipo: "Fondo"
valuta: "EUR"
conto: "NEF - Famiglia"
---

| Data | Tipo | Quantità | Prezzo | Valore | Commissioni | Tasse |
|---|---|---|---|---|---|---|
| 2025-04-01 | Acquisto | 3.983 | 25.11 | 100 | 0 | 0 |
```

I tipi di movimento letti sono `Acquisto`, `Vendita` e `Dividendo`. `Valore` è l'importo in valuta dello strumento; per gli strumenti in valuta estera serve anche la colonna `Valore EUR`. Il `ticker` è facoltativo: senza, la quotazione si tiene a mano. Il cambio EUR/USD è lo strumento con id `EURUSD` (`ticker: "EURUSD=X"`, `valuta: "USD"`), senza movimenti.

**`Quotazioni/<id>`**: una pagina per strumento con la tabella `| Data | Prezzo |`, la più recente in alto. Le pagine automatiche sono create da `quotazioni.crea`; per gli strumenti senza `ticker` si crea a mano la pagina con la tabella e si aggiunge una riga a ogni aggiornamento.

**`Conti/<nome>`**: una pagina per conto, con tag `conto`. Il nome del conto è il valore del campo `conto` degli strumenti, e la valorizzazione si ottiene con `${investimenti.valorizzazione("<nome>")}`.

## Limiti

- L'endpoint di Yahoo Finance non è ufficiale e può cambiare o bloccare le richieste; i fondi hanno spesso un ritardo di uno o due giorni.
- Il P/L è al lordo di imposte e commissioni a livello di conto.
- Space Lua è indicato come sperimentale nella documentazione di SilverBullet.

## Codice

### Elenco strumenti

`investimenti.strumenti` e la funzione di servizio `str`.

```space-lua
investimenti = investimenti or {}

--- Strumenti (pagine con tag strumento) ordinati per id.
--- Un attributo assente in una select supera il controllo "if x" pur avendo type "nil":
--- str() lo riporta a un vero nil, così "if s.ticker" funziona.
local function str(x) if type(x) == "string" then return x end end

function investimenti.strumenti()
  local out = {}
  for _, p in ipairs(query[[
    from p = index.pages("strumento")
    select {name = p.name, nome = p.displayName, ticker = p.ticker,
      valuta = p.valuta, conto = p.conto, tipo = p.tipo}
  ]]) do
    table.insert(out, {id = string.sub(p.name, 11),  -- "Strumenti/" = 10 caratteri
      nome = str(p.nome) or p.name, ticker = str(p.ticker),
      valuta = str(p.valuta) or "EUR", conto = str(p.conto), tipo = str(p.tipo)})
  end
  table.sort(out, function(a, b) return a.id < b.id end)
  return out
end
```

### Funzioni Yahoo

`quotazioni.yahoo`, `test`, `cerca` e `verifica`.

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

### Creazione e aggiornamento

`quotazioni.crea` e `quotazioni.aggiorna`.

```space-lua
quotazioni = quotazioni or {}

--- Crea le pagine Quotazioni/<id> mancanti (solo strumenti con ticker). Restituisce gli id creati.
function quotazioni.crea()
  local create = {}
  for _, s in ipairs(investimenti.strumenti()) do
    if s.ticker and not space.pageExists("Quotazioni/" .. s.id) then
      space.writePage("Quotazioni/" .. s.id,
        "---\ntags: quotazioni\n---\n\n# " .. s.nome .. "\n\n| Data | Prezzo |\n|---|---|\n")
      table.insert(create, s.id)
    end
  end
  return create
end

--- Scarica e unisce le quotazioni (per data) nelle pagine Quotazioni/<id>.
--- range: periodo Yahoo (default "1mo"); ids: elenco di id (default: strumenti con ticker).
function quotazioni.aggiorna(range, ids)
  range = range or "1mo"
  local ticker, tutti = {}, {}
  for _, s in ipairs(investimenti.strumenti()) do
    if s.ticker then
      ticker[s.id] = s.ticker
      table.insert(tutti, s.id)
    end
  end
  ids = ids or tutti
  local esiti, err = {}, 0
  for _, id in ipairs(ids) do
    local nome = "Quotazioni/" .. id
    if ticker[id] and space.pageExists(nome) then
      local testo = space.readPage(nome)
      local i = string.find(testo, "| Data | Prezzo |", 1, true)
      local q = i and quotazioni.yahoo(ticker[id], range) or {errore = "tabella non trovata"}
      if q.righe then
        local m = {}  -- data -> prezzo
        for d, v in string.gmatch(testo, "| (%d%d%d%d%-%d%d%-%d%d) | ([%d%.]+) |") do m[d] = v end
        for _, r in ipairs(q.righe) do m[r.data] = string.format("%.4f", r.prezzo) end
        local date = {}
        for d in pairs(m) do table.insert(date, d) end
        table.sort(date, function(a, b) return a > b end)  -- più recenti in alto
        local righe = {"| Data | Prezzo |", "|---|---|"}
        for _, d in ipairs(date) do table.insert(righe, "| " .. d .. " | " .. m[d] .. " |") end
        space.writePage(nome, string.sub(testo, 1, i - 1) .. table.concat(righe, "\n") .. "\n")
        if #q.righe < 2 then err = err + 1 end
      else
        err = err + 1
      end
      table.insert(esiti, {pagina = nome, righe = q.righe and #q.righe, errore = q.errore})
    end
  end
  editor.flashNotification(#esiti .. " pagine, " .. err .. " errori", err > 0 and "warning" or "info")
  return esiti
end
```

### Comandi

I due comandi sono in blocchi separati, uno per blocco.

```space-lua
command.define {
  name = "Quotazioni: Crea pagine e scarica storico",
  run = function()
    quotazioni.aggiorna("5y", quotazioni.crea())
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

### Movimenti e ultimi prezzi

`investimenti.movimenti`, `investimenti.ultimiPrezzi` e la funzione di servizio `num`.

```space-lua
investimenti = investimenti or {}

local function num(x) return tonumber(x or 0) or 0 end

--- Quote, acquistato e incassato (EUR) per ogni strumento; conto opzionale per filtrare.
function investimenti.movimenti(conto)
  local info = {}
  for _, s in ipairs(investimenti.strumenti()) do info[s.id] = s end

  local per, ids = {}, {}
  for _, r in ipairs(query[[
    from t = index.tables()
    where string.startsWith(t.page, "Strumenti/")
    select {page = t.page, ref = t.ref, tipo = t.tipo, quote = t.quantit_, valore = t.valore_eur or t.valore}
  ]]) do
    local id = string.sub(r.page, 11)  -- "Strumenti/" = 10 caratteri
    if not conto or (info[id] and info[id].conto == conto) then
      local m = per[id]
      if not m then
        m = {id = id, quote = 0, acquistato = 0, incassato = 0}
        per[id] = m
        table.insert(ids, id)
      end
      if r.tipo == "Acquisto" then
        m.quote, m.acquistato = m.quote + num(r.quote), m.acquistato + num(r.valore)
      elseif r.tipo == "Vendita" then
        m.quote, m.incassato = m.quote - num(r.quote), m.incassato + num(r.valore)
      elseif r.tipo == "Dividendo" then
        m.incassato = m.incassato + num(r.valore)
      end
    end
  end
  table.sort(ids)
  local out = {}
  for _, id in ipairs(ids) do table.insert(out, per[id]) end
  return out
end

--- Ultimo prezzo di ogni pagina Quotazioni/<id>: lista {id, data, prezzo, valuta}.
function investimenti.ultimiPrezzi()
  local valuta = {}
  for _, s in ipairs(investimenti.strumenti()) do valuta[s.id] = s.valuta end

  local last = {}
  for _, r in ipairs(query[[
    from t = index.tables()
    where string.startsWith(t.page, "Quotazioni/")
    select {pagina = t.page, ref = t.ref, giorno = t.data, px = t.prezzo}
  ]]) do
    local d, pr = r.giorno, tonumber(r.px)
    -- salta righe prive di data/prezzo
    if type(d) == "string" and pr then
      local l = last[r.pagina]
      if not l or d > l.data then last[r.pagina] = {data = d, prezzo = pr} end
    end
  end

  local out = {}
  for page, r in pairs(last) do
    local id = string.sub(page, 12)  -- "Quotazioni/" = 11 caratteri
    table.insert(out, {id = id, data = r.data, prezzo = r.prezzo, valuta = valuta[id] or "EUR"})
  end
  table.sort(out, function(a, b) return a.id < b.id end)
  return out
end
```

### Valorizzazione

`investimenti.valorizzazione`.

```space-lua
investimenti = investimenti or {}

function investimenti.valorizzazione(conto)
  local function f(x, d) return x and string.format("%." .. d .. "f", x) or "n.d." end
  local px, fx = {}, nil
  for _, p in ipairs(investimenti.ultimiPrezzi()) do
    px[p.id] = p
    if p.id == "EURUSD" then fx = p.prezzo end
  end
  local righe, tot, nd = {}, {a = 0, i = 0, v = 0, pl = 0}, 0
  for _, m in ipairs(investimenti.movimenti(conto)) do
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
    table.insert(righe, {strumento = m.id, quote = f(m.quote, 3), prezzo_eur = f(prezzo, 4),
      data_prezzo = p and p.data or "n.d.", acquistato = f(m.acquistato, 2),
      incassato = f(m.incassato, 2), valore = f(valore, 2), pl = f(pl, 2), pl_pct = f(pct, 2)})
  end
  table.insert(righe, {strumento = "TOTALE" .. (nd > 0 and " (esclusi " .. nd .. " senza prezzo)" or ""),
    acquistato = f(tot.a, 2), incassato = f(tot.i, 2), valore = f(tot.v, 2), pl = f(tot.pl, 2)})
  return righe
end
```
