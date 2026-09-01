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
-- GPX MAP - TEST
-- Rendering di una traccia GPX con Leaflet + OpenStreetMap
-- ============================================================

function gpxMap(path)
  if not path or path == "" then
    return widget.markdownBlock("**GPX:** percorso file non specificato.")
  end

  if not space.fileExists(path) then
    return widget.markdownBlock(
      "**GPX:** file non trovato: `" .. path .. "`"
    )
  end

  local gpx = space.readFile(path)

  if not gpx or gpx == "" then
    return widget.markdownBlock(
      "**GPX:** file vuoto: `" .. path .. "`"
    )
  end

  local script = string.format([==[
const gpxText = %q;

async function initMap() {
  await loadJsByUrl(
    "https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"
  );

  await loadJsByUrl(
    "https://unpkg.com/@tmcw/togeojson@5.8.1/dist/togeojson.umd.js"
  );

  const leafletCss = document.createElement("link");
  leafletCss.rel = "stylesheet";
  leafletCss.href =
    "https://unpkg.com/leaflet@1.9.4/dist/leaflet.css";

  document.head.appendChild(leafletCss);

  const xml = new DOMParser().parseFromString(
    gpxText,
    "application/xml"
  );

  const parserError = xml.querySelector("parsererror");

  if (parserError) {
    document.getElementById("gpx-status").textContent =
      "Errore nella lettura del file GPX.";
    return;
  }

  const geojson = toGeoJSON.gpx(xml);

  const map = L.map("gpx-map");

  L.tileLayer(
    "https://tile.openstreetmap.org/{z}/{x}/{y}.png",
    {
      maxZoom: 19,
      attribution: "&copy; OpenStreetMap contributors"
    }
  ).addTo(map);

  const track = L.geoJSON(geojson, {
    style: {
      weight: 4
    }
  }).addTo(map);

  const bounds = track.getBounds();

  if (bounds.isValid()) {
    map.fitBounds(bounds, {
      padding: [20, 20]
    });

    document.getElementById("gpx-status").remove();
  } else {
    document.getElementById("gpx-status").textContent =
      "Il file GPX non contiene una traccia visualizzabile.";
  }
}

initMap().catch(function(error) {
  document.getElementById("gpx-status").textContent =
    "Errore GPX: " + error.message;
});
]==], gpx)

  return widget.sandbox {
    html = [==[
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
          id="gpx-status"
          style="
            padding:8px 0;
            font-family:sans-serif;
            font-size:0.9em;
          ">
          Caricamento percorso GPX...
        </div>
      </div>
    ]==],

    script = script,

    markdown = "GPX: " .. path
  }
end
```
