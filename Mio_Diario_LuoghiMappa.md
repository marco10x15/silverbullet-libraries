---
name: "Library/MG/Mio_Diario_LuoghiMappa"
tags: meta/library
description: "Mappa dei luoghi."
version: "0.0-02"
versionDate: 2026-09-04
pageDecoration.prefix: "🗺️ "
share.uri: "github:marco10x15/silverbullet-libraries/Mio_Diario_LuoghiMappa.md"
---

# Mio Diario — Mappa dei Luoghi

**Versione:** 0.0-02 — 04 settembre 2026

## Descrizione

Visualizza su una mappa Leaflet le pagine `luoghi/...` che dispongono dell'attributo frontmatter:

```yaml
coordinate: [45.899167, 6.129444]
```

Le coordinate sono interpretate nel formato:

```text
[latitudine, longitudine]
```

La funzione pubblica è unica:

```space-lua
${luoghiMap()}
```

Il comportamento dipende dalla pagina corrente:

- nelle pagine `luoghi/...` visualizza il luogo corrente e i soli figli diretti;
- nelle pagine `Diario/...`, `Viaggi/...` e nelle altre pagine legge l'attributo frontmatter `luoghi`;
- è possibile passare esplicitamente una lista di pagine luogo;
- le pagine senza coordinate valide vengono ignorate;
- non viene eseguita alcuna modifica persistente delle note.

La mappa utilizza:

- Leaflet 1.9.4;
- tile OpenStreetMap;
- etichette permanenti con `displayName`, posizionate direttamente sulle coordinate;
- popup al click con `displayName`, `tipoAmministrativo` e link alla pagina SilverBullet;
- Bottom Widget automatico nelle pagine `Diario/...`, `luoghi/...` e `Viaggi/...`;
- zoom automatico sui luoghi disponibili.

## Frontmatter Luoghi

Esempio:

```yaml
---
displayName: Annecy
tipoAmministrativo: comune
divisioneIntermedia: FR-74
wikipedia: https://it.wikipedia.org/wiki/Annecy
coordinate: [45.899167, 6.129444]
---
```

Per la mappa vengono utilizzati direttamente:

- `displayName`;
- `tipoAmministrativo`;
- `coordinate`.

`divisioneIntermedia` e `wikipedia` restano disponibili alla pagina ma non sono necessari al rendering della versione 0.0-02.

## Utilizzo

### Pagina luogo

```space-lua
${luoghiMap()}
```

Su una pagina come:

```text
luoghi/FRA/ARA/Annecy
```

visualizza:

1. Annecy, se dispone di coordinate;
2. le sole sottopagine dirette di Annecy che dispongono di coordinate.

I discendenti di livello successivo non vengono inclusi automaticamente.

### Pagina Diario o Viaggio

Con frontmatter:

```yaml
luoghi:
  - "[[luoghi/FRA/ARA/Annecy]]"
  - "[[luoghi/FRA/ARA/Chamonix-Mont-Blanc]]"
```

è sufficiente:

```space-lua
${luoghiMap()}
```

È supportato anche un singolo valore:

```yaml
luoghi: "[[luoghi/FRA/ARA/Annecy]]"
```

### Lista esplicita

```space-lua
${luoghiMap({
  luoghi = {
    "luoghi/FRA/ARA/Annecy",
    "luoghi/FRA/ARA/Chamonix-Mont-Blanc"
  }
})}
```

La lista può contenere sia nomi pagina sia wikilink.

### Disattivare i figli diretti

Sulle pagine `luoghi/...`:

```space-lua
${luoghiMap({children = false})}
```


## Bottom Widget automatico

La libreria registra un listener sull'hook ufficiale:

```space-lua
hooks:renderBottomWidgets
```

La mappa viene mostrata automaticamente in fondo esclusivamente alle pagine con path:

```text
Diario/...
luoghi/...
Viaggi/...
```

Il comportamento resta quello della funzione `luoghiMap()`:

- `Diario/...`: usa il frontmatter `luoghi`;
- `Viaggi/...`: usa il frontmatter `luoghi`;
- `luoghi/...`: mostra il luogo corrente e i soli figli diretti.

Se nessun luogo dispone di coordinate valide, il Bottom Widget non mostra alcun messaggio e non occupa spazio.

La funzione `${luoghiMap()}` resta disponibile per l'uso manuale in qualsiasi pagina.

## Navigazione dalla mappa

Ogni etichetta mantiene il popup Leaflet.

Nel popup viene visualizzato il pulsante:

```text
Apri pagina
```

Il pulsante usa il syscall pubblico disponibile nel sandbox:

