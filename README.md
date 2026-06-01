# guacamole-droidspaces-kernel

LineageOS 17.1 (Android 10) kernel for OnePlus 7 Pro (guacamole) with Droidspaces support — container namespaces, TUN/VETH, iptables MASQUERADE, loop devices, devtmpfs.

## What's here

| File | Purpose |
|------|---------|
| `kernel-config-backup` | Kernel `.config` with all Droidspaces flags enabled |
| `kernel-patches.diff` | Compilation fixups for NDK Clang toolchain |
| `build-workflow.yml` | GitHub Actions CI — builds with NDK Clang r416183b |
| `GUIDE.md` | Full build/walkthrough guide |

## Key flags enabled

```
CONFIG_PID_NS=y, CONFIG_IPC_NS=y, CONFIG_UTS_NS=y  (namespaces)
CONFIG_TUN=y, CONFIG_VETH=y                         (tunnels)
CONFIG_BLK_DEV_LOOP=y, CONFIG_DEVTMPFS=y            (loop mounts)
CONFIG_NETFILTER_XT_TARGET_MASQUERADE=y             (tethering)
# CONFIG_ANDROID_PARANOID_NETWORK is not set
```

## Toolchain

**Must use NDK Clang (clang-r416183b) + GCC 4.9** — system Clang 14 and GCC 12 both produce non-booting kernels on msm-4.14.

## Build

```bash
make ARCH=arm64 CC=clang CLANG_TRIPLE=aarch64-linux-gnu- \
  CROSS_COMPILE=aarch64-linux-android- \
  KCFLAGS="-Wno-error -Wno-unknown-warning-option" \
  -j$(nproc)
```

Output: `arch/arm64/boot/Image.gz-dtb`

See `GUIDE.md` for the full process including boot.img repacking and flashing.
