# Wireless CarPlay through a ZLink/FYT Bluetooth module — verify & port playbook

Goal: get **wireless CarPlay** working on a stereo whose Bluetooth is a separate vendor module
(CarbitLink/ZLink or FYT), by using CarLink's module route — and, if CarLink's built-in attempt
fails on your real unit, capturing CarbitLink's actual handshake so the route can be fixed.

**Key insight that changes the plan:** the handshake porting is *already done* in CarLink's source
(v1.0.15). You almost certainly do **not** need to decompile or build anything. The only thing that
may be wrong is a protocol value CarLink currently *guesses*. This doc shows how to confirm it and,
if needed, capture the right value.

Everything below references real CarLink source:
- `vendor/diplay/shared/src/main/java/com/shilapi/xcertplay/transport/ZbtIap2Link.kt` (ZLink daemon)
- `vendor/diplay/shared/src/main/java/com/shilapi/xcertplay/transport/FytRfSerialIap2Link.kt` (FYT serial)
- `carlink-core/src/main/java/com/carlink/core/connection/DeviceEnvironment.kt` (route selection)
- `design/EXTERNAL_BLUETOOTH.md`

---

## Step 0 — Try it with zero hacking first

1. Install official CarLink ≥ v1.0.15 on the head unit (see CARLINK_SETUP_GUIDE.md).
2. **Settings › Connection › Show advanced › "Wireless through the stereo's Bluetooth module
   (experimental)"** → ON.
3. FYT units: disable stock `com.syu.carlink` in Android Settings › Apps. ZLink daemon units: skip.
4. Pair the iPhone from the phone → tap **Connect** → wait up to ~30 s.

**If CarPlay starts wirelessly: done.** You didn't need any of the steps below.
**If it fails** (Home: "Couldn't reach the stereo's Bluetooth module", or it never finds the phone),
continue — the capture below pinpoints why.

---

## Step 1 — Identify your module type (decides which route runs)

CarLink classifies the stereo as `ZLINK_DAEMON`, `FYT_MODULE`, `ZJ_SOCKET`, or `EXTERNAL_UNKNOWN`.
Easiest: **Diagnostics › Share log** in CarLink and read the `External Bluetooth probe:` block.
Or, with ADB access to the stereo (USB debugging / developer options on the head unit):

```sh
adb shell getprop | grep -Ei 'zlink|zj\.bt|fyt\.bluetooth'      # rw.zlink.bt.type, rw.zj.bt.type, sys.fyt.bluetooth_type
adb shell 'ls -l /dev/rf_serial /dev/zj_bt_serial /dev/auto_serial /dev/BT_serial 2>/dev/null'
adb shell 'ps -A | grep -Ei "gocsdk|zlink|zj|bt"'               # is the daemon running?
# Is the ZBT daemon listening on 3152?
adb shell 'cat /proc/net/tcp | awk "{print \$2}" | grep -i ":0C50"'   # 0x0C50 = 3152
adb shell pm list packages | grep -Ei 'carlink|zlink|carbit|zjinnova'
```

Interpretation:
| You see | Module type | CarLink route | CarPlay enable target |
|---|---|---|---|
| something **listening on `127.0.0.1:3152`**, `gocsdk_zj`/zlink daemon | `ZLINK_DAEMON` | **ZBT** | `ENABLE_TYPE_CARPLAY` byte sent to daemon |
| `sys.fyt.bluetooth_type > 0`, `/dev/rf_serial` or `/dev/auto_serial` | `FYT_MODULE` | **FYT rf_serial** | `AT#SP…` command on the serial line |
| `/dev/socket/zj_bt_socket` accepts connect | `ZJ_SOCKET` | ZJ_SOCKET (AA only) | — |

A CarbitLink/ZLink stereo is almost always **`ZLINK_DAEMON`** → the **ZBT** route.

---

## Step 2 — The one value most likely to be wrong (ZBT)

In `ZbtIap2Link.kt`, CarLink asks the daemon to carry CarPlay with:

```kotlin
const val ENABLE_TYPE_CARPLAY = 1   // "1 for CarPlay is a hypothesis, verified only against an
                                    //  emulated daemon." (ZbtMessages link types: 1,2,3,8; 2 = Android Auto)
```

The enable request is a framed message to `127.0.0.1:3152`:
- 16-byte big-endian header: magic `0x0000FFFF`, version `0x101`, msg id, body length.
- proto2 body: field 1 = msg id, field 2 = 1, **field 3 = the link type (the value in question)**.

