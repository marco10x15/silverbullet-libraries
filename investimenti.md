---
name: "Library/MG/investimenti"
tags: meta/library
version: "0.13"
versionDate: 2026-10-16
pageDecoration.prefix: "💸 "
share.uri: "github:marco10x15/silverbullet-libraries/investimenti.md"
---

# 💸 Investimenti

**Versione:** 0.13 — 16.10.2026 (Produzione)

Libreria per monitorare gli investimenti con pagine Markdown: movimenti per strumento, quotazioni storiche scaricate da Yahoo Finance e valorizzazione con profitto/perdita (P/L), per singolo strumento, per conto o per portafoglio. Le pagine Markdown restano la fonte primaria; la libreria scrive solo nelle pagine `Quotazioni/...`, con `quotazioni.crea` e `quotazioni.aggiorna`.

Richiede la libreria `DateFormat` per formattare la data dei prezzi; senza, la data resta nel formato ISO.

## Funzioni

### investimenti.valorizzazione(pagina)

Mostra la valorizzazione in EUR come tabella con le colonne `strumento` (link allo strumento, con il suo `displayName`), `quote`, `prezzo_eur`, `data_prezzo`, `acquistato`, `incassato`, `valore`, `pl` e `%`, più una riga TOTALE in grassetto quando gli strumenti sono più di uno. Sotto i 600 px ogni riga diventa una scheda. P/L = valore + incassato − acquistato, al lordo di imposte e commissioni di conto. Le colonne numeriche sono allineate a destra con due decimali (`quote` con fino a 8 decimali, senza zeri finali), `%` è centrata con il simbolo `%`. I prezzi in USD sono convertiti dividendo per l'ultimo prezzo dello strumento `EURUSD` (cambio USD per 1 EUR). Una posizione chiusa vale 0; uno strumento senza prezzo mostra `n.d.` ed è escluso dal totale, che lo segnala. Il prezzo è mostrato con due decimali: un prezzo come 0,9990 appare come 1,00, mentre il valore è calcolato con il prezzo completo.

Quali strumenti includere dipende dalla pagina in cui è usata: in `Conti/<nome>` (conto o portafoglio) quelli elencati nell'attributo `strumento` del frontmatter; in `Strumenti/<id>` solo lo strumento della pagina; altrove tutti gli strumenti.

**pagina** (stringa, opzionale): nome della pagina da cui ricavare l'elenco degli strumenti, per esempio `"Conti/NEF - Famiglia"`. Se omesso, è la pagina aperta nell'editor. Serve per mostrare la valorizzazione di un conto in una pagina diversa, per esempio un cruscotto.

### investimenti.valore(pagina)

Versione compatta di `valorizzazione`: tabella con le sole colonne Investimento (link con `displayName`), Acquistato, Attuale (valore di mercato in EUR), P/L e %, più la riga TOTALE. Stesso calcolo di `valorizzazione`.

- **`pagina`** (stringa, opzionale): come in `investimenti.elenco`; se omessa, la pagina corrente.

### investimenti.grafico(pagina)

Grafico lineare delle quotazioni (Chart.js in un widget isolato), con i pulsanti di periodo 7G, 1M, 3M, 6M, YTD, 1A, 5A e Tutto (predefinito 6M). Inserito in una pagina `Quotazioni/...` mostra la serie; in una pagina `Strumenti/<id>` aggiunge i punti di acquisto (triangoli, al prezzo unitario pagato). Chart.js viene caricato da cdnjs.

- **`pagina`** (stringa, opzionale): `Quotazioni/...` o `Strumenti/<id>`; se omessa, la pagina corrente.

### investimenti.torta(pagina) e investimenti.tortaRischio(pagina)

Ciambelle del valore attuale (EUR): `torta` per strumento, `tortaRischio` per classe di rischio 1-7 letta dall'attributo `KID` degli strumenti (gli strumenti senza `KID` vanno in "n.d."). Usano `valorizzazioneDati`, quindi gli strumenti senza prezzo o chiusi non compaiono.

- **`pagina`** (stringa, opzionale): come in `investimenti.elenco`.

### investimenti.allocazione(pagina)

Le due ciambelle (valore per strumento e valore per classe di rischio KID) in un solo widget con griglia CSS `auto-fit`: affiancate su schermo largo, impilate su schermo stretto. Sostituisce la coppia `torta` + `tortaRischio` nelle viste automatiche; le due funzioni restano disponibili singolarmente.

