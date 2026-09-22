---
layout: post
title: "WLAN Pi Field Guide"
categories: [tech]
description: >
  Notes on the WLAN Pi — how to get into it, what the profiler really does,
  why 6 GHz beaconing fails on an Intel card, and the wireless-from-the-shell
  commands worth keeping.
---

I picked up a wlan pi at a wlanpros conference. Pretty cool little device. Very handy when playing around with wifi. Decided to write up some notes on it with Claude. Posting them here for reference 🙂
{:.lead}

These were written against my own unit — a WLAN Pi running `wlanpi-core` 2.0.0 on Debian bullseye — rather than from the generic docs, so the addresses and versions below are mine. The procedures are general.

* this list will be replaced by the table of contents
{:toc}

## Getting in

Five doors into the same device. Which one you use depends on what you're doing and what's plugged in.

| Door | Address | Use it for |
|---|---|---|
| Web front end | `https://169.254.42.1/` | Profiler, Kismet, Grafana, speed test, network info. The main console. |
| Cockpit | `COCKPIT` in the nav | Browser-based shell, service control, logs, resource graphs. No SSH client needed. |
| SSH | `wlanpi@169.254.42.1` | Everything else. Default credentials are `wlanpi` / `wlanpi`, forced to change on first boot. |
| Front panel | The OLED + buttons | Mode switching, IP display, shutdown — no laptop required. |
| Core API | `:31415/docs` | Swagger UI for the REST API that drives everything above. |

### Cockpit is the one people miss

It sits in the web nav and gives you a full terminal in the browser, plus service management and journal access. It's the fastest route to a root shell, and it sidesteps SSH host-key problems entirely — different transport, different trust store.

### The two networks

`169.254.42.1` is the USB-OTG link — plug the Pi into your laptop with a USB cable and it appears as a network adapter. It's link-local, always there, and needs no infrastructure. `192.168.2.179` is the wired ethernet address from DHCP on my LAN.

This distinction matters more than it looks: several features take over the Wi-Fi radio, and OTG keeps you connected while that happens.

#### The API behind it all

`wlanpi-core` is a FastAPI service on port 31415. Every GUI feature is a call to it, and its Swagger UI is browsable:

```
http://169.254.42.1:31415/docs
http://169.254.42.1:31415/api/v1/openapi.json
```

This unit runs **core 2.0.0 with 24 endpoints**. Endpoints need a bearer token; the docs page itself doesn't.

## Profiler

The signature WLAN Pi feature, and the one nothing else does as conveniently. The Pi stands up a fake AP; when a client tries to associate, the profiler captures the association request and decodes exactly what that client claims it can do — then refuses the association, because it only ever needed the one frame.

### What you get

A capability report per client: supported spatial streams, channel widths, 802.11k/v/r support, band support, and the newer Wi-Fi 6/6E/7 feature bits. This is how you answer "will this handset actually use my 80 MHz channels?" without taking the vendor's word for it.

### Running it

1. Web UI → **PROFILER** → start it
2. On the client device, find the profiler's SSID and attempt to join
3. The join fails — that's expected and by design
4. Read the report back in the PROFILER menu

Each profiled client produces a text report and a matching pcap of the association frame, so you can go back and look at the raw information elements yourself. Reports are written to `/var/www/html/profiler/`, which is what the web UI reads — so command-line runs still appear in the GUI.

### Bands and generations are different axes

Worth separating, because the naming actively encourages confusion:

| Axis | Values | Means |
|---|---|---|
| Band | 2.4 · 5 · 6 GHz | **Where** — which slice of spectrum |
| Generation | Wi-Fi 4 · 5 · 6 · 7 | **How** — which protocol, i.e. 802.11n / ac / ax / be |

The two are largely independent. Wi-Fi 6 runs on 2.4 and 5 GHz; Wi-Fi 7 runs on all three. Only Wi-Fi 5 (802.11ac) is band-locked, to 5 GHz.

> **Wi-Fi 6E is not a generation.** The "E" is **Extended** — it means Wi-Fi 6 (802.11ax) operating in the newly opened 6 GHz band. Same protocol, new spectrum. It has nothing to do with outdoor use.
>
> The outdoor association is a real rule attached to a different thing: 6 GHz has power classes, and *Low Power Indoor* is indoor-only, while *Standard Power* outdoor operation requires AFC coordination. That's a 6 GHz regulatory matter, not what the E stands for.

Wi-Fi 7 (802.11be) adds 320 MHz channels, 4K-QAM, and **Multi-Link Operation** — a client using more than one band at the same time rather than picking one. MLO is what "STR" refers to in the WLPC Prague session title.

### What "advertises Wi-Fi 7" actually means

The profiler is a fake AP on **one channel, in one band**. It does not cascade through generations, and there's no fallback ladder.

What the default does is put the newest capability elements into its beacon. A Wi-Fi 7 client sees them and answers with its own EHT capabilities; an older client doesn't recognise them, ignores them, and answers with whatever it does support. Advertising the newest standard costs older clients nothing — **each client simply replies with as much as it understands**.

So the two controls are independent: **the channel sets the band, the flags set the generation.**

| Flag | Effect |
|---|---|
| (default) | Advertises Wi-Fi 7 over WPA2. Elicits the most from every client *except* EHT |
| `--wpa3_personal` | WPA3-only: SAE with PMF **required**. Needed to elicit 802.11be |
| `--wpa3_personal_transition` | WPA2/WPA3 mixed. PMF optional, so EHT may still not appear |
| `--no11be` / `--no11ax` / `--no11r` | Make the fake AP pretend the standard doesn't exist |
| `--noAP` | Listen only — no beacons, just capture association requests |
| `--read PCAP` | Re-analyse a saved capture offline |

Downgrade deliberately when you want to see how a client behaves against an older AP — or when a client chokes on 11be beacons, which some genuinely do.

> **Check your version's flags before copying anything.** The CLI has changed between releases. Upstream documentation describes a `--security-mode` option with `wpa2` / `ft-wpa2` / `wpa3-mixed` / `ft-wpa3-mixed` values that **does not exist** in the build on this unit, which uses the two `--wpa3_personal*` switches above instead. There is likewise no `security_mode` key in this version's `config.ini` — adding one is silently ignored.
>
> Run `sudo profiler -h` first and trust that over any document, this one included.

