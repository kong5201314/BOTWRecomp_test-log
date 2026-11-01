BOTWRecompAndroid 0.2.4-beta — Mali-G1-Ultra MC12 test attachments
================================================================================

Device    : OPPO Find X9 Pro (PLG110)
SoC / GPU : MediaTek Dimensity 9500 / Arm Mali-G1-Ultra MC12
Driver    : v1.r54p1-11eac0.d66d1d5ec15c63b7cede1928bfff931d, Vulkan 1.3.305
OS        : Android 17 (SDK 37), ColorOS V17.0.0
App       : 0.2.4-beta (versionCode 24), stock Mali driver (no Turnip package)
Game      : 1.8.2, recompiled mode
Rebuild   : NO. The game code was NOT rebuilt going from 0.2.1 to 0.2.4
            (the app never asked for a rebuild), so A and C use the same
            game binary. This rules out "stale game code" as an explanation.

WHAT IS IN THIS FOLDER
--------------------------------------------------------------------------------
A-0.2.4-normal-launch.log    0.2.4, launched from the app icon, no env vars
B-0.2.4-fix-mode.log         0.2.4, same build, HOST_IMPORT=0 + NO_GRES=1
C-0.2.3-normal-launch.log    0.2.3 baseline, launched from the app icon

A-0.2.4-inventory-no-text.png   screenshot of A: inventory open, every item
                                name blank, description panel an empty black box
B-0.2.4-title-text-ok.png       screenshot of B: title menu text fully present

The .log files contain ONLY lines emitted by the app (log tag
BOTWRecompAndroid). The rest of logcat was unrelated system noise and has been
removed, so the files stay small and every line is relevant. Timestamps are
device local time (UTC+8).

RESULT SUMMARY
--------------------------------------------------------------------------------
                         A (normal)              B (fix mode)
guest heap imported      yes, 3120 MB            no
fallback line emitted    0 occurrences           yes, HOST_IMPORT=0
title / inventory text   blank                   correct
in-game fps              30.0 fps, max 34-38 ms 30.0 fps

KEY LINES — A (0.2.4, normal launch)
--------------------------------------------------------------------------------
  [vk] gpu 0: Mali-G1-Ultra MC12 (api 1.3.305)
  [vk-cap] storage range 2048 MB, storage offset alignment 64, ...
  [kernel] SetHeapSize 3120 MB
  [vk] guest heap imported for shader global memory: 3120 MB
  [vk] vertex cache off: no page write tracking

KEY LINES — B (0.2.4, fix mode)
--------------------------------------------------------------------------------
  [vk] guest heap import left off (LARECOMP_VK_HOST_IMPORT=0)
  [vk] no heap import: shader global memory through CPU copies
  [vk] gmem off: stage 0 shader ... loads

KEY LINES — C (0.2.3, normal launch)
--------------------------------------------------------------------------------
  [vk] gpu 0: Mali-G1-Ultra MC12 (api 1.3.305)
  [vk-cap] storage range 2048 MB, ...
  [kernel] SetHeapSize 3120 MB
  [vk] guest heap imported for shader global memory: 3120 MB

C vs A: those four lines are identical, and the fallback line is absent in
both. So 0.2.4 changed how the shared memory is cached (the 2-9 fps regression
is gone, 30 fps is now held), but the host_import decision itself did not
change on this GPU.

REPRODUCING A vs B
--------------------------------------------------------------------------------
A: adb shell am start -W -a android.intent.action.MAIN \
       -c android.intent.category.LAUNCHER \
       -n com.botwrecomp.android/com.botwrecompandroid.BOTWRecompAndroidActivity

B: adb shell am start -W -n com.botwrecomp.android/com.botwrecompandroid.BOTWRecompAndroidActivity \
       --es env "LARECOMP_VK_HOST_IMPORT=0,LARECOMP_VK_NO_GRES=1,LARECOMP_NO_LAUNCHER=1"
