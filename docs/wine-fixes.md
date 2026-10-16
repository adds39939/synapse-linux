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

## Patches 01-20: device enumeration, HID and USB

Wine here is 11.16. The first twenty patches fill gaps in the Windows device APIs that
Electron applications use to find and talk to their hardware:

| Patch | What it does |
| --- | --- |
| 01 | `propsys` knows the HID property names (`System.DeviceInterface.Hid.*`) |
| 02 | `DeviceWatcher` raises its `Updated` and `Removed` events |
| 03 | `hidclass.sys` publishes each HID interface's usage page, usage and IDs as properties |
| 04 | DevQuery applies the filters it is given |
| 05 | DevQuery enumerates interface properties from both registry levels |
| 06 | `IDeviceInformation2` is implemented |
| 07 | `wineusb.sys` registers `GUID_DEVINTERFACE_USB_DEVICE` for every device |
| 08 | `winusb.dll` is implemented, on top of `wineusb.sys` |
| 09 | `SPDRP_PHYSICAL_DEVICE_OBJECT_NAME` is answered |
| 10 | Overlapped WinUSB transfers complete instead of failing with `ERROR_IO_PENDING` |
| 11 | `IReference<UINT16>` exists |
| 12 | `System.Devices.Children` is a known property |
| 13 | cfgmgr32 can query device nodes, not only interfaces |
| 14 | `DeviceInformation.CreateFromIdAsync` is implemented |
| 15 | Unnamed property keys are formatted in the case Windows uses |
| 16 | `FindAllAsync` with a device information kind is implemented |
| 17 | DevQuery string filters compare strings, not buffer sizes |
| 18 | `wineusb.sys` reads MS OS descriptors and registers the interface GUIDs they name |
| 19 | `Windows.Devices.Usb` is implemented |
| 20 | Wine's tray icon windows are named after their application, so desktops can remember them |

The one Synapse visibly leans on is 07. App Engine's `rz-usb-detect` enumerates exactly
that interface to learn which Razer devices are plugged in, and without it Synapse sees
no USB devices at all. Patch 23 builds on the same driver.

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

### Software compositing -- white and stale windows

**Symptom:** the flyout that opens from Synapse's tray icon is a white box, or nothing at
all. A window maximized by dragging it to the top of the screen grows, but what is in it
stays the size it was -- until the pointer happens to move over it, sometimes.

**Cause:** App Engine is Chromium, and Chromium draws its windows from its GPU process,
with Direct3D 11 -- wined3d on OpenGL here -- into windows that belong to another
process, the browser process. Wine draws into a window of the same process directly,
but into one of another process only offscreen: into a GL window of its own, which it
copies into the real one after each frame. Two things go wrong with that:

* the GPU process only renders offscreen while the window is clipped by another one.
  Otherwise it renders into a GL window that it cannot attach to a window of another
  process, and nothing it draws reaches the screen: a window never drawn stays blank, and
  one that has changed size keeps the last frame that was copied;
* offscreen, the copy is taken as the frame is swapped. Wine's GLX path waits for the
  swap to complete first (`glXWaitForSbcOML`); its EGL path, which it takes with NVIDIA's
  driver, does not, and copies the frame before. Right after a resize that is the frame
  drawn for the old size, and Chromium, which only draws when something changes, may not
  draw another.

**Fix:** none in Wine. The launcher starts App Engine with `--disable-gpu-compositing`,
which has Chromium's browser process paint each of its windows itself, with GDI, as any
Win32 program does. Its transparent windows, like the flyout, become layered windows,
which Wine draws with their alpha channel. Synapse's pages are mostly still: maximized
on a 4K panel it takes about 2% of a core.

App Engine started by Razer's own installer or updater rather than by the launcher --
once, at the end of the first install -- runs without the switch until it is next started
from the launcher.

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

### 26. A device that is gone is not in the present set

**Symptom:** after one Razer device is swapped for another -- a Naga V2 Pro for a Naga
V3 Pro -- Synapse opens a tab for the new one and never connects to it. Its log retries
until it gives up, though the HID devices it is looking for are there:

```
connectHidDevice(productId:232, ..., deviceContainerId:{153200E8-…}) hidDevices found base on productId, length:0
```