### Three bands means three runs

A client tailors its association request to the band it is joining, so the elements genuinely differ:

* **2.4 GHz** — HT capabilities. VHT doesn't exist here, so you learn nothing about 802.11ac
* **5 GHz** — HT + VHT + HE + EHT. The richest single report
* **6 GHz** — carries the *HE 6 GHz Band Capabilities* element, which appears nowhere else

#### Channel width needs no configuration

Width support is advertised as a **capability, not a negotiation**. A 160 MHz-capable client says so even when your profiler sits on a 20 MHz channel — it's carried in the Supported Channel Width Set fields of the HT, VHT and HE capability elements. You never need to run the profiler wide to discover that a client is wide.

*A minority of clients trim what they advertise to match the AP. If a width looks implausibly low for a device you know is capable, treat it as worth a second look rather than as fact.*

### Setting the channel

There is no channel option in the web UI, and on core 2.0.0 there cannot be — the API exposes no profiler endpoints, so the GUI can only start and stop the service. Configuration lives outside it.

#### Persistent

```bash
sudo nano /etc/wlanpi-profiler/config.ini
```

```
[GENERAL]
channel: 149
interface: wlan0
ssid:
ft_disabled: False
he_disabled: False
be_disabled: False
```

The `*_disabled` keys read as you'd hope but catch people out: **`True` means the feature is switched off**. Setting `be_disabled: True` stops the AP advertising 802.11be, so clients stop reporting it — which looks exactly like a client that doesn't support Wi-Fi 7. Leave all three `False` unless you are deliberately testing against an older AP.

Security is **not settable here** in this version — it is a CLI flag only, which means a GUI-started profiler always beacons WPA2.

#### Per run — what you want for a band sweep

Stop the service first, or the two fight over `wlan0`:

```bash
sudo systemctl stop wlanpi-profiler

sudo profiler -c 6       # 2.4 GHz
sudo profiler -c 36      # 5 GHz
sudo profiler -f 5975    # 6 GHz, PSC channel 5
```

Settings resolve in the order **command line → environment → config file → defaults**.

### Safe channels by region

Two things constrain the choice: **DFS** in 5 GHz — where the AP must perform a radar check before beaconing, which the profiler shouldn't have to wait through — and the fact that **Europe has only opened the lower half of 6 GHz** (5945–6425 MHz), while the US and Canada have the full band.

| Band | United States | Canada | EU / Czechia | Safe everywhere |
|---|---|---|---|---|
| 2.4 GHz | 1–11 | 1–11 | 1–13 | **6** |
| 5 GHz non-DFS | 36–48, 149–165 | 36–48, 149–165 | 36–48 | **36** |
| 6 GHz PSC | 5–229 | 5–229 | 5–85 | **5** · 5975 MHz |

> **Legal is not the same as usable.** That table is regulatory. The radio has opinions of its own, and on this unit they disagree:
>
> ```
> * 5180.0 MHz [36]  (22.0 dBm) (no IR)   ← starts, beacons nothing
> * 5745.0 MHz [149] (22.0 dBm)           ← works
> ```
>
> Channel 36 is legal everywhere and **this card will not beacon on it**. Channel 149 works and is **unavailable in Europe**. There is currently no 5 GHz channel that is both legal in the EU and usable on this hardware.
>
> **Channel 6 on 2.4 GHz carries no restrictive flags at all** — it is the only choice that travels. Plan on it for Prague, and check 5 GHz on arrival with the command below rather than assuming.

```bash
# look for "(no IR)"
iw phy phy0 info | grep -E "5180.0 MHz|5745.0 MHz"
```

#### Set the regulatory domain first

The radio refuses to transmit on channels it believes are illegal for its configured country, which presents as "the profiler started but no SSID appears."

```bash
sudo iw reg set CZ    # US · CA · CZ
iw reg get
```

> **Two 6 GHz traps.**
>
> **Use `-f`, not `-c`.** 6 GHz channel numbering restarts at 1, so "channel 1" is ambiguous between bands. Centre frequency is `5950 + 5 × n` MHz.
>
> **Stay on a PSC channel.** Clients only actively scan the Preferred Scanning Channels — 5, 21, 37, 53, 69, 85, every 80 MHz. On any other channel a 6 GHz client may never find your SSID, which looks exactly like "6 GHz is broken."

### Troubleshooting: "No IR found"

The profiler's most common failure looks like this:

```
[WARNING] No IR found in iw channel information for 5 (5975)
          which _may_ cause packet injection to fail!
```

**IR means Initiate Radiation.** It's a per-channel regulatory permission: may this radio *transmit first*, rather than only listen and reply? A fake AP must beacon unprompted, so without IR the profiler starts, claims to be beaconing, and radiates nothing. No association request ever arrives, and no profile appears.

#### Read the regulatory state properly

```bash
iw reg get
```

This prints **two** domains, and the difference between them is the whole diagnosis:

| Block | What it is |
|---|---|
| global | The system-wide setting — what `iw reg set` changes |
| phy#0 | What the radio itself will actually honour |

> **If phy#0 says self-managed,** the driver carries its own regulatory database and **`iw reg set` will not move it**. You can set the global domain to anything you like and the radio will carry on obeying its internal table — frequently `country 00`, the maximally conservative world domain. Intel cards behave this way as standard.

#### Does the phy declare 6 GHz?

```bash
iw reg get | grep "59[2-9][0-9] - "
```

A line covering `5925 – 7125` means 6 GHz is available to you now. No output has **two quite different causes**, so identify the card before concluding anything:

```bash
lspci -nn | grep -i network
lsusb | grep -i "wireless\|wlan\|802.11"
```

| Card | Reading |
|---|---|
| 2.4/5 GHz part<br>(AX200-class) | Genuinely no 6 GHz radio. Nothing will change it |
| 6E / Wi-Fi 7 part<br>(AX210, BE200-class) | The radio is capable; the **driver hasn't unlocked the band**. A different problem entirely |

#### The Intel 6 GHz trap

