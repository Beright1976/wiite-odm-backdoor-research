# Device Forensics — C17S Hardware and Software Database

This section contains the complete forensic database for the LOKMAT APPLLP 5 MAX (C17S). These two files are the single source of truth for every verified parameter of this device. Every value in the TWRP build, every finding in the security report, and every decision in the clean ROM build traces back to an entry in one of these databases.

Nothing in these files is assumed, borrowed from similar devices, or inferred from documentation. Every entry was extracted directly from the physical device via BROM-level binary extraction, live ADB root session, or direct binary analysis.

## Documents

| Document | Description |
|---|---|
| [LOKMAT_5_MAX_FORENSICS_REVERSE_ENGINEERING_DATABASE.txt](LOKMAT_5_MAX_FORENSICS_REVERSE_ENGINEERING_DATABASE.txt) | Hardware forensics database — physical device identity, partition geometry, memory layout, boot image binary analysis, kernel parameters, SoC confirmed values, and all TWRP build results with failure history and resolutions |
| [C17S_SOFTWARE_Forensics.txt](C17S_SOFTWARE_Forensics.txt) | Software forensics database — complete software digital twin of the device across 17 sections covering the full Android stack from bootloader to application layer, including HAL inventory, framework binary analysis, APK inventory (all 127 apps), driver inventory, crypto stack mapping, and security finding evidence |

## Evidence Sources

All entries in these databases derive from one of three authoritative evidence sources:

**Source A — BROM-level binary extraction**
All 42 partitions extracted as individual `.bin` files using mtkclient while the device was in Boot ROM mode. No Android OS involvement — raw flash contents. Stored at `/home/beright1976/lokmat_5_max_stock/`.

**Source B — Live ADB root session**
Direct shell access to the running device with Magisk root and SELinux permissive. Allows reading of protected files, live property inspection, and mount table verification that cannot be faked by the OS reporting layer.

**Source C — Direct binary analysis**
Hex analysis, strings extraction, androguard static analysis, ELF header inspection, and DEX parsing applied directly to extracted partition contents and APK files.

Every database entry that matters includes which source it came from. If a value has no source citation, treat it as unverified.

## How to Use These Databases

### Finding a specific value
Both files are plain text. Use grep:

```bash
grep -n "BOARD_KERNEL_OFFSET" LOKMAT_5_MAX_FORENSICS_REVERSE_ENGINEERING_DATABASE.txt
grep -n "services.jar" C17S_SOFTWARE_Forensics.txt
grep -n "Stopwatch" C17S_SOFTWARE_Forensics.txt
```

### Navigating sections
Both files use `####` section headers. Jump between sections:

```bash
grep -n "^####" C17S_SOFTWARE_Forensics.txt
grep -n "^####" LOKMAT_5_MAX_FORENSICS_REVERSE_ENGINEERING_DATABASE.txt
```

### Cross-referencing with the build
Every value in the TWRP device tree at [Lokmat5MAX_wiite-c17s_mt6765](https://github.com/Beright1976/Lokmat5MAX_wiite-c17s_mt6765) has a corresponding entry in the hardware database. If a build value and a database entry conflict, the database is authoritative — the database reflects the physical device.

## Key Database Entries by Topic

**Boot image geometry**
All confirmed offsets, addresses, and mkbootimg args — hardware database, boot image section.

**Partition table**
All 42 partitions with exact sizes, by-name symlinks, and filesystem types — hardware database, partition section.

**FBE / crypto stack**
Full mapping of the encryption chain from Keymaster through libpuresoftkeymasterdevice — software database, crypto section.

**Security findings evidence**
Raw evidence strings, file paths, and binary artifacts that support every finding in `01_security_findings/` — software database, security section.

**APK inventory**
All 127 pre-installed applications with source partition, privilege level, and security assessment — software database, application layer section.

**HAL inventory**
All hardware abstraction layer binaries with version, ABI, and dependency mapping — software database, HAL section.

**Framework hooks**
Evidence for `WiitePackageManagerUtil` and `Configuration.isSpecialApp()` — software database, framework section.

## Important Notes

- The hardware SoC reports as MT6762V/WB at the BROM level but runs MT6765 HALs at the platform level. This is confirmed intentional ODM configuration — not an error. Both values are correct in their respective contexts.
- `BOARD_TAGS_OFFSET` is confirmed at `0x07880000`. Do not alter this value — it was verified against the stock boot image binary and deviating from it causes boot failure.
- The `--dtb_offset` flag is explicitly excluded from `BOARD_MKBOOTIMG_ARGS`. This is confirmed correct for this device. Do not add it.
