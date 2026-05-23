# Porting R960-ReSukiSU to Other Galaxy Watches

This guide is for people who want to fork this project and adapt it to another Samsung Galaxy Watch model, such as a Galaxy Watch4, Watch5, Watch6, or Watch7 variant.

The original project was built for:

| Item | SM-R960 reference value |
|---|---|
| Device | Samsung Galaxy Watch6 Classic |
| Model | `SM-R960` |
| Codename | `wise6bl` |
| Product | `wise6blue` |
| Firmware | `R960XXU2CZB6` |
| Android | `16` |
| Kernel | `5.15.180-33003255-abR960XXU2CZB6` |
| Platform | `s5e5515` / `erd5515` |
| Root solution | ReSukiSU |
| Bootloader | Unlocked required |

> Do not flash SM-R960 builds on another watch. Every Galaxy Watch model has different firmware, boot images, partition sizes, kernel source, defconfig, rollback index, and AVB metadata.

---

## What this repo does

The SM-R960 workflow:

1. Downloads Samsung Open Source kernel tarballs from a GitHub Release.
2. Downloads stock firmware `boot.img`, `init_boot.img`, and `vbmeta.img` from a Bitfrost or however you can get your stock firmware.
3. Extracts Samsung kernel source.
4. Applies ReSukiSU.
5. Applies Samsung/ReSukiSU manual hook patches.
6. Disables Samsung custom-kernel blockers where present.
7. Builds the custom kernel `Image`.
8. Replaces the stock `boot.img` kernel with the custom `Image`.
9. Re-footers `boot.img` and `init_boot.img`.
10. Generates a new top-level `vbmeta.img`.
11. Creates a flashable AP tar containing only:

```text
boot.img
init_boot.img
vbmeta.img
```

12. Publishes the AP tar and build artifacts to GitHub Releases.

---

# 1. Requirements

You need:

- An unlocked bootloader.
- Stock firmware for your exact watch build.
- Samsung Open Source kernel package for your watch model/build, or the closest available matching release.
- A working recovery method with Odin, NetOdin, Brokkr, or your preferred Samsung flashing tool.
- GitHub account.
- GitHub Actions enabled.
- ReSukiSU Manager APK compatible with your watch userspace ABI, usually ARM/ARMv7 on older Wear OS builds.

---

# 2. Collect target watch information

Run this from the target watch shell:

```sh
echo "===== BASIC KERNEL INFO ====="
uname -a
uname -r
cat /proc/version 2>/dev/null

echo
echo "===== DEVICE / BUILD PROPS ====="
getprop ro.product.model
getprop ro.product.device
getprop ro.product.name
getprop ro.product.vendor.device
getprop ro.bootloader
getprop ro.build.PDA
getprop ro.build.version.incremental
getprop ro.build.fingerprint
getprop ro.csc.sales_code
getprop ro.build.version.release
getprop ro.build.version.sdk

echo
echo "===== BOOTLOADER / AVB STATE ====="
getprop ro.boot.verifiedbootstate
getprop ro.boot.vbmeta.device_state
getprop ro.boot.flash.locked
getprop ro.boot.warranty_bit
getprop ro.boot.vbmeta.avb_version
getprop ro.boot.vbmeta.digest
getprop ro.boot.vbmeta.hash_alg
getprop ro.boot.veritymode

echo
echo "===== HARDWARE ====="
getprop ro.boot.hardware
getprop ro.hardware
getprop ro.board.platform
getprop ro.product.board

echo
echo "===== BLOCK PARTITIONS ====="
ls -l /dev/block/by-name 2>/dev/null | grep -Ei "boot|init_boot|vendor_boot|dtbo|vbmeta|recovery|kernel"

echo
echo "===== PROC PARTITIONS ====="
cat /proc/partitions 2>/dev/null | grep -Ei "boot|dtbo|vbmeta|mmc|sda|sdb|dm-"

echo
echo "===== KERNEL CONFIG ====="
ls -l /proc/config.gz 2>/dev/null
zcat /proc/config.gz 2>/dev/null | grep -Ei "CONFIG_LOCALVERSION|CONFIG_IKCONFIG|CONFIG_GKI|CONFIG_KPROBES|CONFIG_MODULES|CONFIG_MODULE_SIG|CONFIG_LTO|CONFIG_CC_IS_CLANG|CONFIG_ARM64|CONFIG_ANDROID|CONFIG_SECURITY_SELINUX|CONFIG_OVERLAY_FS|CONFIG_KALLSYMS|CONFIG_KALLSYMS_ALL"
```

