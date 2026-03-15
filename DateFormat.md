---
name: "Library/MG/DateFormat"
tags: meta/library
pageDecoration.prefix: "⏳ "
---

# date.format(dataISO, formato)

La funzione `date.format`  serve a restituire una data formattata da una data ISO utilizzando un codice per il formato da restituire.
Inoltre verifica se la data ISO è valida, se il giorno supera i limiti del mese, lo corregge al giorno massimo e antepone `~` al risultato.  

## The widget accepts following arguments:

*   `dataISO` - required - A string with a date in ISO format YYYY-MM-DD
*   `formato` - optional - a string that define the forma of the output

## Example of format accepted and output provided

date.format(today) ${date.format(date.today())}

| Cod | Output    | Usato | Descrizione del risultato atteso|
| :---: | :---: | :---: | :--- |
|    | ${date.format(date.today())} | si | Numerica giorno.mese.anno |
| N1 | ${date.format(date.today(),"N1")} | si | Numerica giorno.mese.anno |
| N2 | ${date.format(date.today(),"N2")} | ? | Mese abbreviato e punto|
| N3 | ${date.format(date.today(),"N3")} | ? |Completa con mese in lettere |
| L1 | ${date.format(date.today(),"L1")} | ? | Data con giorno della settimana e mese in lettere |
| C1  | ${date.format(date.today(),"C1")} |  ? | Giorno e mese in numeri |
| C2  | ${date.format(date.today(),"C2")} |  ? | abbreviato, giorno, mese abbr.|
| C3  | ${date.format(date.today(),"C3")} | si | Giorno mese abbr.|
| C4  | ${date.format(date.today(),"C4")} | ? | Giorno della settimana corto, giorno e mese numerico |
| C5  | ${date.format(date.today(),"C5")} | ? |Mese abbreviato e anno | 
| YY  | ${date.format(date.today(),"YY")} | si | Solo l’anno |
| M1  | ${date.format(date.today(),"M1")} | ? | Mese esteso |
| M2  | ${date.format(date.today(),"M2")} |  ? | Mese abbrevito a 3 lettere |
| M3  | ${date.format(date.today(),"M3")} |  ? | Giorno abbreviazione a 2 lettere |
| DD  | ${date.format(date.today(),"DD")} |  ? | Giorno con due cifre |
| D1  | ${date.format(date.today(),"D1")} |  ? | Giorno settimana esteso |
| D2  | ${date.format(date.today(),"D2")} |  ? |Giorno settimana abbreviato |
| D3  | ${date.format(date.today(),"D3")} |  ? | Giorno settimana cortissimo |
| WN  | ${date.format(date.today(),"WN")} |  ? | Numero della settimana |
| ND  | ${date.format(date.today(),"ND")} |  ? |Numero giorno dell’anno |
|  | 2025-02-32 => ${date.format("2025-02-32", "N1")} | ? | Data ISO errata restituisce circa GG.MM.AAAA|

## Implementation
  
## date.format(dataISO, formato)
```space-lua
-- function date.format(dataISO, formato)
-- Version: 1.0
local date = date or {}
function date.format(dataISO, formato)
    formato = formato or "N1"
    local anno, mese, giorno = dataISO:match("(%d%d%d%d)%-(%d%d)%-(%d%d)")
    if not anno then return "no data" end
    -- if not mese then return anno end
    anno = tonumber(anno)
    mese = tonumber(mese)
    giorno = tonumber(giorno)

    local giorno_max = giorni_del_mese(anno, mese)
    local errore = false
    if giorno > giorno_max then
        giorno = giorno_max
        errore = true
    end

    local wday = giorno_della_settimana(anno, mese, giorno)
    local giorno_sett = giorni_settimana[wday + 1]
    local giorno_abbr = giorni_abbr[wday + 1]
    local giorno_short = giorni_short[wday + 1]

    local mese_nome = mesi[mese]
    local mese_abbr = mesi_abbr[mese]
    local mese_short = mesi_short[mese]

    local nd = tostring(giorno_dell_anno(anno, mese, giorno))
    local wn = numero_settimana(anno, mese, giorno)

    local format_map = {
        N1 = string.format("%02d.%02d.%s", giorno, mese, anno),
        N2 = string.format("%d %s. %s", giorno, mese_abbr, anno),
        N3 = string.format("%d %s %s", giorno, mese_nome, anno),
        L1 = string.format("%s %d %s %s", giorno_sett, giorno, mese_nome, anno),
        C1 = string.format("%02d.%02d", giorno, mese),
        C2 = string.format("%s. %02d %s", giorno_abbr, giorno, mese_abbr),
        C3 = string.format("%02d %s.", giorno, mese_abbr),
        C4 = string.format("%s %02d.%02d", giorno_short, giorno, mese),
        C5 = string.format("%s %s", mese_abbr, anno),
        YY = tostring(anno),
        M1 = mese_nome,
        M2 = mese_abbr,
        M3 = mese_short,
        DD = string.format("%02d", giorno),
        D1 = giorno_sett,
        D2 = giorno_abbr,
        D3 = giorno_short,
        WN = wn,
        ND = nd
    }

    local risultato = format_map[formato] or ""
    if errore then
        risultato = "~" .. risultato
    end
    return risultato
end
-- -------------------------
-- Tabelle di localizzazione
local mesi = {
    "gennaio", "febbraio", "marzo", "aprile", "maggio", "giugno",
    "luglio", "agosto", "settembre", "ottobre", "novembre", "dicembre"
}

local mesi_abbr = {
    "gen", "feb", "mar", "apr", "mag", "giu",
    "lug", "ago", "set", "ott", "nov", "dic"
}

local mesi_short = {
    "Ge", "Fe", "Ma", "Ap", "Ma", "Gi",
    "Lu", "Ag", "Se", "Ot", "No", "Di"
}

local giorni_settimana = {
    "domenica", "lunedì", "martedì", "mercoledì", "giovedì", "venerdì", "sabato"
}

local giorni_abbr = {
    "dom", "lun", "mar", "mer", "gio", "ven", "sab"
}

local giorni_short = {
    "Do", "Lu", "Ma", "Me", "Gi", "Ve", "Sa"
}

local function giorni_del_mese(anno, mese)
    local giorni = { 31,28,31,30,31,30,31,31,30,31,30,31 }
    if (anno % 4 == 0 and anno % 100 ~= 0) or (anno % 400 == 0) then
        giorni[2] = 29
    end
    return giorni[mese]
end

local function giorno_della_settimana(anno, mese, giorno)
    if mese < 3 then
        mese = mese + 12
        anno = anno - 1
    end
    local k = anno % 100
    local j = math.floor(anno / 100)
    local h = (giorno + math.floor((13 * (mese + 1)) / 5) + k + math.floor(k / 4) + math.floor(j / 4) + 5 * j) % 7
    return (h + 6) % 7
end

local function giorno_dell_anno(anno, mese, giorno)
    local somma = 0
    for i = 1, mese - 1 do
        somma = somma + giorni_del_mese(anno, i)
    end
    return somma + giorno
end

local function numero_settimana(anno, mese, giorno)
    local giorno_anno = giorno_dell_anno(anno, mese, giorno)
    local primo_giorno = giorno_della_settimana(anno, 1, 1)
    local offset = (primo_giorno + 6) % 7
    return string.format("%02d", math.floor((giorno_anno + offset - 1) / 7) + 1)
end
```

## More Examples & Discussions
