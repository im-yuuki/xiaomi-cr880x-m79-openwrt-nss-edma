# Build & flash guide — Xiaomi CR880x OpenWrt (NSS offload)

Target: IPQ5018 + QCN6122 board, 256 MB RAM, **ESMT F50D1G41LB SPI NAND** on the tested unit, 128 MB flash. Build toolchain: Linux (≥ 16 GB RAM) in a Docker container (OrbStack/Docker Desktop works; this setup used OrbStack + ubuntu:24.04).

> All offsets are from the MIBIB partition table. Do **not** change flash offsets unless you know exactly what you're doing.

## 0. What you need

- Linux build host with ~20 CPU threads, ≥ 16 GB RAM, ≥ 60 GB disk
- The submodule `openwrt-nss-edma` (this repo's `.gitmodules` pins upstream [kuncy7/openwrt-nss-edma](https://github.com/kuncy7/openwrt-nss-edma) `c3po-tag-8021q`; apply `patches/` per `docs/nand-ecc.md`)
- Working serial console for the board (115200/8N1; 120000 bps captures cleaner)
- TFTP server on the host for the board's U-Boot (or from a Linux machine on the same LAN)
- `git`, `curl`, build prerequisites (below)

## 1. Clone sources

```bash
git submodule update --init         # pinned upstream c3po-tag-8021q
```

The submodule defaults to upstream `c3po-tag-8021q`. Apply `patches/` first (see `docs/nand-ecc.md`) — the ECC fix is the only board-specific kernel patch; everything else (DSA, qca8k fabric, qca-nss, ath11k NSS offload) comes with the submodule.

## 2. Build dependencies (inside the build VM)

```bash
apt-get update && apt-get install -y \
  antlr3 asciidoc autoconf automake bash bc binutils bison \
  build-essential bzip2 cmake fastjar file flex g++ gawk gcc-multilib \
  gettext git g++-multilib help2man intltool libncurses-dev libssl-dev \
  libxml-parser-perl libyaml-dev make patch perl pkg-config python3-dev \
  python3-setuptools python3-yaml rsync swig time unzip wget zstd zlib1g-dev \
  zlib1g-dev ca-certificates
```

Do **not** build as root — build as a non-privileged user (e.g. `useradd -m builder`). Host configure scripts bail on root (`tools/tar` failure: `you should not run configure as root`).

## 3. Feeds + defconfig

```bash
./scripts/feeds update -a
./scripts/feeds install -a
```

Base config for 256 MB devices is `.github/ci/ipq50xx/256m.config` + `common.config`. To build **only** `xiaomi_cr8806` (CR8806/CR8808/CR8809/AX3000T M81), take those two files as starting point, keep only:

```
CONFIG_TARGET_qualcommax_ipq50xx_DEVICE_xiaomi_cr8806=y
```

plus your own package choices (LuCI, wpad-mbedtls, zram-swap — the build used in this archive also selects `luci`, `luci-app-nss`, `luci-app-sqm`, `wpad-mbedtls`, `zram-swap`, `htop`, `ethtool`). Add to the same `.config`:

```
CONFIG_TARGET_qualcommax_ipq50xx_DEVICE_xiaomi_cr8806=y
CONFIG_PACKAGE_MAC80211_NSS_SUPPORT=y
CONFIG_ATH11K_NSS_SUPPORT=y
CONFIG_NSS_DRV_VLAN_ENABLE=y
CONFIG_NSS_DRV_IPV6_ENABLE=y
CONFIG_NSS_DRV_VIRT_IF_ENABLE=y
CONFIG_PACKAGE_nss-tools-dwmac=y
CONFIG_PACKAGE_nss-firmware=y
CONFIG_NSS_FIRMWARE_VERSION_12_5=y
CONFIG_NSS_MEM_PROFILE_LOW=y
CONFIG_ATH11K_MEM_PROFILE_256M=y
CONFIG_PACKAGE_zram-swap=y
CONFIG_PACKAGE_wpad-mbedtls=y
# if you want a reproducible config with the extroot overlay:
```

The overlay in `files/` (`lib/preinit/09_mount_overlay_data`, `etc/config/fstab`) attaches partition `overlay` (mtd20, 31 MiB) as `ubi1`, creates `rootfs_data`, then drops the 1 MiB `rootfs_data` in slot `rootfs_1` — the NAND extroot pivot lands on ~27 MB instead of 10 MB. If you **don't** want extroot, delete those two files from the image (the build will then use the slot's own small `rootfs_data`).

Run:

```bash
make defconfig   # pick up group config updates automatically
make -j$(nproc)
```

Artifacts land in `bin/targets/qualcommax/ipq50xx/`:

```
openwrt-qualcommax-ipq50xx-xiaomi_cr8806-initramfs-uImage.itb
openwrt-qualcommax-ipq50xx-xiaomi_cr8806-squashfs-factory.ubi
openwrt-qualcommax-ipq50xx-xiaomi_cr8806-squashfs-sysupgrade.bin
sha256sums, manifest, config.buildinfo, feeds.buildinfo
```

Before flashing, check the image against `sha256sums` from the same build.

## 4. Flash

### Phase 0 — host prep

- Set your host's Ethernet to an address in the board's subnet (e.g. 192.168.0.100/24) — or use any other TFTP server on the same link.
- Connect board's **WAN** port to host.
- Serve artifacts via TFTP: any server works, e.g. in Docker:

```bash
docker run -d --name tftp -p 69:69/udp -v "$PWD":/srv/tftp:ro ubuntu:24.04 \
  bash -c 'apt update -qq && apt install -y -qq tftpd-hpa >/dev/null && in.tftpd --listen --secure /srv/tftp'
```

### Phase 1 — initramfs boot check (writes nothing)

```
tftpboot 44000000 openwrt-qualcommax-ipq50xx-xiaomi_cr8806-initramfs-uImage.itb
bootm 44000000
```

Watch boot log for: kernel 6.18 loads, `spi-nand` "ESMT SPI NAND was found", 22 SMEM partitions, both `ath11k` probes OK, `mount_root switching to ubifs overlay`. Root shell on console or SSH into 192.168.1.1.

### Phase 2 — wipe runtime user data (optional)

- `overlay` (mtd20) was erased from U-Boot before install; `overlay`/`data`/crash logs in MiWiFi are already wiped in this archive's flow.
- Partitions to leave alone: `0:SBL1`, `0:MIBIB`, `0:QSEE`, `0:APPSBL`, `0:APPSBLENV`, `0:ART`, `bdata`. Do not erase them.

### Phase 3 — write to slot 1

```
tftpboot 44000000 openwrt-qualcommax-ipq50xx-xiaomi_cr8806-squashfs-factory.ubi
nand erase 0x2e80000 0x2400000
nand write 44000000 0x2e80000 0x13c0000   # size from 'Bytes transferred'
```

(Factory image is exactly `0x13C0000` = 20,709,376 bytes for this build's LuCI-heavy image.)

If doing in-place `sysupgrade`: copy `...-sysupgrade.bin` to `/tmp`, `sysupgrade` normally — `flag_boot_rootfs` stays `1`, slot 1 updated, slot 0 untouched.

### Phase 4 — boot slot 1

```
setenv flag_boot_rootfs 1
setenv flag_last_success 1
saveenv
reset
```

Board boots OpenWrt from slot 1 (slot 0 MiWiFi stays untouched for rollback).

## 5. Post-install verification

```bash
cat /proc/mtd               # all partitions present, 0:art (lowercase)
dmesg | grep -iE "nand|ecc|ath11k|nss|qca8k" | head
logread -e nss | tail -5    # "NSS wired plane + ECM up"
mount | grep overlay        # /dev/ubi1_0 on /overlay type ubifs (~27 MB)
df -h                       # /rom ~13.3M + /overlay ~23M free
iw phy | head -80           # both phys list HE160
/etc/init.d/wireless restart ; iw dev
```

Throughput sanity test: `iperf3 -s` on another machine on the WAN side; from the board `iperf3 -c <server>`; expect ~500–900 Mbit/s through NSS offload path (CPU largely idle during forwarding).

## 6. Rollback

If anything goes wrong:

```
setenv flag_boot_rootfs 0
setenv flag_last_success 0
saveenv
reset
```

Board boots MiWiFi from slot 0; no data lost.

## 7. Remarks

- The kernel cmdline is `root=/dev/ubiblock0_1` baked into the FIT; dual-boot is selected by `bootmiwifi` reading `flag_boot_rootfs`. OpenWrt itself does **not** need to know which slot it is in.
- `overlay` (mtd20) is reused as OpenWrt UBIFS; this archive erases it first. If you want MiWiFi-style factory reset, recreate it manually — but it has no value for OpenWrt use.
- `mtd21` (`data`) is the MiWiFi persistent-binding partition; erased/restored via U-Boot `nand erase` / `nand write` from `mtd21-restore.bin` (in `.private/`, not shipped).
