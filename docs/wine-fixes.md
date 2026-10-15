# The Wine changes

What each change fixes, and why. None of this is needed to use the repo -- it is the
record of how Synapse was made to work.

## How Synapse 4 is put together

Knowing this up front explains most of what follows.

* **Razer's installer** (`RazerSynapseInstaller.exe`) is a DotNetZip self-extractor
  around `RazerInstaller.exe`, a .NET Framework 4.8 WPF application. It asks Razer's
  manifest server what to install, then downloads and silently runs two packages.
* **Razer App Engine** is an Electron application. The Synapse window is a web app it
  loads from `https://apps.razer.com/synapse/`, and each device has its own pages under
  `apps.razer.com/synapse/products/<id>/`.
* **Talking to devices** is plain user-mode HID. App Engine's `node-rz-hid` module is
  hidapi with Razer's additions, and Synapse configures a device with 91-byte feature
  reports -- the same 90-byte report protocol OpenRazer uses, plus the report id. No
  Razer kernel driver is in that path, so Wine's HID stack can carry it all the way to
  `/dev/hidraw*`.
* **Synapse's own in-app installer** then fetches a set of common modules -- the
  mapping and lighting engines, Razer's Game Manager service, per-device driver
  packages -- verifying every file's Authenticode signature before it writes it.

## Patches 01-20: the NZXT CAM series

Wine here is 11.16, the same base as [NZXT CAM on Linux][cam], and its twenty patches
are carried unchanged. They are generic fixes -- device enumeration, HID interface
properties, WinUSB, cfgmgr32 device nodes, named tray icon windows -- and their
write-up is in that project's `docs/wine-fixes.md`.

The one Synapse visibly leans on is 07: `wineusb.sys` registering
`GUID_DEVINTERFACE_USB_DEVICE`. App Engine's `rz-usb-detect` enumerates exactly that
interface to learn which Razer devices are plugged in, and without it Synapse sees no
USB devices at all. Patch 23 builds on the same driver.

[cam]: https://github.com/adds39939/nzxt_cam_linux

## Prefix settings

### Windows 10, not 7 -- "SERVER ACCESS UNAVAILABLE"

**Symptom:** Razer's installer opens to *SERVER ACCESS UNAVAILABLE -- Service
disconnected. Please check your internet connection*, with the network fine.

**Cause:** `winetricks dotnet48` installs .NET with the prefix set to Windows 7 and
leaves it there. The installer reports the Windows version to Razer's manifest server,
and Razer serves Synapse 4 to Windows 10 and later only:

```
products?os=WINDOWS&osver=7&arch=64...   ->  []
products?os=WINDOWS&osver=10&arch=64...  ->  ['Anne', 'Chroma', 'GameBooster2', ...]
```

