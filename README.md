# BOTWRecompAndroid — Mali-G1-Ultra MC12 test logs

Attachments for **BOTWRecompAndroid** (upstream: `BOTWRecompAndroid` issue **#44**, "Mali text rendering / GPU driver").
Everything here was produced on a **stock Mali driver** (no Turnip package involved), so the logs isolate the
app's own host-import decision from any third-party driver behaviour.

---

## Device

| Item | Value |
|---|---|
| Device | OPPO Find X9 Pro (PLG110) |
| SoC / GPU | MediaTek Dimensity 9500 / **Arm Mali-G1-Ultra MC12** |
| Driver | `v1.r54p1-11eac0...`, **Vulkan 1.3.305** |
| OS | Android 17 (SDK 37), ColorOS V17.0.0 |
| App | **0.2.4-beta** (versionCode 24) and **0.2.3-beta** |
| Game | 1.8.2, recompiled mode |
| Rebuild | **NO** — see note below |

> **The game code was not rebuilt between 0.2.1 and 0.2.4.** The app never asked for a rebuild, so all three
> logs run the *same* game binary. That rules out "stale recompiled code" as an explanation for the text loss —
> the only variable is the app version and the `LARECOMP_VK_HOST_IMPORT` flag.

---

## Repository layout

```
README.md                        this file
logs/A-0.2.4-normal-launch.log   0.2.4, launched from the app icon, no env vars
logs/B-0.2.4-fix-mode.log        0.2.4, same build, LARECOMP_VK_HOST_IMPORT=0 + NO_GRES=1
logs/C-0.2.3-normal-launch.log   0.2.3 baseline, launched from the app icon
screenshots/A-0.2.4-inventory-no-text.png   A: inventory open, item names blank, description panel empty
screenshots/B-0.2.4-title-text-ok.png       B: title menu text fully present
BOTWRecomp-0.2.4-MaliG1Ultra-logs.zip       the same set bundled, plus the raw unfiltered logcat excerpts
```

The `.log` files contain **only** lines emitted by the app (log tag `BOTWRecompAndroid`). Unrelated system
logcat noise was stripped so every remaining line is relevant. Timestamps are device local time (UTC+8).

---

## Result summary

| | **A** (0.2.4, normal) | **B** (0.2.4, fix mode) | **C** (0.2.3, normal) |
|---|---|---|---|
| guest heap imported | yes, 3120 MB | **no** | yes, 3120 MB |
| fallback line emitted | **0 occurrences** | yes (`HOST_IMPORT=0`) | **0 occurrences** |
| title / inventory text | **blank** | **correct** | **blank** |
| in-game fps | 30.0 fps, max 34–38 ms | 30.0 fps | 2–9 fps |

Two separate things happened between 0.2.3 and 0.2.4, and they must not be confused:

1. **Performance: fixed.** 0.2.3 dropped to 2–9 fps on this GPU (the uncached shared-memory change from #71).
   0.2.4 is back to a solid 30 fps with max frame time 34–38 ms.
2. **Text: still broken.** The `host_import` decision is unchanged. 0.2.4 still imports the 3120 MB guest heap,
   so glyph data stays out of reach and every string renders blank.

---

## Key lines — A (0.2.4, normal launch, text blank)

```
[vk] gpu 0: Mali-G1-Ultra MC12 (api 1.3.305)
[vk-cap] storage range 2048 MB, storage offset alignment 64, ...
[kernel] SetHeapSize 3120 MB
[vk] guest heap imported for shader global memory: 3120 MB
[vk] vertex cache off: no page write tracking
```

## Key lines — B (0.2.4, fix mode, text correct)

```
[vk] guest heap import left off (LARECOMP_VK_HOST_IMPORT=0)
[vk] no heap import: shader global memory through CPU copies
[vk] gmem off: stage 0 shader ... loads
```

## Key lines — C (0.2.3, normal launch, text blank)

```
[vk] gpu 0: Mali-G1-Ultra MC12 (api 1.3.305)
[vk-cap] storage range 2048 MB, ...
[kernel] SetHeapSize 3120 MB
[vk] guest heap imported for shader global memory: 3120 MB
```

**C vs A:** those four lines are byte-identical and the fallback line is absent in both. So the caching change
landed, but the import *decision* on this GPU did not.

---

## Why it breaks — the arithmetic

| Quantity | Value |
|---|---|
| `maxStorageBufferRange` on this driver | **2048 MB** |
| Guest heap requested by the game (`SetHeapSize`) | **3120 MB** |
| Offset where the font/glyph atlas lives | **≈ 2.7 GiB** |

The adapter advertises a 2 GiB storage range, the game asks for 3.1 GiB, and the glyph data sits above the
2 GiB line. The import therefore cannot be represented as a single host-visible binding, and the text path
silently produces nothing. Turning the import off (`LARECOMP_VK_HOST_IMPORT=0`) makes the runtime route shader
global memory through CPU copies, which is slower in theory but on this GPU measures **identical** — 30.0 fps
either way, because the bottleneck here is not the copy path.

---

## Suggested fix direction

The whitelist in `runtime/nvn/vulkan/device.c` matches on the exact string:

```c
strstr(g_vk.props.deviceName, "Mali-G2-Ultra")
```

This device reports **`Mali-G1-Ultra MC12`** — one digit off (`G1` vs `G2`), so it misses the list and keeps the
import enabled. Two options, in order of robustness:

1. **Decide on the capability, not the name.** If `maxStorageBufferRange` (or the corresponding
   `maxMemoryAllocationSize`) is smaller than the requested guest heap, disable host import regardless of GPU
   family. That fixes every Mali part at once, including future ones.
2. **If a name list must be kept,** match the family prefix rather than the full string
   (`Mali-G1-Ultra`, `Mali-G2-Ultra`, `Mali-G7xx`, `Immortalis-*`, …) or the `Mali-` vendor prefix plus the
   storage-range check.

Related prior art in this repo's issue tracker: the same symptom was reported on Mali-G2-Ultra, and the
runtime-side workaround there was the same env var. This report adds a device that is one character outside the
current list, plus proof that the 0.2.4 caching fix did not change the import decision.

---

## Reproducing A vs B

A — normal launch (text blank):

```bash
adb shell am start -W -a android.intent.action.MAIN \
  -c android.intent.category.LAUNCHER \
  -n com.botwrecomp.android/com.botwrecompandroid.BOTWRecompAndroidActivity
```

B — fix mode (text correct):

```bash
adb shell am start -W \
  -n com.botwrecomp.android/com.botwrecompandroid.BOTWRecompAndroidActivity \
  --es env "LARECOMP_VK_HOST_IMPORT=0,LARECOMP_VK_NO_GRES=1,LARECOMP_NO_LAUNCHER=1"
```

The activity is `exported` with `launchMode=singleInstance`, so the `env` extra is only read on a cold start —
`FLAG_ACTIVITY_CLEAR_TASK` or a force-stop is needed between A and B, otherwise the extra is ignored and you
get A's behaviour from both commands.

Tip for anyone doing the same test: use `-W` and force-stop between runs, and grep the log for
`guest heap import` — that single line tells you which branch you are on before the game even reaches the title
screen.

---

## Log capture method

```bash
adb logcat -c
# launch, wait for the title screen / inventory to be visible
adb logcat -d -s BOTWRecompAndroid:V > A-0.2.4-normal-launch.log
```

Filtered by tag only, then noise lines removed by hand. Nothing else was edited.
