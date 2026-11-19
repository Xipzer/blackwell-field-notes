# Blackwell field notes

Field notes from tuning one water-cooled RTX 5090 Founders Edition over three days (17 to 19 Nov 2026): what we
tried, what broke, the numbers, and what we still do not understand.

**Status (18 Nov 2026, 18:30 UTC): done.** Two final profiles are saved and tested in real borderless fullscreen.
The fix for the random Time Spy Extreme GT2 hang was not more margin per voltage but a **soft cap in the curve**:
the max curve with every point up to 1,060 mV held flat at 3,090 MHz (Profile 1) or 3,075 MHz (Profile 2), and the
points above left alone. It costs no fps. Forcing Resizable BAR for 3DMark and raising the crossbar clock added
about 2 % on top. GT2 went from about 153 fps (old harness, window across monitors) to 176.6 fps. Validated Time
Spy Extreme: **24,961 overall, 29,265 graphics** (rank 92 of about 3,270 for the same CPU and GPU, see
[Results](#results)). Summary below, the full story in [section 8](#8-18-nov-evening-the-final-profiles-and-how-we-got-there).

**Update (18 Nov, 19:20 to 20:02 UTC): voltage lever.** Raising the NVVDD REL floor in the VBIOS override image
(one field, 900 -> 1,150 mV) lifted the core voltage limit from 1,120 to 1,170 mV, and a ramp on the top of the
curve now lets light loads run 3,337 to 3,382 MHz instead of about 3,172 to 3,195. Heavy loads never reach that
voltage, so the soft cap is unchanged. Both profiles are saved with the new top; see
[Voltage lever](#voltage-lever). Runtime voltage offsets through mVolt+ did not work on this driver (Q27).

**Update (19 Nov, morning UTC): the soft cap now runs to 1,100 mV.** With the higher voltage limit, heavy loads
also climbed higher: Fire Strike Ultra ran 3,180 to 3,217 MHz at 1.065 to 1.08 V, right on the old curve's step from
3,090 to 3,247 MHz at 1,065 to 1,070 mV. Two driver resets followed in heavy scenes (about 610 to 620 W). The
"heavy loads never reach that voltage" claim from the evening before was wrong (Q28). Fix: every point up to
1,100 mV is now held at the wall (3,090 / 3,075 MHz), then a ramp to 3,440 / 3,380 MHz at 1,150 mV. TSE GT2 3 of 3
clean at 176.1 to 176.6 fps; the 179.0 fps screening result came from the unsafe step and is withdrawn. Validated
Fire Strike Ultra: **34,409 overall, 37,457 graphics** (+1.3 % over the previous best). See
[Heavy loads climb too](#heavy-loads-climb-too-19-nov).

**Update (19 Nov, late morning UTC): final config validated.** Owner 3DMark app runs on the final config (Profile 1
with the 19 Nov shape, REL 1,150 image, XBAR +400, ReBAR forced on all four 3DMark driver profiles): Time Spy
Extreme **25,025 overall, 29,380 graphics** (new best, +0.4 % graphics), Fire Strike Ultra run 2 **34,563 overall,
37,465 graphics** (new best, within 8 graphics points of run 1), Steel Nomad 16,523 (no gain), Steel Nomad Stress
Test **99.3 %**, 20 loops, clean. Heavy loads now draw about 45 W more for the same Steel Nomad score; see
[Results](#results) and the caveat in [Voltage lever](#voltage-lever). OCCT 3D Adaptive on the final config,
Variable 20 min then Switch 20 min: no errors, no driver resets (it peaks at about 3,037 MHz, so it does not reach the
3,090 MHz heavy-load edge; the 3DMark runs cover that).

**Update (19 Nov, afternoon UTC): the overclock is applied without Afterburner.** The curve, the power limit, the
memory offset and the XBAR offset can all be written to the driver natively, through the driver's own
`nvapi64.dll`. A read-only probe matched Afterburner's applied profile on 127 of 127 points, a write test passed
with every value read back and no driver reset, and [blackout-tray](https://github.com/Xipzer/blackout-tray) now
applies Profile 1 and XBAR +400 at startup and after a driver reset (the reset path has not yet been exercised by a
real reset). Afterburner stays installed but no longer starts with Windows. One trap: a native memory offset write
zeroes the whole curve, so the curve has to go last (Q29). These are private driver interfaces, tested on one card
and one driver. See [Without Afterburner](#without-afterburner-19-nov).

## Final profiles

| | Profile 1 (max) | Profile 2 (one step down) |
|---|---|---|
| Curve (since 19 Nov morning) | Profile 1 max curve, every point at or under **1,100 mV** held to **3,090 MHz** | the same, held to **3,075 MHz** |
| Top of the curve (since 19 Nov, [Heavy loads climb too](#heavy-loads-climb-too-19-nov)) | straight ramp from 3,090 MHz at 1,100 mV to **3,440 MHz at 1,150 mV**, flat above: 1,110 mV 3,160, 1,125 mV 3,265, 1,140 mV 3,370, 1,150 and 1,170 mV 3,440 | ramp from 3,075 at 1,100 mV to **3,380 at 1,150 mV**, flat above: 1,110 mV 3,136, 1,125 mV 3,228, 1,140 mV 3,319, 1,150 and 1,170 mV 3,380 |
| Top of the curve, 18 Nov 20:00 (replaced) | cap only up to 1,060 mV, 3,247 MHz at 1,070 mV, ramp to +255 at 1,170 mV (1,150 mV 3,451) | the same with +195 |
| Light-load top (720p GT1) | 19 Nov shape: 1 of 1 clean, max 1.135 V; 18 Nov +255 ramp: 4 of 4 clean, up to 3,360 to 3,382 MHz read | 18 Nov +195 ramp: 4 of 4 clean, up to 3,337 to 3,345 MHz read |
| VBIOS image | 700 W override with the NVVDD REL floor at 1,150 mV (core voltage limit 1,170 mV) | same |
| Stored in (since 19 Nov afternoon, [Without Afterburner](#without-afterburner-19-nov)) | blackout-tray profile 1, applied natively at startup and after a driver reset; also still in Afterburner slot 1 | blackout-tray profile 2; also still in Afterburner slot 2 |
| Power / memory | 700 W (VBIOS override, 121 %), memory +3000 | same |
| GT2 runs (borderless fullscreen) | 6 clean, 1 hang in 7: **at the edge** | **6 of 6 clean** |
| GT2 fps, XBAR +300, ReBAR default | 172.8 to 173.4 | 171.9 to 173.4 |
| GT2 fps, ReBAR forced, XBAR +300 | 175.4, 175.9, 175.9 (3 of 3 clean) | not run |
| GT2 fps, ReBAR forced, XBAR +400 / +500 | 176.6 / 176.4 to 176.9 (1 and 3 runs, clean) | not run |
| GT2 fps, 19 Nov shape (cap to 1,100 mV + top ramp), ReBAR forced, XBAR +400 | **176.6, 176.1, 176.2** (3 of 3 clean, max 1.085 V) | not run |
| Fire Strike Ultra, 3DMark app (19 Nov) | run 1 34,409 overall, 37,457 graphics; run 2 **34,563 overall, 37,465 graphics** (best), both valid | not run |
| Time Spy Extreme, 3DMark app (19 Nov) | **25,025 overall, 29,380 graphics**, valid, GT2 held 3,090 MHz | not run |
| Steel Nomad, 3DMark app (19 Nov) | 16,523 (165.24 fps), valid | not run |
| Steel Nomad Stress Test (19 Nov) | **99.3 %**, 20 loops, best 16,539, worst 16,426, clean | not run |

Applies to both:

- **Test state**: Time Spy Extreme GT2 direct launch in true borderless fullscreen (`is_windowed: false`,
  `is_borderless: true`, swapchain 1920x1080). About 172 to 173 fps there against about 141 in the fallback window
  (Q1). Nothing measured in the fallback window carries over.
- **Resizable BAR forced for 3DMark** with NVIDIA Profile Inspector on the driver's "3DMark TimeSpy" profile:
  `rBAR - Enable` (0x000F00BA) = 1, size limit left empty. 173.0 and 173.5 fps before, 175.3 and 175.5 forced,
  173.1 with it off again: +1.2 % (Q23). Also forced for Fire Strike, Port Royal and Steel Nomad on their own
  predefined driver profiles (Q23).
- **Crossbar clock (XBAR) +400**. An Afterburner profile does not store the offset and every driver reset drops it
  (Q10, Q17). Until 19 Nov a logon task sent it (`MSIAfterburner.exe /wpXBARCLK_off,400`); since then blackout-tray
  writes it natively together with the profile, at startup and after a driver reset (Q29).
- **No tuning app at startup (since 19 Nov afternoon)**: blackout-tray applies the power limit, memory offset, XBAR
  offset and curve itself. Afterburner is installed but not started with Windows; we start it by hand as a fallback
  and for monitoring during tuning sessions.
- **6 clean runs prove little**: they only rule out hang rates above about 39 % (Q19). Profile 1 hung once in 7, so
  it sits on the edge; Profile 2 is the one to fall back to if anything hangs in daily use.

Data: [data/final-profiles.json](data/final-profiles.json) (clock per voltage point for both),
[data/tse-trials.csv](data/tse-trials.csv) (every run).

Test your own card. Do not copy our numbers. Every chip, board and loop is different.

## Results

Validated 3DMark Time Spy Extreme, official app run with a valid result (green check), 18 Nov 2026, Profile 1
(measured):

| | Score |
|---|---|
| Overall | **24,961** |
| Graphics | **29,265** |
| CPU | 13,615 |
| 3DMark average for the same CPU and GPU | 22,702 (overall) |

Setup for that run: Profile 1 (the original max curve soft-capped flat at 3,090 MHz for every point at or under
1,060 mV, 3,247 MHz above), XBAR +400 (a logon task at the time), Resizable BAR forced on the driver's 3DMark TimeSpy profile,
700 W through the runtime VBIOS override (no flash), memory +3000, all monitors on the 5090, iGPU disabled, CPU 57x
all-core at a fixed 1.320 V and DDR5-8400 with tuned timings (the daily settings from raptor-lake-field-notes). It is
the daily profile: water-cooled, no hardware mods, no chilled water.

Progression (measured):

| | Before | After |
|---|---|---|
| TSE graphics score | 28,223 (old max curve, hangs at random in GT2) | 29,265 (soft-capped curve, stable): **+3.7 %** |
| GT2 fps, our harness | about 153 (old harness: a window across monitors, BenQ on the iGPU) | 176.6 (borderless fullscreen, all monitors on the 5090, ReBAR forced, XBAR +400) |

The fps step is not a clean like-for-like gain: most of it is the harness and display fixes (Q1, Q2), the rest is
ReBAR (+1.2 %) and XBAR. The score step is like-for-like (both are app runs).

Steel Nomad Stress Test, official app run, 18 Nov about 19:10 UTC, Profile 1 (cap 3,090) before the voltage lever,
XBAR +400, the 700 W image with `runtime_voltage_offsets=1` (measured): **passed, 99.1 % frame rate stability**,
20 loops, best loop 16,645, worst 16,492.

After the voltage lever (measured, our harness, borderless fullscreen):

| | Before | After |
|---|---|---|
| Core voltage limit (mVolt+ `--status`, `nvvdd_max`) | 1,120 mV | 1,170 mV |
| Light load (TSE GT1 at 720p, about 250 W): clock read | 3,172 to 3,195 MHz at 1.055 to 1.06 V | 3,360 to 3,382 MHz at up to 1.135 V (Profile 1, top +255) |
| TSE GT2 fps, Profile 1, ReBAR forced, XBAR +400 | 176.6 | 176.6, 176.1, 176.2 (19 Nov shape, 3 of 3 clean) |

The light-load test is CPU bound at 720p (about 275 to 278 fps whatever the clock), so the clock is the metric
there, not fps. The 179.0 fps GT2 run first reported here (18 Nov, 1 run) is **withdrawn**: it ran heavy moments on
the old curve's step above 1,065 mV, the same place two driver resets came from the next morning (Q28). With the
cap extended to 1,100 mV, GT2 is back at 176.1 to 176.6 fps, the same as before the voltage lever: in GT2 the gain
was the unsafe step, not the new top.

Validated 3DMark Fire Strike Ultra, official app run with a valid result, 19 Nov 2026 09:45 UTC, Profile 1 with the
19 Nov shape (cap 3,090 MHz up to 1,100 mV, ramp to 3,440 MHz at 1,150 mV), XBAR +400, ReBAR forced on the driver
profile that holds the Fire Strike exes (Q23) (measured):

| | Score |
|---|---|
| Overall | **34,409** |
| Graphics | **37,457** (previous best 36,975 on 18 Nov with the 3,090 cap and no voltage top: **+1.3 %**) |
| Physics | 51,213 |
| Combined | 16,366 |
| 3DMark average / best for the same hardware | 32,300 / 42,152 (overall) |

Two owner Fire Strike Ultra runs in between do not count: after a driver reset had wiped memory +3000 (memory clock
read 13,801 / 14,001 MHz) and XBAR, one scored 33,184 graphics (31,833 overall); another without XBAR scored
36,538 graphics (33,395 overall). A driver reset silently undoes both (Q17); check them before trusting a score.

### Final config, 19 Nov (measured)

Owner 3DMark app runs, all valid, on the final config: Profile 1 with the 19 Nov shape (cap 3,090 MHz up to
1,100 mV, ramp to 3,440 MHz at 1,150 mV), the REL 1,150 image, XBAR +400, ReBAR forced on all four 3DMark driver
profiles (TimeSpy, Ice Storm, Port Royal DLSS, Steel Nomad), 700 W, memory +3000.

| Test | Result | Before |
|---|---|---|
| Time Spy Extreme | **25,025 overall, 29,380 graphics** (+0.4 %), CPU 13,602; GT2 clock held at 3,090 MHz | 24,961 / 29,265 (18 Nov, before the voltage lever) |
| Fire Strike Ultra run 2 | **34,563 overall, 37,465 graphics**, physics 51,248, combined 16,704 | run 1: 34,409 / 37,457 (repeatable within 8 graphics points) |
| Steel Nomad | 16,523 (165.24 fps) | 18 Nov stress loops 16,492 to 16,645: no gain |
| Steel Nomad Stress Test | **99.3 %**, 20 loops, best 16,539, worst 16,426, clean | 99.1 %, best 16,645, worst 16,492 (18 Nov, before the voltage lever, no ReBAR) |
| OCCT 3D Adaptive (Personal 17.1.12, load type Light) | Variable 20 min, then Switch 20 min: **no errors detected, no driver resets** | earlier run on the old Profile 1: Variable and Switch (19 + 10 min) pass |

Time Spy Extreme and Fire Strike Ultra are new bests. Steel Nomad gains nothing from ReBAR or the voltage top. The
stress test is the long heavy-load reliability pass of the final config. The OCCT run is clean, with a caveat
measured from telemetry: OCCT never drove the core above about 3,037 MHz (Variable) and peaked at about 636 W and
1.10 V, so it does not reach the 3,090 MHz heavy-load edge. That edge is covered by the 3DMark runs above.

### How this compares

3DMark's search for i9-14900KS + RTX 5090, sorted by overall score (about 3,270 results), put the 18 Nov Time
Spy Extreme run (24,961) at **rank 92, about the top 3 %** (measured, 18 Nov; the 19 Nov 25,025 was not re-ranked). The overall score is about 10 % above the 3DMark average for the
same hardware.

Ranks 1 to 7 are extreme-overclocking team entries: GPU core 3,457 to 3,547 MHz, memory 2,250 MHz, CPU 6.2 to
6.4 GHz, overall 27,448 to 29,081, graphics 31,737 to 34,078. These are extreme-overclocking setups (XOC firmware,
external voltage control, chilled water) built for single benchmark passes. Their graphics scores are 8 to 16 %
above ours; ours comes from an unmodified card on a profile that is stable for daily use. The list is sorted by overall score, which includes the CPU, so the rank by graphics score alone would be
higher; we did not measure it.

The tools used here are in [blackwell-overclock-autopilot](https://github.com/Xipzer/blackwell-overclock-autopilot).
The CPU and RAM side of the same machine is in [raptor-lake-field-notes](https://github.com/Xipzer/raptor-lake-field-notes).

### Hardware and silicon

The setup behind that result, so you can judge how reachable it is (measured, fuse values from the Afterburner
`MAKEREPORT`, Q16):

| | |
|---|---|
| GPU | RTX 5090 Founders Edition |
| GPU silicon | SPEEDO 2109 in a fused range of [2018.94, 2284.87], about 0.34 of the way up (lower third); IDDQ NVVDD 42300, MSVDD 14050; NVVDD Vrel 1,120 mV class |
| VBIOS | stock 98.02.2E.00.03, never flashed; power limit raised to 700 W by a runtime override (Vix Loader) |
| Driver | 621.24 |
| Cooling | custom water loop with three radiators, order pump, radiator, GPU, radiator, CPU, radiator; ambient room air, no chiller |
| Mods | none: no shunt mod, no voltage controller mod, no external voltage control |
| CPU | i9-14900KS, SP101 (P-core quality 101, E-core 86), 57x P / 46x E / ring 50, fixed 1.320 V daily |
| RAM | 2x24 GB DDR5-8400 (Hynix M-die), tuned timings |
| Board | Mini-ITX Z790 |

A GPU binned below the median, plus methodical tuning (a correct fullscreen harness, iGPU removal, a soft-capped
curve, XBAR and ReBAR), reached about the top 3 % for this CPU and GPU pairing. The gap to the top of the list is
hardware mods and sub-ambient cooling, not silicon luck (estimate, from the top entries above).

## Quirks index

Every quirk with what we saw, the remedy and public sources is in [QUIRKS.md](QUIRKS.md). Short version:

| # | Quirk | Remedy | Public reports |
|---|---|---|---|
| [Q1](QUIRKS.md#q1) | Direct 3DMark launch runs in a 3840x2160 window across monitors, not fullscreen; exclusive fullscreen only on the first run after boot | `is_windowed: false` + `is_borderless: true` + `swapchain_resolution` = display size; check window style (`0x94000000` good, `0x14CF0000` windowed) | none found |
| [Q2](QUIRKS.md#q2) | A monitor on the Intel iGPU costs about 5.5 % in GT2 (cross-adapter copy) | all monitors on the 5090; iGPU off in Windows and BIOS (`IGPU Multi-Monitor` = Disabled) | yes (Microsoft, forums) |
| [Q3](QUIRKS.md#q3) | HWiNFO running hid that penalty (fps "states") | same as Q2 | none found |
| [Q4](QUIRKS.md#q4) | "A DSC monitor takes two heads" did not stop 4 monitors (Neo G9 240 Hz DSC + three 1080p) on one 5090 | none needed; BenQ via HDMI-to-DP active adapter | NVIDIA KB (the limit), forum (Blackwell change) |
| [Q5](QUIRKS.md#q5) | Old Intel iGPU driver 31.0.101.5592 crashes HWiNFO in `ze_intel_gpu64.dll`; Windows Update reinstalls it | remove the oem package + `ExcludeWUDriversInQualityUpdate=1`, or HWiNFO `oneAPI=0` | yes (HWiNFO forum) |
| [Q6](QUIRKS.md#q6) | HWiNFO free cannot CSV-log from the command line; sensor indices move between restarts | read shared memory (`SensorsSM=1`), match values by name every poll | yes (HWiNFO, the Pro limit) |
| [Q7](QUIRKS.md#q7) | HWiNFO INI keys (`SensorsSM`, `oneAPI`, `PresentMon`, `NvmlSupport`) | table | |
| [Q8](QUIRKS.md#q8) | Effective, XBAR, SYS clocks and limit flags under GT2 | table | |
| [Q9](QUIRKS.md#q9) | "Performance Limit - Utilization" = GPU not fully busy | use as a load check | yes (HWiNFO author) |
| [Q10](QUIRKS.md#q10) | Afterburner Beta 5 "out of warranty" mode (`OEM=3`), `/wpXBARCLK_off,N`: +300 = +2.6 % in the window; in fullscreen +400 (2,737 MHz) = 176.6 fps, ceiling about 0.9 x core | +400 daily; not stored in a profile, so re-apply after every restart and driver reset (a logon task until 19 Nov, natively since, Q29) | yes (VideoCardz, TweakTown) |
| [Q11](QUIRKS.md#q11) | Global settings live in `Profiles\MSIAfterburner.cfg`; `HwPollPeriod` defaults to 1000 ms | edit that file; 100 ms while logging | |
| [Q12](QUIRKS.md#q12) | Afterburner shared memory goes stale after an Afterburner restart | logger checks signature and timestamp, reopens | none found |
| [Q13](QUIRKS.md#q13) | `[Startup]` rewritten when the UI applies settings | check it after UI use | |
| [Q14](QUIRKS.md#q14) | Voltage slider and `NVVDD_off_Vrel` ignored on driver 621.x | none at runtime here (Q27); raise the NVVDD REL floor in the image instead | yes (Vix Editor) |
| [Q15](QUIRKS.md#q15) | RTSS error popup after installing Afterburner Beta 5 | reinstall RTSS | similar (Guru3D) |
| [Q16](QUIRKS.md#q16) | `MAKEREPORT`: SPEEDO 2109 (about a third into its fuse range), NVVDD Vrel 1,120 mV, MSVDD Vrel 1,070 mV with -50 mV | read-only diagnostics | yes (VideoCardz describes the report) |
| [Q17](QUIRKS.md#q17) | Driver reset wipes curve, memory offset, cap and XBAR offset | re-apply after event 4101 (blackout-tray does it natively since 19 Nov, Q29; not yet exercised by a real reset) | |
| [Q18](QUIRKS.md#q18) | GT2 hangs at a heavy-load clock wall (3,045 to 3,090 MHz) whatever the voltage; per-band margin and `-lgc` caps did not stop it | soft cap in the curve: points up to 1,060 mV flat at 3,090 / 3,075 MHz, top untouched | owner reports of 3.1 to 3.15 GHz game-stable |
| [Q19](QUIRKS.md#q19) | 8 runs prove little: 16 clean = under 17 %, 60 clean = under 5 % | 1 / 3 / 10 to 12 run method | yes (rule of three) |
| [Q20](QUIRKS.md#q20) | Two harnesses at once: about 90 fps, invalid | one chain at a time | |
| [Q21](QUIRKS.md#q21) | CUDA, FurMark, OCCT, Steel Nomad do not find the GT2 limit | test with a real 3D load at the hang clocks | |
| [Q22](QUIRKS.md#q22) | `-lgc F,F` does not hold, LUID changes per boot, displays must stay awake, locked drives | see the page | |
| [Q23](QUIRKS.md#q23) | Resizable BAR is not used by 3DMark Time Spy on its own; forcing it gave +1.2 % in GT2 | NVIDIA Profile Inspector, 3DMark TimeSpy profile, `rBAR - Enable` (0x000F00BA) = 1 | yes (NPI rBAR settings, forums) |
| [Q24](QUIRKS.md#q24) | With one GPU in the system, Afterburner drops the `GPU1 ` prefix from monitoring names (core voltage = `GPU voltage`) | logger looks up both names | none found |
| [Q25](QUIRKS.md#q25) | Per-band offsets can make the curve non-monotonic and pin hangs on the wrong band | cap each point at the lowest clock above it (suffix minimum) | |
| [Q26](QUIRKS.md#q26) | The voltage read at a hang sits about 10 mV under the V/F point in use | attribute hangs near a band edge to the band above | |
| [Q27](QUIRKS.md#q27) | mVolt+ runtime NVVDD offsets on driver 621.24 report success but `rail_offsets_ignored_by_driver: true`, even with a `runtime_voltage_offsets=1` image; the core limit stays 1,120 mV | static route: VBIOS image `nvvdd_rel_equation_min_mv` 900 -> 1,150 (limit 1,170 mV after a restart) | none found |
| [Q28](QUIRKS.md#q28) | Raising the core voltage limit moves heavy loads up the curve too: a clock step that was out of reach (1,065 to 1,070 mV here) becomes reachable and hangs | extend the soft cap to where heavy loads can now go (here 1,100 mV), ramp the top above it | none found |
| [Q29](QUIRKS.md#q29) | Curve, power limit, memory offset and XBAR offset can be applied natively through `nvapi64.dll`, without Afterburner (driver 621.24). A native memory offset write zeroes every curve offset; a memory offset set by Afterburner reads 0 in Pstates20; writes need admin (-137); a zeroed buffer is ignored | order: power, memory, XBAR, curve last, then read back; read, change, write back; private interfaces, keep a tuning app installed as fallback | yes (the open-source projects the layouts come from) |

Before trusting any number, run the [clean measurement checklist](QUIRKS.md#clean-measurement-checklist).

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
| Monitors | Samsung Odyssey Neo G9 5120x1440 240 Hz (DSC), Dell AW2521H 1080p 360 Hz, ASUS XG258Q 1080p 240 Hz, BenQ XL2420Z 1080p 144 Hz; until 18 Nov 15:44 UTC the BenQ was on the Intel iGPU, since then all four are on the 5090 and the iGPU is off |
| Tools | MSI Afterburner 4.6.7 Beta 2 (Beta 5 build 19412 from 18 Nov) with RTSS 7.3.7, HWiNFO64 8.61 (free), PresentMon 2.6.0, nvidia-smi, 3DMark (Time Spy Extreme, Steel Nomad), FurMark 2.11, OCCT 17.1.12, our CUDA checks, nvflash 5.867 (read only), Vix Editor 0.1.45, Vix Loader 0.5.0, NVIDIA Profile Inspector (ReBAR), blackout-tray (applies the overclock natively since 19 Nov; Afterburner is then only started by hand) |
| Cooling | GPU core about 18 to 20 C over water at 600 to 700 W; water 22 to 36 C during the tests |

## Summary: what works and what does not

| Tried | Result |
|---|---|
| Per-voltage search with a CUDA check | works, but finds compute limits 30 to 110 MHz above what 3D survives |
| Per-voltage search with FurMark (3D) | works, but a heavy steady load overstates the limit: the hot core drops temperature bins and never runs the clock it was given |
| Curve from the real clocks proven in 3D (Profile 1) | passes warm: 3 Time Spy Extreme runs, Steel Nomad Stress Test 99.2 % |
| Profile 1 cold (water 22 to 27 C) | hangs at random in Time Spy Extreme GT2 |
| Lowering the 1000 to 1025 mV points by 15 or 30 MHz | still hangs (both on run 2) |
| Whole curve minus 30 MHz | 17 Nov: 7 clean runs, then most likely a hang on run 8; 18 Nov `is_windowed: false`: hang on run 6 of 12 |
| Clock caps 3,202 and 3,232 MHz | no effect: heavy scenes never reach them |
| Clock caps 3,060 and 3,090 MHz (600 W, windowed) | 13 of 13 cold runs clean, but measured windowed (a different load) |
| Clock caps 3,060 and 3,090 MHz (700 W, `is_windowed: false`) | **both hang** (run 9 and run 4 of 12) |
| Whole curve minus 30 MHz (700 W, `is_windowed: false`) | hang on run 6 of 12 |
| Whole curve minus 45 or minus 60 MHz (700 W, `is_windowed: false`) | **12 of 12 clean** each; minus 45 with XBAR +300 also 12 of 12 |
| Per voltage band ("P1 v1": up to 980 mV +15, 981 to 1,005 -30, 1,006 to 1,030 -75, 1,031 to 1,060 -45 vs Profile 1, XBAR +300) | 11 of 11 clean in the window |
| Per voltage band again, in borderless fullscreen (1,006 to 1,030 mV band from -60 down to -120) | **never stopped the hangs**: always at 3,045 to 3,090 MHz, the voltage rising with each step |
| **Soft cap in the curve: points up to 1,060 mV flat at 3,090 / 3,075 MHz, top untouched (borderless fullscreen)** | **6 clean, 1 hang in 7 / 6 of 6 clean**, no fps cost: the final profiles |
| Resizable BAR forced for 3DMark (NVIDIA Profile Inspector) | GT2 +1.2 % (173.3 to 175.4 fps), 3 of 3 clean |
| XBAR +400 / +500 in borderless fullscreen | 176.6 fps either way (+300: 175.7); +500 3 of 3 clean |
| All "fullscreen" runs above | **ran in a 3840x2160 window** after all (Q1); lighter than fullscreen, so margins are lower bounds |
| Crossbar clock +300 MHz (Afterburner out of warranty mode) | XBAR 2,415 to 2,707 MHz, GT2 +2.6 %, no extra hangs |
| Intel iGPU off, all monitors on the 5090 | GT2 about 5.5 % faster in the window test (Q2) |
| Borderless fullscreen harness (`is_borderless: true`, 1920x1080 swapchain) | fullscreen on every run, about 172 fps against 141 in the fallback window |
| Memory +0 instead of +3000 | still hangs (hard freeze, windowed run 2) |
| Clock caps 3,135 and 3,150 MHz (700 W) | hang (run 5 and run 1) |
| Memory +2000 instead of +3000 | still hangs |
| Fresh reboot (driver state reset) | still hangs |
| 700 W VBIOS override | works (power.max_limit 700 W, best Time Spy Extreme graphics 28,223), but Time Spy peaks at about 592 W: power was not the limit |
| Stock VBIOS at 600 W | still hangs (hard hang): the override is not the cause |
| NVVDD and MSVDD REL ceiling 1,110 -> 1,150 mV | no measurable effect (Afterburner's report shows NVVDD Vrel 1,120 mV, MSVDD Vrel 1,070 mV) |
| MSVDD REL floor 1,100 mV | 4 hangs in 9 runs vs 5 in 8 without: not a fix |
| Afterburner voltage slider, `/wpNVVDD_off_Vrel` | no effect on driver 621.01 and later, which ignores runtime voltage offsets (source: Vix Editor and mVolt+ documentation) |
| mVolt+ runtime NVVDD offsets (`--nvvdd-offsets 0,25,25`) on a `runtime_voltage_offsets=1` image | **ignored** on 621.24: success reported, `rail_offsets_ignored_by_driver: true`, limit stays 1,120 mV (Q27) |
| **NVVDD REL floor 900 -> 1,150 mV in the image + a ramp on the curve top** | **core limit 1,120 -> 1,170 mV; light loads 3,337 to 3,382 MHz instead of about 3,180** ([Voltage lever](#voltage-lever)); but heavy loads climbed too and hit the old step at 1,065 to 1,070 mV (2 driver resets, Q28) |
| **Soft cap extended to 1,100 mV, ramp 1,100 to 1,150 mV (19 Nov)** | **TSE GT2 3 of 3 clean (176.1 to 176.6 fps), Fire Strike Ultra graphics 37,457 (+1.3 %), valid** |
| Memory +0 to +3000 at a fixed cap | +1.4 % fps from +0 to +3000; no effect on the hang |
| Afterburner open or closed, `OEM=0` or `3`, nvidia-smi logger, CPU spin, P-core pinning | no fps difference |
| **Curve, power limit, memory offset and XBAR offset written natively through `nvapi64.dll`, no Afterburner (19 Nov, driver 621.24)** | **works**: every value read back, no driver reset; a memory write zeroes the curve, so the curve goes last (Q29, [Without Afterburner](#without-afterburner-19-nov)) |
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
  1,200 mV are never used (with the stock NVVDD REL floor; raising it lifts the limit to 1,170 mV, see
  [Voltage lever](#voltage-lever)).
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

**Final (18 Nov evening):** the daily Profile 1 is this curve with every point up to 1,060 mV held flat at
3,090 MHz, and Profile 2 the same at 3,075 MHz; see [Final profiles](#final-profiles) and section 8. The idea of
flattening at about 3,090 MHz came up early on 18 Nov, was dropped when `-lgc` caps hung in the window, and came
back as a curve shape in true fullscreen.

Full curve: [data/profile1-curve.json](data/profile1-curve.json).

## 4. The Time Spy Extreme GT2 hang (measured)

### What it looks like

Every hang has the same shape in a 100 ms Afterburner log:

1. A heavy scene: last heavy sample 2,990 to 3,150 MHz at 985 to 1,050 mV, 410 to 600 W, core 38 to 49 C.
2. Within one sample (100 ms), power collapses to 120 to 155 W while usage stays at 98 to 100 %. The GPU has
   stopped mid-frame.
3. Because power fell, the clock jumps to the top of the curve (3,120 to 3,270 MHz).
4. A driver reset (event 4101; nvlddmkm event 14 "PF FLR"), or a full Windows freeze that needs a power cycle.

Until 18 Nov our "100 ms" logs were really 1 s data repeated (section 6). With true 100 ms polling, one more step
shows between 1 and 2: the readings freeze for a few samples. A hang at 700 W with a 3,060 MHz cap (`is_windowed: false`, fallback window):

| Time (UTC) | Clock | Voltage | Power | Usage | Core | XBAR | SYS | Aux |
|---|---|---|---|---|---|---|---|---|
| 09:39:06.328 | 2,932 | 0.960 V | 528.7 W | 100 % | 47 C | 2,407 | 2,437 | 1.040 V |
| 09:39:06.438 to .765 | 2,932 | 0.960 V | 427.2 W, same value 4 samples | 100 % | 47 C | 2,407 | 2,437 | 1.040 V |
| 09:39:06.873 | 3,052 | 0.975 V | 116.0 W | 100 % | 36 C | 2,475 | 2,520 | 1.055 V |

The driver reset came about 20 s after the freeze. The crossbar clock (XBAR), system clock (SYS) and aux voltage
(MSVDD), newly logged on 18 Nov, were steady right up to each hang: about 2,340 to 2,440 MHz, 2,370 to 2,480 MHz
and 1.01 to 1.055 V under GT2 load. No dip or spike before the hang, so no trigger signature there.

The clock jump in step 3 is a symptom, not the cause. We first read it as a light-load boost crash at the top of the
curve and lowered the top and the cap for hours; that chased the symptom. Passing runs show no such dip at the same
moment.

The hangs come 27 to 121 s after the test starts, at random, warm or cold, more often cold. 32 logged hang
signatures are in [data/hang-events.csv](data/hang-events.csv) (found with the autopilot's `hangscan`, one by hand;
three flagged lines are our own CUDA ramp test, marked as not a hang).

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
| Profile 1, 700 W, memory +0 | 1 (hard freeze) | 2 |

The harness runs in the table above were all windowed (`is_windowed: true` in the copied workload). From 18 Nov
09:00 UTC the harness asked for fullscreen (`is_windowed: false`), 700 W, memory +3000. We called these
"fullscreen" at the time; they were not (Q1): 3DMark fell back to a 3840x2160 window spread over several monitors,
one of them on the iGPU. Still a fair A/B within the table, but a lighter load than real fullscreen:

| Change (`is_windowed: false`, fallback window) | Hangs | Runs |
|---|---|---|
| Profile 1 | **3** | **8** |
| Cap 3,090 MHz | 1 | 4 (stops at the first hang) |
| Cap 3,060 MHz | 1 | 9 (real 2,932 MHz at 960 mV, about 530 W, 47 C at the hang) |
| Whole curve 30 MHz lower at every voltage, no cap | 1 | 6 |
| Whole curve 60 MHz lower at every voltage, no cap | **0** | **12** |
| Whole curve 45 MHz lower | **0** | **12** |
| Whole curve 45 MHz lower, XBAR +300 | **0** | **12** |
| Per band, P1 v1 (section 7) | 0 | 11 (4 hangs on the way to it) |

VBIOS for these runs: the 700 W override. The image with the MSVDD REL floor was loaded at 08:09 and no unload was
logged after it, so it was most likely still in place.

Every run and its setup: [data/tse-trials.csv](data/tse-trials.csv).

Reading (estimate): the FurMark-derived curve is about 30 to 60 MHz too aggressive at every voltage for GT2's
heaviest scenes. A clock cap does not add margin: it only moves the operating point down the same overclocked curve
to a lower voltage point with the same lack of margin. That is also why the "hang zone" seemed to move every time
the curve or the cap moved. Lowering the whole curve costs about 1 fps (153.6 to 155.1 fps at minus 60 vs 154.5 to
156.3 for Profile 1).

### Windowed vs fullscreen

The workload file copied from a 3DMark run for the harness had `"is_windowed": true`, and the window spread over
several monitors. Setting `"is_windowed": false` did not fix it: 3DMark went exclusive fullscreen only on the first
run after a boot and otherwise fell back to an ordinary 3840x2160 window. What works on every run is
`"is_windowed": false` with `"is_borderless": true` and `"swapchain_resolution"` set to the display's native size
(Q1, section 7). Every harness hang rate before 18 Nov 16:23 UTC was measured in a window.

The fps "states" (about 155 in most sessions, 167 to 173 in some, 137 later) were the display setup, not the GPU:
see section 7.

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
loading or CPU test in between. At 700 W, Profile 1 hung in 5 of 8 direct GT2 runs (windowed; 3 of 8 with `is_windowed: false`, also a window in the end)
while full app runs passed. We use the harness to compare settings, not to predict how often a full run or a game
fails.

### What we run now

Superseded by the [final profiles](#final-profiles): a soft cap written into the curve itself (section 8), not a
clock lock. The `-lgc` caps below passed windowed at 600 W but hung with `is_windowed: false` at 700 W; we keep
the notes for reference:

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
| NVVDD and MSVDD REL ceiling 1,110 -> 1,150 mV | no measurable effect. The REL (reliability) voltage is computed per chip. Afterburner's extended report (Q16) shows NVVDD Vrel 1,120 mV and MSVDD Vrel 1,070 mV on this card; we had first written "about 1,070 mV" for the core, which mixed up the rails. The REL floor (`rel_equation_min`) is the field that raises it |
| MSVDD REL floor 900 -> 1,100 mV | 4 hangs in 9 runs vs 5 in 8 without: not a fix (small samples). HWiNFO still read MSVDD at 1.07 to 1.09 V under load with it loaded |
| `runtime_voltage_offsets` 0 -> 1 (on top of the image above) | loaded 18 Nov 15:03. Meant to let mVolt+ apply voltage offsets again on 621.x, but mVolt+ v0.50.13 offsets were ignored by driver 621.24 (Q27) |
| `nvvdd_rel_equation_min_mv` 900 -> 1,150 (the NVVDD REL floor, on top of the image above, the only field changed) | **works**: core voltage limit 1,120 -> 1,170 mV after a restart; used for the curve top ([Voltage lever](#voltage-lever)) |

Background (source: the editor's documentation and changelog):

- Driver 621.01 and later ignore NVVDD and MSVDD runtime voltage offsets on RTX 50 cards: the driver reports
  success and nothing changes. Afterburner's voltage slider is a no-op there. The editor's `runtime_voltage_offsets=1`
  adds hooks so a runtime tool can apply offsets again; on our card and driver the offsets were still ignored (Q27).
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

The full list, with remedies and public sources, is [QUIRKS.md](QUIRKS.md). The ones that cost us most:

- **The benchmark was not running the way we thought** (Q1): a day of "fullscreen" hang rates ran in a window.
- **A second display adapter changed the result** (Q2, Q3): a monitor on the iGPU cost about 5.5 %, and HWiNFO
  running hid most of it, so fps jumped between "states".
- **A driver reset drops the curve, memory offset, clock cap and XBAR offset** (Q17). A step after a reset runs at
  stock and looks like a pass; we treat a voltage far over the target as "settings lost".
- **Log at 100 ms, really** (Q11, Q12): Afterburner polls at 1 s by default, its global settings are in
  `Profiles\MSIAfterburner.cfg`, and its shared memory goes stale if it restarts under a running logger.
- **BitLocker, the DXGI LUID and sleeping displays** each made launches "fail" without running (Q22).
- **Crash-once-per-boot theory** (source: Igor's Lab reports the Blackwell boost table can come back wrong after a
  driver reset until a reboot): tested, a fresh boot hung on run 1 too.
- **GPU tuning crashes do not corrupt data** beyond files written in the last minute or two, like a power cut.
  chkdsk, sfc and a WSL disk check after the first crashes were all clean.

## 7. 18 Nov afternoon: display mode, iGPU and XBAR (measured)

### Where the fps "states" came from

With the same curve, GT2 scored anywhere from 125 to 173 fps over the day. Core clocks under load were the same
in every state; only the display setup changed. Curves differ between some rows, but within one state they moved
fps by 2 % at most.

| Display state during the run | GT2 fps |
|---|---|
| `is_windowed: true`, window across monitors (17 Nov, 600 W, Profile 1) | about 147 to 150 |
| `is_windowed: false`, fallback 3840x2160 window, BenQ on the iGPU (most runs, 09:07 to 13:10) | 152.4 to 156.3 |
| same, HWiNFO running (12:20 to 12:54) | 160.0 to 162.5 |
| same window, iGPU disabled (BenQ dark), HWiNFO open or closed | 161.3 to 162.0 (165.7 with P1 v1 and XBAR +300) |
| P1 v1, 15:10 to 15:27 (cause not found) | 136.5 to 137.8, back to 156.3 to 156.5 after a GPU device restart |
| fallback window, all 4 monitors on the 5090, iGPU off (window read: 3856x2199, style `0x14CF0000`) | 125.0 just after the move, then 138.2 to 141.0 |
| sessions that were never checked (09:05, 09:21 to 09:25, 12:11 to 12:16, 13:26 to 13:31, 15:08) | 165.5 to 173.3 |
| exclusive fullscreen, 1920x1080 swapchain, first run after a boot | 172.5 |
| **borderless fullscreen, 1920x1080 swapchain** (`0,0 to 1920,1080`, style `0x94000000`) | **171.8, 171.8, 172.0** |

So: our harness fps only compare within one session and one desktop state, and the hang rates were measured on a
load that rendered about 18 % fewer frames than fullscreen. In the fallback window PresentMon showed the GPU idle about 13 % of each
frame and HWiNFO flagged "Performance Limit - Utilization" (Q9). The fixes are in Q1 and Q2; the checklist is at
the end of [QUIRKS.md](QUIRKS.md#clean-measurement-checklist).

### The iGPU penalty

| State (Profile 1, cap 3,090, fallback window) | GT2 fps |
|---|---|
| BenQ on the iGPU, HWiNFO closed | 153.1, 152.7, 152.9, 152.8 |
| BenQ on the iGPU, HWiNFO running | 162.5, 160.0, 160.9 |
| iGPU disabled, HWiNFO closed | 161.3, 162.0, 161.6 |
| iGPU disabled, HWiNFO running | 161.7, 161.5 |

About 5.5 % for one desktop monitor on the Intel iGPU, hidden by HWiNFO's iGPU polling. All four monitors now run
on the 5090, with the iGPU disabled in Windows and in the BIOS (Q2, Q3, Q4).

### Crossbar clock (XBAR)

Afterburner 4.6.7 Beta 5's out of warranty mode can offset the crossbar clock (`/wpXBARCLK_off,N`, Q10):

| XBAR offset | XBAR under load | GT2 fps |
|---|---|---|
| +0 | 2,415 MHz (0.81 x core) | 153.7, 154.0 |
| +100 | 2,505 MHz | 155.8 |
| +200 | 2,602 MHz | 156.9 |
| +300 | 2,707 MHz (0.91 x core) | 157.8 (+2.6 %) |

One driver reset at +150, on a curve that hung anyway. A 12-run series (curve minus 45, XBAR +300) and the 11-run
P1 v1 confirmation (XBAR +300) were clean. The offset is lost at every driver reset and restart.

### Per voltage band ("P1 v1")

Starting from the curve 45 MHz lower everywhere with XBAR +300, each band was moved in 15 MHz steps, 3 runs per
step, relative to Profile 1:

| Band | Settled at | Hangs on the way |
|---|---|---|
| up to 980 mV | +15 (not taken further) | none |
| 981 to 1,005 mV | -30 | 1,035 mV / 3,105 MHz at -15 (landed in the band above, which was backed off too) |
| 1,006 to 1,030 mV | -75 | 1,015 / 3,052 at -30; 1,010 / 3,075 while the band above was tried at -30; 1,020 / 3,082 in the first confirmation at -60 |
| 1,031 to 1,060 mV | -45 | |

Confirmation: 11 of 11 clean, 158.7 to 160.0 fps. The 1,006 to 1,030 mV band needs the most margin on this chip.
All in the fallback window. Repeated in borderless fullscreen, the per-band search did not converge (section 8).

### Voltage instead of clock

Afterburner's voltage controls do nothing on driver 621.x (Q14). Moving the P1 v1 curve 25 and 50 mV up the voltage
axis (same clocks, more voltage) ran clean once each at 1.04 and 1.06 V median and 463 and 516 W, in the fallback
window: not enough to say anything.

## 8. 18 Nov evening: the final profiles and how we got there

### The only valid test state

From 16:27 UTC every run used the fixed harness: `is_windowed: false`, `is_borderless: true`, swapchain 1920x1080,
which gave a borderless window exactly covering the 1080p display (`0,0 to 1920,1080`, style `0x94000000`) on every
run. GT2 ran about 172 to 173 fps there against about 141 in the fallback window. Everything before is a lighter
load and its hang rates do not carry over (Q1).

### Per-band offsets: a dead end

We repeated the per-band search (offsets relative to Profile 1 for four bands: up to 980 mV, 981 to 1,005, 1,006
to 1,030 and 1,031 to 1,060 mV; XBAR +300; 1 run per screening step):

| Step | Result |
|---|---|
| up to 980 mV, -45 to +0 | all clean, 171.6 to 172.6 fps: settled at +0 |
| 981 to 1,005 mV, -45 to +30 | clean, 172.4 to 173.2; +45 hung: settled at +30 |
| 1,006 to 1,030 mV at -45 | hung: kept at -60 |
| 1,031 to 1,060 mV, -45 and -30 | clean; -15 hung: settled at -30 |
| reliability, 1,006 to 1,030 at -60, then -75 | hung at 1,010 mV / 3,067 MHz, then 1,010 mV / 3,090 MHz |

Here we found a method flaw (Q25): +30 on the 981 to 1,005 mV band against -60 or -75 on the band above made the
curve fall as voltage rose, so the driver no longer followed the bands and hangs were pinned on the wrong band.
The second pass capped every point at the lowest clock above it (suffix minimum) and started over:

| Offsets (up to 980 / 981 to 1,005 / 1,006 to 1,030 / 1,031 to 1,060 mV) | Result |
|---|---|
| 0 / -60 / -75 / -15 to +15 | clean; +30 on the top band hung at 1,015 mV / 3,082 MHz |
| 0 / -60 / -90 / +15 | 2 clean, then hung at 1,000 mV / 3,045 MHz |
| 0 / -75 / -90 / +15 | hung at 1,010 mV / 3,090 MHz |
| 0 / -75 / -105 / +15 | 2 clean, then hung (no reading) |
| 0 / -75 / -120 / +15 | 1 clean, then hung at 815 mV / 2,655 MHz |
| -15 / -75 / -120 / +15 | 1 clean, then hung at 1,020 mV / 3,090 MHz |

Lowering the 1,006 to 1,030 mV band from -60 to -120 MHz never stopped the hangs. They kept landing at 3,045 to
3,090 MHz, and the voltage read at the hang rose step by step (1,000, 1,010, 1,015, 1,020 mV): each cut only moved
the same clocks to a higher-voltage point. That pointed at a clock limit under heavy load, not a per-voltage
margin. (The voltage read at a hang sits about 10 mV under the V/F point in use, Q26.) One hang read 815 mV /
2,655 MHz, far from the others; we did not find out why.

### The soft cap

So we tested a clock wall directly: the Profile 1 curve with every point up to 1,060 mV held to at most W MHz
(flat from where the curve first reaches W), and every point above 1,060 mV untouched, so light loads still reach
3,247 MHz at higher voltage. Monotonic by construction. 3 runs per step:

| W (MHz) | Result | GT2 fps |
|---|---|---|
| 3,045 | clean 3 of 3 | 172.7 to 173.5 |
| 3,060 | clean 3 of 3 | 173.0 to 173.4 |
| 3,075 | clean 3 of 3 | 172.9 to 173.3 |
| 3,090 | clean 3 of 3 | 172.8 to 173.4 |
| 3,105 | run 1 173.5, run 2 hung at 1,020 mV / 3,090 MHz | |

Final pass: cap 3,090 hung once (1,020 mV / 3,082 MHz on the first run), then ran 3 clean (172.9 to 173.3);
cap 3,075 ran 3 of 3 clean (171.9 to 173.4). Totals: **cap 3,090 = 6 clean, 1 hang in 7 (Profile 1, the max,
at the edge); cap 3,075 = 6 of 6 (Profile 2)**. Both saved to Afterburner (slot 1 and Startup, slot 2), the
original Profile 1 curve archived.

No fps cost: about 173 fps in GT2 with or without the cap. Telemetry from the final Profile 1 runs (100 ms, samples
with usage at least 95 % and over 300 W): 95 % at about 3,064 MHz, 1.015 V, 570 W; 5 % above 3,100 MHz (about
3,107) at about 1.02 V and 536 W. Reading (estimate): the heaviest scenes hit a clock wall near 3,090 MHz on this
chip whatever the voltage, and the flat stretch also keeps the driver from hopping between V/F points during load
transients; light moments still run above 3,100 MHz at higher voltage, which is where the curve above 1,060 mV
earns its keep. Why the earlier `-lgc` caps hung while the soft cap holds is not proven; the obvious differences
are the untouched top of the curve and the test state (window then, fullscreen now).

### Resizable BAR and the crossbar clock

| Change (final Profile 1, borderless fullscreen) | GT2 fps |
|---|---|
| ReBAR driver default, XBAR +300 | 173.5, 173.0 |
| ReBAR forced (NPI, `rBAR - Enable` 0x000F00BA = 1) | 175.5, 175.3 |
| ReBAR off again (control) | 173.1 |
| ReBAR forced, 3 reliability runs | 175.9, 175.9, 175.4 |
| + XBAR +400 (2,737 MHz) | 176.6 |
| + XBAR +500 (2,775 MHz, about 0.9 x core, the ceiling) | 176.6, 176.4, 176.9, 176.5 |

ReBAR stays forced for 3DMark (Q23). XBAR +400 was applied by a logon task (Q10) until 19 Nov; since then
blackout-tray writes it natively ([Without Afterburner](#without-afterburner-19-nov)).

### Voltage lever

The soft cap only touches points up to 1,060 mV. Above that the curve was flat at 3,247 MHz, and the card never
went past 1,120 mV: that is the chip's NVVDD Vrel (Q16). The question was whether more voltage at the top buys
clock in light moments, where the card sits at the top of the curve.

**Runtime offsets: no (measured, 19:20 UTC).** mVolt+ v0.50.13 with `--nvvdd-offsets 0,25,25` on driver 621.24,
with the image that has `runtime_voltage_offsets=1` loaded, returned success, but its own state reported
`rail_offsets_ignored_by_driver: true` and `nvvdd_max` stayed at 1,120 mV (Q27).

**Static route: the NVVDD REL floor (measured).** The REL voltage is computed per chip from SPEEDO and then
clamped between the REL equation floor and ceiling in the VBIOS (our reading of the editor's field names, borne
out by the result below; our chip computes 1,120 mV, Q16). Raising the floor above the computed value raises the
limit:

1. Vix Editor, one field on top of the loaded image: `nvvdd_rel_equation_min_mv` 900 -> 1,150. The full field
   diff showed only that field changed (101 bytes), integrity check valid.
2. Loaded with the Vix Loader, restart, verified byte for byte against the stored image.
3. mVolt+ `--status` (read-only) after the restart: `nvvdd_max` **1,170 mV** (REL 1,150 plus the ALT +20 step),
   was 1,120.

Under load the voltage tops out at about 1.135 V read, about 1.145 V set, which is the REL 1,150 limit; the ALT
+20 above it is not used (measured; the read sits about 10 mV under the point in use, Q26).

**Who uses it (measured, 18 Nov; corrected 19 Nov).** On 18 Nov 4K TSE GT1 (about 540 W) never went above
1.02 V / 3,075 MHz, and we concluded heavy loads do not use the new top. That was wrong: the next morning Fire
Strike Ultra and GT2 ran heavy scenes at 1.065 to 1.085 V (see [Heavy loads climb too](#heavy-loads-climb-too-19-nov)).
Light loads do: TSE GT1 at 720p (about 250 W) sits at the curve
top, 3,172 to 3,195 MHz at 1.055 to 1.06 V with the old flat top.

**Caveat: heavy loads pay for it in power (measured, 19 Nov).** With the REL 1,150 image heavy loads ride up the
flat cap region of the curve. Steel Nomad on the final config ran 3,030 to 3,037 MHz at 1.035 to 1.04 V and 660 to
678 W, against about 2,977 MHz at 1.01 V and about 625 W on 18 Nov: about 45 W more, closer to the 700 W limit, for
the same score (16,523 against 16,492 to 16,645). The higher voltage limit only pays off in lighter loads (Fire
Strike Ultra graphics +1.3 %); in heavy loads it costs power and buys nothing.

**Ladder (measured).** Top of the Profile 1 curve as a straight ramp from 3,247 MHz at 1,070 mV to 3,247 + D at
1,170 mV, light 720p GT1, 1 run per screening step:

| D (MHz) | Max clock read | Result |
|---|---|---|
| 0 (flat, baseline) | 3,195 at 1.06 V | clean |
| +60 | 3,255 at 1.135 V | clean |
| +105 | 3,277 | clean |
| +150 | 3,315 | clean |
| +195 | 3,345 | clean; 3 of 3 reliability runs clean (3,337 each) |
| +255 | 3,382 | clean; 3 of 3 reliability runs clean (3,360 to 3,367) |
| +315 | 3,457 at 1.145 V | **hung**, driver reset after 16 s |

So the light-load ceiling on this chip is between about 3,380 MHz (clean) and 3,457 MHz (hang), near 3,450
(estimate). The read clocks sit below the curve points they run on; we did not separate temperature bins from the
read offset. Fps at 720p stayed at about 275 to 278 whatever the clock, because that test is CPU bound: the clock
is the metric.

**Saved (20:00 UTC).** Profile 1 = cap 3,090 up to 1,060 mV + top +255 (1,100 mV 3,324, 1,125 3,387, 1,150 3,451,
1,170 3,502), slot 1 and Startup. Profile 2 = cap 3,075 + top +195 (1,100 mV 3,306, 1,150 3,403, 1,170 3,442), one
step under, slot 2. Both keep 3,247 MHz at 1,070 mV.

**Heavy check (measured, 1 run; withdrawn 19 Nov).** TSE GT2 on that Profile 1 gave 179.0 fps, clean (176.6
before). We read it as lighter moments inside GT2 reaching higher clocks. The next morning showed heavy scenes
running on the 3,247 MHz step above 1,065 mV instead, the step that then caused two driver resets; the 179.0 is not
a result of a safe profile and is withdrawn.

### Heavy loads climb too (19 Nov)

**Two crashes (measured).** Both in owner loads with Profile 1 as saved at 20:00 the evening before:

| Time (UTC) | Load | Last heavy samples before the power collapse |
|---|---|---|
| 19 Nov 00:06 | owner load, about 615 W | 3,030 to 3,090 MHz at 1.04 to 1.055 V; nvlddmkm 14/153 TDR |
| 19 Nov 09:28 | Fire Strike Ultra, XBAR +400 | 3,030 to 3,052 MHz at 1.03 to 1.045 V, about 610 to 620 W |

Both have the hang signature from section 4 (power collapses while usage stays at the top). Before the REL
1,150 image the same kind of heavy load ran at 1.015 to 1.035 V; now it ran at 1.03 to 1.055 V.

**Why (measured, plus our reading).** A Fire Strike Ultra run just before the second crash (09:22 to 09:24) ran
**3,180 to 3,217 MHz at 1.065 to 1.08 V** in heavy scenes. That is the old curve's step: the soft cap held 3,090 MHz
only up to 1,060 mV, then the curve jumped to 3,247 MHz at 1,065 to 1,070 mV. With the old 1,120 mV limit heavy
loads never got near it; with the 1,170 mV limit the driver moves them up the curve too, and the step became
reachable. The cause we judged: the shape of the curve between 1,060 and 1,100 mV, not the light-load top itself
(Q28). The read voltages at the hang sit about 10 mV under the point in use (Q26).

**Fix (measured).** Extend the soft cap to where heavy loads can now go, and start the ramp above it:

| mV | Profile 1 | Profile 2 |
|---|---|---|
| 950 | 3,015 | 3,015 |
| 1,000 to 1,100 | **3,090** (flat) | **3,075** (flat) |
| 1,110 | 3,160 | 3,136 |
| 1,125 | 3,265 | 3,228 |
| 1,140 | 3,370 | 3,319 |
| 1,150 and 1,170 | **3,440** (flat) | **3,380** (flat) |

Saved 19 Nov 09:29 to Afterburner slot 1 and Startup (Profile 1) and slot 2 (Profile 2). Points up to 950 mV are
unchanged.

**Checks (measured, borderless fullscreen, XBAR +400, ReBAR forced):**

| Run | Result | Max voltage |
|---|---|---|
| TSE GT2 1 | 176.6 fps, clean | 1.085 V |
| TSE GT2 2 | 176.1 fps, clean | 1.085 V |
| TSE GT2 3 | 176.2 fps, clean | 1.085 V |
| Light 720p GT1 | 274.1 fps, clean | 1.135 V |

GT2 still spends about half its time at or above 1.06 V (292 to 357 of about 650 samples per run), now at the 3,090 MHz
wall instead of on the step, and the light load still reaches the top of the voltage range. GT2 fps is back to the
pre-lever 176.6: the 179.0 came from the step.

**Validated (measured).** Fire Strike Ultra app run, 19 Nov 09:45: 34,409 overall, **37,457 graphics** (previous
best 36,975, +1.3 %), physics 51,213, combined 16,366, valid. See [Results](#results).

**Validated later the same morning (measured):** Fire Strike Ultra run 2 34,563 / 37,465 graphics, Time Spy Extreme
25,025 / 29,380 graphics, Steel Nomad Stress Test 99.3 % (see [Final config, 19 Nov](#final-config-19-nov-measured)).

**Still open.** Real games (light, CPU bound loads are where the new top runs) and the 10 to 12 run final pass.
OCCT 3D Adaptive (Variable 20 min, Switch 20 min) is done: clean, but it stays below the 3,090 MHz edge (about
3,037 MHz peak).

### Without Afterburner (19 Nov)

**Why.** Two things kept a tuning app in the daily path: an Afterburner profile does not store the XBAR offset
(Q10), and a driver reset drops every setting (Q17). The setup had grown to Afterburner at logon, a logon task for
XBAR and a task on the reset events. The goal was one small tray tool that applies everything, and no resident
tuning app.

**Read-only probe (measured, driver 621.24, about 15:00 UTC).** A small test program called the driver's own
library, `nvapi64.dll`, through the private interfaces that tuning apps use, and only read:

| Setting | What the driver returned |
|---|---|
| Curve offsets (private curve table) | the Afterburner Profile 1 offsets on 127 of 127 core points, no mismatch |
| Power limit | 121.739 % of the default, which is 700 W (allowed range 69.565 to 121.739 %) |
| XBAR offset (clock domains table, domain 1, byte offset `0x53C`) | 400000 kHz, the live +400 |
| Memory offset (Pstates20, the public structure) | 0, although the memory clock read 17,001 MHz (+3000 set by Afterburner) |
| Voltage boost | 0 (the slider does nothing on this driver, Q14) |

The clock domains layout had been validated in public only up to driver 610.88 (source: rtx5090-xbar-control,
below); it read correctly on 621.24.

**Write test (measured, elevated, about 15:10 UTC).** Smallest steps first, each one read back:

1. Curve table written back unchanged: accepted, read back identical (0 of 255 slots differ).
2. Power limit written back unchanged: accepted, read back 121.739 %.
3. XBAR offset 400 -> 300 -> 400 MHz: each value read back, the curve untouched.
4. Memory +3000 through SetPstates20: accepted, memory clock 17,001 MHz, XBAR offset kept, but **all 127 core curve
   offsets were zeroed** (two non-core slots kept their values).
5. Curve restored natively from the saved copy: read back identical.

No driver reset and no nvlddmkm or 4101 event. A light 720p GT1 run afterwards: core 3,270 MHz at 1.135 V, XBAR
2,835 MHz, memory 17,001 MHz, no hang, 266 fps. The morning run on the same profile had the same 1.135 V top and
XBAR at 2,827 MHz (274.1 fps; that test is CPU bound, so the clocks are the metric).

**The order.** Power limit, memory offset, XBAR offset, curve last, then read the curve back. Because the memory
write zeroes the curve, a profile that is refused half way ends on the stock curve, not on a wrong one.

**What else it showed (measured unless marked):**

- The memory offset set by Afterburner did not show in the Pstates20 read (0 at +3000). After the native write the
  same read returned 3000000 kHz. Where Afterburner keeps it we do not know.
- No voltage control this way: the voltage boost reads 0 and runtime Vrel offsets stay ignored on 621.x (Q14, Q27).
  The voltage limit still comes from the VBIOS image ([Voltage lever](#voltage-lever)).
- Writes need admin: an unelevated write is refused with -137, NVAPI's "invalid user privilege". Reads do not
  (as recorded in the blackout-tray notes).
- A zeroed buffer is silently ignored: read the table, change it, write it back (as recorded in the blackout-tray
  notes).
- The settings stay after the writing program has exited, until a driver reset or a restart: the light run above
  ran after the test program had ended.

**Result.** [blackout-tray](https://github.com/Xipzer/blackout-tray) now holds both final profiles: profile 1 (cap
3,090, read from the card, identical to Afterburner slot 1 on 127 of 127 points) and profile 2 (cap 3,075, from the
offsets in slot 2), each with power limit 121.739 % and memory +3000, plus the XBAR offset (+400). It applies them
10 s after it starts, when the profile is switched or re-applied from its panel, and 15 s after a driver reset. It
loads `nvapi64.dll` only while applying and polls nothing: a driver reset is noticed through a subscription to the
System event log (Display 4101, nvlddmkm 14 and 153). **That reset path has not yet been exercised by a real driver
reset.** Checked so far: profile 2 applied, then profile 1, each read back; memory 17,001 MHz, power limit 700 W,
XBAR offset 400000 kHz, no driver events.

Afterburner no longer starts with Windows (its logon task and the XBAR logon task are disabled). It stays installed
as the fallback and for monitoring during tuning sessions: the 100 ms telemetry logger reads Afterburner's shared
memory, so it only works while Afterburner runs.

**Caveats.** These are private driver interfaces. NVIDIA does not document them, and a future driver can change or
block them. Tested on one card and one driver. The tray tool applies a profile, it does not find one: the search
and the stress tests still need a tuning app. Setup, buffer layouts and interface ids:
[docs/gpu.md](https://github.com/Xipzer/blackout-tray/blob/main/docs/gpu.md).

**Credit.** We did not work the layouts out ourselves. They come from these open-source projects and write-ups,
checked again on our card before use:

- XBAR and the clock domains table: https://github.com/SHANAjam/rtx5090-xbar-control (MIT)
- Curve tables: https://github.com/arcnmx/nvapi-rs (MIT) and
  https://vuplea.github.io/simple-nvidia-undervolt/DEVELOPMENT.html
- Memory through Pstates20 and the curve reset it causes: https://erfi.dev/reference/vfctl-gpu-curve-tool/
- The Afterburner developer naming the interfaces: https://www.overclock.net/posts/29608363/

### From start to finish

| State | GT2 fps |
|---|---|
| Profile 1, old harness (fallback window across monitors, one on the iGPU) | about 153 |
| borderless fullscreen, iGPU off | about 172 to 173 |
| + soft cap (stable) | about 173 |
| + ReBAR forced | 175.4 to 175.9 |
| + XBAR +400 | **176.6** |
| + voltage top (REL floor 1,150 mV, top +255), 18 Nov | 179.0 (1 run; withdrawn, it ran on the unsafe step, Q28) |
| + soft cap to 1,100 mV, ramp to 3,440 at 1,150 mV, 19 Nov | 176.6, 176.1, 176.2 (3 of 3 clean) |

The first jump is a measurement fix, not a speed-up the card gained: the old harness rendered a lighter load.
The real gains on top of a correct measurement are ReBAR (+1.2 %) and XBAR (+0.5 % from +300 to +400, +2.6 % from
+0 to +300 in the window).

### Silicon

SPEEDO 2109, about a third into its fuse range (Q16). A heavy-load wall at about 3,090 MHz is in line with the
3.1 to 3.15 GHz owners commonly report as game-stable for this card (estimate), so this is an ordinary chip.

## Open questions

1. What in GT2's heavy scenes needs 30 to 60 MHz more margin at every voltage than FurMark, a CUDA ramp, OCCT and
   Steel Nomad? Is it a specific unit (RT, async compute, the fabric) or the load profile? XBAR, SYS and aux
   voltage showed nothing before the hang.
2. Was the higher rate with a cold core real? Our guess was extra temperature bins; but the hangs also came at 46
   to 49 C, and many cold runs were windowed harness runs.
3. Would a higher NVVDD REL floor help? Answered for light loads: yes, it lifts the core limit to 1,170 mV and
   light loads gain about 140 to 210 MHz ([Voltage lever](#voltage-lever)). Heavy loads climb up the curve too, so
   the soft cap had to be extended to 1,100 mV (Q28); GT2 gains nothing. Do games gain, and does the new top hold
   in them?
4. After one reboot, direct GT2 launches ran at about 90 fps instead of 149, capped or not, then later went back to
   149. We did not find out why; the hang results look the same in both states. After section 7 a display or
   adapter state is the first suspect, but we did not check the window then.
5. Does the hang happen in games? Not known yet; the plan is to watch nvlddmkm and 4101 events for one to two weeks.
6. Why does a soft cap hold where `-lgc` caps and per-band margin did not? Our reading is a clock wall plus fewer
   V/F point hops (section 8); it is not proven. Is the wall at the same clock in games?
7. Why does exclusive fullscreen (`is_borderless: false`) hold only for the first run after a boot?
8. Why did a 600 W setting with a 3,060 cap (2 runs with `is_windowed: false`) run about 110 MHz lower than the same cap at 700 W,
   while drawing only about 460 W? Not yet explained.
9. Why did the fallback window run slower with all four monitors on the 5090 (138 to 141 fps) than with one of them
   on the iGPU (153 to 162)? Not explained; it no longer matters once the run is fullscreen.
10. Why did the MSVDD REL floor of 1,100 mV leave MSVDD at 1.07 to 1.09 V under load (Q16)?
11. Does the native re-apply after a driver reset work in practice? The event subscription is in place but no real
    reset has happened since ([Without Afterburner](#without-afterburner-19-nov)). And will the private interfaces
    survive the next driver?
12. Where does Afterburner keep its memory offset, if not in the Pstates20 value we read (Q29)?

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
11. **Check how the benchmark really runs.** A windowed workload is a different load; a day of hang rates was measured
    in the wrong mode.
12. **Caps and power limits do not add margin.** They move the operating point along the same curve. Margin comes
    from lowering the curve.
13. **Check the logger's real sample rate.** Repeated values every ten lines meant 1 s data, not 100 ms.
14. **Check the window, not the setting.** `is_windowed: false` did not make the run fullscreen; reading the window
    rectangle and style did tell us.
15. **One display adapter while measuring.** A monitor on the iGPU changed fps by 5.5 %, and a monitoring tool
    changed how much.
16. **When fps moves but clocks do not, look outside the GPU** (display path, compositor, other adapters) before
    chasing memory, clock stretching or monitoring overhead.
17. **When hangs land on the same clock whatever the voltage, it is a clock wall.** Per-band margin only moves the
    same clock to a higher voltage. Cap the clock in the curve and leave the top alone.
18. **Keep the curve monotonic** when editing parts of it, or the driver will not do what the bands say (Q25).
19. **Check the driver profile too.** Resizable BAR on in the BIOS did not mean 3DMark used it (Q23).
20. **A "success" from a tool is not a change.** mVolt+ reported success while the driver ignored the offsets; read
    the limit back (Q27). The static VBIOS field did what the runtime call did not.
21. **Find the load that uses the change.** The voltage top is invisible in heavy GT2 at 4K and obvious in a light
    720p run; test where the card actually sits at the top of the curve.
22. **A higher voltage limit moves heavy loads up the curve too.** A step that heavy loads could not reach before
    becomes reachable; extend the soft cap to where they can now go, and re-test heavy loads after any change to
    the top (Q28).
23. **One write can undo another.** The native memory offset write zeroed the whole curve. Read everything back
    after each write, and fix the order (Q29).

## What we got wrong

- **A clock "hang zone" from about 3,000 MHz up.** Caps at 3,060 and 3,090 MHz looked like a fix (13 of 13 cold runs
  at 600 W), but those runs were windowed. With `is_windowed: false` at 700 W both caps hung, the 3,060 cap at a real 2,932 MHz and
  960 mV. The zone seemed to move with every curve or cap because there was none: the whole curve lacked 30 to 60
  MHz of margin, and a cap only slides along that curve.
- **"700 W raises the hang rate."** The 600 W caps that looked clean were windowed, the 700 W caps that hung ran with
  `is_windowed: false`. A power limit is a ceiling, and GT2 drew about 460 to 560 W there, under 700 W. Withdrawn
  until 600 W and 700 W are compared in the same mode.
- **The clock jump read as a light-load boost crash** (section 4): a symptom, as noted above.
- **Memory +3000 as a suspect.** Memory +0 also hung (windowed, run 2, hard freeze), so memory is not the only cause.
- **"100 ms" telemetry.** It was 1 s data until 18 Nov (section 6).
- **"Fullscreen" from 18 Nov 09:00.** Setting `is_windowed: false` was not enough; those runs were in a 3840x2160
  window (Q1). Labels in the data now say what the window really was, where known.
- **"True fullscreen is harsher"** (18 Nov 15:49). It rested on one hang 27 s into the first run with a 1080p
  swapchain, whose window we never saw. Unproven.
- **"This chip's REL is about 1,070 mV."** That is MSVDD's Vrel; NVVDD's is 1,120 mV (Q16).
- **"Hangs need margin at every voltage; caps do not add it."** True for `-lgc` caps in the window, wrong as a rule:
  in fullscreen, per-band margin never stopped the hangs, and a soft cap written into the curve did (section 8).
- **"Heavy loads never reach the new voltage top"** (18 Nov 20:00). Based on one 4K GT1 run. With the 1,170 mV limit
  heavy Fire Strike Ultra scenes ran at 1.065 to 1.08 V, hit the old step at 1,065 to 1,070 mV and caused two driver resets; the
  179.0 fps GT2 result came from that step and is withdrawn (Q28).
- **The fps "states" as a GPU effect.** We tested memory offset, stock curve, Afterburner monitoring, out of warranty
  mode, XBAR, CPU idle and thread placement before finding the display path (section 7).

## Data

| File | What it holds |
|---|---|
| [data/vf-results.json](data/vf-results.json), [data/vf-steps.csv](data/vf-steps.csv) | compute per-voltage search |
| [data/vf-3d.json](data/vf-3d.json) | FurMark per-voltage search, lock and real clocks |
| [data/curve-history.csv](data/curve-history.csv) | every curve candidate and its Time Spy or Steel Nomad result |
| [data/curve-candidate-1.json](data/curve-candidate-1.json) | the first curve (compute limits minus 45 MHz), historical |
| [data/profile1-curve.json](data/profile1-curve.json) | the max curve, offset per point (the base of both final profiles) |
| [data/final-profiles.json](data/final-profiles.json) | the final Profile 1 and Profile 2: clock per voltage point, settings, results |
| [data/tse-trials.csv](data/tse-trials.csv) | every automated GT2 run: setup, power, VBIOS, cap, result, and `display_mode` (what the window really was, where known; most runs before 18 Nov 16:23 were in a window) |
| [data/hang-events.csv](data/hang-events.csv) | hang signatures found in the 100 ms telemetry |
| [data/vbios-override.json](data/vbios-override.json) | VBIOS fields changed and what each did |
| [data/afterburner-report.json](data/afterburner-report.json) | the card's binning and rail limits from Afterburner's extended report |
| [QUIRKS.md](QUIRKS.md) | every quirk, its remedy and public sources, and the clean measurement checklist |

Times are UTC.

## Licence

Text, tables and figures: CC BY 4.0. See [LICENSE.md](LICENSE.md).