**Cause:** `node-rz-hid` reads each HID device's container ID the way old hidapi did:
it enumerates the interfaces of a `DIGCF_PRESENT | DIGCF_DEVICEINTERFACE` set, and for
the n-th interface asks `SetupDiEnumDeviceInfo` for the n-th device of the same set. On
Windows those are the same device, because a set of present interfaces holds only the
devices they belong to.

In Wine it held every device ever registered under the interface class. A device that
was unplugged keeps its registry keys, with its interfaces unlinked: setupapi skipped
the interfaces but added the device anyway. Every device unplugged since the prefix was
made -- here the old Naga's ten HID devices, and a game controller from long ago --
shifted the pairing by one, and the new mouse's collections were read as the old mouse's
and the ASUS LED controller's:

```
interface 4  1532:00e8 mi_00        device[4]  HID\VID_0B05&PID_1AA6&MI_02    {0B051AA6-…}
interface 5  1532:00e8 mi_01&col01  device[5]  HID\VID_1532&PID_00A8&MI_00   {153200A8-…}
...                                                         19 interfaces, 30 devices
```

It worked for the first device only because nothing had been unplugged yet; unplugging
any device winebus exposes -- a Razer one, or a game controller -- breaks every device
listed after it.

**Fix:** with `DIGCF_PRESENT`, a device goes into an interface set only if one of its
interfaces is linked -- what Windows does. The same prefix then gives 19 interfaces and
19 devices, each the interface's own.

### 27. Every process carried its own copy of App Engine's executable

**Symptom:** Synapse took 6 GB sitting in the tray. Each of App Engine's Electron
processes held 210-250 MB of private memory before it had done any work, and Synapse
runs nine or more of them: a renderer for every hidden window (tray menu, background
manager, lighting engine, notifications, Game Manager proxy, each device page) as
well as the main, GPU and network processes.

**Cause:** `RazerAppEngine.exe` is linked with 512-byte file alignment, as nearly every
Windows binary is, so its sections start at file offsets that are not page-aligned:
`.text` is at 0x400 in the file and 0x1000 in memory. `mmap()` cannot map that, so Wine
falls back to `pread()`-ing each section into anonymous memory -- a private copy in
every process that loads the image, where Windows shares one between them all. In
`/proc/<pid>/smaps` it is the anonymous, executable mapping at the image base:

```
140001000-14ad87000 r-xp 00000000 00:00 0          Rss: 177688 kB  Anonymous: 177688 kB
```

Every DLL App Engine ships (`libGLESv2.dll`, `ffmpeg.dll`, ...) was loaded the same way.

**Fix:** patch 27 has the wineserver write those sections once into a temp file laid
out the way the image is in memory, and hand it to every process mapping the image,
which maps the sections from it copy-on-write. Their pages are then shared like any
other file mapping:

```
140001000-14ad87000 r-xp 00001000 103:07 205807475   /var/data/wineprefix/#205807475 (deleted)
```

The temp file is an unnamed file in the prefix directory (`O_TMPFILE`), not memory, so
it behaves like any executable on disk: only the pages some process uses are in memory,
and the kernel can drop those and read them back. A copy in memory would hold every
page of every image, used or not -- 330 MB here, 95 MB of which nothing ever touched.
Where the prefix's filesystem cannot make such a file, it is a memfd.

The copy is kept per file, next to the one Wine already keeps for shared PE sections,
and lives as long as anything has the image mapped. Wine refuses to open a mapped image
for writing, so it cannot go stale.

### 28. Transparent huge pages, 2 MB at a time

**Symptom:** 4.6 GB of Synapse's 5.6 GB of anonymous memory was in transparent huge
pages (`AnonHugePages` in `/proc/<pid>/smaps_rollup`) -- 876 MB of the Game Manager
proxy's 932 MB. Most of that was the copies and the table that 27 and 29 get rid of,
but not all of it.

**Cause:** Arch's kernel sets `/sys/kernel/mm/transparent_hugepage/enabled` to
`always`. Windows programs commit memory a page at a time -- Chromium's allocators
reserve large regions and commit into them sparsely -- but with THP on `always` the
first touch anywhere in a 2 MB-aligned stretch of a committed region brings in a whole
2 MB page, and khugepaged goes on to collapse more of them for as long as Synapse runs.

**Fix:** patch 28 has every Wine process opt out with `prctl(PR_SET_THP_DISABLE)`,
which child processes and the wineserver inherit.

### 29. A terabyte of reserved address space cost hundreds of MB per process

