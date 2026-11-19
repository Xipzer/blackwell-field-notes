# Quirks and remedies (RTX 5090, Windows 11, driver 621.24)

Every quirk we hit while tuning one RTX 5090 Founders Edition, with what we saw, what fixed it, and whether we found
anyone else reporting the same thing. Short index in the [README](README.md#quirks-index).

Labels as in the README: **Measured** (our logs), **Source** (public, linked), **Estimate** (our reasoning).
"Nothing public found" means a short web search on 18 Nov 2026 turned up no report of the same thing; it does not
mean nobody has seen it. Times are UTC.

Machine for context: i9-14900KS on an ASRock Z790I Lightning WiFi (Intel UHD 770 iGPU), RTX 5090 FE on water,
MSI Afterburner 4.6.7 Beta 5 (build 19412) with RTSS 7.3.7, HWiNFO64 8.61 (free), PresentMon 2.6.0, 3DMark
(Steam) with Time Spy, four monitors: Samsung Odyssey Neo G9 5120x1440 240 Hz (DSC), Dell AW2521H 1080p 360 Hz,
ASUS XG258Q 1080p 240 Hz, BenQ XL2420Z 1080p 144 Hz.

---

## 3DMark harness

<a id="q1"></a>
### Q1. A direct 3DMark launch silently runs in a window instead of fullscreen

**What we saw (measured).** Our harness starts one test directly:
`3DMarkTimeSpy.exe --in <workload.json> --out <result.json>` (see
[blackwell-overclock-autopilot](https://github.com/Xipzer/blackwell-overclock-autopilot/blob/main/docs/VALIDATION.md)).
The workload JSON has `is_windowed`, `is_borderless`, `swapchain_resolution` and `scaling_mode` keys. We read the
test window's rectangle and style about 30 s into each run (`GetWindowRect`, `GetWindowLong(GWL_STYLE)`):

| Workload settings | What happened | Window | GT2 fps |
|---|---|---|---|
| `is_windowed: true` (as copied from a normal 3DMark run) | ordinary window across several monitors | not read at the time | about 149 (17 Nov) |
| `is_windowed: false` (exclusive fullscreen), swapchain as copied | ordinary window, never fullscreen | 0,0 to 3856,2199, style `0x14CF0000` | 125 to 173, depending on the desktop state (Q2, Q3) |
| `is_windowed: false`, swapchain 1920x1080 | fullscreen on the **first run after a boot only**; every later run fell back to the window | first: fullscreen; later: 3856x2199, `0x14CF0000` | 172.5 first, then 141.0 |
| `is_windowed: true`, `is_borderless: true`, swapchain 1920x1080 | ordinary window | 3856x2199, `0x14CF0000` | 140.4 |
| **`is_windowed: false`, `is_borderless: true`, swapchain 1920x1080** | **borderless fullscreen on the target 1080p display, every run** | **0,0 to 1920,1080, style `0x94000000`** | **171.8, 171.8, 172.0** |

Style `0x14CF0000` includes `WS_CAPTION` and `WS_THICKFRAME`: a normal window with a title bar, here 3840x2160 plus
borders, larger than the 1920x1080 display it started on, so it spread onto the next monitors. Style `0x94000000`
is `WS_POPUP | WS_VISIBLE | WS_CLIPSIBLINGS`: a borderless window exactly the size of the display.

**Why it matters (measured).** The fallback window is a lighter load. In it, GT2 ran at about 141 fps against 172
in borderless fullscreen (18 % fewer frames). A PresentMon capture of a fallback run showed the GPU waiting 0.92 ms
of every 7.09 ms frame (GPU busy 6.07 ms, about 13 % idle), and HWiNFO's "Performance Limit - Utilization" flag
was set during the load (Q9). So every hang rate measured in the fallback window understates the load and
overstates the stability margin. Its fps only compare with other runs in the same session and state.

**Remedy.** Write `is_windowed: false`, `is_borderless: true` and `swapchain_resolution` = the native size of the
display the test should fill, before every launch. Then check the window during the run: if its style has the
`WS_CAPTION` bit (`0x00C00000`), or its size is not the display's, do not count the run. The autopilot's `tse`
command now does both (`[3dmark] borderless`, `swapchain`, `window_check`).

We do not know why exclusive fullscreen worked only on the first run after a boot. It was on a four-monitor
desktop with mixed refresh rates (Q2); with one monitor it may behave.

**Public sources.** UL documents that every test renders at a fixed resolution and is then scaled to the desktop
resolution, and that the app warns "Workload appears to be running in Windowed mode, this invalidates this result"
for a normal run:
- https://support.benchmarks.ul.com/support/solutions/articles/44001789016-does-my-desktop-resolution-affect-my-3dmark-score-
- https://steamcommunity.com/app/223850/discussions/0/1744479063995557069

Nothing public found on direct launches falling back to a window, or on the `is_borderless` fix.

<a id="q2"></a>
### Q2. A monitor on the Intel iGPU costs about 5.5 % in GT2

**What we saw (measured).** Three monitors were on the 5090 and the BenQ was on the motherboard's HDMI output
(Intel UHD 770). The fallback 3DMark window (Q1) spread onto it. With HWiNFO closed, same settings, alternating:

| State | GT2 fps |
|---|---|
| iGPU enabled, BenQ on it | 153.1, 152.7 |
| iGPU disabled in Device Manager (BenQ dark) | 161.3, 162.0 |

That is 5 to 6 % (about 5.5 %). At the idle desktop the 5090 sent about 380 MB/s over PCIe (`nvidia-smi dmon -s t`),
which we read as the cross-adapter copy of the desktop to the iGPU (estimate). Core clocks were the same in every
state; only the fps moved.

Ruled out on the same harness, two runs each: Afterburner open or closed, Afterburner `OEM=0` or `3`, a 1 s
nvidia-smi logger, a one-thread CPU spin, pinning 3DMark to P-core threads, HWiNFO's PresentMon and NVML options.
See [data/tse-trials.csv](data/tse-trials.csv) (labels `hwab`, `cpuab`, `smiab`, `hwbisect`, `pcoreab`, `igpuab`,
`igpu2`, `oowtest`).

**Remedy.** Put every monitor on the NVIDIA card and turn the iGPU off:
1. Windows: `pnputil /disable-device "<iGPU instance id>"` (or Device Manager, Disable device).
2. BIOS: on this ASRock board, `IGPU Multi-Monitor` = Disabled with `Primary Graphics Adapter` = PCI Express (we
   wrote it with AMI SCEWIN; the BIOS menu does the same). Applied at the next restart.

The Founders Edition has three DisplayPort outputs and one HDMI. The BenQ's 144 Hz input is DisplayPort, so it now
hangs off the card's HDMI output through a USB-powered active HDMI-to-DisplayPort adapter (Cable Matters), at
144 Hz.

**Caveat.** Our 3DMark window spread onto the iGPU monitor. We did not separately measure a game that stays on
a dGPU monitor while the iGPU only shows a desktop.

**Public sources.** Microsoft describes the cost of cross-adapter copies on hybrid laptops (frames rendered on
one GPU and shown by another) and measured about 16 % more fps on average when that copy is avoided; forum
answers say the same for desktops when the game window touches the iGPU's display:
- https://devblogs.microsoft.com/directx/optimizing-hybrid-laptop-performance-with-cross-adapter-scan-out-caso/
- https://superuser.com/questions/1762779/use-both-integrated-and-discrete-graphics-card-with-different-screens
- https://linustechtips.com/topic/1375577-multi-monitor-from-the-igpu-performance

<a id="q3"></a>
### Q3. HWiNFO hid the iGPU penalty

**What we saw (measured).** With the BenQ on the iGPU, GT2 ran 160.0 to 162.5 fps while HWiNFO was running and
152.8 to 152.9 fps when it was closed. With the iGPU disabled, HWiNFO made no difference (161.5 to 161.7). So
HWiNFO did not speed anything up; it took away most of the iGPU penalty. The fast runs coincided with iGPU events
in HWiNFO ("Wake disabled GPUs" polling) and with Level Zero crashes (Q5).

**Reading (estimate).** HWiNFO's polling keeps the iGPU out of its low-power state, so the cross-adapter copy to
its monitor costs less. This made our fps jump between "states" depending on whether HWiNFO was open, which we
first chased as a core-side effect (memory offset, stretched clocks, Afterburner monitoring, XBAR).

**Remedy.** Same as Q2: no monitor on the iGPU. Then HWiNFO open or closed makes no difference.

Nothing public found about HWiNFO masking a mixed-adapter penalty.

<a id="q4"></a>
### Q4. "A DSC monitor uses two display heads" did not stop four monitors on one 5090

**Concern (source).** NVIDIA says a DSC display can take two of the GPU's four internal heads when its pixel rate
exceeds one head's limit, so two such displays leave no room for a third:
- https://nvidia.custhelp.com/app/answers/detail/a_id/5338/

**What we saw (measured).** All four monitors, including the Neo G9 at 5120x1440 240 Hz with DSC, run at full
refresh on the 5090 at once: Neo G9 240 Hz, AW2521H 360 Hz, XG258Q 240 Hz, XL2420Z 144 Hz.

**Public sources.** NVIDIA's RTX 5090 page lists up to four independent displays ("4K 165Hz", or two at "4K 360Hz
or 8K 100Hz with DSC", others "based on available bandwidth"). A forum post says Blackwell doubled the per-head
pixel rate so a 4K 240 Hz DSC display fits in one head (it cites TFTCentral; unconfirmed by NVIDIA):
- https://www.nvidia.com/en-us/geforce/graphics-cards/50-series/rtx-5090/
- https://www.nvidia.com/en-us/geforce/forums/geforce-graphics-cards/5/587431/rtx-5090-can-it-drive-three-aw3225qf-4k-240hz-dsc/

If you plan more than one high pixel-rate DSC display, test it; our case was one DSC display plus three 1080p ones.

---

## Drivers and HWiNFO

<a id="q5"></a>
### Q5. An old Intel iGPU driver crashes HWiNFO, and Windows Update keeps putting it back

**What we saw (measured).** HWiNFO 8.60 hung or crashed at GPU detection. The fault was in `ze_intel_gpu64.dll`
(Intel's oneAPI Level Zero runtime) from Intel graphics driver 31.0.101.5592 (July 2024). That version came in as
a board-specific package, which Windows ranks above Intel's generic one. We removed it on 15 Nov and the iGPU went
to 32.0.101.7092; by 18 Nov Windows Update had reinstalled the old board package and HWiNFO crashed again.

**Remedy.**
1. Find the package: `pnputil /enum-drivers` (look for `iigd_dch.inf` with the old version), export it as a backup
   (`pnputil /export-driver oemNN.inf <folder>`), then `pnputil /delete-driver oemNN.inf /uninstall` (no `/force`).
2. Stop Windows Update from bringing it back: policy "Do not include drivers with Windows Updates", registry
   `HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\ExcludeWUDriversInQualityUpdate` = 1 (DWORD). This
   blocks all driver updates through Windows Update; install GPU drivers yourself.
3. Or keep the driver and turn off HWiNFO's oneAPI support: Settings, Safety, "oneAPI Support" off (`oneAPI=0` in
   `HWiNFO64.INI`).
4. Or, as we did in the end, disable the iGPU (Q2).

**Public sources.** The same `ze_intel_gpu64.dll` crash in HWiNFO 8.60 with an older Intel driver, the oneAPI
workaround, and the author's advice to update the Intel driver; an older thread where Windows Update installed the
bad Intel driver; Microsoft's documentation of the policy:
- https://hwinfo.com/forum/threads/hwinfo64-8-60-6120-crashes-during-initialization-%E2%80%94-intel-level-zero-dll-asus-g16-core-ultra-9-185h.11389
- https://www.hwinfo.com/forum/threads/hwinfo64-crash-on-windows-10-solved.7116
- https://learn.microsoft.com/en-us/windows/deployment/update/waas-wu-settings

<a id="q6"></a>
### Q6. HWiNFO free cannot log from the command line; use its shared memory

**Source.** Starting a sensor log from the command line (`-l`, `-poll_rate`, `-max_time`) needs HWiNFO64 Pro. The
free version also limits the shared-memory interface to 12 hours per session:
- https://www.hwinfo.com/forum/threads/supported-command-line-parameters-in-hwinfo64-pro.8549
- https://www.hwinfo.com/licenses

**What we did (measured).** Set `SensorsSM=1` and read the shared memory (`Global\HWiNFO_SENS_SM2`) with a small
PowerShell logger that writes the values we want to CSV. Two traps:
- **Sensor indices move between HWiNFO restarts.** After a restart our columns came out scrambled. Look each value
  up by sensor and label name on every poll, not by index.
- **Restart the logger after HWiNFO restarts**, and restart it after the 12 hour limit.

On this board, starting HWiNFO also made a separate fan controller lose the Super I/O chip for about 45 s (fans
went to full speed). Plan HWiNFO starts accordingly.

<a id="q7"></a>
### Q7. HWiNFO INI keys we used

`HWiNFO64.INI` next to the exe (writing it needs admin rights), section `[Settings]`:

| Key | What it does |
|---|---|
| `SensorsOnly=1` | start in sensors-only mode |
| `SensorsSM=1` | publish the shared memory (needed for Q6) |
| `MinimalizeMainWnd=1` | start minimised |
| `oneAPI=0` / `1` | Intel Level Zero support (0 avoids the Q5 crash) |
| `PresentMon=0` / `1` | HWiNFO's own frame-time capture |
| `NvmlSupport=0` / `1` | read NVIDIA values through NVML too |

With HWiNFO running, `PresentMon` and `NvmlSupport` on or off made no fps difference (160.0 to 160.8 fps in all
four combinations). Defaults when a key is absent: not verified.

<a id="q8"></a>
### Q8. What HWiNFO measured under GT2 load

2,264 load samples (core clock over 2,800 MHz) on 18 Nov from 12:11 to 16:01, mixed curve settings, most with the
XBAR offset +300 (Q10). Medians (range):

| Value | Median | Range |
|---|---|---|
| GPU clock (requested) | 3,060 MHz | 2,812 to 3,195 |
| GPU effective clock | 3,060 MHz | |
| GPU clock (measured) | 3,076 MHz | |
| Crossbar clock, reported / measured | 2,685 / 2,700 MHz | 2,400 to 2,820 |
| System domain clock | 2,415 MHz | 2,295 to 2,565 |
| Core voltage (NVVDD) | 1.010 V | 0.925 to 1.085 |
| MSVDD output voltage | 1.07 V | 1.07 to 1.09 |
| GPU power | 523 W | up to 608 |
| NVVDD output power / MSVDD output power | 291 W / 98 W | up to 419 / 125 |
| Performance Limit flags: Power / Thermal / Reliability Voltage / Max Operating Voltage / Utilization | 0 / 0 / 0 / 0 / **1** | Power reached 1 at times |

The effective clock matched the requested clock at the median, so there was no sign of clock stretching under
load. Afterburner's aux voltage reading for the same rail sat lower (about 1.04 to 1.055 V); the two tools read
different points.

<a id="q9"></a>
### Q9. "Performance Limit - Utilization" means the GPU was not fully busy

**Source.** HWiNFO's author: the flag means the GPU's engines are not sufficiently utilised, so the clock is held
below its maximum; normal at idle. NVIDIA calls these performance limiters (PerfCap reasons):
- https://www.hwinfo.com/forum/threads/performance-limit-utilization-permanently-on-yes.6150/
- https://skatterbencher.com/nvidia-gpu-boost-performance-limiters/

**What we saw (measured).** The flag stayed at 1 through GT2 runs in the fallback window (Q1), together with
PresentMon showing about 13 % GPU wait per frame. Under a full fullscreen load you would expect it at 0. Use it as a
quick check that the benchmark really loads the GPU.

---

## MSI Afterburner 4.6.7 Beta 5

<a id="q10"></a>
### Q10. "Out of warranty" mode, the crossbar clock and `/wp` writes

**Source.** Beta 5 (build 19412) adds an "out of warranty" voltage control mode that exposes the private XBAR and
SYS clock domains, aux voltage monitoring, a `/rp` `/wp` command line and an extended report. MSI does not support
problems caused by it:
- https://videocardz.com/newz/msi-afterburner-unlocks-hidden-nvidia-gpu-controls-through-out-of-warranty-mode
- https://www.tweaktown.com/news/114056/latest-msi-afterburner-beta-unlocks-xbar-overclocking-on-rtx-gpus-through-new-out-of-warranty-mode/index.html

**How (measured).** Settings, voltage control "out of warranty", which writes `OEM=3` in
`Profiles\MSIAfterburner.cfg`. Then, for example:

```
"C:\Program Files (x86)\MSI Afterburner\MSIAfterburner.exe" /wpXBARCLK_off,300
```

**XBAR offset results (measured, GT2, Profile 1 curve, cap 3,090 MHz, fallback window):**

| XBAR offset | Median XBAR under load | GT2 fps |
|---|---|---|
| +0 | 2,415 MHz | 153.7, 154.0 |
| +100 | 2,505 MHz | 155.8 |
| +200 | 2,602 MHz | 156.9 |
| +300 | 2,707 MHz | 157.8 (+2.6 %) |
| +150 | 2,550 MHz | driver reset after 25 s (at a curve that hung anyway) |

XBAR is fed by MSVDD and ran at 0.81 x the core clock at +0 (2,415 / 2,992). Research put its ceiling at about
0.90 x core; +300 gave 2,707 / 2,977 = 0.91, so +300 is about as far as it goes here. A 12-run and an 11-run series with
+300 were clean (Q18). OEM 0 against OEM 3 made no fps difference (153.7 / 154.4 against 153.9 / 153.6 / 154.5).

**In true borderless fullscreen (measured, 18 Nov 18:14 to 18:27, final Profile 1, ReBAR forced, Q23):**

| XBAR offset | XBAR under load | GT2 fps |
|---|---|---|
| +300 | 2,647 MHz | 175.4 to 175.9 (3 runs) |
| +400 | 2,737 MHz | 176.6 (1 run) |
| +500 | 2,775 MHz | 176.6, then 176.4, 176.9, 176.5 (3 of 3 clean) |

From +400 to +500 the crossbar clock barely moved (about 0.9 x the core clock, the ratio ceiling) and fps did not
change. We set +400 as the daily value, below the ceiling with the same fps. Until 19 Nov a logon task sent it
(`MSIAfterburner.exe /wpXBARCLK_off,400` after Afterburner had started); since then it is written natively, without
Afterburner (Q29).

**Gotcha.** The XBAR offset is not stored in a profile here: a driver reset or a restart clears it, so re-apply it
with the curve after every reset.

<a id="q11"></a>
### Q11. Afterburner's global settings file, and 1 s polling by default

The live global settings are in `Profiles\MSIAfterburner.cfg`, not in `MSIAfterburner.cfg` next to the exe (that one
is a template; edits there did nothing). `HwPollPeriod` defaults to 1000 ms, so a logger reading Afterburner's
shared memory every 100 ms just repeats each 1 s value ten times. Set `HwPollPeriod=100` there while you need
fine telemetry, and back to 1000 for daily use.

Nothing public found on the template file; the polling period is a documented setting.

<a id="q12"></a>
### Q12. Afterburner's shared memory goes stale after an Afterburner restart

If Afterburner restarts while a logger holds `MAHMSharedMemory` open, the new instance does not publish into the
old mapping. The logger keeps reading frozen values (signature `0xDEAD` at offset 0, or a timestamp at offset 20
that stops moving). Two series of our runs, including one hang, have no usable telemetry because of this. Check
both fields and reopen the mapping when either goes stale (the autopilot's `tools/mahmlog.ps1` does).

Nothing public found.

<a id="q13"></a>
### Q13. Afterburner rewrites `[Startup]` when the UI applies settings

On 17 Nov the `[Startup]` section of the card's profile file reverted to an old fixed-offset curve with a 30 %
fan after Afterburner's UI applied settings. After any UI use, check that `[Startup]` matches the profile you
want at boot.

Since 19 Nov Afterburner no longer starts with Windows on this machine (Q29), so this only matters to you if
Afterburner applies your profile at boot.

<a id="q14"></a>
### Q14. The voltage slider and `NVVDD_off_Vrel` do nothing on driver 621.x

**Source.** Driver 621.01 and later ignore runtime NVVDD and MSVDD voltage offsets on RTX 50 cards: the call
reports success and nothing changes. The Vix Editor's `runtime_voltage_offsets=1` adds hooks for a runtime tool
(mVolt+) to apply offsets again:
- https://github.com/Panchovix/nvidia-vbios-vix-editor

**What we saw (measured).** Afterburner's voltage slider and the out-of-warranty `/wpNVVDD_off_Vrel` and
`/wpMSVDD_off_Vrel` writes changed nothing on 621.24. We built an image with `runtime_voltage_offsets=1` (loaded
and verified 18 Nov 15:03); mVolt+ offsets were still ignored on it (Q27). To test "more voltage at the same
clock" we moved the whole curve up the voltage axis instead (P1 v1 curve +25 and +50 mV): no hangs in one run
each, but those runs were in the fallback window (Q1), so they decide nothing. What did raise the voltage limit was
a static image field, the NVVDD REL floor (Q27). Read through the driver's own interface on 19 Nov, the voltage
boost value was 0 (Q29).

<a id="q15"></a>
### Q15. RTSS needed a reinstall after installing Afterburner Beta 5

After Beta 5 went in over an older beta, RTSS showed an error popup on start until RTSS itself (7.3.7) was
reinstalled. Similar advice from the Afterburner author's forum for a different RTSS load error after an update
(uninstall both, install the current builds):
- https://forums.guru3d.com/threads/after-latest-windows-10-update-i-get-a-direct3d9-loading-error-from-rivatunerstaticsserver-when-launching-afterburner.461368

<a id="q16"></a>
### Q16. The extended report (`MAKEREPORT`): what is in it

`MSIAfterburner.exe /wpMAKEREPORT,8,<path>.txt` (out of warranty mode) writes a read-only report. Ours, taken on
18 Nov with the 700 W override image loaded (REL ceilings 1,150 mV, MSVDD REL floor 1,100 mV):

| Part | Our card |
|---|---|
| GPU binning | SPEEDO 2109 (fuse range shown as 2018.94 to 2284.87), OC bin 0, IDDQ NVVDD 42300, IDDQ MSVDD 14050 |
| NVVDD rail | Vmin 800 mV, **Vrel 1,120 mV**, Vop 1,140 mV, Vov 1,200 mV, offsets +0/+0/+0/+0 mV |
| MSVDD rail | Vmin 870 mV, **Vrel 1,070 mV**, Vop 1,140 mV, Vov 1,200 mV, offsets +0/**-50**/+0/+0 mV |
| Clock domains | GPCCLK, XBARCLK, MCLK (17,001 MHz = memory +3000), SYSCLK, MSDCLK, PWRCLK, PCIEGENCLK, DISPCLK, HUBCLK |
| Clock propagation | XBARCLK drives SYS, MSD, PWR, HUB and DISP clocks on MSVDD; GPCCLK to XBARCLK ratio relations |
| V/F relations | GPCCLK on NVVDD (points 0 to 126, up to 3,450 MHz); XBAR, SYS, MSD, PWR on MSVDD |
| Also | every V/F point, fuse values, reliability limit entries, thermal and power policies |

Reading:
- The NVVDD Vrel of 1,120 mV matches the highest core voltage we ever saw under load (1.115 to 1.120 V). Our
  earlier note that "the chip's REL is about 1,070 mV" mixed up the rails: 1,070 mV is MSVDD's.
- MSVDD shows a -50 mV offset, which fits NVIDIA's MSVDD change from driver 572.60 (Vix Editor changelog). With
  the 1,100 mV MSVDD floor image loaded, HWiNFO still read MSVDD at 1.07 to 1.09 V under load (Q8), so the floor
  did not lift MSVDD to 1.10 V. Why is open.
- SPEEDO is a per-chip speed fuse; 2109 sits about a third of the way into the range shown
  ((2109 - 2019) / (2285 - 2019) = 0.34). We have no public table of 5090 SPEEDO values to compare with. The
  heavy-load wall we found (about 3,090 MHz in GT2, Q18) is in line with the 3.1 to 3.15 GHz that owners commonly
  report as game-stable (estimate from scattered owner posts, for example
  https://www.overclock.net/threads/revisited-rtx-5090-msi-vanguard-soc-le-aorus-master-vbios-comparison-stock-vs-2-undervolts-vs-overclock-on-4-synthetics-and-3-games.1815273
  and the review roundup at https://shattered.io/rtx-5090-overclocking-guide), so this looks like an ordinary
  chip, not a weak one.

---

## Driver and tuning behaviour

<a id="q17"></a>
### Q17. A driver reset wipes the curve, memory offset, clock cap and XBAR offset

After a driver reset (event 4101), the card runs at stock until settings are applied again. A test step after a
reset looks like a pass at stock clocks; treat a run whose voltage is far above the target as "settings lost".
Re-apply profile, memory offset, `nvidia-smi -lgc` cap and `/wpXBARCLK_off` after every reset (a task on event
4101 works for the cap). A memory clock reading of 14,001 MHz after a reset (stock) tells you the offset is gone.

Since 19 Nov a tray tool re-applies the power limit, memory offset, XBAR offset and curve natively at startup and
15 s after a driver reset (Q29). The reset path has not yet been exercised by a real reset.

<a id="q18"></a>
### Q18. The GT2 hang is a heavy-load clock wall; a soft cap in the curve fixed it

Everything here is Time Spy Extreme Graphics Test 2, direct launches. Until 18 Nov 16:23 every run was in a window
(Q1); everything from 16:29 on is true borderless fullscreen. Full narrative: README section 8.

- **Signature.** A heavy scene at about 3,045 to 3,150 MHz and 1,000 to 1,045 mV, 550 to 600 W; power then
  collapses to about 120 to 155 W while usage stays 100 %; the clock jumps to the top of the curve because power
  fell; the driver resets seconds later. The jump is a symptom, not the cause.
- **In the window (before 16:23):** uniform shifts of minus 45 and minus 60 MHz and a per-band curve ("P1 v1")
  ran clean 12 of 12 and 11 of 11; `-lgc` caps at 3,060 and 3,090 hung. None of that carried over to fullscreen.
- **In fullscreen, per-band offsets were a dead end.** Lowering the 1,006 to 1,030 mV band from -60 to -120 MHz
  (-135 was next) never stopped the hangs. They kept landing at 3,045 to 3,090 MHz while the voltage read at the
  hang rose with each step (1,000, 1,010, 1,015, 1,020 mV): lowering one band only moved the same clocks to a
  higher voltage point. On the way we found a method flaw (Q25).
- **What worked: a soft cap.** The Profile 1 curve with every point up to 1,060 mV held flat at a cap, points
  above untouched. Ladder, 3 runs each: 3,045, 3,060, 3,075 and 3,090 MHz clean; 3,105 hung on run 2 at 1,020 mV /
  3,090 MHz. Final pass: cap 3,090 = 6 clean, 1 hang in 7 (Profile 1, at the edge); cap 3,075 = 6 of 6
  (Profile 2). Same fps as without a cap, about 173 in GT2.
- **Why it is different from an `-lgc` cap (estimate).** An `-lgc` cap clips every point, the top too, and those
  caps hung in the window (one at 2,932 MHz and 960 mV). The soft cap leaves the curve above 1,060 mV alone, so in
  light moments the card still runs above 3,100 MHz at higher voltage: in the final Profile 1 runs, 5 % of the
  heavy samples (usage at least 95 %, over 300 W) were above 3,100 MHz at about 1.02 V and 536 W, the rest at
  about 3,064 MHz, 1.015 V and 570 W. Our reading: a clock wall under the heaviest load, plus a flat stretch that
  keeps the driver from hopping between V/F points during load transients. Not proven.
- **Power limit is a ceiling, not a load.** GT2 drew about 460 to 610 W; raising the limit from 600 to 700 W
  changes nothing unless the card was at the limit.

<a id="q19"></a>
### Q19. Run counts and what they can prove

Hangs here are random events, so count runs.

| Clean runs in a row | Hang rate ruled out (95 %, one-sided) |
|---|---|
| 8 | above 31 % |
| 12 | above 22 % |
| 16 | above 17 % |
| 60 | above 5 % |

(Exact bound 1 - 0.05^(1/n); the "rule of three", 3/n, is close above 30 runs.) 8 runs cannot tell a 5 in 8 hang
rate from 3 in 8. Our method since 18 Nov: **1 run per screening step, 3 runs to keep a step, 10 to 12 only for
the final pass**, and a longer run count only for the final profile.
- https://en.wikipedia.org/wiki/Rule_of_three_(statistics)
- https://www.jhanley.biostat.mcgill.ca/c607/ch08/zero_numerator.pdf

<a id="q20"></a>
### Q20. Two harnesses at once give nonsense

On 18 Nov 11:00 to 11:12 two scripts launched 3DMark at the same time. Every run scored about 88 to 97 fps (against
about 154). Run one chain at a time, and refuse to start if a `3DMarkTimeSpy.exe` is already running. The runs are
in the data, marked INVALID.

<a id="q21"></a>
### Q21. Other loads do not find the 3D limit

CUDA checks find limits 30 to 110 MHz above what 3D survives; FurMark overstates the limit (a hot core drops
temperature bins and never runs the clock it was given); FurMark load steps, a CUDA load ramp, OCCT 3D Adaptive
(which never passed about 3,020 MHz here) and the Steel Nomad Stress Test did not reproduce the GT2 hang. Keep a real
3D load that reaches the hang clocks in the plan.

<a id="q22"></a>
### Q22. Smaller things

- **`nvidia-smi -lgc F,F` does not hold F** in heavy 3D on Blackwell: about 100 MHz under. Use `-lgc 210,CAP` as a
  ceiling only.
- **The DXGI adapter LUID changes every boot.** 3DMark workloads name the GPU by it; a stale one makes the launch
  fail in a way that looks like a hang.
- **Displays must stay awake.** A sleeping display aborts 3DMark ("window_focus_lost: Display resolution changed").
  Dim them instead.
- **A BitLocker data drive can stay locked after a restart**, so 3DMark on it cannot start: check the exe exists
  before counting results.
- **Memory +3000 is worth about 1.4 % in GT2** here (cap 3,090: 151.6 fps at +0, 152.1 to 152.8 at +1000, 153.6 to
  153.9 at +2000 to +3000).
- **Stock curve vs Profile 1 at the same 3,090 MHz cap**: 147.8 to 148.0 against 153.1 fps, because the stock curve
  reaches 3,090 only near its top voltage and runs lower clocks in heavy scenes.
- **`pkill -f PATTERN` can kill your own shell** when the pattern is in that shell's command line.
- **Check the GPU is idle** before any unattended test (`nvidia-smi --query-gpu=utilization.gpu,power.draw`).

<a id="q23"></a>
### Q23. Forcing Resizable BAR for 3DMark Time Spy: +1.2 % in GT2

**Source.** ReBAR on in the BIOS is not enough on NVIDIA cards: the driver enables it per application through its
profile database, and 3DMark Time Spy is not on the list. NVIDIA Profile Inspector (NPI) can force it per profile;
results vary per game, so test each and do not set it globally:
- https://nvprofileinspector.com/rebar/
- https://www.xda-developers.com/nvidia-disabling-resizable-bar-in-games-even-though-bios-says-its-on
- https://babeltechreviews.com/z390-resizable-bar-performance/

**What we did (measured, 18 Nov 18:04 to 18:12).** In NPI, profile "3DMark TimeSpy" (`3dmarktimespy.exe`),
`rBAR - Enable` (0x000F00BA) = 1, size limit left empty, applied with `nvidiaProfileInspector.exe -silentImport`.
Final Profile 1, XBAR +300, borderless fullscreen:

| State | GT2 fps |
|---|---|
| driver default | 173.5, 173.0 |
| ReBAR forced | 175.5, 175.3 |
| off again (control) | 173.1 |

+1.2 % (173.3 to 175.4 on average). Profile 1 with ReBAR forced then ran 3 of 3 clean at 175.4 to 175.9, so the
extra load did not show a new hang. Driver updates can reset NPI settings; check after each one.

**Other 3DMark tests: the .nip import silently does nothing (measured, 18 Nov).** Importing new profiles named
"3DMark Fire Strike", "3DMark Port Royal" and "3DMark Steel Nomad" with `-silentImport` exited 0, but the export
showed no change. The executables already belong to predefined NVIDIA profiles with other names, and an executable
can only sit in one profile. Asking the driver which profile owns each executable (NVAPI DRS,
`NvAPI_DRS_FindApplicationByName`) gave:

| Executable | Predefined driver profile |
|---|---|
| `3DMarkICFWorkload.exe`, `3DMarkICFDemo.exe` (Fire Strike) | 3DMark Ice Storm |
| `3DMarkPortRoyal.exe` | 3DMark Port Royal DLSS |
| `3DMarkSteelNomad.exe` | 3DMark - Steel Nomad |
| `3DMarkTimeSpy.exe` | 3DMark TimeSpy |

Setting `rBAR - Enable` = 1 on those profiles through NVAPI DRS (`NvAPI_DRS_SetSetting`, then
`NvAPI_DRS_SaveSettings`) worked, and an NPI export confirmed all four. The gain on those tests is not measured yet.

<a id="q24"></a>
### Q24. One GPU in the system: Afterburner drops the `GPU1 ` prefix

**What we saw (measured).** With the iGPU disabled (Q2), Afterburner's monitoring sources in its shared memory and
in `Sources=` (`Profiles\MSIAfterburner.cfg`) lost their `GPU1 ` prefix: `Core clock`, `GPU voltage`, `Power`,
`GPU usage`, `GPU temperature`, `Crossbar clock`, `System clock`, `Aux voltage`, `Memory clock`. Note the core
voltage is `GPU voltage`, not `Voltage`. A logger keyed on `GPU1 voltage` wrote empty columns without any error.

**Remedy.** Look up the `GPU1 ` name first and fall back to the plain name (the autopilot's `tools/mahmlog.ps1` and
`tools/mahm1.ps1` do), and tick any newly named graph in Afterburner's Monitoring tab so it is published. Start a
new logger in a separate PowerShell call from the one that stops the old one; doing both in one call left the new
logger hung without writing.

Nothing public found.

<a id="q25"></a>
### Q25. Per-band offsets can make the curve non-monotonic

**What we saw (measured, 18 Nov 16:29 to 17:00).** With per-band offsets relative to Profile 1, raising a lower
band while lowering the band above it (981 to 1,005 mV at +30, 1,006 to 1,030 at -60) put lower-voltage points
above higher ones (for example 1,000 mV at about 3,150 MHz against 1,010 mV at 3,067). The driver then picks
clocks that do not follow the bands, and a hang gets blamed on the wrong band.

**Remedy.** After applying band offsets, cap every point at the lowest clock of any higher-voltage point (a suffix
minimum), so the curve never falls as voltage rises. The autopilot's `abcurve.monotonic()` does this. With the fix
the per-band search still did not converge (Q18), but its results became readable.

<a id="q26"></a>
### Q26. The voltage read at a hang sits about 10 mV under the V/F point in use

**What we saw (measured).** In the 100 ms Afterburner log, the voltage of the last heavy sample before a hang read
about 10 mV under the V/F point the card was running under load. So a hang read at 1,020 mV near a band boundary
can belong to the band above. Attribute such hangs with that in mind, or better, avoid band attribution and test
whole curve shapes (Q18).

<a id="q27"></a>
### Q27. mVolt+ runtime voltage offsets are ignored on driver 621.24; the NVVDD REL floor works

**What we saw (measured, 18 Nov 19:20).** mVolt+ v0.50.13, `--nvvdd-offsets 0,25,25`, driver 621.24, with a
Vix-loaded image that has `runtime_voltage_offsets=1` (Q14). The call returned `"success": true`, and the state it
returned in the same reply said `"rail_offsets_ignored_by_driver": true`; `nvvdd_max_mv` stayed 1,120 and the
NVVDD offsets read back as 0. So the hooks from `runtime_voltage_offsets=1` did not get the offsets through on
this card and driver.

**Remedy (measured).** Raise the limit in the image instead: Vix Editor, `nvvdd_rel_equation_min_mv` 900 -> 1,150
(only that field changed, 101 bytes), load with the Vix Loader, restart. The REL voltage is computed per chip (here
1,120 mV) and clamped by the REL equation floor (our reading of the field names, borne out by the result), so a
floor above the computed value wins. After the restart mVolt+ `--status` read `nvvdd_max_mv` 1,170 (REL 1,150 plus
the ALT +20 step), and the running image matched the stored one byte for byte. Under load the voltage tops out at
about 1.135 V read (about 1.145 V set, the REL 1,150 limit); the ALT +20 is not used. Only light loads reach it:
4K TSE GT1 at about 540 W stayed at or under 1.02 V. Results: README
[Voltage lever](README.md#voltage-lever).

Nothing public found on the offsets being ignored with `runtime_voltage_offsets=1` set.

<a id="q28"></a>
### Q28. A higher voltage limit moves heavy loads up the curve too

**What we saw (measured, 19 Nov).** After the REL 1,150 image (Q27) raised the core limit from 1,120 to 1,170 mV,
heavy loads no longer stayed at 1.015 to 1.035 V. A Fire Strike Ultra run ran heavy scenes at 3,180 to 3,217 MHz
and 1.065 to 1.08 V: on the curve's step from 3,090 MHz (soft cap, up to 1,060 mV) to 3,247 MHz at 1,065 to
1,070 mV. With the 1,120 mV limit heavy loads had never got there. Two driver resets followed in heavy scenes
(19 Nov 00:06 and 09:28, about 610 to 620 W), last heavy samples 3,030 to 3,090 MHz at 1.03 to 1.055 V, then the
usual power collapse. The evening before, one 4K TSE GT1 run at or under 1.02 V had suggested heavy loads would
not use the new top; one load did not represent every heavy load.

**Remedy.** Extend the soft cap to where heavy loads can now go and ramp the top above it. Here: every point up to
1,100 mV at the wall (3,090 MHz Profile 1, 3,075 Profile 2), a straight ramp to 3,440 / 3,380 MHz at 1,150 mV, flat
above. Result: TSE GT2 3 of 3 clean (176.6, 176.1, 176.2 fps, max 1.085 V, heavy load held at the wall), light 720p
clean at 1.135 V, Fire Strike Ultra graphics 37,457 (+1.3 %), valid. After any change to the top of the curve,
re-test a heavy load and log its voltage.

Nothing public found.

<a id="q29"></a>
### Q29. The overclock can be applied natively, without Afterburner; a memory write zeroes the curve

**Why we looked.** An Afterburner profile does not store the XBAR offset (Q10) and a driver reset drops every
setting (Q17), so the daily setup needed Afterburner at logon plus two tasks. We wanted one small tray tool and no
resident tuning app.

**What we saw (measured, 19 Nov, about 15:00 to 15:25, driver 621.24).** The driver's own library, `nvapi64.dll`,
takes all four settings through private interfaces (the ones tuning apps use).

Read only, first:

- The private curve table held the Afterburner Profile 1 offsets on 127 of 127 core points.
- The power limit read 121.739 % of the default (700 W).
- The clock domains table read 400000 kHz for XBAR (domain 1, byte offset `0x53C`), the live +400. That layout had
  been validated in public only up to driver 610.88.

Then written, each step read back:

- Curve and power limit written back unchanged: accepted, read back identical.
- XBAR 400 -> 300 -> 400 MHz: each value read back, curve untouched.
- Memory +3000 through SetPstates20: accepted, and it **zeroed all 127 core curve offsets**. The XBAR offset
  stayed.
- Curve restored natively: read back identical. No driver reset, no nvlddmkm or 4101 event. A light 720p run
  afterwards showed the clocks of the applied profile: core 3,270 MHz at 1.135 V, XBAR 2,835 MHz, memory
  17,001 MHz.

**Quirks inside the quirk:**

- **A memory offset write zeroes the curve** (measured; also reported by the vfctl write-up below). Order: power
  limit, memory, XBAR, curve last, then read the curve back.
- **Afterburner's memory offset did not show in the Pstates20 read** (measured): the offset read 0 while the memory
  clock read 17,001 MHz (+3000). After the native write the same read returned 3000000 kHz. So "offset 0" there
  does not prove the memory is at stock; check the memory clock.
- **No voltage this way** (measured): the voltage boost value reads 0, and runtime Vrel offsets (mVolt+ style) stay
  ignored on 621.x (Q14, Q27).
- **Writes need admin, reads do not**: an unelevated write is refused with -137 (as recorded in the blackout-tray
  notes; -137 is NVAPI's "invalid user privilege").
- **A zeroed buffer is silently ignored** (as recorded in the blackout-tray notes): read the table, change it,
  write it back.

**Remedy / result.** [blackout-tray](https://github.com/Xipzer/blackout-tray) holds both final profiles (cap 3,090
and cap 3,075) and the XBAR offset, and applies them at startup, when the profile is switched and after a driver
reset. The reset is noticed through a System event log subscription; **that path has not yet been exercised by a
real driver reset.** Afterburner's start with Windows is disabled; it stays installed as the fallback and for
monitoring during tuning sessions. Layouts, interface ids and setup:
[docs/gpu.md](https://github.com/Xipzer/blackout-tray/blob/main/docs/gpu.md). The story with all numbers: README
[Without Afterburner](README.md#without-afterburner-19-nov).

**Caveats.** Private interfaces that NVIDIA does not document: a future driver can change or block them. Tested on
one card (RTX 5090 FE) and one driver (621.24).

**Source.** The layouts are not ours. They come from these projects and write-ups, checked again on our card:
- https://github.com/SHANAjam/rtx5090-xbar-control (MIT): XBAR and the clock domains table
- https://github.com/arcnmx/nvapi-rs (MIT) and https://vuplea.github.io/simple-nvidia-undervolt/DEVELOPMENT.html:
  curve tables
- https://erfi.dev/reference/vfctl-gpu-curve-tool/: memory through Pstates20 and the curve reset it causes
- https://www.overclock.net/posts/29608363/: the Afterburner developer naming the interfaces

We did not search for other reports of Afterburner's memory offset reading 0 in Pstates20.

---

## Clean measurement checklist

Before you trust a number from a GPU stability or fps test:

1. **One display adapter.** Every monitor on the NVIDIA card; iGPU disabled in Windows and in the BIOS (Q2).
2. **Fullscreen on one display.** For 3DMark direct launches: `is_windowed: false`, `is_borderless: true`,
   `swapchain_resolution` = native size of the target display (Q1).
3. **Check the window during the run.** Rectangle = display size, style without `WS_CAPTION` (`0x94000000` is
   good, `0x14CF0000` is a window).
4. **Check the GPU is fed.** PresentMon GPU wait near zero; HWiNFO "Performance Limit - Utilization" at 0 (Q9).
5. **Check the clocks are real.** HWiNFO effective clock close to the requested clock; memory clock not 14,001 MHz
   (settings lost, Q17); XBAR offset re-applied if you use one.
6. **One harness at a time**, nothing else using the GPU (Q20).
7. **Same session for comparisons.** Alternate A and B within one session; do not compare fps across restarts or
   desktop changes.
8. **Same driver profile state**: ReBAR forced or not for the benchmark (Q23) and the XBAR offset applied (Q10).
9. **Count runs** (Q19), and log at 100 ms with Afterburner `HwPollPeriod=100` (Q11) and a logger that survives
   an Afterburner restart (Q12).
10. **Record the state with each result**: water temperature, VBIOS image, driver, display mode, monitors.
