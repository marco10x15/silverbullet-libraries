---
name: "Library/MG/Mio_Diario_1"
tags: meta/library
pageDecoration.prefix: "📃 "
share.uri: "github:marco10x15/silverbullet-libraries/Mio_Diario_1.md"
---
# Top Widget - wTopDiario()
**TODO**
* link a galleria foto

## Funzioni di navigazione pagine, gestione dei metadati.
Set di widget, space-lua, query usate per il mio diario.
Descrizione
Inserisce per ogni pagina Diario una intestazione con:
- navigazione pagina precedente e successiva
- link a al Viaggio, se la pagina appartiene a un Viaggio
- elenco ei Tag della pagina del Diario

## Esempio
`${wTopDiario("Diario/2026-04-20")}`
${wTopDiario("Diario/2026-04-25")}

## Configurazione
Per attivare il Widget occorre:
```space-lua
config.set("std.widgets.wTopDiario.enabled", true)
```

## Implementazione
```space-lua
-- priority: -1
if config.get("std.widgets.wTopDiario.enabled", true) then
  event.listen {
    name = "hooks:renderTopWidgets",
    run = function(e)
        return wTopDiario()
    end
  }
end
```

### Funzioni utilizzate
**Built In**
* space.readPage(pageName)
* index.extractFrontmatter(content)

**Da altre librerie**
* page.prec()
* page.succ()
* page.title(pageName)
* page.nome(pageName)

### Funzione

${page.prec("Diario/2026-04-20")}
${page.succ("Diario/2026-04-20")}
${page.title("Diario/2026-04-20")}
${page.nome("Diario/2026-04-20")}

```space-lua
-- priority: 10
function wTopDiario(path)
  local pageName = path or editor.getCurrentPage()
  if string.startsWith(pageName, "index") then
      return 
  end
  if not string.startsWith(pageName, "Diario") then
      return 
  end

  -- Titolo
  local titolo = page.title(pageName)
  if not titolo or #titolo == 0 then
    titolo = page.nome(pageName)
  end

  -- Navigazione
  local prev = page.prec()
  local next = page.succ()

  local result = string.format("# [[%s|⬅️]]%s[[%s|➡️]]\n", prev, titolo, next)
  
  -- Frontmatter (safe)
  local content = space.readPage(pageName)
  local fm = index.extractFrontmatter(content) or {}
  local fmd = fm.frontmatter or {}

  -- Description
  if fmd.description then
    result = result .. "📒 " .. fmd.description .. "\n"
  end

  -- Viaggio
  if fmd.Viaggio then
    result = result .. "🧭 " .. fmd.Viaggio .. "\n"
  end
  --
  -- Tags (safe)
-- -Con l'introduzione in SilverBullet 2.10 del rendering dei tag
-- nel frontmatter collassato, non è più necessario il rendering dei Tags nel Widgets
--  Disattivato il 07.08.2026
--  if type(fmd.tags) == "table" and #fmd.tags > 0 then
--    local tagsConHash = {}
--    for _, tag in ipairs(fmd.tags) do
--      table.insert(tagsConHash, "#" .. tag)
--    end

--    local elencoTag = "ℹ " .. table.concat(tagsConHash, ", ") .. "\n"
--    result = result .. elencoTag
--  end

  -- Galleria Foto
  if type(fmd.galleriafoto) == "table" and #fmd.galleriafoto > 0 then
    local elencoFoto = "📷 " .. table.concat(galleriafoto, ", ") .. "\n"
    result = result .. elencoFoto
  end

  return result
end
```
