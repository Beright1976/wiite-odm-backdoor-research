# Wiite ODM Backdoor Research
### Forensic Security Research: Topwise / Wiite / Linswear — MT6765 Platform Supply Chain Compromise

**Researcher:** Albert Pittman (Beright1976)  
**CVE Status:** Pending assignment  
**Research Status:** Active — security findings complete, clean ROM build in progress  

---

## What This Is

This repository documents the complete forensic investigation of the **LOKMAT APPLLP 5 MAX (C17S)** — an Android 10 smartwatch running on a MediaTek MT6762V/WB SoC with MT6765 platform HALs, manufactured by the Chinese ODM **Topwise / Wiite / Linswear**.

What started as a TWRP recovery build for an undocumented device became a full security disclosure. The investigation confirmed that the ODM's Board Support Package (BSP) contains deliberate, multi-layer backdoor infrastructure embedded in the Android framework — not a coding error, not an accident.

The same BSP is deployed across multiple device categories:

- Android watch phones (LOKMAT and other brands)
- Mobile POS terminals
- Logistics and inventory handheld scanners

**This is a supply chain vulnerability. Any device built on this BSP carries the same backdoor infrastructure.**

---

## Key Findings Summary

### 1. UID 1000 Persistent Backdoor Agent (`Stopwatch.apk`)
Located at `/vendor/operator/app/Stopwatch/Stopwatch.apk`. Presents as a stopwatch utility. Is not.

The application runs as `android.uid.system` (UID 1000), inheriting total system authority. It boots on every device start via `BOOT_COMPLETED` and provides:

- **Silent APK installation** — `AppUtils.installApp()` with `INSTALL_PACKAGES` inherited via system UID. No user interaction required.
- **Arbitrary shell execution** — `ShellUtils` class with root-level access on an already-unlocked device
- **Hardware identifier harvesting** — IMEI, IMSI, WiFi MAC, SIM serial via `DeviceUtils` and `PhoneUtils`
- **WiFi intelligence** — full network scan including SSID, BSSID, gateway, and subnet
- **Privilege escalation surface** — `MyService` exported without permission guard; any third-party app can bind to a UID 1000 process and execute privileged code

DEX string analysis confirms Chinese development environment: hardcoded Alibaba DNS (`223.5.5.5`), Baidu SDK fragments (`Baidurack`), and Tencent QQ Browser authority string (`com.tencent.mtt.fileprovider`) — none of which have any legitimate explanation in a stopwatch utility.

Persistence mechanism: stored in the read-only vendor partition. Survives factory reset. Cannot be removed by any standard user operation. Firmware replacement is the only path to removal.

---

### 2. Framework-Level Security Subversion (`services.jar`)

The ODM surgically modified the Android framework binary to protect its agents from standard OS security enforcement:

- **`WiitePackageManagerUtil`** — custom class injected into `PackageManagerService.java` to exempt designated packages from permission validation
- **`Configuration.isSpecialApp()`** — non-standard method injected into `ActivityManagerService.java` and `DisplayPolicy.java` to exempt ODM packages from background execution limits, power management restrictions, and task-killing

These hooks protect a specific suite of agents:

| Agent | Location | Function |
|---|---|---|
| `WearCleanTaskPro.apk` | `/product/app/` | "Executioner" — kills all non-whitelisted user processes |
| `WearAppFreeze.apk` | `/product/app/` | Suspends and immobilizes third-party applications |
| `WearDeviceDeamPix` | `/product/priv-app/` | High-privilege native bridge to nRF52832 hardware stack |
| `AdupsFota` | `/product/priv-app/` | Primary persistent spyware agent |

---

### 3. Backdoor Registry (`pms_sysapp_grant_permission_list.txt`)

Located at `/system/etc/permissions/pms_sysapp_grant_permission_list.txt`.

This file explicitly grants system-level permissions to ODM packages while **bypassing the standard Android Package Manager grant logic entirely**. It is a purpose-built permission escalation registry with no legitimate AOSP equivalent.

---

### 4. Enterprise Threat Multiplier — BSP Ghost Entries

Forensic analysis of `fstab.mt6765` reveals entries for hardware components absent from the watch but standard in industrial handhelds: `md1dsp`, `md1arm7`, `md3img`, `odmdtbo`. Inactive drivers for POS thermal printers, HDMI output, and biometric fingerprint scanners are present in the kernel.

