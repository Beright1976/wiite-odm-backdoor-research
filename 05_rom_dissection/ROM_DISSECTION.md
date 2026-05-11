# ROM Dissection — LOKMAT APPLLP 5 MAX (C17S)
## Complete Stock Firmware Analysis

**Researcher:** Albert Pittman (beright1976)  
**Device:** LOKMAT APPLLP 5 MAX (C17S)  
**Method:** TWRP partition pull + binary forensics  
**Status:** Complete  

---

## Why Standard Tools Failed

Every standard Android reverse engineering workflow assumes a cooperative firmware. Tools like `adb pull`, `apktool`, and SP Flash Tool partition extraction are built around the assumption that what the OS reports is what the device contains. On the C17S that assumption is false at every layer.

The Topwise/Wiite/Linswear BSP has hardcoded RPATH values embedded directly into binaries and libraries, modified `framework.jar` and `services.jar` at the binary level, and an ODM-modified crypto stack that shares library names with AOSP libraries while containing entirely different symbol tables. A developer using standard tools gets a picture of what the ODM wants them to see — not what is actually there.

Getting the real picture required:

- **TWRP with full FBE decryption** — without this, partition pulls are incomplete. `/data` is inaccessible, metadata is encrypted, and file permissions and SELinux contexts are stripped. The TWRP build documented in `04_twrp_build/` was a prerequisite for this entire section.
- **Ghidra** — binary-level disassembly of `services.jar`, native libraries, and ODM-modified framework binaries
- **JADX** — DEX decompilation of APKs including the Stopwatch backdoor agent
- **binwalk** — firmware image structure analysis and embedded payload extraction
- **patchelf / readelf** — ELF header inspection, RPATH mapping, dependency chain tracing
- **mtkclient BROM extraction** — all 42 partitions pulled as raw binary at the BROM level, bypassing the OS entirely
- **Direct hex analysis** — the only reliable method for identifying ODM modifications when binary headers are intact but symbol tables have been altered

Without this toolchain applied in combination, the backdoor infrastructure in this firmware would remain invisible.

---

## The Super Partition Problem

The first ROM dissection attempt failed. The extracted super partition was incomplete.

`super` on this device is a dynamic partition container holding `system`, `vendor`, and `product` as logical sub-partitions. Standard extraction via `lpunpack` against a super image pulled through `adb` or SP Flash Tool produces a structurally valid output — files extract, the directory tree looks correct, APKs open. The incompleteness is not obvious.

What was missing: file metadata, SELinux contexts, and the complete permission structure. Android's dynamic partition system writes extended attributes and security labels into the filesystem at the block level. An `adb pull` traversal does not preserve these — it copies file contents only. Several ODM binaries that depend on their SELinux context for privilege inheritance appeared present but were non-functional in analysis because their context labels were gone.

The solution was TWRP with full FBE decryption. TWRP mounts the partitions at the block device level with full filesystem access — not through the Android content provider layer. A `tar` archive pulled through TWRP ADB preserves:

- File permissions (owner, group, mode bits)
- Extended attributes (`security.selinux` context labels)
- Symlink targets
- Sparse file structure
- Directory metadata

This is why building TWRP first was not optional. The ROM dissection depends on data that only a decrypting block-level recovery can provide.

---

## Partition Pull Procedure

All three primary partitions were pulled via TWRP ADB after confirming FBE decryption was active:

```bash
# Confirm decryption active — /data must show real content
adb shell ls /data/data/

# Pull system partition — preserving all metadata
adb shell tar -czp /system | adb pull - system_full.tar.gz

# Pull vendor partition
adb shell tar -czp /vendor | adb pull - vendor_full.tar.gz

# Pull product partition
adb shell tar -czp /product | adb pull - product_full.tar.gz
```

Super partition raw binary was also extracted via mtkclient in BROM mode for block-level analysis:

```bash
python3 mtk.py r super super_raw.bin
python3 lpunpack.py super_raw.bin super_unpacked/
```

The BROM extraction and the TWRP tar pull were cross-referenced. Discrepancies between the two identified three files where the running OS was serving modified content versus what was on the block device — consistent with runtime hooks in `services.jar`.

---

## Framework Binary Modification — The Core Finding

The most significant ROM dissection finding is the surgical modification of `services.jar` — the compiled Android framework services binary.

`services.jar` is the largest and most critical component in the Android framework. It contains `PackageManagerService`, `ActivityManagerService`, `WindowManagerService`, and hundreds of other core OS services. On a stock AOSP device, `services.jar` is compiled from the AOSP source tree. On this device it has been modified at the DEX bytecode level after compilation.

### WiitePackageManagerUtil

Identified via Ghidra DEX analysis of `services.jar`.

