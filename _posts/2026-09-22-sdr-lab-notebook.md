---
layout: post
title: "SDR Lab Notebook"
categories: [tech]
description: >
  Software-defined radio notes — what SDR actually is, the HackRF Pro and the
  Nooelec NESDR Mini 2+ side by side, the SMA vs RP-SMA antenna trap, Mac setup,
  spectrum sweeps, Wi-Fi channel maps, and project ideas.
---

SDR, software defined radio, has always interested me. Here is a guide put together for sdr in general, and some specific information for the HackRF Pro and the NooElec R820T2 SDR & DVB-T NESDR Mini 2+.
{:.lead}

These notes are written against my own bench — a HackRF Pro and an RTL-SDR dongle driven from an M1 MacBook — so the specifics are mine, but the concepts and commands are general. It's a living document; I'll keep adding to it.

* this list will be replaced by the table of contents
{:toc}

## What SDR actually is

A software-defined radio is generic RF hardware plus software that does everything else. The board just moves RF ↔ raw IQ samples; modulation, demodulation and protocol decoding all happen in software on the host. That's the whole idea — and it's why one little box can do FM, spectrum analysis, ADS-B and more.

* **Pure RF, no protocol smarts.** The HackRF has zero knowledge of Wi-Fi, FM or ADS-B. Host software (GNU Radio, gqrx, dump1090, rtl_433, URH…) turns the raw IQ into audio, bits or messages.
* **Instantaneous bandwidth ≈ 20 MHz.** The HackRF digitises only ~20 MHz at any instant (20 MS/s). Everything wider is a trick of *time*.
* **Receiver vs sweeper — parked vs swept.** A *receiver* (gqrx) parks that 20 MHz window on one spot and shows it live, so it can demodulate and listen. A *sweeper* (`hackrf_sweep`) rapidly re-tunes the window across a wide range and stitches the pieces together — broad coverage, but time-sliced, so fast bursts can slip by. Same window, different use.

## The gear

### HackRF Pro (Great Scott Gadgets, 2025)

| | |
|---|---|
| Frequency | **100 kHz – 6 GHz** operating (tunable 0 Hz–7.1 GHz) — a wider low end than the HackRF One's 1 MHz–6 GHz |
| Sample rate | up to **20 MS/s**, plus an extended-precision **16-bit** mode at low rates (the One is 8-bit only) |
| Duplex | **Half-duplex** — RX *or* TX, never both at once |
| Connector | SMA on the `RF/ANT` port; USB-C to the host |