Save this output. You will use it to edit the workflow.

---

# 3. Values you must change for another watch

Do not reuse the SM-R960 values unless your target watch really matches them.

| Value | What it means | SM-R960 example |
|---|---|---|
| `MODEL` | Watch model | `SM-R960` |
| `DEVICE` | Device codename | `wise6bl` |
| `PRODUCT` | Product name | `wise6blue` |
| `FIRMWARE` | Firmware build | `R960XXU2CZB6` |
| `ANDROID_VERSION` | Android version | `16` |
| `SECURITY_PATCH` | Boot security patch prop | `2026-02-05` |
| `KERNEL_VERSION` | Kernel version | `5.15.180` |
| `LOCALVERSION` | Kernel suffix from `uname -r` | `-33003255-abR960XXU2CZB6` |
| `TARGET_SOC` | Platform/SOC | `s5e5515` |
| `DEFCONFIG` | Kernel defconfig | `s5e5515-wise6blue_defconfig` |
| `BOOT_PARTITION_SIZE` | boot partition size in bytes | `50331648` |
| `INIT_BOOT_PARTITION_SIZE` | init_boot partition size in bytes | `16777216` |
| `ROLLBACK_INDEX` | AVB rollback index | `2` |

---

# 4. Calculate partition sizes

From `/proc/partitions`, sizes are shown in KiB.

Example:

```text
259        9      49152 mmcblk0p17
```

Convert KiB to bytes:

```text
49152 * 1024 = 50331648
```

So the boot partition size is:

```text
50331648
```

Use your watch's values, not the SM-R960 values.

---

# 5. Fork the repo

1. Fork the repo.
2. Rename it for your watch if wanted.

Examples:

```text
R860-ReSukiSU
R870-ReSukiSU
R890-ReSukiSU
R920-ReSukiSU
R930-ReSukiSU
R960-ReSukiSU
```

3. Update the README and workflow names.
4. Keep workflow release commands using:

```yaml
--repo "${{ github.repository }}"
```

That lets the workflow keep working after repo renames.

---

# 6. Upload Samsung kernel source tarballs

Create a GitHub Release named:

```text
source-tarballs
```

Upload the Samsung Open Source kernel tarballs for your watch.

Example assets:

```text
Kernel.tar.gz
YOUR_MODEL_KERNEL.tar.gz
```

If your files have different names, update the workflow to match.

## GitHub CLI example

```sh
REPO="YourGitHubName/YourRepoName"

gh release create source-tarballs   --repo "$REPO"   --title "Samsung Open Source kernel tarballs"   --notes "Samsung Open Source kernel tarballs for this watch."

gh release upload source-tarballs   Kernel.tar.gz   YOUR_MODEL_KERNEL.tar.gz   --repo "$REPO"   --clobber
```

---

# 7. Upload stock firmware images

Create another GitHub Release for stock firmware images.

Recommended tag format:

```text
firmware-yourmodel-yourbuild
```

Example:

```text
firmware-r960-czb6
```

Upload:

```text
boot.img or boot.img.lz4
init_boot.img or init_boot.img.lz4
vbmeta.img or vbmeta.img.lz4
```

## GitHub CLI example

```sh
REPO="YourGitHubName/YourRepoName"

gh release create firmware-yourmodel-yourbuild   --repo "$REPO"   --title "Stock firmware boot images"   --notes "Stock boot/init_boot/vbmeta images for target firmware."

gh release upload firmware-yourmodel-yourbuild   boot.img.lz4   init_boot.img.lz4   vbmeta.img.lz4   --repo "$REPO"   --clobber
```

---

# 8. Edit the workflow for your watch

Edit:

```text
.github/workflows/build-*.yml
```

## Update workflow name

Example:

```yaml
name: Build SM-R860 ReSukiSU AP
```

## Update release input defaults