```javascript
syscall("editor.navigate", place.name)
```

In questo modo la navigazione avviene direttamente verso la pagina Markdown SilverBullet corrispondente, senza costruire URL dipendenti dall'installazione e senza uscire dall'applicazione.

## Etichette dei luoghi

La versione 0.0-02 non visualizza marker grafici separati.

Il nome del luogo, ricavato da `displayName`, viene renderizzato direttamente come `L.divIcon()` sulla coordinata geografica. In questo modo etichetta e posizione sono gestite dallo stesso oggetto Leaflet e non esiste più il disallineamento tra marker e tooltip.

L'etichetta viene mostrata leggermente sopra la coordinata, senza simboli aggiuntivi.

`tipoAmministrativo` resta disponibile nei dati e viene mostrato nel popup al click. Dal popup è inoltre possibile aprire direttamente la pagina SilverBullet del luogo. In questa versione `tipoAmministrativo` non modifica l'aspetto grafico dell'etichetta.

## Decisioni

La combinazione adottata è:

**`coordinate: [lat, lon]` nel Markdown → Object Index → unica `luoghiMap()` → Leaflet sandbox.**

Per le pagine `luoghi/...` vengono inclusi **il luogo corrente e i figli diretti**, senza ricorsione automatica.

Il Markdown rimane la fonte primaria dei dati. La funzione interroga l'Object Index di SilverBullet e produce esclusivamente una visualizzazione derivata.

La funzione è collocata in una libreria separata ma può essere richiamata indifferentemente da pagine Diario, Luoghi e Viaggi.

## Implementazione

