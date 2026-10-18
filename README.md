# Xbox 360 pad rumble bridge (xrumbled)

A Magisk module that adds rumble to USB Xbox 360-class gamepads on Android when the kernel's `xpad` driver has no force feedback.

Built and tested on a Redmi Pad SE (Android 14, HyperOS, Magisk) with an 8BitDo Pro 2 in X-input mode. Other pads and devices may work but are untested.

**Current version: v0.4.1.** The daemon only runs while a pad is plugged in, so it uses no CPU when no controller is connected.

---

## How it works

1. **`bin/xrumbled`** is a small background daemon that talks to the pad over USB and handles the rumble. This is the part that actually vibrates the controller.
2. **`service.sh`** runs at boot. It turns on Android's `vibrate_input_devices` setting, so normal system vibration (games, haptics) is routed to gamepads. Then it starts the USB watcher (below).
3. **`hotplug.sh`** decides whether the daemon should be running. It starts `xrumbled` when a matching pad is present and stops it when the pad is gone.
4. **`xrumbled.conf`** lists which USB IDs count as your pad and turns debugging on or off.

### How a pad is recognised

`hotplug.sh` reads `/sys/bus/usb/devices/` and starts the daemon if either is true:

- a device's `idVendor:idProduct` matches a `pad=` line in `xrumbled.conf`, or
- any USB interface has the Xbox 360 signature: class `ff`, subclass `5d`, protocol `01`.

The second rule means most 360-class pads are detected automatically, even if their IDs aren't in the config.

### Files

| File | Purpose |
|------|---------|
| `module.prop` | Magisk module info |
| `service.sh` | Boot script: sets vibration routing, starts the USB watcher |
| `hotplug.sh` | Starts or stops the daemon depending on whether the pad is present |
| `xrumbled.conf` | Config (pad IDs, debugging) |
| `bin/xrumbled` | The rumble daemon |
| `uninstall.sh` | Stops the daemon and watcher when the module is removed |

Config location after install: `/data/adb/modules/xrumbled/xrumbled.conf`

---

## Install

1. Flash the zip in Magisk and reboot.
2. Plug in the pad. The daemon should start within a couple of seconds.

To remove it, delete the module in Magisk and reboot.

---

## Config

```
# pad=VID:PID   extra USB ids that should start the rumble daemon (one per line)
debugging=false
pad=0020:0b21
pad=045e:028e
```

| Option | Meaning |
|--------|---------|
| `pad=VID:PID` | USB ID of a pad that should start the daemon. Repeat the line for more pads. |
| `debugging=true` | Writes the daemon output to `/data/local/tmp/xrumbled.log`. Off by default. |

To find your pad's ID, plug it in and run (as root):

```
for d in /sys/bus/usb/devices/*; do echo $d $(cat $d/idVendor 2>/dev/null):$(cat $d/idProduct 2>/dev/null); done
```

The pad is usually the entry that is not a `usb1` / `usb2` root hub. Edit the config, then unplug and replug the pad.

---

## CPU polling optimisation

### The problem (v0.3)

The original setup ran the daemon all the time. While no pad was connected, the daemon looped forever:

1. open `/dev/bus/usb`, list every device,
2. open each one and read its descriptors,
3. `sleep(2)` and repeat.

That is a wake-up every 2 seconds, doing real work each time, whether or not a controller was ever plugged in.

### The fix (v0.4 and later)

Nothing polls. The module asks Android to tell it when USB devices change:

```
boot
 └─ service.sh
     ├─ runs hotplug.sh once (pad may already be plugged in)
     └─ inotifyd  ──blocks, using no CPU──┐
                                           │ USB node created or deleted
                                           ▼
                                      hotplug.sh
                                  pad present, daemon not running → start xrumbled
                                  pad gone,    daemon running     → stop xrumbled
```

