---
name: "Library/MG/Mio_Diario_Testing"
tags: meta/library
description: "Widget, funzioni, in fase di sviluppo. ATTENZIONE! PERICOLO!"
version: "0.0-00"
versionDate: 2026-09-01
pageDecoration.prefix: "📔 "
share.uri: "github:marco10x15/silverbullet-libraries/Mio_Diario_Testing.md"
---

# ⚠️️⚠️ Funzioni Diario in Sviluppo ⚠⚠

**Versione:** 0.0-00 — 01.09.2026

La libreria è autonoma.

## Main Features

- `collage(images)` — visualizza un elenco di immagini dello Space come collage responsive.
- Le immagini possono essere indicate direttamente tramite path, ad esempio `media/foto.jpg`.
- Il numero di immagini per riga viene determinato automaticamente dalla larghezza disponibile.
- Le immagini mantengono le proporzioni e vengono ritagliate tramite `object-fit: cover`.
- Nessuna modifica persistente delle pagine o dei file immagine.

## Configurazione

Configurazione minima:

```space-lua
${collage {
  "media/foto1.jpg",
  "media/foto2.jpg",
  "media/foto3.jpg"
}}
```

Configurazione completa con i valori di default:

```space-lua
${collage {
  "media/foto1.jpg",
  "media/foto2.jpg",
  "media/foto3.jpg",
  "media/foto4.jpg",
  "media/foto5.jpg"
}}
```

Il layout utilizza i valori definiti nello Space Style:

- larghezza base elemento: `220px`
- altezza immagine: `180px`
- spazio tra immagini: `4px`
- adattamento immagine: `object-fit: cover`

## Uso

Esempio in una pagina Diario:

```space-lua
${collage {
  "media/2026-08-15-01.jpg",
  "media/2026-08-15-02.jpg",
  "media/2026-08-15-03.jpg",
  "media/2026-08-15-04.jpg"
}}
```

Il numero di colonne non viene definito in Lua.

Flexbox utilizza automaticamente la larghezza disponibile e dispone le immagini su una o più righe.

Virtual Page:

Non utilizzata.

Ricerca immagini:

Non utilizzata.

La funzione riceve esplicitamente l'elenco dei file da visualizzare. Non vengono effettuate scansioni o query dello Space.

## Implementazione

```space-lua
function collage(images)
  local items = {}

  for _, path in ipairs(images or {}) do
    table.insert(items,
      dom.div {
        class = "photo-collage-item",
        "![](<" .. path .. ">)"
      }
    )
  end

  return widget.htmlBlock(
    dom.div {
      class = "photo-collage",
      table.unpack(items)
    }
  )
end
```

```space-style
.photo-collage {
  display: flex;
  flex-wrap: wrap;
  gap: 4px;
  width: 100%;
}

.photo-collage-item {
  flex: 1 1 220px;
  height: 180px;
  overflow: hidden;
}

.photo-collage-item p {
  margin: 0;
  width: 100%;
  height: 100%;
}

.photo-collage-item img {
  display: block;
  width: 100%;
  height: 100%;
  object-fit: cover;
}
```