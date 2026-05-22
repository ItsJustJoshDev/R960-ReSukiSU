# R960-ReSukiSU

Custom ReSukiSU kernel/AP build workflow and usage notes for the Samsung Galaxy Watch6 Classic **SM-R960 / wise6bl**.

This project targets the firmware and kernel version below:

| Item | Value |
|---|---|
| Device | Samsung Galaxy Watch6 Classic |
| Model | `SM-R960` |
| Codename | `wise6bl` |
| Product | `wise6blue` |
| Firmware | `R960XXU2CZB6` |
| Android | `16` |
| Kernel | `5.15.180-33003255-abR960XXU2CZB6` |
| Platform | `s5e5515` / `erd5515` |
| Bootloader | Unlocked required |
| Verified boot after custom flash | Expected: `orange` |
| Warranty bit after unlock/custom flash | Expected: `1` |

---

## Current status

Confirmed working:

- ReSukiSU kernel boots on `SM-R960`.
- ReSukiSU Manager can detect the kernel as **Working**.
- Root shell works through ReSukiSU's `libksud.so` command:

```sh
"$KSUD" debug su -g
```

The running kernel should show:

```sh
CONFIG_KSU=y
CONFIG_KSU_MANUAL_HOOK=y
CONFIG_KSU_MULTI_MANAGER_SUPPORT=y
```

Known notes:

- Normal `su` may not exist by default.
- ReSukiSU root works even if `/system/bin/su` is missing.
- Do **not** use a permanent `/system/bin/su` wrapper module on this watch. It can cause bootloops.
- Some manager builds may crash or fail because of Samsung/Wear OS seccomp.
- Use an ARM/ARMv7-compatible ReSukiSU Manager APK for the watch.
- Spoofed managers may use a different package name, so commands must auto-detect `libksud.so`.

---

## Important warning

This is experimental.

Flash only after bootloader unlock. Use only on matching firmware:

```text
SM-R960 / wise6bl / R960XXU2CZB6
```

Flashing custom images can trip warranty/custom binary state and can cause bootloops or data loss.

---

# Basic ADB notes

If you are already inside an interactive watch shell, **do not** prefix commands with `adb shell`.

Wrong inside watch shell:

```sh
adb shell
```

Correct inside watch shell:

```sh
id
uname -a
```

From outside the watch shell, for example from a PC or Termux on a phone:

```sh
adb connect WATCH_IP:PORT
adb shell
```

---

# Verify the custom kernel booted

Run this from the watch shell:

```sh
echo "===== BOOT / KERNEL ====="
uname -a
uname -r
getprop ro.boot.verifiedbootstate
getprop ro.boot.vbmeta.device_state
getprop ro.boot.warranty_bit

echo
echo "===== KSU CONFIG FROM RUNNING KERNEL ====="
zcat /proc/config.gz 2>/dev/null | grep -Ei "CONFIG_KSU|CONFIG_KERNELSU|CONFIG_RESUKISU|CONFIG_KSU_MANUAL_HOOK" || echo "No KSU config symbols found"
```

Expected after the custom ReSukiSU AP boots:

```text
ro.boot.verifiedbootstate = orange
ro.boot.warranty_bit = 1
CONFIG_KSU=y
CONFIG_KSU_MANUAL_HOOK=y
CONFIG_KSU_MULTI_MANAGER_SUPPORT=y
```

---

# ReSukiSU Manager notes

There are two manager situations:

1. **Non-spoofed manager**
   - Package name is known:
   - `com.resukisu.resukisu`

2. **Spoofed manager**
   - Package name may be different.
   - Commands must scan installed packages for `libksud.so`.

Use the spoofed-manager commands if you are unsure.

---

# Root shell commands

## Method A: Non-spoofed ReSukiSU Manager

Use this when the manager package is:

```text
com.resukisu.resukisu
```

Find `libksud.so`:

```sh
PKG=com.resukisu.resukisu
LIB="$(dumpsys package "$PKG" | sed -n 's/.*legacyNativeLibraryDir=//p' | head -1)"
KSUD="$LIB/arm/libksud.so"

echo "PKG=$PKG"
echo "LIB=$LIB"
echo "KSUD=$KSUD"
ls -l "$KSUD"
```

Start root shell:

```sh
"$KSUD" debug su -g
```

If root works, the prompt changes from:

```text
wise6bl:/ $
```

to:

```text
wise6bl:/ #
```

Test root:

```sh
id
whoami
getenforce
uname -a
```

Expected:

```text
uid=0(root)
```

---

## Method B: Spoofed ReSukiSU Manager

Use this when the manager package is spoofed, renamed, or unknown.

Auto-find `libksud.so`:

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
```

Start root shell:

```sh
"$KSUD" debug su -g
```

If root works, the prompt changes to:

```text
wise6bl:/ #
```

Test root:

```sh
id
whoami
getenforce
```

---

# One-shot root command for spoofed manager

Paste this from normal shell when you just want to enter root quickly:

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
"$KSUD" debug su -g
```

---

# Install APK packages without reboot

APK installs do not need a reboot.

If the APK is already on the watch:

```sh
pm install -r /sdcard/Download/App.apk
```

If you are pushing from a PC or phone Termux:

```sh
adb push App.apk /sdcard/Download/App.apk
adb shell pm install -r /sdcard/Download/App.apk
```

---

# ReSukiSU module install template

Most ReSukiSU/KSU modules need root to install.

## Step 1: Put module ZIP in Downloads

From outside the watch shell:

```sh
adb push ModuleName.zip /sdcard/Download/ModuleName.zip
```

