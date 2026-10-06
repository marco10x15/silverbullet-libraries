---
name: "Library/MG/View Widgets"
tags: meta/library
description: "View generali."
version: "0.02"
versionDate: 2026-10-06
pageDecoration.prefix: "⚙️ "
share.uri: "github:marco10x15/silverbullet-libraries/View_Widgets.md"
---

# View Widgets


## page.viewSubpages(orderBy, direction, separator)

Restituisce una View con l'elenco delle sottopagine dirette della pagina corrente.

 Sintassi:

```
${page.viewSubpages()}
${page.viewSubpages("name")}
${page.viewSubpages("name", "desc")}
${page.viewSubpages("displayName", "asc")}
${page.viewSubpages("date", "desc")}
${page.viewSubpages("date", "desc", " ")}
```

Parametri:

`orderBy`         Campo utilizzato per l'ordinamento:
* "name"          nome/percorso della pagina (default)
* "displayName"   attributo displayName
* "date"          attributo date

`direction`       Direzione dell'ordinamento:
* "asc"           crescente (default)
* "desc"          decrescente

`separator`     Separazione tra una pagina e la successiva:
* omesso          nessuna separazione aggiuntiva (default)
* " "             una riga vuota

**Pagine restituite:**
Vengono mostrate esclusivamente le sottopagine dirette della pagina corrente. I livelli gerarchici successivi non vengono inclusi.

**Visualizzazione:**
Per ogni pagina vengono mostrati:

1. wikilink alla pagina, usando displayName quando disponibile e name come fallback;
2. description, se presente e non vuota;
3. tags, escludendo il tag "page".

Esempi:

`${page.viewSubpages()}`

name crescente, nessuna separazione aggiuntiva.

`${page.viewSubpages("date", "desc")}`

date decrescente, nessuna separazione aggiuntiva.

`${page.viewSubpages("date", "desc", " ")}`

date decrescente, una riga vuota tra le pagine.

## Implementazione
```space-lua
page = page or {}

function page.viewSubpages(orderBy, direction, separator)
  orderBy = orderBy or "name"
  direction = direction or "asc"

  if orderBy ~= "name"
    and orderBy ~= "displayName"
    and orderBy ~= "date"
  then
    error(
      'page.viewSubpages: orderBy deve essere "name", "displayName" oppure "date"'
    )
  end

  if direction ~= "asc" and direction ~= "desc" then
    error(
      'page.viewSubpages: direction deve essere "asc" oppure "desc"'
    )
  end

  if separator ~= nil and separator ~= " " then
    error(
      'page.viewSubpages: separator deve essere omesso oppure " "'
    )
  end

  return widget.new {
    content = function()
      local currentPage = editor.getCurrentPage()
      local prefix = currentPage .. "/"
      local pages

      if orderBy == "displayName" then
        if direction == "desc" then
          pages = query[[
            from p = index.subPages(currentPage)
            where string.sub(p.name, 1, #prefix) == prefix
              and not string.find(string.sub(p.name, #prefix + 1), "/")
            order by p.displayName desc
          ]]
        else
          pages = query[[
            from p = index.subPages(currentPage)
            where string.sub(p.name, 1, #prefix) == prefix
              and not string.find(string.sub(p.name, #prefix + 1), "/")
            order by p.displayName
          ]]
        end

      elseif orderBy == "date" then
        if direction == "desc" then
          pages = query[[
            from p = index.subPages(currentPage)
            where string.sub(p.name, 1, #prefix) == prefix
              and not string.find(string.sub(p.name, #prefix + 1), "/")
            order by p.date desc
          ]]
        else
          pages = query[[
            from p = index.subPages(currentPage)
            where string.sub(p.name, 1, #prefix) == prefix
              and not string.find(string.sub(p.name, #prefix + 1), "/")
            order by p.date
          ]]
        end

      else
        if direction == "desc" then
          pages = query[[
            from p = index.subPages(currentPage)
            where string.sub(p.name, 1, #prefix) == prefix
              and not string.find(string.sub(p.name, #prefix + 1), "/")
            order by p.name desc
          ]]
        else
          pages = query[[
            from p = index.subPages(currentPage)
            where string.sub(p.name, 1, #prefix) == prefix
              and not string.find(string.sub(p.name, #prefix + 1), "/")
            order by p.name
          ]]
        end
      end

      local result = {}
      local pageSeparator = "\n"

      if separator == " " then
        pageSeparator = "\n\n"
      end

      for _, p in ipairs(pages) do
        local item = {}
        local tags = {}

        for _, tag in ipairs(p.tags or {}) do
          if tag ~= "page" then
            table.insert(tags, "#" .. tag)
          end
        end

        table.insert(
          item,
          "[[" .. p.name .. "|" .. (p.displayName or p.name) .. "]]"
        )

        if p.description and p.description ~= "" then
          table.insert(item, p.description)
        end

        if #tags > 0 then
          table.insert(item, table.concat(tags, " "))
        end

        table.insert(result, table.concat(item, "\n"))
      end

      return table.concat(result, pageSeparator)
    end,
  }
end
```