Intel parts use **Location Aware Regulatory**: the card refuses to enable 6 GHz until it has learned its country by *hearing beacons while associated as a client*. That has a consequence specific to this tool — in monitor mode the card never associates, so LAR never resolves, the domain stays `country 00`, and the 6 GHz channels are never added to the table. The profiler then reports "No IR found" because the frequency genuinely isn't in the list to have a flag.

Intel also disables standalone AP operation above 2.4 GHz in its driver, as a deliberate regulatory-risk decision. The profiler dodges that on 5 GHz by injecting raw frames from a monitor interface rather than running hostapd — but injection can only tune to channels the phy already knows about, which is why 5 GHz profiling works and 6 GHz does not.

The sometimes-effective workaround is to make LAR resolve first: associate the card as a normal client to a 6 GHz-capable AP, let it learn the country, *then* drop into monitor mode without rebooting. It needs a 6 GHz AP in earshot and it doesn't always survive the mode change.

> **Why chipset choice matters more than specification.** Intel radios are excellent clients and mediocre tools. Monitor mode, injection and AP operation are exactly where their driver policy is most restrictive, and a card can be fully Wi-Fi 7 capable on paper while refusing to do the one thing a profiler needs.
>
> This is the actual reason the WLAN Pi community favours particular Qualcomm and MediaTek parts, and what a card swap buys you — not more capability on the box, but a driver that will use it.

#### The other flags you'll see

| Flag | Meaning for the profiler |
|---|---|
| NO-IR | Cannot transmit first. Fatal — no beacons |
| PASSIVE-SCAN | Listen only on this channel until an AP is heard |
| IR-CONCURRENT | May transmit only while already associated to an AP on that channel. Restrictive, but raw injection from a monitor interface often still succeeds |
| DFS | Radar check required before beaconing. Avoid for profiling |
| NO-OUTDOOR | Indoor use only. Doesn't block profiling |

2.4 GHz entries typically carry none of these, which is why 2.4 GHz profiling is the most reliable fallback when 5 GHz misbehaves.

### Case study: chasing 6 GHz on a BE200

Worked through on this unit in September 2026. The conclusion is negative, but the intermediate steps are the useful part — and none of it is documented upstream.

#### Starting state

Card is an **Intel BE200** (`8086:272b`) — Wi-Fi 7, tri-band, 320 MHz. Kernel `6.12.13-v8-wlanpi+`. Every 6 GHz attempt produced "No IR found" and silence.

| Stage | phy#0 country | 6 GHz channels | Can beacon? |
|---|---|---|---|
| As found | `00` (world) | 59 × `(disabled)` | No — band absent |
| After wpa_supplicant scan | **`CA`** | 58 × `(no IR)`, 0 disabled | No — cannot initiate |
| Profiler on 5975 MHz | `CA` | tunable | **No — 0 TX packets** |

#### The bootstrap that does work

Starting the supplicant is enough to pull the card out of the world domain. It does *not* need to associate — the scan alone lets it hear country IEs and resolve:

```bash
sudo wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf
iw reg get | sed -n '/phy#0/,$p' | head -3
iw phy phy0 info | grep "5975.0 MHz"
```

That flips `country 00` to the real domain and moves 6 GHz from `(disabled)` to present. If you ever fit a card that *can* beacon in 6 GHz, this is the prerequisite step — without it the band simply isn't there.

#### Where it dies

With the band present, the phy's 6 GHz regulatory entry reads:

```
(5945 - 7065 @ 1180), (6, 22), NO-OUTDOOR, AUTO-BW, IR-CONCURRENT, PASSIVE-SCAN
```

**IR-CONCURRENT** permits transmission only while concurrently associated on that channel. Satisfying it in 6 GHz would require associating to a 6 GHz AP, which requires 6 GHz transmission — and a single radio chain cannot be associated on one channel while beaconing on another. The condition is unsatisfiable in this configuration.

#### The measurement that settles it

Don't trust the profiler's own output — it reports "starting beacon transmissions" whether or not frames leave the radio. Count them:

```bash
cat /sys/class/net/wlan0mon/statistics/tx_packets; sleep 6; \
  cat /sys/class/net/wlan0mon/statistics/tx_packets
```

A beaconing AP at 100 TU emits roughly 60 frames in six seconds. This card produced **0**, with `tx_errors` and `tx_dropped` also zero — the frames were never attempted. This counter check is the fastest way to distinguish "not beaconing" from "beaconing but the client can't see it," and it works on any band.

A second tell: `iw dev` showed `wlan0mon` parked on channel 25 (6075 MHz) after a request for 5975 MHz. The tuning request wasn't honoured either.

> **Firmware is deliberately pinned — leave it.** `/lib/firmware/` holds `iwlwifi-gl-c0-fm-c0-90.ucode` alongside `-92.ucode.disabled` and `-92.ucode.original`, root-owned and dated January 2025. The driver requests 93, then 92, finds neither, and falls back to **90**.
>
> That rename is a WLAN Pi image decision, almost certainly trading newer firmware for working monitor mode and injection. Restoring 92 or promoting 94 to chase 6 GHz risks losing the injection everything else depends on. Don't.

### Managing saved profiles

There is no delete button in the web UI and no `--clean` flag. Removal is manual, and the files are root-owned, so `sudo` is required:

```bash
ls -la /var/www/html/profiler/clients/   # one directory per client MAC
ls -la /var/www/html/profiler/reports/   # daily CSV summaries

sudo rm -r /var/www/html/profiler/clients/*
sudo rm /var/www/html/profiler/reports/*
```

This deletes the text reports, the JSON, and the association-frame pcaps together — the pcaps are the part worth copying off first if a capture was interesting. Newer profiler builds may also write to `~/.local/share/wlanpi-profiler/clients/`, so check there if entries seem to survive.

## Kismet

Passive wireless discovery and IDS at `169.254.42.1:2501`. It channel-hops, logs every device and SSID it hears, and raises alerts on suspicious patterns. Purely a receiver — it never transmits.

### Reading the device list

The columns that matter are **Type** (AP, Client, Bridged, Device), **Sgn** in dBm, **Chan**, and **BSSID**. A device showing `n/a` signal was inferred from someone else's frames rather than heard directly.

#### Signal reference

