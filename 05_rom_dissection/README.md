# ROM Dissection — Stock Firmware Analysis

This section documents the complete dissection of the LOKMAT APPLLP 5 MAX (C17S) stock ROM — all three primary partitions (system, vendor, product), the full APK inventory, HAL mapping, driver inventory, framework binary analysis, and cryptographic library audit.

## Documents

| Document | Description |
|---|---|
| [ROM_DISSECTION.md](ROM_DISSECTION.md) | Complete ROM dissection report — partition pull methodology, super partition problem and solution, framework binary modification analysis, 127-APK inventory, HAL inventory, driver inventory, SELinux policy analysis, and full toolchain |

## Prerequisites

This entire section depends on `04_twrp_build/`. Without TWRP with full FBE decryption, partition pulls are incomplete — file metadata, SELinux contexts, and permission structures are lost. The ROM dissection was performed after TWRP was operational.

## Key Findings

- `services.jar` confirmed modified at DEX bytecode level — `WiitePackageManagerUtil` and `Configuration.isSpecialApp()` injected into the Android framework
- 127 pre-installed applications inventoried across 6 partition paths
- 9 applications in `/vendor/operator/app/` all carrying UID 1000 system privileges — including the Stopwatch backdoor
- Ghost hardware drivers confirmed — POS printer, fingerprint scanner, HDMI, enterprise baseband — confirming shared BSP with enterprise device categories
- Full keymaster dependency chain traced via `/proc/PID/maps` — confirmed pure software, no TEE
- ODM-modified crypto libraries identified via symbol table comparison — RPATH hardcoding confirmed as the mechanism behind standard deployment failures

## Raw Evidence

The raw logs and dumps underlying this analysis are stored locally at:

```
~/Downloads/LOKMAT_C17S_Forensics/DEVICE_LOGS&DUMPS/
```

Key files referenced in this analysis:

| File | Content |
|---|---|
| `dumpsys_package.txt` | Complete package manager dump — all installed packages |
| `hal_manifest.txt` | Live hwservicemanager HAL registration |
| `keymaster_deps.txt` | Keymaster dependency chain from live process |
| `plat_sepolicy.cil` | SELinux policy source |
| `modules_full.txt` | Complete kernel module list |
| `fstab.mt6765` | Stock vendor fstab — ghost partition evidence |
| `live_process_map.txt` | `/proc/PID/maps` output for keymaster process |
| `vendor_lib64_hw.txt` | Complete HAL binary inventory |
| `installed_packages_path.txt` | All 127 APKs with full paths |
