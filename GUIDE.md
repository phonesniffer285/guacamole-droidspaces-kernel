# Custom Kernel Build Guide — OnePlus 7 Pro (guacamole)
## LineageOS 17.1 (Android 10) — Droidspaces + Ethernet Tethering

---

## Overview

Stock LineageOS 17.1 kernel has **all namespace features disabled** (`CONFIG_PID_NS=n`, `CONFIG_IPC_NS=n`, `CONFIG_UTS_NS=n`). Droidspaces requires these to create isolated containers. This guide builds a custom kernel with namespaces, TUN, VETH, and iptables MASQUERADE support.

---

## Files in this directory

| File | Purpose |
|------|---------|
| `Image.gz-dtb` | Compiled kernel binary (NDK Clang 12) |
| `ndk-boot.img` | Pre-built boot.img ready to flash |
| `vbmeta.img` | Stock vbmeta (flash with `--disable-verification`) |
| `kernel-config-backup` | Kernel `.config` with all required flags |
| `build-workflow.yml` | GitHub Actions CI workflow |
| `Magisk-v30.7.zip` | Root installer (needed for namespace access) |

---

## Required kernel config changes

All five critical flags are **already enabled** in `kernel-config-backup`. Stock had these disabled:

```
CONFIG_NAMESPACES=y           # (was y in stock — ok)
CONFIG_UTS_NS=y               # (was n)
CONFIG_IPC_NS=y               # (was n)
CONFIG_PID_NS=y               # (was n)
CONFIG_TUN=y                  # (was y — already ok)
CONFIG_VETH=y                 # (was y — already ok)
```

Additional flags for ethernet tethering:

```
CONFIG_NETFILTER_XT_TARGET_MASQUERADE=y
CONFIG_IP_NF_FILTER=y
CONFIG_NF_NAT_REDIRECT=y
CONFIG_DEVTMPFS=y
CONFIG_DEVTMPFS_MOUNT=y
# CONFIG_ANDROID_PARANOID_NETWORK is not set
```

---

## Toolchain

**CRITICAL:** Must use Android NDK Clang, NOT system clang or GCC 12.

| Toolchain | Result |
|-----------|--------|
| GCC 12 (Ubuntu 22.04) | Compiles, **does not boot** |
| Clang 14 (Ubuntu 22.04) | Compiles, **does not boot** |
| NDK Clang 12.0.5 (r416183b) | **Compiles and boots** |

### Working toolchain setup

```bash
# Clone NDK Clang (clang-r416183b, lineage-20.0 branch)
git clone --depth=1 -b lineage-20.0 \
  https://github.com/LineageOS/android_prebuilts_clang_kernel_linux-x86_clang-r416183b \
  ndk-clang

# Clone NDK GCC 4.9 (for assembler/linker, lineage-19.1 branch)
git clone --depth=1 -b lineage-19.1 \
  https://github.com/LineageOS/android_prebuilts_gcc_linux-x86_aarch64_aarch64-linux-android-4.9 \
  ndk-gcc

export PATH="$PWD/ndk-clang/bin:$PWD/ndk-gcc/bin:$PATH"
```

---

## Build commands

```bash
# Clone kernel source
git clone --depth=1 -b lineage-17.1 \
  https://github.com/LineageOS/android_kernel_oneplus_sm8150 \
  kernel-guacamole
cd kernel-guacamole

# Configure
cp kernel-config-backup .config
make ARCH=arm64 \
  CC=clang CLANG_TRIPLE=aarch64-linux-gnu- \
  CROSS_COMPILE=aarch64-linux-android- \
  olddefconfig

# Build
make ARCH=arm64 \
  CC=clang CLANG_TRIPLE=aarch64-linux-gnu- \
  CROSS_COMPILE=aarch64-linux-android- \
  KCFLAGS="-Wno-error=unknown-warning-option -Wno-format \
           -Wno-return-type -Wno-implicit-int \
           -Wno-unused-but-set-variable -Wno-unused-variable \
           -U__FORTIFY_SOURCE" \
  -j$(nproc)
```

Output: `arch/arm64/boot/Image.gz-dtb`

---

## Kernel compilation patches

