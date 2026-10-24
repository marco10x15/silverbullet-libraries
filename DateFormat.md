---
name: "Library/MG/DateFormat"
tags: meta/library
pageDecoration.prefix: "⏳ "
version: 0.7
share.uri: "github:marco10x15/silverbullet-libraries/DateFormat.md"
---

# date.format(dataISO, formato)

`date.format()` formatta una data ISO `YYYY-MM-DD` utilizzando un pattern leggibile.

Il sistema canonico utilizza formati come:

```
date.format("2026-09-25", "DD.MM.YYYY")
date.format("2026-09-25", "D mmmm YYYY")
date.format("2026-09-25", "mmm YYYY")
```

Per compatibilità rimangono disponibili anche tutti i codici legacy precedenti, come `N1`, `N2`, `C5`, `D1`, ecc.

## Utilizzo

```
date.format(dataISO, formato)
```

Parametri:

*   `dataISO`: obbligatorio, data nel formato ISO `YYYY-MM-DD`;
*   `formato`: opzionale; se omesso viene utilizzato `DD.MM.YYYY`.

Esempi:

`date.format("2026-09-25")` restituisce: `25.09.2026`

`date.format("2026-09-25", "mmm YYYY")` restituisce: `set 2026`

`date.format("2026-09-25", "dddd D mmmm YYYY")` restituisce: `venerdì 25 settembre 2026`

## Pattern disponibili

| Pattern | Esempio | Descrizione |
| --- | --- | --- |
| `D` | `5` | Giorno senza zero iniziale |
| `DD` | `05` | Giorno a due cifre |
| `dd` | `Lu` | Giorno settimana corto |
| `ddd` | `lun` | Giorno settimana abbreviato |
| `dddd` | `lunedì` | Giorno settimana esteso |
| `M` | `9` | Mese senza zero iniziale |
| `MM` | `09` | Mese a due cifre |
| `mmm` | `set` | Mese abbreviato |
| `mmmm` | `settembre` | Mese esteso |
| `YYYY` | `2026` | Anno a quattro cifre |
| `WW` | `39` | Numero settimana |
| `DDD` | `268` | Numero progressivo del giorno nell'anno |

I token possono essere combinati liberamente.

Esempi:

```
DD.MM.YYYY
YYYY-MM-DD
D mmmm YYYY
ddd DD mmm
mmm YYYY
DD/MM/YYYY
```

## Compatibilità legacy

I vecchi codici continuano a essere supportati.

| Legacy | Pattern equivalente |
| --- | --- |
| `N1` | `DD.MM.YYYY` |
| `N2` | `D mmm. YYYY` |
| `N3` | `D mmmm YYYY` |
| `L1` | `dddd D mmmm YYYY` |
| `C1` | `DD.MM` |
| `C2` | `ddd. DD mmm` |
| `C3` | `DD mmm.` |
| `C4` | `dd DD.MM` |
| `C5` | `mmm YYYY` |
| `YY` | `YYYY` |
| `M1` | `mmmm` |
| `M2` | `mmm` |
| `M3` | formato legacy mese a due lettere |
| `DD` | `DD` |
| `D1` | `dddd` |
| `D2` | `ddd` |
| `D3` | `dd` |
| `WN` | `WW` |
| `ND` | `DDD` |

`YY` mantiene intenzionalmente il comportamento legacy e restituisce ancora l'anno a quattro cifre.

`M3` rimane un formato esclusivamente legacy perché utilizza la particolare abbreviazione del mese a due lettere della vecchia libreria.

## Gestione date non valide

Se il giorno supera il massimo previsto per il mese, viene corretto al giorno massimo e il risultato viene preceduto da `~`.

Esempio:

```
date.format("2025-02-32", "DD.MM.YYYY")
```

restituisce:

```
~28.02.2025
```

Mese non valido, giorno `00` o input non conforme a `YYYY-MM-DD` restituiscono:

```
no data
```

# date.diff(dataA, dataB)

`date.diff()` calcola la differenza tra due date ISO `YYYY-MM-DD` e la restituisce come tabella con i campi `anni`, `mesi` e `giorni`. L'ordine dei parametri è indifferente: le date vengono ordinate automaticamente. Se `dataB` è omessa viene utilizzata la data odierna. Il calcolo non usa `os.time` e quindi non dipende da fuso orario o ora legale. Se una delle due date non è conforme a `YYYY-MM-DD`, restituisce `nil`.

