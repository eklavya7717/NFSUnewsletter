# Getting CarLink working on a ZLink / CarbitLink-type Android stereo

Practical setup guide for **[CarLink](https://github.com/F-e-n-y-x/CarLink)** (F-e-n-y-x/CarLink,
AGPL-3.0) — "CarPlay and Android Auto in one app, for any Android car stereo." Written for a
head unit whose stock app is **CarbitLink / ZLink** (a ZLink-type unit).

Everything here is drawn from CarLink's own documentation
([README](https://github.com/F-e-n-y-x/CarLink#readme),
[User guide](https://github.com/F-e-n-y-x/CarLink/blob/main/docs/USER_GUIDE.md),
[Troubleshooting](https://github.com/F-e-n-y-x/CarLink/blob/main/docs/TROUBLESHOOTING.md),
[External Bluetooth modules](https://github.com/F-e-n-y-x/CarLink/blob/main/design/EXTERNAL_BLUETOOTH.md)).

---

## TL;DR

1. **Download the official release APK** onto the stereo (or a PC, then copy by USB).
   Do **not** build it yourself — see [Why you must use the release](#why-the-official-release-apk-and-not-a-build).
2. **Install it on the head unit** (Android 7.1+), not on your phone.
3. **Connect by USB cable first.** This gives you full CarPlay and Android Auto and is
   **unaffected** by the stereo's Bluetooth limitation.
4. **Wireless CarPlay will most likely NOT work on this stereo** — its Bluetooth is a separate
   vendor module that apps cannot use for CarPlay. This is a hardware/firmware wall, not a setup
   mistake. USB is the reliable path, identical experience.

---

## Install

1. On the stereo's browser (or a computer, then copy the file over by USB stick/cable), open:
   **https://github.com/F-e-n-y-x/CarLink/releases/latest**
2. Download **`CarLink-X.Y.Z.apk`**.
3. *(Optional but recommended)* Verify the download — download the matching
   `CarLink-X.Y.Z.apk.sha256` from the same release and run:
   ```sh
   sha256sum -c CarLink-X.Y.Z.apk.sha256      # prints "CarLink-X.Y.Z.apk: OK"
   ```
   The official build is signed with CarLink's release key; certificate SHA-256:
   `03:74:68:CA:5B:EB:70:EA:5F:68:54:33:29:B0:33:68:24:7D:FB:39:3D:3F:62:6A:6B:47:2D:10:E5:65:7D:21`
4. Open the APK on the stereo. Android asks to **"Allow from this source"** (Install unknown
   apps) — allow it, go back, tap **Install**.
5. Open CarLink and run first-time setup (**Allow all** permissions → pick car brand / **Other** →
   pair or plug in).

> **Tip:** [Obtainium](https://github.com/ImranR98/Obtainium) can auto-track CarLink releases —
> paste `https://github.com/F-e-n-y-x/CarLink` into *Add app*. CarLink also self-updates from
> GitHub Releases (official build only).

---

## Why the official release APK, and not a build

CarLink's troubleshooting is explicit:

> **"CarPlay can't sign in" — This build has no CarPlay authentication files: you are running a
> build from source. Install the release APK from Releases. Android Auto works either way.**

CarPlay requires Apple authentication material that ships **only** in the official signed release.
A self-compiled APK (or any APK signed with a different key) **cannot do CarPlay** and cannot
update the official app. So the earlier "fork the repo and build it" plan does **not** help the
CarPlay goal — download the release instead.

---

## USB — the path that always works

Plug the iPhone (or Android phone) into the stereo with a good USB cable.

- iPhone → **CarPlay over USB**; Android phone → **Android Auto over USB**. CarLink detects which.
- Accept **"Use CarPlay"** (or Android Auto's prompts) on the phone; keep it unlocked. First
  connection can take up to ~30 seconds.
- **USB CarPlay and USB Android Auto are unaffected** by the Bluetooth-module limitation below.

If CarPlay connects over USB but the screen is black with sound (seen on some iPhones on iOS 27):
**Settings › Display › Direct video output** → *Turn on and reconnect*.

Expected quirk: during USB CarPlay the iPhone may show **"Personal Hotspot"** and Android shows a
VPN key icon. That is just the local cable link CarPlay needs — no internet, no mobile data spent
on the stereo.

---

## Wireless — the honest situation for a ZLink/CarbitLink unit

**CarPlay, wirelessly: almost certainly not available on this stereo.**

Cheap Android stereos (FYT units, **ZLink-type units** — yours) do **not** use Android's Bluetooth
for phones. Their Bluetooth is a **separate module on a serial link, driven by vendor firmware**;
the phone pairs with that module, and Android's own Bluetooth adapter can't reach the phone through
it. Wireless CarPlay begins with an iAP2 Bluetooth handshake over Android's `BluetoothSocket` —
and on these stereos **nothing written there reaches the phone**.

CarLink detects this and refuses before it fails confusingly, showing:

> **"Wireless isn't available on this stereo yet."** This stereo's Bluetooth is a separate module
> run by the stereo's own firmware (common on FYT and ZLink-type units), and apps cannot use it…
> Connect the phone with a USB cable; it works the same.

Re-pairing or toggling Bluetooth does **not** help. **This is the definitive answer to "will
wireless work?" — for CarPlay on a ZLink module, no.** Use USB.

### Wireless Android Auto — one experimental thing to try (Android phones only)

CarLink has **ported** the wireless-handshake routes for these modules from upstream
`open-headunit` 3.5.0 into its Android Auto engine (the "hacker route" from our earlier
discussion — already done by the project, not something you need to build):

- **ZBT** (ZLink-type modules) — talks to the module daemon on `127.0.0.1:3152`.
- **BLINK** (FYT's DUDUAUTO module) — talks over `/dev/auto_serial`.

It's exposed as **Settings › Connection › Show advanced › "Wireless through the stereo's Bluetooth
module (experimental)"** (shown only on such stereos). Caveats, straight from the docs:

- **Off by default, and "not yet verified on a real stereo."**
- **Android Auto only — CarPlay is not covered.** So this does nothing for an iPhone.
- On FYT units you must first disable the stock `com.syu.carlink` app (Android Settings › Apps),
  because it owns the same serial port. (ZLink units use a different route.)

If you have an Android phone, you can try it; if it's an iPhone, skip it and use USB.

### Help get your stereo supported

CarLink's **Diagnostics › Report on GitHub** (or *Share log* →
[open an issue](https://github.com/F-e-n-y-x/CarLink/issues/new/choose)) sends a hardware probe
(module classification, serial-port nodes, system properties, the `127.0.0.1:3152` daemon check,
installed stock apps) that tells the maintainer **which module route your specific stereo needs**.
This is the realistic path to wireless being added for your unit.

---

## Quick decision tree

```
Do you want CarPlay (iPhone)?
├─ Yes → Use the official release APK, connect by USB cable.  ✅ Works.
│        Wireless CarPlay on a ZLink/CarbitLink stereo → not available. Use USB.
│
└─ Android Auto (Android phone)?
   ├─ USB cable            → ✅ Works.
   └─ Wireless (optional)  → Try Settings › Connection › Show advanced →
                             "Wireless through the stereo's Bluetooth module (experimental)".
                             Unverified; FYT units must disable com.syu.carlink first.
```

---

## Requirements recap

- **Display:** Android **7.1+** (API 25) on ARM (`arm64-v8a` / `armeabi-v7a`) — your car stereo.
  Install CarLink **there**, not on your phone.
- **Phone:** iPhone (CarPlay) or Android phone (Android Auto); a USB cable, or Bluetooth + Wi-Fi
  for wireless.
- **Package:** `com.carlink.app`, signed with CarLink's release key (fingerprint above).

---

*This guide summarizes CarLink's published documentation for convenience. CarLink is third-party
software (AGPL-3.0); it is not affiliated with this repository. Always verify the APK you install
as described above.*
