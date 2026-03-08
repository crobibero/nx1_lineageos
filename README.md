# LineageOS 23.2 for BLUEFOX NX1

Build instructions for LineageOS 23.2 (Android 15) on the BLUEFOX NX1 (MT6768/MT6769, Helio G85).

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
  -b lineage-23.2 \
  --git-lfs \
  --depth=1
```

---

## 2. Add the local manifest

```bash
mkdir -p .repo/local_manifests
curl -o .repo/local_manifests/roomservice.xml \
  https://raw.githubusercontent.com/crobibero/nx1_lineageos/master/roomservice.xml
```

Or clone this repo and copy the file:

```bash
git clone https://github.com/crobibero/nx1_lineageos.git
cp nx1_lineageos/roomservice.xml .repo/local_manifests/roomservice.xml
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
- The `hardware/mediatek` branch used is `lineage-23.2`.

# BlueFox NX1 – LineageOS Flashing Instructions
==============================================

The NX1 uses **Virtual A/B (VAB)** partitioning with recovery embedded in
`vendor_boot`. All images are built to `out/target/product/nx1/`.

Prerequisites
-------------

- ADB and fastboot installed on your PC
- USB cable
- OEM unlocking enabled in Developer Options (Settings → Developer Options →
  OEM unlocking)

First-Time Install (via fastboot)
----------------------------------

### 1. Unlock the bootloader

```bash
adb reboot bootloader
fastboot flashing unlock
```

Confirm on-device when prompted. **This wipes the device.**

### 2. Flash the boot images

```bash
fastboot flash boot boot.img
fastboot flash init_boot init_boot.img
fastboot flash vendor_boot vendor_boot.img
fastboot flash dtbo dtbo.img
fastboot flash vbmeta vbmeta.img --disable-verity --disable-verification
fastboot flash vbmeta_system vbmeta_system.img --disable-verity --disable-verification
fastboot flash vbmeta_vendor vbmeta_vendor.img --disable-verity --disable-verification
```

### 3. Boot into recovery

```bash
fastboot reboot recovery
```

Or use the hardware key combo (power + vol-up) if the above does not work.

### 4. Wipe data

In LineageOS Recovery: **Factory Reset → Format data/factory reset** → confirm.

### 5. Sideload the OTA zip

In LineageOS Recovery: **Apply Update → Apply from ADB**, then on your PC:

```bash
adb sideload lineage-23.2-20260227-UNOFFICIAL-nx1.zip
```

### 6. Reboot

Select **Reboot System** in recovery, or run:

```bash
adb reboot
```

Subsequent Updates (OTA sideload)
----------------------------------

No wipe is needed for updates.

```bash
adb reboot recovery
# In recovery: Apply Update → Apply from ADB
adb sideload lineage-23.2-20260227-UNOFFICIAL-nx1.zip
```

Notes
-----

- Do **not** flash `system.img`, `vendor.img`, etc. individually via fastboot —
  this is a dynamic partitions device and those go through the OTA mechanism.
- To wipe the super partition (e.g. to recover from a bad flash):
  ```bash
  fastboot wipe-super super_empty.img
  ```
- If the device does not support standard `fastboot flashing unlock` (some
  MT6768 devices use a vendor-specific unlock flow), check whether BlueFox
  provides an unlock tool, or use SP Flash Tool to flash a preloader that
  enables fastboot mode.