| RSSI | Means |
|---|---|
| −30 dBm | Same room as the AP |
| −50 dBm | Excellent |
| −60 dBm | Good |
| −67 dBm | Design floor: voice & video |
| −75 dBm | Basic data only |
| −85 dBm | Approaching noise floor |

*−67 dBm is the conventional design threshold for real-time traffic — the number most survey specs are written against.*

### Yes to pcap

Kismet logs continuously from the moment it starts; there's no capture button to press. It writes a `.kismet` file (SQLite) containing devices, alerts and packets. Convert afterwards:

```bash
kismetdb_to_pcap --in /path/to/log.kismet --out capture.pcapng
```

Find the configured log directory in `/etc/kismet/kismet_logging.conf` if the default isn't where you expect.

### No to deauth

Kismet has no injection capability at all — it's architecturally a listener. What it does instead is the inverse, and arguably more useful: the **Alerts** tab flags deauthentication floods, spoofed APs, and known attack signatures when it hears them. "Is something deauthing my clients?" is the question you actually get asked in the field, and this answers it.

> **If you need to transmit,** deauth transmission means `aireplay-ng` or `mdk4`, neither of which ships on the WLAN Pi image. Own equipment or written authorisation only — the FCC has issued substantial fines for deauth-based Wi-Fi blocking, and "it was a test" is not a defence.

## Packet capture

The highest-value thing this device does. The Pi's radio goes into monitor mode and streams frames live into Wireshark on your Mac — no files to shuffle, full 802.11 dissection, your own display filters.

> **Wiring constraint.** Capture consumes the Pi's main Wi-Fi adapter, so you must reach the Pi over **wired or a second adapter**. The OTG link at `169.254.42.1` satisfies this — which is why it's the right way to be connected while capturing.

### Option A — Airtool 2 (documented route)

Airtool SSHes in, sets monitor mode, tunes the channel, runs `tcpdump`, and pipes frames back to Wireshark. Paid, with a 3-day trial.

1. Install **Wireshark** and **Airtool 2**
2. Airtool → **Preferences → Sensors** → add `169.254.42.1`
3. Start a capture; enter the interface, e.g. `wlan0`
4. Choose channel and channel width
5. Enter SSH credentials — stored in Keychain thereafter

### Option B — sshdump (free)

Wireshark ships an `sshdump` extcap, so you can skip Airtool if you set monitor mode yourself:

```bash
sudo ip link set wlan0 down
sudo iw dev wlan0 set monitor none
sudo ip link set wlan0 up
sudo iw dev wlan0 set channel 36 HT40+
```

Then in Wireshark pick **SSH remote capture: sshdump**, set host `169.254.42.1`, user `wlanpi`, and use this remote command:

```
tcpdump -i wlan0 -U -w -
```

Less polished — channel changes are manual — but free, and the frames are identical. The exact `iw` incantation varies by driver; if it errors, the driver may need the interface renamed or a `monitor`-type virtual interface added instead.

## Grafana

A local Grafana instance at `/app/grafana`, fed by InfluxDB on the device. It is **not** connected to any internet account — nothing to reset online, and any Google identity is irrelevant to it.

> **Login.** The admin account is named **`wlanpi`**, not `admin`. This trips up nearly everyone, because `grafana-cli` talks about "admin" throughout while resetting whatever account holds user ID 1.

#### Resetting the password

```bash
sudo grafana-cli admin reset-admin-password <newpassword>
```

That resets user ID 1. To confirm what the accounts actually are:

```bash
sudo sqlite3 /var/lib/grafana/grafana.db \
    "select id, login, email, is_admin from user;"
```

```
1|wlanpi|admin@localhost|1
2|sa-1-wlanpi|sa-1-wlanpi|0
```

User 2 is a service account the Pi uses to provision its own dashboards — leave it alone.

### Data streams

Under **GRAFANA → DATA STREAMS**, each stream is a collector you switch on individually. Three run on the base hardware:

| Stream | What it records |
|---|---|
| Internet Monitoring | Continuous throughput and latency to the internet over time — the "was it slow at 3pm?" dashboard |
| WLAN Pi Health | CPU, memory, temperature, disk. Watch this during long captures |
| Scanner WLAN0 | Repeated Wi-Fi scans logged over time — signal levels per BSSID, historical rather than instantaneous |

The greyed-out entries — **Oscium WiPry Clarity** (2.4/5/6 GHz) and **MetaGeek Wi-Spy DBx** (2.4/5 GHz) — are spectrum analysers. The software support is there; they need the physical USB dongle plugged in. That's the only way to get true RF spectrum data, as opposed to Wi-Fi frames: non-802.11 interference like microwaves, video senders and radar only shows up here.

## Speed test

Two different measurements that get confused constantly.

| Test | Measures |
|---|---|
| LibreSpeed | **Your laptop ↔ the Pi.** Local only. Over OTG it measures the USB path; associated to the Pi's hotspot it measures real wireless throughput |
| Internet test | **The Pi ↔ the internet**, over whatever uplink the Pi has |

The one at `/speedtest/librespeed` in the web UI is the local one. A reading of 0.00 simply means it hasn't been run.