```yaml
source_release_tag:
  default: "source-tarballs"

firmware_release_tag:
  default: "firmware-yourmodel-yourbuild"
```

## Update source tarball filenames

Original SM-R960 style:

```sh
test -f source/Kernel.tar.gz
test -f source/R960XXU1CYK2_Kernel.tar.gz
```

Change to your uploaded filenames:

```sh
test -f source/Kernel.tar.gz
test -f source/YOUR_MODEL_KERNEL.tar.gz
```

And extraction:

```sh
tar -xzf source/Kernel.tar.gz -C work/kernel
tar -xzf source/YOUR_MODEL_KERNEL.tar.gz -C work/kernel --strip-components=1
```

## Update defconfig

Find defconfigs:

```sh
find work/kernel/arch/arm64/configs -type f | sort
```

Original SM-R960:

```sh
make O=out_resukisu s5e5515-wise6blue_defconfig
```

Replace with the correct defconfig for your watch:

```sh
make O=out_resukisu YOUR_DEFCONFIG
```

## Update platform variables

Original:

```sh
export TARGET_SOC=s5e5515
export ARCH=arm64
export LLVM=1
export LLVM_IAS=1
```

Update `TARGET_SOC` if your Samsung source README or build scripts require a different value.

## Update LOCALVERSION

Get this from:

```sh
uname -r
```

Example:

```text
5.15.180-33003255-abR960XXU2CZB6
```

The local version part is:

```text
-33003255-abR960XXU2CZB6
```

Workflow line:

```sh
./scripts/config --file out_resukisu/.config --set-str LOCALVERSION "YOUR_LOCALVERSION" || true
./scripts/config --file out_resukisu/.config -d LOCALVERSION_AUTO || true
```

## Update Android version and security patch

Original:

```sh
--prop com.android.build.boot.os_version:16
--prop com.android.build.boot.security_patch:2026-02-05
```

Change to your firmware values.

## Update rollback index

Original SM-R960 used:

```sh
--rollback_index 2
```

Verify your rollback index using stock AVB info:

```sh
python3 avbtool.py info_image --image boot.img
python3 avbtool.py info_image --image init_boot.img
python3 avbtool.py info_image --image vbmeta.img
```

Do not lower rollback index. A lower value can cause:

```text
SW REV CHECK FAIL
```

## Update partition sizes

Original:

```sh
--partition_size 50331648   # boot
--partition_size 16777216   # init_boot
```

Replace with your target watch partition sizes.

## Update output names

Example:

```text
R860_ReSukiSU_AP.tar
R860_ReSukiSU_AP.tar.md5
Image-ReSukiSU-R860
```

---

# 9. ReSukiSU integration notes

The workflow uses:

```text
https://raw.githubusercontent.com/ReSukiSU/ReSukiSU/main/kernel/setup.sh
```

The build should enable at least:

```text
CONFIG_KSU=y
CONFIG_KSU_MANUAL_HOOK=y
```

The SM-R960 build also used:

```text
CONFIG_KSU_MULTI_MANAGER_SUPPORT=y
```

After boot, check:

```sh
zcat /proc/config.gz | grep -Ei "CONFIG_KSU|CONFIG_KSU_MANUAL_HOOK|CONFIG_KSU_MULTI_MANAGER_SUPPORT"
```

---

# 10. Manual hook notes

For Samsung non-GKI kernels, ReSukiSU may need manual hooks in:

```text
fs/stat.c
fs/exec.c
fs/open.c
kernel/reboot.c
```

The SM-R960 workflow kept the required hooks and removed a bad source-specific `fstat64_ret` injection that caused:

```text
error: use of undeclared identifier 'fd'
ksu_handle_fstat64_ret(&fd, &statbuf);
```

When porting to another watch:

1. Keep required hooks.
2. Avoid optional hooks until the required-hook build works.
3. Capture full compile logs.
4. Fix one compile error at a time.

---

# 11. Samsung custom-kernel blockers

The workflow attempts to disable these if present:

```text
CONFIG_MODULE_SIG_PROTECT
CONFIG_UH
CONFIG_RKP
CONFIG_KDP
CONFIG_SECURITY_DEFEX
CONFIG_SVSMC
CONFIG_SAMSUNG_PRODUCT_SHIP
```

