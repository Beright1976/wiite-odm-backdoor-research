# TWRP Build — C17S LOKMAT APPLLP 5 MAX

This section documents the complete TWRP 11 build for the LOKMAT APPLLP 5 MAX (C17S). This is a fully functional recovery with 100% FBE decryption, zero errors in the recovery log, and complete partition access on a device that had no prior public documentation of any kind.

## Status

**Complete.** Full FBE decryption achieved. Zero hardware TEE reliance. Zero linker errors. Zero kernel panics.

## Documents

| Document | Description |
|---|---|
| [TWRP_c17s_LOKMAT5MAX_WHITEPAPER.md](TWRP_c17s_LOKMAT5MAX_WHITEPAPER.md) | Architectural white paper — the complete engineering narrative of every problem encountered and exactly how each was solved, from ODM library collision through ghost partition detection to kernel fscrypt API mismatch |

## Device Tree

The working TWRP device tree produced by this build is maintained at:

**[github.com/Beright1976/Lokmat5MAX_wiite-c17s_mt6765](https://github.com/Beright1976/Lokmat5MAX_wiite-c17s_mt6765)**

Clone that repository for the complete device tree including all BoardConfig.mk values, fstab, init.rc files, prebuilt blobs, and crypto isolation bubble configuration.

## Build Outputs

The following outputs were produced by the final verified build:

| File | Description |
|---|---|
| `recovery.img` | The flashable TWRP recovery image — verified against partition size constraint |
| `dtb.img` | Device Tree Blob — embedded in boot image header v2 |
| `ramdisk-recovery.img` | Recovery ramdisk image |
| `ramdisk-recovery.cpio` | Recovery ramdisk CPIO archive — extractable for ramdisk content inspection |
| `ramdisk.img` | Base ramdisk image |
| `kernel` | Prebuilt kernel binary used in the build |

## Seven Engineering Problems Solved

This build required solving seven distinct failures, each requiring forensic analysis at the binary or kernel level before a fix could be designed. The whitepaper covers all seven in full detail. Summary:

| # | Problem | Solution |
|---|---|---|
| 1 | ODM-modified library collision in the Android 11 linker namespace | Crypto isolation bubble at `/crypto_isolation/` with `patchelf --set-rpath` on all 26+ ODM blobs |
| 2 | `hw_get_module()` passthrough HAL discovery failing across namespace boundaries | VFS bind mount projecting `/crypto_isolation/hw/` as a real filesystem mount at `/vendor/lib64/hw/` |
| 3 | `hwservicemanager` parse failure on VNDK block + `keymaster@3.0` infinite retry loop | Minimal empty framework manifest + `keymaster@3.0` declared as passthrough in device manifest |
| 4 | FBE key material on the wrong partition — ghost `metadata` partition | All references replaced with `md_udc` — the real OEM metadata mount point |
| 5 | `libfs_mgr` aborting on TWRP 4-column fstab format | Full conversion to AOSP 5-column fstab format |
| 6 | `FS_IOC_ADD_ENCRYPTION_KEY` ioctl missing on kernel 4.9 | `TW_USE_FSCRYPT_POLICY := 1` + `install_keyring` in init.rc `on-fs` handler |
| 7 | Face unlock preventing automatic decryption (suspected) | Forensic credential analysis confirmed no credential enrolled — empty CE key wrap confirmed |

## Flash Procedure

Flash via mtkclient in BROM mode. Device must be powered off with USB disconnected before entering BROM mode.

```bash
# Flash recovery only
python3 mtk.py w recovery recovery.img

# Verify flash (read back and compare)
python3 mtk.py r recovery recovery_verify.img
md5sum recovery.img recovery_verify.img
```

Do not use SP Flash Tool — mtkclient is the verified method for this device.

## Build Environment Reference

Full build environment specification is in `02_forensic_protocol/OUTLINE.md` Section 0. The completed C17S-specific values for all sections of that outline are in `03_device_forensics/LOKMAT_5_MAX_FORENSICS_REVERSE_ENGINEERING_DATABASE.txt`.
