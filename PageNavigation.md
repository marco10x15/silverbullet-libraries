---
name: "Library/MG/PageNavigation"
tags: meta/library
pageDecoration.prefix: "📃 "
share.uri: "github:marco10x15/silverbullet-libraries/PageNavigation.md"

share.mode: pull
---
share.hash: eb690bee
# Page Navigation Functions

Funzioni di navigazione pagine e gestione metadati

## Example of format accepted and output provided
|Funzione|Descrizione
|---|---|
|widgets.breadcrumb()|Crea il [[Library/my/Widgets/breadcrumb]] fino alla pagina padre|

|Funzione|restituisce|Descrizione|
|---|---|---|
|page.parents(path)|table| _genitori_ di quella attuale|
|page.sister()|table|_sorelle_ di quella attuale|
|page.child(path)|table|_figli_ di quella attuale|
|~~page.meta(where)~~|table|_meta dati_ della pagina where|
|page.up()|string|_padre_ di quella attuale|
|page.prec()|string|_sorella precedente_|
|page.succ()|string|_sorella successiva_|
|page.nome()|string|_estrae il nome della pagina_ dall’ultima parte del path|
|~~page.tags()~~|table|_estrae i tag della pagina_ |
|page.lev() |number|Restituisce il livello della pagina|

## Implementation

### page.parents(path)
```space-lua
-- function page.parents(path) - Restituisce una table con l'elenco dei genitori
page = page or {}
function page.parents(path)
  local path = path or editor.getCurrentPage()
  local livelli = string.split(path, "/")
  local genitori = {}
  local pathParziale = ""

  for i = 1, #livelli - 1 do
    pathParziale = (i == 1) and livelli[i] or (pathParziale .. "/" .. livelli[i])
    table.insert(genitori, pathParziale)
  end
  return genitori
end
```

### page.sister()
```space-lua
-- function page.sister() - raccolta delle pagine "Sorelle", ovverosia allo stesso livello.
page = page or {}
function page.sister()
  local pa = query[[from index.tag "page"
  where string.startsWith(name, page.up() .. "/") 
  and not string.match(name, page.up() .. "/.+/")
  ]]
  return pa
end
```

### page.prec()
```space-lua
-- function page.prec()
page = page or {}
function page.prec()
  local filtro1 = page.up() .. "/"
  local filtro2 = page.up() .. "/.+/"
  local paa = editor.getCurrentPage()
  local pa = query[[from index.tag "page"
              where string.startsWith(name, filtro1) 
              and not string.match(name, filtro2)
              and name < paa
              order by name desc
              limit 1]]
  -- Se pa è nil o vuota, restituisce una stringa vuota
  if not pa or #pa == 0 then
    return ""
  end 

  return pa[1].name
end
```

### page.succ()
```space-lua
-- function page.succ() -- Pagian successiva della pagina attuale.
page = page or {}
function page.succ()
  local filtro1 = page.up() .. "/"
  local filtro2 = page.up() .. "/.+/"
  local paa = editor.getCurrentPage()
  local pa = query[[from index.tag "page"
              where string.startsWith(name, filtro1) 
              and not string.match(name, filtro2)
              and name > paa
              order by name 
              limit 1]]
    -- Se pa è nil o vuota, restituisce una stringa vuota
    if not pa or #pa == 0 then
      return ""
    end
    return pa[1].name
end
```

### page.up()
```space-lua
-- function page.up()
page = page or {}
function page.up(path)
  local path = path or editor.getCurrentPage()
  local parts = {}
  
  -- Suddivide il percorso in segmenti
  for part in string.split(path, "/") do
    table.insert(parts, part)
  end
  for i = 1, #parts - 1 do
    if i == 1 then
      currentPath = parts[i]
    else
      currentPath = currentPath .. "/" .. parts[i]
    end
    
  end
  return currentPath
end
```

### page.child(path)
```space-lua
-- function page.child()
page = page or {}
function page.child(path)
  local path = path or editor.getCurrentPage()
  local parts = {}
  local pa = query[[from index.tag "page"
    where string.startsWith(name, path .. "/") 
    and not string.match(name, path .. "/.+/")
    ]]
  
  if not pa or #pa == 0 then
    -- "😮 Nessuna pagina."
    return nil 
  end 

  return pa
end
```

## page.lev()
```space-lua
-- function page.lev()
page = page or {}
function page.lev(path)
  local path = path or editor.getCurrentPage()
  
  return #string.split(path, "/")
end
```


## More Examples & Discussions
