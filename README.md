# Genshin Impact FPS Unlocker — 7.2 update

Fork of [34736384/genshin-fps-unlock](https://github.com/34736384/genshin-fps-unlock) (via `lupixele`),
updated so the frame-rate unlock works again on current Genshin Impact builds and hardened against the
false-positive / kick problems described below.

- Externally sets the game's frame limit (no drivers, no game files touched)
- Handle-protection bypass included (the game is started by the unlocker)
- OS and CN clients
- Tested against `GenshinImpact.exe` **7.2.0** (451,130,280 bytes, 2026‑11‑04)

## The stub resolver: unchanged upstream code

This fork's stub is the upstream `UnlockerStub/dllmain.cpp` **verbatim** (the only edit is
`__declspec(align(8))` -> `alignas(8)` so it also builds with clang/zig). Nothing custom was added to the
resolver, because upstream's resolver already handles 7.2.0.

An earlier revision of this fork replaced it with a "scan for `C7 05 ... 3C 00 00 00 C3` setters" method.
That method matched statics that are *not* the live frame-limit field and returned success, so
`SetupData()` never fell through to the working code and the game stayed at 60 fps. It has been removed.

### What upstream does, verified on 7.2.0

Upstream scans the `il2cpp` section for

```
B9 3C 00 00 00 E8      ; mov ecx, 60 ; call rel32
```

and keeps the first hit whose call target begins with `E9`. On `GenshinImpact.exe` 7.2.0
(451,130,280 bytes, 2026-11-04) that is:

| Step | Address |
| --- | --- |
| `B9 3C 00 00 00 E8` hit | `0x6d6d52c` (also `0x6e1b573`) |
| call target, `E9` thunk | `0x146c6c0` -> `0x16ba230` |
| setter body | `89 0D 2E 65 EF 03 C3` = `mov [rip+0x03EF652E], ecx ; ret` |
| resolved static | `0x55b0764` |

`89 0D disp32` and `C7 05 disp32` encode the displacement the same way, so the same arithmetic
(`rip + disp + 6`) recovers the static for both. Only 2 of the 22 raw pattern hits pass the `E9` filter,
and both resolve to the same field, which is why upstream only needs the first one.

## Antivirus / Smart App Control

This is not malware: it sets one integer inside your own game process. Detection happens because

1. the exe carries an **unsigned** `UnlockerStub.dll` and loads it into the game with
   `SetWindowsHookEx` (that is the handle-protection bypass), and
2. heuristic engines score `WriteProcessMemory` / `SetWindowsHookEx` / self-elevation patterns highly —
   `Wacatac`, `Gen:Suspicious.*`, `Trojan:Script/Wacatac` and friends are typical.

Changes in this fork to reduce that noise:

- the stub is only extracted when it is missing or its SHA‑256 differs (no per-launch "dropper" behaviour)
- the window hook is unhooked when the game exits and on dispose (it used to leak across restarts)
- `ReleaseMin` builds contain no debug info; version/product metadata is set (`3.6.0.0`)
- the stub is built from source in this repo — you can rebuild and compare

What actually removes the warning is **code signing**. Upstream used a free SignPath Foundation
certificate; a reproduced build will never have that reputation. If Smart App Control blocks the stub,
either sign your build, or add an exclusion, or keep Smart App Control off.

Note that current clients ship `AntiCheatExpert/` and `HoYoKProtect.sys` next to the game; no FPS
unlocker is "undetectable" against a kernel driver, and none should claim to be.

## Forks (surveyed 2026‑11‑08)

| Repository | State | Notes |
| --- | --- | --- |
| `34736384/genshin-fps-unlock` | upstream, last push 2025‑12‑16 (v3.5.0) | 3.6k ★, 257 forks |
| `xiaonian233/genshin-fps-unlock` | active, releases up to game 6.7 | ~840 ★, Chinese, own per-version patterns |
| `winTEuser/Genshin_StarRail_fps_unlocker` | active, last commit 2026‑10‑05 | ~725 ★, Genshin + HSR, own pattern set, shellcode/mem-load, many extras |
| `einkoro/genshin-fps-unlock` | active, 2026‑08‑23 | keeps this tree alive |
| `OscarSheng`, `Error44s`, `YaeSakura0v0`, `3096`, `vvull`, `Arbitraryo`, `Enderderder` | updated forks | mostly one-off fix releases |
| `sefinek/Genshin-FPS-Unlocker` (Genshin Stella Mod) | fork with ReShade integration | documents Smart App Control blocking the stub |
| ~230 other forks | stale | identical to upstream at fork time |

Newest genuinely-maintained code that supports current game versions is `winTEuser`'s and
`xiaonian233`'s; both replaced the pattern logic wholesale rather than fixing this tree's resolver.

## Other fixes in this update

- **Game process priority was never applied** — the combo was bound to `Config.Priority` but nothing
  passed it to the process. Now `SetPriorityClass` is called on launch.
- **Process-handle leak**: the poll loop created a `Process` for every running process once a second and
  disposed only the one it kept.
- **Window hook leaked** between game restarts (`UnhookWindowsHookEx` was never called).
- **Removed the update nag** to `https://ys.ex-m.net/fps-unlock/version`, a third-party endpoint
  unrelated to this project, which pushed users to another project's downloads.
- Stub sources no longer use MSVC-only idioms; `GNUmakefile` builds the stub with zig.

## Building

Requirements: .NET 8 SDK (or newer) and [zig](https://ziglang.org/) 0.17, or Visual Studio 2022 (MSVC).

```sh
mkdir -p .build
zig cc -target x86_64-windows-gnu -O2 -c UnlockerStub/Zydis.c -o .build/zydis.o
zig c++ -target x86_64-windows-gnu -shared -O2 -std=c++23 -o UnlockerStub/UnlockerStub.dll \
    UnlockerStub/dllmain.cpp UnlockerStub/Utils.cpp .build/zydis.o -ldbghelp -lntdll
cp UnlockerStub/UnlockerStub.dll unlockfps_nc/Resources/UnlockerStub.dll

dotnet publish unlockfps_nc/unlockfps_nc.csproj -c ReleaseMin -r win-x64 -f net8.0-windows \
    --self-contained false -p:PublishSingleFile=true -p:PublishReadyToRun=false
```

`make` (with zig + dotnet installed) does the same on Linux/macOS. With MSVC, build
`UnlockerStub/UnlockerStub.vcxproj` (x64, Release) and copy the dll to `unlockfps_nc/Resources/`.

The output is a single `unlockfps_nc.exe` (~4.2 MB) that needs the .NET Desktop Runtime 8.0.

## Usage

1. Run `unlockfps_nc.exe` as administrator (it self-elevates; the game needs it anyway).
2. First run: point it at `GenshinImpact.exe` (or use *Set Path*; it also finds HoYoPlay installs).
3. Press **Start Game** — the unlocker launches the game and keeps the limit set while it runs.
4. Leave it running in the tray.

Do not put the exe inside the game folder.

## Notes / limits

- HoYoverse is aware of this tool; only the frame-rate unlock is covered by that tolerance. Other
  third-party plugins (the DLL list in Settings) are at your own risk.
- **Mobile UI** (touch layout) is *not* verified on 7.x — upstream issue #640 reports it broken, and this
  update does not change that subsystem.
- Stutter above the GPU's real capability is not a bug of the unlocker.
- If the game is started from the official launcher instead of the unlocker, the unlocker attaches through
  the window hook; that path is unchanged.

## Legal

MIT, same as upstream. Use at your own risk; nothing here is affiliated with HoYoverse.
