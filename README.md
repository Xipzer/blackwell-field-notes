# Blackwell field notes

Field notes from tuning one water-cooled RTX 5090: what we tried, what broke, the numbers, and what we learned.

**Status: IN PROGRESS.** The per-voltage search is still running. More voltage points (900 mV, and the top of
the curve at 1100 mV and up under light load) are being measured, and this page will be updated.

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
2. Write one offset to every point of a spare profile slot and apply it.
3. Pin the clock with `nvidia-smi -lgc F,F`, so the driver runs exactly the point under test (voltage V, clock F).
4. Run a CUDA FP32 checksum check with pulsed load for 60 to 90 s. Pass = 0 errors, no driver reset, and the
   card actually ran the lock.
5. Raise F in 30 MHz steps, then 15 MHz near the edge, until the first failure. One failure per voltage.

## Results so far (measured, heavy load)

Clocks are the lock targets. The clock the card reports sits 15 to 23 MHz under any lock.

| Voltage | Factory clock | Highest pass | First fail | Offset at pass / fail | How it failed |
|---|---|---|---|---|---|
| 1090 mV | 2,925 | 3,225 | none | +300 / - | 3,255 not reached: the card would not hold 1090 mV under heavy load |
| 1050 mV | 2,850 | **3,300** | 3,330 | +450 / +480 | driver hang, then bugcheck 0x133 |
| 1000 mV | 2,715 | **3,195** | 3,225 | +480 / +510 | PC went down (DPC watchdog bugcheck) |
| 950 mV | 2,535 | **3,030** | 3,045 | +495 / +510 | PC went down |
| 900 mV | 2,002 | 2,497 so far | in progress | +495 / - | in progress |
| 1100 mV and up | | | | | planned, light-load search |

Step-by-step log: [data/vf-results.json](data/vf-results.json).

What the table says:

- **The offset headroom is about the same at every voltage** (+450 to +510), while the clocks are about 105 to
  165 MHz apart per 50 mV. Estimate: the factory curve already follows the silicon's own slope, so one good
  offset fits most of the curve. Per point the driver would allow up to +1000.
- **The edge offset rose about +30 MHz for each 50 mV lower.** Useful to start the next voltage close to its edge.
- **Every limit ended in a hard crash**: a driver hang followed by a DPC_WATCHDOG_VIOLATION (0x133) blue screen,
  never a clean "driver stopped responding and has recovered".
- **Heavy load hides the top of the curve.** Under heavy compute the card held its voltage near 1.05 V (power,
  current and reliability limits). At light load it runs the top of the curve, to 1.12 V and higher (owner
  screenshot: 3,165 MHz at 26 % load with +180). So the top points need their own light-load test.

## Steel Nomad (measured)

| Setting | Score | Average clock | Notes |
|---|---|---|---|
| +180 core / +3000 memory | 15,280 | 2,865 MHz | power-capped at 600 W; PCIe 5.0 x16; the iGPU was not involved |
| +300 core / +3000 memory | 15,429 | 2,962 MHz | +3 % clock gave only about +1 % score |
| Steel Nomad Stress Test, 20 loops | 98.9 % frame rate stability (pass) | | max core 51 C |

Reading (estimate): Steel Nomad at this level is bound by memory bandwidth and the 600 W power limit more than by
core clock. More core offset buys little here.

How that compares (source): 600 W RTX 5090s in public results score about 15,250 to 15,950.

- overclock.net, stock vs undervolt vs overclock on an RTX 5090:
  https://www.overclock.net/threads/rtx-5090-msi-vanguard-soc-launch-edition-comparison-stock-vs-undervolt-vs-overclock-4090-comparison-on-4-synthetics-and-3-games.1815041
- ComputerBase, Steel Nomad results: https://www.computerbase.de/artikel/grafikkarten/3dmark-steel-nomad-benchmark.88643
- Hardwareluxx Steel Nomad leaderboard: https://www.hardwareluxx.de/community/threads/3dmark-steel-nomad-rangliste.1353763

So this card sits in the normal range for a 600 W 5090, not at the top of it.

## The transient boost crash, and the fix

What happened (measured): with a large offset (+300), the card was stable in games and benchmarks, but crashed in
quiet moments: menus, alt-tab, loading screens. At light load the boost jumps straight to the top of the curve,
about 3,200 MHz, a point that no heavy load ever runs and nothing had tested. The driver crashed there.

The fix: cap the top of the clock range and keep the bottom free.

```
nvidia-smi -lgc 210,3180
```

- Needs administrator rights.
- Does not survive a reboot. Add a logon task (the tool's `tools/install-tasks.ps1 -CapMax 3180` does this).
- `nvidia-smi -rgc` removes it.

A matching public report of RTX 5090 hangs and TDR failures at light load:
https://rog-forum.asus.com/t5/gaming-notebooks/rtx-5090-laptop-gpu-hang-and-video-tdr-failure-regression/td-p/1157396/page/2

## Lessons

1. **Check your tool before you believe a ceiling.** Our first pass reported "not reached" almost everywhere.
   The cause: we required the reported clock within 8 MHz of the lock, but it always sits 15 to 23 MHz under.
   With a 30 MHz tolerance the same steps were plain passes.
2. **Record the last pass before each step.** Our first crash-resume code guessed the last pass after a crash and
   recorded 3,015 MHz at 950 mV when 3,030 had passed. The tool now stores the last pass in the step marker.
3. **Start near the edge.** Each edge costs a blue screen and a reboot. Use the slope from the voltages already done.
4. **GPU tuning crashes do not corrupt data.** A GPU fault does not touch RAM or disk contents; the exposure is
   the same as a power cut, which NTFS journaling handles. We ran `chkdsk /scan`, `sfc` and a WSL disk check
   after the first crash: all clean. We stopped checking after every crash.
5. **Afterburner details that cost time:**
   - its Profiles folder is under Program Files and needs admin rights to write;
   - `MSIAfterburner.exe -Profile5` applies a slot to the running instance;
   - the core voltage shows in its shared memory (MAHMSharedMemory) only after enabling the voltage graph in
     Settings, Monitoring; nvidia-smi does not report voltage at all;
   - the V/F curve in the profile is hex: a 12-byte header with the point count, then per point three floats
     (voltage mV, base MHz, offset MHz), then zero padding.
6. **A pass in a compute check is not the whole story.** The crash in daily use came from a point the check never
   ran: the top of the curve at light load.

## Licence

Text, tables and figures: CC BY 4.0. See [LICENSE.md](LICENSE.md).