```space-lua
-- ============================================================
-- Mio Diario - Mappa dei Luoghi
-- Versione: 0.0-02
--
-- Visualizza pagine luogo dotate di:
--   coordinate: [latitudine, longitudine]
--
-- PAGINE LUOGHI
--   luogo corrente + figli diretti
--
-- ALTRE PAGINE
--   valori del frontmatter `luoghi`
--
-- BOTTOM WIDGET
--   automatico su Diario/, luoghi/, Viaggi/
--
-- NAVIGAZIONE
--   popup -> editor.navigate(page.name)
--
-- DIPENDENZE
--   Leaflet 1.9.4
--   OpenStreetMap
-- ============================================================


-- ------------------------------------------------------------
-- Normalizzazione di un nome pagina / wikilink
-- ------------------------------------------------------------

local function luogoLinkTarget(value)
  if value == nil then
    return nil
  end

  local s = tostring(value)

  local target =
    string.match(s, "^%[%[([^]|]+)")

  return target or s
end


-- ------------------------------------------------------------
-- Etichetta del luogo
-- ------------------------------------------------------------

local function luogoLabel(page)
  if page.displayName and
     tostring(page.displayName) ~= "" then
    return tostring(page.displayName)
  end

  local name = tostring(page.name or "")

  return string.match(name, "([^/]+)$")
    or name
end


-- ------------------------------------------------------------
-- Coordinate
--
-- Formato previsto:
--   coordinate: [lat, lon]
-- ------------------------------------------------------------

local function luogoCoordinate(page)
  local c = page.coordinate

  if type(c) ~= "table" then
    return nil
  end

  local lat = tonumber(c[1])
  local lon = tonumber(c[2])

  if not lat or not lon then
    return nil
  end

  if lat < -90 or lat > 90 or
     lon < -180 or lon > 180 then
    return nil
  end

  return {
    lat = lat,
    lon = lon
  }
end


-- ------------------------------------------------------------
-- Ricerca di una pagina nell'Object Index
-- ------------------------------------------------------------

local function luogoPage(name)
  if not name or name == "" then
    return nil
  end

  local rows = query[[
    from p = index.pages()
    where p.name == name
    limit 1
  ]]

  return rows[1]
end


-- ------------------------------------------------------------
-- Figli diretti
--
-- index.subPages() restituisce tutte le pagine sottostanti.
-- Il controllo sul resto del path mantiene esclusivamente
-- il livello immediatamente successivo.
-- ------------------------------------------------------------

local function luogoDirectChildren(name)
  local result = {}
  local prefix = name .. "/"

  local pages = query[[
    from p = index.subPages(name)
    select p
  ]]

  for _, page in ipairs(pages) do
    local pageName = tostring(page.name or "")

    if string.sub(pageName, 1, #prefix) == prefix then
      local rest =
        string.sub(
          pageName,
          #prefix + 1
        )

      if rest ~= "" and
         not string.find(
           rest,
           "/",
           1,
           true
         ) then
        table.insert(result, page)
      end
    end
  end

  return result
end


-- ------------------------------------------------------------
-- Dati di un luogo
-- ------------------------------------------------------------

local function luogoMarker(page)
  local c = luogoCoordinate(page)

  if not c then
    return nil
  end

  return {
    name = tostring(page.name or ""),
    label = luogoLabel(page),

    lat = c.lat,
    lon = c.lon,

    tipoAmministrativo =
      tostring(
        page.tipoAmministrativo
        or ""
      )
  }
end


-- ------------------------------------------------------------
-- Aggiunge un luogo evitando duplicati
-- ------------------------------------------------------------

local function addMarker(markers, seen, page)
  if not page or not page.name then
    return
  end

  local name = tostring(page.name)

  if seen[name] then
    return
  end

  seen[name] = true

  local marker = luogoMarker(page)

  if marker then
    table.insert(markers, marker)
  end
end


-- ------------------------------------------------------------
-- Converte frontmatter `luoghi` in lista
-- ------------------------------------------------------------

local function luogoValues(value)
  if value == nil then
    return {}
  end

  if type(value) == "table" then
    return value
  end

  return {value}
end


-- ------------------------------------------------------------
-- Raccoglie i luoghi da visualizzare
-- ------------------------------------------------------------

local function collectLuoghi(options)
  options = options or {}

  local markers = {}
  local seen = {}

  local currentName =
    editor.getCurrentPage()

  local explicit =
    options.luoghi


  -- ----------------------------------------------------------
  -- Lista esplicita
  -- ----------------------------------------------------------

  if explicit ~= nil then
    for _, value in ipairs(
      luogoValues(explicit)
    ) do
      local name =
        luogoLinkTarget(value)

      addMarker(
        markers,
        seen,
        luogoPage(name)
      )
    end

    return markers
  end


  -- ----------------------------------------------------------
  -- Pagina luoghi/...
  -- ----------------------------------------------------------

  if string.sub(currentName, 1, 7) == "luoghi/" then
    local currentPage =
      luogoPage(currentName)

    addMarker(
      markers,
      seen,
      currentPage
    )

    if options.children ~= false then
      for _, child in ipairs(
        luogoDirectChildren(currentName)
      ) do
        addMarker(
          markers,
          seen,
          child
        )
      end
    end

    return markers
  end


  -- ----------------------------------------------------------
  -- Diario, Viaggi e altre pagine
  -- ----------------------------------------------------------

  local currentPage =
    luogoPage(currentName)

  if not currentPage then
    return markers
  end

  for _, value in ipairs(
    luogoValues(currentPage.luoghi)
  ) do
    local name =
      luogoLinkTarget(value)

    addMarker(
      markers,
      seen,
      luogoPage(name)
    )
  end

  return markers
end


-- ------------------------------------------------------------
-- Escape di una stringa Lua per JavaScript
-- ------------------------------------------------------------

local function jsString(value)
  return string.format(
    "%q",
    tostring(value or "")
  )
end


-- ------------------------------------------------------------
-- Serializzazione minimale Lua -> JavaScript
--
-- Evita dipendenze JSON aggiuntive.
-- ------------------------------------------------------------

local function markersToJavascript(markers)
  local rows = {}

  for _, marker in ipairs(markers) do
    table.insert(
      rows,
      "{" ..
      "name:" .. jsString(marker.name) .. "," ..
      "label:" .. jsString(marker.label) .. "," ..
      "lat:" .. tostring(marker.lat) .. "," ..
      "lon:" .. tostring(marker.lon) .. "," ..
      "tipoAmministrativo:" ..
        jsString(marker.tipoAmministrativo) ..
      "}"
    )
  end

  return "[" ..
    table.concat(rows, ",") ..
    "]"
end


-- ============================================================
-- Funzione pubblica
-- ============================================================

function luoghiMap(options)
  options = options or {}

  local markers =
    collectLuoghi(options)

  if #markers == 0 then
    if options.silentEmpty then
      return widget.new {}
    end

    return widget.markdownBlock(
      "_Nessun luogo con coordinate disponibile per la mappa._"
    )
  end

  local markerData =
    markersToJavascript(markers)

  return widget.sandbox {
    html = [[
      <div style="width:100%;">
        <div
          id="luoghi-map"
          style="
            width:100%;
            height:450px;
            border-radius:6px;
            overflow:hidden;
          ">
        </div>
      </div>
    ]],

    script = string.format([==[
      const places = %s;


      // ========================================================
      // Etichetta geografica
      //
      // Il nome del luogo è l'icona Leaflet.
      // Non vengono usati marker o tooltip separati.
      // ========================================================

      function escapeHtml(value) {
        return String(value || "")
          .replaceAll("&", "&amp;")
          .replaceAll("<", "&lt;")
          .replaceAll(">", "&gt;")
          .replaceAll('"', "&quot;")
          .replaceAll("'", "&#039;");
      }


      function labelIcon(place) {
        return L.divIcon({
          className: "",

          html:
            '<div class="luogo-label-marker">' +
            escapeHtml(place.label) +
            '</div>',

          iconSize: null,
          iconAnchor: [0, 0],
          popupAnchor: [0, -12]
        });
      }


      // ========================================================
      // Inizializzazione
      // ========================================================

      async function main() {
        await loadJsByUrl(
          "https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"
        );

        const css =
          document.createElement("link");

        css.rel = "stylesheet";
        css.href =
          "https://unpkg.com/leaflet@1.9.4/dist/leaflet.css";

        document.head.appendChild(css);


        const style =
          document.createElement("style");

        style.textContent = `
          .luogo-label-marker {
            display: inline-block;
            transform: translate(-50%%, -110%%);
            padding: 3px 6px;
            box-sizing: border-box;

            background: rgba(255, 255, 255, 0.90);
            border: 1px solid #555;
            border-radius: 4px;

            font: 600 12px sans-serif;
            line-height: 1.2;
            white-space: nowrap;

            cursor: pointer;
          }
        `;

        document.head.appendChild(style);


        const map =
          L.map("luoghi-map");

        L.tileLayer(
          "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
          {
            maxZoom: 19,
            attribution:
              '&copy; OpenStreetMap contributors'
          }
        ).addTo(map);


        const group =
          L.featureGroup()
            .addTo(map);


        for (const place of places) {
          const marker =
            L.marker(
              [place.lat, place.lon],
              {
                icon: labelIcon(place)
              }
            )
            .addTo(group);

          const popup =
            document.createElement("div");

          const title =
            document.createElement("strong");

          title.textContent =
            place.label;

          popup.appendChild(title);

          if (place.tipoAmministrativo) {
            const type =
              document.createElement("div");

            type.textContent =
              place.tipoAmministrativo;

            type.style.marginTop =
              "2px";

            popup.appendChild(type);
          }

          const openButton =
            document.createElement("button");

          openButton.type =
            "button";

          openButton.textContent =
            "Apri pagina";

          openButton.style.marginTop =
            "8px";

          openButton.style.cursor =
            "pointer";

          openButton.addEventListener(
            "click",
            async () => {
              await syscall(
                "editor.navigate",
                place.name
              );
            }
          );

          popup.appendChild(openButton);

          marker.bindPopup(popup);
        }


        if (places.length == 1) {
          map.setView(
            [places[0].lat, places[0].lon],
            13
          );
        } else {
          map.fitBounds(
            group.getBounds(),
            {
              padding: [30, 30],
              maxZoom: 14
            }
          );
        }
      }


      main().catch(error => {
        const element =
          document.getElementById(
            "luoghi-map"
          );

        element.innerHTML =
          '<div style="padding:12px;font-family:sans-serif;">' +
          'Errore mappa: ' +
          String(error.message || error) +
          '</div>';
      });
    ]==], markerData),

    markdown =
      "Mappa dei luoghi"
  }
end


-- ============================================================
-- Bottom Widget
--
-- Visualizzato esclusivamente su:
--   Diario/...
--   luoghi/...
--   Viaggi/...
-- ============================================================

local function luoghiMapBottomEnabled(pageName)
  if not pageName then
    return false
  end

  return
    string.sub(pageName, 1, 7) == "Diario/" or
    string.sub(pageName, 1, 7) == "luoghi/" or
    string.sub(pageName, 1, 7) == "Viaggi/"
end


event.listen {
  name = "hooks:renderBottomWidgets",

  run = function(e)
    local pageName =
      editor.getCurrentPage()

    if not luoghiMapBottomEnabled(pageName) then
      return
    end

    return luoghiMap({
      silentEmpty = true
    })
  end
}
```

## Note versione 0.0-02

Questa è la prima versione di test.

Sono intenzionalmente esclusi:

- discendenti ricorsivi delle pagine luogo;
- gestione automatica delle sovrapposizioni tra etichette molto vicine;
- icone o stili specifici basati su `tipoAmministrativo` o sull'attributo `tipo`;
- lettura o geocodifica automatica da Wikipedia;
- modifica automatica delle coordinate nel frontmatter.

Le funzionalità escluse potranno essere valutate dopo il test del Bottom Widget e della navigazione su dati reali.