**Symptom:** the background manager's renderer held a 268 MB block that was resident
and entirely zero: 67,358 of its 67,361 pages.

**Cause:** Wine keeps every page's protection in a table, one byte per 4 KB page of
address space. V8 reserves its sandbox up front -- about 1.4 TB in each of App
Engine's renderers -- and reserving writes the protection bytes for the whole range.
They are zero, but writing them faults in up to 350 MB of table per process.

**Fix:** patch 29 drops whole pages of the table that are being set to zero
(`madvise(MADV_DONTNEED)`) instead of writing them. They read back as zero without
being resident until something is committed in that range.

### 30. Direct3D buffers came in 64 MB pieces

**Symptom:** App Engine's GPU process had 165 MB mapped from `/dev/nvidiactl`, 131 MB
of it in two 64 MB mappings that were resident from end to end.

**Cause:** Chromium draws through Direct3D 11, which Wine implements in wined3d on top
of OpenGL. wined3d sub-allocates buffers out of 64 MB chunks, at least one for each
kind of memory in use, and keeps each chunk mapped -- which NVIDIA's driver backs with
pinned system memory. Windows that draw a few panels use a sliver of one.

**Fix:** patch 30 makes the chunks 4 MB. The allocator keeps its 4 KB minimum block,
so every allocation is aligned as before; a buffer over 2 MB gets memory of its own,
as one over 32 MB did before.

### What it adds up to

PSS of every process in the sandbox, from `/proc/<pid>/smaps_rollup`, with Synapse's
window closed to the tray.

| Process | Before | After |
| --- | ---: | ---: |
| Main | 822 MB | 262 MB |
| GPU | 714 MB | 208 MB |
| Network service | 244 MB | 27 MB |
| Renderer: Game Manager proxy | 935 MB | 45 MB |
| Renderer: tray menu | 666 MB | 62 MB |
| Renderer: device page | 628 MB | 73 MB |
| Renderer: background manager | 625 MB | 67 MB |
| Renderer: lighting engine | 621 MB | 60 MB |
| Renderer: notifications | 606 MB | 47 MB |
| **Total** | **5,991 MB** | **1,189 MB** |

The second total includes two renderers the first run did not have open, Synapse's own
page (181 MB) and its window (40 MB); App Engine closes the first of them some time after
the window is closed.

What is left of NVIDIA's driver in the GPU process is 38 MB mapped from
`/dev/nvidiactl`, down from 165 MB. Its largest remaining allocation is 49 MB of pixel
data in a Unix-side malloc arena, which grows with what the window drew while it was open.

### 31. A window that draws its own frame gets no title bar from the desktop

**Symptom:** maximized, Synapse gets the desktop's title bar across its top, over its
own -- its tabs, and its minimize, maximize and close buttons, half hidden underneath.

**Cause:** Synapse's window is a frameless Electron window with the `WS_CAPTION` and
`WS_THICKFRAME` styles: Chromium answers `WM_NCCALCSIZE` with a client area the size of
the whole window and draws a title bar of its own inside it. winex11 asks the window
manager for decorations only for a window that has a non-client area, and a normal
Synapse window has none. A maximized one does. On Windows a maximized window overhangs
the monitor by its frame on every side, and Chromium leaves that frame -- 4 pixels here
-- around the client area, for Windows to put off the screen:

```
window (-4,-4)-(3844,2164)    client (0,0)-(3840,2158)
```

That was enough for winex11 to decorate the window the way its style asks for, with a
title bar, buttons and a border. Wine leaves the window manager the non-client area those
take, 41 pixels at the top here, and KDE drew its title bar over the top 37 pixels of
Synapse's client area.

**Fix:** win32u decides that a window draws its own frame when the decorations its style
asks for would cover part of its client area, and says so to the driver with a new
`WINE_SWP_CUSTOM_FRAME` flag; winex11 then asks for no decorations. A maximized one is
shown only over the area it is maximized to, the monitor's work area -- where Windows
shows it too, its frame being off the screen:

```
window (-4,-4)-(3844,2164)    client (0,0)-(3840,2158)    X window 3840x2160, no decorations
```

Windows with frames of the usual kind are unaffected: Notepad keeps KDE's title bar,
normal and maximized.

### 32. A click on a tray icon lets its owner come to the front

**Symptom:** right-clicking Synapse's tray icon opens its menu now and then, and more often
does nothing.

