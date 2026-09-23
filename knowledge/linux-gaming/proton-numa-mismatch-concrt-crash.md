---
title: Proton crash on multi-socket (NUMA) hosts in DLLs using the MSVC Concurrency Runtime (Fallout 4 LooksMenu f4ee.dll)
tags: [proton, wine, numa, concrt, fallout4, looksmenu, f4ee, dual-socket, xeon, crash]
verified: 2026-09-23
platform: linux
---

## Problem
Fallout 4 (1.10.163, F4SE, Mod Organizer 2, Proton 9.0) crashed every time on
**New Game** on a dual-socket Xeon workstation (2 sockets, 48 threads, 2 NUMA
nodes). Buffout 4 log:

```
Unhandled exception "EXCEPTION_ACCESS_VIOLATION" at ... f4ee.dll+0076BD4
RSI ... (Concurrency::details::ThreadScheduler*)
R14 ... (Concurrency::details::GlobalCore::TopologyObject*)
stack: ... (char*) "GetTraceEnableFlags"
```

f4ee.dll is LooksMenu. Its data files were fine. The crash is in the
**MSVC Concurrency Runtime (ConcRT)** statically linked into the DLL, while it
builds its CPU topology the first time a parallel task runs. The disassembly
at the crash site sets a processor bit in `node[1]`'s mask, but the node array
was sized for one node.

Root cause: Wine/Proton reports NUMA **inconsistently** on multi-node hosts:

| Proton | `GetNumaHighestNodeNumber` | NumaNode records in `GetLogicalProcessorInformationEx` |
|---|---|---|
| 9.0 (Beta), Hotfix, 10.0 (10.0-4b) | 0 | 0, 1 |
| Experimental (Sept 2026, 11.0-100) | 1 | 0, 1 |

ConcRT sizes its node table from the first answer and fills it from the second,
so it writes out of bounds. Any Windows program or DLL that uses ConcRT (for
example `concurrency::parallel_for` or PPL tasks) is probably affected on
multi-socket Linux hosts. Single-socket desktops report one node and never hit
this.

## Solution
Wine reads NUMA topology from `/sys/devices/system/node/online` and
`/sys/devices/system/node/node*/cpumap` (visible with `strings` on
`files/lib/wine/*-unix/ntdll.so`). Give the game a private mount namespace
where `/sys/devices/system/node` shows **one node containing all CPUs**, then
drop back to the real UID. Nothing changes system-wide and no root is needed
(requires unprivileged user namespaces, which Ubuntu 24.04 allows by default).

`~/tools/onenode.sh`:

```sh
#!/bin/sh
# Present a single NUMA node (all CPUs) to the wrapped command, then run it as the original user.
# Steam launch option:  /home/USER/tools/onenode.sh %command%
if [ "$ONENODE_STAGE" != 2 ]; then
  exec unshare -r -m env ONENODE_STAGE=2 ONENODE_UID="$(id -u)" ONENODE_GID="$(id -g)" "$0" "$@"
fi
F="${XDG_RUNTIME_DIR:-/tmp}/onenode.$$"
mkdir -p "$F/node0"
echo 0 > "$F/online"
# union of all real nodes' cpumaps -> node0
python3 - "$F/node0/cpumap" <<'PY'
import glob,sys
m=0
for p in glob.glob('/sys/devices/system/node/node[0-9]*/cpumap'):
    m|=int(open(p).read().strip().replace(',',''),16)
h='%x'%m; h=h.rjust((len(h)+7)//8*8,'0')
open(sys.argv[1],'w').write(','.join(h[i:i+8] for i in range(0,len(h),8))+'\n')
PY
echo 0 > "$F/has_cpu"; echo 0 > "$F/possible"
mount --bind "$F" /sys/devices/system/node || { echo "onenode: bind mount failed" >&2; exit 1; }
exec unshare -U --map-user="$ONENODE_UID" --map-group="$ONENODE_GID" env -u ONENODE_STAGE -u ONENODE_UID -u ONENODE_GID "$@"
```

