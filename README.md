<div align="center">

<img src="flatpak/io.github.adds39939.SynapseLinux.svg" alt="" width="104" height="104">

<h1>Razer Synapse on Linux</h1>

**Razer Synapse 4 via Wine** — the official app, installed by Razer's own installer,
configuring your Razer hardware directly.

[![Wine](https://img.shields.io/badge/wine-11.16%20(bundled)-44d62c?style=flat-square&labelColor=1c1c1e)](WINE_VERSION)
[![License](https://img.shields.io/badge/license-MIT%20%2B%20LGPL--2.1-6b7280?style=flat-square&labelColor=1c1c1e)](#licensing)

[**Install**](#install) · [What works](#what-works) · [Troubleshooting](docs/troubleshooting.md) · [How it works](docs/wine-fixes.md)

</div>

<br>

A Flatpak with a patched Wine and a launcher that make Razer Synapse work on Linux. Nothing
of Razer's is in here: the first run downloads Razer's installer from Razer, and Synapse
and its updates come from Razer as usual.

<br>

![Synapse running under Wine](docs/dashboard.png)

<sub>**The dashboard** — a Naga V2 Pro and its HyperSpeed receiver, found over HID with the
battery level read from the mouse.</sub>

| Customize | Performance |
| --- | --- |
| ![Customize](docs/customize.png) | ![Performance](docs/performance.png) |

<sub>Both pages show what is read from the mouse; changes on Performance are written back to it.</sub>

---

## Install

1. Download `synapse-linux.flatpak` from a release (or [build it](#building-it-yourself)), and
   install it:

   ```bash
   flatpak install --user synapse-linux.flatpak
   ```

2. Install the udev rule — the one step that needs root. Without it Synapse finds no
   devices and cannot remap their buttons:

   ```bash
   flatpak run --command=razer-synapse-setup io.github.adds39939.SynapseLinux --udev-rule \
     | sudo tee /etc/udev/rules.d/60-razer-synapse.rules >/dev/null
   sudo udevadm control --reload-rules
   sudo udevadm trigger --subsystem-match=hidraw --subsystem-match=usb \
     --subsystem-match=input --subsystem-match=misc
   ```

3. Start **Razer Synapse** from your application menu. The first run sets up the Wine
   prefix, which takes a few minutes, then opens Razer's installer: choose **Install**.

Everything the app writes lives in `~/.var/app/io.github.adds39939.SynapseLinux/`. To
remove it all: `flatpak uninstall --delete-data io.github.adds39939.SynapseLinux`.

## What works

Verified with a **Naga V2 Pro** on its HyperSpeed receiver (`1532:00a8`), and a **Naga V3
Pro** paired to a **Mouse Dock Pro** (`1532:00a4`):

* ✅ **Razer's installer** — the current one, from Razer, installs Synapse 4 and the App
  Engine it runs on
* ✅ **Synapse's own module install** — the drivers, lighting and mapping engines it
  fetches after first start
* ✅ **The mouse and its receiver are detected**, with battery level and charging state
  that update live
* ✅ **Device settings are read from the mouse** — sensitivity stages, polling rate,
  which side plate is fitted, lighting brightness and effect
* ✅ **Changes are written back to the mouse**, and the mouse acknowledges each one
* ✅ **Lighting** — effects and brightness chosen in Synapse show on the mouse
* ✅ **Button assignments, as on Windows** — Synapse runs the mouse in driver mode and
  remaps it on the PC: keys, the AI launcher, Synapse's wheel handling, all at the mouse's
  8000 Hz ([how](docs/wine-fixes.md#35-razers-filter-driver-done-by-winebus))
* ✅ **Pairing a mouse to the Mouse Dock Pro** from Synapse
* ✅ **The window** — maximized, from Synapse's own button or by dragging it to the top
  of the screen, it fills the screen with only Synapse's own title bar
* ✅ **The tray icon** — its menu opens on every right-click, and its flyout opens Synapse
* ✅ **Display scaling** matches your desktop's, so it is not tiny on a 4K panel

Not tried yet: macros, Hypershift, firmware updates, and other Razer devices — nothing
here is specific to the Nagas. Razer Game Manager Service (Linked Games, Cortex) does not
run under Wine.

## Notes

* Synapse's *start with Windows* setting starts it in the tray on login.
* If you use OpenRazer, keep its kernel modules off the devices Synapse runs, and stop its
  daemon while Synapse runs — see [troubleshooting](docs/troubleshooting.md).
* If the window is the wrong size for your display, set the scale yourself:
  `flatpak override --user --env=RAZER_SYNAPSE_SCALE=1.5 io.github.adds39939.SynapseLinux`

## Building it yourself

```bash
flatpak install -y flathub org.freedesktop.Platform//25.08 \
    org.freedesktop.Sdk//25.08 org.freedesktop.Sdk.Extension.mingw-w64//25.08
flatpak-builder --user --install --force-clean build-dir flatpak/io.github.adds39939.SynapseLinux.yml
```

This builds Wine 11.16 with the patches in [`patches/`](patches) — what each one fixes is
in [`docs/wine-fixes.md`](docs/wine-fixes.md).

## Licensing

This repository is MIT (see `LICENSE`). The Wine in the Flatpak is built from the patched
Wine sources here and is LGPL-2.1-or-later (see `NOTICE`). No Razer code or artwork is
included; "Razer" and "Synapse" are Razer's trademarks, and this project is not affiliated
with or endorsed by Razer.
