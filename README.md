# Windows rebuild from one file (winget configure)

Two `winget configure` files that describe my Windows 11 setup, so a clean install can be rebuilt with one command instead of clicking through installers.

This is my own list, not a recommended baseline. Read the app lists before you run anything.

## What's in here

| File | Contents |
|---|---|
| `workstation.dsc.yaml` | Core: 3 Windows settings + 76 apps (dev tools, homelab access, everyday apps, browsers and chat, a few Explorer/UI tweaks) |
| `workstation-extras.dsc.yaml` | Extras: 25 apps (hardware monitoring, peripherals, game launchers, flight sim tooling). Skip it if you don't have the hardware |

The Windows settings in the core file:

- Developer Mode on
- File extensions and hidden files shown in Explorer
- Dark mode for apps and the system

Apps are installed from the `winget` source by package ID. Versions are not pinned, so you get whatever is current.

## Quick start

You need Windows 11 with winget 1.6 or newer (I tested on 1.11.510).

1. Enable configuration once. It's off by default on a fresh install:

```powershell
winget configure --enable
```

2. Check the file:

```powershell
winget configure validate -f workstation.dsc.yaml
```

3. Open Terminal **as administrator** and run it:

```powershell
winget configure -f workstation.dsc.yaml --accept-configuration-agreements
```

4. Optionally, run the extras file the same way:

```powershell
winget configure -f workstation-extras.dsc.yaml --accept-configuration-agreements
```

Notes:

- `--accept-configuration-agreements` accepts the third-party license terms on your behalf. Check what you're installing first.
- Some installers show UAC prompts, and the run waits on them. It takes a while.
- The core file includes WSL and Docker Desktop. They need virtualization and a reboot, and Docker Desktop may fail inside a VM without nested virtualization.
- `ExplorerPatcher`, `Windhawk` and `Seelen UI` change how the Windows shell looks and behaves. Remove them from the file if you don't want that.

## What I tested

I ran a cut-down file (3 settings + 14 apps) on a clean Windows 11 VM in VMware: Git, GitHub CLI, VS Code, Node 22, Python 3.13, Windows Terminal, 7-Zip, Notepad++, VLC, Everything, PowerToys, Tailscale, Chrome and Discord.

- All 14 apps installed. The run took 14 min 26.8 s, including time spent waiting on UAC prompts I hadn't clicked.
- File extensions showed and dark mode applied.
- Developer Mode did **not** apply on that run. Re-running it from a non-admin terminal reported: "Toggling Developer Mode requires this resource to be run as an Administrator." I added `securityContext: elevated` to that unit and recommend an elevated terminal. I haven't confirmed that this fix works yet.

I haven't run the full core or extras files end to end. The package IDs outside the 14 above came from my own `winget export` or the winget catalog, and I haven't installed them from these files. To check every ID against the winget source before you run (a missing one shows as `NOT FOUND`):

```powershell
Select-String -Path .\workstation.dsc.yaml -Pattern '^ {8}id: (\S+)' | ForEach-Object {
    $id = $_.Matches[0].Groups[1].Value
    winget show --id $id --exact --source winget --accept-source-agreements *> $null
    '{0,-48} {1}' -f $id, $(if ($LASTEXITCODE -eq 0) { 'ok' } else { 'NOT FOUND' })
}
```

Change the file name to check the extras file.

## What it doesn't restore

- Logins, profiles, history and saved passwords. Apps come back signed out.
- App settings, and VS Code extensions and settings.
- Exact versions. A rebuild gets newer builds than an old install may have.
- Drivers, vendor suites, and licensed software such as Office or Adobe. I left them out on purpose.
- SSH keys and git credentials.
- Taskbar and Start layout, default apps and display scaling. I haven't tested whether any of these carry over.

## Building your own

Start from your own export:

```powershell
winget export -o apps.json --include-versions
```

The export is JSON, and `winget configure` reads YAML, so each package needs to become a unit like this:

```yaml
- resource: Microsoft.WinGet.DSC/WinGetPackage
  id: Git_Git
  directives:
    description: Git
    allowPrerelease: true
  settings:
    id: Git.Git
    source: winget
```

Expect to edit the result by hand:

- Apps installed outside winget (installed manually, from the Microsoft Store, or vendor tools) show up as "not available from any source" warnings. Add the ones you want by their winget ID.
- Apps installed for both the system and your user can appear twice. Keep one.
- Runtimes, drivers and vendor suites usually don't belong in the file, because apps pull in what they need.
