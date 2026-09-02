---
name: "Library/MG/Mio_Diario_Testing"
tags: meta/library
description: "Widget, funzioni, in fase di sviluppo. ATTENZIONE! PERICOLO!"
version: "0.2-00"
versionDate: 2026-09-02
pageDecoration.prefix: "⚠️ "
share.uri: "github:marco10x15/silverbullet-libraries/Mio_Diario_Testing.md"
---

# ⚠️️⚠️ Funzioni Diario in Sviluppo ⚠⚠

**Versione:** 0.2-00 — 02.09.2026

La libreria è autonoma.

## Main Features

## Configurazione



## Uso


## Implementazione

```space-lua
-- ============================================================
-- Mio Diario - GPX Map
-- Versione: 0.2-00
--
-- Visualizza una traccia GPX in una pagina SilverBullet v2.
--
-- UTILIZZO
--
--   ${gpxMap("media/20260830.gpx")}
--
-- FUNZIONI
--
--   - mappa interattiva OpenStreetMap
--   - traccia GPX originale
--   - zoom automatico sul percorso
--   - marker P = partenza
--   - marker A = arrivo
--   - distanza totale
--   - durata, quando disponibile
--   - quota minima e massima
--   - dislivello positivo e negativo quando attendibile
--
-- FONTE DEI DATI
--
--   Il file GPX nello Space è la fonte primaria.
--   La funzione non modifica il GPX né la pagina Diario.
--
-- LETTURA
--
--   space.readDocument(path)
--     -> encoding.utf8Decode()
--     -> DOMParser
--     -> toGeoJSON
--     -> Leaflet
--
-- GPX SUPPORTATI
--
--   Sono supportati i GPX correttamente interpretabili
--   dal DOMParser utilizzato nel widget.
--
--   I vecchi GPX GeoSetter che generano errori XML nel
--   DOMParser non vengono corretti o normalizzati
--   automaticamente.
--
-- ALTIMETRIA
--
--   I dati <ele> dei dispositivi GPS possono contenere
--   oscillazioni significative.
--
--   Per evitare dislivelli fuorvianti vengono applicati:
--
--     copertura minima quote : 90%
--     smoothing spaziale     : 150 m
--     oscillazione massima   : 5
--
--   Lo smoothing interessa esclusivamente le statistiche.
--   La geometria mostrata sulla mappa rimane quella originale.
--
--   Se il profilo non supera i controlli:
--
--     dislivello n.d.
--
-- DIPENDENZE ESTERNE
--
--   Leaflet 1.9.4
--   @tmcw/togeojson 5.8.1
--   OpenStreetMap
--
--   JavaScript/CSS vengono caricati da unpkg.com.
--   Le tile vengono caricate da OpenStreetMap.
--
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
            padding:8px 2px 0 2px;
            font-family:sans-serif;
            font-size:0.9em;
          ">
          Caricamento percorso...
        </div>

        <div
          id="gpx-elevation"
          style="
            padding:2px 2px 2px 2px;
            font-family:sans-serif;
            font-size:0.82em;
            opacity:0.75;
          ">
        </div>

      </div>
    ]],

    script = string.format([==[

      const gpxText = %q;

      const MIN_ELEVATION_COVERAGE = 0.90;
      const ELEVATION_SMOOTH_METERS = 150;
      const MAX_ELEVATION_OSCILLATION = 5;


      // ========================================================
      // Distanza Haversine
      // ========================================================

      function distanceMeters(a, b) {
        const R = 6371000;
        const rad = Math.PI / 180;

        const lat1 = a[1] * rad;
        const lat2 = b[1] * rad;

        const dLat =
          (b[1] - a[1]) * rad;

        const dLon =
          (b[0] - a[0]) * rad;

        const h =
          Math.sin(dLat / 2) ** 2 +
          Math.cos(lat1) *
          Math.cos(lat2) *
          Math.sin(dLon / 2) ** 2;

        return 2 * R * Math.asin(
          Math.min(1, Math.sqrt(h))
        );
      }


      // ========================================================
      // Estrazione tracce dal GeoJSON
      // ========================================================

      function collectTracks(geojson) {
        const tracks = [];

        for (const feature of geojson.features || []) {
          const geometry = feature.geometry;

          if (!geometry) {
            continue;
          }

          if (
            geometry.type === "LineString" &&
            geometry.coordinates.length > 0
          ) {
            tracks.push(
              geometry.coordinates
            );
          }

          if (geometry.type === "MultiLineString") {
            for (const line of geometry.coordinates) {
              if (line.length > 0) {
                tracks.push(line);
              }
            }
          }
        }

        return tracks;
      }


      // ========================================================
      // Distanza totale
      //
      // Tracce/segmenti distinti non vengono collegati
      // artificialmente fra loro.
      // ========================================================

      function calculateDistance(tracks) {
        let total = 0;

        for (const track of tracks) {
          for (let i = 1; i < track.length; i++) {
            total += distanceMeters(
              track[i - 1],
              track[i]
            );
          }
        }

        return total;
      }


      // ========================================================
      // Durata
      // ========================================================

      function calculateDuration(xml) {
        let first = null;
        let last = null;

        for (
          const point of
          xml.querySelectorAll("trkpt")
        ) {
          const node =
            point.querySelector("time");

          if (!node) {
            continue;
          }

          const value =
            Date.parse(node.textContent);

          if (!Number.isFinite(value)) {
            continue;
          }

          if (
            first === null ||
            value < first
          ) {
            first = value;
          }

          if (
            last === null ||
            value > last
          ) {
            last = value;
          }
        }

        if (
          first === null ||
          last === null ||
          last <= first
        ) {
          return null;
        }

        return last - first;
      }


      function formatDuration(milliseconds) {
        const totalMinutes =
          Math.round(
            milliseconds / 60000
          );

        const hours =
          Math.floor(
            totalMinutes / 60
          );

        const minutes =
          totalMinutes %% 60;

        if (hours === 0) {
          return minutes + " min";
        }

        if (minutes === 0) {
          return hours + " h";
        }

        return (
          hours +
          " h " +
          minutes +
          " min"
        );
      }


      // ========================================================
      // Lettura dati altimetrici
      // ========================================================

      function readElevationPoints(xml) {
        const points = [];

        let totalPoints = 0;
        let elevationPoints = 0;

        for (
          const point of
          xml.querySelectorAll("trkpt")
        ) {
          totalPoints++;

          const lat =
            Number(
              point.getAttribute("lat")
            );

          const lon =
            Number(
              point.getAttribute("lon")
            );

          const eleNode =
            point.querySelector("ele");

          let elevation = null;

          if (eleNode) {
            const value =
              Number(
                eleNode.textContent
              );

            if (Number.isFinite(value)) {
              elevation = value;
              elevationPoints++;
            }
          }

          if (
            Number.isFinite(lat) &&
            Number.isFinite(lon)
          ) {
            points.push({
              coord: [lon, lat],
              elevation: elevation
            });
          }
        }

        return {
          points: points,

          totalPoints:
            totalPoints,

          elevationPoints:
            elevationPoints,

          coverage:
            totalPoints > 0
              ? elevationPoints / totalPoints
              : 0
        };
      }


      // ========================================================
      // Smoothing altimetrico spaziale
      //
      // Le finestre sono espresse in metri percorsi e non
      // in numero di campioni GPS.
      // ========================================================

      function smoothElevation(data, windowMeters) {
        const valid = [];

        for (const point of data.points) {
          if (
            Number.isFinite(
              point.elevation
            )
          ) {
            valid.push(point);
          }
        }

        if (valid.length < 2) {
          return [];
        }

        const result = [];

        let elevations = [
          valid[0].elevation
        ];

        let travelled = 0;
        let previous = valid[0];

        for (let i = 1; i < valid.length; i++) {
          const current = valid[i];

          travelled +=
            distanceMeters(
              previous.coord,
              current.coord
            );

          elevations.push(
            current.elevation
          );

          if (
            travelled >=
            windowMeters
          ) {
            result.push(
              elevations.reduce(
                (sum, value) =>
                  sum + value,
                0
              ) / elevations.length
            );

            elevations = [];
            travelled = 0;
          }

          previous = current;
        }

        if (elevations.length > 0) {
          result.push(
            elevations.reduce(
              (sum, value) =>
                sum + value,
              0
            ) / elevations.length
          );
        }

        return result;
      }


      // ========================================================
      // Analisi altimetrica
      // ========================================================

      function analyseElevation(data) {
        if (
          data.totalPoints === 0 ||
          data.coverage <
            MIN_ELEVATION_COVERAGE
        ) {
          return {
            valid: false,
            reason: "coverage"
          };
        }

        const profile =
          smoothElevation(
            data,
            ELEVATION_SMOOTH_METERS
          );

        if (profile.length < 2) {
          return {
            valid: false,
            reason: "profile"
          };
        }

        let min = profile[0];
        let max = profile[0];

        let ascent = 0;
        let descent = 0;
        let totalVariation = 0;

        for (
          let i = 1;
          i < profile.length;
          i++
        ) {
          const previous =
            profile[i - 1];

          const current =
            profile[i];

          min =
            Math.min(
              min,
              current
            );

          max =
            Math.max(
              max,
              current
            );

          const difference =
            current - previous;

          totalVariation +=
            Math.abs(difference);

          if (difference > 0) {
            ascent += difference;
          } else {
            descent +=
              -difference;
          }
        }

        const range =
          max - min;

        const oscillation =
          range > 0
            ? totalVariation / range
            : 0;

        const valid =
          oscillation <=
          MAX_ELEVATION_OSCILLATION;

        return {
          valid: valid,

          reason:
            valid
              ? null
              : "oscillation",

          min: min,
          max: max,

          ascent: ascent,
          descent: descent,

          oscillation:
            oscillation
        };
      }


      // ========================================================
      // Marker partenza / arrivo
      // ========================================================

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


      // ========================================================
      // Inizializzazione
      // ========================================================

      async function main() {

        // ------------------------------------------------------
        // Dipendenze
        // ------------------------------------------------------

        await loadJsByUrl(
          "https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"
        );

        await loadJsByUrl(
          "https://unpkg.com/@tmcw/togeojson@5.8.1/dist/togeojson.umd.js"
        );

        const css =
          document.createElement("link");

        css.rel =
          "stylesheet";

        css.href =
          "https://unpkg.com/leaflet@1.9.4/dist/leaflet.css";

        document.head.appendChild(css);


        // ------------------------------------------------------
        // Parsing GPX
        // ------------------------------------------------------

        const xml =
          new DOMParser()
            .parseFromString(
              gpxText,
              "application/xml"
            );

        const parserError =
          xml.getElementsByTagName(
            "parsererror"
          )[0];

        if (parserError) {
          throw new Error(
            "formato GPX non supportato"
          );
        }


        // ------------------------------------------------------
        // Conversione GeoJSON
        // ------------------------------------------------------

        const geojson =
          toGeoJSON.gpx(xml);

        const tracks =
          collectTracks(geojson);

        if (tracks.length === 0) {
          throw new Error(
            "nessuna traccia GPX trovata"
          );
        }


        // ------------------------------------------------------
        // Mappa
        // ------------------------------------------------------

        const map =
          L.map("gpx-map");

        L.tileLayer(
          "https://tile.openstreetmap.org/{z}/{x}/{y}.png",
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
                weight: 4
              }
            }
          ).addTo(map);

        const bounds =
          layer.getBounds();

        if (!bounds.isValid()) {
          throw new Error(
            "limiti della traccia non validi"
          );
        }

        map.fitBounds(
          bounds,
          {
            padding: [20, 20]
          }
        );


        // ------------------------------------------------------
        // Partenza / arrivo
        // ------------------------------------------------------

        const firstTrack =
          tracks[0];

        const lastTrack =
          tracks[
            tracks.length - 1
          ];

        const start =
          firstTrack[0];

        const finish =
          lastTrack[
            lastTrack.length - 1
          ];

        L.marker(
          [
            start[1],
            start[0]
          ],
          {
            icon:
              markerIcon("P")
          }
        )
          .addTo(map)
          .bindTooltip(
            "Partenza"
          );

        L.marker(
          [
            finish[1],
            finish[0]
          ],
          {
            icon:
              markerIcon("A")
          }
        )
          .addTo(map)
          .bindTooltip(
            "Arrivo"
          );


        // ------------------------------------------------------
        // Statistiche
        // ------------------------------------------------------

        const distance =
          calculateDistance(
            tracks
          );

        const duration =
          calculateDuration(
            xml
          );

        const elevationData =
          readElevationPoints(
            xml
          );

        const elevation =
          analyseElevation(
            elevationData
          );


        // ------------------------------------------------------
        // Riga principale
        // ------------------------------------------------------

        const info = [];

        info.push(
          (
            distance / 1000
          ).toFixed(1) +
          " km"
        );

        if (duration !== null) {
          info.push(
            formatDuration(
              duration
            )
          );
        }

        if (elevation.valid) {
          info.push(
            "↑ " +
            Math.round(
              elevation.ascent
            ) +
            " m"
          );

          info.push(
            "↓ " +
            Math.round(
              elevation.descent
            ) +
            " m"
          );
        }

        document
          .getElementById(
            "gpx-info"
          )
          .textContent =
            info.join(" · ");


        // ------------------------------------------------------
        // Informazioni altimetriche
        // ------------------------------------------------------

        const elevationInfo =
          document.getElementById(
            "gpx-elevation"
          );

        if (elevation.valid) {
          elevationInfo.textContent =
            "Quota " +
            Math.round(
              elevation.min
            ) +
            "–" +
            Math.round(
              elevation.max
            ) +
            " m";

        } else if (
          elevation.reason ===
          "coverage"
        ) {
          elevationInfo.textContent =
            "Quota parziale · dislivello n.d.";

        } else {
          elevationInfo.textContent =
            "Dislivello n.d.";
        }
      }


      // ========================================================
      // Gestione errori
      // ========================================================

      main().catch(
        function(error) {

          document
            .getElementById(
              "gpx-info"
            )
            .textContent =
              "Errore GPX: " +
              error.message;

          document
            .getElementById(
              "gpx-elevation"
            )
            .textContent = "";
        }
      );

    ]==], gpx),

    markdown =
      "GPX: " .. path
  }
end
```