This confirms the Topwise/Wiite BSP is a multi-purpose enterprise firmware base. The backdoor infrastructure in this BSP is not consumer-grade bloatware — it was built for enterprise deployment and is present in devices running in logistics, point-of-sale, and inventory management infrastructure globally.

---

### 5. Cryptographic Facade

The device reports Keymaster 4.0. Forensic mapping confirms zero hardware TEE involvement. The stack uses `libpuresoftkeymasterdevice.so` and ODM-modified `libcrypto.so` (OpenSSL 1.1.1w) with altered symbol tables that conflict with AOSP libraries of the same filename.

The FBE CE encryption key is wrapped with a default empty credential. The encryption is a UI-only facade — the vault is open.

---

## Repository Structure

```
wiite-odm-backdoor-research/
│
├── 01_security_findings/     # Vulnerability reports, APK analysis, CVE tracking
├── 02_forensic_protocol/     # The methodology: how to forensically dissect any
│                             # black-box Android device from zero
├── 03_device_forensics/      # Complete hardware and software database for the C17S
│                             # (the digital twin — the single source of truth)
├── 04_twrp_build/            # TWRP 11 build results, build configuration, boot log
│                             # analysis, all resolved failures documented
├── 05_rom_dissection/        # Stock ROM analysis: all 127 APKs, HAL inventory,
│                             # framework binary analysis, driver inventory
└── 06_clean_rom/             # Clean AOSP ROM build (in progress)
```

---

## Related Repositories

| Repository | Description |
|---|---|
| [twrp-build-protocol](https://github.com/Beright1976/twrp-build-protocol) | The complete protocol for building TWRP recovery for any undocumented MTK device from binary forensics |
| [Lokmat5MAX_wiite-c17s_mt6765](https://github.com/Beright1976/Lokmat5MAX_wiite-c17s_mt6765) | The working TWRP 11 device tree for the C17S — full FBE decryption, zero errors in recovery log |
| [MT6897-Warrior-Toolkit](https://github.com/Beright1976/MT6897-Warrior-Toolkit) | Same methodology applied to the Lenovo TB373FU / TB375FC (MT6897) |

---

## Affected Devices

The following device categories are confirmed or suspected to run this BSP:

- LOKMAT APPLLP 5 MAX (C17S) — **confirmed, forensically verified**
- Other LOKMAT / APPLLP series watch phones — suspected
- Topwise / Wiite / Linswear branded Android watch phones — suspected
- Mobile POS terminals using the MT6765 Wiite BSP — suspected
- Logistics and inventory handheld scanners using the MT6765 Wiite BSP — suspected

If you have a device from this ODM family and want to verify, the forensic protocol in `02_forensic_protocol/` gives you the exact methodology to confirm whether your device carries this BSP.

---

## Mitigation

For any device running this BSP, the only path to a trusted state is **complete firmware replacement**. Factory reset does not remove the backdoor infrastructure — it lives in the read-only vendor and product partitions.

Minimum removal targets (the burn list):

```
/vendor/operator/app/Stopwatch/
/product/priv-app/AdupsFota/
/product/priv-app/WearDeviceDeamPix/
/product/app/WearCleanTaskPro/
/system/priv-app/HeilsFaceUnlockDM101/
/system/etc/permissions/pms_sysapp_grant_permission_list.txt
```

Framework restoration requires recompiling `services.jar` from AOSP source to remove `WiitePackageManagerUtil` and `Configuration.isSpecialApp()` hooks.

---

## Disclosure Status

| Item | Status |
|---|---|
| CVE application | Submitted — pending assignment |
| MITRE contact | In progress |
| XDA community disclosure | Published |
| Media outreach | In progress |
| Dynamic analysis (network capture) | Pending |
| Full JADX decompile of `CompanyActivity` | Pending |

---

## Methodology Note

Every finding in this repository is forensically verified. Nothing is assumed or borrowed from similar devices. All values trace to one of three authoritative sources:

- **BROM-level binary extraction** via mtkclient (all 42 partitions as individual `.bin` files)
- **Live ADB root session** on the physical device (SELinux permissive, Magisk root)
- **Direct binary analysis** of extracted partition contents (hex, strings, androguard, Ghidra)

The hardware and software forensics database in `03_device_forensics/` is the single source of truth for all device parameters. A mathematically correct rebuild of the device environment can be reproduced from that database alone.

---

## License

GPL-3.0 — See [LICENSE](LICENSE) for details.

Security research findings are open. If you build on this work, keep it open.
