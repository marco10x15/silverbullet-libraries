---
name: Library/MG/stampa
version: 0.08.2
versionDate: 2026-10-22
tags: meta/library
---

# Stampa — HTML e PDF da Markdown

Libreria Space Lua per **SilverBullet 2.12.x**. Genera un'anteprima HTML A4/A3 della pagina corrente e consente di stampare su carta o di salvare in PDF tramite il browser. Il Markdown nello Space **non viene modificato**.

**Versione:** 0.08.2 (scelta del formato A4/A3 e orientamento; mantiene invariata la pipeline 0.08.1). Sviluppata con significativo supporto di AI; da validare nell'istanza SilverBullet 2.12.1.

## Installazione e dipendenze

- Pubblicare questo file nel repository `marco10x15/silverbullet-libraries` come `stampa.md` (radice del repository), con nome interno `Library/MG/stampa`.
- Installarlo con il Library Manager di SilverBullet e verificare che la libreria sia attiva nello Space desiderato.
- Per stampare i diagrammi Mermaid in SVG, installare la libreria [`silverbulletmd/silverbullet-diagram/Mermaid.md`](https://github.com/silverbulletmd/silverbullet-diagram/blob/main/Mermaid.md), aggiungendo la funzione `mermaid.toSVG()` verificata durante i test. La versione upstream ordinaria potrebbe non esporre tale funzione.
- Per stampare formule `$...$` e `$$...$$` installare `Library/mrmugame/Silverbullet-Math` e i suoi file KaTeX associati. La libreria stampa riutilizza `latex.katex.renderToString()` e carica il CSS locale di KaTeX.
- Non sono richiesti Paged.js, un servizio Docker o Chart.js per le funzionalità attuali.

Se la pagina contiene un blocco Mermaid ma `mermaid.toSVG()` non è disponibile, il convertitore inserisce un messaggio d'errore al posto del diagramma: la libreria continua a usare il browser per la stampa.

## Uso

1. Aprire la pagina che si desidera stampare.
2. Avviare il comando **`Stampa: Anteprima Markdown`** dalla palette.
3. Selezionare il formato di stampa (A4 verticale predefinito, A4 orizzontale, A3 verticale oppure A3 orizzontale).
4. Selezionare la modalità di esportazione del frontmatter; in modalità **Standard + selezione** attivare o disattivare i campi nel menu `☐/☑`, poi scegliere **Conferma selezione**.
5. Se i tag sono selezionati, scegliere come presentarli.
6. Si apre una nuova scheda HTML: controllare il documento e usare **Ctrl+P** per stampare o salvare in PDF.

L'operazione può essere annullata dai menu. Se il browser blocca i popup, occorre autorizzare l'apertura della scheda per lo Space SilverBullet.

## Modalità frontmatter

| Modalità | Contenuto |
|---|---|
| Standard (predefinita) | `displayName` come H1, `description` come abstract *senza etichetta*, `date` come data italiana, se presenti |
| Standard + selezione | Come Standard, più i campi aggiuntivi spuntati |
| Tutti i campi YAML | I tre campi standard più tutti gli altri attributi dichiarati nel YAML |
| Solo contenuto | Ignora completamente il frontmatter |

I campi selezionabili derivano dalle **chiavi top-level effettivamente scritte nel blocco YAML**, non da metadati tecnici calcolati dal parser: `range` non verrà proposto se non esiste nel sorgente. `displayName`, `description` e `date` non vengono ripetuti tra gli altri metadati. Non vengono stampate schede speciali ADR/Persona: i relativi campi, se presenti, possono essere scelti come dati ordinari.

### Regole editoriali

- **Titolo:** quando `displayName` è presente, l'intestazione utilizza H1 e i titoli del corpo scendono di un livello (fino a H6). Se il primo H1 del corpo è identico al titolo, il duplicato viene soppresso.
- **Abstract:** `description` è convertita da Markdown e stampata in corsivo con un discreto bordo verticale, **senza la parola “Abstract”**.
- **Data:** le stringhe ISO `YYYY-MM-DD` vengono rese nel formato italiano `D mese YYYY` (per esempio `21 ottobre 2026`). Date non riconosciute restano inalterate.
- **Tag:** quattro formati: separati da virgola (consigliato), `#hashtag`, etichette grafiche oppure campo metadati. Per stringhe YAML scalari, i tag sono conservati come un unico valore; per array vengono separati correttamente.
- **Altri valori:** stringhe, liste, booleani e oggetti YAML vengono convertiti in testo HTML leggibile; i wiki-link vengono mostrati tramite etichetta leggibile.

## Formattazione A4/A3

Formato selezionabile a ogni stampa: **A4 verticale** (predefinito), **A4 orizzontale**, **A3 verticale**, **A3 orizzontale**. Margini **20 mm** per lato;  Arial/Helvetica **11 pt**, interlinea **1,25**; titoli blu ardesia `#243c56`, interlinea **1,5** e maggiore spazio sopra i titoli. Anteprima browser su foglio bianco centrato e sfondo grigio; ombre e sfondo non compaiono sulla stampa. Il CSS è incorporato nella libreria.

Nel dialogo di stampa del browser scegliere anche il medesimo formato carta e orientamento selezionati in SilverBullet; utilizzare scala 100%, intestazioni/piè di pagina del browser disattivati e margini del browser disattivati, perché i margini sono gestiti da `@page`.

## Pipeline e contenuti supportati

`editor.getText()` → scelta formato pagina → `index.extractFrontmatter()` → scelta frontmatter → `markdown.bakeSections()` → `markdown.expandMarkdown()` → conversione fence Mermaid con `mermaid.toSVG()` e formule KaTeX → `markdown.markdownToHtml()` → HTML con CSS A4 → finestra browser.

Supportati nei test: Markdown ordinario e multipagina, tabelle lunghe, immagini locali, risultati Lua che generano Markdown (incluse sezioni baked), blocchi di codice e diagrammi Mermaid statici in SVG. Per i diagrammi vengono riconosciuti i blocchi fenced a tre backtick con linguaggio `mermaid`, sulla propria riga; sono possibili limiti con varianti di fence o annidamenti.

## Evidenziazioni e attributi inline (v0.07)

- `==testo==`: stile giallo tenue `#fff2b3` per i tag HTML `mark` e per le classi di evidenziazione emesse dal convertitore.
- `[campo: valore]`: stile giallo chiarissimo `#fff7d6` quando l'HTML generato include `.sb-attribute`; viene enfatizzato anche il nome del campo, se il renderer lo marca separatamente.
- Gli stili non interpretano né modificano il Markdown sorgente. Se il convertitore HTML emette soltanto testo semplice senza tag dedicati, il CSS non può ricostruire l'evidenziazione: in tal caso occorre controllare l'HTML concreto e aggiungere una conversione mirata. Il contenuto dentro i blocchi di codice non viene trasformato.
- Per Firefox, abilitare **Stampa sfondi** nel dialogo di stampa, se lo sfondo giallo non compare nel PDF.

## Limiti conosciuti

- I **widget live** `widget.new` che dipendono dall'editor possono produrre il testo `This view requires the live editor.`: non vengono automaticamente catturati.
- Widget DOM, stili `space-style` dell'editor e i canvas **Chart.js** non vengono automaticamente riprodotti nell'HTML indipendente.
- Il baking rivaluta le sezioni compatibili con la conversione statica; non è un sostituto universale dell'editor live.
- Checklist e riferimenti ai task possono conservare identificativi tecnici generati da SilverBullet.
- Il riconoscimento delle chiavi YAML è limitato ai nomi top-level semplici; non è un editor YAML completo.
- La selezione dei metadati non è memorizzata tra due stampe.
- Il rendering dipende dal browser; il layout e le interruzioni di pagina possono differire tra motori di stampa.

## API pubblica

`stampa.anteprima()` apre la procedura di scelta ed esportazione. Non invocare questa funzione all'interno di `${...}` in una pagina Markdown, perché le espressioni vengono rivalutate dal rendering dell'editor. Utilizzare esclusivamente il comando esplicito **Stampa: Anteprima Markdown**.

## Cronologia

- **0.08.2 — 22.10.2026:** scelta A4/A3 verticale/orizzontale prima delle opzioni frontmatter, adeguamento CSS `@page` e dimensioni di anteprima; nessuna modifica alla conversione Markdown, Mermaid o KaTeX.

- **0.07 — 22.10.2026:** aggiunto CSS per evidenziazioni `==...==` e attributi inline `[campo: valore]`, senza modificare la pipeline già validata.
- **0.06 — 21.10.2026:** consolidamento; rimozione delle funzioni e dei comandi di test, mantenendo selezione multipla dei campi YAML, conversione Mermaid e CSS A4 validati. Namespace pubblico `stampa`.

## Implementazione Space Lua

```space-lua
stampa = stampa or {}

local CSS_STAMPA = [==[
/* SilverBullet 2.12.x — Stampa HTML, profilo approvato v0.4 */
@page { size: A4 portrait; margin: 20mm; }

:root { color-scheme: light; }
html { -webkit-print-color-adjust: exact; print-color-adjust: exact; }
body {
  margin: 0;
  background: #fff;
  color: #242b34;
  font-family: "Arial", "Helvetica Neue", sans-serif;
  font-size: 11pt;
  line-height: 1.25;
  overflow-wrap: anywhere;
}
/* Titoli: circa una riga libera prima, interlinea 1,5 */
h1, h2, h3, h4 {
  color: #243c56;
  font-weight: 650;
  line-height: 1.5;
  break-after: avoid;
  page-break-after: avoid;
}
h1 { font-size: 20pt; margin: 0 0 0.65em; padding-bottom: 0.22em; border-bottom: 1px solid #cbd6e1; }
h2 { font-size: 15pt; margin: 1.5em 0 0.55em; }
h3 { font-size: 12.5pt; margin: 1.5em 0 0.45em; }
h4 { font-size: 11pt; margin: 1.5em 0 0.4em; }
p { margin: 0 0 0.65em; orphans: 3; widows: 3; }
/* SilverBullet può emettere span.p intervallati da <br>. */
span.p { line-height: inherit; }
strong { font-weight: 700; }
/* Evidenziazione Markdown ==testo== e HTML <mark>. */
mark, .sb-highlight, .sb-mark {
  background-color: #fff2b3;
  color: #242b34;
  padding: .05em .12em;
  border-radius: 2px;
  -webkit-box-decoration-break: clone;
  box-decoration-break: clone;
}
/* Attributi inline [campo: valore] riconosciuti dal renderer HTML. */
.sb-attribute {
  background-color: #fff7d6;
  color: #3e3930;
  border: 1px solid #f1e4a6;
  border-radius: 3px;
  padding: .05em .2em;
  -webkit-box-decoration-break: clone;
  box-decoration-break: clone;
}
.sb-attribute-name, .sb-attribute .sb-atom {
  font-weight: 600;
  color: #68521b;
}
/* Evita che un eventuale stile dell'editor colori l'attributo interno. */
.sb-attribute .sb-meta { color: inherit; }

a { color: #245a88; text-decoration: underline; text-decoration-color: #a4b4c5; }
ul, ol { margin: 0.4em 0 0.8em; padding-left: 1.45em; }
li { margin: 0.2em 0; }
li > ul, li > ol { margin: 0.25em 0; }
blockquote { margin: 0.8em 0 1em; padding: 0.35em 0.85em; border-left: 3px solid #93a9bf; color: #465565; background: #f6f8fa; }
hr { border: 0; border-top: 1px solid #ccd5dd; margin: 1.2em 0; }
img { max-width: 100%; height: auto; object-fit: contain; break-inside: avoid; }
figure { margin: 0.8em 0; break-inside: avoid; }
figcaption { font-size: 9pt; color: #607080; margin-top: 0.25em; }
table { width: 100%; border-collapse: collapse; font-size: 9.5pt; line-height: 1.25; margin: 0.75em 0 1em; }
thead { display: table-header-group; }
tfoot { display: table-footer-group; }
tr { break-inside: avoid; page-break-inside: avoid; }
th, td { border-bottom: 1px solid #d9e0e6; padding: 6px 8px; text-align: left; vertical-align: top; }
th, thead td { background: #edf2f6; font-weight: 700; border-bottom: 1.5px solid #aabac8; color: #253e56; }
.sb-table-align-right { text-align: right; }
.sb-table-align-center { text-align: center; }
pre, code { font-family: "Consolas", "Liberation Mono", monospace; font-size: 9pt; }
pre { background: #f3f5f7; border: 1px solid #e1e5e9; border-radius: 3px; padding: 0.75em; white-space: pre-wrap; overflow-wrap: anywhere; break-inside: avoid; }
code { background: #f3f5f7; padding: 0.05em 0.2em; }
pre code { padding: 0; background: none; }
.mermaid-print { width: 100%; margin: 0.8em auto 1.2em; text-align: center; break-inside: avoid; }
.mermaid-print svg { max-width: 100%; max-height: 215mm; width: auto; height: auto; }
.errore { color: #a12632; font-weight: 700; }
@media screen {
  body.preview-page { width: 210mm; min-height: 297mm; margin: 24px auto; padding: 20mm; box-sizing: border-box; background: white; box-shadow: 0 4px 24px rgba(0,0,0,.16); }
  html.preview { background: #e7eaee; min-height: 100%; }
}
@media print {
  body.preview-page { width: auto; min-height: 0; margin: 0; padding: 0; box-shadow: none; }
  a { text-decoration-color: #9aaab9; }
}
]==]

local function escapeHtml(s)
  return (tostring(s or "")
    :gsub("&", "&amp;")
    :gsub("<", "&lt;")
    :gsub(">", "&gt;")
    :gsub('"', "&quot;"))
end

local function generaSVG(source)
  if not mermaid or type(mermaid.toSVG) ~= "function" then
    error("mermaid.toSVG() non disponibile in Mermaid.md")
  end
  local svg = mermaid.toSVG(source)
  if type(svg) ~= "string" or not svg:find("<svg", 1, true) then
    error("Mermaid non ha restituito SVG")
  end
  return svg
end

-- T6: conserva la posizione dei blocchi Mermaid con fence ```.
-- Limitazione: fence annidate, ~~~ e fence con più di tre backtick
-- non sono ancora gestite da questo prototipo.
local function convertiMermaid(testo)
  testo = "\n" .. testo:gsub("\r\n", "\n") .. "\n"
  local diagrammi = {}
  local prefisso = "SBPRINTMERMAIDTOKEN"
  local errori = 0
  while testo:find(prefisso, 1, true) do
    prefisso = prefisso .. "X"
  end
  local modificato, trovati = testo:gsub(
    "\n[ \t]*```[ \t]*mermaid[ \t]*\n(.-)\n[ \t]*```[ \t]*",
    function(source)
      local indice = #diagrammi + 1
      local token = prefisso .. indice .. "END"
      local ok, result = pcall(generaSVG, source)
      if ok then
        diagrammi[indice] = '<div class="mermaid-print">' .. result .. '</div>'
      else
        errori = errori + 1
        diagrammi[indice] = '<pre class="errore">' .. escapeHtml(result) .. '</pre>'
      end
      return "\n\n" .. token .. "\n\n"
    end
  )
  modificato = modificato:sub(2)
  local html = markdown.markdownToHtml(modificato)
  for i, svg in ipairs(diagrammi) do
    local token = prefisso .. i .. "END"
    html = html:gsub('<span class="p">' .. token .. '</span>', function() return svg end)
    html = html:gsub('<p>' .. token .. '</p>', function() return svg end)
    html = html:gsub(token, function() return svg end)
  end
  return html, trovati, errori
end

-- v0.08: converte formule KaTeX in HTML statico, senza widget live.
-- Non valuta formule nei fence o nel codice inline.
local function convertiMatematica(testo)
  local segnaposti, nErrori = {}, 0
  local prefisso = "SBPRINTKATEXTOKEN"
  while testo:find(prefisso, 1, true) do prefisso = prefisso .. "X" end

  local function crea(formula, display)
    local indice = #segnaposti + 1
    local token = prefisso .. indice .. "END"
    local ok, result = pcall(function()
      if not latex or not latex.katex then
        error("Silverbullet-Math / KaTeX non disponibile")
      end
      return latex.katex.renderToString(formula, {
        trust = false,
        throwOnError = true,
        displayMode = display,
      })
    end)
    if ok and type(result) == "string" then
      segnaposti[indice] = display and
        ('<div class="sb-print-math-block">' .. result .. '</div>') or
        ('<span class="sb-print-math-inline">' .. result .. '</span>')
    else
      nErrori = nErrori + 1
      segnaposti[indice] = '<span class="errore">Formula KaTeX: ' ..
        escapeHtml(tostring(result)) .. '</span>'
    end
    return token
  end

  -- Protegge i blocchi fenced (qualsiasi linguaggio) e il codice inline.
  local protected, protectedPrefix = {}, "SBPRINTCODETOKEN"
  while testo:find(protectedPrefix, 1, true) do protectedPrefix = protectedPrefix .. "X" end
  local fence, fenceLen, fenceChar, mermaidFence = nil, 0, nil, false
  local lines = {}
  testo = testo:gsub("\r\n", "\n")
  for line in (testo .. "\n"):gmatch("([^\n]*)\n") do
    local indent, marker = line:match("^(%s*)([`~]+)")
    if fence then
      if marker and marker:sub(1,1) == fenceChar and #marker >= fenceLen and
        line:match("^%s*[`~]+%s*$") then
        fence = nil
      end
      if mermaidFence then
        lines[#lines+1] = line
      else
        protected[#protected + 1] = line
        lines[#lines+1] = protectedPrefix .. #protected .. "END"
      end
    elseif marker and #indent <= 3 and #marker >= 3 and
      (marker:match("^`+$") or marker:match("^~+$")) then
      fence = true; fenceChar = marker:sub(1,1); fenceLen = #marker
      mermaidFence = line:match("^%s*```%s*mermaid%s*$") ~= nil
      if mermaidFence then
        lines[#lines+1] = line
      else
        protected[#protected + 1] = line
        lines[#lines+1] = protectedPrefix .. #protected .. "END"
      end
    else
      -- Blocchi inline `...` sulla stessa riga.
      local processed = line:gsub("(`+)(.-)%1", function(ticks, content)
        protected[#protected+1] = ticks .. content .. ticks
        return protectedPrefix .. #protected .. "END"
      end)
      lines[#lines+1] = processed
    end
  end
  local unprotected = table.concat(lines, "\n")

  -- Blocco $$ multilinea o nella stessa riga.
  unprotected = unprotected:gsub("%$%$(.-)%$%$", function(formula)
    return "\n\n" .. crea(formula, true) .. "\n\n"
  end)
  -- Inline $...$; evita il caso dei delimitatori $$ residui.
  unprotected = unprotected:gsub("%$([^$\n]+)%$", function(formula)
    if formula:match("^%s*$") then return "$" .. formula .. "$" end
    return crea(formula, false)
  end)

  for i, value in ipairs(protected) do
    local token = protectedPrefix .. i .. "END"
    unprotected = unprotected:gsub(token, function() return value end)
  end
  return unprotected, segnaposti, prefisso, nErrori
end

local function inserisciMatematica(html, segnaposti, prefisso)
  for i, result in ipairs(segnaposti) do
    local token = prefisso .. i .. "END"
    html = html:gsub('<span class="p">' .. token .. '</span>', function() return result end)
    html = html:gsub('<p>' .. token .. '</p>', function() return result end)
    html = html:gsub(token, function() return result end)
  end
  return html
end

-- v0.06: selezione non persistente dei metadati YAML originali.
local function campoPresente(v)
  return v ~= nil and v ~= ""
end

local function stringa(v)
  if type(v) == "boolean" then return v and "Sì" or "No" end
  return tostring(v or "")
end

local mesi = {
  "gennaio", "febbraio", "marzo", "aprile", "maggio", "giugno",
  "luglio", "agosto", "settembre", "ottobre", "novembre", "dicembre"
}
local function valoreData(v)
  local s = stringa(v)
  local y, m, d, suffix = s:match("^(%d%d%d%d)%-(%d%d)%-(%d%d)(.*)$")
  if y and tonumber(m) >= 1 and tonumber(m) <= 12
     and tonumber(d) >= 1 and tonumber(d) <= 31
     and (suffix == "" or suffix:match("^[T ]")) then
    return tonumber(d) .. " " .. mesi[tonumber(m)] .. " " .. y
  end
  return s
end

local function nomeCampo(k)
  local nomi = {
    tags = "Tag", owner = "Responsabile", status = "Stato",
    role = "Ruolo", email = "Email", phone = "Telefono",
    reviewed = "Revisionato", supersededBy = "Sostituito da",
    luoghi = "Luoghi", Viaggio = "Viaggio"
  }
  return nomi[k] or k
end

local function tagLista(v)
  if type(v) == "string" then
    if v == "" then return {} end
    return {v}
  end
  if type(v) == "table" then
    local items = {}
    for _, x in ipairs(v) do items[#items+1] = stringa(x) end
    return items
  end
  return {}
end

local function tagHTML(tags, modo)
  local v = tagLista(tags)
  if #v == 0 then return "" end
  if modo == "hash" then
    for i, x in ipairs(v) do v[i] = "#" .. x end
  end
  if modo == "chips" then
    local spans = {}
    for _, x in ipairs(v) do
      spans[#spans+1] = '<span class="fm-tag">' .. escapeHtml(x) .. '</span>'
    end
    return table.concat(spans, " ")
  end
  return escapeHtml(table.concat(v, modo == "hash" and " " or ", "))
end

local function valoreHTML(v, k, tagModo, depth)
  depth = depth or 0
  if depth > 4 then return "…" end
  if k == "tags" then return tagHTML(v, tagModo) end
  if type(v) == "table" then
    local vals = {}
    if #v > 0 then
      for _, x in ipairs(v) do
        vals[#vals+1] = valoreHTML(x, "", tagModo, depth+1)
      end
      return table.concat(vals, " · ")
    end
    local ks = {}
    for key in pairs(v) do ks[#ks+1] = key end
    table.sort(ks, function(a,b) return tostring(a)<tostring(b) end)
    for _, key in ipairs(ks) do
      vals[#vals+1] = escapeHtml(key) .. ": " .. valoreHTML(v[key], tostring(key), tagModo, depth+1)
    end
    return table.concat(vals, "; ")
  end
  local s = stringa(v)
  if k == "date" or k == "reviewed" or k:match("[Dd]ate$")
     or k:match("^data") then s = valoreData(s) end
  -- Evita che i wiki-link siano stampati con le doppie parentesi.
  s = s:gsub("%[%[([^%]]+)%]%]", function(x)
    return x:match("|(.+)$") or x
  end)
  return escapeHtml(s)
end

-- Campi realmente scritti nel YAML: il parser può aggiungere metadati
-- tecnici (es. range) che non devono apparire nella stampa.
local function nomiYAML(testo)
  local keys, seen = {}, {}
  local normalized = testo:gsub("\r\n", "\n")
  local yaml = normalized:match("^%-%-%-\n(.-)\n%-%-%-\n")
  if not yaml then yaml = normalized:match("^%-%-%-\n(.-)\n%-%-%-$") end
  if not yaml then return keys end
  for line in (yaml .. "\n"):gmatch("([^\n]*)\n") do
    if line:match("^%S") and not line:match("^%s*#") then
      local key = line:match('^([%w_%-]+)%s*:')
      if key and not seen[key] then
        seen[key] = true
        keys[#keys+1] = key
      end
    end
  end
  return keys
end

local function altriCampi(fm, sourceKeys)
  local ks = {}
  for _, k in ipairs(sourceKeys) do
    if k ~= "displayName" and k ~= "description" and k ~= "date"
       and fm[k] ~= nil then
      ks[#ks+1] = k
    end
  end
  return ks
end

local function scegliDaElenco(title, options)
  return editor.filterBox(title, options)
end

-- Multi-selezione senza JavaScript: filtro nativo, spunta e conferma.
local function selezionaCampi(keys)
  local selected = {}
  while true do
    local options = {{name="✓ Conferma selezione", value="__ok", description="Termina e crea anteprima"}}
    for _, k in ipairs(keys) do
      options[#options+1] = {
        name=(selected[k] and "☑ " or "☐ ") .. nomeCampo(k),
        description=k,
        value=k
      }
    end
    local scelta = scegliDaElenco("Frontmatter: seleziona i campi", options)
    if not scelta then return nil end
    if scelta.value == "__ok" then return selected end
    selected[scelta.value] = not selected[scelta.value]
  end
end

local function scegliOpzioni(fm, sourceKeys)
  local keys = altriCampi(fm, sourceKeys)
  local modes = {
    {name="Standard: titolo, description, date", value="s"},
    {name="Standard + selezione campi", value="c"},
    {name="Tutti i campi YAML", value="t"},
    {name="Solo contenuto, senza frontmatter", value="n"},
  }
  local scelta = scegliDaElenco("Stampa: frontmatter", modes)
  if not scelta then return nil end
  local mode = scelta.value
  local selected = {}
  if mode == "t" then
    for _, k in ipairs(keys) do selected[k] = true end
  elseif mode == "c" then
    selected = selezionaCampi(keys)
    if not selected then return nil end
  end

  local tagModo = "comma"
  if selected.tags then
    local tagOption = scegliDaElenco("Formato tag", {
      {name="Separati da virgole", value="comma"},
      {name="Hashtag", value="hash"},
      {name="Etichette grafiche", value="chips"},
      {name="Campo metadati", value="field"},
    })
    if not tagOption then return nil end
    tagModo = tagOption.value
  end
  return {mode=mode, fields=selected, tagModo=tagModo}
end

local function frontmatterHTML(fm, opts)
  if opts.mode == "n" then return "", false end
  local out = {}
  local titolo = campoPresente(fm.displayName) and stringa(fm.displayName) or nil
  if titolo then
    out[#out+1] = '<h1 class="fm-document-title">' .. escapeHtml(titolo) .. '</h1>'
  end
  if campoPresente(fm.description) then
    local testo = markdown.markdownToHtml(stringa(fm.description))
    out[#out+1] = '<div class="fm-abstract">' .. testo .. '</div>'
  end
  if campoPresente(fm.date) then
    out[#out+1] = '<div class="fm-date">' .. escapeHtml(valoreData(fm.date)) .. '</div>'
  end
  local keys = {}
  for k in pairs(opts.fields) do
    if opts.fields[k] and fm[k] ~= nil then keys[#keys+1] = k end
  end
  table.sort(keys)
  if #keys > 0 then
    local rows = {}
    for _, k in ipairs(keys) do
      local v = valoreHTML(fm[k], k, opts.tagModo)
      if k == "tags" and opts.tagModo ~= "field" then
        rows[#rows+1] = '<div class="fm-tags">' .. v .. '</div>'
      else
        rows[#rows+1] = '<div class="fm-field"><span class="fm-label">' ..
          escapeHtml(nomeCampo(k)) .. '</span><span>' .. v .. '</span></div>'
      end
    end
    out[#out+1] = '<section class="fm-details fm-generic">' ..
       table.concat(rows, "\n") .. '</section>'
  end
  return table.concat(out, "\n"), titolo ~= nil
end

-- Scala soltanto i titoli HTML, dopo il parsing Markdown.
-- Il titolo originale identico al displayName è rimosso se è il primo H1.
local function sistemaTitoli(html, title)
  if not title then return html end
  local escaped = escapeHtml(title)
  local found = false
  html = html:gsub("<h1([^>]*)>(.-)</h1>", function(attrs, inner)
    if not found then
      found = true
      if inner == escaped then return "" end
    end
    return '<h1' .. attrs .. '>' .. inner .. '</h1>'
  end)
  html = html:gsub("<(/?)h([1-5])([^>]*)>", function(slash, lev, attrs)
    return "<" .. slash .. "h" .. (tonumber(lev)+1) .. attrs .. ">"
  end)
  return html
end

-- Formato e orientamento: scelta a ogni stampa, senza salvataggio.
local function scegliFormatoPagina()
  local scelta = editor.filterBox("Stampa: formato pagina", {
    {name="A4 verticale (predefinito)", value="a4v"},
    {name="A4 orizzontale", value="a4o"},
    {name="A3 verticale", value="a3v"},
    {name="A3 orizzontale", value="a3o"},
  })
  if not scelta then return nil end
  local formati = {
    a4v = {css="A4 portrait", width=210, height=297},
    a4o = {css="A4 landscape", width=297, height=210},
    a3v = {css="A3 portrait", width=297, height=420},
    a3o = {css="A3 landscape", width=420, height=297},
  }
  return formati[scelta.value]
end

function stampa.anteprima()
  local formato = scegliFormatoPagina()
  if not formato then return end
  local pagina = editor.getCurrentPage()
  local testo = editor.getText()
  local estratto = index.extractFrontmatter(testo, {removeFrontMatterSection=true})
  local fm = estratto.frontmatter or {}
  local corpo = estratto.text or testo
  local opts = scegliOpzioni(fm, nomiYAML(testo))
  if not opts then return end

  local intestazione, haTitolo = frontmatterHTML(fm, opts)
  local baked = markdown.bakeSections(corpo, pagina)
  local espanso = markdown.expandMarkdown(baked)
  local withMath, mathTags, mathPrefix, erroriMath = convertiMatematica(espanso)
  local html, numeroMermaid, erroriMermaid = convertiMermaid(withMath)
  html = inserisciMatematica(html, mathTags, mathPrefix)
  if haTitolo then html = sistemaTitoli(html, stringa(fm.displayName)) end

  local CSS_FM = [[
.fm-document-title { margin-top: 0; }
.fm-abstract { margin: .5em 0 1em; padding: .25em 0 .25em 1em; border-left: 3px solid #93a9bf; font-style: italic; line-height: 1.35; color: #465565; }
.fm-date { color: #637587; font-size: 9.5pt; margin: .3em 0 1em; }
.fm-details { margin: .8em 0 1.3em; padding: .65em .9em; break-inside: avoid; font-size: 9.5pt; }
.fm-adr, .fm-person { border: 1px solid #d9e0e6; border-left: 3px solid #243c56; border-radius: 5px; background: #f8fafb; }
.fm-generic { border-top: 1px solid #d9e0e6; border-bottom: 1px solid #d9e0e6; }
.fm-card-label { font-size: 8pt; font-weight: bold; letter-spacing: .08em; color: #61768a; margin-bottom: .6em; }
.fm-field { display: grid; grid-template-columns: 32% 1fr; column-gap: 10px; padding: 3px 0; }
.fm-label { font-weight: bold; color: #46596b; }
.fm-tags { color: #50677c; padding: 3px 0; }
.fm-tag { display: inline-block; border: 1px solid #d6e1ea; background: #eef3f7; padding: 1px 6px; margin: 2px 3px 2px 0; border-radius: 4px; font-size: 9pt; }
]]
  -- CSS KaTeX con URL assoluto: i font restano relativi al CSS.
  local urlPrefix = tostring(system.getURLPrefix() or "/")
  local origin = tostring(js.window.location.origin)
  local katexBase
  if urlPrefix:match("^https?://") then
    katexBase = urlPrefix
  else
    if urlPrefix:sub(1, 1) ~= "/" then
      urlPrefix = "/" .. urlPrefix
    end
    katexBase = origin .. urlPrefix
  end
  if katexBase:sub(-1) ~= "/" then
    katexBase = katexBase .. "/"
  end
  local katexCss = katexBase .. ".fs/Library/mrmugame/Silverbullet-Math/katex.min.css"
  -- Regole dinamiche successive al CSS originale: prevalgono sui valori A4.
  local CSS_PAGINA = string.format([[
@page { size: %s; margin: 20mm; }
@media screen {
  body.preview-page { width: %dmm; min-height: %dmm; }
}
]], formato.css, formato.width, formato.height)
  local documento = table.concat({
    '<!DOCTYPE html>',
    '<html lang="it" class="preview"><head><meta charset="utf-8">',
    '<title>Stampa - ' .. escapeHtml(pagina) .. '</title>',
    '<base href="' .. escapeHtml(js.window.document.baseURI) .. '">',
    '<link rel="stylesheet" href="' .. escapeHtml(katexCss) .. '">',
    '<style>', CSS_STAMPA, CSS_PAGINA, CSS_FM,
    '.sb-print-math-block { margin: .8em 0; overflow: visible; break-inside: avoid; text-align: center; }',
    '.sb-print-math-inline { white-space: nowrap; }',
    '.sb-print-math-block .katex-display { margin: 0; }',
    '</style>',
    '</head><body class="preview-page">', intestazione, html, '</body></html>'
  }, "\n")
  local finestra = js.window.open("", "_blank")
  if not finestra then
    editor.flashNotification("Popup bloccato dal browser")
    return
  end
  finestra.document.open()
  finestra.document.write(documento)
  finestra.document.close()
  editor.flashNotification("Mermaid: " .. numeroMermaid .. " diagrammi, " .. erroriMermaid .. " errori; KaTeX: " .. #mathTags .. " formule, " .. erroriMath .. " errori")
end

command.define {
  name = "Stampa: Anteprima Markdown",
  run = function() stampa.anteprima() end
}
```
