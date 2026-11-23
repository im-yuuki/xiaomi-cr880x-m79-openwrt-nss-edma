# cr880x-openwrt

Running **OpenWrt snapshot (kernel 6.18.60)** on the **Xiaomi CR880x** family — **CR8806 / CR8808 / CR8809 / Redmi AX3000 M81** — Qualcomm IPQ5018 + QCN6122, 256 MB RAM, 128 MB NAND (ESMT SPI NAND on the tested CR8808 unit).

This repository archives the work done to get OpenWrt (with **Qualcomm NSS hardware offload**) running on that board:

- **`docs/`** — device profile, U-Boot/console notes, build & flash guide, verification results.
- **`openwrt-nss-edma/`** — git submodule tracking **`kuncy7/openwrt-nss-edma` @ `c3po-tag-8021q`**. The archive carries the vendor patches separately: `patches/` (the one board-specific commit + the `files/` reproducible-build overlay), `overlay/` (the same files, pre-copied), and `docs/nand-ecc.md`.

The base platform is [`kuncy7/openwrt-nss-edma`](https://github.com/kuncy7/openwrt-nss-edma) branch `c3po-tag-8021q` — the community Qualcomm NSS EDMA effort for IPQ5018 (upstream Linux 6.18, `stmmac` driver, `kmod-qca-dwmac-nss`). Neither this repo nor the fork redistribute Xiaomi proprietary blobs: everything is in-tree open-source or built by the vendor build system.

## Status

Working snapshot — boots from NAND, both radios (2.4 GHz IPQ5018 + 5 GHz QCN6122, 802.11ax/HE160 on DFS channel 100 verified), NSS wired-plane offload (`fw_mask=0x3`, ECM accelerating), ath11k Wi-Fi offload, LuCI present. Not a supported OpenWrt release; a community build for this hardware, kept as an archive of the work.

## NSS kmod feed (ipq50xx-2026.11.11/256m)

Upstream's official image ships a build-specific `public-key.pem` in `/etc/apk/keys/` that verifies that release's kmod feed. See [`docs/nss-feed.md`](docs/nss-feed.md) for the `ipq50xx-2026.11.11/256m` public key and setup instructions.

## Layout

```
.github/
  workflows/build-cr880x.yml — scheduled / manual firmware + kmod-feed build
  ci/cr8806.config           — firmware and optional APK package selection
docs/
  device.md        — hardware profile (summary, sanitized)
  uboot-notes.md   — U-Boot console notes + safety rules
  build-flash.md   — build from source, install, extroot, verify, rollback
  verify.md        — what was verified on hardware
openwrt-nss-edma/  — submodule (upstream c3po-tag-8021q)
.private/          — NOT in git: dumps, nvram, logs, local notes
```

## Automated builds

`.github/workflows/build-cr880x.yml` runs on manual dispatch
(`Actions → Build CR880x firmware + kmod feed → Run workflow`). The 14-day
schedule (`cron: 0 0 */14 * *`, i.e. days 1, 15, 29 UTC) is currently commented
out in the workflow — uncomment the two lines under `on:` to enable it. Note
that GitHub's day-of-month cron is not a true 14-day interval across a month
boundary; there is no gate, every scheduled trigger builds.

The workflow:

1. pulls `openwrt-nss-edma` to the newest commit on upstream `c3po-tag-8021q`
   (not the pinned gitlink), then applies `patches/`;
2. builds the `xiaomi_cr8806` image with `.github/ci/cr8806.config`;
3. publishes the images **and the build's signed APK repository** (kernel
   modules and userspace packages) to GitHub Pages under
   `gh-pages`, one `ipq50xx-YYYY.MM.DD-N/` directory per build (newest 3 kept;
   a rebuild of the same run gets an `-aN` suffix).

After a successful build on the default branch, CI advances the repository's
submodule pointer to the exact upstream commit used to build the firmware.
It never rolls the pointer back or overwrites a concurrent push.

Check **Publish to GitHub Releases** in the manual dispatch form to also
publish a prerelease for that build after the Pages job succeeds. The
checkbox is off by default. The prerelease includes the firmware,
checksums, build metadata, and an archive of the matching signed APK feed.
The archive remains available even after Pages removes the older build.

The `gh-pages` branch already exists (seeded with `.nojekyll` and an empty
`index.md`). Enable Pages in the repo settings: **Settings → Pages →
Build and deployment → Source: Deploy from a branch → `gh-pages` / root**.
The image's `distfeeds.list` is generated during CI and already points at that
build's feed, so `apk update && apk add kmod-wireguard` works on a matching
image. The feed is signed with a build-local throwaway key; its public half is
embedded in the image and published as `public-key.pem` — the private key never
leaves the runner.

CI images include LuCI and NSS offload, but leave ZRAM, SQM and extra diagnostics
out of the default install. After `apk update`, install only what you need:

```sh
apk add zram-swap                            # ZRAM; pulls in the matching kmod
apk add luci-app-sqm sqm-scripts-nss tc-full  # SQM with the NSS shaper
apk add nssinfo                             # NSS statistics
apk add htop iperf3 bind-dig dumpimage       # OpenWrt's snapshot feeds
```

The build stays in the GitHub-hosted runner workspace: `/mnt` is not
guaranteed to be a separate disk. The job logs actual free space and only
cleans unused preinstalled tools when the workspace starts below 40 GiB.

GitHub Actions Cache saves `dl/` after downloads and `.ccache/` (up to 4 GB)
after compilation. New upstream or feed revisions can reuse the last source
cache and a compatible compiler cache; the firmware and APKs are rebuilt.
The build log reports compiler cache hits and misses. For builds spaced
14 days apart, set **Settings → Actions → General → Cache settings →
Cache retention** to 21 days or more if available. The default is 7 days,
so infrequent builds may start cold; cache misses do not block the build.

## Quick start (pointers only; details in docs/build-flash.md)

1. `git submodule update --init
`
2. Build in a Linux VM ≥ 16 GB RAM (e.g. OrbStack/Docker on macOS). See `docs/build-flash.md`.
3. Flash following `docs/build-flash.md` via U-Boot console; keep MiWiFi in **slot 0** as fallback — do not erase it unless you accept losing it.
4. After flash: expect `mount` showing `overlay /dev/ubi1_0` and `df -h` ≈ `/rom` 13 MB + `/overlay` ~27 MB free (see `docs/verify.md`).

## Working notes

- The project does **not** ship binaries of board dumps; see `.gitignore` (`.private/`).
- Commit convention: Conventional Commits.
- Device dumps/secrets never belong in git.
