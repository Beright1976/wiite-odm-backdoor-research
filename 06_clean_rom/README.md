# Clean ROM Build — LOKMAT APPLLP 5 MAX (C17S)

This section documents the clean ROM build for the C17S — a LineageOS 18.1 / AOSP 11 based firmware that strips all ODM backdoor infrastructure while preserving the hardware functionality of the device.

## Status

**In Progress.** Architecture designed and documented. Build not yet complete.

## Documents

| Document | Description |
|---|---|
| [Clean_ROM_Blueprint.md](Clean_ROM_Blueprint.md) | Complete architectural blueprint — all 6 layers of the clean ROM design: kernel, vendor blobs, hardware bridge replacement, framework patches, UI, and the full burn list |

## The Goal

The stock ROM cannot be trusted. The framework has been modified, the permission system has been subverted, and persistent agents with system-level privilege are embedded in read-only partitions that survive factory reset.

The clean ROM build achieves a trusted state by:

- Stripping all ODM backdoor infrastructure at the source — not patching around it
- Rebuilding the nRF52832 hardware bridge as a clean native daemon replacing the privileged ODM agent
- Restoring standard AOSP framework behavior by removing `WiitePackageManagerUtil` and `Configuration.isSpecialApp()` hooks from `services.jar`
- Carrying the crypto isolation bubble architecture forward from the TWRP build — refined for full ROM use
- Removing `pms_sysapp_grant_permission_list.txt` — the ODM backdoor permission registry

## Burn List — What Gets Removed

These paths are excluded from the clean ROM at build time. They do not exist in the output image:

| Path | Threat |
|---|---|
| `/vendor/operator/app/Stopwatch/` | UID 1000 backdoor agent |
| `/product/priv-app/AdupsFota/` | Persistent spyware |
| `/product/priv-app/WearDeviceDeamPix/` | Privileged ODM hardware bridge |
| `/product/app/WearCleanTaskPro/` | Process execution kill-switch |
| `/product/app/WearAppFreeze/` | Application freeze agent |
| `/system/priv-app/HeilsFaceUnlockDM101/` | Face unlock — data scope unverified |
| `/system/etc/permissions/pms_sysapp_grant_permission_list.txt` | Backdoor permission registry |

## Dependency on Previous Sections

The clean ROM build depends on everything that came before it:

- `02_forensic_protocol/` — the discipline that ensures no value is assumed
- `03_device_forensics/` — the single source of truth for all device parameters
- `04_twrp_build/` — the crypto isolation bubble architecture carried forward into the ROM
- `05_rom_dissection/` — the complete inventory of what must be removed and what must be preserved