A custom class `WiitePackageManagerUtil` has been injected into `PackageManagerService`. This class intercepts permission grant calls for a hardcoded list of package names and returns a pre-approved grant result, bypassing the standard permission validation logic. The package names in the list correspond exactly to the ODM agent suite — the Stopwatch backdoor, AdupsFota, WearCleanTaskPro, WearAppFreeze, and WearDeviceDeamPix.

The effect: these packages cannot have their permissions revoked by any standard Android mechanism. The package manager will report them as granted. Attempting to revoke them via `pm revoke` returns success but the next permission check for the same package returns granted again.

### Configuration.isSpecialApp()

Identified via Ghidra analysis of the `Configuration` class within `services.jar`.

`isSpecialApp()` is a non-standard method injected into Android's `Configuration` class. It is called from `ActivityManagerService` and `DisplayPolicy` to determine whether a package should be exempt from:

- Background process killing
- Battery optimization restrictions  
- App standby bucketing
- Display policy enforcement

The method contains a hardcoded package list identical to the one in `WiitePackageManagerUtil`. Any package on this list runs permanently in the foreground, cannot be killed by the system, and is exempt from all power management enforcement regardless of user settings.

This is not a standard ODM customization. There is no AOSP mechanism that provides this behavior. It is purpose-built infrastructure.

---

## APK Inventory — 127 Pre-installed Applications

The complete APK inventory across all partitions was compiled from the TWRP partition pulls. 127 pre-installed applications were identified and categorized.

### Partition Distribution

| Partition | Path | APK Count | Privilege Level |
|---|---|---|---|
| system | `/system/app/` | 31 | Standard system |
| system | `/system/priv-app/` | 18 | Privileged system |
| vendor | `/vendor/app/` | 4 | Standard vendor |
| vendor | `/vendor/operator/app/` | 9 | **UID 1000 system level** |
| product | `/product/app/` | 41 | Standard product |
| product | `/product/priv-app/` | 24 | Privileged product |

### Critical Security-Relevant Applications

| APK | Location | Risk | Finding |
|---|---|---|---|
| `Stopwatch.apk` | `/vendor/operator/app/` | **CRITICAL** | UID 1000 backdoor agent — full analysis in `01_security_findings/` |
| `AdupsFota.apk` | `/product/priv-app/` | **CRITICAL** | Persistent spyware — documented exfiltration history |
| `WearDeviceDeamPix.apk` | `/product/priv-app/` | HIGH | Privileged native bridge to nRF52832 hardware stack |
| `WearCleanTaskPro.apk` | `/product/app/` | HIGH | Process execution control — kills non-whitelisted apps |
| `WearAppFreeze.apk` | `/product/app/` | HIGH | Suspends and immobilizes third-party applications |
| `HeilsFaceUnlockDM101.apk` | `/system/priv-app/` | HIGH | Face recognition system — data collection scope unverified |
| `WapiCertManager.apk` | `/system/app/` | MEDIUM | Certificate management — installs trusted roots |
| `SensorHub.apk` | `/vendor/operator/app/` | MEDIUM | nRF52832 sensor bridge — data routing unverified |

### ODM Sensor Stack APKs

A suite of UART bridge applications for the nRF52832 coprocessor were identified in `/vendor/operator/app/`:

```
WearFitnessUart.apk     — Fitness sensor data bridge
WearHeartRateUart.apk   — Heart rate sensor data bridge
WearSdkUart.apk         — SDK-level hardware bridge
WearSpoUart.apk         — SpO2 sensor data bridge
WearSyncContact.apk     — Contact synchronization
WearSettings.apk        — ODM settings interface
WearRecorder.apk        — Recording capability
```

All of these applications share the `vendor/operator/app/` path, meaning they all inherit UID 1000 system-level privileges. None of them require system privileges to perform their stated sensor bridge functions. The privilege level is unjustified by the stated function.

---

## HAL Inventory

Hardware Abstraction Layer binaries were catalogued from `/vendor/lib64/hw/` and cross-referenced against the live `hwservicemanager` registration log.

### Key HAL Findings

| HAL | Binary | Finding |
|---|---|---|
| `keymaster@4.0` | `android.hardware.keymaster@4.0-service` | ODM-modified — symbol table differs from AOSP. Contains hard `DT_NEEDED` on `keymaster@3.0.so`. Requires crypto isolation bubble. |
| `gatekeeper@1.0` | `gatekeeper.default.so` | ODM-modified — requires VFS bind mount for passthrough discovery. |
| `camera@3.4` | `libcameracustom.so` | Custom ODM implementation — `gc030a` sensor confirmed via string analysis |
| `audio primary` | `audio.primary.mt6765.so` | Standard MTK audio HAL |
| `sensors@2.0` | Routes via nRF52832 UART | Sensor data sourced from coprocessor, not direct kernel driver |

### The Keymaster Chain — Confirmed Software-Only

Full dependency trace via `/proc/PID/maps` on the running keymaster process:

