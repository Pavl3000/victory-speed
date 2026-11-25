# ztrx

**ztrx** runs the Xbox Series build of the game natively on Windows PCs with
NVIDIA graphics cards, from game files you supply yourself.

**This repository contains no game files or assets.** You need your own copy
of the game. Not affiliated with or endorsed by its developer or publisher.

## Status (2026-11-25)

Tested on Windows 11 with an **NVIDIA GeForce RTX 3050** (8 GB):

| | |
|---|---|
| Boot, first-boot screens, intro, opening credits | works |
| Prologue | playable from the cell to the boat chase: cell, cafeteria, robbery, police chase (wanted level, minimap), boat, binoculars |
| Stability | 1 h 30 min+ in one session without a crash |
| Frame rate | 1-3 fps in the world (the first run is slower while shaders translate; they are cached after that) |
| Known problems | anything in shadow is black and bright scenes overexpose (the game's ray-traced lighting does not run yet); some blocky artifacts; boats sit low in the water |

## This branch: `merge/nvidia`

fxceless's third session (`a253706`, `0b0074e`) is merged as well: a frame log and a card meter, a driver watch, copies in place measured and fixed, our threads on the fast cores, a forbidden self-copy removed, and draw windows measured again.

The branch `merge/fxceless-latest` carries that on top.

`merge/nvidia` is **ztrx plus the frame-rate work of
[fxceless/victory-fork](https://github.com/fxceless/victory-fork)**, merged
from `fxceless/main` at `f85101b` (2026-11-25). The two forks share the same
upstream point (`ffe310e`), so every source file merged without a conflict;
only this README did, and it keeps ztrx's text plus this note. Nothing from
that work is enabled by default.

What comes from fxceless, in their words:

- Frame rate, second night: a submit thread of the layer's own, a copy queue
  for fetches, indirect draw windows made safer, a profiler; all behind
  switches.
- Frame rate, processor side: a cheap fingerprint in the descriptor sync,
  draw-table slots left to their draws, no failing write-watch calls, card
  fetches inside their flights.
- The first room renders right: byte-copied descriptor records are mirrored,
  views off their copy's elements get their own bytes, card room for the first
  world frame, levels on demand run on the card.

Switches that came with it (off by default unless said otherwise; each is
described in a comment where it is read):

| Switch | Effect |
|---|---|
| `VICTORY_ASYNC_SUBMIT=1` | submissions prepared and sent by a thread of the layer, not by the title's render thread |
| `VICTORY_QUEUE2=1` | fetches of what shaders wrote go to a copy queue |
| `VICTORY_INDIRECT_WINDOW=1` | a window of indirect draws recorded as one dispatch; it lost the device three times on their machine before the fix, and their checker `VICTORY_INDIRECT_WINDOW_CHECK=1` is what found the cause |
| `VICTORY_CARD_FETCH_INFLIGHT=1` | card fetches kept inside their flights; their copy-back thread's wait went from 181 ms a frame to a fraction of it |
| `VICTORY_LEVELS_ON_DEMAND=1` | a texture the title fills takes card memory only for the levels the title reaches |
| `VICTORY_CARD_BUFFERS_MIN_MB=1`, `VICTORY_CARD_BUFFERS_MB=1024` | room kept for the card copies of title buffers |
| `VICTORY_INPLACE` | title memory opened as a graphics heap, buffers used in place instead of copied (first stage only) |
| `VICTORY_PAGE_WRITES=guards`, `VICTORY_DSTORAGE_RETRIES=0` | more page-write guards; page-destination reads retried |
| `VICTORY_MERGE_GAP=64 VICTORY_MERGE_RUNS=4096` | title bytes under flights' shader writes joined over small gaps (not exact) |
| `VICTORY_PROFILE=1` | sampling profiler inside the layer (diagnostics build) |
| `VICTORY_WAVE_WIDE=1` | the 64-lane barrier form they had to turn off after a device hang |

Their measured settings on that machine: `VICTORY_RELAY=2`, or
`VICTORY_LATE_WAIT=3 VICTORY_WIDE_BACK=1 VICTORY_CARD_BUFFERS_MIN_MB=1
VICTORY_CARD_BUFFERS_MB=1024`, took 420-450 ms a frame down to 240-350 ms.

## Requirements

- Windows 10/11 x64, a CPU with AVX2
- A Direct3D 12 GPU with shader model 6.6 and mesh shaders (NVIDIA RTX 20 series or newer)
- Rust (the version in `rust-toolchain.toml`)
- Several GB of free Windows commit memory (raise the paging file if `crashes.log` reports error 1455)

## Build

```powershell
cargo build --release -p victory
```

## Play

```powershell
powershell -ExecutionPolicy Bypass -File scripts\play.ps1 -Game "<your game folder>\<the game's exe>"
```

`scripts\play.ps1` keeps the shader cache and saves in folders you choose
(`-Cache`, `-Data`) and presses no buttons for you. Controller: XInput pads
(a DualShock/DualSense through DS4Windows works). With no pad connected the
keyboard acts as one while the game window has focus.

## Faster settings (optional, off by default)

`VICTORY_RELAY=2`, or `VICTORY_LATE_WAIT=3 VICTORY_WIDE_BACK=1` cut frame
time on the test machines. Set them in PowerShell before running the script.

## Debugging switches

| Switch | Effect |
|---|---|
| `VICTORY_CAPTURE=<dir>` + `VICTORY_CAPTURE_FRAMES=<list>` | save those frames as PNG (diagnostics build) |
| `VICTORY_KEYS=30:Enter,35:X` | press keys at those seconds (diagnostics build) |
| `VICTORY_D3D12_DRED=1` | on a GPU failure, report the commands in flight |
| `VICTORY_PIPELINE_WAIT_MS=24` | shorter per-frame wait for pipelines still being built |

Diagnostics build: `cargo build --profile diag -p victory --features diagnostics`

## Credits

- **[VIctory](https://github.com/Marvisak/VIctory)** by Marvisak and its
  contributors is the base of everything here: the loader, the runtime
  services, the Direct3D 12 back end and the shader recompiler. Licensed
  GPL-3.0-or-later; so is this fork.
- **[fxceless/victory-fork](https://github.com/fxceless/victory-fork)** by
  fxceless: the frame-rate work merged into `merge/nvidia` (see above).
- **[XCL](https://github.com/ivall/xcl)** by ivall and its contributors
  (GPL-2.0-only) was used as a reference for how the same title behaves under
  another layer. Fixes inspired by it were written anew in VIctory's code.
- The guard over every view of a memory page is code by **myokex**
  ([victory-amd](https://github.com/myokex/victory-amd), commit `c6dc4d8`),
  which adapts pull request #115 here by Spexivus, its correction by joroyt and
  a part of VIctory #1122 by tightfall. The read-beside fix for kernel reads
  into title memory is by **Joshua Odom** (commit `10ea38e` of the same fork
  family). Both are carried here unchanged apart from one merge conflict.

## License

GPL-3.0-or-later (see `LICENSE`). Based on VIctory (Marvisak) and victory-fork (fxceless).
