# Architectural White Paper
## LOKMAT APPLLP 5 MAX (C17S) — TWRP 11 / Android 10 FBE Decryption

**Lead Engineer:** Albert Pittman (beright1976)  
**Status:** 100% FBE Decryption Achieved — Zero Hardware TEE Reliance  
**Repository:** [github.com/Beright1976/Lokmat5MAX_wiite-c17s_mt6765](https://github.com/Beright1976/Lokmat5MAX_wiite-c17s_mt6765)  

---

## The Premise

The modern standard for TWRP development is "compile and pray." Developers copy vendor blobs into a device tree, build, and patch whatever crashes. Every tutorial, every reference tree, every tool in the ecosystem is built around this workflow. It fails completely on the LOKMAT APPLLP 5 MAX.

The OEM, Topwise/Wiite/Linswear, shipped a device where nearly every standard assumption about the Android crypto stack is wrong. The SoC is a "Franken-SoC" — physical MT6762 silicon running the MT6765 software HAL. The keymaster is pure software but built against ODM-modified libraries that share names with standard libraries and collide fatally with any other version of themselves. The metadata partition is a decoy. The fstab uses a hybrid format that breaks the AOSP parser. The kernel is too old for the TWRP 11 fscrypt API. The HIDL service registry enters an infinite retry loop looking for a HAL that does not exist.

Each of these failures had to be understood at the binary and kernel level before it could be fixed. This document describes exactly what was found and how each problem was solved. All findings are forensically verified against the live device.

---

## 1. The Crypto Isolation Bubble
### Solving: ODM-modified library collision in the Android 11 linker namespace

**The Problem:**

The OEM compiled the entire keymaster and gatekeeper stack against modified versions of `libcrypto.so` (OpenSSL 1.1.1w), `libhidlbase.so`, `libutils.so`, and other common libraries. The modifications are at the binary level — symbol tables differ from both AOSP and standard ODM versions. When these blobs are placed anywhere the Android 11 linker can find them alongside TWRP's own copies of the same libraries, the wrong library loads, symbols misresolve, and the services crash before keymaster code ever executes.

Standard approaches — `PRODUCT_COPY_FILES` to `/system/lib64/`, symlinks, and `TARGET_RECOVERY_DEVICE_MODULES` — all place the blobs in the default linker namespace where collision is inevitable.

**The Discovery:**

Live memory mapping of the running Android 10 OS was the only reliable method for identifying the complete dependency chain. `readelf`, `strings`, and `dlopen` tracing each gave partial pictures. Only mapping `/proc/PID/maps` on the running keymaster process revealed every library actually loaded at runtime, including transitive dependencies the static analysis missed.

**The Solution:**

A root-level containment zone at `/crypto_isolation/` in the ramdisk. The keymaster and gatekeeper service binaries run with `LD_LIBRARY_PATH=/crypto_isolation`, forcing every dependency to resolve exclusively from the bubble. All 26+ ODM blobs have `patchelf --set-rpath /crypto_isolation` applied to enforce the namespace boundary at the ELF RUNPATH level — even if `LD_LIBRARY_PATH` were bypassed, the embedded RUNPATH prevents escape.

**The Critical Detail — Bionic:**

Android's bionic (`libc.so`, `libm.so`, `libdl.so`) is NOT inside the bubble. Bionic is what makes syscalls to the Linux kernel. Loading a second bionic — even the same version — under a different linker causes ABI conflicts in pthread table layouts, `__libc_init` behavior, and TLS structures that crash the process immediately. The bubble contains everything ABOVE bionic. The ODM crypto chain calls standard POSIX symbols (`malloc`, `pthread`, `mmap`) that are stable across Android 10 and 11 at the ABI level. TWRP 11's bionic handles all kernel communication. The ODM chain handles all crypto logic. Neither is aware of the other's existence.

**The Result:**

`android.hardware.keymaster@4.0-service` runs as root in a stable binder loop. `android.hardware.gatekeeper@1.0-service` initializes and registers successfully. Zero signal crashes. Zero linker errors. Both services survive indefinitely.

---

## 2. The VFS Bind Mount — Breaking the Treble Namespace Wall
### Solving: hw_get_module() passthrough HAL discovery across namespace boundaries

**The Problem:**

The gatekeeper service uses `hw_get_module()` to load its passthrough implementation at runtime. This function searches `/vendor/lib64/hw/` for a file named `gatekeeper.default.so`. With the crypto bubble, this file lives at `/crypto_isolation/hw/gatekeeper.default.so`. Symlinks from `/vendor/lib64/hw/` into the bubble were the first attempted solution. They failed.

When vndksupport's passthrough loading code follows a symlink that crosses into a non-vendor path, the Android 11 linker detects the namespace boundary crossing and refuses to load the library with SIGABRT. The linker's Treble enforcement sees "this path is outside /vendor, this is a namespace violation" and aborts.

**The Solution:**

The Linux kernel Virtual File System bind mount. A single init.rc line:

```
mount none /crypto_isolation/hw /vendor/lib64/hw bind
```

This projects the `crypto_isolation/hw/` directory as a real, native filesystem mount at `/vendor/lib64/hw/`. From the kernel's perspective the files are actually at the vendor path — there is no symlink, no redirection, no path that crosses a namespace boundary. The Android 11 linker inspects the path, confirms it is a real `/vendor` directory, and loads the library without triggering any namespace enforcement. The gatekeeper implementation loads from what it sees as a legitimate vendor location while actually executing ODM code from inside the isolation bubble, with all its RUNPATH-enforced dependencies resolving back to `/crypto_isolation/`.

**The Result:**

`gatekeeper.default.so` loads cleanly. The passthrough HAL discovery path completes without namespace violations. Gatekeeper registers on hwbinder and stabilizes.

---

## 3. The VINTF Manifest Engineering
### Solving: hwservicemanager parse failures and the keymaster@3.0 infinite loop

**The Problem (Part 1 — VNDK block):**

The TWRP 11 build system auto-generates a framework manifest containing `<vndk><version>30</version></vndk>`. The `hwservicemanager` on this Android 10 platform cannot parse the `<vndk>` element inside a `type="framework"` manifest and logs a fatal parse error on every HAL registration attempt, corrupting the service registry state.

**The Solution:**

A minimal empty framework manifest deployed to `/system/etc/vintf/manifest.xml`:

```xml
<manifest version="1.0" type="framework">
</manifest>
```

No VNDK block. `hwservicemanager` parses it in one pass and proceeds.

**The Problem (Part 2 — keymaster@3.0 infinite loop):**

After resolving the VNDK issue, TWRP's fscrypt layer entered an infinite hang at splash screen. Forensic analysis identified the cause: the TWRP 11 `Keymaster.cpp` calls `KmDevice::enumerateAvailableDevices()` which internally calls `IKeymasterDevice::getService()` for both `@4.0` and `@3.0` interfaces. The `@4.0` call succeeds. The `@3.0` call blocks indefinitely waiting for a service that does not exist and will never start.

The root cause traced to `android.hardware.keymaster@4.0.so` — the ODM version has a hard `DT_NEEDED` dependency on `android.hardware.keymaster@3.0.so`. This is confirmed present in the stock VNDK-29 version of the same library, meaning it is standard MTK Android 10 behavior, not a Topwise modification. When `enumerateAvailableDevices()` calls `listManifestByInterface` for `@3.0`, `hwservicemanager` finds nothing, sets `foundDefault=false`, and falls through to `getService("default")` — which blocks with a one-second retry loop, permanently freezing the main thread.

**The Solution:**

Declaring `keymaster@3.0` in the device manifest with transport type `passthrough`:

```xml
<hal format="hidl">
    <name>android.hardware.keymaster</name>
    <transport>passthrough</transport>
    <version>3.0</version>
    <interface>
        <name>IKeymasterDevice</name>
        <instance>default</instance>
    </interface>
</hal>
```

`hwservicemanager` now returns immediately for the `@3.0` query with a passthrough result. `enumerateAvailableDevices()` completes. The main thread is unblocked. fscrypt proceeds to key installation.

**The Result:**

`hwservicemanager` parses the framework manifest cleanly. Both keymaster interfaces resolve without blocking. fscrypt proceeds to the key installation phase.

---

## 4. The Ghost Partition — Identifying the Real Metadata Location
### Solving: FBE key material stored on the wrong partition

**The Problem:**

Standard Android 10 MTK devices store FBE key material on the `/metadata` partition. TWRP's fscrypt layer looks for `/metadata`. This device has a partition named "metadata" at `/dev/block/by-name/metadata` (`mmcblk0p11`). Reading it showed all zeros — completely empty. Decryption failed silently every boot.

**The Forensic Discovery:**

Analysis of the stock device's `/proc/mounts` revealed the truth. The stock Android OS mounts `/dev/block/by-name/md_udc` (`mmcblk0p10`) to the `/metadata` path — not the partition literally named "metadata." Cross-referencing against the stock vendor fstab confirmed this: `md_udc` is the deliberate metadata mount point for this OEM. The physical metadata partition (`mmcblk0p11`) is scaffolding — created by the stock Android 10 partition generator as part of a generic partition table template but never written to, never mounted, never used.

The actual FBE key blobs (`keymaster_key_blob`, `encrypted_key`, `secdiscardable`) live on `md_udc`. Every build that mounted `mmcblk0p11` as `/metadata` was reading an empty partition and failing to find any key material. The fscrypt layer had no keys to work with and returned failure with no diagnostic output indicating the root cause.

**The Solution:**

All references to `/dev/block/by-name/metadata` replaced with `/dev/block/by-name/md_udc` throughout the fstab and init.rc. The metadata mount in the init.rc `on-fs` handler explicitly names `md_udc` so no path that follows the partition label can accidentally target the ghost partition.

**The Result:**

`/metadata` mounts on every boot containing real key material. `fscrypt_initialize_systemwide_keys` finds the `keymaster_key_blob` and proceeds to key derivation.

---

## 5. The fstab Dual-Parser Problem
### Solving: libfs_mgr aborting on TWRP-format fstab entries

**The Problem:**

TWRP's fstab supports a device-specific 4-column format with `flags=` syntax:

```
/system ext4 system flags=display="System";logical;backup=1
```

This format is parsed by TWRP's own partition manager. The TWRP 11 fscrypt layer calls `libfs_mgr`'s `ReadDefaultFstab()` independently of TWRP's partition manager. `libfs_mgr` uses a strict POSIX fstab parser that expects exactly 5 columns. When it reads a `flags=` line it calls `strtok_r` for the 5th column (`fs_mgr_options`), receives `NULL` because there are only 4 tokens on that line, and executes `goto err` — aborting the ENTIRE parse, including all valid 5-column crypto entries above it. `fscrypt_initialize_systemwide_keys` receives an empty fstab, cannot find the `fileencryption` policy for `/data`, and fails three times consecutively.

**The Solution:**

Converting the entire fstab to pure AOSP 5-column format. TWRP's own partition manager accepts 5-column format — the `flags=` syntax is optional. Without display names and backup flags, TWRP uses partition mount points as display names and applies default backup behavior. The functional impact is cosmetic only. `libfs_mgr` parses the complete file without errors.

The `fileencryption` flag is specified as `aes-256-xts` only — not `aes-256-xts:aes-256-cts`. The filename encryption mode is set via `ro.crypto.volume.filenames_mode` in the product makefile. Specifying the filename mode in the fstab causes a policy mismatch with the fscrypt superblock written when `/data` was originally formatted by the stock OS, breaking decryption.

**The Result:**

`libfs_mgr` reads the complete fstab without parse errors. `fscrypt_initialize_systemwide_keys` finds the `/data` `fileencryption` policy and the `/metadata` mount point. Key derivation proceeds.

---

## 6. The Kernel 4.9 fscrypt API Mismatch
### Solving: FS_IOC_ADD_ENCRYPTION_KEY does not exist on this kernel

**The Problem:**

TWRP 11 is built against Android 11 headers. The fscrypt layer uses `FS_IOC_ADD_ENCRYPTION_KEY` — a kernel ioctl introduced in Linux 4.14. This device runs kernel 4.9.190+. The ioctl does not exist. When called, the kernel returns `ENOTTY` (inappropriate ioctl for device).

The TWRP 11 fscrypt code in `KeyUtil.cpp` handles this correctly — it calls `isFsKeyringSupported()` which tests for `ENOTTY` and falls back to the legacy `keyctl`/`add_key` path if the ioctl is unavailable. This fallback exists precisely for pre-4.14 kernels. However the fallback requires an "fscrypt" named keyring to exist in the session keyring before key installation is attempted. Without it, `fscryptKeyring()` calls `keyctl_search()`, gets -1, and returns failure.

**The Solution:**

Two components:

1. `TW_USE_FSCRYPT_POLICY := 1` in `BoardConfig.mk` — forces the legacy fscrypt v1 key policy path in the TWRP build
2. `install_keyring` in the `on-fs` init.rc handler — creates the "fscrypt" session keyring that the legacy fallback path requires

**The Result:**

`isFsKeyringSupported()` correctly detects the missing ioctl and falls back to the legacy path. The fscrypt keyring exists in the session keyring. `add_key()` adds the device key with the correct "logon" type and "fscrypt:" prefixed name. `fscrypt_initialize_systemwide_keys` succeeds.

---

## 7. Confirming the Lock — Forensic Credential Analysis
### Why automatic decryption works without user interaction

During debugging, a question arose: if this device uses a proprietary face unlock service as its only lockscreen option, has the standard Android credential system been modified in a way that would prevent TWRP from decrypting automatically?

Forensic analysis of the live device answered this definitively:

- `/data/system/gatekeeper/` does not exist — no credential handle enrolled
- `locksettings.db` shows `lockscreen.disabled=1`, `lockscreen.password_type_alternate=0`
- No synthetic password enrolled under any user ID

The face unlock is a UI layer only. The underlying Android credential system has no enrolled credential. The FBE CE key was wrapped with the default empty credential at format time. TWRP's automatic decryption attempt uses exactly this empty credential. No password prompt is required or shown. Decryption completes silently on every boot.

The iTrustee TEE, RPMB keys, FDE keys, and hardware identity keys found in live memory mapping are all present but none are in the FBE key derivation path. The keymaster chain is confirmed as:

```
libkeymaster4.so → libpuresoftkeymasterdevice.so → libcrypto.so (OpenSSL 1.1.1w) → kernel AES operations
```

Hardware-free at every step except RPMB rollback counter storage.

---

## Conclusion

This build demonstrates that FBE decryption on heavily customized ODM devices is achievable when standard methods fail — but only through direct forensic analysis of the live hardware at the binary, kernel, and filesystem level.

The tools that made this possible were not build flags or reference trees. They were `/proc/PID/maps`, `patchelf`, VFS bind mounts, kernel ioctl analysis, and direct interrogation of SQLite databases on the live device.

Every decision in this device tree is documented with the exact forensic evidence that justified it. The crypto isolation bubble architecture is reproducible on any device where ODM-modified vendor blobs are incompatible with the recovery environment's linker namespace. The ghost partition detection methodology applies to any MTK device where the fstab lies about where key material is stored.

**The vault is open.**

---

*beright1976 | 2026*