```
android.hardware.keymaster@4.0-service
  └── libkeymaster4.so
        └── libpuresoftkeymasterdevice.so    ← pure software, no TEE
              └── libcrypto.so (OpenSSL 1.1.1w, ODM-modified)
                    └── kernel AES via syscall
```

No TEE involvement at any step. The device reports Keymaster 4.0 compliance. It is a software keymaster running on a modified OpenSSL stack. The iTrustee TEE is present in memory but not in the key derivation path.

---

## Driver Inventory — The Ghost Hardware

`fstab.mt6765` and the kernel `modules_full.txt` reveal drivers for hardware not present in the watch:

| Driver | Hardware It Supports | Present in Watch |
|---|---|---|
| Thermal printer driver | POS receipt printers | NO |
| HDMI output stack | External display | NO |
| Fingerprint HAL (`fpc1020`) | Biometric scanner | NO |
| Barcode scanner input | Handheld scanners | NO |
| `md1dsp`, `md1arm7`, `md3img` | Enterprise baseband | NO |
| `odmdtbo` | Enterprise device tree overlay | NO |

These drivers are not dead code left in accidentally. They are maintained, current, and match the production versions used in Topwise/Wiite enterprise handheld devices. The BSP is shared. The same firmware base — and the same backdoor infrastructure — is active in those enterprise devices.

---

## Cryptographic Library Analysis

ODM-modified library identification via `readelf` symbol table comparison against AOSP equivalents:

| Library | ODM Version | AOSP Version | Difference |
|---|---|---|---|
| `libcrypto.so` | OpenSSL 1.1.1w (ODM) | OpenSSL 1.1.1w (AOSP) | Symbol table altered — 23 symbols differ in offset. Causes namespace collision with AOSP copy. |
| `libhidlbase.so` | VNDK-29 (ODM) | VNDK-29 (AOSP) | Linked against ODM `libutils.so` — RPATH hardcoded to `/system/lib64/vndk-29/` |
| `libutils.so` | ODM | AOSP | Contains additional ODM symbols not in AOSP version |

The RPATH hardcoding in `libhidlbase.so` is the mechanism that makes standard blob deployment approaches fail. The binary will not load its dependencies from anywhere except the hardcoded path, regardless of `LD_LIBRARY_PATH`. This is why the crypto isolation bubble requires `patchelf --set-rpath` applied to all blobs — it is the only way to override an embedded RPATH.

---

## SELinux Policy Analysis

The SELinux policy was extracted in both compiled binary (`plat_sepolicy.cil`) and source form. Key findings:

- SELinux is set to **PERMISSIVE** in the ODM configuration (`ro.boot.selinux=permissive`)
- The `pms_sysapp_grant_permission_list.txt` file is explicitly referenced in the ODM sepolicy as a trusted permissions source
- ODM agent processes (`wiite_stopwatch`, `adupsfota`) have custom domains with `permissive` declarations even within the policy — meaning they would be permissive even if the global policy were set to enforcing
- `WearDeviceDeamPix` has a custom SELinux domain with explicit `allow` rules for direct hardware device node access bypassing standard HAL abstraction

---

## Tools Required — The Real Toolchain

This dissection required tools that are not part of any standard Android development workflow:

| Tool | Use |
|---|---|
| **Ghidra** | DEX bytecode analysis of `services.jar` — identifying `WiitePackageManagerUtil` and `Configuration.isSpecialApp()` |
| **JADX** | APK decompilation — full Java source reconstruction of Stopwatch backdoor, AdupsFota, sensor bridge suite |
| **binwalk** | Firmware image structure — identifying embedded payloads in partition images |
| **patchelf** | ELF RPATH modification — required to understand and fix the library collision problem |
| **readelf** | ELF header and symbol table inspection — identifying ODM modifications |
| **mtkclient** | BROM-level partition extraction — bypassing OS entirely for raw block access |
| **lpunpack** | Dynamic partition extraction from super image |
| **TWRP (custom)** | Block-level partition pull with full metadata preservation — prerequisite for everything else |
| **hex editor** | Direct binary inspection — the final authority when every other tool gives a misleading result |

No single tool was sufficient. The ODM modifications are specifically designed to appear compliant to surface-level analysis. Only the combination of binary-level forensics, live process memory mapping, and block-level partition access reveals the complete picture.

---

## Conclusion

The stock ROM of the C17S is not a consumer Android device with some bloatware. It is an enterprise-grade ODM platform with a framework that has been surgically modified to protect a persistent agent suite from standard Android security enforcement.

The modifications are not errors. They are not optimization shortcuts. The architecture of `WiitePackageManagerUtil`, the `isSpecialApp()` hook, the `pms_sysapp_grant_permission_list.txt` registry, and the UID 1000 Stopwatch agent represents deliberate, coordinated engineering across multiple layers of the Android stack.

Documenting it required rebuilding the entire trust chain from the BROM level up — which is why TWRP came first, why the forensic protocol exists, and why the database in `03_device_forensics/` is the prerequisite for everything in this section.
