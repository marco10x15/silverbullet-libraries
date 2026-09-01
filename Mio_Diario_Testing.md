---
name: "Library/MG/Mio_Diario_Testing"
tags: meta/library
description: "Widget, funzioni, in fase di sviluppo. ATTENZIONE! PERICOLO!"
version: "0.0-00"
versionDate: 2026-09-01
pageDecoration.prefix: "⚠️ "
share.uri: "github:marco10x15/silverbullet-libraries/Mio_Diario_Testing.md"
---

# ⚠️️⚠️ Funzioni Diario in Sviluppo ⚠⚠

**Versione:** 0.0-00 — 01.09.2026

La libreria è autonoma.

## Main Features

## Configurazione



## Uso


## Implementazione

```space-lua
-- ============================================================
-- Mio Diario - GPX Map
-- Versione: 0.1
--
-- Visualizza su una mappa interattiva un percorso contenuto
-- in un documento GPX dello Space SilverBullet.
--
-- Funzioni:
--   - mappa OpenStreetMap tramite Leaflet
--   - traccia del percorso GPX
--   - zoom automatico sul percorso
--   - marker di partenza e arrivo
--   - distanza totale
--   - dislivello positivo
--   - dislivello negativo
--
-- Utilizzo:
--
--   ${gpxMap("media/20260830.gpx")}
--
-- Il file GPX rimane la fonte primaria dei dati.
-- La funzione non modifica né il GPX né la pagina Diario.
--
-- Dipendenze esterne:
--   Leaflet 1.9.4
--   @tmcw/togeojson 5.8.1
--   OpenStreetMap
--
-- Le librerie JavaScript vengono attualmente caricate
-- da unpkg.com. Le tile cartografiche vengono caricate
-- da OpenStreetMap.
--
-- Requisiti GPX:
--   - GPX valido
--   - almeno una traccia/percorso con coordinate
--   - <ele> necessario per il calcolo dei dislivelli
--
-- Note:
--   Distanza e dislivelli sono calcolati al momento del
--   rendering e non vengono salvati nel Markdown.
-- ============================================================


function gpxMap(path)
  if not path or path == "" then
    return "GPX: percorso non specificato"
  end

  local data = space.readDocument(path)

  if not data then
    return "GPX non leggibile: " .. path
  end

  local gpx = encoding.utf8Decode(data)

  return widget.sandbox {
    html = [[
      <div style="width:100%;">
        <div
          id="gpx-map"
          style="
            width:100%;
            height:450px;
            border-radius:6px;
            overflow:hidden;
          ">
        </div>

        <div
          id="gpx-info"
          style="
            padding:8px 2px 2px 2px;
            font-family:sans-serif;
            font-size:0.9em;
          ">
          Caricamento percorso...
        </div>
      </div>
    ]],

    script = string.format([==[
      const gpxText = %q;


      // --------------------------------------------------------
      // Distanza tra due coordinate con formula di Haversine
      // --------------------------------------------------------

      function distanceMeters(a, b) {
        const R = 6371000;
        const rad = Math.PI / 180;

        const lat1 = a[1] * rad;
        const lat2 = b[1] * rad;

        const dLat = (b[1] - a[1]) * rad;
        const dLon = (b[0] - a[0]) * rad;

        const h =
          Math.sin(dLat / 2) ** 2 +
          Math.cos(lat1) *
          Math.cos(lat2) *
          Math.sin(dLon / 2) ** 2;

        return 2 * R * Math.asin(Math.sqrt(h));
      }


      // --------------------------------------------------------
      // Estrae tutte le coordinate LineString dal GeoJSON
      // --------------------------------------------------------

      function collectTracks(geojson) {
        const tracks = [];

        for (const feature of geojson.features || []) {
          const geometry = feature.geometry;

          if (!geometry) {
            continue;
          }

          if (geometry.type === "LineString") {
            tracks.push(geometry.coordinates);
          }

          if (geometry.type === "MultiLineString") {
            for (const line of geometry.coordinates) {
              tracks.push(line);
            }
          }
        }

        return tracks;
      }


      // --------------------------------------------------------
      // Calcola distanza e dislivelli
      //
      // Coordinate GeoJSON:
      //   [longitudine, latitudine, elevazione]
      // --------------------------------------------------------

      function calculateStats(tracks) {
        let distance = 0;
        let ascent = 0;
        let descent = 0;

        for (const track of tracks) {
          for (let i = 1; i < track.length; i++) {
            const previous = track[i - 1];
            const current = track[i];

            distance += distanceMeters(previous, current);

            if (
              previous.length >= 3 &&
              current.length >= 3 &&
              Number.isFinite(previous[2]) &&
              Number.isFinite(current[2])
            ) {
              const difference = current[2] - previous[2];

              if (difference > 0) {
                ascent += difference;
              } else {
                descent += -difference;
              }
            }
          }
        }

        return {
          distance,
          ascent,
          descent
        };
      }


      // --------------------------------------------------------
      // Marker semplici senza dipendenza dalle immagini Leaflet
      // --------------------------------------------------------

      function markerIcon(label) {
        return L.divIcon({
          className: "",
          html:
            '<div style="' +
            'width:26px;' +
            'height:26px;' +
            'border-radius:50%%;' +
            'background:white;' +
            'border:2px solid #333;' +
            'display:flex;' +
            'align-items:center;' +
            'justify-content:center;' +
            'font:bold 13px sans-serif;' +
            'box-sizing:border-box;' +
            '">' +
            label +
            '</div>',
          iconSize: [26, 26],
          iconAnchor: [13, 13]
        });
      }


      // --------------------------------------------------------
      // Inizializzazione
      // --------------------------------------------------------

      async function main() {
        await loadJsByUrl(
          "https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"
        );

        await loadJsByUrl(
          "https://unpkg.com/@tmcw/togeojson@5.8.1/dist/togeojson.umd.js"
        );

        const css = document.createElement("link");

        css.rel = "stylesheet";
        css.href =
          "https://unpkg.com/leaflet@1.9.4/dist/leaflet.css";

        document.head.appendChild(css);


        // ------------------------------------------------------
        // Parsing GPX
        // ------------------------------------------------------

        const xml = new DOMParser().parseFromString(
          gpxText,
          "application/xml"
        );

        if (xml.querySelector("parsererror")) {
          throw new Error("file GPX XML non valido");
        }

        const geojson = toGeoJSON.gpx(xml);

        const tracks = collectTracks(geojson);

        if (tracks.length === 0) {
          throw new Error(
            "il GPX non contiene una traccia visualizzabile"
          );
        }


        // ------------------------------------------------------
        // Mappa
        // ------------------------------------------------------

        const map = L.map("gpx-map");

        L.tileLayer(
          "https://tile.openstreetmap.org/{z}/{x}/{y}.png",
          {
            maxZoom: 19,
            attribution: "&copy; OpenStreetMap contributors"
          }
        ).addTo(map);

        const layer = L.geoJSON(geojson, {
          style: {
            weight: 4
          }
        }).addTo(map);

        const bounds = layer.getBounds();

        if (!bounds.isValid()) {
          throw new Error(
            "impossibile determinare i limiti del percorso"
          );
        }

        map.fitBounds(bounds, {
          padding: [20, 20]
        });


        // ------------------------------------------------------
        // Partenza e arrivo
        // ------------------------------------------------------

        const firstTrack = tracks[0];
        const lastTrack = tracks[tracks.length - 1];

        const start = firstTrack[0];
        const finish = lastTrack[lastTrack.length - 1];

        L.marker(
          [start[1], start[0]],
          {
            icon: markerIcon("P")
          }
        )
          .addTo(map)
          .bindTooltip("Partenza");

        L.marker(
          [finish[1], finish[0]],
          {
            icon: markerIcon("A")
          }
        )
          .addTo(map)
          .bindTooltip("Arrivo");


        // ------------------------------------------------------
        // Statistiche
        // ------------------------------------------------------

        const stats = calculateStats(tracks);

        const distanceKm =
          (stats.distance / 1000).toFixed(1);

        const ascent =
          Math.round(stats.ascent);

        const descent =
          Math.round(stats.descent);

        document.getElementById("gpx-info").textContent =
          distanceKm +
          " km · ↑ " +
          ascent +
          " m · ↓ " +
          descent +
          " m";
      }


      main().catch(function(error) {
        document.getElementById("gpx-info").textContent =
          "Errore GPX: " + error.message;
      });
    ]==], gpx),

    markdown = "GPX: " .. path
  }
end
```
end
```
