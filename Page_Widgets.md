---
name: "Library/MG/Page Widgets"
tags: meta/library
pageDecoration.prefix: "⚙️ "
share.uri: "github:marco10x15/silverbullet-libraries/Page_Widgets.md"
---
# Page Widgets

## page.nome(where)
**Funzione che riceve un path e restituisca il valore dell'ultima parte.**

Questa funzione estrae il nome dalla pagina.

Nome di questa pagina: ${editor.getCurrentPage()}
Stringa estratta dal nome: ${page.nome(editor.getCurrentPage())}.

La funzione riconosce il caso particolare in cui la stringa contenga una data in forma ISO e restituisce solo la data ISO.

### Esempio
-- Esempi di utilizzo:
${page.nome("Inbox/New Page/2025-04-01")} vs ${page.nome("Inbox/New Page/2025-04-01")} 
${page.nome("Inbox/New Page/2025-02-00")} vs ${page.nome("Inbox/New Page/2025-02-00")}     
${page.nome("Notes/2023-12-25-Mattina")} vs ${page.nome("Notes/2023-12-25-Mattina")} 
${page.nome("Tasks/Important/2024-07-15-Meeting")} vs ${page.nome("Tasks/Important/2024-07-15-Meeting")} 
${page.nome("Random/String/No-Date-Here")}  vs ${page.nome("Random/String/No-Date-Here")} 
${page.nome("OnlyOnePart")} vs ${page.nome("OnlyOnePart")}

### Implementazione

```space-lua
page = page or {}
function page.nome(path)
  -- Estrai la parte finale dopo l'ultimo "/"
  local lastPart = path:match(".*/(.*)$") or path

  -- Cerca una data in formato ISO (AAAA-MM-GG)
  local dateISO = lastPart:match("(%d%d%d%d%-%d%d%-%d%d)")
  if dateISO then return dateISO end

  -- Sostituisci gli underscore con spazi
  local cleaned = lastPart:gsub("_", " ")

  -- Decodifica i caratteri URL-encoded (es. %27 → ')
  cleaned = cleaned:gsub("%%(%x%x)", function(hex)
    return string.char(tonumber(hex, 16))
  end)

  return cleaned
end
```

## page.title(where)
**Restituisce il titolo della pagina dal primo Header H1 presente della pagina**

Se _where_ è omesso utilizza la pagina corrente.

### Esempio
per la pagina restituisce: 
[[Diario/2026-05-09]] => ${page.title("Diario/2026-05-09")}
[[Diario/2026-04-25]] => ${page.title("Diario/2026-04-25")}
[[Diario/1900-01-01]] => ${page.title("Diario/1900-01-01")}

### Implementazione

```space-lua
-- function page.title(where) 
-- Restituisce il titolo della pagina
-- dal primo Header H1 della pagina
page = page or {}

function page.title(where)
  if space.pageExists(where) then
  -- Estrai frontmatter
  local fm = index.extractFrontmatter(space.readPage(where))
  
  if fm.frontmatter.title then
    return fm.frontmatter.title
  else
    -- Se non esiste title in frontmatter
    -- cerca di estrarlo dal primo header H1
    local out = query[[
      from t = index.tag "header"
      where t.page == where
      and t.level == 1
      order by t.pos
      select {title = t.text}
      limit 1
    ]]
    if #out == 0 then
      return ""
    else
      return tostring(out[1].title)
    end
  end
  -- if space.pageExists(where) è falso
  -- la pagina non esite
  else 
    return "La pagina non esiste"
  end
end
```
