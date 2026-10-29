# ike-scan for OpenWrt routers (aarch64, static)

Prebuilt static `ike-scan` 1.9.5 for aarch64 OpenWrt routers (GL.iNet, etc.).
OpenWrt's opkg repos do not ship ike-scan, so this fills the gap for anyone
who needs to probe IKE servers directly from their router.

## What is ike-scan?

ike-scan is a command-line tool that sends IKE (Internet Key Exchange)
probes to a target and reports what comes back. It speaks both IKEv1 and
IKEv2, and it works by crafting packets and reading responses — no need for
a full IPsec client, no need for the other side to cooperate beyond
responding to IKE.

## What you can use it for

- **Probe any IKE server**: VPN gateways, firewalls, ePDGs, IPsec endpoints.
  Send a probe, see if it responds, what IKE version it speaks, and what
  transforms it offers.
- **Diagnose IPsec connectivity**: Is the server reachable? Does it answer
  IKE at all? Does it reject your proposal (and how)? Where does the
  handshake stall?
- **Fingerprint IKE implementations**: Vendor IDs and backoff patterns
  identify what software the other side runs.
- **WiFi Calling (VoWiFi) diagnosis**: WiFi Calling uses IKEv2 to reach the
  carrier's ePDG. Probing the ePDG tells you whether the IKEv2 path from
  your router is open, even when no phone is around to trigger a call.
  (See the ePDG example below.)

It is not limited to WiFi Calling — any scenario that needs IKE
packet interaction works.

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

## Examples

### Probe any IKEv2 server

```sh
ike-scan --ikev2 -M <target-IP>
```

A `notify` response proves the target is reachable and speaks IKEv2.
`INVALID_KE_PAYLOAD` is expected with default proposals — the point is that
it *responded*. No response at all means the target ignores IKE (or the
path is blocked).

### WiFi Calling ePDG example

T-Mobile US ePDG `epdg.epc.mnc260.mcc310.pub.3gppnetwork.org` → 208.54.39.131:

```sh
ike-scan --ikev2 -M 208.54.39.131
# 208.54.39.131  Notify message 17 (INVALID_KE_PAYLOAD)
# HDR=(CKY-R=..., IKEv2)
# 0 returned handshake; 1 returned notify
```

The `notify` response proves the ePDG is reachable and the IKEv2 path from
your router is open. An IKEv1 probe to the same host returns nothing
(0 handshake, 0 notify) — the ePDG only speaks IKEv2.

## Why static?

OpenWrt uses musl libc, not glibc. A dynamically linked binary built for
Debian/Ubuntu (glibc) will not run on OpenWrt. This binary is statically
linked, so it runs on any aarch64 Linux, musl or glibc.

## Build

See `BUILD.md` for the full cross-compile recipe (Bootlin aarch64 musl
toolchain + hand-written config.h, no autotools needed on the build host).
