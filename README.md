# Plugin_InventoryViewV5

The **Inventory View** plugin for [Genie 5](https://github.com/GenieClient/Genie5) —
a port of Etherian's `InventoryView` (v1.8) for the Genie 5 platform.

It records what each of your characters owns — on their person, in a **vault**
(if you have a vault book), a **deed register**, a **home**, and **Trader
storage** — into a per-character tree saved to `InventoryView.xml`, and lets you
search that catalog across all of your characters. Results render into the
plugin's own **Inventory View** dock window:

```
Inventory catalog — 2 character(s)
──────────────────────────────────────
Renucci
  Inventory
    a darkened leather backpack
      a tarnished silver ring
      a small healing potion
    a woolen cloak
  Vault
    a reinforced iron strongbox
      a stack of documents
```

> **Status: early release, not yet exercised against a live session.** The scan
> state machine is a faithful 1:1 port of the Genie 4 plugin, but the Genie 5
> XML parser delivers list text slightly differently than Genie 4 did. The scan
> relies on DragonRealms sending the inventory/vault/home lists as main-window
> text with their leading-space indentation intact. If a scanned tree ever looks
> flat, run `/iv debug` then `/iv scan` and watch the `[IV dbg] sp= lvl=` trace.

## Requirements

- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0) (to build).
- A Genie 5 install with the plugin system (host version ≥ 5.0).

## Build

```sh
dotnet build -c Release
```

Output: `bin/Release/net8.0/Plugin_InventoryViewV5.dll`.

The project references the Genie 5 plugin contract (`Genie.Plugins.Abstractions`)
from a committed copy in `lib/`, so no NuGet feed is required to compile.

## Install

Copy `Plugin_InventoryViewV5.dll` into your Genie 5 plugins folder:

- **Windows:** `%APPDATA%\Genie5\Plugins\`
- **macOS:** `~/Library/Application Support/Genie5/Plugins/`
- **Linux:** `~/.local/share/Genie5/Plugins/` (or `$XDG_DATA_HOME/Genie5/Plugins/`)

Then in Genie 5 either:

- **Reconnect** — plugins load on connect, or
- **Plugins → Load → Plugin_InventoryViewV5.dll** (menu), or
- `#plugin load Plugin_InventoryViewV5` (command bar).

## Commands

| Command | What it does |
|---|---|
| `/iv scan` | Scan the current character (inventory → vault → deed register → home → Trader storage) and save. The **Inventory View** window pops up with the result when the scan finishes. |
| `/iv open` (or `/iv list`) | Show the full catalog of every scanned character in the **Inventory View** dock window. |
| `/iv search <text>` | Show every catalogued item matching `<text>` (each by its full path) in the **Inventory View** window. |
| `/iv reload` | Reload `InventoryView.xml` from disk (run after scanning on another running instance). |
| `/iv wiki <item>` | Open Elanthipedia/drservice for `<item>` in your browser (via `#browser`). |
| `/iv export [path]` | Export the whole catalog to a CSV file (defaults to `{AppData}/Genie5/InventoryView_export.csv`). |
| `/iv debug` | Toggle a per-line scan trace (handy for diagnosing indentation). |

`/inventoryview` is accepted as the long form of `/iv`.

At the end of a scan the plugin prints **Scan Complete.** and re-emits
`InventoryView scan complete` through the parse pipeline, so a login/automation
script can wait on it:

```
send /iv scan
waitforre ^InventoryView scan complete
```

## Manage

From the **Plugins** menu: **Enable / Disable**, **Unload**, **Load**. Or from
the command bar:

```
#plugin list
#plugin disable Inventory View
#plugin enable  Inventory View
#plugin unload  Inventory View
#plugin load    Plugin_InventoryViewV5
#plugin reload
```

Show/hide the window via **Window → Plugin Windows → Inventory View**.

## How it works

`/iv scan` walks the same sequence the Genie 4 plugin did — inventory list →
vault book → deed register → home → Trader storage book — parsing each list by
its leading-space indentation into a nested item tree, and saving on completion.
The trigger strings and the commands it sends to the game are unchanged from the
Genie 4 original, so it behaves identically against DragonRealms.

The plugin has no UI dependency — it writes formatted text to a *named window*
via the host API (`SetWindow("Inventory View", …)`), and the host surfaces that
window as a dock panel.

### What changed from the Genie 4 plugin

- **Its own dock window instead of a WinForms form.** Genie 5 plugins are UI-free
  and reference only `Genie.Plugins.Abstractions`, but they create their own
  panel through the host's named-window seam. The old `TreeView` form
  (search/expand/collapse/wiki/export/reload buttons) becomes the `/iv` commands
  plus the monospace text tree.
- **Non-blocking roundtime wait.** The Genie 4 code blocked the parse thread with
  `Thread.Sleep(rt * 1000)` after the inventory list; this port schedules the
  follow-up with a non-blocking `Task.Delay`, so a scan never stalls the game loop.
- **End-of-home on the prompt.** Genie 4 watched for a bare `>` text line to end
  the home scan; Genie 5 surfaces that as `OnPrompt()`.
- **Cross-platform.** Wiki lookups use the host's `#browser` command instead of
  `Process.Start`, and `InventoryView.xml` resolves under `{AppData}/Genie5`
  (honoring macOS/Linux paths), not the Windows-only Genie config dir.
- **Persistence cleanup.** `ItemData.parent` is `[XmlIgnore]` instead of being
  nulled-and-rebuilt around every save. The on-disk XML is unchanged, so an
  existing `InventoryView.xml` from the Genie 4 plugin still loads.

## License

[GPL-3.0](LICENSE) — same as Genie 5 and the Genie 4 ecosystem.

## Credits

Behaviour ported from Etherian's
[Plugin_InventoryView](https://github.com/GenieClient/Plugin_InventoryView) for
Genie 4 (itself forked from EtherianDR/InventoryView).