Example:

```sh
adb push mountify.zip /sdcard/Download/mountify.zip
```

Or manually copy the ZIP to:

```text
/sdcard/Download/
```

---

## Step 2: Enter root shell

### Non-spoofed manager

```sh
PKG=com.resukisu.resukisu
LIB="$(dumpsys package "$PKG" | sed -n 's/.*legacyNativeLibraryDir=//p' | head -1)"
KSUD="$LIB/arm/libksud.so"

"$KSUD" debug su -g
```

### Spoofed manager

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

Your prompt must become:

```text
wise6bl:/ #
```

---

## Step 3: Install the module from the root shell

Once you are at the `#` root prompt:

```sh
MODZIP=/sdcard/Download/mountify.zip

"$KSUD" module install "$MODZIP"
"$KSUD" module list
sync
reboot
```

Replace:

```text
/sdcard/Download/mountify.zip
```

with your module path.

---

# Mountify module install example

From normal watch shell:

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
ls -lh /sdcard/Download/mountify.zip
"$KSUD" debug su -g
```

Then inside the root `#` shell:

```sh
MODZIP=/sdcard/Download/mountify.zip

id
"$KSUD" module install "$MODZIP"
"$KSUD" module list
sync
reboot
```

---

# Module commands

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

Undo uninstall mark:

```sh
"$KSUD" module undo-uninstall MODULE_ID
```

Run module action:

```sh
"$KSUD" module action MODULE_ID
```

Module configuration:

```sh
"$KSUD" module config MODULE_ID
```

---

# Installing modules without immediately rebooting

You can install a module without rebooting:

```sh
"$KSUD" module install /sdcard/Download/ModuleName.zip
"$KSUD" module list
sync
```

You can try to trigger service stages manually:

```sh
"$KSUD" post-fs-data
"$KSUD" services
"$KSUD" boot-completed
```

However, many modules still require a reboot to fully apply if they modify:

- `/system`
- `/product`
- `/vendor`
- overlays
- framework files
- boot animations
- fonts
- system properties
- privileged apps
- SELinux-related behavior

For those modules, reboot is still required:

```sh
reboot
```

---

# Do not use a permanent su wrapper

Do **not** create a module that overlays:

```text
/system/bin/su
/system/xbin/su
```

on this watch.

That wrapper can cause bootloops because it runs too early, before the manager APK path and `libksud.so` are stable. Use the direct `KSUD` commands instead:

```sh
"$KSUD" debug su -g
```

If a `resukisu_su_wrapper` module was already created, remove it from a root shell:

```sh
rm -rf /data/adb/modules/resukisu_su_wrapper
rm -rf /data/adb/modules_update/resukisu_su_wrapper
sync
reboot
```

If the watch is bootlooping because of that module, disable modules using your recovery/bootloop-protection method, then delete:

```text
/data/adb/modules/resukisu_su_wrapper
/data/adb/modules_update/resukisu_su_wrapper
```

---

# Troubleshooting

## `su: inaccessible or not found`

This is expected on some ReSukiSU builds.

Use:

```sh
"$KSUD" debug su -g
```

Do not install the permanent `su` wrapper module on this watch.

---

## `KSUD=/lib/arm/libksud.so`

This means the manager package was not found, so the path was built from an empty variable.

Fix by using the spoofed-manager auto-detect block:

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

echo "$KSUD"
```

---

## `module install` gives permission denied

You are not in root shell.

Wrong prompt:

```text
wise6bl:/ $
```

Correct prompt:

```text
wise6bl:/ #
```

Start root first:

```sh
"$KSUD" debug su -g
```

Then retry:

```sh
"$KSUD" module install /sdcard/Download/ModuleName.zip
```

---

## `/data/adb` gives permission denied

That is normal from non-root shell.

Enter root first:

```sh
"$KSUD" debug su -g
```

Then:

```sh
ls -la /data/adb
```

---

## Manager says working but `su` is missing

That is expected on some ReSukiSU builds.

Use:

```sh
"$KSUD" debug su -g
```

Do not create a permanent `su` wrapper on this watch.

---

## Seccomp / SIGSYS manager crashes

Some ReSukiSU manager builds may crash on Samsung Wear OS with logs like:

```text
Fatal signal 31 (SIGSYS), code 1 (SYS_SECCOMP)
```

Use a manager build that works on ARM/ARMv7 Wear OS, or use `libksud.so` directly through the commands above.

---

# Build outputs

The GitHub Actions workflow publishes:

```text
R960_ReSukiSU_AP.tar
R960_ReSukiSU_AP.tar.md5
Image-ReSukiSU-R960
resukisu.config
SHA256SUMS.txt
RELEASE_NOTES.txt
```

Recommended flash file:

```text
R960_ReSukiSU_AP.tar
```

Use `.tar.md5` only if Odin/Brokkr requires it.

---

# GitHub Actions inputs

Recommended inputs:

```text
resukisu_setup_url = https://raw.githubusercontent.com/ReSukiSU/ReSukiSU/main/kernel/setup.sh
source_release_tag = source-tarballs
firmware_release_tag = firmware-r960-czb6
```

Required release assets:

`source-tarballs`:

```text
Kernel.tar.gz
R960XXU1CYK2_Kernel.tar.gz
```

`firmware-r960-czb6`:

```text
boot.img.lz4
init_boot.img.lz4
vbmeta.img.lz4
```

---

# License

Kernel-side work and patches are licensed under **GPL-2.0-only**, matching the Linux/Samsung kernel source.

Third-party components such as ReSukiSU remain under their original upstream licenses.