Not every kernel has every symbol. The workflow should not fail just because a symbol does not exist, but it should fail if one remains enabled when it should be disabled.

---

# 12. Run GitHub Actions

Run the workflow manually with inputs like:

```text
resukisu_setup_url = https://raw.githubusercontent.com/ReSukiSU/ReSukiSU/main/kernel/setup.sh
source_release_tag = source-tarballs
firmware_release_tag = firmware-yourmodel-yourbuild
```

Expected release outputs:

```text
YOURMODEL_ReSukiSU_AP.tar
YOURMODEL_ReSukiSU_AP.tar.md5
Image-ReSukiSU-YOURMODEL
resukisu.config
SHA256SUMS.txt
RELEASE_NOTES.txt
```

---

# 13. Flashing notes

The test AP tar should contain only:

```text
boot.img
init_boot.img
vbmeta.img
```

Recommended first flash file:

```text
YOURMODEL_ReSukiSU_AP.tar
```

Use `.tar.md5` only if your flashing tool requires it.

If the watch shows:

```text
SW REV CHECK FAIL
```

check rollback index and firmware binary revision.

If the watch shows a vbmeta verification failure, inspect:

```sh
python3 avbtool.py info_image --image boot.img
python3 avbtool.py info_image --image init_boot.img
python3 avbtool.py info_image --image vbmeta.img
```

---

# 14. Verify root after boot

Install an ARM/ARMv7-compatible ReSukiSU Manager.

Check kernel config:

```sh
zcat /proc/config.gz 2>/dev/null | grep -Ei "CONFIG_KSU|CONFIG_KSU_MANUAL_HOOK|CONFIG_KSU_MULTI_MANAGER_SUPPORT"
```

If Manager shows:

```text
Working
Built-in
```

the kernel interface is working.

---

# 15. Root shell commands

Normal `su` may not exist. Use `libksud.so`.

## Non-spoofed manager

Use when package is:

```text
com.resukisu.resukisu
```

```sh
PKG=com.resukisu.resukisu
LIB="$(dumpsys package "$PKG" | sed -n 's/.*legacyNativeLibraryDir=//p' | head -1)"
KSUD="$LIB/arm/libksud.so"

echo "KSUD=$KSUD"
ls -l "$KSUD"

"$KSUD" debug su -g
```

## Spoofed manager

Use when package name is changed or unknown:

```sh
KSUD=""

for PKG in $(pm list packages | cut -d: -f2); do
  LIB="$(dumpsys package "$PKG" 2>/dev/null | sed -n 's/.*legacyNativeLibraryDir=//p' | head -1)"
  [ -z "$LIB" ] && LIB="$(dumpsys package "$PKG" 2>/dev/null | sed -n 's/.*nativeLibraryDir=//p' | head -1)"

  for CAND in "$LIB/arm/libksud.so" "$LIB/arm64/libksud.so" "$LIB/libksud.so"; do
    if [ -x "$CAND" ]; then
      KSUD="$CAND"
      break 2
    fi
  done
done

echo "KSUD=$KSUD"
ls -l "$KSUD"

"$KSUD" debug su -g
```

If root works, prompt changes to:

```text
wise6bl:/ #
```

Test:

```sh
id
whoami
getenforce
```

---

# 16. Installing modules

Push a module ZIP:

```sh
adb push ModuleName.zip /sdcard/Download/ModuleName.zip
```

Enter root using the spoofed manager auto-detect method:

```sh
KSUD=""

for PKG in $(pm list packages | cut -d: -f2); do
  LIB="$(dumpsys package "$PKG" 2>/dev/null | sed -n 's/.*legacyNativeLibraryDir=//p' | head -1)"
  [ -z "$LIB" ] && LIB="$(dumpsys package "$PKG" 2>/dev/null | sed -n 's/.*nativeLibraryDir=//p' | head -1)"

  for CAND in "$LIB/arm/libksud.so" "$LIB/arm64/libksud.so" "$LIB/libksud.so"; do
    if [ -x "$CAND" ]; then
      KSUD="$CAND"
      break 2
    fi
  done
done

"$KSUD" debug su -g
```

