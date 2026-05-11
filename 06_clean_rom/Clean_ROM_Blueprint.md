# Architectural Blueprint: LOKMAT APPLLP 5 MAX (C17S) Clean ROM
## Target: LineageOS 18.1 / AOSP 11

**Goal:** Strip ODM backdoor infrastructure, preserve nRF52832 hardware bridge, restore standard AOSP framework behavior.  
**Status:** Architecture complete — build in progress.

---

## 1. Kernel Layer (Prebuilt)

Kernel source is not available. The stock kernel binary is carried as a prebuilt.

| Item | Value |
|---|---|
| Kernel binary | `Image.gz` (4.9.190+) |
| Built-in drivers to leverage | `wiite_corp` (char 10:48) and `wiite_con` (char 10:51) |
| Cmdline requirement | Must carry `androidboot.hardware=mt6765` to satisfy ODM HALs despite physical MT6762 silicon |

---

## 2. Vendor Layer and Blobs — Isolation Bubble 2.0

The crypto isolation bubble from the TWRP build is carried forward and refined for full ROM use. The ROM operates within a system-wide VNDK-29 environment, so the containment zone path is updated to avoid namespace collision.

**Containment Zone:** `/vendor/lib64/hw/crypto_bubble/`

### Mandatory Blobs — The Untouchables

These blobs must remain isolated. They cannot be placed in the standard linker namespace.

| Blob | Reason |
|---|---|
| `libkeymaster4.so` | ODM-modified — collides with AOSP copy |
| `libkeymaster4support.so` | ODM-modified — collides with AOSP copy |
| `android.hardware.gatekeeper@1.0-impl.so` | Depends on ODM `libcrypto.so` |
| `libSoftGatekeeper.so` | Depends on ODM `libcrypto.so` |

All blobs in the containment zone must have RPATH enforced:

```bash
patchelf --set-rpath /vendor/lib64/hw/crypto_bubble <blob>.so
```

### VFS Bind Mount

Re-implement the TWRP bind mount in `init.target.rc`:

```
mount none /vendor/lib64/hw/crypto_bubble /vendor/lib64/hw bind
```

This projects the containment zone as a real filesystem mount at the vendor HAL path, satisfying the Android 11 linker's Treble namespace enforcement without symlinks.

---

## 3. Hardware Bridge — nRF52832 Native Stack

The privileged ODM agent `WearDeviceDeamPix` is burned. Its function — managing the nRF52832 coprocessor — is replaced with a clean native daemon.

### wiite_bridge_daemon

A C++ service that manages the hardware interface directly:

| Property | Value |
|---|---|
| Interface | `/dev/ttyS1` |
| Protocol | 115200 baud, 8N1 |
| Init sequence | Write `0x07`, `0x08`, `0xF3` to `/sys/devices/virtual/misc/wiite_corp_ctrl/command` |
| Run as | `UID 1001 (telephony)` or `root` with scoped sepolicy |
| SELinux | Custom domain with `ttyS1_device:chr_file rw_file_perms` |

### Sensor HAL

Use `android.hardware.sensors@2.0-service-mediatek`. Provide `sensors.mt6765.so` in its own isolation zone if it carries ODM-modified `libutils` dependencies — apply the same RPATH strategy as the crypto bubble.

---

## 4. Framework Layer — Surgical AOSP Restoration

### Kill-Switch Removal

Remove all ODM hooks from the framework source before compilation:

| Target | Action |
|---|---|
| `WiitePackageManagerUtil` in `PackageManagerService.java` | Remove entirely — restore standard permission grant logic |
| `Configuration.isSpecialApp()` calls in `ActivityManagerService.java` | Remove all call sites |
| `Configuration.isSpecialApp()` calls in `DisplayPolicy.java` | Remove all call sites |
| `Configuration.isSpecialApp()` method definition | Remove from `Configuration` class |

Any retained APK that calls `Configuration.isSpecialApp()` will crash on the clean framework — audit all remaining ODM APKs before inclusion.

### RIL Capability Bypass

The ODM baseband enforces capability control via `capctrl` service and `mtkradioex` calls. Without the ODM certificate chain from `WapiCertManager.apk`, modem capability enablement calls will fail. Resolution options:

1. Pre-authenticate by carrying the ODM certificates from `WapiCertManager.apk` into the clean build's trusted certificate store
2. Determine whether the modem accepts permissive mode for `mtkCap1` service without ODM certificate validation

This is an open item requiring dynamic analysis against the running clean ROM.

---

## 5. UI and Cosmetics — Standard AOSP Approach

The ODM hides the navigation and status bars through framework hooks. The clean ROM replaces this with a standard AOSP Runtime Resource Overlay.

### Navigation Bar RRO

Create overlay `vendor.wiite.overlay.ui`:

```xml
<!-- config_showNavigationBar -->
<bool name="config_showNavigationBar">false</bool>

<!-- status_bar_height -->
<dimen name="status_bar_height">0dp</dimen>
```

### Power Button Hook

Implement a clean long-press handler in `PhoneWindowManager.java` to trigger Recents — replacing the ODM's dirty implementation with standard AOSP behavior.

---

## 6. Burn List — What Gets Removed

These paths are excluded from the clean ROM at build time. They do not exist in the output image:

| Path | Threat | Why It Cannot Stay |
|---|---|---|
| `/vendor/operator/app/Stopwatch/` | UID 1000 backdoor agent | System-level persistent agent with shell execution and silent APK install |
| `/product/priv-app/AdupsFota/` | Persistent spyware | Documented data collection — survives factory reset |
| `/product/priv-app/WearDeviceDeamPix/` | Privileged ODM hardware bridge | Replaced by `wiite_bridge_daemon` — no justification for priv-app level |
| `/product/app/WearCleanTaskPro/` | Process execution kill-switch | Kills non-whitelisted apps — protected by framework hooks |
| `/product/app/WearAppFreeze/` | Application freeze agent | Immobilizes third-party apps — protected by framework hooks |
| `/system/priv-app/HeilsFaceUnlockDM101/` | Face unlock — data scope unverified | Unknown data collection scope — JADX decompile pending |
| `/system/etc/permissions/pms_sysapp_grant_permission_list.txt` | Backdoor permission registry | No AOSP equivalent — purpose-built for ODM privilege escalation |

---

*Architecture: beright1976 | 2026*
