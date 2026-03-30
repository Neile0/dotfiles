# Kelvin Systems — JetBrains Theme

Two install options depending on how much you want to change.

## Option 1 — Editor colours only (no packaging required)

`Settings → Editor → Color Scheme → ⚙️ → Import Scheme → kelvin-systems.icls`

Covers syntax highlighting across all JetBrains IDEs (PyCharm, Rider, WebStorm, etc.)

## Option 2 — Full UI theme (status bar, tool windows, tabs)

Package the plugin first — requires a JDK:

```bash
jar cf kelvin-systems-theme.jar META-INF/ kelvin-systems.theme.json kelvin-systems.icls
```

Then: `Settings → Plugins → ⚙️ → Install Plugin from Disk → select the .jar`

Activate: `Settings → Appearance → Theme → Kelvin Systems`