- **`inotifyd`** is a built-in Android tool. It sleeps in the kernel until a node under `/dev/bus/usb` is created or deleted, then runs `hotplug.sh`. No timer, no loop.
- **Debounce:** `hotplug.sh` waits one second for the USB nodes to settle, checks, waits another second, and checks again. This catches changes that happen during enumeration.
- **Lock:** a lock directory (`/data/local/tmp/xrumbled.lock`) means only one check runs at a time, so a burst of events can't start two daemons.
- **Fallback:** if `inotifyd` isn't available anywhere (system, Magisk busybox or KernelSU busybox), the module falls back to running the daemon permanently, as before.

### Result

| State | v0.3 | v0.4.1 |
|-------|------|--------|
| No pad connected | Wakes every 2 s, scans USB | No process, no wake-ups |
| Pad plugged in | Daemon running | Daemon starts within a couple of seconds |
| Pad unplugged | Daemon keeps scanning | Daemon stopped |

### What remains

While a pad is connected, the daemon still wakes on its own `poll()` timeouts (roughly once a second, plus a short one in one code path). This is small and only happens while a pad is in use. It is inside the compiled daemon, so removing it needs a change to the daemon's source.

---

## Debugging

### 1. Is the daemon running?

```
su -c "ps -A | grep xrumbled"
```

- Pad unplugged: should print nothing.
- Pad plugged in: should show `xrumbled`.

### 2. Is the pad being detected?

```
su -c "for d in /sys/bus/usb/devices/*; do echo \$d \$(cat \$d/idVendor 2>/dev/null):\$(cat \$d/idProduct 2>/dev/null); done"
```

Check that your pad appears and that its ID is listed in `xrumbled.conf`. To check the interface signature:

```
su -c "cat /sys/bus/usb/devices/1-1:1.0/bInterfaceClass /sys/bus/usb/devices/1-1:1.0/bInterfaceSubClass /sys/bus/usb/devices/1-1:1.0/bInterfaceProtocol"
```

Replace `1-1` with your pad's entry. A 360-class pad prints `ff`, `5d`, `01`.

### 3. Does the watcher see USB changes?

```
su -c "inotifyd - /dev/bus/usb/001:nd"
```

Plug and unplug the pad. Lines should print. If nothing prints, try the other bus folders under `/dev/bus/usb/`. Press Ctrl+C to stop.

### 4. Run the check by hand

```
su -c "sh /data/adb/modules/xrumbled/hotplug.sh"
```

This does one check and starts or stops the daemon to match. It is the same thing the watcher runs on every USB event.

### 5. Turn on logging

Set `debugging=true` in `xrumbled.conf`, then unplug and replug the pad.

```
su -c "cat /data/local/tmp/xrumbled.log"
```

This shows the daemon's own output. Turn debugging back off afterwards so nothing keeps writing to the log.

### 6. Check vibration routing

```
su -c "settings get system vibrate_input_devices"
```

Should print `1`. If it doesn't, rumble from normal apps won't reach the pad.

### Common problems

| Symptom | Likely cause | What to do |
|---------|--------------|------------|
| Daemon never starts | Pad ID not in the config and the pad doesn't have the 360 interface signature | Add a `pad=VID:PID` line (see Config) |
| Daemon starts but no rumble | The app isn't sending vibration, or `vibrate_input_devices` is off | Check step 6 and the log in step 5 |
| Daemon runs with the pad unplugged | Stale process, or the watcher isn't running | Run step 4; check `ps -A | grep inotifyd` |
| Nothing happens after flashing | Needs a reboot | Reboot once |
| `service.sh` can't find `inotifyd` | Not on this ROM | The module falls back to always-on mode; see the fallback above |

---

## Version history

| Version | Change |
|---------|--------|
| v0.3 | Daemon started at boot and ran permanently, scanning USB every 2 s |
| v0.4 | Event-driven: `inotifyd` plus `hotplug.sh`, config file, optional logging |
| v0.4.1 | Pad detection by interface signature as well as ID; your pad's ID added to the defaults |

---

## Credits

Module and daemon by Jamagoze.