For throughput testing proper, use iperf rather than a browser — see [Field quick wins](#field-quick-wins).

## Modes

Modes reconfigure networking wholesale and reboot the device. You are in one mode at a time, and leaving Classic means the Classic tools — Kismet, Grafana, profiler — are unavailable until you switch back.

| Mode | What it becomes |
|---|---|
| Classic | Default. All the analysis tools in this guide |
| Wi-Fi Console | A wireless serial console server |
| Server | DHCP + TFTP + terminal server at once — a lab bench in a bag |
| Hotspot | A test AP you fully control |
| Bridge | Bridges wired and wireless |

### Wi-Fi Console — the one worth learning

Plug USB-to-serial cables into the Pi, leave it in the rack, and console into your switches from a desk down the hall. Up to eight adapters via a USB hub.

```bash
sudo /usr/sbin/wconsole_switcher on   # reboots
sudo /usr/sbin/wconsole_switcher off  # back to Classic
```

Or from the front panel: **Menu → Modes → Wi-Fi Console**.

| Setting | Default |
|---|---|
| SSID | `wifi_console` |
| Passphrase | `wifipros` |
| Channel | 1 |
| IP | `172.16.43.1` |

#### The port scheme

**The TCP port is the baud rate.** Telnet to the port matching the speed you want:

```bash
telnet 172.16.43.1 9600
```

| Port | Meaning |
|---|---|
| 2400 / 4800 / 9600 / 19200 | That baud rate, first adapter |
| 11520 | 115200 baud — truncated because 115200 exceeds the maximum TCP port of 65535 |
| 9601, 9602 … | Last digit is the adapter number, so 9602 is adapter 2 at 9600 baud |
| 2001–2008 | Cisco USB console cables |

> **Change the passphrase before any customer site.** `wifipros` is published in the public documentation. Anyone in range who has ever touched a WLAN Pi can join that SSID and land on the console of whatever you have cabled up. Fine on your bench; a serious exposure in a live environment.
>
> Edit `/etc/wlanpi-wconsole/conf/hostapd.conf` while in Classic mode, or run `quickstart-wconsole` for guided setup, then switch.

### Server mode

DHCP and TFTP on wired ethernet, an AP on wireless, plus the console server — for staging gear, pushing firmware, or building a lab with no infrastructure. Wired subnet `172.16.42.0/24`, wireless `172.16.43.0/24`.

It is deliberately **non-persistent** — it reverts on reboot, so a forgotten DHCP server can't follow you onto a production network. That's a safety feature, not a bug.

## Field quick wins

Small features that save disproportionate amounts of time.

### Which switch port am I in?

The Pi listens for **LLDP and CDP** and reports the neighbour it hears — switch name, port ID, VLAN. Plug in, look at the network info, and you know where you are without touching the switch. Available in the web UI's network section, the front panel, and the API's network info endpoint.

### Find that port physically

The **Ethernet port blinker** flashes the Pi's link so the corresponding switch port LED flickers visibly — for when LLDP tells you it's `Gi1/0/23` but the labelling in the rack disagrees.

### Throughput testing

Run the Pi as an iperf server and push traffic from a client — the honest way to measure wireless throughput, unaffected by internet conditions.

```bash
# on the Pi
iperf3 -s

# from a client
iperf3 -c <pi-address> -t 30
iperf3 -c <pi-address> -t 30 -R   # reverse direction
```

Both `iperf` and `iperf3` are managed services on the device.

### Remote scanning radio

The Wi-Fi scanner can feed **Wi-Fi Explorer Pro** on macOS, turning the Pi into a remote sensor. Useful when you want the scanning radio somewhere your laptop isn't — the far end of a warehouse, or a spot you can't stand in during business hours.

### Bluetooth and chat-bot

The Pi pairs over Bluetooth for tethered access without cables, and a Telegram chat-bot exists for poking it remotely — a pre-MCP answer to the same "control it from somewhere else" question.

## Wireless from the shell

> **NetworkManager will not help you here.** `wlan0` is deliberately left **unmanaged** by NetworkManager so the profiler, Kismet and the scanner can claim the radio without a fight. `nmcli dev wifi list` returns nothing and `nmcli dev wifi connect` fails with *"Scanning not allowed while unavailable"*.
>
> The native tools on this device are `iw`, `wpa_supplicant` and `dhclient`. Everything below uses those. You *can* hand the interface to NM with `sudo nmcli dev set wlan0 managed yes`, but hand it back afterwards or the WLAN Pi tooling will contend with it.

### WLAN Pi's own helpers

| Command | Does |
|---|---|
| `wifichannel` | Channel ↔ centre frequency, both directions. `-2.4`, `-5`, `-6` list a whole band — and the 6 GHz listing marks **PSC** channels, which is what you need when picking a channel clients will actually scan |
| `reachability` | One-shot gateway / DNS / internet check |
| `wlanpi-update` | The supported image update path. Use this rather than raw apt |
| `wavemon -i wlan0` | Live signal, noise and rate display. Genuinely useful while walking |
| `profiler` | At `/usr/sbin/profiler` — see [Profiler](#profiler) |

```bash
wifichannel 5975
wifichannel -6 | grep "PSC: Yes"    # the channels clients actively scan
```

### Identify the radio

```bash
lspci -nn | grep -i network              # model + PCI id
lspci -k | grep -A3 -i network           # plus the driver in use
sudo dmesg | grep -i iwlwifi             # firmware load, errors
iw dev                                   # interfaces, type, channel
```

#### Capabilities

```bash
iw phy phy0 info | grep -E "^[[:space:]]*Band [0-9]+:"   # 1=2.4  2=5  4=6 GHz
iw phy phy0 info | grep -i eht                        # Wi-Fi 7 / 802.11be
iw phy phy0 info | grep -E "320 MHz|160 MHz"            # channel widths
iw phy phy0 info | sed -n '/Supported interface modes/,/Band 1/p'
```

Per-channel state is the part that matters when something won't transmit. `(disabled)` means the regulatory domain removed the channel; `(no IR)` means it exists but the radio may not initiate radiation on it:

```bash
iw phy phy0 info | grep "5975.0 MHz"
```

### Regulatory domain

```bash
iw reg get                    # prints global AND phy#0 — read both
sudo iw reg set CA            # US · CA · CZ · GB …
sudo dmesg | grep -iE "iwlwifi.*(reg|country|lar)"
```

#### Two regulatory models, and why yours ignores you

`iw reg get` prints two blocks because Linux has two ways of deciding what a radio may transmit:

| Model | Rules live in | Does `iw reg set` work? |
|---|---|---|
| **Kernel-managed**<br>most Atheros, MediaTek, Realtek, Broadcom | The kernel's `regulatory.db` | **Yes** — you tell it the country and it believes you |
| **Self-managed**<br>Intel, and this card | The device's own firmware | **No** — it determines its own location and enforces its own rules |

This is a liability decision, not a technical one. If software could declare any country, a device could be made to transmit illegally and the certification would be worthless. Intel's answer is that the firmware decides and the operating system doesn't get a vote.

#### How a self-managed radio works out where it is

It has two possible inputs:

* **ACPI tables from the platform firmware.** On an x86 laptop the OEM stamps the regulatory data into BIOS. The card reads it at boot and knows its country immediately.
* **Location Aware Regulatory (LAR).** The card listens for beacons and reads the *Country Information Element* that APs broadcast, then infers its location from what it hears.

**A Raspberry Pi CM4 has no ACPI regulatory tables.** That input simply doesn't exist here, so the card falls back entirely on LAR — and until it hears a beacon it has no idea where it is.

#### What country 00 means

Not "no rules" — the opposite. `country 00` is the **world regulatory domain**: roughly the intersection of every country's restrictions, which is what a device that doesn't know its location must assume. No 6 GHz at all, most of 5 GHz passive-scan or no-IR, and 2.4 GHz trimmed to the channels legal everywhere. Every "why won't it transmit" problem in this guide traces back to it.

> **The RF-isolated chamber problem.** A self-managed card booted in a shielded chamber hears no beacons, never resolves, and stays in the world domain indefinitely. There is no command that rescues it — which is a genuine nuisance for anyone testing in an anechoic or shielded environment.
>
> The standard answer is to put a reference AP inside the chamber broadcasting the Country IE for the domain under test. The card learns from it exactly as it would in the wild. That is normal test-lab practice, not a workaround.
>
> The same mechanism is worth understanding honestly: the Country IE is unauthenticated, so a radio that trusts it can be taught the wrong country by anything transmitting one. That is a known limitation of the design. Using it to operate outside your own regulatory allowance is the part that is unlawful — the mechanism is neutral, the transmission is what regulators care about.

#### Don't memorise which vendors are which

Vendor generalisations break down — Broadcom spans both models, with the FullMAC `brcmfmac` parts firmware-driven and older SoftMAC ones kernel-managed. The card will tell you, and one command beats a table:

```bash
iw reg get     # does the phy line say "(self-managed)"?
```

#### What happens when the air disagrees

LAR is not "last beacon wins." Implementations aggregate across multiple BSSIDs, weight by signal, and want a consistent picture before committing — the exact thresholds aren't published.

The behaviour that *is* predictable is what happens under conflict, and it follows from one principle running through all of this:

**Regulatory logic fails toward restriction, never toward permission.**

A card hearing one AP advertising `CN` and another advertising `CA` does not pick one, and certainly does not pick the more permissive. It now has *less* confidence about its location than before, and the conservative answer to low confidence is the intersection of the candidates — in practice the most restrictive of them, or a retreat to `country 00`.

The practical consequences are worth holding on to:

* Adding a contradictory beacon to a live environment makes your position **worse**, not better — a card that had resolved to a usable domain drops back toward the world domain and loses channels it had.
* This is exactly why the shielded-chamber technique works and the same trick on a lab bench doesn't. **Isolation removes the conflict.** One reference AP in a quiet room gives an unambiguous story; an open office gives an argument, and arguments resolve downward.
* It is also the honest answer to whether this is a security hole. It's a trust weakness, but one that **fails safe** — the easy direction to push a radio is toward fewer privileges, not more.

And if the goal were more spectrum, the choice of target matters: `CN` is *more* restrictive than `CA` in 5 GHz and has no 6 GHz allocation at all. `US` is the permissive one. Picking the wrong country costs you channels you already had.

#### What actually moves it

A scan. That's all — the supplicant doesn't need to associate:

```bash
sudo wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf
iw reg get | sed -n '/phy#0/,$p' | head -3
```

Setting `country=` in `wpa_supplicant.conf` is the closest thing to a persistent lever, and it still only applies when the supplicant runs. **The domain resets to `00` on every boot**, so check it before trusting any channel rather than assuming last session's state survived.

### Interface up, down, and unblocked

```bash
sudo ip link set wlan0 up
sudo ip link set wlan0 down
sudo iw dev wlan0 set type managed       # back from monitor mode
sudo iw dev wlan0mon del                 # remove a stale monitor vif
```

If the interface refuses to come up or scan, check for a soft block before anything else — it's the single most common dead end:

```bash
rfkill list
sudo rfkill unblock wifi
```

And when something else already holds the radio:

```bash
sudo fuser -v /dev/wlan0 2>/dev/null; sudo lsof -i | grep wlan0
systemctl list-units --state=running | grep -iE "kismet|scanner|profiler|wpa"
```

### Scanning

```bash
sudo iw dev wlan0 scan                            # full scan (needs root)
sudo iw dev wlan0 scan | grep -iE "SSID|signal|freq"
sudo iw dev wlan0 scan -u                         # include vendor IEs
```

Scan specific frequencies — far faster than a full sweep, and the only sane way to look at 6 GHz PSC channels:

```bash
sudo iw dev wlan0 scan freq 5975 6055 6135 6215 | grep -E "BSS|SSID"
```

Pull the BSSID, frequency and SSID together:

```bash
sudo iw dev wlan0 scan | awk '/^BSS/{b=$2} /freq:/{f=$2} /SSID:/{print b, f, substr($0, index($0,$2))}'
```

### Joining a network

#### Open — and MAC-based authentication

No config file needed. This is also how you join an **MBA** network: MAC-Based Authentication is open from the client's point of view — the AP authorises your MAC against RADIUS and the client does nothing special.

```bash
sudo ip link set wlan0 up
sudo iw dev wlan0 connect "Guest-WiFi"
sudo dhclient wlan0
```

For MBA testing you need to know the MAC being presented, and it must be stable — a randomised MAC will fail authorisation unpredictably:

```bash
ip link show wlan0 | grep ether
```

Captive portals: find where you're being redirected before reaching for a browser.

```bash
curl -v http://example.com 2>&1 | grep -i location:
```

#### Everything else goes through wpa_supplicant

Edit **`/etc/wpa_supplicant/wpa_supplicant.conf`**. The header applies to every profile below:

```
ctrl_interface=/run/wpa_supplicant
update_config=1
country=CA
```

`country=` is the practical way to set the regulatory domain on this device — it applies when the supplicant starts.

#### WPA2-PSK

```
network={
    ssid="MyNetwork"
    key_mgmt=WPA-PSK
    psk="passphrase"
}
```

To avoid the passphrase sitting in cleartext, generate the hashed form:

```bash
wpa_passphrase "MyNetwork" "passphrase"
```

#### WPA3-Personal (SAE)

```
network={
    ssid="MyNetwork"
    key_mgmt=SAE
    sae_password="passphrase"
    ieee80211w=2
}
```

`ieee80211w=2` is PMF *required*, which WPA3 mandates. Omit it and association fails.

#### WPA2/WPA3 transition

```
network={
    ssid="MyNetwork"
    key_mgmt=SAE WPA-PSK
    psk="passphrase"
    sae_password="passphrase"
    ieee80211w=1
}
```

#### 802.1X — PEAP-MSCHAPv2

```
network={
    ssid="CorpNet"
    key_mgmt=WPA-EAP
    eap=PEAP
    identity="username"
    password="password"
    phase2="auth=MSCHAPV2"
    ca_cert="/etc/wpa_supplicant/ca.pem"
}
```

Omitting `ca_cert` connects without validating the server certificate. Fine for a quick test, wrong everywhere else — and it hides exactly the misconfiguration you're usually hunting.

#### 802.1X — EAP-TLS

```
network={
    ssid="CorpNet"
    key_mgmt=WPA-EAP
    eap=TLS
    identity="user@example.com"
    ca_cert="/etc/wpa_supplicant/ca.pem"
    client_cert="/etc/wpa_supplicant/client.pem"
    private_key="/etc/wpa_supplicant/client.key"
    private_key_passwd="keypassword"
}
```

#### WPA3-Enterprise

As above, but swap the key management and require PMF:

```
    key_mgmt=WPA-EAP-SHA256
    ieee80211w=2
```

### Bring it up, and tear it down

```bash
sudo wpa_supplicant -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf
sudo wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf   # background
sudo wpa_supplicant -dd -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf  # debug
sudo dhclient wlan0
```

Run it in the **foreground** the first time — the association and EAP exchange scroll past live, and `-dd` is the difference between "it didn't work" and knowing which EAP step failed. Only background it once the profile is proven.

```bash
sudo killall wpa_supplicant
sudo dhclient -r wlan0
sudo ip addr flush dev wlan0
```

Clean up properly between attempts. Stale `dhclient` processes are a classic source of "it worked yesterday" — `ps aux | grep dhclient` and kill strays.

### Verify

```bash
iw dev wlan0 link              # associated? BSSID, signal, rate
iw dev wlan0 info              # mode, channel, MAC
ip addr show wlan0             # did DHCP land?
ip route                       # default gateway
ping -c 3 -I wlan0 8.8.8.8     # force the source interface
wavemon -i wlan0               # live signal
```

The `-I wlan0` on ping matters on this device — with ethernet also connected, an unqualified ping happily tests the wrong interface and tells you nothing.

#### Files you'll edit

| Path | Holds |
|---|---|
| `/etc/wpa_supplicant/wpa_supplicant.conf` | Network profiles, `country=` |
| `/etc/wlanpi-profiler/config.ini` | Profiler channel, interface, SSID, feature toggles |
| `/etc/wlanpi-wconsole/conf/hostapd.conf` | Wi-Fi Console SSID and passphrase |
| `/var/www/html/profiler/` | Profiler output — clients and reports |

## Keeping it updated

> **Read this first.** **Debian 11 "bullseye" reached end of life on 31 August 2026.** Its LTS window closed after five years, and security updates have stopped. That is why the security repository's Release file shows as expired — the repo is no longer being maintained, not misconfigured.
>
> Everything below still works, but understand what you're doing: repairing third-party repos on an EOL base gets Grafana and InfluxDB updating again while the operating system itself stops receiving security fixes. **The real fix is a reflash to a current image.**

### Use the supported updater first

Before reaching for apt, there is a WLAN Pi tool that knows about this image:

```bash
sudo wlanpi-update
```

It handles the project's own packages as a set. Raw `apt` is for everything underneath it, and is where the errors below come from.

### The three tiers, in order of nerve required

| Command | What it does |
|---|---|
| `apt update && apt upgrade` | Routine and safe. Upgrades packages without adding or removing any. Run it often |
| `apt full-upgrade` | Also allows adds and removals. Needed when `upgrade` reports packages **kept back** — which it does quietly, with no error. On a stable release you can simply use this as your default |
| Release upgrade | Editing sources to a new Debian release. A planned procedure, never an experiment |

*`full-upgrade` and `dist-upgrade` are the same command; `full-upgrade` is the newer, less misleading name.*

#### Release upgrade rules

* Read that release's upgrade notes first — they list that release's specific traps
* One release at a time. bullseye → bookworm → trixie, never skipping
* Disable third-party repos *before* starting
* Use `full-upgrade`, not `upgrade`
* Never over bare SSH — use `tmux` or the console, so a dropped link doesn't leave a half-configured system
* Snapshot or back up first. On a VM this is the whole ballgame

### The four apt errors, decoded

#### 1 · bullseye-backports has no Release file

Backports are removed when a release is archived. There is no fix beyond dropping the line — comment it out in `/etc/apt/sources.list`, or repoint it at `archive.debian.org` if you specifically need something from it.

#### 2 · bullseye-security Release file expired

The EOL above. Nothing to repair — the repository stopped being updated. If you need apt to stop erroring while you plan the reflash, point it at the archive:

```
deb http://archive.debian.org/debian-security bullseye-security main
```

Archived releases keep serving their final state with an expired Release file, so apt also needs telling not to enforce the validity date. Understand that you're disabling a freshness check, and that you are pinning yourself to packages that will never be patched again.

#### 3 · InfluxDB — NO_PUBKEY DA61C26A0585BD3B

InfluxData rotated their package signing key. Import the current one:

```bash
curl -s https://repos.influxdata.com/influxdata-archive_compat.key \
    | gpg --dearmor \
    | sudo tee /etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg > /dev/null
```

#### 4 · Grafana — EXPKEYSIG 963FA27710458545

Grafana's signing key expired on 23 August 2025 and has since been rotated. Move to the modern keyring layout:

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://apt.grafana.com/gpg.key \
    | sudo gpg --dearmor -o /etc/apt/keyrings/grafana.gpg
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" \
    | sudo tee /etc/apt/sources.list.d/grafana.list
```

Then `sudo apt update`. Errors 3 and 4 are genuine fixes. Errors 1 and 2 are structural, and only a reflash resolves them properly.

> **On the WLAN Pi specifically,** treat this device as an appliance. It is a curated image where services, nginx, and the API are wired together deliberately — a release upgrade in place is far more likely to break that arrangement than to modernise it. Back up anything you care about, then reflash the SD card when the current image ships. That's the supported path, and it's less work than fighting the packaging.

## Open questions

Unresolved as of September 2026, in priority order. Each carries the evidence needed to make it a specific question rather than a vague one — these are what I want to ask the WLAN Pi folks in Prague.

### 1 · What is the current image, and what is the upgrade path?

This unit runs Debian bullseye, which reached **end of life on 31 August 2026** — no further security updates. Ask: which image is current, where is it downloaded from, is a clean SD flash the only route, and does any configuration survive it? Worth asking whether an in-place release upgrade is ever supported, or whether reflash is always the answer.

### 2 · Can the profiler ever beacon on 6 GHz?

Bring the specifics — this is a much better question with them:

* Intel BE200, firmware falls back to **90** because the image disabled 92
* LAR resolves on a supplicant scan: `country 00` → `CA`
* 6 GHz then appears as `IR-CONCURRENT` / `PASSIVE-SCAN`, channels `(no IR)`
* Profiler reports beaconing; `tx_packets` stays at **0**

Then: is 6 GHz profiling possible on *any* Intel part, or is a different chipset mandatory — and which card specifically do they recommend?

### 3 · Which image does the MCP server target?

`wlanpi-mcp` declares `Pre-Depends: python3 (>= 3.13)`, which means Debian trixie. Bullseye has 3.9 and bookworm only reaches 3.11. Is the MCP work aimed at a trixie-based image, and has that image shipped? It also expects a newer `wlanpi-core` than the 2.0.0 here — what core version is the real floor?

### 4 · Why is firmware 92 disabled?

`iwlwifi-gl-c0-fm-c0-92.ucode` is renamed `.disabled` with `.original` kept beside it, root-owned, dated January 2025. Presumably 92+ broke monitor mode or injection. Worth confirming what it broke, whether it's expected to be resolved, and whether that pin is part of why 6 GHz stays shut.

### 5 · Should `security_mode` be in config.ini?

It isn't — the file exposes only `channel`, `interface`, `ssid`, the four `*_disabled` toggles, `listen_only`, `hostname_ssid` and `files_path`. Adding a `security_mode` key is silently ignored; security lives solely in the `--wpa3_personal` / `--wpa3_personal_transition` CLI flags.

Since the GUI only starts the systemd service with file defaults, **a GUI-started profiler always beacons WPA2** — and without WPA3 with PMF required, a client will not advertise EHT. **So the web UI structurally cannot produce a correct Wi-Fi 7 client profile**, and reports a Wi-Fi 7 phone as "802.11be Not supported." Bug, or deliberate? Reasonable feature request either way — exposing the existing flag in `config.ini` would close it.

### 6 · Is pcap parsing the intended route to full detail?

The text report is a curated subset. Full HE MAC/PHY capability bits, RM Enabled Capabilities detail, Power Capability, and vendor IEs such as MBO/OCE are all in the saved association frame but not surfaced. Is `--pcap` re-analysis plus Wireshark the expected path, or is there appetite for richer reports?

### 7 · How should the regulatory domain be handled when travelling?

The phy is self-managed, so `iw reg set` doesn't move it and the card relies on hearing beacons. Taking a Pi from Canada to Czechia changes which channels are legal — channel 149 in `config.ini` is fine at home and unavailable in Europe. Is there a supported way to pin this, or is the practice simply to pick channels legal everywhere?

## Reference

#### Project & documentation

* [WLAN Pi](https://www.wlanpi.com/) — official project site
* [User Guide (V2)](https://userguide.wlanpi.com/) — current docs: hardware, OS, modes, capture
* [Documentation Project](https://wlan-pi.github.io/wlanpi-documentation/) — older V1 docs, still useful for tooling
* [GitHub organisation](https://github.com/WLAN-Pi) — all source, releases, and issue tracking

#### Key repositories

* [wlanpi-core](https://github.com/WLAN-Pi/wlanpi-core) — the FastAPI backend everything calls
* [wlanpi-profiler](https://github.com/WLAN-Pi/wlanpi-profiler) — client capability analyser
* [wlanpi-wconsole](https://github.com/WLAN-Pi/wlanpi-wconsole) — Wi-Fi Console mode, with the full port map
* [wlanpi-mcp](https://github.com/WLAN-Pi/wlanpi-mcp) — MCP server; needs Python 3.13, so trixie

#### Hardware

* [Purchase](https://www.wlanpi.com/purchase) — official reseller list
* [BadgerWiFi · UK](https://www.badgerwifi.co.uk/) — run by Nick Turner, who teaches the Prague deep dive
* [Big QAM · USA](https://bigqam.com/wlan-pi-m4-plus) — run by Josh Schmelzle, core WLAN Pi contributor

#### Community

* [WLAN Pi Resources](https://wlanprofessionals.com/wlan-pi-resources/) — Wireless LAN Professionals' collected material
* [WLAN Pi Advanced · PRG 26](https://www.thewlpc.com/presentations/wlan-pi-advanced-hardware-hacking-wi-fi-7-str-and-ai-interaction-with-wlan-pi-prg-26) — Prague deep dive: hardware, Wi-Fi 7 STR, MCP
* [Issues & discussion](https://github.com/WLAN-Pi/wlanpi-documentation/issues) — the project's practical support channel

*The project has no confirmed public Slack or Discord — GitHub is the primary channel, and the community congregates in person at WLPC events.*

#### Upstream sources

* [Debian 11 LTS end-of-life](https://www.debian.org/News/2026/20260831) — the official EOL announcement, 31 Aug 2026
* [Grafana GPG key rotation](https://grafana.com/blog/grafana-security-update-gpg-signing-key-rotation/) — why the signature expired, and the current key
* [InfluxData key rotation](https://www.influxdata.com/blog/package-signing-key-rotation/) — background on the missing public key

---

*Written against a WLAN Pi running `wlanpi-core` 2.0.0 on Debian bullseye, September 2026. Addresses, versions and the Grafana account name are specific to my unit — the procedures are general. Defaults quoted from WLAN Pi documentation should be verified against your own image version before relying on them in the field.*
