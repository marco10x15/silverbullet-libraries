---
name: "Library/MG/Mie_Configurazioni"
tags: meta/library
pageDecoration.prefix: "⚙️ "
---
---
displayName: Mie_Configurazioni
description: Configurazione personalizzata del mio space SilverBullet space.
tags: meta
---

Run ${widgets.commandButton "System: Reload"} to reload.

## User configuration
Configurazioni aggiuntive a quelle di [[CONFIG]]

```space-lua
actionButton.define {
  icon = "map",
  description = "Viaggio",
  command = "Viaggio: Menu",
  priority = 2.5,
  dropdown = false,
}
```

## Appunti da Readeck
```space-lua
config.set("readeck", {
  apiUrl = "http://172.30.250.14:8000",
  webUrl = "https://readeck.fm-nas.net",
  tokenPage = "Library/MG/token",
})
```

## Rimuove la scritta da frontmatter collassato
```space-style
.cm-frontmatterFoldStatus {
  display: none !important;
}
```

## Larghezza schermo
```space-style
html {
  --editor-width: min(1024px, 94vw);
}
```

## Managed by the Configuration Manager
The block below is maintained by the ${widgets.commandButton("Configuration Manager", "Configuration: Open")}. Prefer editing it through the UI, although simple hand edits should survive.

```space-lua
-- managed-by: configuration-manager
config.set("frontmatterFolding.foldByDefault", "always")
config.set("frontmatterFolding.foldByDefaultLines", 1)
config.set("std.widgets.toc.enabled", false)
config.set("std.widgets.linkedTasks.enabled", false)
config.set("index.task.all", false)
config.set("journal.enabled", false)
```
