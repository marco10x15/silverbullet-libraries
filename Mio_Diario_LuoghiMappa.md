---
name: "Library/MG/Mio_Diario_LuoghiMappa"
tags: meta/library
description: "Mappe Leaflet dei luoghi e dei percorsi GPX del Diario."
version: "0.6-01"
versionDate: 2026-09-08
pageDecoration.prefix: "🗺️ "
share.uri: "github:marco10x15/silverbullet-libraries/Mio_Diario_LuoghiMappa.md"
---

# Mio Diario — Mappa dei Luoghi

**Versione:** 0.6-01 — 08 settembre 2026

## Dipendenze

Questa libreria richiede:

```text
Library/MG/Mio_Diario
Library/MG/DateFormat
```

`Mio_Diario` fornisce le funzioni `luoghi.*` e l'aggregazione dei Viaggi.
`DateFormat` è una libreria esterna personale e fornisce `date.format()`.

Leaflet 1.9.4 e toGeoJSON 5.8.1 vengono caricati a runtime dal sandbox.

## Implementazioni canoniche

La responsabilità dei renderer è definita così:

* `luoghiMap()` è il renderer **canonico standalone** della mappa dei luoghi;
* `gpxMap()` è il renderer **canonico standalone** dei GPX ed è quello usato
  dalla Virtual Page `GPX/YYYY-MM-DD`;
* `mediaLuoghiWidget()` è l'implementazione **canonica dei Bottom Widget
  compositi** `📋 / 🗺️ / 🧭 / 📷`;
* `photoGalleryWidget()` rimane il renderer standalone della PhotoGallery nella
  libreria dedicata.

`mediaLuoghiWidget()` mantiene un proprio sandbox perché deve alternare più
viste nello stesso frame. Non sostituisce le API standalone e non registra
listener autonomi.

## Descrizione

Visualizza su una mappa Leaflet le pagine `luoghi/...` che dispongono dell'attributo frontmatter:

```yaml
coordinate: [45.899167, 6.129444]
```

Le coordinate sono interpretate nel formato:

```text
[latitudine, longitudine]
```

Le funzioni pubbliche sono:

```space-lua
${luoghiMap()}
${gpxMap()}
```

`gpxMap()` mantiene anche la chiamata esplicita compatibile con la precedente
libreria `Mio_Diario_GPX 0.2-01`:

```space-lua
${gpxMap("media/537GPX.gpx")}
```

Sulle pagine `Diario/...`, se il path non viene passato, `gpxMap()` usa
l'attributo indicizzato `date` della pagina e cerca il GPX corrispondente in
`media/`. La convenzione primaria è `media/YYYYMMDD*.gpx`; viene accettato
anche `media/YYYY-MM-DD*.gpx` per compatibilità con esportazioni giornaliere.

Il comportamento dipende dalla pagina corrente:

- nelle pagine `luoghi/...` visualizza il luogo corrente e i soli figli diretti;
- nelle pagine `Diario/...` legge l'attributo frontmatter `luoghi`;
- nelle pagine `Viaggi/...` aggrega i luoghi delle pagine Diario appartenenti al viaggio;
- nelle altre pagine usa l'attributo frontmatter `luoghi`, se presente;
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


## Bottom Widget e orchestrazione

Questa libreria **non registra listener `hooks:renderBottomWidgets`**.

La posizione e l'ordine dei Bottom Widget sono gestiti da `Mio_Diario`, che
passa a `mediaLuoghiWidget()` i dati già aggregati per il contesto corrente:

* `luoghi/...` — luogo corrente, figli diretti ed eventuali GPX correlati;
* `Diario/...` — luoghi della giornata, GPX della data e PhotoGallery disponibile;
* `Viaggi/...` — luoghi aggregati, GPX e PhotoGallery delle giornate del viaggio.

Le funzioni `${luoghiMap()}` e `${gpxMap()}` rimangono disponibili per uso
manuale e per le Virtual Page.

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
-- Versione: 0.6-01
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
--   luoghi: automatico su Diario/, Viaggi/
--   GPX: automatico su Diario/
--
-- NAVIGAZIONE
--   popup -> editor.navigate(page.name)
--
-- DIPENDENZE
--   Leaflet 1.9.4
--   OpenStreetMap
-- ============================================================


-- ------------------------------------------------------------
-- Coordinate e dati Luogo
-- ------------------------------------------------------------

local function luogoCoordinate(page)
  local c =
    page and page.coordinate

  if type(c) ~= "table" then
    return nil
  end

  local lat = tonumber(c[1])
  local lon = tonumber(c[2])

  if not lat or not lon then
    return nil
  end

  if lat < -90 or lat > 90
    or lon < -180 or lon > 180
  then
    return nil
  end

  return {
    lat = lat,
    lon = lon
  }
end


local function luogoMarker(page)
  local c =
    luogoCoordinate(page)

  if not c then
    return nil
  end

  return {
    name =
      tostring(page.name or ""),
    label =
      luoghi.label(
        page,
        luoghi.basename(
          tostring(page.name or "")
        )
      ),
    lat = c.lat,
    lon = c.lon,
    tipoAmministrativo =
      tostring(
        page.tipoAmministrativo
        or ""
      )
  }
end


local function uniquePages(pages)
  local result = {}
  local seen = {}

  for _, page in ipairs(pages or {}) do
    if page
      and page.name
      and not seen[page.name]
    then
      table.insert(result, page)
      seen[page.name] = true
    end
  end

  return result
end


local function collectLuoghiPages(options)
  options = options or {}

  if options.pages then
    return uniquePages(
      options.pages
    )
  end

  if options.luoghi ~= nil then
    return luoghi.resolvePages(
      options.luoghi
    )
  end

  local currentName =
    editor.getCurrentPage()

  if string.startsWith(
    currentName,
    "Viaggi/"
  )
    and type(viaggiDiarioInfo) == "function"
  then
    local diarioInfo =
      viaggiDiarioInfo(currentName)

    local values = {}
    local seen = {}

    for _, info in ipairs(diarioInfo or {}) do
      for _, value in ipairs(
        info.luoghi or {}
      ) do
        local target =
          luoghi.wikilinkTarget(value)
          or tostring(value or "")

        if target ~= ""
          and not seen[target]
        then
          table.insert(values, value)
          seen[target] = true
        end
      end
    end

    return luoghi.resolvePages(values)
  end

  if string.startsWith(
    currentName,
    "luoghi/"
  ) then
    local result = {}

    local current =
      luoghi.getPage(currentName)

    if current then
      table.insert(result, current)
    end

    if options.children ~= false then
      for _, child in ipairs(
        luoghi.children(currentName)
        or {}
      ) do
        table.insert(result, child)
      end
    end

    return uniquePages(result)
  end

  local current =
    luoghi.getPage(currentName)

  if not current then
    return {}
  end

  return luoghi.resolvePages(
    current.luoghi
  )
end


local function pagesToMarkers(pages)
  local result = {}

  for _, page in ipairs(
    uniquePages(pages)
  ) do
    local marker =
      luogoMarker(page)

    if marker then
      table.insert(result, marker)
    end
  end

  return result
