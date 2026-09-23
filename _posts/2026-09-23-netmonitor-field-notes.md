---
layout: post
title: "NetMonitor Field Notes"
categories: [tech]
description: >
  Building a native SwiftUI app that watches interfaces, sockets and Wi-Fi RF in
  real time — and the five macOS traps that each cost a rebuild cycle, including
  the one where asking for Location permission is what breaks Location.
---

Tool to monitor wi-fi on your macbook.
{:.lead}

**NetMonitor** is a native SwiftUI app that watches network interfaces, open sockets and Wi-Fi RF in real time. These are the build notes — and more usefully, the set of macOS traps that made it take four rebuilds longer than it should have.

Built against macOS 26.6 on an M1 Pro, Swift 6.3, with **Command Line Tools only — no Xcode installed and none needed**. Ad-hoc signed.

* this list will be replaced by the table of contents
{:toc}

## Build and run

Swift Package Manager the whole way. `build_app.sh` compiles a release build, assembles the `.app` bundle by hand, and ad-hoc signs it.

```bash
# build, install, launch
./build_app.sh
cp -R NetMonitor.app /Applications/
open /Applications/NetMonitor.app

# iterate with visible stderr (see the stderr trap below)
swift build && .build/debug/NetMonitor

# what the app itself reports
tail -f /tmp/netmonitor.log
```

> **Keep disk headroom.** An `ENOSPC` part-way through `swift build` leaves a poisoned `.build` cache: later builds cheerfully report "Build complete" from stale objects, and produce an app that launches, draws its window, and shows nothing at all. `rm -rf .build` is the cure — and it's the first thing to suspect whenever behaviour and source disagree.

## Where every number comes from

Two polling cadences on a utility queue, publishing to `@Published` properties on main.

One design decision worth stating plainly: **shelling out beats the C APIs here.** `getifaddrs` exposes only 32-bit byte counters, which wrap every 4 GB — on a busy Wi-Fi link that's often enough to make the rate display nonsense.

| Shown in app | Source | Notes |
|---|---|---|
| Name, status, MTU, IPv4/IPv6, MAC | `ifconfig -a` | `status: active` is the real link-up test; `<UP…>` in the flags only means admin-up |
| RX/TX bytes, packets, errors | `netstat -ib` | 64-bit counters. Read only the `<Link#N>` rows — the per-address rows double-count |
| Live ↓/↑ rates | derived | Δbytes ÷ Δt between 1 Hz samples, guarded against counter resets |
| Sockets, process, PID, state | `lsof -i -n -P` | Every 2 s. `-n -P` skips DNS and service-name lookups, which keeps it around 45 ms |
| RSSI, noise, TX rate, channel, band, width | CoreWLAN `CWInterface` | Not Location-gated — these keep working even when SSID comes back `nil` |
| SSID, BSSID, security, country, PHY | CoreWLAN `CWInterface` | **Location-gated.** Read the Location trap below before touching anything here |
| Nearby BSSIDs | `scanForNetworks(withSSID: nil)` | Blocks for 2–4 seconds, so it runs on its own queue and never on main |

## The k / v / r badges

The app shows a small badge per AP for 802.11k, v and r support. Those are parsed byte-by-byte out of each AP's `CWNetwork.informationElementData` — the raw beacon information elements. This is capability the AP *advertises*, which is the honest limit of what macOS will tell a third-party app.

| Badge | Element | Test applied |
|---|---|---|
| **k** | 70 (0x46) — RM Enabled Capabilities | Element present at all |
| **v** | 127 (0x7F) — Extended Capabilities | Bit 19 set (octet 2, bit 3) = BSS Transition Management |
| **r** | 48 (0x30) — RSN | AKM suite `00-0F-AC:3` (FT-802.1X) or `:4` (FT-PSK) |

> **There is no 802.11k neighbour report here, and there can't be.** A real neighbour report means sending a Radio Measurement request frame to the associated AP and reading its response, and macOS exposes no public API for that at any entitlement level. So the badges answer *"which APs support assisted roaming?"* — not *"what does this AP think its neighbours are?"* For the latter, a monitor-mode capture is the route, not CoreWLAN.

That distinction is worth being precise about, because it's the kind of thing a dashboard can very easily imply it knows and doesn't.

## Five traps

Each of these cost a rebuild cycle, and they share a failure mode worth naming: **the app keeps running and looks entirely plausible while quietly showing nothing true.** No crash, no error, no red text. That's the expensive kind of bug.

### 1 · Asking for Location is what breaks Location

*macOS / TCC*

