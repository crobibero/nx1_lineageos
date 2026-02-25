# LineageOS 22.1 for BLUEFOX NX1

Build instructions for LineageOS 22.1 (Android 15) on the BLUEFOX NX1 (MT6768/MT6769, Helio G85).

## Device Specs

| Item | Value |
|------|-------|
| SoC | MediaTek MT6768 (Helio G85) |
| Architecture | arm64 + arm32 (cortex-a55) |
| Kernel | GKI 6.6.82-android15-8 (prebuilt) |
| Screen | 540×1168 |
| RAM | 4 GB |
| Storage | eMMC, Dynamic Partitions, Virtual A/B |
| Android | 15 (API 35) |

---

## Prerequisites

- Linux build host (Ubuntu 22.04 LTS recommended)
- ~300 GB free disk space
- 16 GB RAM minimum (32 GB recommended)
- Python 3, Git, Curl

### Install build dependencies

```bash
sudo apt update
sudo apt install -y \
  bc bison build-essential ccache curl flex g++-multilib gcc-multilib \
  git gnupg gperf imagemagick lib32ncurses5-dev lib32readline-dev \
  lib32z1-dev liblz4-tool libncurses5-dev libsdl1.2-dev libssl-dev \
  libxml2 libxml2-utils lzop pngcrush rsync schedtool squashfs-tools \
  xsltproc zip zlib1g-dev python3 python-is-python3 openjdk-11-jdk
```

### Install `repo`

```bash
mkdir -p ~/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

---

## 1. Initialize the LineageOS source

```bash
mkdir -p ~/android/lineage
cd ~/android/lineage
repo init \
  -u https://github.com/LineageOS/android.git \
  -b lineage-22.1 \
  --git-lfs \
  --depth=1
```

---

## 2. Add the local manifest

```bash
mkdir -p .repo/local_manifests
curl -o .repo/local_manifests/roomservice.xml \
  https://raw.githubusercontent.com/crobibero/android_manifest_bluefox_nx1/master/roomservice.xml
```

Or clone this repo and copy the file:

```bash
git clone https://github.com/crobibero/android_manifest_bluefox_nx1.git
cp android_manifest_bluefox_nx1/roomservice.xml .repo/local_manifests/roomservice.xml
```

---

## 3. Sync sources

```bash
cd ~/android/lineage
repo sync -c -j$(nproc) --no-tags --no-clone-bundle --force-sync
```

This downloads ~50 GB and may take several hours.

---

## 4. Configure ccache (optional but recommended)

```bash
export USE_CCACHE=1
export CCACHE_EXEC=$(which ccache)
ccache -M 50G
```

Add both export lines to `~/.bashrc` to persist across sessions.

---

## 5. Build

```bash
cd ~/android/lineage
source build/envsetup.sh
breakfast nx1
brunch nx1
```

Output images will be in:

```
out/target/product/nx1/
```

Key images to flash:

| Image | Partition |
|-------|-----------|
| `boot.img` | boot_a / boot_b |
| `init_boot.img` | init_boot_a / init_boot_b |
| `vendor_boot.img` | vendor_boot_a / vendor_boot_b |
| `dtbo.img` | dtbo_a / dtbo_b |
| `super.img` (or sparse images) | super |
| `vbmeta.img` | vbmeta_a / vbmeta_b |
| `vbmeta_system.img` | vbmeta_system_a / vbmeta_system_b |
| `vbmeta_vendor.img` | vbmeta_vendor_a / vbmeta_vendor_b |

---

## 6. Flash

### Via SP Flash Tool (Windows / Linux)

1. Use SP Flash Tool with the MT6768 scatter file.
2. Select **Download Only** mode.
3. Flash the output images to their respective partitions.

### Via fastboot (if device is already unlocked)

```bash
# Reboot to fastboot
adb reboot bootloader

# Flash A slot (repeat with _b suffix for B slot if needed)
fastboot flash boot_a boot.img
fastboot flash init_boot_a init_boot.img
fastboot flash vendor_boot_a vendor_boot.img
fastboot flash dtbo_a dtbo.img

# Flash super (logical partitions)
fastboot wipe-super super_empty.img
# then sideload or flash sparse system/vendor/product/etc images

# Flash vbmeta images (--disable-verity for bringup testing)
fastboot flash vbmeta_a vbmeta.img
fastboot flash vbmeta_system_a vbmeta_system.img
fastboot flash vbmeta_vendor_a vbmeta_vendor.img

fastboot reboot
```

---

## Repos

| Purpose | Repository |
|---------|-----------|
| Device tree | [crobibero/android_device_bluefox_nx1](https://github.com/crobibero/android_device_bluefox_nx1) |
| Vendor blobs | [crobibero/android_vendor_bluefox_nx1](https://github.com/crobibero/android_vendor_bluefox_nx1) |
| MTK IMS / VoLTE | [techyminati/android_vendor_mediatek_ims](https://github.com/techyminati/android_vendor_mediatek_ims) |
| MediaTek HALs | [LineageOS/android_hardware_mediatek](https://github.com/LineageOS/android_hardware_mediatek) |

---

## Notes

- SELinux is set to **permissive** during initial bringup. Do not use permissive builds as daily drivers.
- The kernel is a prebuilt GKI image; kernel source is not required for builds.
- VoLTE/IMS support requires `vendor/mediatek/ims` (included via the local manifest).
- The `hardware/mediatek` branch used is `lineage-22.1`.
