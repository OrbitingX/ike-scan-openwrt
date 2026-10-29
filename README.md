# ike-scan for OpenWrt routers (aarch64, static)

Prebuilt static `ike-scan` 1.9.5 for aarch64 OpenWrt routers (GL.iNet, etc.).
OpenWrt's opkg repos do not ship ike-scan, so this is a manual build for
routers that need IKE probing (WiFi Calling / VoWiFi diagnosis, IPsec troubleshooting).

## Quick install on your router (no manual transfer needed)

Download the release zip directly on the router and install:

```sh
opkg update && opkg install unzip
cd /tmp
wget https://github.com/OrbitingX/ike-scan-openwrt/releases/download/v1.9.5/ike-scan-openwrt-aarch64-v1.9.5.zip
unzip -o ike-scan-openwrt-aarch64-v1.9.5.zip -d ike-scan-pkg
cp ike-scan-pkg/ike-scan /usr/bin/ike-scan
chmod +x /usr/bin/ike-scan
mkdir -p /usr/share/ike-scan
cp ike-scan-pkg/ike-vendor-ids ike-scan-pkg/ike-backoff-patterns /usr/share/ike-scan/
ike-scan --version
# should print: ike-scan 1.9.5
```

No scp, no USB, no manual file transfer — the router pulls it straight
from GitHub Releases. (If your router has no `unzip`, the `opkg install`
line above adds it. If you prefer curl over wget, swap the download line.)

## Files in this repo

- `ike-vendor-ids` — vendor ID patterns file (goes to `/usr/share/ike-scan/`).
- `ike-backoff-patterns` — backoff patterns file (goes to `/usr/share/ike-scan/`).
- `config.h.aarch64-musl` — the manual config.h used for the cross-compile (no autotools on the build host).
- `BUILD.md` — how this binary was built, so you can rebuild it yourself.

The binary itself (`ike-scan-aarch64-static`, 222 KB) is attached to the
[Releases](https://github.com/OrbitingX/ike-scan-openwrt/releases) as
`ike-scan-openwrt-aarch64-v1.9.5.zip` (binary + data files in one zip).
GitHub's web uploader rejects extensionless binaries, so the zip is the
distribution format — do not try to upload the bare binary.

## Use (WiFi Calling ePDG probe)

WiFi Calling uses IKEv2, not IKEv1. Probe the ePDG with IKEv2:

```sh
ike-scan --ikev2 -M <ePDG-IP>
```

Example (T-Mobile US ePDG `epdg.epc.mnc260.mcc310.pub.3gppnetwork.org` → 208.54.39.131):

```sh
ike-scan --ikev2 -M 208.54.39.131
# 208.54.39.131  Notify message 17 (INVALID_KE_PAYLOAD)
# HDR=(CKY-R=..., IKEv2)
# 0 returned handshake; 1 returned notify
```

A `notify` response proves the ePDG is reachable and speaks IKEv2.
`INVALID_KE_PAYLOAD` is expected with default proposals — the point is that
it *responded*. No response at all (IKEv1 probe) means IKEv1 is ignored.

## Why static?

OpenWrt uses musl libc, not glibc. A dynamically linked binary built for
Debian/Ubuntu (glibc) will not run on OpenWrt. This binary is statically
linked, so it runs on any aarch64 Linux, musl or glibc.

## Build

See `BUILD.md` for the full cross-compile recipe (Bootlin aarch64 musl
toolchain + hand-written config.h, no autotools needed on the build host).