An empty product list is indistinguishable, to the installer, from a server it could
not reach. (`Anne` is Synapse 4's internal name.)

**Fix:** the setup runs `winetricks win10` after .NET is in.

### winebus does not expose mice or keyboards

**Symptom:** with every other fix in, Synapse lists the receiver over USB but never
opens a device, and the HID device list holds nothing from Razer at all.

**Cause:** winebus deliberately ignores any hidraw device whose first top-level
collection is a generic-desktop mouse or keyboard, leaving those to the windowing
system:

```c
if (usages->Usage == HID_USAGE_GENERIC_MOUSE || usages->Usage == HID_USAGE_GENERIC_KEYBOARD)
{
    WARN("Ignoring unsupported %04X:%04X hidraw mouse/keyboard\n", vid, pid);
    return FALSE;
}
```

Every interface of a Razer mouse is one of those -- the Naga V2 Pro's receiver has a
boot mouse and two boot keyboards, and Razer's configuration reports go to the mouse
one.

**Fix:** none needed in Wine. winebus already has a per-vendor override, read from
`HKLM\System\CurrentControlSet\Services\WineBus\Devices\<VID>`, and `Hidraw = 1` under
`1532` turns hidraw on for Razer's devices. The setup writes it and the launcher
re-asserts it.

This does not give Wine a second pointer. Nothing in Wine turns HID mouse reports into
pointer movement -- `mouhid.sys` binds touchscreens only -- so the devices become
something a program can open, and nothing more. The desktop keeps its own copy of
every report, since hidraw hands one to each reader.

### Display scaling

Synapse renders at 100%, so on a 4K panel at 145% it is small. The launcher reads the
`Xft.dpi` the desktop publishes to every XWayland client -- 139 for 145% -- and writes
it to Wine's `HKCU\Control Panel\Desktop\LogPixels`. Chromium takes Wine's DPI as its
device pixel ratio and renders at that scale, so the result is sharp rather than a
100% window stretched by the compositor. The setup applies it before Razer's installer
opens, so the installer comes up at the right size too; WPF scales by system DPI.

## The patches

### 21. A process still being created must not be listed

**Symptom:** App Engine starts, logs

```
whenReady Error: ...mjpeg_hotpatch.cpp(148)\mjpeg-hotpatch.node!...: Exception(1) 80070005 Access denied.
```

and never opens a window.

**Cause:** `forceMjpegPriority()` (from App Engine's `mjpeg-hotpatch` module, which
patches webcam negotiation in Chromium's video-capture process) walks a toolhelp
snapshot for its own children and reads each one's PEB:

```
OpenProcess(0x438, ...)                        -> ok
NtQueryInformationProcess(ProcessBasicInfo)    -> InheritedFromUniqueProcessId == self
ReadProcessMemory(pbi.PebBaseAddress, 0x2c8)   -> ERROR_ACCESS_DENIED  -> throws
```

and any failure throws, which aborts App Engine's `whenReady` handler. The child it
tripped on was one Electron was creating on another thread at that moment. Wine's
server puts a process on the list from the moment the parent asks for it, before the
child has run at all:

```
get_process_info() = 0 { pid=01f0, ppid=0020, peb=00000000, start_time=5555555555555555, exit_code=259 }
read_process_memory( handle=0500, addr=00000000 ) = ACCESS_DENIED
```

Windows never shows a process in that state: by the time it is on the list, its PEB
exists.

**Fix:** `list_processes` leaves out processes whose startup has not completed. They
appear a moment later, once they have initialised.

### 22. wintrust cannot verify AppX/MSIX packages

**Symptom:** Synapse comes up and immediately reports *Installation Failed*, with

```
simpleAddUserAppFile ... result:false, reason:Error: This PE file is not trusted
```

**Cause:** Synapse's in-app installer passes every file it downloads through
`WinVerifyTrust` before writing it. All the PE files verify fine. The one that does not
is `razerwdl.msix` -- Razer's Windows Dynamic Lighting package -- because Wine has no
subject interface package for AppX/MSIX, which Windows provides in `AppxSip.dll`:

```
WinVerifyTrust(razerwdl.msix) = 0x800b0003   TRUST_E_SUBJECT_FORM_UNKNOWN
```

One failed file fails the whole module install.

**Fix:** `patches/22-…` gives wintrust an AppX SIP, registered under the same GUID as
Windows' (`{0AC5DF4B-CE07-4DE2-B76E-23C839A09FD1}`):

* an `IsMyFileType` function that recognises a ZIP archive containing
  `AppxSignature.p7x` -- by contents, since every ZIP starts with the same magic;
* `CryptSIPGetSignedDataMsg` returning the PKCS#7 inside `AppxSignature.p7x`, after its
  `PKCX` header;
* a real digest check. A package's signed digest is not a hash of the file but a
  concatenation of tagged hashes, which softpub now recomputes and compares:

  | tag | hash of |
  |---|---|
  | `AXPC` | every byte before `AppxSignature.p7x`'s local header |
  | `AXCD` | the central directory without the signature's record, plus the ZIP64 end records rewritten as if it had never been added |
  | `AXCT` | `[Content_Types].xml`, uncompressed |
  | `AXBM` | `AppxBlockMap.xml`, uncompressed |
  | `AXCI` | `AppxMetadata/CodeIntegrity.cat`, uncompressed, when present |

  Each was confirmed against Razer's signed package before any C was written. The chain
  and signer checks are softpub's existing ones. Tampering is caught: flipping a byte of
  a member's data fails `AXPC`, and one in the directory fails `AXCD`, both with
  `TRUST_E_BAD_DIGEST`.

Before this, softpub skipped the hash check for anything that was not a PE file, with a
FIXME. AppX packages now get the full check.

### 23. One container per physical USB device

**Symptom:** the dashboard briefly shows a tab for the device, then *NO DEVICE FOUND*.
The log has the device page loading with nothing to identify it by:

```
products/167/mw/index.html?containerId=
hid.removeDevice ==> 168-, usbInstanceId: undefined ... device not found
```

**Cause:** Synapse pairs what `rz-usb-detect` sees on the USB side with the HID devices
`node-rz-hid` can open by `DEVPKEY_Device_ContainerId`. On Windows every node of one
physical device -- the USB device, its interfaces and the HID collections on them --
shares one container ID. In Wine, `wineusb.sys` reported none, and `winebus.sys` made a
unique one per HID interface, seeded with the time.

**Fix:** both drivers derive the same GUID from where the device is plugged in -- its
vendor and product id, bus number, and the chain of hub ports -- which libusb gives
`wineusb` and the parent `usb_device`'s `busnum` and `devpath` give `winebus`:

```
USB\VID_1532&PID_00A8\512&256&8&3                     {153200A8-0008-5501-0300-000000000000}
USB\VID_1532&PID_00A8&MI_00\512&256&8&3               {153200A8-0008-5501-0300-000000000000}
HID\VID_1532&PID_00A8&MI_00\256&00000000000000A81532… {153200A8-0008-5501-0300-000000000000}
HID\VID_1532&PID_00A8&MI_01&COL01\…                   {153200A8-0008-5501-0300-000000000000}
```

It is the same on every run, and different for the same model in another port.
Devices winebus gets from somewhere other than udev keep the unique IDs they had.

With that, Synapse opens the device:

```
hid.openDevice \\?\hid#vid_1532&pid_00a8&mi_00#…#{4d1e55b2-…} successfully
```

and reads battery level, sensitivity stages, polling rate and which side plate is
fitted from the mouse. Writes reach it too: a polling rate set in Synapse is what
OpenRazer reads back from the mouse once Synapse is closed.

### 24. A window emptied by its application must stay empty

**Symptom:** while Synapse runs, clicks do nothing in a 440×400 area at the top left of
the primary monitor. Nothing is drawn there -- it is an invisible window that stays on
top of everything, on the desktop as much as in Wine.

**Cause:** App Engine's `SysUtilsNative.dll` watches for the foreground window changing,
and makes a window to do it from. It creates the window, shows it, and then shrinks it to
nothing:

```
CreateWindowExW(WS_EX_LAYERED|WS_EX_TOOLWINDOW|WS_EX_TOPMOST, L"RzMonitorForegroundWindowClass",
                L"RzMonitorForegroundWindow", WS_POPUP, CW_USEDEFAULT, CW_USEDEFAULT, 440, 400, ...)
ShowWindow(hwnd, SW_SHOWNORMAL)
SetLayeredWindowAttributes(hwnd, 0, 1, LWA_ALPHA)
SetWindowPos(hwnd, NULL, 0, 0, 0, 0, SWP_NOZORDER)          <- 0x0
```

On Windows that leaves a window with no area. winex11 means to do the same: an X
window cannot be 0x0, so it holds an empty Win32 window as a 1x1 X window with an empty
bounding shape -- invisible, and nothing to click. But it also copies the size the window
manager reports back into the Win32 rect, and for an empty window that size is never the
real one:

* the 1x1 stand-in comes back, and the window becomes 1x1; or
* the resize is deferred while the window's map is still being negotiated, a configure
  from the initial 440x400 map arrives after the app's `SetWindowPos`, and the window is
  put back to 440x400 -- with its full-size shape, and so its full-size input area.

Which one happens is a race; the second is the dead zone.

**Fix:** `window_update_client_config` no longer takes a size from X for a window whose
Win32 rect is empty. The window keeps the 0x0 it was given, its X window keeps the empty
shape, and X delivers input only inside a window's bounding shape:

```
rect=(0,0)-(0,0)    bounding=[]    input=[(0, 0, 1, 1)]     <- three restarts, the same each time
```

### 25. Razer devices stay in normal mode

**Symptom:** a button reassigned in Synapse -- side button 1 to `A` -- still types `1`,
everywhere, while Synapse runs. With Synapse closed, it types `A`.

**Cause:** Razer devices have a *driver mode* (`0x03`), set with command class `0x00`
id `0x04` and read back with `0x84`. In it a mouse stops applying the button
assignments stored in it and reports raw keys, for Razer's filter driver to translate
on the PC (see below). Synapse's lighting engine switches every RGB device into driver
mode as it is set up, unless Razer's online device manifest marks it
`skipSetDriverMode`, and again whenever the device is re-initialised -- a lighting or
profile change is enough:

```
if (E.isChromaSupported) { if (E.isPowerOn) { if (!E.skipSetDriverMode) { ...
    await setDeviceMode({ arg1: 3, arg2: 0 })
```

Wine has no filter driver, so the raw `1` goes straight to the desktop. Synapse does
store every assignment in the mouse's on-board profile as well
(`taskRunnerSyncAllMappingToObm`), which the firmware applies in normal mode: with the
mouse put back in normal mode under a running Synapse, the button types `A` and the
lighting Synapse streams to it carries on unchanged.

**Fix:** `patches/25-…` makes winebus keep Razer devices (vendor `1532`) in normal mode.
Razer's configuration protocol is 90-byte feature reports:

```
[0] status  [1] transaction id  [2-3] remaining  [4] protocol  [5] data size
[6] command class  [7] command id  [8-87] arguments  [88] crc (XOR of 2-87)  [89] reserved
```

* a *set device mode* request for driver mode goes to the device as normal mode;
* the device's answers to *set* and *get device mode* are reported back as driver mode,
  with the crc recomputed, so Synapse believes what it asked for and does not keep
  asking;
* a device found already in driver mode -- left that way by a session that did not get
  to restore it, here or under Windows -- is put back in normal mode the first time an
  application reads its mode.

```
set [..,0,4,3,..]   ->  winebus: keeping "/dev/hidraw10" in normal mode instead of driver mode
get [..,0,4,3,..]       (the device said 0; Synapse is told 3)
```

Synapse's own restore when it quits (`configureDevicesDefaultData`, normal mode) passes
through untouched. The `RazerDriverMode` bus option
(`HKLM\System\CurrentControlSet\Services\WineBus`, `REG_DWORD` 1) turns all of this off.

## Things that are not fixed, and why

### Razer's filter driver, and driver mode

Synapse's device modules install Razer's per-device kernel drivers -- for the Naga V2
Pro, `RzDev_00a7.sys`, attached by its INFs as an upper filter on the HID mouse and a
lower filter on the USB interface. Wine's PnP manager never loads upper or lower
filters, so the driver can neither attach nor do any harm. Razer's mapping engine is
configured with `connectFilterDriver: true`, logs `fail to probe filter driver`, and
keeps retrying for as long as Synapse runs.

On Windows the filter is half of PC-side remapping: with the device in driver mode it
hands the mapping engine the raw button presses and swallows them, and the mapping
engine acts on them -- with `SendInput`, low-level hooks and raw input. Patch 25 keeps
devices out of driver mode instead, so their firmware applies the assignments Synapse
stores in them. What that leaves out is whatever only the PC side can do -- PC-side
macros, launching programs, text, Windows shortcuts. Making those work would take three
things: grabbing the device's evdev nodes so the desktop stops seeing the raw presses,
emulating the filter driver's interface to the mapping engine (a private protocol,
still to be reverse-engineered), and carrying `SendInput` out of Wine to a uinput
device, since inside Wine it reaches only Windows programs. The `RazerDriverMode` bus
option is the switch that would hand driver mode back to Synapse for that.

### Razer Game Manager Service

Installed by Synapse's modules for Linked Games and Cortex, not for devices. It watches
for games starting through Event Tracing for Windows -- a real-time kernel trace of
process events, through Microsoft's TraceEvent library -- and Wine implements no kernel
trace sessions:

```
[ERR] Call not implemented. (0x80070078)
[ERR] Failed to start process monitor: Access denied. (0x80070005 (E_ACCESSDENIED))
[FTL] UnexpectedExceptionHandler
System.NullReferenceException ... at Microsoft.Diagnostics.Tracing.ETWTraceEventSource.Dispose
```

The service exits and its crash reporter starts -- four times in the first minute of a
fresh install, going by its crash logs. Nothing of Synapse's device support depends on it.

## How the launcher finds Synapse

The `RazerAppEngine.exe` at the top of the install is a stub: it starts the real
Electron process from the newest `app-<version>` directory beside it and exits, which
is how App Engine updates itself. The real one cannot be found by name either -- Wine
rewrites a process's command line to the Windows one only when it fits in the space the
original took, and App Engine's does not, so `ps` shows ` --url-params=apps=synapse`
and nothing else. The launcher identifies it by the image it has mapped:
`app-<version>\RazerAppEngine.exe`, without the `--type=` switch every Chromium helper
process carries.