The two-stage `unshare` matters. With a single `unshare --map-current-user -m`,
`mount` fails ("must be superuser"): the capabilities are lost on `exec` because
the UID isn't 0. So stage 1 maps to root, mounts, then stage 2 creates a nested
user namespace that maps back to the real UID.

Steam launch options (keep any existing variables in front):

```
WINEDLLOVERRIDES="f4se_1_10_163=n,b" /home/USER/tools/onenode.sh %command%
```

Steam's pressure-vessel container (SteamLinuxRuntime_sniper) starts fine inside
it and inherits the overmount. Result: highest node 0, NumaNode records [0],
and LooksMenu character creation works on Proton 9.

### Verify without launching the game
Run a Windows ctypes probe under Proton's `wine` with a scratch prefix, using
python.org's `python-3.12.x-embed-amd64.zip`:

```python
import ctypes, ctypes.wintypes as W
k=ctypes.WinDLL('kernel32')
k.GetNumaHighestNodeNumber.argtypes=[ctypes.POINTER(W.ULONG)]
n=W.ULONG(); k.GetNumaHighestNodeNumber(ctypes.byref(n)); print('highest',n.value)
k.GetLogicalProcessorInformationEx.argtypes=[ctypes.c_int,ctypes.c_void_p,ctypes.POINTER(W.DWORD)]
L=W.DWORD(); k.GetLogicalProcessorInformationEx(1,None,ctypes.byref(L))   # 1 = RelationNumaNode
b=ctypes.create_string_buffer(L.value); k.GetLogicalProcessorInformationEx(1,b,ctypes.byref(L))
off=0; nodes=[]
while off<L.value:
    size=int.from_bytes(b.raw[off+4:off+8],'little'); nodes.append(int.from_bytes(b.raw[off+8:off+12],'little')); off+=size
print('records',nodes)   # consistent when max(records) == highest
```

```sh
WINEPREFIX=/tmp/probe/pfx "$PROTON/files/bin/wine" wineboot -i
WINEPREFIX=/tmp/probe/pfx ~/tools/onenode.sh "$PROTON/files/bin/wine" py/python.exe probe.py
```

## What didn't work
- **`WINE_CPU_TOPOLOGY=12:0,1,...,11`**: changes the core/package records only.
  The NUMA records still list both host nodes with host CPU numbers (up to 47),
  and the game crashes at the same offset.
- **Proton Experimental**: its NUMA answers are consistent, so LooksMenu stops
  crashing, but on this host threads spin-waited forever. MO2's
  DirectoryRefresher looped on `pselect6(1ms)`/`sched_yield` with idle worker
  threads, so the plugin list stayed empty, Run was greyed out, and MO2 hung on
  exit. `PROTON_NO_FSYNC=1 PROTON_NO_ESYNC=1` fixed MO2, but the game then
  crashed every time on the main menu (`Fallout4.exe+22E5B45`, MainMenu loading
  the DLC banner texture).
- **Proton 10.0**: same inconsistency as 9.0.
- `PROTON_USE_XALIA=0` and removing `PROTON_LOG` made no difference to the MO2 hang.
- The thermal and affinity pinning by an unrelated host daemon was not the cause.

## Also seen (Fallout 4 specific)
After the crash was fixed, the New Game loading screen seemed to hang forever.
The main thread was repeatedly failing lookups for loose
`Sound/Voice/Fallout4.esm/RelayTowerAnnouncerVoice/*.xwm` files (they're in
`Fallout4 - Voices.ba2`, so that was a red herring). The real cause was a mod
popup hidden behind the loading screen. **Alt-Tab out and back, then press
Enter/Esc**, and loading continues.

## Context
- Ubuntu 24.04, kernel 6.8.0-107, util-linux 2.39.3, 2x Xeon E5-2690 v3 (HP Z840), NVIDIA GV100
- Steam client Sept 2026; Proton 9.0-203, 10.0-4b, Experimental 11.0-100
- Fallout 4 1.10.163 (downgraded), F4SE 0.6.23, LooksMenu 1.6.20, Buffout 4 1.28.6, MO2 2.5.2
- Related: `mo2-fallout4-linux-proton-setup.md` in this directory