Install module from root shell:

```sh
MODZIP=/sdcard/Download/ModuleName.zip

id
"$KSUD" module install "$MODZIP"
"$KSUD" module list
sync
reboot
```

---

# 17. Module commands

List modules:

```sh
"$KSUD" module list
```

Install module:

```sh
"$KSUD" module install /sdcard/Download/ModuleName.zip
```

Disable module:

```sh
"$KSUD" module disable MODULE_ID
```

Enable module:

```sh
"$KSUD" module enable MODULE_ID
```

Uninstall module:

```sh
"$KSUD" module uninstall MODULE_ID
```

Run module action:

```sh
"$KSUD" module action MODULE_ID
```

Trigger service stages without reboot:

```sh
"$KSUD" post-fs-data
"$KSUD" services
"$KSUD" boot-completed
```

Many modules still require reboot to fully apply.

---

# 18. Do not use a permanent su wrapper

Do not create a module that overlays:

```text
/system/bin/su
/system/xbin/su
```

On the SM-R960 bring-up, a permanent `su` wrapper could cause bootloops because it can run before the manager APK path and `libksud.so` are stable.

Use:

```sh
"$KSUD" debug su -g
```

If a wrapper module was created, remove it from root shell:

```sh
rm -rf /data/adb/modules/resukisu_su_wrapper
rm -rf /data/adb/modules_update/resukisu_su_wrapper
sync
reboot
```

---

# 19. Troubleshooting

## Workflow cannot download avbtool

Use retry/fallback logic or include:

```text
tools/avbtool.py
```

in your repo.

## Build fails with only `Error 2`

Capture full build log:

```sh
set +e
make O=out_resukisu -j"$(nproc)" 2>&1 | tee "$GITHUB_WORKSPACE/dist/resukisu-build.log"
BUILD_STATUS=${PIPESTATUS[0]}
set -e

if [ "$BUILD_STATUS" -ne 0 ]; then
  grep -nEi "error:|fatal:|undefined reference|multiple definition|No such file|Permission denied|ld.lld|clang|modpost|make\[[0-9]+\]: \*\*\*" "$GITHUB_WORKSPACE/dist/resukisu-build.log" | tail -250 || true
  tail -350 "$GITHUB_WORKSPACE/dist/resukisu-build.log" || true
  exit "$BUILD_STATUS"
fi
```

## Permission denied while patching source

Samsung tarballs may extract read-only files.

Add:

```sh
chmod -R u+rwX work/kernel
```

after extraction.

## `module install` gives permission denied

You are not root.

Wrong:

```text
wise6bl:/ $
```

Correct:

```text
wise6bl:/ #
```

Enter root first:

```sh
"$KSUD" debug su -g
```

## `KSUD=/lib/arm/libksud.so`

Manager package was not found. Use the spoofed-manager auto-detect block.

## Seccomp / SIGSYS manager crash

Some manager builds crash on Samsung Wear OS with:

```text
Fatal signal 31 (SIGSYS), code 1 (SYS_SECCOMP)
```

Use a manager build compatible with Samsung Wear OS ARM/ARMv7, or call `libksud.so` directly.

---

# 20. Porting checklist

Before flashing your forked build, confirm:

- [ ] Bootloader is unlocked.
- [ ] Firmware matches the watch.
- [ ] Samsung Open Source package matches or is closest available.
- [ ] Stock `boot.img`, `init_boot.img`, and `vbmeta.img` are from the same firmware.
- [ ] Defconfig is correct.
- [ ] `LOCALVERSION` matches `uname -r`.
- [ ] Partition sizes are correct.
- [ ] Rollback index is correct.
- [ ] Android version and security patch props are correct.
- [ ] AP tar contains only `boot.img`, `init_boot.img`, and `vbmeta.img`.
- [ ] ReSukiSU Manager supports the watch ABI.
- [ ] Stock firmware is ready for recovery.
- [ ] You understand how to remove bad modules if a module causes bootloop.

---

# License

Kernel-side work and patches should remain **GPL-2.0-only**, matching the Linux/Samsung kernel source.

Third-party components such as ReSukiSU remain under their original upstream licenses.
