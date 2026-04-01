---
title: Mod Organizer 2 for Fallout 4 on Linux via Steam/Proton
tags: [mo2, fallout4, proton, steam, linux, modding]
verified: 2026-04-01
platform: linux
---

## Problem
Setting up Mod Organizer 2 for Fallout 4 modding on Ubuntu with Steam/Proton. Vortex doesn't run on Linux. Nexus Collections are Vortex-only.

## Solution

### Install MO2
Use the [modorganizer2-linux-installer](https://github.com/rockerbacon/modorganizer2-linux-installer):

```bash
tar xf mo2installer-6.0.6.tar
./install.sh
```

This creates:
- MO2 instance at `~/Games/mod-organizer-2-fallout4/`
- NXM handler at `~/.local/share/modorganizer2/modorganizer2-nxm-broker.sh`
- Instance symlink at `~/.config/modorganizer2/instances/fallout4`

### Key paths
- FO4 install: `~/.steam/steam/steamapps/common/Fallout 4/`
- Proton prefix: `~/.steam/steam/steamapps/compatdata/377160/`
- MO2 mods: `~/Games/mod-organizer-2-fallout4/mods/`
- MO2 downloads: `~/Games/mod-organizer-2-fallout4/downloads/`

### Required tools
- `protontricks` -- for running MO2's nxmhandler.exe
- `geckodriver` -- if using any browser automation

### Nexus Collections workaround
Collections are Vortex-only. To replicate a collection with MO2:
1. Use the Nexus v2 GraphQL API to get the mod list (needs API key)
2. Download mods individually
3. Install through MO2

Nexus API query for collection mod list:
```graphql
{
  collection(slug: "SLUG", domainName: "fallout4") {
    name
    latestPublishedRevision {
      modFiles {
        file { mod { modId name } fileId name version sizeInBytes }
        optional
      }
    }
  }
}
```

Headers: `apikey: YOUR_KEY`, `User-Agent: Vortex/1.12.0`

## Context
- Ubuntu 24.04, Steam with Proton
- MO2 runs under Wine/Proton via Steam launch
- F4SE goes in the game root directory (not managed by MO2)
- Wabbajack does NOT have a StoryWealth modlist as of April 2026