Si usano prima i mesi interi, poi i giorni residui. Se il giorno iniziale non esiste nel mese di riferimento, viene limitato all'ultimo giorno di quel mese.

Esempi:

`date.diff("2020-03-15", "2026-10-19")` restituisce: `{anni = 6, mesi = 7, giorni = 4}`

`date.diff("2026-01-31", "2026-03-01")` restituisce: `{anni = 0, mesi = 1, giorni = 1}`

# date.days(dataA, dataB)

`date.days()` restituisce il numero di giorni tra due date ISO `YYYY-MM-DD`, con segno: positivo se `dataB` è successiva a `dataA`, negativo altrimenti. Se `dataB` è omessa viene utilizzata la data odierna. Non usa `os.time`. Con date non valide restituisce `nil`.

Esempi:

`date.days("2020-03-15", "2026-10-19")` restituisce: `2409`

`date.days("2026-10-19", "2026-10-12")` restituisce: `-7`

# date.diffText(dataA, dataB, formato, compatta)

`date.diffText()` restituisce la differenza tra due date ISO come testo leggibile. L'ordine delle date è indifferente. Le parti uguali a zero vengono omesse.

## Utilizzo

```
date.diffText(dataA)
date.diffText(dataA, dataB)
date.diffText(dataA, dataB, formato)
date.diffText(dataA, dataB, formato, compatta)
```

Parametri:

*   `dataA`: obbligatorio, data nel formato ISO `YYYY-MM-DD`;
*   `dataB`: opzionale; se omessa (o `nil`) viene utilizzata la data odierna;
*   `formato`: opzionale; se omesso o non riconosciuto viene utilizzato `amg`;
*   `compatta`: opzionale; `1` restituisce la forma compatta, `0` la forma estesa. Se omesso, la forma è compatta quando `dataB` è omessa (differenza rispetto a oggi), estesa altrimenti.

Per indicare `formato` o `compatta` con la data odierna, passare `nil` come `dataB`:

```
date.diffText("1965-12-18", nil, "a")
date.diffText("1965-12-18", nil, "amg", 0)
```

restituiscono, al 19.10.2026: `60a` e `60 anni, 10 mesi e 1 giorno`.

## Formati

Esempi calcolati tra `2020-03-15` e `2026-10-19`:

| Formato | Forma estesa | Forma compatta | Descrizione |
| --- | --- | --- | --- |
| `amg` | `6 anni, 7 mesi e 4 giorni` | `6a 7m 4g` | Anni, mesi e giorni (default) |
| `am` | `6 anni e 7 mesi` | `6a 7m` | Anni e mesi |
| `a` | `6 anni` | `6a` | Solo anni |
| `mg` | `79 mesi e 4 giorni` | `79m 4g` | Mesi totali e giorni |
| `m` | `79 mesi` | `79m` | Solo mesi totali |
| `g` | `2409 giorni` | `2409g` | Giorni totali |
| `wg` | `344 settimane e 1 giorno` | `344s 1g` | Settimane totali e giorni |
| `w` | `344 settimane` | `344s` | Solo settimane totali |

I formati ridotti troncano senza arrotondare: con `am`, 6 anni, 7 mesi e 29 giorni diventano `6 anni e 7 mesi`.

Un formato non riconosciuto viene trattato come `amg`: un errore di battitura non genera errori ma un output diverso da quello atteso.

Esempi:

`date.diffText("1965-12-18", "2026-10-19")` restituisce: `60 anni, 10 mesi e 1 giorno`

`date.diffText("1965-12-18", "2026-10-19", "amg", 1)` restituisce: `60a 10m 1g`

`date.diffText("1965-12-18")` restituisce, al 19.10.2026: `60a 10m 1g`

`date.diffText("2020-03-15", "2026-03-19")` restituisce: `6 anni e 4 giorni`

## Casi particolari