end


local function pagesToListItems(pages)
  local result = {}

  for _, page in ipairs(
    uniquePages(pages)
  ) do
    table.insert(
      result,
      {
        name = page.name,
        label =
          luoghi.label(
            page,
            luoghi.basename(page.name)
          )
      }
    )
  end

  return result
end


local function jsString(value)
  return string.format(
    "%q",
    tostring(value or "")
  )
end


local function markersToJavascript(markers)
  local rows = {}

  for _, marker in ipairs(markers) do
    table.insert(
      rows,
      "{"
        .. "name:" .. jsString(marker.name) .. ","
        .. "label:" .. jsString(marker.label) .. ","
        .. "lat:" .. tostring(marker.lat) .. ","
        .. "lon:" .. tostring(marker.lon) .. ","
        .. "tipoAmministrativo:"
        .. jsString(marker.tipoAmministrativo)
        .. "}"
    )
  end

  return "["
    .. table.concat(rows, ",")
    .. "]"
end


local function listItemsToJavascript(items)
  local rows = {}

  for _, item in ipairs(items) do
    table.insert(
      rows,
      "{"
        .. "name:" .. jsString(item.name) .. ","
        .. "label:" .. jsString(item.label)
        .. "}"
    )
  end

  return "["
    .. table.concat(rows, ",")
    .. "]"
end


-- ============================================================
-- Renderer autonomo Luoghi
-- ============================================================

