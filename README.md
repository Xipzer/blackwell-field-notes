# Blackwell field notes

Field notes from tuning one water-cooled RTX 5090 Founders Edition over two days (17 and 18 Nov 2026): what we
tried, what broke, the numbers, and what we still do not understand.

**Status (18 Nov 2026, morning):** the max curve is done and passes everyday use and stress tests when warm. It still
hangs at random in one heavy scene of 3DMark Time Spy Extreme Graphics Test 2, more often with a cold core, and we
have not found the cause. Clock caps at or under 3,090 MHz stopped it at 600 W. A second, flattened profile is being
tested.

Test your own card. Do not copy our numbers. Every chip, board and loop is different.

The tools used here are in [blackwell-overclock-autopilot](https://github.com/Xipzer/blackwell-overclock-autopilot).
The CPU and RAM side of the same machine is in [raptor-lake-field-notes](https://github.com/Xipzer/raptor-lake-field-notes).

Labels:

- **Measured**: read from a log or a benchmark run on this machine.
- **Source**: a public source says so (link given).
- **Estimate**: our own arithmetic or reasoning.

## The machine

| | |
|---|---|
| GPU | RTX 5090 Founders Edition, VBIOS 98.02.2E.00.03, 600 W limit (700 W with the override below), PCIe 5.0 x16, water block in a custom loop shared with the CPU |
| CPU / board | i9-14900KS, ASRock Z790I Lightning WiFi (see raptor-lake-field-notes) |
| OS / driver | Windows 11, NVIDIA driver 621.24 |
| Tools | MSI Afterburner 4.6.7 Beta 2, nvidia-smi, 3DMark (Time Spy Extreme, Steel Nomad), FurMark 2.11, OCCT 17.1.12, our CUDA checks, nvflash 5.867 (read only), Vix Editor 0.1.45, Vix Loader 0.5.0 |
| Cooling | GPU core about 18 to 20 C over water at 600 to 700 W; water 22 to 36 C during the tests |

## Summary: what works and what does not

| Tried | Result |
|---|---|
| Per-voltage search with a CUDA check | works, but finds compute limits 30 to 110 MHz above what 3D survives |
| Per-voltage search with FurMark (3D) | works, but a heavy steady load overstates the limit: the hot core drops temperature bins and never runs the clock it was given |
| Curve from the real clocks proven in 3D (Profile 1) | passes warm: 3 Time Spy Extreme runs, Steel Nomad Stress Test 99.2 % |
| Profile 1 cold (water 22 to 27 C) | hangs at random in Time Spy Extreme GT2 |
| Lowering the 1000 to 1025 mV points by 15 or 30 MHz | still hangs (both on run 2) |
| Whole curve minus 30 MHz | 7 clean runs, then most likely a hang on run 8 of the session (see data) |
| Clock caps 3,202 and 3,232 MHz | no effect: heavy scenes never reach them |
| Clock caps 3,060 and 3,090 MHz (600 W) | **13 of 13 cold runs clean** |
| Clock caps 3,135 and 3,150 MHz (700 W) | hang (run 5 and run 1) |
| Memory +2000 instead of +3000 | still hangs |
| Fresh reboot (driver state reset) | still hangs |
| 700 W VBIOS override | works (power.max_limit 700 W, best Time Spy Extreme graphics 28,223), but Time Spy peaks at about 592 W: power was not the limit |
| Stock VBIOS at 600 W | still hangs (hard hang): the override is not the cause |
| NVVDD and MSVDD REL ceiling 1,110 -> 1,150 mV | no effect: this chip's REL is about 1,070 mV |
| MSVDD REL floor 1,100 mV | 4 hangs in 9 runs vs 5 in 8 without: not a fix |
| Afterburner voltage slider | no effect on driver 621.01 and later, which ignores runtime voltage offsets (source: Vix Editor and mVolt+ documentation) |
| FurMark, FurMark load steps, a CUDA load ramp, OCCT 3D Adaptive, Steel Nomad Stress Test | none reproduce the hang |

## 1. Per-voltage search, compute (measured)

Method: write an offset to an Afterburner profile, pin the clock with `nvidia-smi -lgc F,F`, run a CUDA FP32
checksum check with pulsed load for 60 s, raise F in 15 to 30 MHz steps until the first failure. Full method:
[METHOD.md](https://github.com/Xipzer/blackwell-overclock-autopilot/blob/main/docs/METHOD.md).

| Voltage | Factory clock | Highest clean (lock) | First fail | Offset clean / fail | How it failed |
|---|---|---|---|---|---|
| 900 mV | 2,002 | **2,497** | 2,512 | +495 / +510 | PC went down |
| 950 mV | 2,535 | **3,030** | 3,045 | +495 / +510 | PC went down; isolated re-test: 5 driver resets, recovered |
| 1000 mV | 2,715 | **3,195** | 3,225 | +480 / +510 | PC went down (0x133); isolated re-test: PC went down again |
| 1050 mV | 2,850 | **3,300** | 3,330 | +450 / +480 | driver hang, then bugcheck 0x133 |
| 1100 mV | 2,947 | **3,382** | none | +435 / - | voltage ceiling (ran 3,322 to 3,352 MHz, 0 errors) |

- The card holds at most 1.115 to 1.120 V under any load with the voltage slider at +100 %; points at 1,150 and
  1,200 mV are never used.
- Isolated re-tests (offset raised only at the voltage under test) failed at the same offsets, so the edges are
  real silicon limits, not an artefact of a uniform offset.

Data: [data/vf-results.json](data/vf-results.json), [data/vf-steps.csv](data/vf-steps.csv).

**A curve from these limits did not survive 3D.** Candidate 1 (each point at its compute limit minus 45 MHz) crashed
Time Spy Extreme at once. Bisecting by voltage region in Time Spy showed the 3D limits 30 to 75 MHz under the
compute limits at 1000 to 1050 mV (1000 mV: +480 in compute, +405 in 3D). The history of every candidate is in
[data/curve-history.csv](data/curve-history.csv); candidate 1 is in [data/curve-candidate-1.json](data/curve-candidate-1.json).

## 2. Per-voltage search, 3D (measured)

The same method with FurMark 2 (Vulkan, artifact scanner on) as the load, the offset only at and above the voltage
under test (+300 below), the live voltage read from Afterburner.

**A heavy steady load overstates the limit.** Under a full FurMark load the core is hot, and the driver drops
temperature bins of about 15 MHz each, so a step locked at 3,150 MHz really ran at 3,060 to 3,100 MHz. FurMark passed
1000 mV at +435 while Time Spy crashed at +420 with 3,150 to 3,172 MHz real. Fix: shrink the FurMark window
(1280x720 down to 320x180: lighter load, cooler core) until the real clock is within 50 MHz of the lock, and record
the real clock. Limits are kept in real MHz.

**Temperature bins move the whole curve.** A cold core runs one or two bins (15 to 30 MHz) higher at every voltage
than a warm one, with the same Afterburner offsets. Offsets written into the Afterburner profile file are applied as
written at any temperature; the driver's bins come on top. (Afterburner's curve editor works on the clocks shown at
the current temperature, so a curve dragged into shape warm gives different offsets from one dragged cold.)

| Voltage | Highest pass (lock) | Real clock | First fail |
|---|---|---|---|
| 850 mV | 1,830 | 1,785 to 1,800 | 1,845 (settings lost after a reset) |
| 875 mV | 2,167 | 2,115 to 2,122 | 2,182 (5 resets, GPU hung) |
| 900 mV | 2,377 | 2,332 | not reached at 2,392 |
| 950 mV | 3,015 | 2,902 to 2,910 (hot) | 3,030 driver reset, 3,045 PC went down |
| 1000 mV | 3,165 | 3,112 to 3,120 | not reached at 3,180 |
| 1025 mV | 3,202 | 3,150 to 3,157 | not reached at 3,217 |
| 1050 mV | 3,285 | 3,232 to 3,240 | 3,300 (driver reset, FurMark hung) |
| 1075 mV | 3,292 | 3,247 | not reached at 3,307 |
| 1100 mV | 3,322 | 3,270 to 3,277 | not reached at 3,337 |

Data: [data/vf-3d.json](data/vf-3d.json).

## 3. The profiles (measured)

**Profile 1, the max curve:** each point at the highest real clock proven in 3D, then bisected in Time Spy, linear in
between, flat at the top so the curve caps itself. Memory +3000, 700 W (Afterburner 121 %).

| Voltage | 850 | 875 | 900 | 950 | 1000 | 1025 | 1050 | 1075 and up |
|---|---|---|---|---|---|---|---|---|
| Clock (MHz) | 1,800 | 2,122 | 2,332 | 3,015 | 3,120 | 3,202 | 3,240 | 3,247 flat |

| Validation | Result |
|---|---|
| Time Spy Extreme, warm, 600 W, three runs in a row | graphics 27,974 / 27,935 / 27,951 |
| Time Spy Extreme, cold (water 25 C), cap 3,232 | graphics 28,053 |
| Time Spy Extreme, 700 W override | graphics **28,223** (best on this curve) |
| Steel Nomad Stress Test, 20 loops, 600 W | **99.2 %**, best loop **15,784**, worst 15,652 |
| Steel Nomad Stress Test, water 43 to 45 C (candidate 7) | 98.7 %, best loop 15,726 |
| Cold Time Spy Extreme GT2 | hangs at random (section 4) |

For comparison (source): an ASUS Astral RTX 5090 run on air posted on Reddit scored 28,604 Time Spy Extreme graphics
at a steady 3,127 to 3,135 MHz with memory +3000, about 2 % above ours at similar clocks; we did not find why (PCIe
5.0 x16, Resizable BAR on). 600 W RTX 5090s in public results score about 15,250 to 15,950 in Steel Nomad:

- ComputerBase, Steel Nomad results: https://www.computerbase.de/artikel/grafikkarten/3dmark-steel-nomad-benchmark.88643
- Hardwareluxx Steel Nomad leaderboard: https://www.hardwareluxx.de/community/threads/3dmark-steel-nomad-rangliste.1353763

**Profile 2 (in progress):** Profile 1 flattened at about 3,090 MHz, so heavy scenes stay under the hang zone; the
test is 16 cold GT2 runs in a row at 700 W, then 3,075 and 3,060 if it fails. Not finished at the time of writing.

Full curve: [data/profile1-curve.json](data/profile1-curve.json).

## 4. The Time Spy Extreme GT2 hang (measured)

### What it looks like

Every hang has the same shape in a 100 ms Afterburner log:

1. A heavy scene: last heavy sample 2,990 to 3,150 MHz at 985 to 1,050 mV, 410 to 600 W, core 38 to 49 C.
2. Within one sample (100 ms), power collapses to 120 to 155 W while usage stays at 98 to 100 %. The GPU has
   stopped mid-frame.
3. Because power fell, the clock jumps to the top of the curve (3,120 to 3,270 MHz).
4. A driver reset (event 4101; nvlddmkm event 14 "PF FLR"), or a full Windows freeze that needs a power cycle.

The clock jump in step 3 is a symptom, not the cause. We first read it as a light-load boost crash at the top of the
curve and lowered the top and the cap for hours; that chased the symptom. Passing runs show no such dip at the same
moment.

The hangs come 27 to 121 s after the test starts, at random, warm or cold, more often cold. 27 logged hang
signatures are in [data/hang-events.csv](data/hang-events.csv) (found with the autopilot's `hangscan`; three
flagged lines are our own CUDA ramp test, marked as not a hang).

### What changes it, and what does not

| Change | Hangs | Runs |
|---|---|---|
| Profile 1, 600 W, warm, full 3DMark app runs | 0 | 3 + the Steel Nomad stress test |
| Profile 1, 600 W, cold (water 22 to 27 C), app and harness runs | 4 | 6 (both passes had a 3,232 cap) |
| 1000 and 1025 mV one or two notches (15 or 30 MHz) down | 1 each | 2 each |
| Whole curve minus 30 MHz | probably 1 (telemetry only) | 8 |
| Memory +2000 | 1 | 1 |
| Fresh reboot (Profile 1, cap 3,202) | 1 | 1 |
| Cap 3,060 MHz, cold, 600 W | **0** | **8** |
| Cap 3,090 MHz, cold, 600 W | **0** | **5** (stopped by the owner's decision, not a failure) |
| Profile 3, uniform +360 (heavy clocks about 3,030 to 3,060) | 0 | 2 |
| 700 W override, top extended to 3,262 to 3,277 | 3 | 14 |
| 700 W, cap 3,150 | 1 | 1 |
| 700 W, cap 3,135 | 1 | 5 (the hang at 3,045 MHz real) |
| Clock locked min=max at 3,135 from 1025 mV | 1 | 1 (ran about 3,000, not 3,135) |
| 600 W with the override loaded | 1 | 1 |
| Stock VBIOS, 600 W | 1 (hard hang) | 1 |
| Profile 1, 700 W, hang-rate count | **5** | **8** |
| Profile 1, 700 W, MSVDD REL floor 1,100 mV | **4** (one hard freeze) | **9** |

Every run and its setup: [data/tse-trials.csv](data/tse-trials.csv).

Reading (estimate): the hang tracks the real clock in heavy scenes, roughly from 3,000 MHz upward with 985 to 1,050
mV, and it is probabilistic: the same voltages ran clean at higher clocks in other seconds. There is no sharp
per-voltage edge we could find. 700 W raises heavy-scene clocks (less power limiting), so it raises the hang rate.

### Things that do not reproduce it

| Load | Result at a setting that hung Time Spy |
|---|---|
| FurMark, steady | pass |
| FurMark load steps (light vsync FurMark plus heavy FurMark every ~6 s, 40 steps) | pass |
| CUDA load ramp (light, stepped ramp to full, back; 23 cycles) | pass, 0 errors |
| OCCT 3D Adaptive, Variable and Switch (19 + 10 min) | pass, but it only reaches about 3,020 MHz on this card |
| Steel Nomad Stress Test | pass |

So only a real 3D workload that reaches the hang clocks finds it here.

### Direct launches are harsher than app runs

The autopilot's harness starts GT2 directly (`3DMarkTimeSpy.exe --in <workload>`), back to back, with no demo,
loading or CPU test in between. At 700 W, Profile 1 hung in 5 of 8 direct GT2 runs while full app runs passed. We
use the harness to compare settings, not to predict how often a full run or a game fails.

### What we run now

The owner chose the unrestricted Profile 1 for daily use and accepts the cold-start risk. Two safer options, both
tested:

- A static cap: `nvidia-smi -lgc 210,3090` (needs admin; a logon task and a task on event 4101 re-apply it, since
  a reboot or a driver reset clears it).
- A cap that follows the coolant: the [blackout-tray](https://github.com/Xipzer/blackout-tray) `[gpu]` section sends
  `cap_cold` below one water temperature and `cap_warm` above another, every minute.

## 5. 700 W without flashing (measured, with risks)

The FE stops at 600 W. Panchovix's [Vix Editor](https://github.com/Panchovix/nvidia-vbios-vix-editor) edits a copy
of the VBIOS and the [Windows Vix Loader](https://github.com/Panchovix/nvidia-windows-vbios-vix-loader) hands it to
the driver through a registry override (value `vbios` in the GPU's display class key). The flash chip is never
written. Step by step, with the scripts:
[VBIOS.md](https://github.com/Xipzer/blackwell-overclock-autopilot/blob/main/docs/VBIOS.md).

What we did:

1. Saved the full image read-only with `nvflash --save` (1,961,983 bytes). NvAPI_GPU_GetVbiosImage returned only
   458,240 bytes, and the editor rejects that truncated image.
2. Ran the editor's Linux command line offline (no-network namespace), after checking the downloads against the
   release checksums.
3. Edited, then diffed all 136 fields of the stock and edited images: only the intended fields changed (226 to 230
   bytes).
4. Loaded with the Vix Loader (needs a restart), checked the running image byte for byte with the loader's verify,
   and `power.max_limit` read 700.00 W. Unloading and restarting gave 600 W back.

| Edit | Result |
|---|---|
| Power limit 600 -> 700 W | works; Time Spy Extreme graphics 28,223, but Time Spy peaks at about 592 W: the curve and the hang set the score, not power |
| NVVDD and MSVDD REL ceiling 1,110 -> 1,150 mV | no effect. The REL (reliability) voltage is computed per chip; on this card about 1,070 mV, under the old ceiling. The REL floor (`rel_equation_min`) is the field that raises it |
| MSVDD REL floor 900 -> 1,100 mV | 4 hangs in 9 runs vs 5 in 8 without: not a fix (small samples) |

Background (source: the editor's documentation and changelog):

- Driver 621.01 and later ignore NVVDD and MSVDD runtime voltage offsets on RTX 50 cards: the driver reports
  success and nothing changes. Afterburner's voltage slider is a no-op there. The editor's `runtime_voltage_offsets=1`
  adds hooks so a runtime tool can apply offsets again; we did not use it.
- MSVDD feeds the fabric (XBAR and SYS clocks); XBAR runs at about 0.90 x the core clock. NVIDIA lowered the MSVDD
  REL by 50 mV from driver 572.60. That is why the MSVDD floor seemed worth testing.

**Connector risk.** The 12V-2x6 connector carries 12 V on six pins, each rated 9.2 to 9.5 A depending on the source.
At 600 W that is 8.3 A per pin, 88 % of a 9.5 A rating (estimate, matching the sources). At 700 W it is between
8.7 A per pin (if the PCIe slot supplies its full 75 W) and 9.7 A per pin (if it all comes through the connector),
at or over the rating, with no margin for an uneven split between pins. Our Time Spy peak of about 592 W is about
8.2 A per pin. Sources:

- HWCooling, why 12V-2x6 connectors fail (9.5 A per contact, 88 % at 600 W):
  https://www.hwcooling.net/en/uneven-resistance-and-lack-of-headroom-why-12v-2x6s-fail
- Wikipedia, 12VHPWR and 12V-2x6 (9.5 A per pin, safety factor 1.14 at 600 W):
  https://en.wikipedia.org/wiki/16-pin_12VHPWR_connector
- VideoCardz, per-pin currents on an RTX 5090 at 575 W (7.8 to 8.5 A, against a 9.2 A specification):
  https://videocardz.com/newz/rtx-5090-12v-2x6-connector-reaches-91-7c-with-dlss-5-16-pin-input-peaks-at-647w

Make sure the plug is fully seated, do not bend the cable near the plug, and watch connector temperature or per-pin
current if you can. Use at your own risk.

Data: [data/vbios-override.json](data/vbios-override.json).

## 6. Practical gotchas (measured)

- **BitLocker:** after some restarts the second drive, with the 3DMark library, stayed locked. Every unattended
  launch then "failed" without running. Check the test can start before counting results.
- **The DXGI adapter LUID changes every boot.** 3DMark workload files name the GPU by it; a stale one makes 3DMark
  refuse the launch, which looked like a hang on the first post-reboot test.
- **Displays must stay awake.** 3DMark aborts with "window_focus_lost: Display resolution changed" if a display
  sleeps. Dim them over DDC instead.
- **`nvidia-smi -lgc F,F` (min = max) does not hold F on Blackwell** in heavy 3D: the card ran about 100 MHz under it.
  Use `-lgc 210,CAP` as a ceiling only.
- **A driver reset drops the curve, the memory offset and the clock cap.** A search step after a reset runs at stock
  and looks like a pass; we now treat a voltage far over the target as "settings lost".
- **Log at 100 ms.** nvidia-smi at 1 s and without voltage missed the event; Afterburner's shared memory at 100 ms,
  flushed per line, survives a hard hang.
- **`pkill -f PATTERN` can kill your own shell** when the pattern is also in that shell's command line.
- **Check the GPU is idle before any test.** An unattended test once collided with a benchmark the owner was running.
- **Crash-once-per-boot theory** (source: Igor's Lab reports the Blackwell boost table can come back wrong after a
  driver reset until a reboot): tested, a fresh boot hung on run 1 too.
- **GPU tuning crashes do not corrupt data** beyond files written in the last minute or two, like a power cut.
  chkdsk, sfc and a WSL disk check after the first crashes were all clean.

## Open questions

1. What in GT2's heavy scene hangs the GPU at about 3,000 to 3,150 MHz when nothing else we ran does? Is it a
   specific unit (RT, async compute, the fabric) or the load profile?
2. Why is the rate higher with a cold core? Our guess: one or two extra temperature bins push heavy-scene clocks into
   the zone; but the hangs also came at 46 to 49 C.
3. Would a higher NVVDD REL floor help? We raised only the MSVDD floor.
4. After one reboot, direct GT2 launches ran at about 90 fps instead of 149, capped or not, then later went back to
   149. We did not find out why; the hang results look the same in both states.
5. Does the hang happen in games? Not known yet; the plan is to watch nvlddmkm and 4101 events for one to two weeks.
6. Profile 2 (flattened at about 3,090 MHz) result: pending.

## Lessons

1. **Check your tool before you believe a ceiling.** Our first pass called every step "not reached": the reported
   clock sits 15 to 23 MHz under any lock and we required 8.
2. **Compute limits are not 3D limits.** Here 30 to 110 MHz apart.
3. **Record real clocks, not offsets.** Temperature bins move the real clock by 15 to 50 MHz.
4. **A heavy steady load hides the limit.** It heats the core and lowers the clock it tests.
5. **Read the 100 ms log before changing anything.** The clock jump before each reset was a symptom.
6. **Run the controlled test first.** Same setting, same condition, N runs, before changing the curve.
7. **Count runs, not single passes.** A hang rate of 1 in 8 passes most single runs.
8. **Cold and warm are different tests.** Validate at the coldest condition the card will see.
9. **One change at a time, and write every result down** with water temperature, driver state and VBIOS.
10. **More power is not more performance** when something else (here, the hang) sets the limit.

## Data

| File | What it holds |
|---|---|
| [data/vf-results.json](data/vf-results.json), [data/vf-steps.csv](data/vf-steps.csv) | compute per-voltage search |
| [data/vf-3d.json](data/vf-3d.json) | FurMark per-voltage search, lock and real clocks |
| [data/curve-history.csv](data/curve-history.csv) | every curve candidate and its Time Spy or Steel Nomad result |
| [data/curve-candidate-1.json](data/curve-candidate-1.json) | the first curve (compute limits minus 45 MHz), historical |
| [data/profile1-curve.json](data/profile1-curve.json) | the max curve, offset per point |
| [data/tse-trials.csv](data/tse-trials.csv) | every automated GT2 run: setup, power, VBIOS, cap, result |
| [data/hang-events.csv](data/hang-events.csv) | hang signatures found in the 100 ms telemetry |
| [data/vbios-override.json](data/vbios-override.json) | VBIOS fields changed and what each did |

Times are UTC.

## Licence

Text, tables and figures: CC BY 4.0. See [LICENSE.md](LICENSE.md).
