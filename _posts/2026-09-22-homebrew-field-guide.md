---
layout: post
title: "Homebrew Field Guide"
categories: [tech]
description: >
  A homebrew cheat sheet in one page — installing, updating and cleaning up
  terminal software on a Mac, plus the Brewfile trick that rebuilds a whole
  machine in one command.
---

Wanted to put together a cheat sheet for homebrew I could access anywhere on the net. Easy system to install and use to add some useful linux tools to OS X.
{:.lead}

## What it is

Homebrew is a **package manager**: a catalog of thousands of free tools (git, gh, python, ffmpeg…) that it downloads, installs and updates for you. Everything lands tidily under `/opt/homebrew`, so nothing pollutes the rest of the system, and anything can be removed with one command.

Two words to know: a **formula** is a command-line tool (like `gh`); a **cask** is a regular Mac app with a GUI (like Firefox or VLC), which brew can install too.

If you don't have it yet, the install command lives on the front page of [brew.sh](https://brew.sh/).

## Finding & installing

```bash
brew search wget
```

**Search the catalog.** Not sure of the exact name? This finds close matches.

```bash
brew info wget
```

**Read the label before you buy.** Shows what a package is, its homepage, and its size.

```bash
brew install wget
```

**Install a tool.** Seconds later it's a command you can type. For GUI apps, add the cask flag: `brew install --cask vlc`.

```bash
brew uninstall wget
```

**Remove it completely.** No dragging to the Trash, no leftover junk.

```bash
brew list
```

**What do I have?** Lists everything brew has installed on this Mac.

## Staying updated

> **The two-word gotcha:** `update` refreshes brew's *catalog* (what versions exist in the world); `upgrade` actually upgrades *your installed software*. Brew usually runs `update` on its own — that's the "Auto-updating Homebrew…" message you see. You only really need `upgrade`.

```bash
brew outdated
```

**What's stale?** Lists installed packages with newer versions available. This is where that cheerful "you have 54 outdated formulae" message comes from — mine had exactly that, from months of ignoring it.

```bash
brew upgrade
```

**Upgrade everything** on that list in one go. To upgrade just one thing: `brew upgrade gh`.

```bash
brew cleanup
```

**Take out the trash.** Deletes old versions and cached downloads — often frees a surprising amount of disk.

A once-a-month routine of `brew upgrade` followed by `brew cleanup` keeps everything current.

## The Brewfile trick

This is the feature nobody mentions to beginners, and it's the best one. Brew can write down everything you have installed:

```bash
brew bundle dump --describe --file=Brewfile
```

That produces a plain text `Brewfile` listing every formula and cask on the machine, with a comment describing each. Then on a new Mac — or after a wipe — you drop that file in place and run:

```bash
brew bundle install
```

One command, and the new machine has your whole toolkit back. It's also a nice thing to keep in version control: a `Brewfile` committed to a git repo is a readable record of your own tool choices over time.

## When something acts up

```bash
brew doctor
```

**Brew's self-diagnosis.** Run this first if anything misbehaves — it names problems and usually prints the exact fix. Warnings are common and mostly harmless; read them, don't panic over them.

> If a brew command fails part-way through, it's almost always safe to just run it again. Brew picks up where it left off.

## There's a GUI now

Homebrew shipped an official Mac app, which is new enough that most people haven't noticed it. It needs macOS 26:

```bash
brew install --cask homebrew-app
```

That puts a Homebrew.app in Applications for browsing, installing and updating packages with a mouse. It's the same packages either way, so nothing breaks by trying it — but being this new, expect rough edges. The terminal commands above remain the reliable path.

## What about the alternatives?

**MacPorts** is the other general-purpose Mac package manager, and it's actually older than Homebrew. It builds more things from source, installs under `/opt/local`, and depends on almost nothing from macOS itself — so it's more self-contained. Smaller catalog, slower installs, smaller community. A respectable tool, but every tutorial you'll find online assumes brew, so going with MacPorts means translating instructions all day. Pick one and stay there; running both is a known way to create confusing conflicts.

**Nix** is the power tool: fully reproducible environments, multiple versions of things side by side, atomic rollbacks. It's genuinely impressive and genuinely steep — it has its own programming language. Worth knowing the name; not worth the detour unless you need reproducible builds.

## Quick reference

| Command | What it does |
|---|---|
| `brew search NAME` | Find a package in the catalog |
| `brew info NAME` | Describe a package before installing |
| `brew install NAME` | Install a command-line tool |
| `brew install --cask NAME` | Install a regular Mac app |
| `brew uninstall NAME` | Remove a package cleanly |
| `brew list` | Show everything installed |
| `brew outdated` | Show what has updates available |
| `brew upgrade` | Upgrade all installed packages |
| `brew cleanup` | Delete old versions, free disk space |
| `brew doctor` | Diagnose problems |
| `brew bundle dump` | Write a Brewfile of everything installed |
| `brew bundle install` | Reinstall everything from a Brewfile |
