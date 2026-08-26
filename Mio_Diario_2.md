> **NOTA: Questo documento è obsoleto. Si prega di fare riferimento a
> - Mio_Diario.md
> - Mio_Diario_Indice.md
> per le informazioni aggiornate.**



---
name: "Library/MG/Mio_Diario_2"
tags: meta/library
pageDecoration.prefix: "📃 "
share.uri: "github:marco10x15/silverbullet-libraries/Mio_Diario_2.md"
---
# Mio Diario - Linked Info Luoghi
Questo widget presenta l'elenco dei Viaggi e dei Giorni in cui siamo stati nel luogo di cui la pagina "luoghi" è aperta

Informazioni per il luogo **[[luoghi/EU/ITA/34/VE/Venezia]]**
${widgets.linkedInfoLuoghi("luoghi/EU/ITA/34/VE/Venezia")}

## Attivazione
```space-lua
config.set("std.widgets.linkedInfoLuoghi.enabled", true)
```

## Implementazione
```space-lua
-- priority: -1
if config.get("std.widgets.linkedInfoLuoghi.enabled") then
  event.listen {
    name = "hooks:renderBottomWidgets",
    run = function(e)
      if string.startsWith(editor.getCurrentPage(), "luoghi/") then
        return widgets.linkedInfoLuoghi()
      end
    end
  }
end
```

## Implementazione 

### linkedInfoLuoghi(pageName)


```space-lua
function widgets.linkedInfoLuoghi(pageName)
  local pageName = pageName or editor.getCurrentPage()
--  local tagName = 
  -- local text = "**Siamo stati qui**\n"
  local text = "## Siamo stati qui\n"
  text = text .. tagBreadcrumb()

  local diarioInfo = query[[
    from l = index.tag "link"
    where l.page != pageName 
    and l.toPage == pageName
    and string.startsWith(l.page, "Diario/")
    select {
      page = l.page,
      ref = l.ref,
      giorno = date.format(l.page),
      title = page.title(l.page),
      snippet = l.snippet
    }
    order by l.page desc
  ]]

  -- Estrazione viaggi
  local viaggi = {}
  local pageDiario = {}
  local seen = {}

  for i = 1, #diarioInfo do
    local res = query[[
      from p = index.tag "page"
      where p.name == diarioInfo[i].page
      select {Viaggio = p.Viaggio,
          anno = date.format(diarioInfo[i].page, "YY")}
      limit 1
    ]]
    -- table.insert(viaggi, res[1].Viaggio)
    if res and #res > 0 then
      local v = res[1].Viaggio
      local a = res[1].anno
      if v and not seen[v] then
        local t = string.format("%s [[Viaggi/%s|%s]]", a, v, v)
        table.insert(viaggi, t)
        seen[v] = true
      end
    end
    -- table.insert(pageDiario, res[1].Viaggio)
    local page = diarioInfo[i].page
    local ref = diarioInfo[i].ref
    local giorno = diarioInfo[i].giorno
    local title = diarioInfo[i].title
    -- local snippet = diarioInfo[i].snippet
    local t = string.format("**[[%s|%s]]** %s", page, giorno, title)
    table.insert(pageDiario, t)
  end
  --
  -- Elenca i viaggi.
  if #viaggi > 0 then
    text = text
      .. "\n🧭 **In questi viaggi:**\n"
      .. table.concat(viaggi, "\n") .. "\n\n"
  end
  --
  -- Elenca i giorni.
  text = text .. "**In questi giorni:**\n"
              .. table.concat(pageDiario, "\n") .. "\n"
  --
  -- Restituisce il risultato.
  return text
end
```

## Renders a tag object in Breadcrump
```space-lua
-- Renders a tag object in Breadcrump
templates.BreadcrumbTag = template.new([==[[[tag:${name}|#${page.nome(name)}]] ]==])

-- priority: 9
function tagBreadcrumb()
  local text = ""
  local parentTags = {}

  -- Recupera il tag più lungo (più specifico)
  local res = query[[
    from p = index.tags()
    where p.page == editor.getCurrentPage() 
    order by #p.name desc
    limit 1
  ]]

  if not res or #res == 0 or not res[1].name then
    return ""
  end

  local tagName = res[1].name

  -- split sicuro (SB usa string.split)
  local tagParts = string.split(tagName, "/")

  for i = 1, #tagParts do
    local slice = {}
    for j = 1, i do
      table.insert(slice, tagParts[j])
    end
    table.insert(parentTags, { name = table.concat(slice, "/") })
  end
  
  if #parentTags > 0 then

    local rendered = query[[
      from t = parentTags
      select templates.BreadcrumbTag(t)
    ]]

    if rendered and #rendered > 0 then
      text = text .. table.concat(rendered, "") .. "\n"
    end
  else
      text = text .. tagName .. "\n"
  end

  return text
end
```
