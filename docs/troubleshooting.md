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

**A button reassigned in Synapse still sends its old key** -- the mouse is in Razer's
*driver mode*, in which it ignores the assignments stored in it and expects Razer's
filter driver to translate its keys on the PC. Wine has no such driver, so this build
keeps Razer devices in normal mode (patch 25 in [wine-fixes.md](wine-fixes.md)). If the
key is still wrong, check that the switch that turns this off is not set:

```bash
grep -i -A3 '^\[System\\\\ControlSet001\\\\Services\\\\winebus\]' $PFX/system.reg  # no "RazerDriverMode"=dword:00000001
```

and, with Synapse quit, what mode the mouse is in, if OpenRazer is installed:

```bash
xxd -p /sys/bus/hid/devices/0003:1532:*/device_mode   # 0000 is normal, 0300 is driver mode
```

Starting Synapse puts a mouse left in driver mode back in normal mode. Assignments only
Synapse itself can perform -- launching programs, text, Windows shortcuts -- do not work
under Wine at all.

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
while both are running, readings in one can come back garbled or as zero. Synapse
re-reads the device's state as it starts. If you want Synapse to own the device, stop
the daemon while it runs:

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
