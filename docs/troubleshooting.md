# Troubleshooting

Everything below assumes the Flatpak. Two shorthands, because they come up constantly:

```bash
FP="flatpak run --command=sh io.github.adds39939.SynapseLinux -c"   # a shell in the sandbox
PFX=~/.var/app/io.github.adds39939.SynapseLinux/data/wineprefix     # the Wine prefix
```

The prefix is a normal directory on your disk -- you can read its logs and settings from
outside the sandbox without any of this. It is only Wine itself that has to be run from
inside, because that is where Wine is.

**Logs** are in three places:

```bash
tail -f $PFX/razer-synapse.log                                                   # the launcher, and Wine
less "$PFX/drive_c/users/$USER/AppData/Local/Razer/RazerAppEngine/User Data/Logs/main.log"   # App Engine
less $PFX/drive_c/ProgramData/Razer/Installer2/Logs/lwi2_*.log                    # Razer's installer
```

`main.log` is the one that says what Synapse did with your devices: look for
`usb.getDevices`, `hid.getDevices` and `hid.openDevice`.

**Every report Synapse exchanges with a device** can be logged too. App Engine has a
switch for it; quit Synapse, then start it with

```bash
flatpak run io.github.adds39939.SynapseLinux --url-params=apps=synapse --logProtocol
```

and read `protocol.log` beside `main.log`. Each `set` is a 90-byte Razer report and the
`get` after it is the device's answer; its second byte is the status -- `2` done, `1`
busy, `3` failed, `4` timed out, `5` not supported. A few `5`s are normal: Synapse asks
about features a given model does not have. Lighting is streamed as `setBatch` frames,
which the log counts but does not show.

**Synapse starts but finds no device** -- work down this list.

1. *The udev rule.* Razer devices stay owned by root until a rule says otherwise, and
   Wine then cannot open them:

   ```bash
   ls -l /dev/hidraw*                    # the Razer ones should be crw-rw----+
   flatpak run --command=razer-synapse-setup io.github.adds39939.SynapseLinux --udev-help
   ```

   The launcher checks this itself and puts the commands in a dialog when it finds a
   Razer device it cannot open. A Flatpak cannot install a udev rule -- udev runs on the
   host, outside every container -- so this one step is on you. After installing it,
   restart Synapse so Wine enumerates the devices again.

   A device that worked without the rule can mean OpenRGB's or OpenRazer's rule was
   covering it. Theirs list models one by one, so a newer device -- a Naga V3 Pro, where
   a Naga V2 Pro worked -- can be left root-only while the old one never was.

2. *winebus's switch for Razer.* Without it Wine does not expose a mouse or keyboard as
   a HID device at all, and `main.log` shows the device under `usb.getDevices` and
   never under `hid.openDevice`. The setup sets it and the launcher re-asserts it every
   start; check it is there with

   ```bash
   grep -i -A2 -F 'winebus\\Devices\\1532' $PFX/system.reg  # wants "Hidraw"=dword:00000001
   ```

3. *The container ID.* If `main.log` shows a device page opening with
   `containerId=` and nothing after it, the Wine in use is not this project's -- patch
   23 is what gives Synapse that ID.

**A button reassigned in Synapse still sends its old key, or a button does nothing** --
Synapse runs Razer devices in *driver mode*, where the firmware sends every button's
default input and Synapse remaps it on the PC, through Razer's filter driver. Under Wine,
winebus stands in for that driver (patch 35 in [wine-fixes.md](wine-fixes.md)): while
Synapse runs, it takes the device's input away from the desktop, hands Synapse what it
asks for, and passes the rest on through a copy of each of the device's input devices.
For that it needs the device's event nodes and `/dev/uinput`, which the udev rule gives
the seat's user -- the rule from before this was added does not:

```bash
ls -l /dev/uinput /dev/input/by-id/*Razer*     # crw-rw----+ means the seat has an ACL
getfacl -p "$(readlink -f /dev/input/by-id/usb-Razer*-event-mouse | head -1)" | grep user:
```

If those are missing, install the rule again as in the README, then unplug the device
and plug it back in. With Synapse running there should be a copy of each of the device's
input devices, under `/sys/devices/virtual/input`, with the same names:

```bash
grep -l Razer /sys/devices/virtual/input/input*/name
```

and `WINEDEBUG=+razer` in the launcher's log shows each node grabbed, each hook Synapse
sets, and each input it takes or sends.

