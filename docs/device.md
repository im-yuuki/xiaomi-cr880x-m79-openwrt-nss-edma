# Xiaomi CR880x — OpenWrt + NSS bring-up archive

## What this is

Archive of the work needed to bring OpenWrt (with Qualcomm **NSS hardware offload**) up on the **Xiaomi CR880x** family — CR8806 / CR8808 / CR8809 / Redmi AX3000 (M79/M81 mainboard).

The goal: replace the stock MiWiFi firmware with OpenWrt while keeping Qualcomm's official offload path (~900 Mbit/s wired forwarding, Wi-Fi on NSS path) instead of slow-path software routing.

## Hardware summary (sanitized; no MACs, no keys)

| Item | Value |
|---|---|
| Model | CR8806 / CR8808 / CR8809 M79 (CR8808 board tested; unit-dependent) |
| SoC | Qualcomm IPQ5018 (2× Cortex-A53) |
| Wi-Fi | 5 GHz: QCN6122 (ath11k); 2.4 GHz: IPQ5018 (ath11k) |
| RAM | 256 MiB @ 0x40000000 |
| NAND | 128 MiB SPI NAND, **ESMT F50D1G41LB** on the tested unit (page 2048 + 64 OOB) |
| Switch | QCA8337 (wired) |
| U-Boot | U-Boot 2016.01 (MiWiFi, 32-bit), boots aarch64 via TZ monitor |
| Stock firmware | MiWiFi based on OpenWrt 18.06-snapshot (LEDE), kernel 4.4.60 |
| machid | 0x8040000 (IPQ5018 QSDK machid) |
| Console | `serial@78AF000` (BLSP1 uart1), 115200 bps; 120000 bps captures cleaner |

## Critical hardware quirks

1. **SPI NAND ECC on ESMT units = 4-bit only.** The fork's DTS (`ipq5018-cr880x.dtsi`) hardcodes `nand-ecc-strength = <8>`; the QPIC driver then requires 2128 bytes OOB, exceeds the 64-byte spare area of the ESMT F50D1G41LB, and the NAND probe fails with `-22` (`no mtd partitions`). The stock dtb (`config@mp02.1`) carries `nand-ecc-strength = <4>` — the fix used here drops the override entirely and lets the driver pick by OOB size (see submodule branch `cr8808-bringup-full`, or `ipq5018-cr880x.dtsi` there). ESMT OOB 64 → 4-bit; OOB ≥ 80 → 8-bit.
2. **ART partition is matched by lowercase label.** Kernel's `qcom,smem-part` parser lower-cases SMEM partition names, so `find_mtd_index "0:art"` works despite the factory dtb/MIBIB calling it `0:ART`.
3. **Partition table is dual-slot.** `rootfs`/`rootfs_1` (36 MiB each) + `overlay` (31 MiB) + `data` (13.5 MiB). Slot selection via `flag_boot_rootfs` (primary mechanism; `flag_last_success`/`flag_ota_reboot` are also used by the boot loader). Boot chain stays untouched — rollback is flip of one flag in U-Boot.
4. **Boot wait is on.** UART console available at `boot_wait=on`; `uart_en=1`/`ssh_en=1` needed for console right down to U-Boot.
5. **Bootloader can't see the full 128 MB map easily.** `md.l` on unmapped addresses causes data abort (`data abort`), and the board self-resets. Use only documented addresses (RAM interior around `0x4A000000`).

## What works after install

- Boot from NAND (`ubi.mtd=rootfs_1`), kernel 6.18.60 + NSS offload enabled, both radios up (`ath11k`, `wifi_offload=1`).
- HE160 (5 GHz, DFS channel 100) comes up cleanly — CAC completes without radar detection.
- Wired forwarding test: ~550–573 Mbit/s single stream with ~100% CPU (limits are CPU-bound, not NSS offload); expect more headroom with aggregate flows.
- CPU governor reports `schedutil`, real OPPs 1008/1296 MHz; idle usage 137 MB/169 MB with Wi-Fi offload on.
- 5 GHz client on HE160 associates; throughput via 5GHz radio shows expected ath11k-host-path behavior (still CPU-limited without 8.11ax PHY power).
- `/proc/mtd` shows all partitions, `0:art` present, caldata readable.

## Known limitations

- This is a community snapshot, not OpenWrt supported release. Expect regressions on other unit revisions (NAND variant, board FEM `xpa` vs `ipa/epa`).
- NSS Wi-Fi offload path is the biggest CPU saving; wired offload saturates the gigabit conduit in routing tests.
- OpenWrt's connection to the stock dual-boot layout is pragmatic (`flag_boot_rootfs`); not a generic A/B user-facing UX. Treat stock MiWiFi as your recovery net (slot 0).
- `wifi` in this snapshot prefers 5 GHz `radio1`; 2.4 GHz works but is CPU-heavy.
- The board has 256 MB RAM; avoid heavy stacks and prefer NSS offload. The hardware tests used an older build with zram; CI images leave it off by default.

## Where the details live

- [`uboot-notes.md`](./uboot-notes.md) — U-Boot console notes, quirks, safe/unsafe commands
- [`build-flash.md`](./build-flash.md) — build & flash & extroot & verify & rollback
- [`verify.md`](./verify.md) — verification log (what was measured/checked, with outputs)
- `openwrt-nss-edma/` — submodule, actual source patches (`git log` there)
