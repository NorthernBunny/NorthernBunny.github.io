---
layout: post
title: "Building a Kali Pi Network Tool"
categories: [tech]
description: >
  Notes on turning a Raspberry Pi 4 into a portable wireless diagnostic kit —
  Kali Linux, a Wi-Fi 6E Alfa radio, a HackRF Pro, the tier-3 toolkit, and an
  MCP server so an LLM can drive it.
---

Inspired by the YouTube video by David Bombal, [Raspberry Pi 5 Kali Linux install in 10 minutes (with WiFi hacking)](https://www.youtube.com/watch?v=8Slbc9r53d8), and by the [WLAN Pi](https://www.wlanpi.com/) project, I decided to build a network tool by taking a Raspberry Pi 4 Model B with 8 GB of RAM, an **Alfa AWUS036AXML** USB adapter (MediaTek **MT7921AU** chipset — Wi-Fi 6E, so 2.4 / 5 / 6 GHz), a **HackRF Pro** from [Great Scott Gadgets](https://greatscottgadgets.com/), and some additional Linux utilities and an MCP server. Here are the notes!
{:.lead}

The goal is a pocket wireless-diagnostics box: scan, capture, test throughput, hunt interference, and stand up a controlled test AP — all over SSH or a remote desktop, powered from a wall brick, a battery pack, or a laptop's USB-C. Hostnames, usernames, MAC addresses and IPs below are placeholders; everything else is as-built.

* this list will be replaced by the table of contents
{:toc}

## The build

| | |
|---|---|
| Board | Raspberry Pi **4 Model B, Rev 1.5** · 8 GB RAM |
| Storage | 64 GB microSD, expanded to fill the card |
| OS | Kali GNU/Linux Rolling · kernel 6.12.x |
| Wired | Gigabit ethernet, DHCP — reached by mDNS name so a shifting IP doesn't matter |
| Radio 0 | Onboard Broadcom Wi-Fi (`brcmfmac`) — 2.4 GHz only |
| Radio 1 | **Alfa AWUS036AXML** · MediaTek MT7921AU (`0e8d:7961`, driver `mt7921u`) — **2.4 / 5 / 6 GHz, Wi-Fi 6E** |
| SDR | HackRF Pro — 100 kHz to 6 GHz, on a USB 3.0 port |

One nice detail: MAC prefixes identify the hardware. `00:c0:ca` is ALFA Network's registered vendor code and `d8:3a:dd` is the Raspberry Pi Foundation's — handy for picking the unit out of a scan or an ARP table.

### Port map, and the two structural constraints

The Pi 4 has **one USB-C** and **one 40-pin header**, and almost every design decision falls out of that.

| Port | Use |
|---|---|
| USB-C | Power in **or** USB-gadget (OTG) to a host — not both |
| 2× USB 3.0 | The blue pair nearest the RJ45. Best for the HackRF and the Alfa — more bandwidth, cleaner power |
| 2× USB 2.0 | The outer pair — fine for a keyboard |
| 2× micro-HDMI | Video out |
| RJ45 | Gigabit ethernet, data only — **no PoE without a HAT** |
| 40-pin GPIO + 4-pin PoE header | Where a display HAT, a PoE HAT, and GPIO power all compete |

> **The two gotchas that shape the whole build.** First, **USB-C is either power or OTG, never both** — so OTG networking means running off the host's power. Freeing the USB-C for OTG while still powering the box requires feeding 5 V into the GPIO pins, which bypasses the input protection, so it's a deliberate choice rather than a casual one. Second, **there's one 40-pin header**, and a display HAT, a PoE HAT and GPIO power all want it. Stacking needs standoff extenders and gets tall and fragile. The build has to commit to a purpose.

### Power options

* **USB-C PSU** — simplest and most reliable, full 3 A. The bench default. It needs a real 5 V/3 A supply; the HackRF and Alfa together draw a lot.
* **OTG / USB-gadget** — one USB-C cable from a laptop both powers the Pi *and* creates a direct network link, so SSH works with no Wi-Fi or LAN at all. The catch is that the laptop port powers it, and can brown out under RF load.
* **PoE+ HAT** — one cable for power and network, for a fixed sensor. Not native: it needs the official 802.3at HAT on the 4-pin header plus a PoE switch or injector, and it occupies the GPIO stack.
* **USB-C battery** — true standalone. Needs 3 A or PD, and it frees the ethernet port and the Wi-Fi for the actual job.

Enabling OTG networking is two config edits and a reboot:

```
# /boot/firmware/config.txt
dtoverlay=dwc2

# /boot/firmware/cmdline.txt — add right after rootwait:
modules-load=dwc2,g_ether
```

### Screen and buttons

There's no off-the-shelf "diagnostic tool in a box" kit — it's a two-part DIY of an aluminium body plus a display HAT that has to be scripted.

| Kit | What it offers |
|---|---|
| Waveshare 1.3" LCD HAT | 240×240 IPS plus a joystick and 3 buttons, around $15. Closest to the idea, and ships a Python library with demos |
| Pimoroni Display HAT Mini | 2.0" 320×240 with 4 buttons — the best-documented Python library |
| Adafruit PiTFT 2.4"/3.5" | Touchscreen with a few GPIO buttons |
| Argon ONE V2 case | Aluminium, cooling, power button, GPIO break-out. A good body to pair with a HAT |
| Flirc case | Aluminium passive heatsink, gorgeous — but it seals the GPIO, so no HAT |

> **Borrow the UI, don't reinvent it.** The real WLAN Pi is open source, and its front-panel menu system (`wlanpi-fpms`, on GitHub) already drives exactly this kind of small-screen-plus-buttons interface: status, IP, mode, start/stop capture. Adapting that beats writing a screen daemon from scratch.

### The fork: two personas

* **Field handheld** — USB-C battery or OTG-to-laptop, a display HAT with a joystick for at-a-glance status, rugged case, no PoE. The WLAN-Pi-style tool.
* **Fixed sensor** — PoE+ HAT so one cable does power and network, headless, deployed in a rack or ceiling. No screen.

They want incompatible things from that one header, which is why it's a fork and not a feature list.

## Getting in

### SSH, by name

DHCP moves the address around, so reaching the unit by mDNS name saves a lot of hunting:

```bash
ssh user@kali-pi.local
```

Worth doing once: copy a public key over, and the login password is never needed again — which also makes `scp` and the VNC tunnel prompt-free.

```bash
ssh-keygen -t ed25519      # skip if a key already exists
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@kali-pi.local
```

If the name doesn't resolve, the Pi Foundation's vendor prefix finds it on the LAN:

```bash
arp -a | grep -i d8:3a:dd
```

The better fix is a DHCP reservation on the router so the address stops moving.

### Remote desktop over an SSH tunnel

TigerVNC serves the Xfce desktop on display `:1` / port 5901, **bound to localhost**, reached through an SSH tunnel. That matters: nothing extra is exposed on the network, which is the right posture when the unit is plugged into a site nobody on the team controls.

One-time setup on the Pi:

```bash
vncpasswd      # max 8 chars, separate from the login password
```

```bash
printf '#!/bin/sh\nunset SESSION_MANAGER\nunset DBUS_SESSION_BUS_ADDRESS\nexec dbus-run-session -- startxfce4\n' > ~/.vnc/xstartup
chmod +x ~/.vnc/xstartup
```

> **The classic "VNC exits in under 3 seconds" trap.** If the desktop dies immediately with *"session exited too early,"* the cause is a **missing D-Bus session** — Xfce quits instantly without one. The `dbus-run-session` wrapper above is the fix, and `-xstartup ~/.vnc/xstartup` has to be passed explicitly so the server uses that file rather than the system default. This one cost an evening.

Then from the laptop, one alias does tunnel-plus-viewer in a single word:

```bash
alias pivnc='ssh -fNL 5901:localhost:5901 user@kali-pi.local && open vnc://localhost:5901'
```

`-fN` backgrounds the tunnel so there's no window to keep open. It closes later with `pkill -f 5901:localhost:5901`.

### Auto-start VNC on boot

So it never needs hand-starting again — a **systemd user service** with `linger` enabled, which means it runs even before anyone logs in:

```bash
sudo loginctl enable-linger $USER
mkdir -p ~/.config/systemd/user
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now vncserver.service
systemctl --user status vncserver.service
```

The unit file just runs `tigervncserver -fg :1 -localhost yes -geometry 1440x900 -depth 24 -xstartup <path to xstartup>`, with `Restart=on-failure`. The `-fg` matters — it keeps the server in the foreground so systemd can actually track it.

### tmux, so nothing dies with the SSH session

A terminal multiplexer keeps shells running on the Pi independently of the connection. Start a long job, detach, close the laptop, reconnect later, reattach — the job never noticed. Everything starts with the prefix, <kbd>Ctrl</kbd>+<kbd>b</kbd>.

```bash
tmux new -s work          # start a named session
# ... run the long command ...
# press Ctrl-b then d to detach, and walk away
tmux attach -t work       # come back later
```

| Action | Keys |
|---|---|
| New named session | `tmux new -s NAME` |
| Detach, leaving it running | <kbd>Ctrl-b</kbd> <kbd>d</kbd> |
| List sessions | `tmux ls` |
| Re-attach | `tmux attach -t NAME` |
| Split left/right, top/bottom | <kbd>Ctrl-b</kbd> <kbd>%</kbd> / <kbd>Ctrl-b</kbd> <kbd>"</kbd> |
| Move between panes | <kbd>Ctrl-b</kbd> <kbd>←↑↓→</kbd> |
| Scroll-back mode (q exits) | <kbd>Ctrl-b</kbd> <kbd>[</kbd> |

The field habit: SSH in, attach to tmux, and run every capture, scan or upgrade inside it. A dropped Wi-Fi link or a closed lid then can't strand anything mid-job.

## Keeping Kali healthy

Kali is a rolling release, so it drifts fast when shelved. This is the routine that actually completes:

```bash
sudo apt update

# run this one INSIDE tmux
sudo DEBIAN_FRONTEND=noninteractive apt-get full-upgrade -y \
  -o Dpkg::Options::="--force-confdef" -o Dpkg::Options::="--force-confold"

sudo apt autoremove --purge -y && sudo apt clean
sudo reboot
```

> **Why `full-upgrade` and not `apt upgrade`.** On a rolling distro, plain `apt upgrade` refuses to add or remove packages when dependencies shift, so it quietly holds a big chunk back and leaves the upgrade half-done with no error at all. `full-upgrade` resolves those and finishes. The `noninteractive` plus `confdef`/`confold` flags auto-answer the "keep the existing config or take the new one?" prompts — **preserving local edits**, and taking the package default only for files that were never touched.

### When an upgrade stalls

A big rolling upgrade can trip on one package. Mine did, and two moves cleared it:

* **"trying to overwrite X, which is also in package Y"** means a package was renamed without declaring it. Forcing the file overwrite with `dpkg` directly, then letting apt sort out the ordering, is the fix. Going straight to dpkg matters — apt refuses at its planning stage and never applies the override.
* **Recovery should never run interactively.** Prefixing *every* apt and dpkg command with `DEBIAN_FRONTEND=noninteractive` means a stray debconf question can't stop and wait indefinitely.

```bash
sudo dpkg -i --force-overwrite /var/cache/apt/archives/<package>.deb
sudo DEBIAN_FRONTEND=noninteractive apt-get --fix-broken install -y
sudo DEBIAN_FRONTEND=noninteractive apt-get full-upgrade -y
```

That last one should end with "0 upgraded, 0 to remove, 0 not upgraded."

### Two gotchas worth knowing about

> **The Kali signing-key rotation (Feb 2025).** Any image older than that hits `Missing key …ED65462EC8D5E4C5` and apt refuses to update — which is correct, safe behaviour. The fix is Kali's new keyring, and **the checksum must be verified before trusting it**:
>
> ```bash
> sudo wget https://archive.kali.org/archive-keyring.gpg \
>   -O /usr/share/keyrings/kali-archive-keyring.gpg
> sha1sum /usr/share/keyrings/kali-archive-keyring.gpg
> # must print exactly:
> # 603374c107a90a69d983dbcb4d31e0d6eedfc325
> ```
>
> Source: [kali.org/blog/new-kali-archive-signing-key](https://www.kali.org/blog/new-kali-archive-signing-key/). If the hash doesn't match, stop — that keyring shouldn't be used.

> **Captures will fill the card.** A Kismet session left running quietly wrote a **45 GB** `.kismet` file and filled the SD card to 100%, which blocks apt entirely and is a confusing failure to diagnose from the symptoms. Time-box captures, and sweep up after:
>
> ```bash
> df -h /                          # check headroom first
> du -xhd1 ~ | sort -rh | head     # find the hogs
> rm -i ~/*.kismet ~/*.pcapng
> ```

## The two radios

Two independent Wi-Fi interfaces, playing different roles. The onboard radio handles management and uplink; the Alfa is dedicated to capture and test work — it's better silicon and the only one that reaches 5 and 6 GHz.

| Trait | Onboard | Alfa (USB) |
|---|---|---|
| Chipset | Broadcom | MediaTek MT7921AU |
| Driver | `brcmfmac` | `mt7921u` |
| Bands | 2.4 GHz | **2.4 / 5 / 6 GHz**, Wi-Fi 6E |
| Modes | managed · monitor · AP · IBSS · P2P | managed · **monitor** · **AP** · **AP/VLAN** · P2P |
| PHY caps | HT20/40, basic | HT40, **RX LDPC**, Greenfield |
| Role | Management / uplink | **Capture and test work** |

### What those PHY capabilities actually mean

* **RX LDPC** (Low-Density Parity Check) is a stronger error-correcting code in 802.11n/ac/ax, worth roughly 1.5–2 dB of coding gain. The card can decode weak, noisy frames that a basic radio drops, so it **captures more completely at range**. That's the single biggest reason to use the Alfa as the sniffer.
* **RX Greenfield** is an 802.11n high-throughput preamble that abandons legacy compatibility. Rare in the wild, but the card won't miss those frames — mostly it's a marker of a fuller PHY implementation.
* **IBSS** is classic ad-hoc mode, where peers talk directly with no access point. Largely legacy now; Wi-Fi Direct and mesh replaced it.
* **AP/VLAN** is real 802.1Q VLAN support when the card runs as an AP via `hostapd`. That allows a test SSID that tags clients into different VLANs, including RADIUS dynamic-VLAN assignment. **The field use is good:** validating a switch trunk, a DHCP scope, and a RADIUS VLAN policy from the wireless side, using a known-good AP.

### Fingerprinting any adapter

Drop-in commands to identify an unknown card and confirm what it can really do:

```bash
lsusb | grep -iE 'mediatek|alfa|ralink|realtek|atheros'   # card + USB VID:PID
sudo ethtool -i wlan1                                     # driver + firmware version
iw dev wlan1 info                                         # interface, phy, current mode
iw phy phy1 info | grep -E 'Band [0-9]:'                  # bands: 1=2.4  2=5  4=6 GHz
iw phy phy1 info | sed -n '/Supported interface modes/,/valid interface/p'
iw phy phy1 info | grep -E 'MHz \[' | grep -v disabled    # channels actually enabled
```

Band **1** is 2.4 GHz, **2** is 5 GHz, **4** is 6 GHz. An enabled channel shows a dBm power; a locked one shows `(disabled)`, which is almost always the regulatory domain. This is how the AXML was confirmed to really be a 6E card, rather than trusting the box it came in.

### Pin the MACs to the real hardware addresses

NetworkManager randomises Wi-Fi MACs by default, both during scans and during connections. That's good privacy behaviour and actively unhelpful when the goal is finding known gear in a capture. Pinning everything to the permanent burned-in addresses:

```bash
printf '[device]\nwifi.scan-rand-mac-address=no\n\n[connection]\nwifi.cloned-mac-address=permanent\nethernet.cloned-mac-address=permanent\n' | sudo tee /etc/NetworkManager/conf.d/99-no-mac-random.conf
sudo systemctl restart NetworkManager
```

After that, `ip -br link` shows the real MACs. To spoof deliberately for a test, `macchanger` does it; `sudo macchanger -p wlan1` puts it back.

Related: during the upgrade, `macchanger` asks whether to auto-randomise the MAC on every interface up/down. For this build the answer is **No** — stable, known MACs are exactly what's needed for filtering in Wireshark.

## Field recipes

> **Authorization — the hard rule.** Monitor mode, scanning, capture, ping and iperf are **passive**. But anything that **transmits** — injection, deauthentication, running an AP — can disrupt a network, and deauthentication in particular is a denial-of-service. That belongs only on networks the operator owns or has **explicit written authorization** to test. Against a third party it's illegal in most jurisdictions, and "it was just a test" is not a defence. This box is a diagnostic tool for owned gear and sanctioned work.

### 0 · Set the regulatory domain first

A fresh boot comes up in the world domain (`country 00`), which disables every 6 GHz channel and forces 5 GHz DFS channels to passive. Setting the actual country unlocks them:

```bash
sudo iw reg set CA        # the operating country
iw reg get | grep country
```

To make it permanent across reboots:

```bash
echo 'options cfg80211 ieee80211_regdom=CA' | sudo tee /etc/modprobe.d/regdom.conf
```

This is the single most common reason "6 GHz doesn't work" on a card that definitely supports 6 GHz.

### 1 · Monitor mode

```bash
sudo airmon-ng check kill
sudo airmon-ng start wlan1        # becomes wlan1mon
```

Or the manual equivalent, worth knowing because `airmon-ng` isn't always available or wanted:

```bash
sudo ip link set wlan1 down
sudo iw dev wlan1 set type monitor
sudo ip link set wlan1 up
sudo iw dev wlan1 set channel 36
```

Since the session runs over ethernet, killing the Wi-Fi daemons doesn't cost the connection.

### 2 · Survey the air, per band

```bash
sudo airodump-ng --band bg wlan1      # 2.4 GHz
sudo airodump-ng --band a wlan1       # 5 GHz
```

6 GHz is awkward: `--band` has no letter for it, so the PSC discovery channels get hopped by frequency instead. Those are the channels APs actually beacon on, which is where clients look first:

```bash
sudo airodump-ng -C 5975,6055,6135,6215,6295,6375,6455,6535 wlan1
```

A useful shell function — hunt one SSID across all three bands in a single word:

```bash
apscan() { sudo airodump-ng --essid "$1" -C 2412,2437,2462,5180,5200,5220,5240,5260,5500,5745,5765,5785,5825,5955,5975,6055,6135,6215 wlan1; }
```

### 2b · Kismet's web dashboard — the comfortable survey

Kismet is a **server started from the shell**, and its interface is a live **web dashboard** — there's no separate GUI app, which confuses people at first. It auto-hops every band the card supports, and any browser can view it.

```bash
# on the Pi
sudo iw reg set CA
sudo nmcli device set wlan1 managed no
mkdir -p ~/captures && cd ~/captures
sudo kismet -c wlan1
```

```bash
# from the laptop — tunnel the dashboard over SSH, then open it
ssh -fNL 2501:localhost:2501 user@kali-pi.local
open http://localhost:2501
```

On first visit it prompts for an admin login to be created in the browser. The tabs worth knowing:

| Tab | What it shows |
|---|---|
| Devices | Every AP and client — SSID, type, encryption, signal history, channel, vendor OUI, client count, QBSS load |
| SSIDs | Grouped by advertised and probed SSID — who's beaconing, and what clients are looking for |
| Alerts | Built-in WIDS — deauth floods, spoofed APs, attack signatures, new-device alerts |
| Channels | Devices-per-frequency histogram across 2.4/5/6 — congestion at a glance |
| Data Sources | Add or pause radios, lock or hop channels, watch per-source packet rates |

Everything in the UI is also a REST endpoint (`curl http://localhost:2501/system/status.json`), so it's scriptable. Adding `--no-logging` gives a live look with zero disk footprint — otherwise, remember the 45 GB lesson above.

### 3 · Capture to a file

```bash
# focused capture on one AP's channel
sudo airodump-ng -c 36 --bssid <AP-BSSID> -w cap wlan1mon
```

```bash
# or raw pcap for Wireshark
sudo tcpdump -i wlan1mon -w survey.pcap
sudo tshark -i wlan1mon -Y 'wlan.fc.type_subtype==0x08'   # beacons only
```

A safe self-test that confirms the card can inject at all, without transmitting at anyone:

```bash
sudo aireplay-ng --test wlan1mon
```

### 4 · Throughput and latency — the bread and butter

This is what most "the Wi-Fi is slow" calls actually need, and it's entirely passive.

```bash
ping -c 5 <gateway-ip>
mtr -rwzbc 20 8.8.8.8              # per-hop loss and latency report
```

```bash
iperf3 -s                          # on the far end
iperf3 -c <server-ip> -t 30 -P 4   # 4 parallel streams, 30 seconds
```

```bash
watch -n1 'iw dev wlan1 link'      # live AP, signal in dBm, TX bitrate
```

Signal rule of thumb: −30 dBm is excellent, −67 is the design floor for voice and video, −70 is marginal, −80 is poor.

One honest caveat, learned the hard way: **a single speed test is not a sound measurement.** Too many variables move between runs. Repeated, controlled runs come first — and testing the wired path before the wireless one is what splits "slow Wi-Fi" into an air problem versus a wired or WAN one.

### 5 · Put it back

```bash
sudo airmon-ng stop wlan1mon
sudo systemctl restart NetworkManager
```

## The toolkit

What's installed, by job, and — more usefully — *when each one is the right thing to reach for*.

### Wireless

| Tool | Reach for it for |
|---|---|
| kismet | Site surveys, finding rogue or unexpected APs, seeing every AP and client across 2.4/5/6 at once |
| aircrack-ng suite | Targeted capture on a specific channel (airodump-ng), injection self-tests, roaming behaviour tests |
| wavemon | Watching signal and SNR from the *client's* point of view while walking a floor |
| horst | A quick RF-health and retry-rate read without spinning up Wireshark |
| hcxdumptool / hcxtools | Auditing WPA2 strength on an owned network, or checking whether an AP exposes a PMKID. An active audit tool, not a monitor |
| bettercap | Authorized pentests — ARP/DNS-spoof testing, traffic interception, BLE recon |
| iw · rfkill · wpa_cli | Setting monitor mode, channel or band; unblocking a radio; debugging why a client won't associate |

### Wired discovery

| Tool | Reach for it for |
|---|---|
| **lldpd** (`lldpcli`) | ★ The moment the box lands on a switchport and the question is the switch name, port ID and native VLAN — "where is this plugged in?" This is the first thing to run |
| cdpr | Cisco gear that speaks CDP instead of LLDP |
| nmap · ncat | Mapping what's on a subnet, confirming a port is reachable, grabbing a banner |
| arp-scan · netdiscover | "What's *really* on this segment" — catches devices that ignore ping |
| snmpwalk | Pulling interface counters and error/discard rates to confirm a port or link problem |
| dhcping | "Is DHCP even responding on this VLAN?" during an onboarding failure |
| avahi-browse · nbtscan | Finding printers and Apple devices (mDNS), or Windows hosts (NetBIOS) |

### Throughput and visibility

| Tool | Reach for it for |
|---|---|
| iperf3 | Measuring real bandwidth, and splitting an air problem from a wired one |
| nuttcp | A second opinion, strong on UDP loss and jitter — VoIP and video path testing |
| mtr | Intermittent latency or loss, because it shows *which hop* is hurting |
| ethtool | Any "slow wired" call — confirming it negotiated gigabit full-duplex and isn't piling up CRC errors |
| tshark · wireshark · termshark | Ground truth on the wire. `termshark` is the TUI, for deep analysis over SSH with no X |
| tcpdump | Grabbing a quick pcap anywhere, to open in Wireshark later |
| iftop · nload · iptraf-ng | "What's hogging the link right now?" |
| zeek | Passive visibility and forensics — turns traffic into structured conn/dns/http/ssl logs |

### Lab and emulation

| Tool | Reach for it for |
|---|---|
| mininet | Reproducing a topology or testing routing/SDN behaviour without physical gear |
| `tc` / netem | Impairing a link on purpose — adding latency, loss, jitter or a rate cap to reproduce a customer's "it's laggy" and watch how an app copes |

`tc`/netem deserves more attention than it gets. Being able to *manufacture* the conditions someone is complaining about, on demand, is often the fastest route to understanding the complaint.

## An MCP server, so an LLM can drive it

This is the part I find most interesting. An MCP server on the Pi turns it into an AI-drivable instrument: a Claude client calls its tools and the work runs on the Pi. Brain on the laptop, hands on the Pi.

### How it's wired

* **Server** — Python (the `mcp` SDK, FastMCP) living in a venv on the Pi. Each tool runs a **fixed command with validated arguments**. There is deliberately no "run any shell command" tool.
* **Transport** — the client launches it over SSH and MCP's stdio rides the SSH pipe. This works cleanly because passwordless SSH is already set up and a non-interactive SSH prints no MOTD to corrupt the stream.
* **Discovery** — the client asks for the tool list at connect time. Nothing is hardcoded on the model's side, so adding a tool to `server.py` is all that's required.

It's **not a daemon.** Nothing runs at boot. The *client* spawns `ssh pi python server.py` when it connects, the process lives only while that client is attached, and each client gets its own copy. Checking whether it's running: `pgrep -af mcp/server.py`.

Adding a tool is just a decorated function:

```python
@mcp.tool()
def uptime() -> str:
    """How long the Pi has been up."""
    return _run(["uptime", "-p"])
```

### The tool menu

| Tool | What it does | Phrasing that triggers it |
|---|---|---|
| `pi_status` | Board, kernel, uptime, CPU temp, memory, disk, IPs, regulatory domain | "what's the status?" |
| `radios` | Each radio's driver, MAC, bands, modes, current mode | "which radios and what bands?" |
| `wifi_scan` | Nearby APs — SSID, channel, frequency, signal, security | "scan wifi, list the 5 GHz APs" |
| `rf_scan` | HackRF spectrum sweep → noise floor, strongest peaks, occupancy | "RF sweep 2400–2500" |
| `ping` | Ping a host from the Pi | "ping 8.8.8.8" |
| `iperf3_client` | Throughput test to an iperf3 server | "iperf3 to <host> for 10s" |
| `monitor_capture` 🔒 | Time-boxed 802.11 capture to a pcap — **gated**, see below | "capture 2437 MHz for 20s" |

### Why two scan tools, and how to steer between them

| Tool | Sees | Blind to |
|---|---|---|
| `wifi_scan` | Decoded 802.11 — SSID, channel, signal, security, from beacons | Anything that isn't Wi-Fi |
| `rf_scan` | **Raw RF energy** — every transmitter in the band, Wi-Fi or not | Can't decode; doesn't know *what* a signal is |

Which one runs is steered purely by wording: "Wi-Fi scan" gets `wifi_scan`, "RF" or "spectrum sweep" gets `rf_scan`, and "compare" or "first… then" gets both. The genuinely useful prompt is the correlated one:

> *"First do a Wi-Fi scan, then an RF sweep of the same 2.4 GHz band. Map each RF peak to a Wi-Fi channel, and flag any peak with no matching AP as possible non-Wi-Fi interference. Then recommend the best channel."*

That's a real diagnostic workflow — and a peak with no beacon behind it is exactly how a wireless camera or an AV sender gets found, since no Wi-Fi tool can see it at all. Rough fingerprints the model can reason from: an **analog camera** is a narrow, continuous, always-on carrier; a **microwave oven** is wide (15–20 MHz), bursty, near 2450 MHz, and only while running; **Bluetooth** is fast-hopping scattered blips across all of 2.4.

Worth instructing it to caveat: one sweep is **one spot at one moment**, values are **relative dB not dBm**, bins are coarse, and a fast burst can slip between sweeps. So the honest output is "consistent with a camera," not a verdict.

### The passphrase gate — stopping the model from acting alone

`monitor_capture` is the first *active* tool: it briefly switches the Alfa into monitor mode, captures frames, then restores managed mode. Because that touches the radio, it's gated three ways:

1. **A passphrase challenge.** The call is refused unless its `passphrase` argument matches a secret file on the Pi. No secret file means the tool is disabled — it **fails closed**. So a human has to supply the passphrase for that session, and the model can never self-authorize.
2. **A time-box.** The duration is clamped to 1–60 seconds, and the capture runs under `timeout` so it always stops.
3. **Narrow root.** Monitor mode needs root, granted through a `sudoers.d` rule that permits only the handful of specific binaries the tool needs, with no password prompt. If the rule is missing, the tool does nothing and prints the setup steps.

Verified behaviour: no passphrase is refused, a wrong passphrase is refused, and a correct passphrase with no sudoers rule is refused with setup instructions and **the radio untouched**. A `finally` block always restores managed mode.

The honest framing: **the passphrase is a human-in-the-loop confirmation, not strong crypto.** Anyone who already holds the SSH key could bypass MCP entirely and run the commands directly. What the gate prevents is the *model* deciding on its own to put a radio into monitor mode — which is the actual risk when an LLM is handed a tool that touches hardware.

### Driving it from a local LLM instead

Ollama runs a model locally but speaks OpenAI-style tool-calling rather than MCP, so a small bridge script connects the two: it launches the Pi's MCP server over SSH, hands the tool list to the model, executes any tool calls on the Pi, and loops. Tested working with `llama3.1`, which called `pi_status` and answered in plain English. The model has to support tool-calling — llama3.1/3.2, qwen2.5, mistral-nemo and similar do; a non-tool model simply never calls anything. Everything stays local: model on the laptop, tools on the Pi, no API key.

## The HackRF side

The SDR gives the kit an RF-layer view — the non-Wi-Fi interference, jammers and noise floor that Kismet and airodump are structurally blind to. Interference hunting is its killer use here.

```bash
sudo apt install -y hackrf gqrx-sdr gnuradio soapysdr-tools inspectrum
lsusb | grep -iE 'great scott|hackrf|1d50'
hackrf_info                      # Board ID 5 = Pro
hackrf_sweep -f 2400:2500        # sweep 2.4 GHz
```

It belongs on a **USB 3.0 port** (the blue pair nearest the RJ45). Full 20 MS/s IQ streaming is heavy over the Pi's USB — sweeps are fine, wide continuous captures want the USB 3 port and patience. Note also that the HackRF shares the USB bus with the Alfa, so a full-rate capture while the Alfa is in monitor mode can drop samples.

There's nothing to eject when finished: the HackRF has no battery and no filesystem state. Ctrl-C out of any running sweep or capture so the tool releases the USB device, then unplug it.

The SDR half is written up properly — antennas, the SMA vs RP-SMA trap, gqrx, QSpectrumAnalyzer, channel maps and the sweep-to-LLM recipe — in a separate [SDR Lab Notebook](/tech/2026-09-22-sdr-lab-notebook/) post, so it isn't repeated here.

## Where this goes next

* **Standalone screen and buttons** — a HAT display with a joystick in a rugged case, driven by a small status-and-menu daemon like the WLAN Pi's FPMS: show IP, radio mode, temperature, battery, plus buttons to reboot, toggle monitor mode, start and stop a capture. That's what closes the polish gap with a real WLAN Pi.
* **OTG networking** — `dwc2` plus `g_ether` so one USB-C cable to the laptop both powers the unit and makes a direct link, with no Wi-Fi or LAN needed at all.
* **Reusable field profiles** — saved Kismet and airodump configs per job (site survey, roaming test, WIDS check), plus auto-cleaning captures so the disk can never fill again.
* **A shareable image** — and specifically a *provisioning script* that reproduces the build from scratch, rather than a raw `dd` of a personal card. Then a clean flashable image, with the MCP server running as a proper HTTP service rather than per-client over SSH.

## Links worth keeping

**Kali** — [kali.org](https://www.kali.org/) · [docs](https://www.kali.org/docs/) · [Kali on Raspberry Pi](https://www.kali.org/docs/arm/raspberry-pi/) · [tool index](https://www.kali.org/tools/) · [r/Kalilinux](https://www.reddit.com/r/Kalilinux/)

**The Alfa card** — [morrownr/USB-WiFi](https://github.com/morrownr/USB-WiFi) is the canonical reference for USB Wi-Fi adapters, chipsets, drivers and monitor/injection support; the place to start for anything MT7921. [ALFA Network](https://www.alfa.com.tw/) is the vendor.

**The appliance idea** — [WLAN Pi](https://www.wlanpi.com/) and [WLAN-Pi on GitHub](https://github.com/WLAN-Pi), the closest sibling project and the one to borrow from. [Raspberry Pi PoE+ HAT](https://www.raspberrypi.com/products/poe-plus-hat/) for the fixed-sensor build.

**HackRF and SDR** — [HackRF Pro](https://greatscottgadgets.com/hackrf/pro/) · [HackRF docs](https://hackrf.readthedocs.io/) · [Software Defined Radio with HackRF](https://greatscottgadgets.com/sdr/), a free video course · [rtl-sdr.com](https://www.rtl-sdr.com/)

**Wireless troubleshooting community** — [WLAN Pros](https://www.wlanpros.com/) and [Wireless LAN Professionals](https://wlanprofessionals.com/) · [CWNP](https://www.cwnp.com/) for the CWAP troubleshooting methodology · [r/wifi](https://www.reddit.com/r/wifi/)

**Tools** — [Wireshark](https://www.wireshark.org/) · [Kismet](https://www.kismetwireless.net/) · [aircrack-ng](https://www.aircrack-ng.org/) · [Zeek](https://www.zeek.org/)

**AI in network operations** — [Model Context Protocol](https://modelcontextprotocol.io/), the open standard the MCP server speaks · [Ollama tool calling](https://ollama.com/blog/tool-support) for the local-LLM path

---

*Built on a Raspberry Pi 4B running Kali rolling, September 2026. Hostnames, usernames, MAC addresses and IPs in this post are placeholders. The active-transmit tooling described here belongs only on networks the operator owns or is authorized to test — everything else is passive and safe anywhere.*