**A Razer device works, but some of its buttons never reach anything** -- even without
Synapse running. Check that a kernel driver has not left some of its interfaces without
one. OpenRazer's modules do that to some devices: with `razeraccessory` loaded, the Mouse
Dock Pro (`1532:00a4`) keeps its first interface, and the other two -- where a mouse paired
to the dock sends its side buttons and Razer keys -- get no driver at all:

```bash
journalctl -k | grep 'skipping secondary interface'
for d in /sys/bus/usb/devices/*:1.*/0003:1532:*; do echo "$d $(basename "$(readlink -f "$d/driver")")"; done
```

An interface with no driver after it has no hidraw or input node, for Synapse or the
desktop. Keep OpenRazer's kernel modules off those devices -- blacklist them, then unplug
the device:

```bash
printf 'blacklist razeraccessory\nblacklist razermouse\n' | sudo tee /etc/modprobe.d/synapse-linux-no-openrazer.conf
sudo modprobe -r razeraccessory razermouse
```

**A window, or the tray icon's flyout, is white, or keeps its old content when it is
resized** -- App Engine is drawing it from its GPU process, which Wine cannot show
reliably for a window of another process. The launcher starts App Engine with
`--disable-gpu-compositing` to avoid that ([wine-fixes.md](wine-fixes.md) has why); App
Engine started by Razer's installer, at the end of the first install, or by its updater,
runs without it. Quit Synapse from its tray menu and start it again. To see whether the
running App Engine has the switch:

```bash
pgrep -af -- '--url-params=apps=synapse'   # the main process; look for --disable-gpu-compositing
```

**"SERVER ACCESS UNAVAILABLE" in Razer's installer**, with the network fine -- the
prefix is set to a Windows version older than 10, and Razer offers Synapse 4 to
Windows 10 and later only. The installer's log says which it reported:

```bash
grep 'Windows Version' $PFX/drive_c/ProgramData/Razer/Installer2/Logs/lwi2_*.log
```

`6.1.7601` is Windows 7. Fix it, then start Synapse again:

```bash
$FP 'winetricks -q win10'
```

**"Installation Failed" inside Synapse, with "This PE file is not trusted" in
`main.log`** -- the Wine in use cannot verify MSIX signatures. Patch 22 is what adds
that; see [wine-fixes.md](wine-fixes.md).

**Nothing happens when I start Synapse** -- if it is already running this is expected:
the second launch asks the running one to bring its window to the front, out of the tray
if that is where it is, and exits. Closing Synapse's window sends it to the tray, as on
Windows; quit it from the tray menu. Otherwise read the launcher's log.

**The window is the wrong size** -- the launcher sets Wine's DPI from the `Xft.dpi` X
resource, which KDE and GNOME publish to their X clients. See what it found and what it
did with it:

```bash
flatpak run --command=razer-synapse-xdpi io.github.adds39939.SynapseLinux   # the DPI, or nothing
grep 'setting Wine' $PFX/razer-synapse.log                                  # what it applied
```

If it prints nothing, your desktop sets no `Xft.dpi` and the launcher falls back to the
settings portal, which reports only whole-number scales. Set it explicitly:

```bash
flatpak override --user --env=RAZER_SYNAPSE_SCALE=1.5 io.github.adds39939.SynapseLinux
```

**Two "Razer Synapse" entries in the KDE system tray** -- only one of them is Synapse.
The other is Plasma's *Background Apps* indicator, which lists every Flatpak running
without a visible window. Keep whichever you prefer: set the other to *Never show
(disabled)* in *Configure System Tray → Entries*, or stop Wine publishing its icon:

```bash
flatpak override --user --env=RAZER_SYNAPSE_TRAY=0 io.github.adds39939.SynapseLinux
```

With Wine's icon off, closing Synapse's window leaves it running with nothing to click;
starting it again brings the window back.

**OpenRazer is installed as well** -- both talk to the device over the same channel, so
while both are running, readings in one can come back garbled or as zero, and OpenRazer
can put a device in driver mode or out of it behind Synapse's back. Synapse re-reads the
device's state as it starts. If you want Synapse to own the device, stop the daemon while
it runs, and see above about its kernel modules:

```bash
systemctl --user stop openrazer-daemon
```

**Starting over**

```bash
flatpak uninstall --delete-data io.github.adds39939.SynapseLinux
```

That takes the prefix, Synapse and its settings with it. To keep the app and only
rebuild the prefix, delete `$PFX` and start Synapse again -- the first-run setup runs
from scratch. To re-run only Razer's installer over an existing prefix:

```bash
flatpak run --command=razer-synapse-setup io.github.adds39939.SynapseLinux --reinstall
```
