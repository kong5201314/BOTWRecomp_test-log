# BOTWRecompAndroid — Mali-G1-Ultra MC12 test logs

Logs and screenshots for **BOTWRecompAndroid** (upstream issue **#44**), all captured on a **stock Mali
driver** (no Turnip / adrenotools package), so every line reflects the app's own decisions.

Two rounds are archived here:

| Round | Versions | What it shows |
|---|---|---|
| **Round 1** (2026-10-30) | 0.2.3 / 0.2.4 | Text missing on this GPU; the import decision was still wrong |
| **Round 2** (2026-11-01) | **0.2.6 / 0.2.9** | 🏆 Text fixed by default, and 0.2.9 cuts GPU memory by 41% |

---

## Device

| Item | Value |
|---|---|
| Device | OPPO Find X9 Pro (PLG110) |
| SoC / GPU | MediaTek Dimensity 9500 / **Arm Mali-G1-Ultra MC12** |
| Driver | `v1.r54p1-11eac0...`, **Vulkan 1.3.305**, stock (drivers folder empty) |
| OS | Android 17 (SDK 37), ColorOS V17.0.0 |
| Game | 1.8.2, recompiled mode, **never rebuilt** between any of these versions |
| Settings | `frame_rate=30`, `show_fps=1`, `resolution=1x`, `upscaler=sgsr1`, `gpu_heap=auto` |
| Launch | **app icon, no environment variables** (unless a filename says otherwise) |

---

## Round 2 — 0.2.6 and 0.2.9 (2026-11-01)

Both runs: same save file, same spot, standing still. Directly comparable.

### Text is fixed — no env vars needed

```
[vk] guest heap import left off (Mali: the game's text went missing, and memory ran out)
[vk] no heap import: shader global memory through CPU copies
```

`left off` appears **once**, `guest heap imported` **zero times**, on a plain icon launch. Title menu, save
list, inventory names and the description panel are all readable — the same screens that were completely
blank in Round 1.

### GPU memory — 0.2.9's ASTC path

| | 0.2.6 | 0.2.9 | Change |
|---|---|---|---|
| gpu mem (in-game, still) | **1,702 MB** | **1,010 MB** | 🔥 **−41%** |
| textures (same scene) | 814 | 1,053 | +29% |
| fps / max frame | 29.9–30.0 / 34–44 ms | 29.8–29.9 / 34–57 ms | unchanged |
| cold start | 796 ms | 679 ms | −15% |

Memory dropped by two fifths while the texture count went *up* by nearly a third — more content in less VRAM,
not less content.

Title screen quality improved visibly too: 0.2.6 renders the pre-rendered artwork as large blue/purple
**block mosaic** (`C-0.2.6-title-mosaic.png`), 0.2.9 renders it as a continuous image
(`D-0.2.9-title-clean.png`). Same build of the game, same driver, only the texture path differs.

### 18-minute session — no drift

| Segment | fps | gpu mem |
|---|---|---|
| 0–3 min (title + load) | 29.0 | 707 MB |
| 3–6 min | 29.4 | 1,061 MB |
| 6–9 min | 28.8 | 1,334 MB |
| 9–12 min | 29.9 | 1,343 MB |
| 12–15 min | 29.8 | 1,330 MB |
| 15–18 min | 29.8 | 1,340 MB |
| 18–21 min | 29.9 | 1,350 MB |

Memory plateaus at ~1,334 MB and then moves less than 20 MB for the remaining 15 minutes; peak for the whole
run is **1,384 MB**. Frame rate never degrades. `Scudo`, `FATAL`, `OOM`, `low memory`, `ANR`: **0 occurrences**.
Exit is clean:

```
21:21:38.906  I BOTWRecompAndroid: window closed
```

### One minor observation

Twice during the session, on `Motion: Auto`:

```
[motion] no gyroscope readings for 1 s while the game reads the sensor (Auto)
[motion] gyroscope readings again (Auto)
```

Readings recover by themselves after ~1 s and gameplay was not affected. Low priority, reported only for
completeness.

---

## Round 1 — 0.2.3 / 0.2.4 (2026-10-30, historical)

| | **A** (0.2.4, normal) | **B** (0.2.4, fix mode) | **C** (0.2.3, normal) |
|---|---|---|---|
| guest heap imported | yes, 3120 MB | no | yes, 3120 MB |
| title / inventory text | **blank** | correct | **blank** |
| in-game fps | 30.0 | 30.0 | 2–9 |

The arithmetic that explained it:

| Quantity | Value |
|---|---|
| `maxStorageBufferRange` on this driver | 2048 MB |
| Guest heap requested by the game | 3120 MB |
| Offset where the glyph atlas lives | ≈ 2.7 GiB |

That is what #121 later addressed by turning import and arena off by default on r54p1+ Mali drivers —
confirmed working on this device in Round 2.

---

## Repository layout

```
README.md
logs/A-0.2.4-normal-launch.log        Round 1: 0.2.4, icon launch, text blank
logs/B-0.2.4-fix-mode.log             Round 1: 0.2.4, HOST_IMPORT=0 + NO_GRES=1, text ok
logs/C-0.2.3-normal-launch.log        Round 1: 0.2.3 baseline, text blank
logs/D-0.2.6-normal-launch.log        Round 2: 0.2.6, icon launch, text ok
logs/E-0.2.9-normal-launch.log        Round 2: 0.2.9, full 18-minute session recording

screenshots/A-0.2.4-inventory-no-text.png    Round 1: every item name blank, black description box
screenshots/B-0.2.4-title-text-ok.png        Round 1: fix mode, title text present
screenshots/C-0.2.6-title-mosaic.png         Round 2: text ok, but blue/purple block mosaic
screenshots/D-0.2.9-title-clean.png          Round 2: same screen, mosaic gone
screenshots/E-0.2.6-inventory-text-ok.png    Round 2: inventory text ok on 0.2.6
screenshots/F-0.2.9-inventory-text-ok.png    Round 2: inventory text ok on 0.2.9
screenshots/G-0.2.9-18min-session.png        Round 2: end of the 18-minute run

BOTWRecomp-0.2.4-MaliG1Ultra-logs.zip        Round 1 bundle
```

Log files contain **only** lines emitted by the app (tag `BOTWRecompAndroid`); system noise was stripped.
Timestamps are device local time (UTC+8).

---

## Reproducing

Normal launch (no env vars):

```bash
adb shell am start -W -a android.intent.action.MAIN \
  -c android.intent.category.LAUNCHER \
  -n com.botwrecomp.android/com.botwrecompandroid.BOTWRecompAndroidActivity
```

Force-mode launch (0.2.4 only, kept for Round 1 reproduction):

```bash
adb shell am start -W \
  -n com.botwrecomp.android/com.botwrecompandroid.BOTWRecompAndroidActivity \
  --es env "LARECOMP_VK_HOST_IMPORT=0,LARECOMP_VK_NO_GRES=1,LARECOMP_NO_LAUNCHER=1"
```

The activity is `exported` with `launchMode=singleInstance`, so the `env` extra is only read on a cold start —
force-stop between runs, otherwise the extra is ignored and both commands give the normal behaviour.

Log capture:

```bash
adb logcat -c
adb logcat -s BOTWRecompAndroid:V > session.log    # run in a second shell for long sessions
```
