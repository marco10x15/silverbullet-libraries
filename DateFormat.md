---
name: "Library/MG/DateFormat"
tags: meta/library
pageDecoration.prefix: "⏳ "
share.uri: "github:marco10x15/silverbullet-libraries/DateFormat.md"
---
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
