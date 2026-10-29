# Building ike-scan 1.9.5 for aarch64 (static, musl)

The build host had no autotools and no aarch64 cross-compiler in apt,
so this recipe uses a prebuilt toolchain + a hand-written config.h.

## 1. Toolchain

Bootlin aarch64 musl toolchain (64-bit host, works on x86_64 Linux):

```sh
curl -L -o bootlin-aarch64.tar.bz2 \
  https://toolchains.bootlin.com/downloads/releases/toolchains/aarch64/tarballs/aarch64--musl--stable-2023.11-1.tar.bz2
tar --no-same-owner -xjf bootlin-aarch64.tar.bz2
# gcc is at: aarch64--musl--stable-2023.11-1/bin/aarch64-linux-gcc
```

(Do NOT use musl.cc's 32-bit toolchain — it segfaults on 64-bit-only VMs.
Do NOT use ARM's developer site tarballs — direct download URLs 404.)

## 2. Source

```sh
git clone https://github.com/royhills/ike-scan.git ike-scan-src
# or: curl -L -o ike-scan-1.9.5.tar.gz https://github.com/royhills/ike-scan/archive/refs/tags/1.9.5.tar.gz
```

Note: the git archive has no pre-generated `configure` script (it needs
autoreconf, which the build host lacked). So we compile directly, bypassing
autotools.

## 3. config.h

Copy `config.h.aarch64-musl` from this repo into the source dir as `config.h`.
It defines what `configure` would have generated for aarch64 musl Linux:

- HAVE_* headers (inttypes, stdint, arpa/inet, netdb, netinet/in, netinet/tcp,
  sys/socket, sys/time, unistd, getopt, signal, sys/stat, fcntl, regex)
- IKE_UINT64 = unsigned long long, IKE_INT64 = long long
- IKE_UINT64_FORMAT = "%llu", IKE_INT64_FORMAT = "%lld"
- NET_SIZE_T = socklen_t
- PACKAGE_STRING = "ike-scan 1.9.5"
- IKEDATADIR = "/usr/share/ike-scan"
- ATTRIBUTE_UNUSED = __attribute__ ((__unused__))

## 4. Compile (static)

```sh
cd ike-scan-src
aarch64-linux-gcc -DHAVE_CONFIG_H -static -O2 -o ike-scan \
  ike-scan.c error.c isakmp.c wrappers.c utils.c mt19937ar.c md5.c sha1.c
aarch64-linux-strip ike-scan
file ike-scan
# ike-scan: ELF 64-bit LSB executable, ARM aarch64, statically linked, stripped
```

Result: ~222 KB static aarch64 binary. No libpcap needed (ike-scan uses
raw/UDP sockets directly, not pcap).

## 5. Data files

Copy `ike-vendor-ids` and `ike-backoff-patterns` from the source dir to
`/usr/share/ike-scan/` on the router, or ike-scan will warn that it cannot
open the vendor ID patterns file (it still works, just without vendor names).
