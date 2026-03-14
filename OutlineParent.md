---
name: "Library/MG-10x15/OutlineParent"
tags: meta/library
pageDecoration.prefix: "🛠️ "
---
# The name of the parent of a indented bullet list

This function will retrieve the name of the parent of one bullet item or task
Visual Example:

## Usage
`${OutlineParent(Parent)}`

## Implementation
```space-lua
function OutlineParent(parent)
    local TaskResult = query [[
        from t = index.tag "task"
        where t.ref == parent
        select { name = t.name }
    ]]

    local ItemResult = query [[
        from i = index.tag "item"
        where i.ref == parent
        select { name = i.name }
    ]]

    -- 1. Se trovo un task
    if TaskResult and #TaskResult > 0 then
        return string.format("◻[[%s|%s]]", parent, tostring(TaskResult[1].name or ""))
    end

    -- 2. Se non c'è un task, provo con item
    if ItemResult and #ItemResult > 0 then
        -- return tostring(ItemResult[1].name or "")
      return string.format("[[%s|%s]]", parent, tostring(ItemResult[1].name or ""))
    end

    -- 3. Nessun risultato
    return ""
end
```

## Discussions to this function
* No discussione at the moment.