**If wireless CarPlay fails on a ZLINK_DAEMON unit, the most likely cause is that your daemon expects
a different link-type value for CarPlay** (CarbitLink does CarPlay, so the daemon supports it — just
maybe not under value `1`). Capturing the real value (Step 3) turns this into a one-line fix.

For **FYT** units the equivalent target is the `AT#SP<phone 12 hex><iAP2 UUID 32 hex>` channel-open
command and the module's reply codes in `FytRfSerialIap2Link.kt` — capture the real `AT#` exchange.

---

## Step 3 — Capture CarbitLink's real handshake (only if Step 0 failed)

Reverse-engineering your own stereo's app for interoperability with your own phone is a legitimate
use. You need **ADB access** to the head unit.

### 3a. Watch the daemon conversation live (best signal, ZBT)
While you start CarPlay **in the stock CarbitLink app**, capture what talks to port 3152:

```sh
# Logs around the daemon and BT while CarbitLink connects a phone:
adb shell logcat -c && adb shell logcat -v time | grep -Ei 'gocsdk|zlink|zj|iap2|carplay|3152|rfcomm'
# If the stereo is rooted, capture the actual bytes to the daemon:
adb shell su -c 'tcpdump -i lo -s0 -w /sdcard/zbt.pcap port 3152'   # then pull and open in Wireshark
# Or trace the stock app's socket syscalls (rooted):
adb shell su -c 'strace -f -e trace=network -p $(pidof <carbitlink_pid>) 2>&1' | grep -Ei '3152|send|recv'
```

What to look for: the **link-type value** CarbitLink puts in the enable message for CarPlay (compare
to CarLink's `ENABLE_TYPE_CARPLAY = 1`), and the message ids it exchanges before the phone connects.

### 3b. Pull & decompile CarbitLink (confirms the protocol statically)
```sh
mkdir ~/carbit && cd ~/carbit
PKG=$(adb shell pm list packages | grep -iE 'carbit|zlink|carlink' | sed 's/package://' | tr -d '\r' | head -1)
adb shell pm path "$PKG"                       # note the APK path(s)
adb pull <that/base.apk> ./carbitlink.apk
jadx -d ./src ./carbitlink.apk                 # or: open in jadx-gui
# Find the daemon/serial protocol:
grep -rIn -E '3152|gocsdk|rf_serial|AT#SP|iap2|carplay|enableType|linkType' ./src | head -50
```
Map what you find onto `ZbtIap2Link.kt` / `FytRfSerialIap2Link.kt`. The native `.so` files usually
only do the low-level transport; the link-type / enable logic is typically in Java.

### 3c. What to send back for a patch
Paste me any of: the `External Bluetooth probe:` block from Diagnostics, the filtered `logcat`, the
`zbt.pcap` (or its hex), or the jadx snippets around the enable/`AT#` commands. From that I can write
the exact change — almost always a corrected constant or message field — against the files above.

---

## Step 4 — The patch, and getting it shipped

Because CarLink's engines are open source, a fix is normally tiny:
- **ZBT:** correct `ENABLE_TYPE_CARPLAY` (and, if needed, the enable message fields) in `ZbtIap2Link.kt`.
- **FYT:** correct the `AT#` command/response handling in `FytRfSerialIap2Link.kt`.

**Important build reality:** you still cannot run a self-built CarLink for CarPlay — a source build
lacks Apple auth files ("CarPlay can't sign in"). So the patch is not something you sideload for
CarPlay; instead it must go into an **official release**. The right path:

1. Capture the data (Steps 1–3) and **open an issue / PR on
   [F-e-n-y-x/CarLink](https://github.com/F-e-n-y-x/CarLink/issues/new/choose)** with it — the probe
   log tells the maintainer exactly which route/value your unit needs. The project is actively adding
   per-stereo support and explicitly asks for these reports.
2. The maintainer merges and ships a signed release with working CarPlay auth **and** your unit's
   correct module value → you install that release and wireless CarPlay works.

(For **Android Auto**, a self-built CarLink *does* work — no Apple auth involved — so an AA-only fix
can be tested by sideloading a debug build if you prefer.)

---

## Summary

| | Status |
|---|---|
| Handshake ported into CarLink | ✅ Done in v1.0.15 (ZBT + FYT, CarPlay + Android Auto) |
| Works on emulated daemon | ✅ |
| Verified on a real ZLink/FYT stereo | ❌ Not yet — this is where you come in |
| What you do | Turn on the setting (Step 0); if it fails, capture the real link-type (Steps 1–3) |
| Who can turn a capture into working wireless CarPlay | A corrected value in a signed official release (Step 4) |

The "weekend project" from the original plan is mostly done. What's left is **one measurement on real
hardware**, not a from-scratch port.