**Cause:** Electron shows a tray icon's menu only once it has made its own window the
foreground window, and gives up if it cannot:

```cpp
if (!SetForegroundWindow(window_)) return;
```

Wine 11 has a foreground lock, as Windows does: a process may take the foreground only if
its last input is no older than that of the process in the foreground. A click on a tray
icon is input for Wine's explorer, whose window the icon is, so the application the click
is meant for is turned away:

```
explorer  set_foreground_window hwnd 0x100fc, mouse 1 -> ret 1     the click
Synapse   set_foreground_window hwnd 0x100fa, mouse 0 -> ret 0     before showing the menu
```

The menu did open when Synapse happened to have had input more recently, as on the
first right-click after using its window. On Windows the taskbar grants the icon's owner
the right as it passes the click on. `AllowSetForegroundWindow`, the call for that, was a
stub in Wine, and explorer never made it.

**Fix:** `AllowSetForegroundWindow` is implemented, with a new server request: a process
that may set the foreground window can let another one do so, once. Explorer's tray
calls it for an icon's owner before relaying a click, and the menu opens on every
right-click.

### 34. A message-only window made as Wine starts up stays message-only

**Symptom:** starting Synapse while it is already running -- from the application menu,
say -- does nothing: the running copy never brings its window back from the tray. Some
sessions only; in others it works.

**Cause:** App Engine is single-instance the way Chromium is. The first copy creates a
message-only window named after its profile directory, and a second copy finds it with
`FindWindowEx( HWND_MESSAGE, ... )`, sends it its command line and exits. In the sessions
where nothing happened, the first copy's window was not message-only at all, but an
ordinary top-level window, which `FindWindowEx( HWND_MESSAGE )` does not look among:

```
window 0x1006a   parent 0x2004c class "#32769"     the desktop window, not the message parent
```

A message-only window goes under the desktop's message parent, which Wine's explorer
creates as it starts, just after the desktop window itself. win32u asks the server for
both and caches the answer per thread. App Engine created its window while explorer was
still starting -- between the two -- and was told there was a desktop window and no message
parent. It cached the desktop window, created the window with no parent, which the server
makes a child of the desktop, and since it had a desktop window cached, never asked again.

**Fix:** a thread told there is no message parent yet waits for explorer to create it --
in steps of 10 ms, for up to a second, after which it has the server create one -- before
creating the window.

### 35. Razer's filter driver, done by winebus

**Symptom:** Synapse puts every RGB device it sets up into *driver mode* (`0x03`, command
class `0x00` id `0x04`), in which the firmware stops applying the assignments stored in
it and sends each button's default input instead, for the PC to remap. Under Wine nothing
did: a side button sent its default key whatever it was assigned, and the buttons that
have no key of their own -- the sensitivity and scroll mode buttons -- did nothing at all.
Synapse's mapping engine said why, every few seconds:

```
MappingEngineInputDevice::ProbeFilterDriverEndpointOnDeviceThread: ... error[fail to probe filter driver]
```

Two earlier patches worked round it. 25 kept devices in normal mode, so the firmware
applied the assignments Synapse also stores on the device, and 33 turned the changes the
firmware then made by itself back into the button presses Synapse expected. That covered
only what a device can do on its own, and every feature that reports back needed the
same treatment one at a time: what only the PC can do -- assignments a device cannot
store, PC-side macros, text, Synapse's scrolling features -- never worked. This patch
replaces both, and lets Synapse run devices the way it does on Windows.

**How it works on Windows:** two Razer kernel drivers, which Synapse's device modules
install.

* `RzDev_<pid>.sys` is a KMDF filter its INFs put above the device's HID keyboard and mouse
  collections (and below its USB interfaces). It sees every `KEYBOARD_INPUT_DATA` and
  `MOUSE_INPUT_DATA` before the system does, and it creates a child device for each:
  `RZCONTROL\VID_1532&PID_<pid>&MI_<nn>`, compatible ID `RAZER\DeviceContol`.
* `RzCommon.sys` binds to those children and gives applications the *Razer Control Device*,
  interface class `{e3be005d-d130-4910-88ff-09ae02f680e9}`.

The mapping engine (`mapping_engine.dll`) enumerates that interface class, keeps the
devices whose container ID is its device's and whose path names its PID, and opens them.
Through `0x8888xxxx` IOCTLs it then asks for

* *hooks* -- particular inputs, a key by scan code or a mouse button by its flag bits, with
  the modifier keys held: the filter holds those back and queues them as events;