function luoghiMap(options)
  options = options or {}

  local pages =
    collectLuoghiPages(options)

  local markers =
    pagesToMarkers(pages)

  local listPages = {}

  if options.listPages then
    listPages =
      uniquePages(
        options.listPages
      )
  elseif options.list ~= nil then
    if options.list == options.luoghi then
      listPages = pages
    else
      listPages =
        luoghi.resolvePages(
          options.list
        )
    end
  end

  local listItems =
    pagesToListItems(
      listPages
    )

  if #markers == 0
    and #listItems == 0
  then
    if options.silentEmpty then
      return nil
    end

    return widget.markdownBlock(
      "_Nessun luogo disponibile._"
    )
  end

  local markerData =
    markersToJavascript(markers)

  local listData =
    listItemsToJavascript(
      listItems
    )

  local caption =
    tostring(
      options.caption
      or "Mappa dei Luoghi"
    )

  local html = nil

  if options.listToggle then
    html = [[
      <div style="width:100%;">
        <div
          style="
            display:flex;
            align-items:center;
            gap:8px;
            margin:0 0 12px 0;
            font-family:ui-monospace,SFMono-Regular,Menlo,Consolas,monospace;
          ">
          <div
            id="luoghi-caption"
            style="font-size:1.55em;line-height:1.2;font-weight:700;">
          </div>

          <button id="luoghi-show-list" type="button"
            title="Elenco" aria-label="Elenco"
            style="border:0;background:transparent;padding:0 2px;font-size:1.05em;cursor:pointer;">📋</button>

          <button id="luoghi-show-map" type="button"
            title="Mappa" aria-label="Mappa"
            style="border:0;background:transparent;padding:0 2px;font-size:1.05em;cursor:pointer;">🗺️</button>
        </div>

        <div id="luoghi-list"
          style="font-family:ui-monospace,SFMono-Regular,Menlo,Consolas,monospace;line-height:1.55;">
        </div>

        <div id="luoghi-map"
          style="display:none;width:100%;height:450px;margin-top:10px;border-radius:6px;overflow:hidden;">
        </div>
      </div>
    ]]
  elseif options.details then
    html = [[
      <details id="luoghi-map-details" style="width:100%;">
        <summary id="luoghi-caption" style="cursor:pointer;"></summary>
        <div id="luoghi-map"
          style="width:100%;height:450px;margin-top:10px;border-radius:6px;overflow:hidden;">
        </div>
      </details>
    ]]
  else
    html = [[
      <div style="width:100%;">
        <div id="luoghi-map"
          style="width:100%;height:450px;border-radius:6px;overflow:hidden;">
        </div>
      </div>
    ]]
  end

  return widget.sandbox {
    html = html,

    script = string.format([==[
      const places = %s;
      const listPlaces = %s;
      const caption = %s;

      let map = null;
      let group = null;
      let mapReady = false;

      function escapeHtml(value) {
        return String(value || "")
          .replaceAll("&", "&amp;")
          .replaceAll("<", "&lt;")
          .replaceAll(">", "&gt;")
          .replaceAll('"', "&quot;")
          .replaceAll("'", "&#039;");
      }

      function syncHeight() {
        const body = document.body;
        const html = document.documentElement;

        globalThis.parent.postMessage(
          {
            type: "setHeight",
            height: Math.max(
              body.offsetHeight,
              html.offsetHeight
            )
          },
          "*"
        );
      }

      function renderList() {
        const target =
          document.getElementById(
            "luoghi-list"
          );

        if (!target) return;

        target.innerHTML = "";

        listPlaces.forEach(
          (place, index) => {
            if (index > 0) {
              target.appendChild(
                document.createTextNode(", ")
              );
            }

            const link =
              document.createElement("a");

            link.href = "#";
            link.textContent =
              place.label;

            link.addEventListener(
              "click",
              async event => {
                event.preventDefault();

                await syscall(
                  "editor.navigate",
                  place.name
                );
              }
            );

            target.appendChild(link);
          }
        );
      }

      function labelIcon(place) {
        return L.divIcon({
          className: "",
          html:
            '<div class="luogo-label-marker">'
            + escapeHtml(place.label)
            + '</div>',
          iconSize: null,
          iconAnchor: [0, 0]
        });
      }

      function fitMap() {
        if (!map || !group) return;

        if (places.length == 1) {
          map.setView(
            [places[0].lat, places[0].lon],
            13
          );
        } else if (places.length > 1) {
          map.fitBounds(
            group.getBounds(),
            {
              padding: [30, 30],
              maxZoom: 14
            }
          );
        }
      }

      async function initMap() {
        if (mapReady || places.length == 0) return;

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
            display:inline-block;
            transform:translate(-50%%,-110%%);
            padding:3px 6px;
            background:rgba(255,255,255,.90);
            border:1px solid #555;
            border-radius:4px;
            font:600 12px sans-serif;
            white-space:nowrap;
          }
        `;

        document.head.appendChild(style);

        map =
          L.map("luoghi-map");

        L.tileLayer(
          "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
          {
            maxZoom: 19,
            attribution:
              "&copy; OpenStreetMap contributors"
          }
        ).addTo(map);

        group =
          L.featureGroup()
            .addTo(map);

        for (const place of places) {
          const marker =
            L.marker(
              [place.lat, place.lon],
              {icon: labelIcon(place)}
            )
            .addTo(group);

          const popup =
            document.createElement("div");

          const title =
            document.createElement("strong");

          title.textContent =
            place.label;
          popup.appendChild(title);

          const button =
            document.createElement("button");

          button.type = "button";
          button.textContent = "Apri pagina";
          button.style.marginTop = "8px";

          button.addEventListener(
            "click",
            async () => {
              await syscall(
                "editor.navigate",
                place.name
              );
            }
          );

          popup.appendChild(button);
          marker.bindPopup(popup);
        }

        mapReady = true;
        fitMap();

        setTimeout(
          () => {
            map.invalidateSize();
            fitMap();
            syncHeight();
          },
          50
        );
      }

      async function showList() {
        const list =
          document.getElementById(
            "luoghi-list"
          );
        const mapElement =
          document.getElementById(
            "luoghi-map"
          );

        if (list) list.style.display = "block";
        if (mapElement) mapElement.style.display = "none";
        setTimeout(syncHeight, 20);
      }

      async function showMap() {
        if (places.length == 0) return;

        const list =
          document.getElementById(
            "luoghi-list"
          );
        const mapElement =
          document.getElementById(
            "luoghi-map"
          );

        if (list) list.style.display = "none";
        if (mapElement) mapElement.style.display = "block";

        await initMap();

        setTimeout(
          () => {
            if (map) {
              map.invalidateSize();
              fitMap();
            }
            syncHeight();
          },
          50
        );
      }

      async function main() {
        const captionElement =
          document.getElementById(
            "luoghi-caption"
          );

        if (captionElement) {
          captionElement.textContent = caption;
        }

        renderList();

        const listButton =
          document.getElementById(
            "luoghi-show-list"
          );
        const mapButton =
          document.getElementById(
            "luoghi-show-map"
          );

        if (listButton) {
          if (listPlaces.length == 0) {
            listButton.style.display = "none";
          } else {
            listButton.addEventListener(
              "click",
              showList
            );
          }
        }

        if (mapButton) {
          if (places.length == 0) {
            mapButton.style.display = "none";
          } else {
            mapButton.addEventListener(
              "click",
              showMap
            );
          }
        }

        const details =
          document.getElementById(
            "luoghi-map-details"
          );

        if (details) {
          details.addEventListener(
            "toggle",
            () => {
              if (details.open) {
                initMap().catch(console.error);
              }
              setTimeout(syncHeight, 50);
            }
          );
        }

        if (!listButton && places.length > 0) {
          await initMap();
        }

        syncHeight();
      }

      main().catch(error => {
        const element =
          document.getElementById(
            "luoghi-map"
          );

        if (element) {
          element.innerHTML =
            '<div style="padding:12px;font-family:sans-serif;">Errore mappa: '
            + String(error.message || error)
            + '</div>';
        }

        syncHeight();
      });
    ]==],
      markerData,
      listData,
      jsString(caption)
    ),

    markdown = caption
  }
end



-- ============================================================
-- GPX
--
-- Integrazione della precedente Mio_Diario_GPX 0.2-01.
--
-- Uso:
--   gpxMap("media/file.gpx")
--   gpxMap()
--
-- Su Diario/... senza path esplicito:
--   frontmatter date: YYYY-MM-DD
--            ->
--   media/YYYYMMDD*.gpx
--   media/YYYY-MM-DD*.gpx
--
-- Il GPX resta la fonte primaria. Non viene modificato.
-- ============================================================


-- ------------------------------------------------------------
-- Data e lookup GPX
-- ------------------------------------------------------------

local function gpxDateInfo(value)
  local raw =
    tostring(value or "")

  local year, month, day =
    string.match(
      raw,
      "(%d%d%d%d)%-(%d%d)%-(%d%d)"
    )

  if not year then
    return nil
  end

  return {
    iso =
      year .. "-"
      .. month .. "-"
      .. day,
    compact =
      year
      .. month
      .. day
  }
end


local function gpxPageDate(pageName)
  if not pageName
    or not string.startsWith(
      pageName,
      "Diario/"
    )
  then
    return nil
  end

  local rows = query[[
    from p = index.pages()
    where p.name == pageName
      and p.date
    select {
      date = p.date
    }
    limit 1
  ]]

  if not rows
    or not rows[1]
  then
    return nil
  end

  return gpxDateInfo(
    rows[1].date
  )
end


function gpxPathForDate(value)
  local d =
    gpxDateInfo(value)

  if not d then
    return nil
  end

  local compactPrefix =
    "media/" .. d.compact

  local isoPrefix =
    "media/" .. d.iso

  local docs = query[[
    from doc = index.documents()
    where
      (
        string.startsWith(
          doc.name,
          compactPrefix
        )
        or
        string.startsWith(
          doc.name,
          isoPrefix
        )
      )
      and string.lower(
        doc.extension or ""
      ) == "gpx"
    order by doc.name
    select {
      name = doc.name,
      extension = doc.extension
    }
  ]]

  if docs and docs[1] then
    return docs[1].name
  end

  return nil
end


function gpxPathForPage(pageName)
  pageName =
    pageName
    or editor.getCurrentPage()

  local d =
    gpxPageDate(pageName)

  if not d then
    return nil
  end

  return gpxPathForDate(
    d.iso
  )
end


function gpxCatalogByDate()
  local docs = query[[
    from doc = index.documents()
    where
      string.startsWith(
        doc.name,
        "media/"
      )
      and string.lower(
        doc.extension or ""
      ) == "gpx"
    order by doc.name
    select {
      name = doc.name,
      extension = doc.extension
    }
  ]]

  local byDate = {}

  for _, doc in ipairs(docs or {}) do
    local iso =
      string.match(
        doc.name,
        "^media/(%d%d%d%d%-%d%d%-%d%d)"
      )

    if not iso then
      local y, m, d =
        string.match(
          doc.name,
          "^media/(%d%d%d%d)(%d%d)(%d%d)"
        )

      if y then
        iso =
          y .. "-" .. m .. "-" .. d
      end
    end

    if iso
      and not byDate[iso]
    then
      byDate[iso] = doc.name
    end
  end

  return byDate
end


function gpxTracksForDiarioInfo(
  diarioInfo,
  byDate
)
  byDate =
    byDate
    or gpxCatalogByDate()

  local tracks = {}
  local seen = {}

  for _, info in ipairs(
    diarioInfo or {}
  ) do
    local d =
      gpxDateInfo(info.date)

    if d
      and byDate[d.iso]
      and not seen[d.iso]
    then
      table.insert(
        tracks,
        {
          date = d.iso,
          label =
            date.format(info.date),
          path =
            byDate[d.iso]
        }
      )

      seen[d.iso] = true
    end
  end

  return tracks
end


virtualPage.define {
  pattern = "GPX/(.+)",

  run = function(isoDate)
    if not string.match(
      isoDate,
      "^%d%d%d%d%-%d%d%-%d%d$"
    ) then
      return "# Percorso GPX\n\n_Data non valida._"
    end

    local diarioPage =
      "Diario/" .. isoDate

    local path =
      gpxPathForPage(
        diarioPage
      )

    if not path then
      return "# Percorso "
        .. isoDate
        .. "\n\n_Nessun file GPX associato._"
    end

    local escapedPath =
      string.gsub(
        path,
        '"',
        '\\"'
      )

    return "# Percorso "
      .. isoDate
      .. "\n\n"
      .. "${gpxMap(\""
      .. escapedPath
      .. "\", {details = false})}"
  end
}


-- ------------------------------------------------------------
-- Lettura GPX
-- ------------------------------------------------------------

local function gpxRead(path)
  if not path or path == "" then
    return nil
  end

  local ok, data =
    pcall(
      space.readDocument,
      path
    )

  if not ok or not data then
    return nil
  end

  return data
end


-- ------------------------------------------------------------
-- Renderer GPX
--
-- Compatibilità:
--   gpxMap("media/file.gpx")
--
-- Automatico:
--   gpxMap()
--   gpxMap({details = true})
-- ------------------------------------------------------------

function gpxMap(path, options)
  if type(path) == "table"
    and options == nil
  then
    options = path
    path = nil
  end

  options = options or {}

  if not path then
    path =
      gpxPathForPage(
        options.pageName
        or editor.getCurrentPage()
      )
  end

  if not path then
    if options.silentEmpty then
      return nil
    end

    return widget.markdownBlock(
      "_Nessun file GPX associato alla pagina._"
    )
  end

  local data =
    gpxRead(path)

  if not data then
    if options.silentEmpty then
      return nil
    end

    return widget.markdownBlock(
      "_Impossibile leggere `" ..
      tostring(path) ..
      "`._"
    )
  end

  local gpxBase64 =
    encoding.base64Encode(data)

  local html = nil

  if options.details then
    html = [[
      <details id="gpx-map-details" style="width:100%;">
        <summary style="cursor:pointer;">Mappa del percorso GPX</summary>

        <div
          id="gpx-stats"
          style="
            margin:10px 0 8px 0;
            font-family:sans-serif;
            font-size:0.95em;
          ">
        </div>

        <div
          id="gpx-map"
          style="
            width:100%;
            height:450px;
            border-radius:6px;
            overflow:hidden;
          ">
        </div>
      </details>
    ]]
  else
    html = [[
      <div style="width:100%;">
        <div
          id="gpx-stats"
          style="
            margin:0 0 8px 0;
            font-family:sans-serif;
            font-size:0.95em;
          ">
        </div>

        <div
          id="gpx-map"
          style="
            width:100%;
            height:450px;
            border-radius:6px;
            overflow:hidden;
          ">
        </div>
      </div>
    ]]
  end

  local script =
    [[
      const gpxBase64 = "]] ..
    gpxBase64 ..
    [[";

      let gpxMapInstance = null;
      let gpxStarted = false;


      function decodeBase64Utf8(value) {
        const binary = atob(value);
        const bytes =
          new Uint8Array(binary.length);

        for (
          let i = 0;
          i < binary.length;
          i++
        ) {
          bytes[i] =
            binary.charCodeAt(i);
        }

        return new TextDecoder(
          "utf-8"
        ).decode(bytes);
      }


      function haversine(a, b) {
        const R = 6371000;
        const rad =
          value =>
            value * Math.PI / 180;

        const dLat =
          rad(b.lat - a.lat);

        const dLon =
          rad(b.lon - a.lon);

        const lat1 = rad(a.lat);
        const lat2 = rad(b.lat);

        const h =
          Math.sin(dLat / 2) ** 2 +
          Math.cos(lat1) *
          Math.cos(lat2) *
          Math.sin(dLon / 2) ** 2;

        return 2 * R *
          Math.asin(
            Math.min(
              1,
              Math.sqrt(h)
            )
          );
      }


      function trackPoints(xml) {
        const nodes =
          Array.from(
            xml.querySelectorAll(
              "trkpt"
            )
          );

        return nodes
          .map(node => {
            const lat =
              Number(
                node.getAttribute("lat")
              );

            const lon =
              Number(
                node.getAttribute("lon")
              );

            const eleNode =
              node.querySelector("ele");

            const ele =
              eleNode
                ? Number(eleNode.textContent)
                : null;

            if (
              !Number.isFinite(lat) ||
              !Number.isFinite(lon)
            ) {
              return null;
            }

            return {
              lat,
              lon,
              ele:
                Number.isFinite(ele)
                  ? ele
                  : null
            };
          })
          .filter(Boolean);
      }


      function distanceMeters(points) {
        let total = 0;

        for (
          let i = 1;
          i < points.length;
          i++
        ) {
          total +=
            haversine(
              points[i - 1],
              points[i]
            );
        }

        return total;
      }


      // Dislivello coerente con Mio_Diario_GPX:
      // - copertura quota minima 90%
      // - aggregazione per finestre di circa 150 m
      // - oscillazioni <= 5 m ignorate
      function elevationStats(points) {
        if (points.length < 2) {
          return null;
        }

        const withElevation =
          points.filter(
            p => p.ele !== null
          );

        if (
          withElevation.length /
          points.length <
          0.90
        ) {
          return null;
        }

        const samples = [];

        let bucketDistance = 0;
        let bucketElevations = [];

        function closeBucket() {
          if (
            bucketElevations.length == 0
          ) {
            return;
          }

          const sum =
            bucketElevations.reduce(
              (a, b) => a + b,
              0
            );

          samples.push(
            sum /
            bucketElevations.length
          );

          bucketDistance = 0;
          bucketElevations = [];
        }

        if (points[0].ele !== null) {
          bucketElevations.push(
            points[0].ele
          );
        }

        for (
          let i = 1;
          i < points.length;
          i++
        ) {
          bucketDistance +=
            haversine(
              points[i - 1],
              points[i]
            );

          if (points[i].ele !== null) {
            bucketElevations.push(
              points[i].ele
            );
          }

          if (bucketDistance >= 150) {
            closeBucket();
          }
        }

        closeBucket();

        if (samples.length < 2) {
          return null;
        }

        let ascent = 0;
        let descent = 0;

        for (
          let i = 1;
          i < samples.length;
          i++
        ) {
          const delta =
            samples[i] -
            samples[i - 1];

          if (Math.abs(delta) <= 5) {
            continue;
          }

          if (delta > 0) {
            ascent += delta;
          } else {
            descent += -delta;
          }
        }

        return {
          ascent,
          descent
        };
      }


      function renderStats(points) {
        const target =
          document.getElementById(
            "gpx-stats"
          );

        if (!target) {
          return;
        }

        const distance =
          distanceMeters(points);

        const elevation =
          elevationStats(points);

        const parts = [
          "Distanza: " +
          (distance / 1000)
            .toFixed(1) +
          " km"
        ];

        if (elevation) {
          parts.push(
            "Dislivello: +" +
            Math.round(
              elevation.ascent
            ) +
            " m / -" +
            Math.round(
              elevation.descent
            ) +
            " m"
          );
        } else {
          parts.push(
            "Dislivello: n.d."
          );
        }

        target.textContent =
          parts.join(" · ");
      }


      function syncSandboxHeight() {
        const body = document.body;
        const html =
          document.documentElement;

        const height =
          Math.max(
            body.offsetHeight,
            html.offsetHeight
          );

        globalThis.parent.postMessage(
          {
            type: "setHeight",
            height: height
          },
          "*"
        );
      }


      async function initGpxMap() {
        if (gpxStarted) {
          if (gpxMapInstance) {
            gpxMapInstance.invalidateSize();
          }

          return;
        }

        gpxStarted = true;

        await loadJsByUrl(
          "https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"
        );

        await loadJsByUrl(
          "https://unpkg.com/@tmcw/togeojson@5.8.1/dist/togeojson.umd.js"
        );

        const css =
          document.createElement("link");

        css.rel = "stylesheet";
        css.href =
          "https://unpkg.com/leaflet@1.9.4/dist/leaflet.css";

        document.head.appendChild(css);

        const xmlText =
          decodeBase64Utf8(
            gpxBase64
          );

        const xml =
          new DOMParser()
            .parseFromString(
              xmlText,
              "application/xml"
            );

        const parserError =
          xml.querySelector(
            "parsererror"
          );

        if (parserError) {
          throw new Error(
            "GPX XML non valido"
          );
        }

        const geojson =
          toGeoJSON.gpx(xml);

        const points =
          trackPoints(xml);

        if (points.length == 0) {
          throw new Error(
            "nessun punto traccia nel GPX"
          );
        }

        renderStats(points);

        const map =
          L.map("gpx-map");

        gpxMapInstance = map;

        L.tileLayer(
          "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
          {
            maxZoom: 19,
            attribution:
              "&copy; OpenStreetMap contributors"
          }
        ).addTo(map);

        const layer =
          L.geoJSON(
            geojson,
            {
              style: {
                weight: 4,
                opacity: 0.85
              }
            }
          ).addTo(map);

        const start =
          points[0];

        const end =
          points[
            points.length - 1
          ];

        L.marker(
          [start.lat, start.lon]
        )
          .bindTooltip("Partenza")
          .addTo(map);

        L.marker(
          [end.lat, end.lon]
        )
          .bindTooltip("Arrivo")
          .addTo(map);

        const bounds =
          layer.getBounds();

        if (bounds.isValid()) {
          map.fitBounds(
            bounds,
            {
              padding: [25, 25],
              maxZoom: 16
            }
          );
        } else {
          map.setView(
            [start.lat, start.lon],
            14
          );
        }

        setTimeout(
          () => {
            map.invalidateSize();
            syncSandboxHeight();
          },
          50
        );
      }


      const details =
        document.getElementById(
          "gpx-map-details"
        );

      if (details) {
        details.addEventListener(
          "toggle",
          () => {
            if (details.open) {
              initGpxMap()
                .then(() => {
                  setTimeout(
                    syncSandboxHeight,
                    50
                  );
                })
                .catch(showGpxError);
            } else {
              setTimeout(
                syncSandboxHeight,
                50
              );
            }
          }
        );
      } else {
        initGpxMap()
          .catch(showGpxError);
      }


      function showGpxError(error) {
        const element =
          document.getElementById(
            "gpx-map"
          );

        if (element) {
          element.innerHTML =
            '<div style="padding:12px;font-family:sans-serif;">' +
            "Errore GPX: " +
            String(
              error.message || error
            ) +
            "</div>";
        }

        syncSandboxHeight();
      }
    ]]

  return widget.sandbox {
    html = html,
    script = script,
    markdown =
      "Mappa del percorso GPX"
  }
end



-- ============================================================
-- Componente UI unificato
-- ============================================================

local function tracksToJavascript(tracks)
  local rows = {}

  for _, track in ipairs(tracks or {}) do
    table.insert(
      rows,
      "{"
        .. "date:" .. jsString(track.date) .. ","
        .. "label:" .. jsString(track.label) .. ","
        .. "path:" .. jsString(track.path)
        .. "}"
    )
  end

  return "["
    .. table.concat(rows, ",")
    .. "]"
end


local function photosToJavascript(photos)
  local rows = {}

  for _, photo in ipairs(photos or {}) do
    table.insert(
      rows,
      "{"
        .. "date:" .. jsString(photo.date) .. ","
        .. "label:" .. jsString(photo.label) .. ","
        .. "embeddedUrl:" .. jsString(photo.embeddedUrl) .. ","
        .. "fullUrl:" .. jsString(photo.fullUrl)
        .. "}"
    )
  end

  return "["
    .. table.concat(rows, ",")
    .. "]"
end


function mediaLuoghiWidget(options)
  options = options or {}

  local mapPages =
    uniquePages(
      options.mapPages
      or {}
    )

  local listPages =
    uniquePages(
      options.listPages
      or {}
    )

  local markers =
    pagesToMarkers(
      mapPages
    )

  local listItems =
    pagesToListItems(
      listPages
    )

  local tracks =
    options.gpxTracks
    or {}

  local photos =
    options.photos
    or {}

  if #markers == 0
    and #listItems == 0
    and #tracks == 0
    and #photos == 0
  then
    if options.silentEmpty then
      return nil
    end

    return widget.markdownBlock(
      "_Nessun contenuto disponibile._"
    )
  end

  local caption =
    tostring(
      options.caption
      or "Luoghi visitati"
    )

  return widget.sandbox {
    html = [[
      <div style="width:100%;">
        <div
          style="
            display:flex;
            align-items:center;
            gap:9px;
            margin:0 0 12px 0;
            font-family:ui-monospace,SFMono-Regular,Menlo,Consolas,monospace;
          ">
          <div
            id="media-caption"
            style="font-size:1.55em;line-height:1.2;font-weight:700;">
          </div>

          <button id="media-list-button" type="button"
            title="Elenco" aria-label="Elenco"
            style="border:0;background:transparent;padding:0 2px;font-size:1.05em;cursor:pointer;">📋</button>

          <button id="media-map-button" type="button"
            title="Mappa dei luoghi" aria-label="Mappa dei luoghi"
            style="border:0;background:transparent;padding:0 2px;font-size:1.05em;cursor:pointer;">🗺️</button>

          <button id="media-gpx-button" type="button"
            title="Percorso GPX" aria-label="Percorso GPX"
            style="border:0;background:transparent;padding:0 2px;font-size:1.05em;cursor:pointer;">🧭</button>

          <button id="media-photo-button" type="button"
            title="PhotoGallery" aria-label="PhotoGallery"
            style="border:0;background:transparent;padding:0 2px;font-size:1.05em;cursor:pointer;">📷</button>
        </div>

        <div id="media-list"
          style="display:none;font-family:ui-monospace,SFMono-Regular,Menlo,Consolas,monospace;line-height:1.55;">
        </div>

        <div id="media-map"
          style="display:none;width:100%;height:450px;border-radius:6px;overflow:hidden;">
        </div>

        <div id="media-gpx"
          style="display:none;width:100%;font-family:ui-monospace,SFMono-Regular,Menlo,Consolas,monospace;">
          <select id="media-gpx-select"
            style="display:none;margin:0 0 8px 0;">
          </select>

          <div id="media-gpx-stats"
            style="margin:0 0 8px 0;">
          </div>

          <div id="media-gpx-map"
            style="width:100%;height:450px;border-radius:6px;overflow:hidden;">
          </div>
        </div>

        <div id="media-photo"
          style="display:none;width:100%;font-family:ui-monospace,SFMono-Regular,Menlo,Consolas,monospace;">
          <div
            style="display:flex;align-items:center;gap:10px;margin:0 0 8px 0;">
            <select id="media-photo-select"
              style="display:none;">
            </select>

            <a id="media-photo-full"
              target="_blank"
              rel="noopener noreferrer">
              Apri galleria ↗
            </a>
          </div>

          <iframe id="media-photo-frame"
            title="PhotoGallery"
            loading="lazy"
            style="display:block;width:100%;height:600px;border:0;border-radius:4px;">
          </iframe>
        </div>
      </div>
    ]],

    script = string.format([==[
      const caption = %s;
      const places = %s;
      const listPlaces = %s;
      const tracks = %s;
      const photos = %s;

      let placesMap = null;
      let placesGroup = null;
      let placesReady = false;
      let gpxMapInstance = null;
      let currentTrackPath = null;

      function syncHeight() {
        const body = document.body;
        const html = document.documentElement;

        globalThis.parent.postMessage(
          {
            type: "setHeight",
            height: Math.max(
              body.offsetHeight,
              html.offsetHeight
            )
          },
          "*"
        );
      }

      function hidePanels() {
        for (const id of [
          "media-list",
          "media-map",
          "media-gpx",
          "media-photo"
        ]) {
          const el =
            document.getElementById(id);

          if (el) {
            el.style.display = "none";
          }
        }
      }

      async function ensureLeaflet() {
        if (!globalThis.L) {
          await loadJsByUrl(
            "https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"
          );
        }

        if (!document.getElementById("media-leaflet-css")) {
          const css =
            document.createElement("link");

          css.id = "media-leaflet-css";
          css.rel = "stylesheet";
          css.href =
            "https://unpkg.com/leaflet@1.9.4/dist/leaflet.css";

          document.head.appendChild(css);
        }
      }

      function escapeHtml(value) {
        return String(value || "")
          .replaceAll("&", "&amp;")
          .replaceAll("<", "&lt;")
          .replaceAll(">", "&gt;")
          .replaceAll('"', "&quot;")
          .replaceAll("'", "&#039;");
      }

      function renderList() {
        const target =
          document.getElementById(
            "media-list"
          );

        target.innerHTML = "";

        listPlaces.forEach(
          (place, index) => {
            if (index > 0) {
              target.appendChild(
                document.createTextNode(", ")
              );
            }

            const link =
              document.createElement("a");

            link.href = "#";
            link.textContent =
              place.label;

            link.addEventListener(
              "click",
              async event => {
                event.preventDefault();

                await syscall(
                  "editor.navigate",
                  place.name
                );
              }
            );

            target.appendChild(link);
          }
        );
      }

      function placeIcon(place) {
        return L.divIcon({
          className: "",
          html:
            '<div style="'
            + 'display:inline-block;'
            + 'transform:translate(-50%%,-110%%);'
            + 'padding:3px 6px;'
            + 'background:rgba(255,255,255,.90);'
            + 'border:1px solid #555;'
            + 'border-radius:4px;'
            + 'font:600 12px sans-serif;'
            + 'white-space:nowrap;">'
            + escapeHtml(place.label)
            + '</div>',
          iconSize: null,
          iconAnchor: [0, 0]
        });
      }

      function fitPlaces() {
        if (!placesMap || !placesGroup) return;

        if (places.length == 1) {
          placesMap.setView(
            [places[0].lat, places[0].lon],
            13
          );
        } else if (places.length > 1) {
          placesMap.fitBounds(
            placesGroup.getBounds(),
            {
              padding: [30, 30],
              maxZoom: 14
            }
          );
        }
      }

      async function initPlacesMap() {
        if (placesReady || places.length == 0) return;

        await ensureLeaflet();

        placesMap =
          L.map("media-map");

        L.tileLayer(
          "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
          {
            maxZoom: 19,
            attribution:
              "&copy; OpenStreetMap contributors"
          }
        ).addTo(placesMap);

        placesGroup =
          L.featureGroup()
            .addTo(placesMap);

        for (const place of places) {
          const marker =
            L.marker(
              [place.lat, place.lon],
              {icon: placeIcon(place)}
            )
            .addTo(placesGroup);

          const popup =
            document.createElement("div");

          const title =
            document.createElement("strong");

          title.textContent =
            place.label;

          popup.appendChild(title);

          const button =
            document.createElement("button");

          button.type = "button";
          button.textContent =
            "Apri pagina";
          button.style.marginTop =
            "8px";

          button.addEventListener(
            "click",
            async () => {
              await syscall(
                "editor.navigate",
                place.name
              );
            }
          );

          popup.appendChild(button);
          marker.bindPopup(popup);
        }

        placesReady = true;
        fitPlaces();
      }

      function decodeDocumentPayload(data) {
        if (typeof data === "string") {
          return data;
        }

        if (data instanceof Uint8Array) {
          return new TextDecoder("utf-8")
            .decode(data);
        }

        if (data instanceof ArrayBuffer) {
          return new TextDecoder("utf-8")
            .decode(new Uint8Array(data));
        }

        if (Array.isArray(data)) {
          return new TextDecoder("utf-8")
            .decode(new Uint8Array(data));
        }

        if (data && Array.isArray(data.data)) {
          return new TextDecoder("utf-8")
            .decode(new Uint8Array(data.data));
        }

        throw new Error(
          "Formato documento GPX non supportato"
        );
      }

      function haversine(a, b) {
        const R = 6371000;
        const rad =
          value =>
            value * Math.PI / 180;

        const dLat =
          rad(b.lat - a.lat);
        const dLon =
          rad(b.lon - a.lon);
        const lat1 = rad(a.lat);
        const lat2 = rad(b.lat);

        const h =
          Math.sin(dLat / 2) ** 2
          + Math.cos(lat1)
          * Math.cos(lat2)
          * Math.sin(dLon / 2) ** 2;

        return 2 * R
          * Math.asin(
            Math.min(
              1,
              Math.sqrt(h)
            )
          );
      }

      function trackPoints(xml) {
        return Array.from(
          xml.querySelectorAll("trkpt")
        )
          .map(node => {
            const lat =
              Number(
                node.getAttribute("lat")
              );
            const lon =
              Number(
                node.getAttribute("lon")
              );
            const eleNode =
              node.querySelector("ele");
            const ele =
              eleNode
                ? Number(eleNode.textContent)
                : null;

            if (
              !Number.isFinite(lat)
              || !Number.isFinite(lon)
            ) {
              return null;
            }

            return {
              lat,
              lon,
              ele:
                Number.isFinite(ele)
                  ? ele
                  : null
            };
          })
          .filter(Boolean);
      }

      function distanceMeters(points) {
        let total = 0;

        for (
          let i = 1;
          i < points.length;
          i++
        ) {
          total +=
            haversine(
              points[i - 1],
              points[i]
            );
        }

        return total;
      }

      function elevationStats(points) {
        if (points.length < 2) return null;

        const withElevation =
          points.filter(
            p => p.ele !== null
          );

        if (
          withElevation.length
          / points.length
          < 0.90
        ) {
          return null;
        }

        const samples = [];
        let bucketDistance = 0;
        let bucketElevations = [];

        function closeBucket() {
          if (bucketElevations.length == 0) return;

          const sum =
            bucketElevations.reduce(
              (a, b) => a + b,
              0
            );

          samples.push(
            sum / bucketElevations.length
          );

          bucketDistance = 0;
          bucketElevations = [];
        }

        if (points[0].ele !== null) {
          bucketElevations.push(
            points[0].ele
          );
        }

        for (
          let i = 1;
          i < points.length;
          i++
        ) {
          bucketDistance +=
            haversine(
              points[i - 1],
              points[i]
            );

          if (points[i].ele !== null) {
            bucketElevations.push(
              points[i].ele
            );
          }

          if (bucketDistance >= 150) {
            closeBucket();
          }
        }

        closeBucket();

        if (samples.length < 2) return null;

        let ascent = 0;
        let descent = 0;

        for (
          let i = 1;
          i < samples.length;
          i++
        ) {
          const delta =
            samples[i]
            - samples[i - 1];

          if (Math.abs(delta) <= 5) continue;

          if (delta > 0) {
            ascent += delta;
          } else {
            descent += -delta;
          }
        }

        return {
          ascent,
          descent
        };
      }

      function renderGpxStats(points) {
        const target =
          document.getElementById(
            "media-gpx-stats"
          );

        const distance =
          distanceMeters(points);

        const elevation =
          elevationStats(points);

        const parts = [
          "Distanza: "
          + (distance / 1000).toFixed(1)
          + " km"
        ];

        if (elevation) {
          parts.push(
            "Dislivello: +"
            + Math.round(elevation.ascent)
            + " m / -"
            + Math.round(elevation.descent)
            + " m"
          );
        }

        target.textContent =
          parts.join(" · ");
      }

      async function renderTrack(track) {
        if (!track || !track.path) return;

        if (
          currentTrackPath === track.path
          && gpxMapInstance
        ) {
          gpxMapInstance.invalidateSize();
          return;
        }

        await ensureLeaflet();

        if (!globalThis.toGeoJSON) {
          await loadJsByUrl(
            "https://unpkg.com/@tmcw/togeojson@5.8.1/dist/togeojson.umd.js"
          );
        }

        const raw =
          await syscall(
            "space.readDocument",
            track.path
          );

        const xmlText =
          decodeDocumentPayload(raw);

        const xml =
          new DOMParser()
            .parseFromString(
              xmlText,
              "application/xml"
            );

        if (xml.querySelector("parsererror")) {
          throw new Error(
            "GPX XML non valido"
          );
        }

        const geojson =
          toGeoJSON.gpx(xml);

        const points =
          trackPoints(xml);

        if (points.length == 0) {
          throw new Error(
            "Nessun punto traccia nel GPX"
          );
        }

        renderGpxStats(points);

        if (gpxMapInstance) {
          gpxMapInstance.remove();
        }

        gpxMapInstance =
          L.map("media-gpx-map");

        L.tileLayer(
          "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
          {
            maxZoom: 19,
            attribution:
              "&copy; OpenStreetMap contributors"
          }
        ).addTo(gpxMapInstance);

        const layer =
          L.geoJSON(
            geojson,
            {
              style: {
                weight: 4,
                opacity: 0.85
              }
            }
          ).addTo(gpxMapInstance);

        const bounds =
          layer.getBounds();

        if (bounds.isValid()) {
          gpxMapInstance.fitBounds(
            bounds,
            {
              padding: [25, 25],
              maxZoom: 16
            }
          );
        }

        currentTrackPath =
          track.path;
      }

      function setupGpxSelect() {
        const select =
          document.getElementById(
            "media-gpx-select"
          );

        tracks.forEach(
          (track, index) => {
            const option =
              document.createElement("option");

            option.value =
              String(index);
            option.textContent =
              track.label || track.date;

            select.appendChild(option);
          }
        );

        if (tracks.length > 1) {
          select.style.display =
            "inline-block";
        }

        select.addEventListener(
          "change",
          () => {
            renderTrack(
              tracks[
                Number(select.value)
              ]
            )
              .then(syncHeight)
              .catch(showError);
          }
        );
      }

      function setupPhotoSelect() {
        const select =
          document.getElementById(
            "media-photo-select"
          );

        photos.forEach(
          (photo, index) => {
            const option =
              document.createElement("option");

            option.value =
              String(index);
            option.textContent =
              photo.label || photo.date;

            select.appendChild(option);
          }
        );

        if (photos.length > 1) {
          select.style.display =
            "inline-block";
        }

        select.addEventListener(
          "change",
          () => {
            showPhoto(
              Number(select.value)
            );
          }
        );
      }

      function showPhoto(index) {
        const photo =
          photos[index];

        if (!photo) return;

        const frame =
          document.getElementById(
            "media-photo-frame"
          );

        const full =
          document.getElementById(
            "media-photo-full"
          );

        if (frame.src !== photo.embeddedUrl) {
          frame.src =
            photo.embeddedUrl;
        }

        full.href =
          photo.fullUrl;
      }

      async function showList() {
        hidePanels();
        document.getElementById(
          "media-list"
        ).style.display = "block";
        setTimeout(syncHeight, 20);
      }

      async function showMap() {
        hidePanels();
        document.getElementById(
          "media-map"
        ).style.display = "block";

        await initPlacesMap();

        setTimeout(
          () => {
            if (placesMap) {
              placesMap.invalidateSize();
              fitPlaces();
            }
            syncHeight();
          },
          50
        );
      }

      async function showGpx() {
        hidePanels();
        document.getElementById(
          "media-gpx"
        ).style.display = "block";

        const select =
          document.getElementById(
            "media-gpx-select"
          );

        await renderTrack(
          tracks[
            Number(select.value || 0)
          ]
        );

        setTimeout(
          () => {
            if (gpxMapInstance) {
              gpxMapInstance.invalidateSize();
            }
            syncHeight();
          },
          50
        );
      }

      async function showPhotos() {
        hidePanels();
        document.getElementById(
          "media-photo"
        ).style.display = "block";

        const select =
          document.getElementById(
            "media-photo-select"
          );

        showPhoto(
          Number(select.value || 0)
        );

        setTimeout(syncHeight, 50);
      }

      function showError(error) {
        const target =
          document.getElementById(
            "media-gpx-stats"
          );

        if (target) {
          target.textContent =
            "Errore GPX: "
            + String(
              error.message || error
            );
        }

        syncHeight();
      }

      async function main() {
        document.getElementById(
          "media-caption"
        ).textContent = caption;

        renderList();
        setupGpxSelect();
        setupPhotoSelect();

        const listButton =
          document.getElementById(
            "media-list-button"
          );
        const mapButton =
          document.getElementById(
            "media-map-button"
          );
        const gpxButton =
          document.getElementById(
            "media-gpx-button"
          );
        const photoButton =
          document.getElementById(
            "media-photo-button"
          );

        if (listPlaces.length == 0) {
          listButton.style.display = "none";
        } else {
          listButton.addEventListener(
            "click",
            showList
          );
        }

        if (places.length == 0) {
          mapButton.style.display = "none";
        } else {
          mapButton.addEventListener(
            "click",
            showMap
          );
        }

        if (tracks.length == 0) {
          gpxButton.style.display = "none";
        } else {
          gpxButton.addEventListener(
            "click",
            () => showGpx().catch(showError)
          );
        }

        if (photos.length == 0) {
          photoButton.style.display = "none";
        } else {
          photoButton.addEventListener(
            "click",
            showPhotos
          );
        }

        if (listPlaces.length > 0) {
          await showList();
        } else if (places.length > 0) {
          await showMap();
        } else if (tracks.length > 0) {
          await showGpx();
        } else if (photos.length > 0) {
          await showPhotos();
        }

        syncHeight();
      }

      main().catch(showError);
    ]==],
      jsString(caption),
      markersToJavascript(
        pagesToMarkers(mapPages)
      ),
      listItemsToJavascript(
        pagesToListItems(listPages)
      ),
      tracksToJavascript(tracks),
      photosToJavascript(photos)
    ),

    markdown = caption
  }
end


-- ============================================================
-- Bottom Widget
--
-- Nessun listener automatico in questa libreria.
-- L'ordine viene orchestrato da `Mio_Diario`.
-- ============================================================
```

## Note versione 0.6-01

* ripristinata la compatibilità pubblica di `${luoghiMap()}`;
* `Viaggi/...` aggrega i luoghi delle pagine Diario del viaggio;
* le query `index.documents()` filtrano `gpx` direttamente nell'indice;
* definite esplicitamente le implementazioni canoniche;
* dichiarate le dipendenze `Mio_Diario` e `DateFormat`;
* corretta la documentazione dei Bottom Widget.

## Note versione 0.6-00

* rimossi i listener automatici Mappa Luoghi e GPX;
* ripristinata la sola Virtual Page `GPX/YYYY-MM-DD` con Leaflet;
* `luoghiMap()` e `gpxMap()` restano renderer autonomi;
* aggiunto `mediaLuoghiWidget()` per `📋 / 🗺️ / 🧭 / 📷`;
* il GPX del componente viene letto solo all'apertura di `🧭` tramite `space.readDocument`;
* eliminati i lookup locali duplicati dei luoghi;
* lookup batch dei luoghi e catalogo GPX unico per le pagine Viaggio.

## Note versione 0.4-00

* Il titolo del componente Luoghi passa da `1.5em` a `1.65em`.
* Aggiunta la Virtual Page indice `GPX`, così il breadcrumb `GPX` non è più un link morto.
* `GPX/YYYY-MM-DD` mostra navigazione verso il GPX precedente e successivo realmente esistenti, non semplicemente data ± 1 giorno.
* La pagina GPX dettaglio include anche `📖 Diario` verso la giornata corrispondente.
* Il catalogo GPX è ricavato da `index.documents()` e non crea pagine Markdown.

## Note versione 0.3-00

* La modalità `listToggle` non usa più `<details>`: mostra sempre l'elenco come vista iniziale.
* Titolo e pulsanti `📋` / `🗺️` sono sulla stessa riga; il titolo usa una dimensione equivalente a un heading `##`.
* La vista Mappa continua a inizializzare Leaflet solo quando richiesta.
* Aggiunta la Virtual Page `GPX/YYYY-MM-DD`, che riusa `gpxPathForPage()` e `gpxMap()` senza creare pagine Markdown.
* La Virtual Page GPX mostra la mappa direttamente, senza `<details>`.

## Note versione 0.2-00

* `luoghiMap()` accetta `caption`, `list` e `listToggle`.
* Con `listToggle = true`, un unico `<details>` alterna **Elenco** e **Mappa** senza duplicare l'aggregazione dei luoghi.
* L'elenco include anche luoghi privi di coordinate; la vista Mappa usa soltanto quelli con `coordinate`.
* I link dell'elenco usano `editor.navigate` dal sandbox e aprono direttamente la pagina `luoghi/...`.
* La mappa automatica separata sulle pagine `Viaggi/...` è rimossa: viene orchestrata da `Mio_Diario` come primo Bottom Widget del viaggio.
* Il comportamento delle mappe su `Diario/...` e `luoghi/...` resta invariato.

## Note versione 0.1-00

* Integrata nella stessa libreria la funzione `gpxMap()` derivata da `Mio_Diario_GPX 0.2-01`; non serve più una libreria GPX separata.
* `gpxMap("media/file.gpx")` resta disponibile per l'uso esplicito.
* Su `Diario/...`, `gpxMap()` senza path usa `date` indicizzato e cerca tramite `index.documents()` prima `media/YYYYMMDD*.gpx`, poi `media/YYYY-MM-DD*.gpx`.
* Il Bottom Widget GPX viene creato soltanto se esiste un documento GPX compatibile con la data della pagina.
* Il GPX è visualizzato in `<details>` chiuso di default con summary `Mappa del percorso GPX`.
* Il rendering riusa Leaflet 1.9.4 e toGeoJSON 5.8.1; il file GPX rimane la fonte primaria e non viene modificato.
* Mantenute le statistiche essenziali: distanza e dislivello positivo/negativo; per il dislivello sono usati copertura quota minima 90%, aggregazione a circa 150 m e filtro delle oscillazioni entro 5 m.
* Il ridimensionamento del sandbox all'apertura/chiusura riusa la soluzione già collaudata per `Mappa dei Luoghi`.

## Note versione 0.0-05

* Nelle pagine `Diario/...` il Bottom Widget usa ora `luoghiMap({silentEmpty = true, details = true})`.
* Se non esistono luoghi con coordinate valide, il widget continua a restituire `nil` e non occupa spazio.
* Le pagine `luoghi/...` mantengono il comportamento già collaudato; `Viaggi/...` resta invariato.

## Note versione 0.0-04

* Corretto il rendering della mappa dentro `<details>`: all'apertura e alla chiusura il sandbox comunica esplicitamente a SilverBullet la nuova altezza dell'iframe.
* All'apertura restano `map.invalidateSize()` e il ricalcolo dell'inquadramento Leaflet.
* Con `silentEmpty = true`, in assenza di coordinate la funzione restituisce `nil` e non crea più un widget vuoto.

## Note versione 0.0-03

* `luoghiMap({details = true})` racchiude la mappa in `<details>` chiuso di default con summary `Mappa dei Luoghi`.
* All'apertura viene eseguito `map.invalidateSize()` e viene ripristinato l'inquadramento Leaflet.
* Il Bottom Widget automatico della libreria resta attivo su `Diario/...` e `Viaggi/...`; sulle pagine `luoghi/...` l'ordine viene gestito da `Mio_Diario`.

## Note versione 0.0-02

Questa è la prima versione di test.

Sono intenzionalmente esclusi:

- discendenti ricorsivi delle pagine luogo;
- gestione automatica delle sovrapposizioni tra etichette molto vicine;
- icone o stili specifici basati su `tipoAmministrativo` o sull'attributo `tipo`;
- lettura o geocodifica automatica da Wikipedia;
- modifica automatica delle coordinate nel frontmatter.

Le funzionalità escluse potranno essere valutate dopo il test del Bottom Widget e della navigazione su dati reali.