**Symptom** — SSID and BSSID return `nil` and the header reads "Wi-Fi off," while RSSI and channel keep reporting correctly.

**Cause** — macOS gates SSID and BSSID behind Location access. An ad-hoc-signed app can never be *granted* it. But merely constructing a `CLLocationManager` earns an explicit **denial**, and CoreWLAN honours that denial by redacting from then on. An app that never asks is never denied.

**Fix** — zero CoreLocation in the target. No import, no manager, no usage-description prompt. There's a comment in `WiFiMonitor.swift` whose entire job is to stop a future helpful refactor from re-adding it.

```
[01:16:00.435] ssid=MyNetwork  bssid=aa:bb:cc:dd:ee:ff  rssi=-32  ch=157
[01:16:01.773] created CLLocationManager …
[01:16:02.461] ssid=nil        bssid=nil                rssi=-32  ch=157
[01:16:03.888] location error: kCLErrorDomain error 1 (denied)
```

This is the one that took longest to believe, because the intuition runs exactly backwards: the permission request is the thing that causes the permission failure.

### 2 · Nested ObservableObjects never redraw

*SwiftUI*

**Symptom** — the window renders correctly once, then every value stays frozen at its launch default forever, while the logs prove fresh data is arriving every second.

**Cause** — the views observe `AppModel`, but the three monitors are *child* `ObservableObject`s held as plain properties. A child's `@Published` change never reaches the parent's `objectWillChange`, so SwiftUI is simply never told to re-render.

**Fix** — in `AppModel.init`, sink each child's `objectWillChange` into the parent's. Cheap at 1 Hz.

### 3 · List selection binds the id type, not the element

*SwiftUI*

**Symptom** — the sidebar renders, highlights on hover, and flatly refuses to select. Compiles without a single complaint.

**Cause** — `List(data, selection:)` binds to the element's **id** type, which here was `String`, but the binding was `Section?`. Those types can never match, so no row is selectable.

**Fix** — `List(selection:) { ForEach(…) { … .tag(s) } }`, with the tag type matching the binding's wrapped type.

### 4 · Table tops out at ten columns

*SwiftUI*

**Symptom** — `error: extra arguments at positions #11, #12 in call`, with the compiler pointing at the *first* column.

**Cause** — `TableColumnBuilder.buildBlock` is only generated up to `C0…C9`. The error message blames the wrong line entirely.

**Fix** — merge related columns. RX and TX rate became one ↓/↑ cell; IPv4 and IPv6 are stacked in a single Address cell. It reads better than twelve narrow columns anyway, so the constraint improved the design.

### 5 · Apps launched by Finder have no stderr

*Tooling*

**Symptom** — every `print` vanishes, and `log show` returns nothing for the process either.

**Cause** — launching via `open` discards stdout and stderr. Running the binary from Terminal restores them — but it also silently lends the app Terminal's TCC permissions, so the bug being chased disappears at exactly the moment it becomes visible.

**Fix** — a small `Diag.swift` that appends to `/tmp/netmonitor.log`. That survives any launch method and tells the truth about the real app identity.

That last one is the trap behind the trap: the obvious debugging move changes the thing being debugged.

## Limits and next moves

* **The ad-hoc signature shifts identity on every rebuild**, so macOS re-evaluates the app each time. Harmless here only because nothing depends on a retained TCC grant.
* **Channel width mapping stops at 160 MHz** — no 320 MHz or EHT yet. Adding `.mode11be` handling needs a Wi-Fi 7 AP in range to test against.
* **The roam log is in-memory** and clears on quit. Worth persisting, plus a CSV export, if it's ever going to back up a real roaming investigation.
* **No per-process bandwidth** — `lsof` gives sockets, not throughput. `nettop -P -L 0` would add per-process rates.
* **Scanning is manual after launch** — one auto-scan at startup, then on demand. A slow repeating scan would suit stationary monitoring, at the cost of hitching the association every time the radio goes off-channel.

## The takeaway

The interesting thing about this build wasn't the SwiftUI or the CoreWLAN parsing — it was that **four of the five traps produced a working-looking app rather than an error.** A frozen UI that renders once, a redacted field that returns `nil`, a list that won't select, a print that goes nowhere: in every case the tooling said everything was fine.

Which is an argument for the fifth file in the project being a logger. `Diag.swift` is thirty lines and it's the reason the Location trap was findable at all — the timestamped before-and-after in that log is the entire diagnosis, and no amount of staring at the source would have produced it.

---

*Built on macOS 26.6, Swift 6.3, Command Line Tools only. SSIDs and BSSIDs in the log excerpts above are placeholders.*
