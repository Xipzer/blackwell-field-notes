# Blackwell field notes

Field notes from tuning one water-cooled RTX 5090: what we tried, what broke, the numbers, and what we learned.

**Status: search done, curve under validation.** The per-voltage search is finished (900 to 1125 mV). A curve
built from it scores 15,678 in Steel Nomad. Its stress test and a hot-loop test (45 C water) are still pending.

Test your own card. Do not copy our numbers. Every chip, board and loop is different.

The tool used here is [blackwell-overclock-autopilot](https://github.com/Xipzer/blackwell-overclock-autopilot).
The CPU and RAM side of the same machine is in [raptor-lake-field-notes](https://github.com/Xipzer/raptor-lake-field-notes).

Labels:

- **Measured**: read from a log or a benchmark run on this machine.
- **Source**: a public source says so (link given).
- **Estimate**: our own arithmetic or reasoning.

## The machine

| | |
|---|---|
| GPU | RTX 5090, 600 W power limit, PCIe 5.0 x16, water-cooled in a custom loop shared with the CPU |
| CPU / board | i9-14900KS, ASRock Z790I Lightning WiFi (see raptor-lake-field-notes) |
| OS / driver | Windows 11, NVIDIA driver 621.24 |
| Tools | MSI Afterburner 4.6.7 Beta 2, nvidia-smi, 3DMark Steel Nomad, our own CUDA check (gpucheck) |
| Cooling | GPU die about 18 to 20 C over water at 600 to 700 W (owner's figure) |

## Method, short

Full version: [METHOD.md](https://github.com/Xipzer/blackwell-overclock-autopilot/blob/main/docs/METHOD.md).

1. Read the factory V/F curve from an Afterburner profile.
2. Write an offset to a spare profile slot and apply it.
3. Pin the clock with `nvidia-smi -lgc F,F`, so the driver runs exactly the point under test (voltage V, clock F).
4. Run a CUDA FP32 checksum check with pulsed load for 60 s. Pass = 0 errors, no driver reset, and the card
   actually ran the lock.
5. Raise F in 30 MHz steps, then 15 MHz near the edge, until the first failure. One failure per voltage.
6. Read the live voltage from Afterburner, to know which point really ran.

## Per-voltage limits (measured)

Clocks are the lock targets. The clock the card reports sits 15 to 23 MHz under any lock.
900 to 1050 mV: heavy load (3 blocks per SM). 1100 mV and up: light load (1 block per SM), because the card only
runs its top voltages at light load.

| Voltage | Factory clock | Highest clean | First fail | Offset clean / fail | How it failed |
|---|---|---|---|---|---|
| 900 mV | 2,002 | **2,497** | 2,512 | +495 / +510 | PC went down |
| 950 mV | 2,535 | **3,030** | 3,045 | +495 / +510 | PC went down; isolated re-test: 5 driver resets, recovered |
| 1000 mV | 2,715 | **3,195** | 3,225 | +480 / +510 | PC went down (0x133); isolated re-test: PC went down again |
| 1050 mV | 2,850 | **3,300** | 3,330 | +450 / +480 | driver hang, then bugcheck 0x133 |
| 1100 mV | 2,947 | **3,382** | none | +435 / - | 3,397 not reached: voltage ceiling (ran 3,322 to 3,352 MHz, 0 errors) |
| 1125 mV | 2,992 | - | none | - | 3,412 not reached: median 1,115 mV |
| 1150, 1200 mV | 3,037, 3,120 | - | none | - | never used: the card holds at most 1.120 V |

Data: [data/vf-results.json](data/vf-results.json) (summary) and [data/vf-steps.csv](data/vf-steps.csv)
(every step: lock, offset, result, errors, live clock, voltage, power, driver resets).

Correction: an earlier version of this page and of the data had 3,015 MHz at 950 mV and 2,482 at 900 mV. Those came
from a crash-resume bug. The right values are 3,030 and 2,497.

What the table says:

- **The edges at 900 to 1000 mV all came at +510.** That looked like a whole-curve effect (the same offset on every
  point), not a per-voltage limit. So we re-tested with the offset raised only at the voltage under test and +300
  below it. 1000 mV failed again at +510 (PC went down) and 950 mV failed again at +510 (5 driver resets, recovered
  without a reboot). **The edges are real silicon limits**, not an artefact of the uniform offset.
- **The offset headroom is about the same at every voltage** (+435 to +495). Estimate: the factory curve already
  follows the silicon's own slope. Per point the driver would allow up to +1000.
- **The top of the curve is voltage limited, not stability limited.** With the voltage slider at +100 % the card
  holds at most 1.115 to 1.120 V under any load (1.125 V only in near-idle moments). Points above that are never
  used. At the top it ran 3,315 to 3,352 MHz with 0 errors and never crashed.
- **The old transient crash at about 3,200 MHz did not come back** (see below). Estimate: the card now runs cooler
  on water than when that crash was seen.

## Crashes

| Ending | Count | Notes |
|---|---|---|
| Driver hang, then DPC_WATCHDOG_VIOLATION (0x133) bugcheck, PC restarts | most limit steps | the usual ending |
| Driver resets, recovered without a reboot | 1 | 950 mV isolated re-test, 5 resets |

GPU tuning crashes do not corrupt data. A GPU fault does not touch RAM or disk contents; the exposure is the same as
a power cut, which NTFS journaling handles. We ran `chkdsk /scan`, `sfc` and a WSL disk check after the first crash:
all clean.

One caveat (measured): files written in the last minute or two before a crash can be lost, the same as with a power
cut. Save and close what you edit before a step, and check files you wrote just before a crash.

## Curve candidate 1 (measured setup, validation pending)

Rule: each point gets its clean limit minus 45 MHz (3 bins of 15 MHz), interpolated between the tested points.

| Voltage | Offset | Clock |
|---|---|---|
| below 880 mV | +300 | untested, kept at the proven daily |
| 900 mV | +450 | 2,452 |
| 950 mV | +450 | 2,985 |
| 1000 mV | +435 | 3,150 |
| 1050 mV | +405 | 3,255 |
| 1100 mV | +390 | 3,337 |
| 1125 mV | +390 | 3,382 |

Plus: memory +3000, power limit 104 %, clock cap 3,330 MHz with `nvidia-smi -lgc 210,3330`.
Full table: [data/curve-candidate-1.json](data/curve-candidate-1.json).

| Validation | State |
|---|---|
| Steel Nomad | done: 15,678 |
| Steel Nomad Stress Test | pending |
| Hot loop (45 C water) | pending |

## Steel Nomad (measured)

Same 600 W power limit in every run.

| Setting | Score | Median core clock | Notes |
|---|---|---|---|
| +180 core / +3000 memory | 15,280 | 2,871 MHz | the starting point |
| +300 core / +3000 memory | 15,429 | 2,962 MHz | +3 % clock gave about +1 % score |
| Curve candidate 1 | **15,678** | 3,052 MHz | max 3,150 MHz, 47 C; +2.6 % over the start |
| Stress Test, 20 loops (earlier uniform offset) | 98.9 % frame rate stability (pass) | | max core 51 C |

How that compares (source): 600 W RTX 5090s in public results score about 15,250 to 15,950. A published full
overclock of the same card class scored 15,635:

- overclock.net, stock vs undervolt vs overclock on an RTX 5090:
  https://www.overclock.net/threads/rtx-5090-msi-vanguard-soc-launch-edition-comparison-stock-vs-undervolt-vs-overclock-4090-comparison-on-4-synthetics-and-3-games.1815041
- ComputerBase, Steel Nomad results: https://www.computerbase.de/artikel/grafikkarten/3dmark-steel-nomad-benchmark.88643
- Hardwareluxx Steel Nomad leaderboard: https://www.hardwareluxx.de/community/threads/3dmark-steel-nomad-rangliste.1353763

Reading (estimate): Steel Nomad at this level is bound by memory bandwidth and the 600 W limit more than by core
clock, so a per-voltage curve gains a few percent, not more. It puts this card just above that published overclock.

## The transient boost crash, and the fix

What happened (measured, earlier): with a large uniform offset (+300), the card was stable in games and benchmarks,
but crashed in quiet moments: menus, alt-tab, loading screens. At light load the boost jumps straight to the top of
the curve, about 3,200 MHz then, a point that no heavy load ever runs and nothing had tested.

The fix: cap the top of the clock range and keep the bottom free.

```
nvidia-smi -lgc 210,3330
```

- Needs administrator rights.
- Does not survive a reboot. Add a logon task (the tool's `tools/install-tasks.ps1 -CapMax 3330` does this).
- `nvidia-smi -rgc` removes it.

In the per-voltage search the top of the curve ran 3,315 to 3,352 MHz with 0 errors and never crashed, so the cap
now sits at 3,330 (it was 3,180).

A matching public report of RTX 5090 hangs and TDR failures at light load:
https://rog-forum.asus.com/t5/gaming-notebooks/rtx-5090-laptop-gpu-hang-and-video-tdr-failure-regression/td-p/1157396/page/2

## Lessons

1. **Check your tool before you believe a ceiling.** Our first pass reported "not reached" almost everywhere.
   The cause: we required the reported clock within 8 MHz of the lock, but it always sits 15 to 23 MHz under.
   With a 30 MHz tolerance the same steps were plain passes.
2. **Record the last pass before each step.** Our first crash-resume code guessed the last pass after a crash and
   recorded 3,015 MHz at 950 mV when 3,030 had passed. The tool now stores the last pass in the step marker.
3. **Start each voltage near its extrapolated edge**, and use 15 MHz steps near it. Each edge costs a crash.
   60 s per step was enough: every failure came within the step.
4. **Raise only the point under test to isolate it.** A uniform offset can make several voltages fail at the same
   offset. Keep the rest of the curve at a safe offset and re-test.
5. **Measure the real top voltage before searching above it.** Our card never goes past 1.120 V under load, so
   points at 1,150 and 1,200 mV are never used, and searching them wastes time.
6. **Read the live voltage.** nvidia-smi does not report it. Afterburner's shared memory (MAHMSharedMemory) does,
   but only after enabling the voltage graph in Settings, Monitoring.
7. **Afterburner details that cost time:**
   - its Profiles folder is under Program Files and needs admin rights to write;
   - `MSIAfterburner.exe -Profile5` applies a slot to the running instance;
   - the V/F curve in the profile is hex: a 12-byte header with the point count, then per point three floats
     (voltage mV, base MHz, offset MHz), then zero padding.
8. **A pass in a compute check is not the whole story.** Validate the final curve in games, a stress test and a
   hot loop. Ours still has the last two pending.

## Licence

Text, tables and figures: CC BY 4.0. See [LICENSE.md](LICENSE.md).