- **`pagina`** (stringa, opzionale): come in `investimenti.elenco`.

### investimenti.rischio(pagina) e investimenti.barraRischio(n)

`rischio` restituisce il testo con la barra di rischio in emoji (es. `🟩🟩🟨🟨🟧⬜⬜ 5/7`): per uno strumento la sua classe, per conti e portafogli la media dei `KID` ponderata sul valore attuale. `barraRischio` disegna la barra per un numero 1-7 e si può usare in altre tabelle.

- **`pagina`** (stringa, opzionale): come in `investimenti.elenco`.
- **`n`** (numero): classe di rischio da 1 a 7, arrotondata; se non è un numero restituisce `n.d.`.

### investimenti.serie(id) e investimenti.acquisti(id)

Dati dei grafici: `serie(pagina)` restituisce le quotazioni `{data, prezzo}` di una pagina `Quotazioni/...` in ordine di data; `acquisti` le righe `Acquisto` di `Strumenti/<id>` come `{data, prezzo unitario}`.

- **`id`** (stringa): id dello strumento, cioè il nome della pagina senza prefisso.

### investimenti.performance(ids)

Variazione percentuale per strumento a 1 mese (30 giorni), da inizio anno (YTD, rispetto all'ultima quotazione dell'anno precedente) e a 1 anno (365 giorni), calcolata sulle quotazioni; restituisce una tabella id → `{m1, ytd, a1}`, con `nil` dove lo storico non basta (strumenti a quotazione manuale con un solo prezzo). Per i fondi in USD usa il cambio EURUSD alla data, quindi la variazione è in EUR. È la fonte delle colonne 1M, YTD e 1A di `investimenti.valore`.

- **`ids`** (lista di stringhe, opzionale): id degli strumenti; se omessa, tutti quelli con quotazioni.

### investimenti.barrePL(pagina)

Barre orizzontali (Chart.js) del P/L per strumento, in verde se positivo e in rosso se negativo, ordinate dal maggiore al minore.

- **`pagina`** (stringa, opzionale): come in `investimenti.elenco`.

### investimenti.scheda(pagina), investimenti.elencoStrumenti() e investimenti.elencoConti()

Viste costruite dal frontmatter. `scheda` (pagine `Strumenti/<id>`): descrizione, tipo, valuta, ISIN, tag, barra di rischio e link ai conti/portafogli che contengono lo strumento (attributo `strumento`). `elencoStrumenti` (pagina `Strumenti`): tabella di tutti gli strumenti con tipo, valuta, ultimo prezzo, data e rischio. `elencoConti` (pagina `Conti`): tabella di conti e portafogli con tipo (dal tag `Conto` o `Portafoglio`), numero di strumenti, acquistato, attuale, P/L e %, calcolati con una sola valorizzazione.

- **`pagina`** (stringa, opzionale, solo `scheda`): `Strumenti/<id>`; se omessa, la pagina corrente.

### Widget automatici

Nessuna chiamata nelle pagine: `investimenti.vista(pagina)` elenca i widget di ogni pagina (funzioni, non ancora calcolate) e la libreria li mostra in testa con listener `hooks:renderTopWidgets`, uno per posizione, se `config.set("investimenti.widgetAutomatici", true)`. `view.define` non è usato: il contenuto di una vista non può essere un widget isolato (`navigator: content cannot return a sandboxed widget`) né un `dom.div` con widget come figli.

Pagine gestite: `Index` (nome modificabile con `investimenti.paginaIndice`): `valore`, `barrePL`, `allocazione`; `Strumenti`: `elencoStrumenti`; `Conti`: `elencoConti`; `Conti/<nome>` e portafogli: rischio medio, `valorizzazione`, `allocazione`; `Strumenti/<id>`: `scheda`, `valorizzazione`, `grafico`; `Quotazioni/<id>`: `grafico`. Se una pagina ha già le chiamate `${...}` corrispondenti, i widget compaiono due volte: vanno tolte dalla pagina. Con un solo strumento `valorizzazione` è un elenco verticale campo/valore, senza intestazione e senza riga TOTALE.

### investimenti.movimenti(pagina)