*   Date uguali: `0` con l'unità più piccola del formato (es. `0 giorni`, o `0g` in forma compatta, con `amg`).
*   Dal 29 febbraio a un 28 febbraio di un anno non bisestile: `11 mesi e 30 giorni`, non `1 anno`.
*   Le date vengono ordinate: una data futura restituisce lo stesso testo di una passata, senza segno. Per conoscere il verso usare `date.days()`.
*   Date non conformi a `YYYY-MM-DD`: `no data`, come `date.format()`.
*   La data odierna è quella locale del dispositivo su cui viene eseguito il codice.


# date.addMonths(dataISO, n)
`date.addMonths()` somma `n` mesi (anche negativi) a una data ISO `YYYY-MM-DD` e restituisce la nuova data ISO. Se il giorno non esiste nel mese risultante viene limitato all'ultimo giorno del mese. Non usa `os.time`. Con date non valide o `n` non numerico restituisce `nil`.

Esempi:
`date.addMonths("2026-08-31", 6)` restituisce: `2027-02-28`
`date.addMonths("2027-11-30", 3)` restituisce: `2028-02-29`
`date.addMonths("2026-10-15", 48)` restituisce: `2030-10-15`
`date.addMonths("2026-03-31", -1)` restituisce: `2026-02-28`

## Implementation