> **Self-test "Loopback FAIL" is a known Pro firmware quirk.** `hackrf_info` on a new Pro often reports **Self-test FAIL → Loopback: FAIL** (r1.2+, firmware 2026.01.x). This is documented on the Great Scott Gadgets GitHub ([#1646](https://github.com/greatscottgadgets/hackrf/issues/1646), [#1696](https://github.com/greatscottgadgets/hackrf/issues/1696)) and is **not** a broken unit — RX works fine, which you can prove with a sweep. Reset with `hackrf_spiflash -R` if it nags; it only matters for TX.

### Nooelec NESDR Mini 2+ (R820T2 tuner + RTL2832U)

| | |
|---|---|
| Tuner | **R820T2** (Rafael Micro) — the gold-standard *RX-only* RTL-SDR tuner, originally optimised for TV reception and now dominant in weak-signal work |
| Demodulator | **RTL2832U** (Realtek) — the USB DVB-T dongle chip; handles IF → baseband and outputs raw IQ to the host |
| Frequency | **~25 MHz – 1.7 GHz** in practice (spec claims 22–2200 MHz; dead zones near DC) |
| Sample rate | up to **3.2 MS/s** (~2.4 MS/s stable), **8-bit** IQ samples |
| Bandwidth | ~2.4 MHz, fixed by the crystal — no programmable bandwidth |
| Duplex | **RX only** — no transmit |
| Connector | SMA female port, expecting SMA *male* antenna plugs |
| Power | Bus-powered USB 2.0, no external supply |

#### Why bother with the R820T2 alongside a HackRF?

* **It's the dirt-cheap wideband receiver.** At roughly $25–40 it's a fifth of the HackRF's price, and RX-only is fine for most scanning: ADS-B, rtl_433, weather, spectrum snapshots.
* **Weak-signal hero.** The R820T2's LNA is excellent. Paired with a decent antenna it can outperform the HackRF on weak VHF/UHF signals — plenty of ADS-B feeders prefer RTL-SDR for distant aircraft.
* **Stiffer filtering.** The crystal-locked ~2.4 MHz bandwidth rejects adjacent-channel interference better than the HackRF's wider configurable filter, which means better SNR in busy bands.
* **They pair well.** Run ADS-B or rtl_433 on the dongle — plug it in and leave it — while the HackRF does wideband sweeps.

> **Trade-off: narrow bandwidth means longer sweeps.** A 2.4 MHz window makes whole-band scans roughly 4× slower than the HackRF, which covers ~20 MHz per step. Use the dongle for *fixed-frequency monitoring* or quick spot checks of narrow bands (ADS-B at 1090 MHz, say). For broad spectrum work the HackRF's wider window wins. Both are receivers, so run them in parallel on different USB ports if you like.

### The antenna kit — what each one is for

No single antenna covers everything; each is cut for a band. The rule of thumb: **SDR antennas are plain SMA and screw straight on, Wi-Fi antennas are RP-SMA and need an adapter.**

| Antenna | Best for | Connector |
|---|---|---|
| ANT500 (ships with HackRF) | 75 MHz–1 GHz telescopic — FM, air band, VHF/UHF, ISM. Extend to roughly a quarter-wave for the target frequency | SMA — direct |
| RaTLSnake telescopic mast | ~25 MHz–1 GHz+ depending on length — the widest low-band reach: FM, air, VHF/UHF, weather | SMA — direct |
| RaTLSnake DVB-T/T2 stubby (blue bands) | ~700–1200 MHz — broadcast TV, GSM, pagers, and **ADS-B aircraft at 1090 MHz** | SMA — direct |
| RaTLSnake helical | ~1100–1800 MHz — L-band, GSM/PCS, upper UHF, and light | SMA — direct |
| Tri-band Wi-Fi 6E paddle | **2.4 / 5 / 6 GHz** — the *only* one that hears Wi-Fi 6E | RP-SMA — adapter |
| 2.4/5 AP antenna | 2.4 / 5 GHz only — no 6 GHz | RP-SMA — adapter |

> **The Nooelec RaTLSnake M6 v2 whip set** gives you a magnetic base with 2 m of RG-58 ending in **SMA male** (standard SMA, 50 Ω), so it screws straight onto the HackRF with no adapter. Three swap-on masts cover roughly 25 MHz to 1.8 GHz between them, and the magnet lets you stick it to a car roof or filing cabinet for a better ground plane. **None of them reach 2.4 GHz and up** — Wi-Fi work still needs the tri-band antenna.

#### SMA vs RP-SMA, and why you keep the adapter

This trips up everyone once. Threads name the gender — male means outer threads — and "RP" (reverse polarity) just swaps the centre pin for a hole. So:

| Connector | Centre | Threads | Where you see it |
|---|---|---|---|
| SMA male | **pin** | outer | Your SDR antenna plug (ANT500, RaTLSnake) |
| SMA female | **hole** | inner | **The HackRF's port** |
| RP-SMA male | **hole** | outer | Your Wi-Fi antenna plug |
| RP-SMA female | **pin** | inner | A Wi-Fi card's jack, or the adapter's antenna side |

SMA male (pin) into the HackRF's SMA female (hole) mates correctly. A Wi-Fi antenna is RP-SMA male, which is a **hole** — put that on the HackRF's female port and it's hole-to-hole, threading on happily while making no electrical contact at all. That's why it needs an **RP-SMA-female → SMA-male adapter**. Tri-band 6E antennas are built for Wi-Fi cards, so they're *all* RP-SMA; hunting for a "plain SMA" 6E antenna is a dead end. Keep the adapter permanently screwed onto the Wi-Fi antenna and treat them as one piece.

## Setup on a Mac

Native USB-C, no adapter needed — the Mac is the easy place to drive the HackRF.

```bash
# install (one-time)
brew install hackrf gqrx                         # tools + receiver GUI
pipx install --backend pip qspectrumanalyzer     # whole-band waterfall (drives hackrf_sweep)
```

```bash
# verify: serial, board ID, firmware
hackrf_info      # expect Board ID 5 for the Pro; loopback FAIL is the known quirk, RX still works
```

### RTL-SDR dongle setup

The RTL-SDR is **plug-and-play on macOS** — no driver install, since libusb and the kernel DVB-T drivers are already there. You only need the user-space tools:

```bash
brew install rtl-sdr        # librtlsdr CLI tools: rtl_fm, rtl_test, rtl_power…
brew install dump1090       # ADS-B decoder (aircraft tracking)
brew install rtl_433        # ISM sensor decoder (433 / 868 / 915 MHz)
```

```bash
rtl_test -t      # quick enumerate test; should list the RTL2832U tuner
```

### Running both radios at once

Both are RX-only or half-duplex, so there's **no conflict** — plug them into different USB ports and run separate apps.

```bash
# terminal 1: HackRF sweeping the whole 2.4 GHz band
hackrf_sweep -f 2400:2500 > sweep.csv
```

```bash
# terminal 2: dongle parked on ADS-B
dump1090 --device-index 1
```

```bash
# or: dongle listening to 433 MHz sensors
rtl_433 -d 1 -f 433920000
```

> **Device indexing.** With both radios plugged in, the OS assigns indices by enumeration order. Use `rtl_test -t` to see RTL-SDR devices and `hackrf_info` for the HackRF. In `rtl_433`, `dump1090` and `rtl_power`, pass `-d 0` or `-d 1` to pick. In gqrx, the device dropdown lists everything detected — select by name (e.g. "RTL2832 DVB-T").

## Command cheat sheet

### HackRF

| Command | What it does |
|---|---|
| `hackrf_info` | Identify board, serial and firmware; run the self-test |
| `hackrf_sweep -f LO:HI` | Wideband spectrum scan from LO to HI MHz, as CSV. `-1` does one shot; Ctrl-C stops a loop |
| `hackrf_transfer` | Core RX/TX — capture raw IQ (`-r`) or transmit a file (`-t`); `-f` frequency, `-s` sample rate |
| `hackrf_biast` | Bias-tee: feed DC up the coax to power an external LNA or active antenna |
| `hackrf_spiflash` | Firmware flash; `-R` resets the device |

### RTL-SDR

| Command | What it does |
|---|---|
| `rtl_test -t` | Enumerate devices and do a quick health check — look for `RTL2832U` or `R820T` |
| `rtl_fm -f FREQ -M WFM -s RATE` | Tune to FREQ in Hz, demodulate wide FM, output to stdout (pipe to a player for audio) |
| `rtl_power -f LO:HI -g GAIN` | Spectrum sweep as compact CSV — fast data collection, not streaming |
| `dump1090 --device-index N` | ADS-B aircraft decoder; web UI on `localhost:8080` |
| `rtl_433 -d N -f FREQ` | ISM sensor decoder for weather stations, thermometers, door sensors |

### Both

| Command | What it does |
|---|---|
| `gqrx` | GUI receiver — tune one frequency, watch the waterfall, demodulate. Detects both radios |
| `qspectrumanalyzer` | Whole-band spectrum and waterfall, using the `hackrf_sweep` backend |

## The wider tool ecosystem

Beyond the basics, these extend things into protocol decode, custom flowgraphs and signal forensics.

### Core infrastructure

| Tool | Install | Purpose |
|---|---|---|
| **libhackrf** | `brew install hackrf` | C library plus Python bindings for HackRF control — tune, capture raw IQ, set gain from your own scripts |
| **SoapySDR** | `brew install soapysdr` | A unified abstraction layer: one API for HackRF, RTL-SDR, USRP and others. Most modern SDR apps use it |
| **soapyhackrf** | `brew install soapyhackrf` | The SoapySDR driver for HackRF — what lets gqrx, CubicSDR and GNU Radio see the device |

### Fixed-frequency monitoring

| Tool | Purpose |
|---|---|
| **rtl_433** | Decodes ISM sensors: weather stations (433.92 MHz), tyre-pressure monitors (315 MHz US / 433 MHz EU), thermostats, door sensors. Supports 100+ device types |
| **dump1090** | ADS-B aircraft tracking — decodes Mode S at 1090 MHz into position, call sign, altitude and speed, with a web UI |
| **readsb** | A modern fork of dump1090-fa with better decoding of noisy frames and network sharing. Drop-in replacement |

### Signal capture and protocol analysis

| Tool | Purpose |
|---|---|
| **inspectrum** | Offline visual signal forensics. Load a raw IQ file captured with `hackrf_transfer -r`, then view the spectrogram, zoom in time and frequency, measure timing and identify modulation |
| **Universal Radio Hacker (URH)** | Capture, visualise and reverse-engineer digital protocols. The GUI walks you from raw IQ through spectrogram analysis to an extracted bit stream and a guess at the protocol structure |

### Custom flowgraphs and scripting

| Tool | Purpose |
|---|---|
| **GNU Radio** | Block-based DSP programming. GNU Radio Companion is a visual editor — drag sources, filters, demodulators and sinks, then wire them together. Start from the bundled examples before writing Python |
| **Python + scipy** | Lower-level custom processing: load an IQ file, filter, detect peaks, plot a spectrogram. Good for batch analysis rather than real-time work |

### Alternative receiver UIs

| Tool | Compared to gqrx |
|---|---|
| **gqrx** | The reference receiver — the most modes (WFM, AM, NFM, LSB, USB, CW), stable and well documented. Best for learning, since every control is visible |
| **CubicSDR** | Modern and touchscreen-friendly, with nicer waterfall rendering but fewer modes out of the box |
| **SDR++** | Lightweight, modular and very fast. Excellent for spectrum monitoring and rapid band switching |

### Which tool for which job

| Task | Tool | Example |
|---|---|---|
| Whole-band sweep | hackrf_sweep or QSpectrumAnalyzer | `hackrf_sweep -f 2400:2500 -1` |
| Single-channel zoom | gqrx or CubicSDR | gqrx at 2437 MHz, Mode = Off |
| Listen to FM radio | gqrx | 96.5 MHz, Mode = WFM, unmute |
| Track aircraft | dump1090 or readsb | `dump1090 --device-index 0` |
| Decode weather / door sensors | rtl_433 | `rtl_433 -d 0 -f 433920000` |
| Study one captured signal | inspectrum | Open a `.iq` file, explore the spectrogram |
| Reverse-engineer a remote | URH | Capture → spectrogram → extract bits → decode |
| Custom modulation or demod | GNU Radio | Wire blocks: source → filter → sink |

## Spectrum analysis and Wi-Fi channel mapping

This is the killer use for Wi-Fi work: seeing the *raw RF*, including non-Wi-Fi interference — microwaves, cameras, cordless phones — that Kismet and airodump are completely blind to.

### 2.4 GHz (2400–2500 MHz)

Channels 1–14 overlap, so the usable non-overlapping set is **1, 6, 11**. Channel 14 (2484 MHz) is Japan-only and offset.

| Ch | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| MHz | 2412 | 2417 | 2422 | 2427 | 2432 | 2437 | 2442 | 2447 | 2452 | 2457 | 2462 | 2467 | 2472 | 2484 |

Running APs on 1 and 6 simultaneously is safe — that's a 25 MHz separation. Channels 1 and 5 will interfere.

### 5 GHz (5150–5875 MHz)

Partitioned into UNII bands. Channels 52–144 require DFS radar detection; 36–48 and 149–165 are always available.

| UNII band | Channels | Centres (MHz) | DFS? | Typical use |
|---|---|---|---|---|
| UNII-1 (low) | 36, 40, 44, 48 | 5180, 5200, 5220, 5240 | no | Always used |
| UNII-2A | 52, 56, 60, 64 | 5260, 5280, 5300, 5320 | **yes** | Rare — DFS hassle |
| UNII-2C | 100–144 (step 4) | 5500–5720 | **yes** | Popular for wide channels |
| UNII-3 (high) | 149, 153, 157, 161, 165 | 5745, 5765, 5785, 5805, 5825 | no | Very popular |

DFS channels may show radar spikes or periodic silence — that's regulatory behaviour, not interference. The non-DFS regions are cleaner for continuous monitoring.

### 6 GHz / Wi-Fi 6E (5925–7125 MHz)

Fifty-nine 20 MHz channels. APs beacon on the **Preferred Scanning Channels**, spaced every 80 MHz — scan these first, because clients only actively scan them.

| PSC ch | 5 | 21 | 37 | 53 | 69 | 85 | 101 | 117 | 133 | 149 | 165 | 181 | 197 | 213 | 229 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| MHz | 5975 | 6055 | 6135 | 6215 | 6295 | 6375 | 6455 | 6535 | 6615 | 6695 | 6775 | 6855 | 6935 | 7015 | 7095 |

Channel formulas, in MHz: 2.4 GHz is `2407 + 5 × ch`; 5 GHz is `5000 + 5 × ch`; 6 GHz is `5950 + 5 × ch`. And 6 GHz needs a tri-band antenna — most 5 GHz scanners miss 6E entirely for want of one.

### Command-line wideband scans

```bash
hackrf_sweep -f 2400:2500 -1                 # whole 2.4 GHz band, one shot, ~12 s
hackrf_sweep -f 2400:2500 > 2p4_sweep.csv    # or save the loop to a file
hackrf_sweep -f 5150:5250 -1                 # 5 GHz UNII-1, channels 36–48
hackrf_sweep -f 5725:5850 -1                 # 5 GHz UNII-3, channels 149–165
hackrf_sweep -f 5150:5875 -1                 # all of 5 GHz including DFS, ~42 s
hackrf_sweep -f 5975:7095 -1                 # 6 GHz across the PSC channels
```

The CSV format is `date, time, hz_low, hz_high, bin_width, samples, dB…` with each further column one power bin. **Values are relative dB, not dBm** — compare signals to the noise floor within a band, never between bands.

### QSpectrumAnalyzer — the whole-band waterfall

QSpectrumAnalyzer drives `hackrf_sweep` behind the scenes and shows a live spectrum on top with a time-domain waterfall below. It's the Ekahau-Sidekick-style view, and unlike gqrx it isn't limited to the HackRF's ~20 MHz live window.

1. **Free the radio.** Quit gqrx or stop any running sweep — one tool owns the HackRF at a time.
2. **Launch it** with `qspectrumanalyzer`.
3. **Pick the backend.** File → Settings → Backend = `hackrf_sweep`, executable left as `hackrf_sweep`, device args blank.
4. **Set the range** in MHz, not Hz. Whole 2.4 GHz is `2400`/`2500`; just channel 6 is `2426`/`2448`; UNII-1 is `5150`/`5250`; full 6 GHz is `5925`/`7125`.
5. **Bin width ≈ 100 kHz** for Wi-Fi — smaller is finer but slower.
6. **Antenna and gain.** Attach the right antenna for the band, then raise the RF/IF/BB gain sliders until signals sit clearly above the noise floor *without flat-topping* the display.
7. **Start**, and turn on max-hold or persistence to catch bursty interferers.

#### Reading the display

* **Top plot:** instantaneous power against frequency. Tall peaks are strong signals; the flat line is your noise floor.
* **Bottom plot:** time on the Y axis, frequency on X, colour as power. Watch for patterns — bursty, sweeping, or regular.
* **Max-hold** keeps peaks visible as they come and go, which is how you catch microwaves, baby monitors and cordless phones.
* **Judge height above the floor**, not raw numbers. Twenty dB above the floor is strong; five dB is barely there.

#### What things look like in 2.4 GHz

* A **broadband hump around 2437 MHz** is Wi-Fi on channel 6, the most common default. If the hump is much wider than 20 MHz, it's a bonded 40 MHz channel.
* **Narrow spikes scattered everywhere** are Bluetooth, hopping 1600 times a second and often reading as fuzz.
* **Regular narrow pulses** suggest a cordless phone or baby monitor.
* **A slow sweeping tone** cycling with the mains is microwave oven leakage.

A real tri-band characterisation from my bench: 2.4 GHz hot and congested with energy piled on channel 6; 5 GHz showing clear peaks at channel 100 and channels 149–157; and at 6 GHz a clear signal near 5955 MHz with the tri-band antenna — but dead-flat noise with a 2.4/5 antenna. **At 6 GHz the antenna is the whole story.**

## gqrx — watch one channel, or listen

gqrx is a live software receiver: park on a frequency, watch a zoomed ~20 MHz waterfall, and optionally demodulate audio. Use it to study *one channel* closely; use QSpectrumAnalyzer for a whole band.

1. **Free the radio** — stop any sweep first.
2. **Launch gqrx.** The first run opens the device dialog.
3. **Configure I/O:** device = your HackRF (or type `hackrf=0`), input rate `20000000` for a ~20 MHz display, no decimation, bandwidth 0.
4. **Press the power button** (top left) to start the DSP.
5. **Tune** by clicking the big frequency readout and typing, e.g. `2437.000 MHz` for 2.4 GHz channel 6.
6. **Mode = Off** if you want to *see* the channel rather than demodulate it.
7. **Gains:** RF/amp = 0 to avoid overload, IF/LNA ≈ 24–32, BB/VGA ≈ 20–30, hardware AGC off.
8. **FFT settings:** size around 16k for detail, max-hold on to catch bursts.

> **gqrx only shows ~20 MHz.** That's the HackRF's live window, so gqrx sees one 20 MHz channel at a time. For a 40/80/160 MHz channel or a whole band, use QSpectrumAnalyzer, which sweeps.

### Using it as an actual radio

Set the mode, un-mute the audio panel, tune, and use squelch to cut the hiss. Use the ANT500 or RaTLSnake telescopic — Wi-Fi antennas hear nothing down here.

| Mode | For |
|---|---|
| WFM (stereo) | FM broadcast, 88–108 MHz — the easiest first test |
| AM | Air band, 118–137 MHz (air traffic control) |
| NFM | VHF/UHF ham, PMR, weather radio around 162 MHz |
| LSB / USB | Single sideband (ham) |

Shortwave and AM broadcast are technically in range, down to 100 kHz, but the HackRF is a poor HF receiver and needs a proper HF antenna — a dedicated HF SDR wins there.

## Field recipe: sweep → any LLM

Raw `hackrf_sweep` output is far too much to paste into a chatbot. This script condenses a few seconds of sweeping into about ten lines you can paste anywhere.

```python
#!/usr/bin/env python3
# rfscan.py - condense hackrf_sweep into a small, LLM-pasteable summary
# usage: python3 rfscan.py LOW_MHZ HIGH_MHZ [SECONDS]   e.g. python3 rfscan.py 2400 2500
import subprocess, sys, time, statistics
lo, hi = sys.argv[1], sys.argv[2]
secs = float(sys.argv[3]) if len(sys.argv) > 3 else 4.0
proc = subprocess.Popen(["hackrf_sweep","-f",f"{lo}:{hi}"], stdout=subprocess.PIPE,
                        stderr=subprocess.DEVNULL, text=True)
mh = {}; t0 = time.time()
try:
    for line in proc.stdout:
        p = line.split(',')
        if len(p) < 7: continue
        try: low=float(p[2]); binw=float(p[4]); dbs=[float(x) for x in p[6:]]
        except ValueError: continue
        for i, db in enumerate(dbs):
            f = round((low + i*binw)/1e6)
            if f not in mh or db > mh[f]: mh[f] = db
        if time.time() - t0 > secs: break
finally:
    proc.terminate()
    try: proc.wait(2)
    except Exception: proc.kill()
if not mh:
    print("no data - HackRF connected + antenna on? check hackrf_info"); sys.exit(1)
fs = sorted(mh); vals = [mh[f] for f in fs]; floor = statistics.median(vals)
kept = []
for db, f in sorted(((mh[f], f) for f in fs), reverse=True):
    if all(abs(f-g) > 4 for _, g in kept): kept.append((db, f))
    if len(kept) >= 6: break
prof = {}
for f in fs: prof.setdefault((f//100)*100, []).append(mh[f])
print(f"# HackRF max-hold sweep, {int(secs)}s | relative dB (NOT dBm) | MHz")
print(f"range {fs[0]}-{fs[-1]} MHz | noise floor {floor:.1f} dB | peak {max(vals):.1f} dB")
print("strongest (dB @ MHz):")
for db, f in kept: print(f"  {db:6.1f} @ {f}")
print("avg dB per 100 MHz:")
print("  " + "  ".join(f"{c}:{sum(v)/len(v):.0f}" for c,v in sorted(prof.items())))
```

Then paste this prompt, followed by the script's output:

```
You are a Wi-Fi / RF analyst. Below is a condensed HackRF spectrum sweep
(max-hold over a few seconds). Values are RELATIVE power in dB, not dBm - judge
each signal by how far it sits ABOVE the noise floor. Frequencies are in MHz.
Please: (1) map the strongest signals to Wi-Fi channels/bands and flag any peak
not aligned to a channel centre as possible non-Wi-Fi interference; (2) rate
congestion; (3) recommend the least-congested channel(s); (4) note caveats
(single spot + moment, coarse bins, relative dB).

Data:
<paste rfscan.py output>
```

## AI and machine learning in SDR work

SDR produces streams of raw data — IQ samples, spectrograms, decoded bits — which are natural inputs for machine learning. The interesting applications:

| Use case | Input | Approach | Tools |
|---|---|---|---|
| **Modulation classification** | IQ samples, 1–10 ms | CNN trained on 2D spectrograms to recognise BPSK, QPSK, FSK, OOK | PyTorch or TensorFlow with a custom dataset |
| **Anomaly detection** | Spectrum snapshots | Autoencoder or Isolation Forest: learn what "normal" looks like, flag deviations | scikit-learn |
| **Signal detection in noise** | Noisy IQ stream | CNN or matched filter with threshold, for weak bursts | GNU Radio or standalone Python |
| **Interference classification** | Waterfall spectrogram | CNN trained on known patterns — microwave bursts, cordless phones, frequency hoppers | Spectrogram exports plus PyTorch |
| **Protocol reversal** | IQ of unknown modulation | Clustering on phase and amplitude, or supervised training on known messages | URH for visual work, Python for batch |
| **LLM analysis** | A spectrum summary like `rfscan.py` output | Ask a model to describe the spectrum or name busy bands — useful for field notes | Any chatbot or API, or a local model via Ollama |

### A worked example: anomaly detection on a parked dongle

1. **Establish a baseline.** Park the RTL-SDR on a band of interest and run `rtl_power` for an hour in a quiet environment, saving frequency/power readings to CSV.
2. **Train.** Load that CSV in scikit-learn and fit an Isolation Forest.
3. **Deploy.** Run live in short snapshots, score each against the model, and alert when the anomaly score spikes — a new transmitter, or interference that wasn't there before.
4. **Close the loop.** Label the alerts (false alarm or real?) and retrain periodically.

### Dataset tips, learned the hard way

* **Label first.** Know exactly what you captured — modulation, frequency, source. Unlabelled captures are useless for supervised learning.
* **Vary conditions.** Capture at different SNR, gain settings and times of day, or the model overfits to one situation.
* **Augment synthetically.** Adding noise and time/frequency shifts to existing captures can grow a dataset tenfold.
* **Baseline with something dumb first.** Try a threshold rule before a neural network — "if the frequency is 2437 and power is above X, it's Wi-Fi" solves a surprising number of problems.
* **Version control the dataset, labels and model checkpoints.** Reproducibility matters more than you expect.

## Project ideas

### Passive analysis

* **Multi-band occupancy survey.** Sweep 2.4, 5 and 6 GHz to map AP density against location and time of day, then plot heatmaps. `hackrf_sweep` plus QSpectrumAnalyzer plus matplotlib.
* **Non-Wi-Fi interference catalogue.** Identify and classify the interferers in each band: microwave ovens as a broadband hump near 2450 MHz, cordless phones as regular FSK spikes, Bluetooth as hopping fuzz, ZigBee on its narrow channels. inspectrum to visualise, URH to measure timing.
* **AP fingerprinting by preamble.** Capture 802.11 preambles from many APs, extract features like modulation accuracy and clock offset, and see whether you can identify an AP model from its RF signature alone. The defensive use is rogue-AP detection: does this MAC's RF profile match what it claims to be?
* **Channel-switching behaviour.** Log how often and when APs change channel — DFS evasion, load balancing, band steering — by timestamping hops in the waterfall.
* **Spurious emissions hunt.** Scan wide around the Wi-Fi bands for out-of-band leakage from APs and clients, and measure the rejection.

### Capture and decode

* **Frame type distribution.** Capture raw 802.11 frames and classify by type, then look at beacon intervals, SSID lengths, supported rates and vendor extensions. Capture with `hackrf_transfer -r`, decode in Wireshark or parse with Scapy.
* **Metadata from encrypted traffic.** Even without decryption, sequence numbers, retry patterns, fragmentation and aggregation tell you about link quality and roaming behaviour. The radio layer tells a story on its own.
* **Roaming trace.** Follow one device as it moves between APs, logging AP MAC, RSSI and data rate, to work out what actually triggers a roam.

### Transmit

> **Transmitting is legally gated, and this is the part to be careful with.** Receiving is always safe; transmitting is regulated everywhere. Stay inside bands you're licensed or permitted to use, keep it to your own equipment, and do it in a controlled environment. Deliberately interfering with other people's wireless — jamming, deauth floods against networks you don't own — is illegal in most countries, and regulators have issued substantial fines for exactly that. "It was a test" is not a defence.

With that said, the legitimate TX experiments worth building toward:

* **Synthetic beacon generation on a test channel**, to measure how quickly clients distinguish a spoofed AP from a real one — the research behind rogue-AP detection. GNU Radio Companion to generate, HackRF to transmit, on your own gear in a controlled space.
* **Receiver robustness benchmarking.** Capture a real preamble, inject known phase and amplitude errors, retransmit, and measure how much corruption it takes before frames are lost.
* **Attack signature capture for building detectors.** Trigger a known attack pattern from a device you control, capture it, and characterise its frame structure and inter-frame timing — so you can build something that recognises it in the wild. This is the defensive half of the work, and it's the half worth publishing.

### A workflow that actually gets somewhere

1. **Define the question.** "Which band has the least interference here?" beats "let's look at some RF."
2. **Gather baseline data** over hours, not seconds. Save the logs, CSVs and screenshots.
3. **Analyse manually first.** Plot it. Look at it. Build intuition before training anything.
4. **Hypothesise**, then test one hypothesis at a time.
5. **Prototype with simple tools.** A threshold rule or a linear fit first; complexity only if it's needed.
6. **Then automate**, run it regularly, and watch the trends.

## Further reading

### Official documentation

* [Great Scott Gadgets — HackRF](https://greatscottgadgets.com/hackrf/) — official docs, firmware, hardware specs
* [HackRF documentation](https://hackrf.readthedocs.io/) — command reference and tutorials
* [HackRF on GitHub](https://github.com/greatscottgadgets/hackrf) — firmware, hardware CAD, issues
* [rtl-sdr (osmocom)](https://github.com/osmocom/rtl-sdr) — the RTL2832U driver and librtlsdr source
* [gqrx](https://gqrx.dk/) — docs and downloads
* [GNU Radio](https://www.gnu.org/software/gnuradio/) — site, docs, flowgraph examples
* [QSpectrumAnalyzer](https://github.com/xmikos/qspectrumanalyzer) — the sweep GUI
* [Nooelec](https://nooelec.com/) — NESDR documentation and support

### Communities

* [r/RTLSDR](https://www.reddit.com/r/RTLSDR/) — large, active, and genuinely newcomer-friendly
* [r/HackRF](https://www.reddit.com/r/HackRF/) — HackRF-specific projects and firmware talk
* [RTL-SDR.com](https://www.rtl-sdr.com/) — weekly news, tutorials and reviews; probably the best single SDR blog
* [Hackaday.io](https://hackaday.io/) — search HackRF or RTL-SDR for thousands of logged projects with code
* GitHub Discussions on the HackRF and rtl-sdr repos, which are where the maintainers actually answer

### Books

* **The Hobbyist's Guide to the RTL-SDR** (Carl Laufer) — the standard beginner's book, theory and practice
* **802.11 Wireless Networks: The Definitive Guide** (Matthew Gast) — the Wi-Fi protocol deep dive; pairs perfectly with packet-capture projects
* The official **GNU Radio tutorials** — free, and the fastest way into GRC

### Conferences

DEF CON has large RF and wireless villages with live demos; ShmooCon runs RF workshops; Hamvention is the big ham gathering with SDR vendors everywhere. Local maker spaces and ham clubs are the underrated option — most cities have one.

### AI and RF

* [DeepSig](https://deepsig.ai/) — publishes open RF datasets and modulation-classification research
* arXiv, for papers on modulation recognition, spectrum sensing and RF anomaly detection
* The official PyTorch image-classification tutorials, which transfer almost directly to spectrograms
* scikit-learn's Isolation Forest docs, for the simplest useful anomaly detection

### Getting started, in order

1. Skim the HackRF docs and watch one tutorial to build a mental model of what's possible.
2. Lurk in r/RTLSDR for a week and pick one project that looks fun.
3. **Replicate that project.** Follow someone else's write-up. Don't invent yet — learn the tools.
4. Read a proper intro book to get modulation, bandwidth and gain straight.
5. Then design your own project around a real question, and document it.

## My kit

For reference, what's on the bench as of September 2026:

**Radios**

* **HackRF Pro** — 100 kHz–6 GHz, 20 MS/s, half-duplex, SMA female port. The main SDR.
* **Nooelec NESDR Mini 2+** — R820T2/RTL2832U, RX-only, ~25 MHz–1.7 GHz. The cheap second receiver.

**Antennas**

* ANT500 (75 MHz–1 GHz, SMA)
* RaTLSnake M6 v2 set — telescopic, DVB-T stubby and helical masts plus a magnetic base, all SMA male
* Tri-band Wi-Fi 6E paddle (RP-SMA, with adapter)
* 2.4/5 AP antenna (RP-SMA)
* RP-SMA-female → SMA-male adapter, which lives permanently on the Wi-Fi antenna

**Software, the short version**

```bash
# minimal: spectrum + decode
brew install hackrf rtl-sdr dump1090 rtl_433
pipx install --backend pip qspectrumanalyzer
```

```bash
# fuller: add receivers and offline analysis
brew install hackrf rtl-sdr gqrx dump1090 readsb rtl_433 inspectrum gnuradio
pipx install urh
```

**A quick both-radios sanity check**

```bash
hackrf_info 2>/dev/null && echo "HackRF found" || echo "HackRF not found"
rtl_test -t 2>/dev/null | grep -q RTL && echo "RTL-SDR found" || echo "RTL-SDR not found"
hackrf_sweep -f 2400:2500 -1 > /tmp/test.csv && wc -l /tmp/test.csv
```

### Where this goes next

* **Transmit experiments** — `hackrf_transfer -t`, replaying captures, generating tones. Legally gated, own gear only.
* **Protocol decoding** — ADS-B with dump1090, ISM sensors with rtl_433, reverse engineering with URH, custom GNU Radio flowgraphs.
* **PortaPack** — a standalone HackRF portable with screen, buttons and battery, so it's radio without a laptop.
* **Wardriving with GPS** — log signals against location and map the result.

---

*Notes written against a HackRF Pro and a Nooelec NESDR Mini 2+ on an M1 MacBook, September 2026. Relative dB is not dBm, receiving is not transmitting, and the antenna is usually the problem.*