Mostra come tabella i movimenti aggregati per strumento: `id` (link allo strumento, con il suo `displayName`), `quote` (nette: acquisti meno vendite), `acquistato` e `incassato` (vendite e dividendi), in EUR. Le colonne numeriche sono allineate a destra, con due decimali per gli importi. Per gli strumenti in valuta estera usa la colonna `Valore EUR` al posto di `Valore`.

**pagina** (stringa, opzionale): come in `investimenti.valorizzazione`: la pagina da cui ricavare l'elenco degli strumenti. Se omesso, è la pagina aperta nell'editor.

### investimenti.valorizzazioneDati(ids)

Restituisce i dati numerici della valorizzazione, non formattati, per riusarli (grafici, schede). Restituisce tre valori: l'elenco delle righe (`id`, `nome`, `quote`, `prezzo`, `data`, `acquistato`, `incassato`, `valore`, `pl`, `pct`, con `nil` dove un dato manca), il totale (`a` acquistato, `i` incassato, `v` valore, `pl`) e il numero di strumenti senza prezzo.

**ids** (elenco di stringhe, opzionale): id degli strumenti da includere, cioè i nomi delle pagine `Strumenti/` senza prefisso. Se omesso, tutti.

### investimenti.movimentiDati(ids)

Legge le tabelle delle pagine `Strumenti/<id>` e somma per ogni strumento le quote nette, l'importo acquistato e l'importo incassato, in EUR. Restituisce una riga per strumento, ordinata per id, con `id`, `quote`, `acquistato` e `incassato` come numeri.

**ids** (elenco di stringhe, opzionale): id degli strumenti da includere. Se omesso, tutti.

### investimenti.pagineQuotazioni()

Restituisce due tabelle: `isin → pagina` e `pagina → isin`, lette dall'attributo `isin` del frontmatter delle pagine sotto `Quotazioni/`. È l'unico punto in cui si risolve il collegamento tra uno strumento e le sue quotazioni. Non ha argomenti.

### investimenti.quotazioniPerStrumento(strumenti, ids)

Legge le tabelle `Quotazioni/...` una sola volta e le raggruppa per strumento (collegamento via `isin`): restituisce una tabella `id → lista di {data, prezzo}`, non ordinata. Scarta le righe senza data o prezzo e le pagine senza strumento corrispondente. Alimenta `ultimiPrezzi` e `performance`.

**strumenti** (elenco, opzionale): il risultato di `investimenti.strumenti()`, se già disponibile.

**ids** (elenco di stringhe, opzionale): id degli strumenti da includere. Se omesso, tutti.

### investimenti.ultimiPrezzi(strumenti)

Per ogni strumento con quotazioni (collegate via `isin`) restituisce la riga con la data più recente: `id`, `data`, `prezzo` e `valuta` (presa dallo strumento, default `EUR`). Scarta le righe senza data o prezzo, quindi non dipende dall'ordine delle righe nella tabella.

**strumenti** (elenco, opzionale): il risultato di `investimenti.strumenti()`, se già disponibile, per non rileggerlo. Se omesso, viene letto.

### investimenti.elenco(pagina)

Restituisce l'elenco degli id degli strumenti di una pagina. In `Conti/<nome>` legge l'attributo `strumento` del frontmatter (un elenco di link `[[Strumenti/<id>]]`) ed estrae gli id; in `Strumenti/<id>` restituisce l'id della pagina; per qualsiasi altra pagina restituisce `nil`, cioè tutti gli strumenti. Un conto senza l'attributo restituisce un elenco vuoto.

**pagina** (stringa, opzionale): nome della pagina. Se omesso, è la pagina aperta nell'editor.

### investimenti.strumenti()

Restituisce l'elenco degli strumenti, una riga per ogni pagina sotto `Strumenti/`, ordinata per id. Ogni riga contiene `id` (nome della pagina senza il prefisso), `nome` (`displayName` o, se manca, il nome della pagina), `isin`, `ticker` (ticker Yahoo, presente solo per gli strumenti con quotazione automatica), `valuta` (default `EUR`) e `tipo`. Aggiungere uno strumento significa solo creare la sua pagina.

Non ha argomenti.

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