```space-lua
date = date or {}


-- ============================================================
-- LOCALIZZAZIONE
-- ============================================================

local mesi = {
  "gennaio",
  "febbraio",
  "marzo",
  "aprile",
  "maggio",
  "giugno",
  "luglio",
  "agosto",
  "settembre",
  "ottobre",
  "novembre",
  "dicembre",
}

local mesi_abbr = {
  "gen",
  "feb",
  "mar",
  "apr",
  "mag",
  "giu",
  "lug",
  "ago",
  "set",
  "ott",
  "nov",
  "dic",
}

-- Formato storico utilizzato esclusivamente da M3.
local mesi_short = {
  "Ge",
  "Fe",
  "Ma",
  "Ap",
  "Ma",
  "Gi",
  "Lu",
  "Ag",
  "Se",
  "Ot",
  "No",
  "Di",
}

local giorni_settimana = {
  "domenica",
  "lunedì",
  "martedì",
  "mercoledì",
  "giovedì",
  "venerdì",
  "sabato",
}

local giorni_abbr = {
  "dom",
  "lun",
  "mar",
  "mer",
  "gio",
  "ven",
  "sab",
}

local giorni_short = {
  "Do",
  "Lu",
  "Ma",
  "Me",
  "Gi",
  "Ve",
  "Sa",
}


-- ============================================================
-- CALCOLI DATA
-- ============================================================

-- Restituisce il numero di giorni del mese.
local function giorni_del_mese(
  anno,
  mese
)
  local giorni = {
    31,
    28,
    31,
    30,
    31,
    30,
    31,
    31,
    30,
    31,
    30,
    31,
  }

  if (
    anno % 4 == 0
    and anno % 100 ~= 0
  )
    or anno % 400 == 0
  then
    giorni[2] = 29
  end

  return giorni[mese]
end


-- Restituisce il giorno della settimana:
--
-- 0 = domenica
-- 1 = lunedì
-- ...
-- 6 = sabato
local function giorno_della_settimana(
  anno,
  mese,
  giorno
)
  if mese < 3 then
    mese =
      mese + 12

    anno =
      anno - 1
  end

  local k =
    anno % 100

  local j =
    math.floor(
      anno / 100
    )

  local h =
    (
      giorno
      + math.floor(
        (13 * (mese + 1)) / 5
      )
      + k
      + math.floor(k / 4)
      + math.floor(j / 4)
      + 5 * j
    ) % 7

  return (h + 6) % 7
end


-- Restituisce il numero progressivo del giorno nell'anno.
local function giorno_dell_anno(
  anno,
  mese,
  giorno
)
  local somma = 0

  for i = 1, mese - 1 do
    somma =
      somma
      + giorni_del_mese(
        anno,
        i
      )
  end

  return somma + giorno
end


-- Restituisce il numero della settimana mantenendo
-- il calcolo utilizzato dalla versione precedente.
local function numero_settimana(
  anno,
  mese,
  giorno
)
  local giorno_anno =
    giorno_dell_anno(
      anno,
      mese,
      giorno
    )

  local primo_giorno =
    giorno_della_settimana(
      anno,
      1,
      1
    )

  local offset =
    (
      primo_giorno
      + 6
    ) % 7

  return string.format(
    "%02d",
    math.floor(
      (
        giorno_anno
        + offset
        - 1
      ) / 7
    ) + 1
  )
end


-- ============================================================
-- COMPATIBILITÀ LEGACY
-- ============================================================

-- I codici legacy vengono convertiti nei nuovi pattern prima
-- della formattazione.
--
-- M3 viene gestito separatamente perché rappresenta una
-- particolare abbreviazione storica del mese a due lettere.
local legacyFormats = {
  N1 = "DD.MM.YYYY",
  N2 = "D mmm. YYYY",
  N3 = "D mmmm YYYY",
  L1 = "dddd D mmmm YYYY",

  C1 = "DD.MM",
  C2 = "ddd. DD mmm",
  C3 = "DD mmm.",
  C4 = "dd DD.MM",
  C5 = "mmm YYYY",

  YY = "YYYY",

  M1 = "mmmm",
  M2 = "mmm",

  DD = "DD",

  D1 = "dddd",
  D2 = "ddd",
  D3 = "dd",

  WN = "WW",
  ND = "DDD",
}


-- ============================================================
-- FORMATTER
-- ============================================================

-- Restituisce il valore del token richiesto.
local function dateFormatToken(
  token,
  anno,
  mese,
  giorno,
  giorno_sett,
  giorno_abbr,
  giorno_short,
  mese_nome,
  mese_abbr,
  wn,
  nd
)
  if token == "YYYY" then
    return tostring(anno)
  end

  if token == "mmmm" then
    return mese_nome
  end

  if token == "mmm" then
    return mese_abbr
  end

  if token == "MM" then
    return string.format(
      "%02d",
      mese
    )
  end

  if token == "M" then
    return tostring(mese)
  end

  if token == "dddd" then
    return giorno_sett
  end

  if token == "ddd" then
    return giorno_abbr
  end

  if token == "dd" then
    return giorno_short
  end

  if token == "DDD" then
    return nd
  end

  if token == "DD" then
    return string.format(
      "%02d",
      giorno
    )
  end

  if token == "D" then
    return tostring(giorno)
  end

  if token == "WW" then
    return wn
  end

  return nil
end


-- Interpreta il pattern senza modificare i caratteri che non
-- appartengono a un token.
--
-- I token più lunghi vengono verificati prima di quelli più corti
-- per evitare sovrapposizioni come:
--
-- mmmm / mmm / MM / M
-- dddd / ddd / dd
-- DDD / DD / D
local function dateFormatPattern(
  formato,
  anno,
  mese,
  giorno,
  giorno_sett,
  giorno_abbr,
  giorno_short,
  mese_nome,
  mese_abbr,
  wn,
  nd
)
  local tokens = {
    "YYYY",
    "mmmm",
    "dddd",
    "mmm",
    "ddd",
    "DDD",
    "MM",
    "DD",
    "WW",
    "dd",
    "M",
    "D",
  }

  local risultato = {}
  local posizione = 1

  while posizione <= #formato do
    local trovato = false

    for _, token in ipairs(tokens) do
      if string.sub(
        formato,
        posizione,
        posizione + #token - 1
      ) == token
      then
        table.insert(
          risultato,
          dateFormatToken(
            token,
            anno,
            mese,
            giorno,
            giorno_sett,
            giorno_abbr,
            giorno_short,
            mese_nome,
            mese_abbr,
            wn,
            nd
          )
        )

        posizione =
          posizione + #token

        trovato = true

        break
      end
    end

    if not trovato then
      table.insert(
        risultato,
        string.sub(
          formato,
          posizione,
          posizione
        )
      )

      posizione =
        posizione + 1
    end
  end

  return table.concat(
    risultato
  )
end


-- ============================================================
-- API PUBBLICA
-- ============================================================

-- Formatta una data ISO YYYY-MM-DD.
--
-- Il formato canonico utilizza pattern leggibili:
--
-- DD.MM.YYYY
-- D mmmm YYYY
-- dddd D mmmm YYYY
-- mmm YYYY
--
-- Rimangono supportati anche i codici legacy.
function date.format(
  dataISO,
  formato
)
  if type(dataISO) ~= "string" then
    return "no data"
  end

  formato =
    formato
    or "DD.MM.YYYY"

  local anno,
    mese,
    giorno =
      string.match(
        dataISO,
        "^(%d%d%d%d)%-(%d%d)%-(%d%d)$"
      )

  if not anno then
    return "no data"
  end

  anno =
    tonumber(anno)

  mese =
    tonumber(mese)

  giorno =
    tonumber(giorno)

  -- Mese non valido.
  if mese < 1
    or mese > 12
  then
    return "no data"
  end

  -- Il giorno zero non rappresenta una data valida.
  if giorno < 1 then
    return "no data"
  end

  local giorno_max =
    giorni_del_mese(
      anno,
      mese
    )

  local errore = false

  -- Mantiene il comportamento storico:
  -- se il giorno supera il massimo del mese viene corretto
  -- e il risultato viene marcato con "~".
  if giorno > giorno_max then
    giorno =
      giorno_max

    errore = true
  end

  local wday =
    giorno_della_settimana(
      anno,
      mese,
      giorno
    )

  local giorno_sett =
    giorni_settimana[
      wday + 1
    ]

  local giorno_abbr =
    giorni_abbr[
      wday + 1
    ]

  local giorno_short =
    giorni_short[
      wday + 1
    ]

  local mese_nome =
    mesi[mese]

  local mese_abbr =
    mesi_abbr[mese]

  local nd =
    tostring(
      giorno_dell_anno(
        anno,
        mese,
        giorno
      )
    )

  local wn =
    numero_settimana(
      anno,
      mese,
      giorno
    )

  -- M3 rimane un formato speciale esclusivamente legacy.
  if formato == "M3" then
    local risultato =
      mesi_short[mese]

    if errore then
      risultato =
        "~"
        .. risultato
    end

    return risultato
  end

  -- Converte gli altri codici legacy nel pattern equivalente.
  formato =
    legacyFormats[formato]
    or formato

  local risultato =
    dateFormatPattern(
      formato,
      anno,
      mese,
      giorno,
      giorno_sett,
      giorno_abbr,
      giorno_short,
      mese_nome,
      mese_abbr,
      wn,
      nd
    )

  if errore then
    risultato =
      "~"
      .. risultato
  end

  return risultato
end

-- ============================================================
-- DIFFERENZA TRA DATE
-- ============================================================

-- Data odierna ISO, usata quando la seconda data è omessa.
local function oggi()
  return os.date("%Y-%m-%d")
end

-- Numero seriale del giorno (calendario gregoriano), senza os.time/timezone.
local function giorno_seriale(dataISO)
  if type(dataISO) ~= "string" then return nil end
  local y, m, d = string.match(dataISO, "^(%d%d%d%d)%-(%d%d)%-(%d%d)$")
  if not y then return nil end
  y, m, d = tonumber(y), tonumber(m), tonumber(d)
  if m < 1 or m > 12 or d < 1 or d > 31 then return nil end
  if m <= 2 then y = y - 1 end
  local era = math.floor(y / 400)
  local yo = y - era * 400
  local mp = m + (m > 2 and -3 or 9)
  local doy = math.floor((153 * mp + 2) / 5) + d - 1
  return era * 146097 + yo * 365 + math.floor(yo / 4) - math.floor(yo / 100) + doy
end

-- Unità: {singolare, plurale, abbreviazione}.
-- a/m/g = parti di date.diff; M = mesi totali; G = giorni totali;
-- W = settimane totali; r = giorni residui dopo le settimane.
local diffUnita = {
  a = {"anno", "anni", "a"},
  m = {"mese", "mesi", "m"},
  g = {"giorno", "giorni", "g"},
  M = {"mese", "mesi", "m"},
  G = {"giorno", "giorni", "g"},
  W = {"settimana", "settimane", "s"},
  r = {"giorno", "giorni", "g"},
}

local diffFormati = {
  a   = {"a"},
  am  = {"a", "m"},
  amg = {"a", "m", "g"},
  m   = {"M"},
  mg  = {"M", "g"},
  g   = {"G"},
  w   = {"W"},
  wg  = {"W", "r"},
}

-- Giorni tra due date ISO (con segno: positivo se b è successiva ad a).
-- b omessa: data odierna.
function date.days(a, b)
  local x, y = giorno_seriale(a), giorno_seriale(b or oggi())
  return x and y and (y - x) or nil
end

-- Differenza tra due date ISO → {anni, mesi, giorni}, oppure nil.
-- b omessa: data odierna. Convenzione: 31 gen → 1 mar = 1 mese e 1 giorno.
function date.diff(a, b)
  b = b or oggi()
  if type(a) ~= "string" or type(b) ~= "string" then return nil end
  if a > b then a, b = b, a end

  local y1, m1, d1 = string.match(a, "^(%d%d%d%d)%-(%d%d)%-(%d%d)$")
  local y2, m2, d2 = string.match(b, "^(%d%d%d%d)%-(%d%d)%-(%d%d)$")
  if not y1 or not y2 then return nil end
  y1, m1, d1 = tonumber(y1), tonumber(m1), tonumber(d1)
  y2, m2, d2 = tonumber(y2), tonumber(m2), tonumber(d2)

  local mesi = (y2 - y1) * 12 + (m2 - m1)
  local giorni = d2 - d1

  if giorni < 0 then
    mesi = mesi - 1
    -- mese precedente a quello della data finale
    local py, pm = y2, m2 - 1
    if pm == 0 then py, pm = y2 - 1, 12 end
    local max = giorni_del_mese(py, pm)
    giorni = max - math.min(d1, max) + d2
  end

  return {anni = math.floor(mesi / 12), mesi = mesi % 12, giorni = giorni}
end

-- Somma n mesi (anche negativi) a una data ISO; il giorno viene limitato
-- all'ultimo giorno del mese risultante. nil se la data non è valida.
function date.addMonths(dataISO, n)
  if type(dataISO) ~= "string" or type(n) ~= "number" then return nil end
  local anno, mese, giorno = string.match(dataISO, "^(%d%d%d%d)%-(%d%d)%-(%d%d)$")
  if not anno then return nil end
  anno, mese, giorno = tonumber(anno), tonumber(mese), tonumber(giorno)
  if mese < 1 or mese > 12 or giorno < 1 then return nil end
  local totale = anno * 12 + (mese - 1) + n
  anno, mese = math.floor(totale / 12), totale % 12 + 1
  return string.format("%04d-%02d-%02d", anno, mese,
    math.min(giorno, giorni_del_mese(anno, mese)))
end

-- Differenza in testo; le parti a zero vengono omesse.
-- b omessa: data odierna. formato omesso o sconosciuto: "amg".
-- compatta = 1: "6a 7m 4g"; 0: "6 anni, 7 mesi e 4 giorni".
-- compatta omessa: compatta se b è omessa, estesa altrimenti.
function date.diffText(a, b, formato, compatta)
  if compatta == nil then compatta = b == nil and 1 or 0 end
  b = b or oggi()
  local r = date.diff(a, b)
  local giorni = date.days(a, b)
  if not r or not giorni then return "no data" end

  local chiavi = diffFormati[formato or "amg"] or diffFormati.amg
  compatta = compatta == 1 or compatta == true

  giorni = math.abs(giorni)
  local v = {
    a = r.anni, m = r.mesi, g = r.giorni,
    M = r.anni * 12 + r.mesi,
    G = giorni,
    W = math.floor(giorni / 7),
    r = giorni % 7,
  }

  local parti = {}
  for _, k in ipairs(chiavi) do
    local n, u = v[k], diffUnita[k]
    if n ~= 0 then
      table.insert(parti, compatta and (n .. u[3]) or (n .. " " .. (n == 1 and u[1] or u[2])))
    end
  end

  -- Tutto a zero: unità più piccola del formato
  if #parti == 0 then
    local u = diffUnita[chiavi[#chiavi]]
    return compatta and ("0" .. u[3]) or ("0 " .. u[2])
  end

  if compatta then return table.concat(parti, " ") end
  if #parti == 1 then return parti[1] end

  local ultima = table.remove(parti)
  return table.concat(parti, ", ") .. " e " .. ultima
end
```

## Esempi

### Nuovo sistema

```
date.format("2026-09-25", "DD.MM.YYYY")
```

`25.09.2026`

```
date.format("2026-09-25", "YYYY-MM-DD")
```