These are applied to the source before building:

| Patch | Reason |
|-------|--------|
| `sed -i 's/ opslalib\///' Makefile` | opslalib creates circular deps |
| Stub `game_start_state`, `sla_switch_enable` etc in `net/opsla/op_sla.c` | Undefined symbols at link |
| `ccflags-y += -I\$(src)` in all dirs with `*trace.c` | Trace header include paths |
| `CFLAGS_ipa.o := -U__FORTIFY_SOURCE` in IPA Makefile | FORTIFY_SOURCE mismatch |
| `ccflags-y += -I\$(src)` in camera, GPU, mdss dirs | Missing include paths |
| Nullify `__bad_copy_from/to` in `include/linux/thread_info.h` | FORTIFY_SOURCE compile errors |
| Stub `proc_page_hot_count_operations` in `fs/proc/base.c` | Missing struct |
| Remove `__rticdata` section attribute in `include/linux/init.h` | `.bss.rtic` relocation truncation |
| `KBUILD_CFLAGS += -Wno-error` appended to Makefile | Clang is stricter than GCC |

---

## Creating boot.img

```bash
# Extract stock boot.img components
unpack_bootimg --boot_img=stock-boot.img --out=extracted/

# Repack with custom kernel
mkbootimg --kernel Image.gz-dtb \
  --ramdisk extracted/ramdisk \
  --dtb extracted/dtb \
  --base 0x00000000 \
  --kernel_offset 0x00008000 \
  --ramdisk_offset 0x02000000 \
  --second_offset 0x00f00000 \
  --tags_offset 0x00000100 \
  --dtb_offset 0x00000001f00000 \
  --pagesize 4096 \
  --header_version 2 \
  --os_version 10.0.0 \
  --os_patch_level 2021-02 \
  --cmdline "androidboot.hardware=qcom androidboot.console=ttyMSM0 \
             androidboot.memcg=1 lpm_levels.sleep_disabled=1 \
             video=vfb:640x400,bpp=32,memsize=3072000 \
             msm_rtb.filter=0x237 service_locator.enable=1 \
             swiotlb=2048 firmware_class.path=/vendor/firmware_mnt/image \
             loop.max_part=7 androidboot.usbcontroller=a600000.dwc3 \
             androidboot.vbmeta.avb_version=1.0 buildvariant=userdebug" \
  -o new-boot.img

# Add AVB footer
avbtool add_hash_footer --image new-boot.img \
  --partition_name boot --partition_size 100663296 --algorithm NONE
```

**Important:** `kernel_size` in boot.img header must be the **actual file size** of Image.gz-dtb, NOT padded. Let mkbootimg calculate it.

---

## Flashing

```bash
# Test boot (temporary — no permanent change)
fastboot boot new-boot.img

# If it boots, flash permanently
fastboot flash boot_a new-boot.img
fastboot flash boot_b new-boot.img
fastboot --disable-verity --disable-verification flash vbmeta_a vbmeta.img
fastboot reboot
```

---

## Known issues

- **WiFi/sound may break** if vendor modules don't match kernel vermagic. This is inherent to non-GKI devices. The `wlan.ko` module may need to be replaced.
- **`fastboot boot boot.img`** reliably tests if the kernel boots without permanent changes. Use this first.
- **AnyKernel3** is an alternative install method: package `Image.gz-dtb` in AnyKernel3 zip and flash via recovery. The `anykernel.sh` must have correct device names and block paths for OnePlus 7 Pro.

---

## Key learnings

1. Stock LineageOS 17.1 kernel has all namespace features **disabled** in the actual build config (even though the defconfig source has them enabled — the build system overrides them).
2. GCC 12 and system Clang 14 both produce non-booting kernels for msm-4.14. Only **NDK Clang** (Android prebuilt, r416183b) works.
3. `kernel_size` in boot.img must be the **exact** file size, not zero-padded.
4. NDK GCC 4.9 (`aarch64-linux-android-` prefix) is needed alongside NDK Clang for the assembler and linker.
5. `-Werror` must be neutralized (`KBUILD_CFLAGS += -Wno-error`) because Clang has stricter warnings than GCC.