Crea la pagina `Quotazioni/<id>` per ogni strumento che ha `ticker` e `isin` e non ha ancora una pagina con quell'`isin`. La pagina contiene il frontmatter `isin`, il titolo, il grafico e la tabella `Data | Prezzo`. Restituisce l'elenco degli id creati e una tabella `id → pagina`, da passare a `quotazioni.aggiorna` (l'indice potrebbe non conoscere ancora le pagine nuove). Gli strumenti senza `isin` vengono saltati.

Non ha argomenti.

### quotazioni.aggiorna(range, ids, pagine)

Scarica le quotazioni degli strumenti con `ticker`, le unisce per data alle righe già presenti (le ultime righe provvisorie vengono corrette) e riscrive la tabella con le date più recenti in alto. Il testo sopra la tabella resta invariato, quello sotto viene riscritto. Mostra una notifica con il numero di pagine e di errori; è un errore anche una serie con meno di 2 punti. Restituisce l'esito per ogni pagina.

**range** (stringa, opzionale): periodo Yahoo da scaricare. Il default è `1mo`; per lo storico completo di una pagina nuova si usa `5y`.

**ids** (elenco di stringhe, opzionale): id degli strumenti da aggiornare, come restituiti da `quotazioni.crea`. Se omesso, sono aggiornati tutti gli strumenti con `ticker` che hanno una pagina `Quotazioni` con lo stesso `isin`.

**pagine** (tabella, opzionale): `id → pagina` per le pagine appena create da `quotazioni.crea`.

## Comandi

- **Quotazioni: Crea pagine e scarica storico**: crea le pagine `Quotazioni/<id>` mancanti e ne scarica 5 anni di storico (`quotazioni.crea` seguita da `quotazioni.aggiorna("5y", ...)`). Da usare quando si aggiunge uno strumento con `ticker` e `isin`.
- **Quotazioni: Aggiorna**: aggiorna l'ultimo mese di tutte le quotazioni automatiche (`quotazioni.aggiorna("1mo")`). È il comando periodico.

## Funzioni di servizio

Sono locali ai blocchi di codice e non fanno parte dell'interfaccia.

- **`str(x)`**: restituisce `x` se è una stringa, altrimenti un vero `nil`. Serve perché un attributo assente letto in una `select` supera il controllo `if` pur avendo tipo `nil`.
- **`num(x)`**: converte `x` in numero, con 0 se manca o non è convertibile.
- **`migliaia(t)`**: separa con uno spazio le migliaia della parte intera di un numero scritto come testo (`15000.00` → `15 000.00`).
- **`f2(x)`**: formatta un numero con due decimali e migliaia separate, oppure `n.d.` se manca.
- **`fq(x)`**: formatta una quantità con fino a 8 decimali, senza zeri finali, con migliaia separate.
- **`data(d)`**: formatta una data ISO come `DD.MM.YYYY` con `date.format` (libreria `DateFormat`); se la libreria manca restituisce la data invariata, se manca la data restituisce `n.d.`.
- **`investimenti.nomi(strumenti)`**: costruisce la tabella id → `displayName` a partire dall'elenco degli strumenti. Condivisa tra i blocchi.
- **`investimenti.chartjs`**: indirizzo di Chart.js su cdnjs, usato da tutti i grafici.
- **`torta(titolo, etichette, valori, colori)`**: costruisce il widget a ciambella usato da `torta` e `tortaRischio`.
- **`tabella(colonne, righe, verticale)`**: costruisce la tabella come widget isolato, con allineamento per colonna, celle con link (allo strumento, o alla pagina indicata in `pagina`) e riga di totale in grassetto. Sotto i 600 px di larghezza ogni riga diventa una scheda (etichetta e valore per ogni campo); con `verticale = true` è un elenco campo/valore senza intestazione. Il contenuto Markdown del widget alimenta il pulsante Copia.
- **`investimenti.js(x)`**: converte un numero o una stringa in testo JavaScript per i dati di grafici e tabelle, con escape di barre inverse, virgolette e a capo.

## Struttura delle pagine

La libreria presuppone tre gruppi di pagine, con questi prefissi fissi. L'appartenenza si deduce dalla cartella, quindi non servono tag su `Strumenti/` e `Quotazioni/`.

**`Strumenti/<id>`**: una pagina per strumento, con una tabella dei movimenti. L'id è l'ISIN o un codice (`BTC`, `PIR-F-I50`); dove esiste, `isin` coincide con il nome della pagina.

```
---
displayName: "NEF Ethical Euro Equity R Acc"
description: "Fondo - NEF - Famiglia"
isin: "LU0102238655"
ticker: "0P00000I8A.F"
tipo: "Fondo"
valuta: "EUR"
KID: 4
---

${investimenti.valorizzazione()}

| Data | Tipo | Quantità | Prezzo | Valore | Commissioni | Tasse |
|---|---|---|---|---|---|---|
| 2025-04-01 | Acquisto | 3.983 | 25.11 | 100 | 0 | 0 |
```

I tipi di movimento letti sono `Acquisto`, `Vendita` e `Dividendo`. `Valore` è l'importo in valuta dello strumento; per gli strumenti in valuta estera serve anche la colonna `Valore EUR`. `KID` (1-7, indicatore sintetico di rischio del KID; maiuscolo) è facoltativo e serve a `rischio` e `tortaRischio`. L'`isin` collega lo strumento alla sua pagina `Quotazioni/...`: serve in ogni strumento che ha quotazioni (anche un codice interno, purché uguale nelle due pagine). Il `ticker` è facoltativo: senza, la quotazione si tiene a mano. Il cambio EUR/USD è lo strumento con id `EURUSD` (`isin: "EURUSD"`, `ticker: "EURUSD=X"`, `valuta: "USD"`), senza movimenti.

**`Quotazioni/<nome>`**: una pagina per strumento con la tabella `| Data | Prezzo |`, la più recente in alto. Il nome è libero; il collegamento allo strumento è l'attributo `isin` nel frontmatter, uguale a quello di `Strumenti/<id>`:

```
---
isin: "PIR-I50"
---
```

Le pagine automatiche sono create da `quotazioni.crea`; per gli strumenti senza `ticker` si crea a mano la pagina con la tabella e si aggiunge una riga a ogni aggiornamento.

**`Conti/<nome>`**: una pagina per conto o per portafoglio. L'attributo `strumento` elenca gli strumenti che contiene; il tag `Conto` indica un conto che contiene gli investimenti, `Portafoglio` un raggruppamento di strumenti che possono stare in conti diversi. La valorizzazione si ottiene con `${investimenti.valorizzazione()}`.

```
---
displayName: "NEF - Famiglia"
valuta: "EUR"
strumento:
  - "[[Strumenti/LU0102238655]]"
  - "[[Strumenti/LU0102240719]]"
tags: Conto
---

${investimenti.valorizzazione()}
```

## Limiti

- L'endpoint di Yahoo Finance non è ufficiale e può cambiare o bloccare le richieste; i fondi hanno spesso un ritardo di uno o due giorni.
- Il P/L è al lordo di imposte e commissioni a livello di conto.
- Lo strumento di una pagina `Strumenti/` è identificato dal nome della pagina, non dall'attributo `isin`: coincidono dove l'ISIN esiste, e il nome funziona anche per `BTC` e per i PIR, che non hanno ISIN.
- Per riusare i dati (altre tabelle, grafici) usare `valorizzazioneDati` e `movimentiDati`, che restituiscono tabelle Lua senza formattazione.

## Codice

### Elenco strumenti e pagina di riferimento

`investimenti.strumenti`, `investimenti.elenco` e la funzione di servizio `str`. `elenco` legge l'attributo `strumento` dei conti con la stessa query di `strumenti` (sottopagine di `Conti`), senza usare tag.

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

### Funzioni Yahoo

`quotazioni.yahoo`, `test`, `cerca` e `verifica`. La riga `quotazioni = quotazioni or {}` è necessaria perché l'ordine di caricamento dei blocchi non è garantito.

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
function quotazioni.aggiorna(range, ids, pagine)
  range = range or "1mo"
  local perIsin = investimenti.pagineQuotazioni()
  local ticker, tutti = {}, {}
  for _, s in ipairs(investimenti.strumenti()) do
    if s.ticker then
      ticker[s.id] = s.ticker
      table.insert(tutti, s.id)
      pagine = pagine or {}
      pagine[s.id] = pagine[s.id] or (s.isin and perIsin[s.isin])
    end
  end
  ids = ids or tutti
  local esiti, err = {}, 0
  for _, id in ipairs(ids) do
    local nome = pagine and pagine[id]
    if ticker[id] and nome and space.pageExists(nome) then
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

### Dati: movimenti e ultimi prezzi

`investimenti.movimentiDati`, `investimenti.ultimiPrezzi` e la funzione di servizio `num`. La `select` di `movimentiDati` include `ref` perché righe identiche verrebbero altrimenti fuse.

```space-lua
investimenti = investimenti or {}

local function num(x) return tonumber(x or 0) or 0 end

--- Quote, acquistato e incassato (EUR) per ogni strumento; ids opzionale per filtrare.
function investimenti.movimentiDati(ids)
  local ammessi
  if ids then
    ammessi = {}
    for _, id in ipairs(ids) do ammessi[id] = true end
  end

  local per, elenco = {}, {}
  for _, r in ipairs(query[[
    from t = index.tables()
    where string.startsWith(t.page, "Strumenti/")
    select {page = t.page, ref = t.ref, tipo = t.tipo, quote = t.quantit_, valore = t.valore_eur or t.valore}
  ]]) do
    local id = string.sub(r.page, 11)  -- "Strumenti/" = 10 caratteri
    if not ammessi or ammessi[id] then
      local m = per[id]
      if not m then
        m = {id = id, quote = 0, acquistato = 0, incassato = 0}
        per[id] = m
        table.insert(elenco, id)
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
  table.sort(elenco)
  local out = {}
  for _, id in ipairs(elenco) do table.insert(out, per[id]) end
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
  local idDi = {}  -- isin -> id strumento
  for _, s in ipairs(strumenti or investimenti.strumenti()) do
    if s.isin and (not ammessi or ammessi[s.id]) then idDi[s.isin] = s.id end
  end

  local per = {}
  for _, r in ipairs(query[[
    from t = index.tables()
    where string.startsWith(t.page, "Quotazioni/")
    select {pagina = t.page, ref = t.ref, giorno = t.data, px = t.prezzo}
  ]]) do
    local id = idDi[perPagina[r.pagina]]
    local pr = tonumber(r.px)
    -- salta righe prive di data/prezzo e pagine non collegate a uno strumento
    if id and type(r.giorno) == "string" and pr then
      per[id] = per[id] or {}
      table.insert(per[id], {r.giorno, pr})
    end
  end
  return per
end

--- Ultimo prezzo di ogni strumento con quotazioni: lista {id, data, prezzo, valuta}.
--- strumenti: risultato di investimenti.strumenti(), se già disponibile.
function investimenti.ultimiPrezzi(strumenti)
  strumenti = strumenti or investimenti.strumenti()
  local per = investimenti.quotazioniPerStrumento(strumenti)
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

### Dati: valorizzazione

`investimenti.valorizzazioneDati` combina movimenti e ultimi prezzi in righe numeriche; non formatta nulla.

```space-lua
investimenti = investimenti or {}

--- Righe {id, nome, quote, prezzo, data, acquistato, incassato, valore, pl, pct}, totale {a, i, v, pl}
--- e numero di strumenti senza prezzo. Prezzi in USD convertiti con lo strumento EURUSD.
function investimenti.valorizzazioneDati(ids)
  local strumenti = investimenti.strumenti()
  local nome = investimenti.nomi(strumenti)

  local px, fx = {}, nil
  for _, p in ipairs(investimenti.ultimiPrezzi(strumenti)) do
    px[p.id] = p
    if p.id == "EURUSD" then fx = p.prezzo end
  end

  local righe, tot, nd = {}, {a = 0, i = 0, v = 0, pl = 0}, 0
  for _, m in ipairs(investimenti.movimentiDati(ids)) do
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
    table.insert(righe, {id = m.id, nome = nome[m.id] or m.id, quote = m.quote, prezzo = prezzo,
      data = p and p.data, acquistato = m.acquistato, incassato = m.incassato,
      valore = valore, pl = pl, pct = pct})
  end
  return righe, tot, nd
end
```

### Presentazione: tabelle

`investimenti.movimenti` e `investimenti.valorizzazione` costruiscono la tabella HTML di un widget. Le funzioni di servizio `migliaia`, `f2`, `fq`, `data` e `tabella` formattano numeri e date e disegnano la tabella; ogni colonna ha titolo, chiave, allineamento e le celle `{id, nome}` diventano link allo strumento.

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
  for _, m in ipairs(investimenti.movimentiDati(ids)) do
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
  local dati, tot, nd = investimenti.valorizzazioneDati(ids)
  local righe = {}
  for _, d in ipairs(dati) do
    table.insert(righe, {strumento = {id = d.id, nome = d.nome}, quote = fq(d.quote),
      prezzo_eur = f2(d.prezzo), data_prezzo = d.prezzo and data(d.data) or "n.d.",
      acquistato = f2(d.acquistato), incassato = f2(d.incassato), valore = f2(d.valore),
      pl = f2(d.pl), pct = d.pct and (f2(d.pct) .. " %") or "n.d."})
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
  }, righe)
end

--- Valorizzazione compatta: investimento, acquistato, attuale, P/L, %, performance 1M/YTD/1A.
--- pagina: vedi investimenti.elenco.
function investimenti.valore(pagina)
  local ids = investimenti.elenco(pagina)
  if ids and #ids == 0 then return widget.markdownBlock("_Nessuno strumento._") end
  local dati, tot, nd = investimenti.valorizzazioneDati(ids)
  local perf = investimenti.performance(ids)
  local function fp(x) return x and string.format("%+.2f %%", x) or "n.d." end
  local righe = {}
  for _, d in ipairs(dati) do
    local f = perf[d.id] or {}
    table.insert(righe, {nome = {id = d.id, nome = d.nome}, acquistato = f2(d.acquistato),
      attuale = f2(d.valore), pl = f2(d.pl), pct = d.pct and (f2(d.pct) .. " %") or "n.d.",
      m1 = fp(f.m1), ytd = fp(f.ytd), a1 = fp(f.a1)})
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
  local per = {}
  for _, d in ipairs(investimenti.valorizzazioneDati(nil)) do per[d.id] = d end
  local righe = {}
  for _, p in ipairs(query[[
    from p = index.subPages("Conti")
    select {name = p.name, nome = p.displayName, tags = p.tags}
  ]]) do
    local ids = investimenti.elenco(p.name) or {}
    local a, i, v = 0, 0, 0
    for _, id in ipairs(ids) do
      local d = per[id]
      if d and d.valore then a, i, v = a + d.acquistato, i + d.incassato, v + d.valore end
    end
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

### Grafici e rischio

`investimenti.serie` e `investimenti.acquisti` leggono quotazioni e acquisti di uno strumento con una query per pagina (l'id è una variabile locale usata nella query). `investimenti.grafico` costruisce il widget Chart.js (via `widget.sandbox` e `loadJsByUrl`) con i pulsanti di periodo; `torta` e `tortaRischio` disegnano le ciambelle; `barraRischio` e `rischio` mostrano la classe 1-7 con emoji.

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

--- Acquisti di uno strumento: lista di {data, prezzo unitario in valuta strumento}.
function investimenti.acquisti(id)
  local out = {}
  for _, r in ipairs(query[[
    from t = index.tables()
    where t.page == "Strumenti/" .. id and t.tipo == "Acquisto"
    select {ref = t.ref, giorno = t.data, px = t.prezzo}
  ]]) do
    local pr = tonumber(r.px)
    if type(r.giorno) == "string" and pr then table.insert(out, {r.giorno, pr}) end
  end
  table.sort(out, function(a, b) return a[1] < b[1] end)
  return out
end

-- Script del grafico lineare: dati in DATI = {serie, acquisti, titolo}.
local JS_LINEA = [[
const RANGE = [{n:"7G",d:7},{n:"1M",d:30},{n:"3M",d:91},{n:"6M",d:182},{n:"YTD",d:-1},{n:"1A",d:365},{n:"5A",d:1826},{n:"Tutto",d:0}];
const dark = window.matchMedia && window.matchMedia("(prefers-color-scheme: dark)").matches;
const txt = dark ? "#ddd" : "#333", grid = "rgba(128,128,128,.25)";
const ultimo = DATI.serie[DATI.serie.length - 1][0];
let chart = null;
function da(r) {
  if (r.d === 0) return "0000-00-00";
  if (r.d < 0) return ultimo.slice(0, 4) + "-01-01";
  const d = new Date(ultimo + "T00:00:00Z");
  d.setUTCDate(d.getUTCDate() - r.d);
  return d.toISOString().slice(0, 10);
}
function disegna(r) {
  const da0 = da(r);
  const s = DATI.serie.filter(function (p) { return p[0] >= da0; });
  const lab = s.map(function (p) { return p[0]; });
  const buy = lab.map(function () { return null; });
  DATI.acquisti.forEach(function (a) {
    const i = lab.findIndex(function (d) { return d >= a[0]; });
    if (i >= 0 && a[0] >= da0 && a[0] >= lab[0]) buy[i] = a[1];
  });
  const ds = [{label: DATI.titolo, data: s.map(function (p) { return p[1]; }),
    borderColor: "#2f6fdb", borderWidth: 1.5, pointRadius: 0, tension: 0}];
  if (DATI.acquisti.length) ds.push({label: "Acquisti", data: buy, showLine: false,
    pointStyle: "triangle", pointRadius: 7, backgroundColor: "#d9822b", borderColor: "#d9822b"});
  if (chart) chart.destroy();
  chart = new Chart(document.getElementById("c"), {type: "line",
    data: {labels: lab, datasets: ds},
    options: {responsive: true, maintainAspectRatio: false, animation: false,
      interaction: {mode: "nearest", axis: "x", intersect: false},
      plugins: {legend: {display: DATI.acquisti.length > 0, labels: {color: txt}}},
      scales: {x: {ticks: {color: txt, maxTicksLimit: 8}, grid: {color: grid}},
               y: {ticks: {color: txt}, grid: {color: grid}}}}});
}
const bar = document.getElementById("b");
RANGE.forEach(function (r) {
  const b = document.createElement("button");
  b.textContent = r.n;
  b.style.cssText = "margin:0 4px 6px 0;padding:2px 10px;cursor:pointer";
  b.onclick = function () { disegna(r); };
  bar.appendChild(b);
});
disegna(RANGE[3]);
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
    html = '<div id="b"></div><div style="position:relative;height:300px"><canvas id="c"></canvas></div>',
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
local function specValore(ids)
  local e, v, c = {}, {}, {}
  for _, d in ipairs(investimenti.valorizzazioneDati(ids)) do
    if d.valore and d.valore > 0 then
      table.insert(e, d.nome)
      table.insert(v, d.valore)
      table.insert(c, COLORI[(#e - 1) % #COLORI + 1])
    end
  end
  if #v > 0 then return {titolo = "Valore per strumento (EUR)", etichette = e, valori = v, colori = c} end
end

-- Dati della ciambella per classe di rischio (attributo KID), o nil.
local function specRischio(ids)
  local rischio = {}
  for _, s in ipairs(investimenti.strumenti()) do rischio[s.id] = s.rischio end
  local per = {}   -- classe (0 = non indicata) -> valore
  for _, d in ipairs(investimenti.valorizzazioneDati(ids)) do
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
  local sp = specValore(ids)
  if not sp then return widget.markdownBlock("_Nessun valore da mostrare._") end
  return torte({sp})
end

--- Ciambella del valore attuale per classe di rischio (attributo KID 1-7 degli strumenti).
function investimenti.tortaRischio(pagina)
  local ids = investimenti.elenco(pagina)
  if ids and #ids == 0 then return widget.markdownBlock("_Nessuno strumento._") end
  local sp = specRischio(ids)
  if not sp then return widget.markdownBlock("_Nessun valore da mostrare._") end
  return torte({sp})
end

--- Le due ciambelle (valore per strumento, valore per rischio) in un solo widget:
--- affiancate su schermo largo, impilate su schermo stretto. pagina: vedi investimenti.elenco.
function investimenti.allocazione(pagina)
  local ids = investimenti.elenco(pagina)
  if ids and #ids == 0 then return widget.markdownBlock("_Nessuno strumento._") end
  local specs = {}
  for _, sp in ipairs({specValore(ids) or false, specRischio(ids) or false}) do
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
  for _, d in ipairs(investimenti.valorizzazioneDati(ids)) do
    local r = rischio[d.id]
    if r and d.valore and d.valore > 0 then somma, peso = somma + r * d.valore, peso + d.valore end
  end
  if peso == 0 then return "n.d." end
  return investimenti.barraRischio(somma / peso)
end
```

### Performance, barre di P/L e widget automatici

`investimenti.performance` calcola la variazione percentuale a 1 mese (30 giorni), da inizio anno e a 1 anno (365 giorni) con una sola scansione delle quotazioni; per i fondi in USD usa il cambio EURUSD, quindi la variazione è in EUR. `investimenti.barrePL` disegna le barre del P/L per strumento (verde guadagno, rosso perdita). `investimenti.vista` e i listener `hooks:renderTopWidgets` mostrano automaticamente i widget in testa alle pagine, solo se `config.set("investimenti.widgetAutomatici", true)`.

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
  for _, d in ipairs(investimenti.valorizzazioneDati(ids)) do
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
