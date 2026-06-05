---
name: "Library/MG/Mio_Diario_3"
description: Implementa la Virtual Tag Page per il caso specifico dei Luoghi del Diario.
tags: meta/library
pageDecoration.prefix: "📃 "
share.uri: "github:marco10x15/silverbullet-libraries/Mio_Diario_3.md"
---
# virtual Page Tag Luoghi

## Funzioni utilizzate

## Templates utilizzati
* templates.OggettoTag (tagItem)
* templates.OggettoPage (pageItem)

```space-lua
-- Renders a page object as a linked list item with Title
templates.OggettoPage = template.new([==[
* [[${name}]] ${page.title(name)}
]==])

-- Renders a page object as a linked list item with full path
templates.fullPageItem = template.new([==[
* [[${name}|${name}]] ${page.title(name)}
]==])

-- Renders an item object
templates.itemItem = template.new([==[
* [[${string.find(ref, "[@#]") and ref or "$" .. ref}]] ${name} ${page.title(name)}
]==])

-- Renders a paragraph object
templates.paragraphItem = template.new([==[
* [[${string.find(ref, "[@#]") and ref or "$" .. ref}]] ${text}
]==])

-- Renders a tag object 
templates.OggettoTag = template.new([==[
[[tag:${name}|#${page.nome(name)}]] ]==])
```

# Overriding
Sovrascrive la pagina tag di default con questa implementazione personalizzata.

# Implementation
```space-lua
-- priority: 9
virtualPage.define {
  pattern = "tag:(.+)",
  run = function(tagName)
    local parts = string.split(tagName, "/")
    local tagView = parts[#parts]

    local text = "# 📌 " .. tagView .. "\n"
    -- if tagView ~= tagName then
    --  text = text .. tagName .. "\n"
    -- end
    local allObjects = query[[
      from index.objects(tagName)
      order by ref desc
    ]]
    local tagParts = tagName:split("/")
    local parentTags = {}
    for i in ipairs(tagParts) do
      local slice = table.pack(table.unpack(tagParts, 1, i))
      if i != #tagParts then
        table.insert(parentTags, {name=table.concat(slice, "/")})
      end
    end
    if #parentTags > 0 then
      text = text .. "## Gerarchia del tag\n"
        .. table.concat(query[[from t = parentTags select templates.OggettoTag(t)]]) .. "\n"
    end
    local subTags = query[[
      from index.tags()
      where string.startsWith(_.name, tagName .. "/")
      select {name=_.name}
    ]]
    if #subTags > 0 then
      text = text .. "## Tags figli\n"
        .. table.concat(query[[from t = subTags select templates.OggettoTag(t)]])
    end
    local taggedPages = query[[
      from o = allObjects where table.includes(o.itags, "page") select templates.OggettoPage(o)
    ]]
    if #taggedPages > 0 then
      text = text .. "## Pagine\n" .. table.concat(taggedPages)
    end
    local taggedParagraphs = query[[
      from o = allObjects where table.includes(o.itags, "paragraph") select templates.paragraphItem(o)
    ]]
    if #taggedParagraphs > 0 then
      text = text .. "## Paragrafi\n" .. table.concat(taggedParagraphs)
    end
    return text
  end
}
```
