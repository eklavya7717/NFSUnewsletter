# Getting CarLink working on a ZLink / CarbitLink-type Android stereo

Practical setup guide for **[CarLink](https://github.com/F-e-n-y-x/CarLink)** (F-e-n-y-x/CarLink,
AGPL-3.0) — "CarPlay and Android Auto in one app, for any Android car stereo." Written for a
head unit whose stock app is **CarbitLink / ZLink** (a ZLink-type unit).

Grounded in CarLink's own docs **and its source** (v1.0.15):
[README](https://github.com/F-e-n-y-x/CarLink#readme),
[User guide](https://github.com/F-e-n-y-x/CarLink/blob/main/docs/USER_GUIDE.md),
[Troubleshooting](https://github.com/F-e-n-y-x/CarLink/blob/main/docs/TROUBLESHOOTING.md),
[External Bluetooth modules](https://github.com/F-e-n-y-x/CarLink/blob/main/design/EXTERNAL_BLUETOOTH.md).

> **Companion doc:** if wireless CarPlay doesn't start on your unit, see
> **[CARLINK_WIRELESS_MODULE_PORT.md](CARLINK_WIRELESS_MODULE_PORT.md)** — how to verify the
> module route and capture CarbitLink's real handshake to fix it.

---

## TL;DR

1. **Install the official release APK** (≥ v1.0.15) on the head unit — Android 7.1+. Do **not**
   build from source: a source build **can't do CarPlay** (no auth files). See
   [Why the release](#why-the-official-release-apk-and-not-a-build).
2. **USB cable = guaranteed, full CarPlay/Android Auto.** Try this first to confirm everything works.
3. **Wireless CarPlay through your stereo's Bluetooth module is now implemented** in CarLink (v1.0.15,
   *experimental*). Turn on **Settings › Connection › Show advanced › "Wireless through the stereo's
   Bluetooth module (experimental)"** and try it. It is **unverified on real ZLink daemons**, so it
   may or may not work on your exact unit — if it doesn't, the port doc shows how to fix it.

---

## Install

1. On the stereo (or a PC, then copy over by USB), open:
   **https://github.com/F-e-n-y-x/CarLink/releases/latest**
2. Download **`CarLink-X.Y.Z.apk`**.
3. *(Recommended)* Verify: download the matching `.sha256` and run
   `sha256sum -c CarLink-X.Y.Z.apk.sha256` → `OK`. Release signing cert SHA-256:
   `03:74:68:CA:5B:EB:70:EA:5F:68:54:33:29:B0:33:68:24:7D:FB:39:3D:3F:62:6A:6B:47:2D:10:E5:65:7D:21`
4. Open the APK on the stereo → **"Allow from this source"** → **Install**.
5. Open CarLink, run first-time setup (**Allow all** → car brand / **Other** → pair or plug in).

> [Obtainium](https://github.com/ImranR98/Obtainium) can auto-track releases
> (`Add app` → `https://github.com/F-e-n-y-x/CarLink`); CarLink also self-updates (official build only).

---

## Why the official release APK, and not a build

CarLink's troubleshooting is explicit:

> **"CarPlay can't sign in" — This build has no CarPlay authentication files: you are running a
> build from source. Install the release APK. Android Auto works either way.**

CarPlay needs Apple authentication material that ships **only** in the official signed release. A
self-compiled APK **cannot do CarPlay**. So "fork and build CarLink" does not help the CarPlay
goal — use the release. (This is also why there was no point building it in a sandbox.)

---

## USB — the path that always works

Plug the iPhone (or Android phone) into the stereo with a good USB cable.

- iPhone → **CarPlay over USB**; Android phone → **Android Auto over USB**. CarLink auto-detects.
- Accept **"Use CarPlay"** on the phone; keep it unlocked. First connect can take ~30 s.
- **USB is unaffected** by the Bluetooth-module limitation below.

Black screen with sound on iOS 27? **Settings › Display › Direct video output** → *Turn on and
reconnect*. The iPhone showing "Personal Hotspot" during USB CarPlay is normal (local cable link, no
internet).

---

## Wireless on a ZLink/CarbitLink unit — the real situation

**Why wireless is hard here:** cheap Android stereos (FYT and **ZLink-type units** — yours) don't use
Android's Bluetooth for phones. Their Bluetooth is a **separate vendor module on a serial link**; the
phone pairs with *that*, and Android's own adapter can't reach the phone through it. Both CarPlay
(iAP2) and Android Auto normally start over Android's Bluetooth — which goes nowhere on these units.

**What CarLink now does about it (v1.0.15):** it carries the wireless handshake *through* the module.
Two routes are implemented, picked automatically from what the module looks like:

- **ZBT** — ZLink-type daemon on TCP `127.0.0.1:3152` (likely your CarbitLink unit).
- **FYT BLINK / rf_serial** — FYT's module over `/dev/rf_serial` or `/dev/auto_serial`.

This covers **both Android Auto and CarPlay** (the published Troubleshooting page still says "CarPlay
is not covered" — that text lags the code; the source and CHANGELOG confirm CarPlay is wired in).

**How to turn it on:**

1. **Settings › Connection › Show advanced › "Wireless through the stereo's Bluetooth module
   (experimental)"** → ON. (CarLink often auto-detects the module and enables it; the toggle is also
   on the Bluetooth chip on Home.)
2. **On an FYT stereo only:** disable the stock **"Car Link"** app (`com.syu.carlink`) in Android
   Settings › Apps — it owns the same serial port. (ZLink daemon units don't need this.)
3. Pair the iPhone from the phone's Bluetooth settings (to the module), then tap **Connect**.

**The honest caveat — it may not work on your exact unit.** From CarLink's own source comments:
the ZBT CarPlay path is *"a hypothesis, so far verified only against an emulated daemon"*, and the
FYT path's protocol was *"read from FYT's own CarPlay, not yet tried on a unit."* So treat it as a
real attempt that might fail on real hardware. **If it fails → [the port doc](CARLINK_WIRELESS_MODULE_PORT.md)**
tells you exactly what to capture to make it work (usually a single protocol value).

**If it doesn't work and you don't want to debug:** USB cable gives you full CarPlay today.

---

## Quick decision tree

```
iPhone / CarPlay:
  USB cable                         → ✅ Works now.
  Wireless (Settings → module route)→ ⚠️ Experimental; try it. If "module didn't answer" or it
                                        won't start → capture the handshake (port doc) → fixable.

Android phone / Android Auto:
  USB cable                         → ✅ Works now.
  Wireless (same module toggle)     → ⚠️ Experimental (ZBT/ZJ_SOCKET/BLINK); try it.
```

---

## Help get your unit officially supported

CarLink's **Diagnostics › Report on GitHub** (or *Share log* →
[new issue](https://github.com/F-e-n-y-x/CarLink/issues/new/choose)) attaches a hardware probe
(module classification, serial-port nodes, system properties, the `127.0.0.1:3152` check, stock
apps). That is the fastest route to getting your specific stereo verified/fixed upstream.

---

## Requirements recap

- **Display:** Android **7.1+** (API 25), ARM — your stereo. Install CarLink **there**, not the phone.
- **Phone:** iPhone (CarPlay) or Android (Android Auto); USB cable, or BT+Wi-Fi for wireless.
- **Package:** `com.carlink.app`, signed with CarLink's release key (fingerprint above).

---

*Summarizes CarLink's published docs and source for convenience. CarLink is third-party software
(AGPL-3.0), not affiliated with this repository. Always verify the APK before installing.*
