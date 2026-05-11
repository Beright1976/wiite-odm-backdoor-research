# Forensic Protocol — TWRP Build Control Outline

This section contains the device-agnostic protocol developed during the LOKMAT APPLLP 5 MAX (C17S) investigation for building TWRP recovery from zero on any undocumented MTK device.

## The Core Principle

> Raw binary is the only authority for boot image values.  
> Compiled output is the only authority for flash readiness.  
> No device assumptions. No borrowed values.

Every value in a TWRP device tree must be forensically extracted from the physical device — not inferred, not copied from a similar device, not assumed from documentation. This protocol enforces that discipline through a structured fill-in framework with 32 explicit validation rules and a mandatory build gate.

## Document

| Document | Description |
|---|---|
| [OUTLINE.md](OUTLINE.md) | The complete TWRP Build Control Outline — 10 sections, 32 validation rules, pre-build gate, post-build verification, and post-flash hardware checklist. Device-agnostic. Fill every slot with verified device values, pass every rule, build is guaranteed. |

## How to Use This Protocol

1. Start at **Section 0** — record your exact host environment before touching anything else. A passing device config on a broken host does not build.
2. Work through **Sections 1–8** in order — each section has explicit dependencies on the previous one.
3. Run **Section 9** validation rules — all 32 must pass before you execute the make command.
4. Execute **Section 10 Build Gate** — every checkbox must be checked pre-build, post-build, and post-flash.

The status markers are enforced:
- `[ ]` = not yet filled
- `[✓]` = filled and validated
- `[!]` = blocked — dependency not yet resolved

Do not mark `[✓]` on a value you have not verified against physical hardware evidence.

## Section Map

| Section | Content |
|---|---|
| 0 | Build Environment — host OS, packages, git identity, source trees |
| 1 | BoardConfig.mk — architecture, kernel, partitions, offsets, crypto |
| 2 | TWRP Flags — all `TW_` variables, display, input, USB, crypto flags |
| 3 | Kernel — binary identity, driver inventory, FBE crypto primitives |
| 4 | fstab — every partition line, all 5 columns, mount flags, ordering rules |
| 5 | Product Makefile — packages, copy files, property overrides |
| 6 | Blob Audit — primary crypto blobs, transitive dependency tree, ABI matching |
| 7 | Device Tree Files — file inventory, init hook parity, daemon binaries |
| 8 | Hardware Runtime — confirmed values from live device |
| 9 | Validation Rules — 32 rules covering every critical build parameter |
| 10 | Build Gate — pre-build, post-build, and post-flash checklists |

## C17S Completed Example

The filled version of this outline for the LOKMAT APPLLP 5 MAX (C17S) is documented in `03_device_forensics/`. The working TWRP device tree produced by following this protocol is at:

[github.com/Beright1976/Lokmat5MAX_wiite-c17s_mt6765](https://github.com/Beright1976/Lokmat5MAX_wiite-c17s_mt6765)

That build achieves full FBE decryption, zero errors in the recovery log, and complete partition access on a device that had no prior public documentation of any kind.
