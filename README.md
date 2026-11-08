# Genshin Impact FPS Unlocker — 7.2 update

Fork of [34736384/genshin-fps-unlock](https://github.com/34736384/genshin-fps-unlock) (via `lupixele`),
updated so the frame-rate unlock works again on current Genshin Impact builds and hardened against the
false-positive / kick problems described below.

- Externally sets the game's frame limit (no drivers, no game files touched)
- Handle-protection bypass included (the game is started by the unlocker)
- OS and CN clients
- Tested against `GenshinImpact.exe` **7.2.0** (451,130,280 bytes, 2026‑11‑04)

## Why the old build stopped working

Since 7.x the unlocker fails with `outdated pattern` (or silently leaves the game at 60 fps). The old
resolver scanned for

```
B9 3C 00 00 00 E8      ; mov ecx, 60 ; call <managed setter>
```

and then walked `jmp`/`call` thunks down to a body that began with `mov [rip+disp], ecx`, reading the
static frame-limit address out of it. On 7.2.0 **all 22** matches of that byte pattern exist, but the
call now goes straight into native code in `.text`, so the `*dest == 0xE9` filter matches nothing and the
walk never resolves — hence "outdated pattern".

### The fix

The game still keeps the limit in static ints and still resets them with a tiny managed setter, compiled to

```
C7 05 disp32 3C 00 00 00 C3   ; mov dword ptr [rip+disp], 60 ; ret
```

The unlocker now finds those setters directly, validates that the target is a committed, writable page
holding a plausible frame rate, and writes the requested value to **every** one of them (7.2 has two
duplicate settings classes). This shape is version independent, and the old thunk walk is kept as a
fallback for older builds.

Verified statically on 7.2.0: the two resolved statics are each read back into a settings object at
`+0x80` (`mov eax, [static] ; mov [rsi+0x80], eax`) — i.e. they are the frame-limit fields.

## Safety: the 120 fps cap

Since December 2025 the server kicks sessions whose target frame rate goes above **120**
(`10310-4001`, typically in Stygian Onslaught / Miliastra). Reported values that get kicked: 144, 165, 275;
120 is safe in every report.

Settings → General → **Cap FPS at 120 (recommended)**, on by default. It clamps whatever you type in the
main window to 120 before it reaches the game. Unchecking it restores the raw value (your risk).

This tool cannot change whether HoYoverse tolerates it — it only stops the tool from tripping the check it
can control.

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
zig c++ -target x86_64-windows-gnu -shared -O2 -std=c++20 -o UnlockerStub/UnlockerStub.dll \
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
4. Leave it running in the tray. `Cap FPS at 120` stays on unless you deliberately turn it off.

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