* *redirection* of every keyboard or mouse input, held back likewise;
* *notification*: copies of every input, which go on to the system as usual;

and reads the events with a request that stays pending until there is one, a 0x130-byte
record. What it makes of them it injects back through `IOCTL 0x88883020`, as keyboard or
mouse input the filter hands on as if the device had sent it. With Synapse's wheel
features, for one, every wheel notch is hooked and Synapse sends its own scrolling back.

Wine can load neither driver -- there is no KMDF -- and the input never passes through
Wine anyway: the desktop reads it from the kernel's evdev nodes.

**Fix:** winebus does both drivers' job, behind the same interface.

* For each Razer USB device it has hidraw nodes for, winebus adds a child to its bus:
  `RZCONTROL\VID_1532&PID_<pid>&MI_00`, without the compatible ID, so no driver is installed
  for it. It is a raw device (`RawDeviceOK`), started as such, and has the device's
  container ID and the control device's interface. It answers RzCommon's requests: which
  events each handle receives, the hook table, redirection, the pending reads, input to
  inject.
* Its Unix side takes the device's input over for as long as some handle wants hooks or
  redirection. It grabs the device's evdev nodes (`EVIOCGRAB`), so the desktop no longer
  reads them, and creates a uinput device for each that copies it -- name, IDs, keys, axes,
  LEDs -- so the desktop applies its settings for the device to the copy. It waits to grab
  a node while one of its keys is down, since the desktop would never see that key come
  up, and hands the LEDs the desktop sets on a copy on to the device, which takes them only
  from whoever holds the grab. A handle that only wants notification gets it without a
  grab. Each frame read from a
  node becomes `KEYBOARD_INPUT_DATA` (set 1 scan codes, `E0` and `E1` flags) or
  `MOUSE_INPUT_DATA` (button flags, the wheels a notch at a time), and goes through the same
  steps as in `RzCommon.sys`: redirection, notification, then the hook table -- a release
  goes to the hook that took the press, a press to the hook for exactly the modifiers held,
  then for a subset of them, then for none. What no hook takes goes on to the copy in the
  same frame, and pointer movement always does. When evdev reports lost events, the keys
  known to be down are brought back in line with the device, so nothing stays held. Input
  the mapping engine injects goes out through a copy that has the key or axis.
* When the last handle closes -- Synapse quitting, or its process ending -- the hook table is
  cleared, the nodes are let go and the copies removed, and the device is the desktop's
  again.

With a Naga V3 Pro on the Mouse Dock Pro, as Synapse connects:

```
control 00007FFFFF30BFF0 is L"\\??\\RZCONTROL#VID_1532&PID_00A4&MI_00#8&5501&0200000000000000#{e3be005d-...}"
filter 1 subscriptions 0x6                          hooks and notification
grabbed "/dev/input/event6" "Razer Razer Mouse Dock Pro", keyboard 0     ... and four more
filter 1 add hook type 0x2 unit 0 buttons 0x400 data 0xff88 modifiers 0    a wheel notch down
filter 1 kind 0x2 type 0x2 buttons 0x400 data 0xff88                       ... hooked
filter 1 mouse flags 0 buttons 0x400 data 0xff88                           and Synapse's own, sent back
filter 1 key 0x1a down                              a button assigned "[" in Synapse
```

`WINEDEBUG=+razer` shows all of this. Passing input on costs nothing measurable: at the
mouse's 8000 Hz, the dock's hidraw node and the copy the desktop reads both showed 32,186
reports in five seconds of circles, 125 µs apart, one frame lost to the start of the
measurement.

The buttons with no key of their own still reach Synapse the way they did: the firmware
reports them as Razer keys (`04 20`, `04 21`, `04 54`) on the device's vendor collection,
which the mapping engine reads from HID itself, and the desktop ignores.

This needs two more lines in the udev rule: the evdev nodes of Razer devices and
`/dev/uinput`, for the seat's user (see `udev/60-razer-synapse.rules`). Without them
winebus logs why, and the device stays in driver mode with nothing to remap it.

What is not done, and logs a FIXME when Synapse asks for it: redirection of pointer
movement (which Razer's driver takes from below the HID stack), absolute pointer
movement, gamepad and raw consumer-control output, and sensor rotation, which Razer's
drivers for these devices do not do either.

## Things that are not fixed, and why

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
