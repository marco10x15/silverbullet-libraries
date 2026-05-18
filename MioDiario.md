---
name: "Library/MG/MioDiario"
tags: meta/library
pageDecoration.prefix: "📃 "
share.uri: "github:marco10x15/silverbullet-libraries/MioDiario.md"
---
# La struttura utilizzata per il mio diario

Documetazione, funzioni di navigazione pagine, gestione dei metadati.

# Virtual Page

## Top Widget - wTopDiario()
### Descrizione
Inserisce per ogni pagina Diario una intestazione con:
- navigazione pagina precedente e successiva
- link a al Viaggio, se la pagina apaprtiene a un Viaggio
- elenco ei Tag della pagina del Diario


### Esempio
`${wTopDiario("Diario/2026-05-08")}`

### Configurazione
Per attivare il Widget occorre:
```lua
config.set("std.widgets.wTopDiario.enabled", true)
```

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

* space.readPage(pageName)
* index.extractFrontmatter(content)

### Funzione

```space-lua
-- priority: 10
function wTopDiario(path)
  local pageName = path or editor.getCurrentPage()
  local function preText(pre, text, post)
    local pre = pre or ""
    local post = post or ""
    if not text or #text == 0 then
      return ""
    else
      return string.format("%s%s%s", pre, text, post)
    end
  end
  
  if string.startsWith(pageName, "Diario/") then
    local content = space.readPage(pageName)
    local fm = index.extractFrontmatter(content)
    
    local result = string.format("⬅️[[%s]]",page.prec())
                    .. " - " 
                    .. string.format("[[%s]]➡️",page.succ())
                    .. "\n"
    if fm.frontmatter.description then
      result = result .. "📒 " .. fm.frontmatter.description .. "\n"
    end
    if fm.frontmatter.Viaggio then
      result = result .. "🧭 [[🧭 " .. fm.frontmatter.Viaggio .. "|".. fm.frontmatter.Viaggio .. "]]\n"
    end
    if fm.frontmatter.luogo then
      result = result .. "🗺️ " .. table.concat(fm.frontmatter.luogo, ", ") .. "\n"
    end
    if fm.frontmatter.tags then
      result = result .. "ℹ " .. table.concat(fm.frontmatter.tags, ", ") .. "\n"
    end

    return result
  end
end
```
