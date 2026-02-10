# Skeletor-ai/logiops Fork Notes

Fork of [PixlOne/logiops](https://github.com/PixlOne/logiops) with community patches applied to fix MX Master 3/3S issues.

## Problem

After every boot, the logid service fails to properly initialize the MX Master 3S scroll wheel. The service must be manually restarted (`systemctl restart logid`) for scrolling to work.

## Root Cause Analysis

The issue has **two contributing factors**:

### 1. Race Condition in Device Detection (Primary)
When the system boots, udev triggers multiple `add` events for the same hidraw device in rapid succession. `logid`'s `DeviceMonitor::addDevice()` is not thread-safe, causing concurrent calls that corrupt the device initialization. The device gets added in a broken state (buttons work, but scroll/hires features fail to initialize).

**Evidence:** PR #513 shows `_addHandler()` being called 4-6 times on the same device within the same second. Adding a mutex serializes these calls and fixes the issue.

### 2. Bolt Receiver Operation Order Bug
For devices connected via Logi Bolt receiver, `ReceiverMonitor::_addHandler` called `addDevice()` before clearing the waiter, which could lead to deadlocks or failed detection.

## Applied Patches

| PR | Title | What it fixes |
|----|-------|---------------|
| [#513](https://github.com/PixlOne/logiops/pull/513) | Guard device adding/removing with mutex | **Boot race condition** - the primary fix for scroll-not-working-after-boot |
| [#460](https://github.com/PixlOne/logiops/pull/460) | Fix Bolt receiver support | Device detection failures with Logi Bolt receivers |
| [#413](https://github.com/PixlOne/logiops/pull/413) | Various fixes for scroll wheel | Low-res axis registration, scroll delta handling, responsive scrolling |
| [#537](https://github.com/PixlOne/logiops/pull/537) | Fix thumbwheel bidirectional scrolling | Left direction on thumbwheel, touch/tap gestures, bidirectional axis support |

## Additional: Contrib Workarounds

For maximum reliability, two workaround files are provided in `contrib/`:

- **`90-logid-restart.rules`** — udev rule that restarts logid when a Logitech hidraw device appears (USB or Bluetooth)
- **`logid.service.d-override.conf`** — systemd override adding bluetooth dependency, restart-on-failure, and a small startup delay

Install instructions are in the files themselves.

## Relevant Issues Found

### Boot/Restart Problems (the core issue)
- **#522** — logid misses MX Master 3S at boot (UHID/hidraw race) — detailed analysis with udev workarounds
- **#508** — MX3 scroll doesn't work on boot — confirms scroll broken, buttons fine
- **#481** — logid service fails to start properly after boot (MX Master 3S)
- **#156** — "Error adding device" — need to restart logid service (50+ affected users)
- **#469** — Mouse not initialized unless being moved
- **#492** — Daemon does not find mice and keyboard when Ubuntu boots/logs in
- **#148** — Driver does not work after suspend
- **#461** — One successful run with MX Master 3S, following failures

### Scroll Wheel Issues
- **#386** — logid stops processing hiresscroll gestures after mouse reboot
- **#370** — Always scroll up with version 0.3
- **#536** — ThumbWheel left direction not working (fix included)
- **#411** — MX Anywhere 3 scroll wheel not working

### Bolt Receiver Issues
- **#463** — MX Master 3S with Bolt not detected
- **#388** — MX Master 3S problem with bolt receiver
- **#402** — Bug in waitForDevice (two devices)

## What's Still Open

- **Suspend/resume** — The mutex fix helps but may not fully solve reconnection after suspend. The udev workaround handles this.
- **Solaar conflict** — If Solaar is installed alongside logiops, they fight for device control. Uninstalling Solaar helps.
- **`io_timeout` config** — Some users report that increasing `io_timeout: 60000` in logid.cfg helps with slow Bluetooth init.
- **Upstream is slow** — The main repo has many unmerged PRs. This fork consolidates the most impactful ones.

## Build Instructions

```bash
mkdir build && cd build
cmake ..
make -j$(nproc)
sudo make install
```

Dependencies: `cmake`, `libevdev-dev`, `libudev-dev`, `libconfig++-dev`, `libglib2.0-dev`, `libdbus-1-dev`
