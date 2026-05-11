################################################################################
#
#   TWRP — BUILD CONTROL OUTLINE
#
#   One document. Complete. Device-agnostic.
#   Fill every slot with verified device values.
#   Pass every validation rule.
#   Build is guaranteed.
#
#   Built from TWRP source outward.
#   No device assumptions. No borrowed values.
#   Raw binary is the only authority for boot image values.
#   Compiled output is the only authority for flash readiness.
#
#   STATUS MARKERS:
#   [ ] = not yet filled
#   [✓] = filled and validated
#   [!] = blocked — dependency not yet resolved
#
################################################################################


████████████████████████████████████████████████████████████████████████████████
█  SECTION 0 — BUILD ENVIRONMENT
█  Every host and source variable must be recorded before any other section
█  is filled. A passing device configuration on a broken host does not build.
████████████████████████████████████████████████████████████████████████████████

┌─ 0.1  HOST SYSTEM ───────────────────────────────────────────────────────────┐
│  Record exact versions. Do not approximate.                                  │
└──────────────────────────────────────────────────────────────────────────────┘

  Host OS distribution                 = ___________________________
  Host OS version                      = ___________________________
        (Ubuntu 20.04 LTS is the verified baseline for AOSP 10 builds)
  Kernel uname -r                      = ___________________________
  Available RAM (GB)                   = ___________________________  (16 GB minimum — 32 GB recommended)
  Available disk space (GB)            = ___________________________  (100 GB minimum)
  CPU core count (nproc)               = ___________________________  (used for -j flag)

  Python version (python3 --version)   = ___________________________
  Make version (make --version)        = ___________________________
  Git version (git --version)          = ___________________________
  repo version (repo --version)        = ___________________________
  OpenJDK version (java -version)      = ___________________________  (N/A if not required)

┌─ 0.2  REQUIRED HOST PACKAGES ────────────────────────────────────────────────┐
│  Each package below must be installed and confirmed present.                 │
│  Verify: dpkg -l <package> | grep ^ii                                        │
└──────────────────────────────────────────────────────────────────────────────┘

  build-essential                      [ ] confirmed
  git                                  [ ] confirmed
  git-core                             [ ] confirmed
  gnupg                                [ ] confirmed
  flex                                 [ ] confirmed
  bison                                [ ] confirmed
  gperf                                [ ] confirmed
  zip                                  [ ] confirmed
  curl                                 [ ] confirmed
  zlib1g-dev                           [ ] confirmed
  libc6-dev                            [ ] confirmed
  libncurses5-dev                      [ ] confirmed
  x11proto-core-dev                    [ ] confirmed
  libx11-dev                           [ ] confirmed
  lib32z1-dev                          [ ] confirmed
  libgl1-mesa-dev                      [ ] confirmed
  libxml2-utils                        [ ] confirmed
  xsltproc                             [ ] confirmed
  unzip                                [ ] confirmed
  python3                              [ ] confirmed
  python3-pip                          [ ] confirmed
  python-is-python3                    [ ] confirmed  (or python2.7 — whichever repo requires)
  bc                                   [ ] confirmed
  lzop                                 [ ] confirmed
  pngcrush                             [ ] confirmed
  schedtool                            [ ] confirmed
  rsync                                [ ] confirmed
  libssl-dev                           [ ] confirmed
  ccache                               [ ] confirmed  (optional — record if used)
  binutils                             [ ] confirmed  (provides readelf — required for Section 6 blob audit)
  binutils-aarch64-linux-gnu           [ ] confirmed  (cross-compiled readelf for arm64 blob analysis)
  binutils-arm-linux-gnueabihf         [ ] confirmed  (cross-compiled readelf for arm32 blob analysis)

  CCACHE enabled                       = ___________________________  (yes/no)
  CCACHE size configured               = ___________________________  (GB — if used)

┌─ 0.3  GIT IDENTITY ──────────────────────────────────────────────────────────┐
│  repo requires git identity. Both values must be set globally.               │
└──────────────────────────────────────────────────────────────────────────────┘

  git config --global user.name        = ___________________________
  git config --global user.email       = ___________________________
  git config --global color.ui         = ___________________________  (false — recommended for clean logs)

┌─ 0.4  SOURCE TREES ──────────────────────────────────────────────────────────┐
│  Record exact branch and commit hash at build time for reproducibility.      │
└──────────────────────────────────────────────────────────────────────────────┘

  TWRP manifest URL                    = ___________________________
  TWRP manifest branch                 = ___________________________
  TWRP HEAD commit hash                = ___________________________  (git -C bootable/recovery log --oneline -1)
  AOSP base tag / branch               = ___________________________
  repo sync mirror / manifest source   = ___________________________  (TUNA or other mirror — record URL)
  repo init flags used                 = ___________________________  (--depth, --no-clone-bundle, etc.)

┌─ 0.5  BUILD IDENTITY ────────────────────────────────────────────────────────┐

  lunch combo                          = ___________________________  (twrp[device]-eng)
  Device codename                      = ___________________________
  Device tree path in source           = ___________________________  (device/[manufacturer]/[codename])
  Build timestamp                      = ___________________________

└──────────────────────────────────────────────────────────────────────────────┘


████████████████████████████████████████████████████████████████████████████████
█  SECTION 1 — BOARDCONFIG.MK
█  Every variable the Android build system and TWRP consume from this file.
████████████████████████████████████████████████████████████████████████████████

┌─ 1.1  ARCHITECTURE ──────────────────────────────────────────────────────────┐
│  ABI bitness must be confirmed against the stock init binary before          │
│  filling TARGET_USES_64_BIT_BINDER. See ABI verification procedure below.   │
└──────────────────────────────────────────────────────────────────────────────┘

  TARGET_ARCH                          = ___________________________
  TARGET_ARCH_VARIANT                  = ___________________________
  TARGET_CPU_VARIANT                   = ___________________________
  TARGET_BOARD_PLATFORM                = ___________________________
  TARGET_CPU_ABI                       = ___________________________
  TARGET_CPU_ABI2                      = ___________________________
  TARGET_2ND_ARCH                      = ___________________________
  TARGET_2ND_ARCH_VARIANT              = ___________________________
  TARGET_2ND_CPU_VARIANT               = ___________________________
  TARGET_2ND_CPU_ABI                   = ___________________________
  TARGET_2ND_CPU_ABI2                  = ___________________________

  ── ABI / BINDER BITNESS VERIFICATION ────────────────────────────────────────
  ── Run: file <extracted_stock_init_binary>
  ── The output declares the exact ELF class and architecture.
  ── This value is authoritative — do not infer from kernel bitness alone.
  ── A 64-bit kernel can run a 32-bit Keymaster HAL. If it does, binder
  ── must be configured for 32-bit IPC or the keymaster daemon segfaults
  ── immediately at startup with no useful log output.

  Stock init binary path (extracted)   = ___________________________
  file command output on stock init    = ___________________________
        (example: ELF 64-bit LSB executable, ARM aarch64 — or — ELF 32-bit LSB executable, ARM)

  Kernel bitness                       = ___________________________  (32 / 64)
  Keymaster HAL bitness                = ___________________________  (32 / 64 — from file on HAL binary)
  Binder bitness required              = ___________________________  (matches Keymaster HAL bitness)

  TARGET_USES_64_BIT_BINDER            = ___________________________
        (true  — if Keymaster HAL is 64-bit)
        (false — if Keymaster HAL is 32-bit, even on a 64-bit kernel)
        (see Rule 32)

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 1.2  BOOT IMAGE ────────────────────────────────────────────────────────────┐
│  All values must come from raw binary hex dump of the boot partition.        │
│  Never trust tool output alone. Verify every offset against raw bytes.       │
└──────────────────────────────────────────────────────────────────────────────┘

  BOARD_BOOTIMG_HEADER_VERSION         = ___________________________
  BOARD_KERNEL_BASE                    = ___________________________
  BOARD_KERNEL_OFFSET                  = ___________________________
  BOARD_RAMDISK_OFFSET                 = ___________________________
  BOARD_TAGS_OFFSET                    = ___________________________
  BOARD_KERNEL_PAGESIZE                = ___________________________
  BOARD_DTB_OFFSET                     = ___________________________  (header v2+ — omit if not used)
  BOARD_OS_VERSION                     = ___________________________
  BOARD_OS_PATCH_LEVEL                 = ___________________________

  ── BOOT IMAGE RAW HEX VERIFICATION (fill from xxd of boot partition) ──────
  ── Authoritative header layout — do not reorder, do not skip ───────────────

  kernel_size   raw bytes at offset 0x08-0x0B (little-endian) = _______________
  kernel_addr   raw bytes at offset 0x0C-0x0F (little-endian) = _______________
  ramdisk_size  raw bytes at offset 0x10-0x13 (little-endian) = _______________
  ramdisk_addr  raw bytes at offset 0x14-0x17 (little-endian) = _______________
  second_size   raw bytes at offset 0x18-0x1B (little-endian) = _______________  (N/A if unused)
  second_addr   raw bytes at offset 0x1C-0x1F (little-endian) = _______________  (N/A if unused)
  tags_addr     raw bytes at offset 0x20-0x23 (little-endian) = _______________
  page_size     raw bytes at offset 0x24-0x27 (little-endian) = _______________
  header_ver    raw bytes at offset 0x28-0x2B (little-endian) = _______________

  NOTE: Rules 1–4 in Section 9 compare BOARD_ offsets against these raw values.
        If any raw value above does not match the corresponding BOARD_ value
        via the offset math, the boot image will be misconfigured and not boot.

┌─ 1.3  KERNEL IMAGE ──────────────────────────────────────────────────────────┐

  BOARD_KERNEL_IMAGE_NAME              = ___________________________
        (zImage / Image / Image.gz / Image.gz-dtb / Image-dtb)
  TARGET_PREBUILT_KERNEL               = ___________________________  (path — prebuilt)
  BOARD_INCLUDE_DTB_IN_BOOTIMG         = ___________________________  (true/false)
  BOARD_PREBUILT_DTBIMAGE_DIR          = ___________________________  (path — if separate DTB)
  BOARD_KERNEL_SEPARATED_DTBO          = ___________________________  (true/false)

  BOARD_MKBOOTIMG_ARGS                += ___________________________
  ── Syntax note: use += for each additional argument on its own line.
  ── Example:
  ──   BOARD_MKBOOTIMG_ARGS += --dtb $(TARGET_PREBUILT_DTB)
  ──   BOARD_MKBOOTIMG_ARGS += --header_version $(BOARD_BOOTIMG_HEADER_VERSION)
  ── Do not place all args on one line. Each += appends to the list correctly.

  BOARD_RAMDISK_USE_LZ4                = ___________________________  (true/false — if LZ4 compressed)

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 1.4  RECOVERY PARTITION ────────────────────────────────────────────────────┐

  BOARD_RECOVERYIMAGE_PARTITION_SIZE   = ___________________________  (bytes — from /proc/partitions × 1024)
  TARGET_NO_RECOVERY                   = ___________________________
  BOARD_USES_RECOVERY_AS_BOOT          = ___________________________
  TARGET_RECOVERY_FSTAB                = ___________________________  (path to fstab file in device tree)

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 1.5  PARTITION SCHEME ──────────────────────────────────────────────────────┐
│  Declare A-only or A/B. This determines recovery layout, slot handling,     │
│  and whether BOARD_USES_RECOVERY_AS_BOOT is structurally required.           │
└──────────────────────────────────────────────────────────────────────────────┘

  Partition scheme                     = ___________________________  (A-only / A/B)

  ── IF A/B (slot-based) ──────────────────────────────────────────────────────
  AB_OTA_UPDATER                       = ___________________________  (true — required for A/B)
  AB_OTA_PARTITIONS                   += ___________________________  (boot system vendor product)
  BOARD_USES_RECOVERY_AS_BOOT          = ___________________________  (true — recovery folded into boot on A/B)
  TARGET_NO_RECOVERY                   = ___________________________  (true — no separate recovery partition on A/B)

  ── IF A-only ────────────────────────────────────────────────────────────────
  AB_OTA_UPDATER                       = ___________________________  (false or omit)
  BOARD_USES_RECOVERY_AS_BOOT          = ___________________________  (false — separate recovery partition)

  NOTE: A/B and A-only are mutually exclusive structures. Mixing values from
        both produces a structurally inconsistent build that will not flash
        or boot correctly. Declare one scheme and apply it consistently.

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 1.6  DYNAMIC PARTITIONS ────────────────────────────────────────────────────┐

  BOARD_SUPER_PARTITION_SIZE           = ___________________________  (bytes)
  BOARD_SUPER_PARTITION_GROUPS         = ___________________________  (group name e.g. main)
  BOARD_[GROUP]_SIZE                   = ___________________________  (bytes — max size of group)
  BOARD_[GROUP]_PARTITION_LIST         = ___________________________  (e.g. system vendor product)
  BOARD_USES_METADATA_PARTITION        = ___________________________

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 1.7  FILESYSTEM ────────────────────────────────────────────────────────────┐

  TARGET_USERIMAGES_USE_EXT4           = ___________________________
  TARGET_USERIMAGES_USE_F2FS           = ___________________________
  BOARD_USERDATAIMAGE_FILE_SYSTEM_TYPE = ___________________________

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 1.8  SECURITY — BOOTLOADER GATE ────────────────────────────────────────────┐
│  A bit-perfect TWRP image is useless if the bootloader rejects it.          │
│  Every field in this section must be filled and verified before flash.       │
│  The bootloader gate is a pre-flash hard stop — not a post-flash debug step.│
└──────────────────────────────────────────────────────────────────────────────┘

  ── AVB CONFIGURATION ────────────────────────────────────────────────────────

  BOARD_AVB_ENABLE                     = ___________________________  (true/false)
  BOARD_VNDK_VERSION                   = ___________________________

  AVB version in use                   = ___________________________  (AVB 1.0 / AVB 2.0 / none)
        (confirm from stock vbmeta partition: avbtool info_image --image vbmeta.img)

  ── OEM TRUST CHAIN ──────────────────────────────────────────────────────────
  ── Identify which OEM-specific trust enforcement is active on this device. ──
  ── Each mechanism has different unlock requirements and flash rules.        ──

  OEM trust mechanism                  = ___________________________
        (none / Samsung Knox / Xiaomi Anti-Rollback / Google Titan /
         MediaTek SBC / OPPO/OnePlus Ops / other — specify)

  Anti-rollback protection active      = ___________________________  (yes/no)
  Anti-rollback index (if active)      = ___________________________
        (confirm from: avbtool info_image --image boot.img | grep rollback)
  Rollback index in TWRP build         = ___________________________
        (must be ≥ stock rollback index — lower value will be permanently rejected)

  ── BOOTLOADER UNLOCK STATE ──────────────────────────────────────────────────

  Bootloader unlock method             = ___________________________
        (fastboot flashing unlock / OEM unlock / EDL / other — specify)
  Bootloader unlock state confirmed    = ___________________________  (yes — confirmed via fastboot getvar unlocked)
  fastboot getvar unlocked output      = ___________________________  (must return: unlocked: yes)

  ── VBMETA FLAGS ─────────────────────────────────────────────────────────────
  ── Required when bootloader is unlocked but AVB enforcement remains active. ─
  ── These flags disable verity and verification to allow unsigned execution. ─

  vbmeta flash strategy                = ___________________________
        (A) bootloader fully unlocked — ignores all signatures — no vbmeta modification needed
        (B) AVB active — must flash modified vbmeta with verification disabled

  ── IF STRATEGY (B) ──────────────────────────────────────────────────────────
  vbmeta flash command                 = ___________________________
        (fastboot flash vbmeta vbmeta.img with --disable-verity --disable-verification flags)
        (exact command: fastboot --disable-verity --disable-verification flash vbmeta vbmeta.img)
  vbmeta_system partition present      = ___________________________  (yes/no — flash separately if yes)
  vbmeta_vendor partition present      = ___________________________  (yes/no — flash separately if yes)

  ── SIGNATURE VERIFICATION ───────────────────────────────────────────────────

  Custom signing key required          = ___________________________  (yes/no)
  Signing key path                     = ___________________________  (N/A if bootloader ignores signatures)
  TWRP image signed                    = ___________________________  (yes/no/N/A)
  Signature verification command       = ___________________________

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 1.9  PLATFORM ──────────────────────────────────────────────────────────────┐

  PLATFORM_VERSION                     = ___________________________
  PLATFORM_SECURITY_PATCH              = ___________________________
  PLATFORM_VERSION_LAST_STABLE         = ___________________________

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 1.10 KERNEL CMDLINE ────────────────────────────────────────────────────────┐
│  Exact string from live /proc/cmdline. Copy verbatim. Add USB controller.   │
│  Use line continuation (\) for multi-line values in the Makefile.            │
└──────────────────────────────────────────────────────────────────────────────┘

  BOARD_KERNEL_CMDLINE                 = ___________________________
                                         ___________________________
                                         ___________________________
                                         ___________________________
                                         ___________________________

  USB controller addition              = androidboot.usbcontroller=_______________
  androidboot.hardware value           = ___________________________
        (must match TARGET_BOARD_PLATFORM exactly — see Rule 29)

┌─ 1.11 ADDITIONAL BUILD FLAGS ────────────────────────────────────────────────┐
│  Set these only if the device requires them. Record N/A if not applicable.  │
└──────────────────────────────────────────────────────────────────────────────┘

  BOARD_FLASH_BLOCK_SIZE               = ___________________________
        (must be a power of 2 and ≥ BOARD_KERNEL_PAGESIZE — see Rule 31)
        (typically: BOARD_KERNEL_PAGESIZE × 64)

  BOARD_ROOT_EXTRA_FOLDERS             = ___________________________  (space-separated paths)
        (N/A if no extra mount points needed in recovery root)

  BOARD_ROOT_EXTRA_SYMLINKS            = ___________________________  (target:link pairs)
        (N/A if no symlinks needed)

  TARGET_COPY_OUT_VENDOR               = ___________________________  (vendor — if Soong vendor staging needed)

  BUILD_BROKEN_DUP_RULES               = ___________________________  (true — only if build system requires it)

└──────────────────────────────────────────────────────────────────────────────┘


████████████████████████████████████████████████████████████████████████████████
█  SECTION 2 — TWRP FLAGS  (TW_ variables)
█  Every flag defined in TWRP source. Each controls a specific capability.
█  Fill or explicitly decide every entry. No skipping.
████████████████████████████████████████████████████████████████████████████████

┌─ 2.1  DISPLAY ───────────────────────────────────────────────────────────────┐

  TW_THEME                             = ___________________________
        (portrait_hdpi / portrait_mdpi / landscape_hdpi / landscape_mdpi / watch_mdpi)
  TARGET_SCREEN_WIDTH                  = ___________________________  (pixels)
  TARGET_SCREEN_HEIGHT                 = ___________________________  (pixels)
  TARGET_RECOVERY_PIXEL_FORMAT         = ___________________________
        (RGBX_8888 / RGBA_8888 / RGB_565 / BGRA_8888)
  TW_BRIGHTNESS_PATH                   = ___________________________  (full sysfs path)
  TW_MAX_BRIGHTNESS                    = ___________________________  (integer — from sysfs)
  TW_DEFAULT_BRIGHTNESS                = ___________________________  (integer — safe usable value)
  TW_SCREEN_BLANK_ON_BOOT              = ___________________________  (true/false)
  TW_NO_SCREEN_BLANK                   = ___________________________  (true/false)
  TW_NO_SCREEN_TIMEOUT                 = ___________________________  (true/false)
  TW_INCLUDE_FB2PNG                    = ___________________________  (true/false)

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 2.2  TOUCH INPUT ───────────────────────────────────────────────────────────┐
│  Confirm axis orientation by running getevent on device. Do not guess.      │
└──────────────────────────────────────────────────────────────────────────────┘

  TW_TOUCHSCREEN_SWAP_XY               = ___________________________  (true/false)
  TW_TOUCHSCREEN_FLIP_X                = ___________________________  (true/false)
  TW_TOUCHSCREEN_FLIP_Y                = ___________________________  (true/false)
  TW_INPUT_BLACKLIST                   = ___________________________  (pipe-separated node names)

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 2.3  CRYPTO / FBE ──────────────────────────────────────────────────────────┐

  TW_INCLUDE_CRYPTO                    = ___________________________
  TW_INCLUDE_CRYPTO_FBE                = ___________________________
  TW_INCLUDE_FBE_METADATA_DECRYPT      = ___________________________
  TW_CRYPTO_USE_SYSTEM_VOLD            = ___________________________  (TWRP version dependent)
  TW_CRYPTO_SYSTEM_VOLD_MOUNT          = ___________________________  (path — if using system vold)
  TW_CRYPTO_SYSTEM_VOLD_DEBUG          = ___________________________  (true/false)
  TW_FORCE_KEYMASTER_VER               = ___________________________  (integer — if needed)

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 2.4  STORAGE ───────────────────────────────────────────────────────────────┐

  TW_INCLUDE_NTFS_3G                   = ___________________________
  TW_INCLUDE_EXFAT                     = ___________________________
  RECOVERY_SDCARD_ON_DATA              = ___________________________
  TW_EXTERNAL_STORAGE_PATH             = ___________________________
  TW_EXTERNAL_STORAGE_MOUNT_POINT      = ___________________________
  TW_DEFAULT_EXTERNAL_STORAGE          = ___________________________
  TW_NO_USB_STORAGE                    = ___________________________
  TW_HAS_NO_RECOVERY_PARTITION         = ___________________________

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 2.5  PLATFORM BEHAVIOR ─────────────────────────────────────────────────────┐

  TW_HAS_DOWNLOAD_MODE                 = ___________________________
        ── SAMSUNG-SPECIFIC FLAG. Set true only if this device uses Samsung
        ── Download Mode (entered via vol-down+home+power or equivalent) for
        ── Odin-based flashing. Set false or omit entirely on all non-Samsung
        ── devices. Setting this true on a non-Samsung device does nothing
        ── harmful but is incorrect configuration noise.

  TW_FORCE_CPUINFO_FOR_DEVICE_ID       = ___________________________
  TW_EXCLUDE_SUPERSU                   = ___________________________
  TW_EXCLUDE_DEFAULT_USB_INIT          = ___________________________
        ── Set true on platforms that do NOT use Qualcomm USB init scripts.
        ── Most non-Qualcomm platforms (MTK, Samsung Exynos, UNISOC, etc.)
        ── require this set to true. Qualcomm platforms typically omit this
        ── flag or set it false. Verify by confirming the USB controller
        ── driver in Section 3.2 is built-in and cross-referencing the
        ── platform vendor's USB init path.

  BOARD_SUPPRESS_SECURE_ERASE          = ___________________________
        ── Set true on platforms where the secure erase ioctl causes the
        ── recovery process to hang during wipe operations. This is not
        ── platform-specific by design — it must be determined by testing
        ── wipe on real hardware. If a wipe operation hangs indefinitely,
        ── set this flag and rebuild. Platforms known to require it include
        ── many eMMC-based devices regardless of SoC vendor.

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 2.6  HARDWARE PATHS ────────────────────────────────────────────────────────┐
│  Only set if the default sysfs path does not exist on this platform.        │
└──────────────────────────────────────────────────────────────────────────────┘

  TW_CUSTOM_CPU_TEMP_PATH              = ___________________________  (full sysfs path / N/A)
  TW_CUSTOM_BATTERY_PATH               = ___________________________  (full sysfs path / N/A)

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 2.7  TOOLS ─────────────────────────────────────────────────────────────────┐

  TW_USE_TOOLBOX                       = ___________________________
  TW_EXTRA_LANGUAGES                   = ___________________________
  TW_INCLUDE_RESETPROP                 = ___________________________
  TW_INCLUDE_LIBRESETPROP              = ___________________________
  TW_INCLUDE_REPACKTOOLS               = ___________________________
  TW_INCLUDE_BASH                      = ___________________________

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 2.8  DYNAMIC PARTITION TOOLS ───────────────────────────────────────────────┐

  TW_INCLUDE_LPTOOLS                   = ___________________________  (true — required for dynamic partition resizing)
  TW_INCLUDE_FASTBOOTD                 = ___________________________  (true/false)
  TW_INCLUDE_LPDUMP                    = ___________________________  (true/false — enables lpdump for partition debugging)
  TW_INCLUDE_LPMAKE                    = ___________________________  (true/false — enables lpmake for partition creation)

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 2.9  LIBRARY LINKING ───────────────────────────────────────────────────────┐

  TW_RECOVERY_ADDITIONAL_RELINK_LIBRARY_FILES += ___________________________
                                                  ___________________________
                                                  ___________________________
        (N/A if no additional library relinking required)

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 2.10 KERNEL MODULE LOADING ─────────────────────────────────────────────────┐

  TW_LOAD_VENDOR_MODULES               = ___________________________  (space-separated .ko paths / N/A)

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 2.11 HAPTICS ───────────────────────────────────────────────────────────────┐

  TW_NO_HAPTICS                        = ___________________________  (true — if haptic driver absent or causes crash)
  TW_HAPTICS_TSPDRV                    = ___________________________  (true — only for Immersion TSPDRV haptic IC)

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 2.12 VERSIONING ────────────────────────────────────────────────────────────┐

  TW_DEVICE_VERSION                    = ___________________________

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 2.13 PLATFORM-SPECIFIC FLAGS ───────────────────────────────────────────────┐
│  These flags apply only to specific SoC families. Fill the ones that apply  │
│  to this device. Mark N/A for all others. Do not omit this section.         │
└──────────────────────────────────────────────────────────────────────────────┘

  ── QUALCOMM-SPECIFIC ────────────────────────────────────────────────────────

  TARGET_RECOVERY_QCOM_RTC_FIX         = ___________________________  (true/false / N/A)
        ── Required on some Qualcomm devices where the RTC clock is not
        ── initialized at recovery boot, causing incorrect system time.
        ── Symptom: certificates fail to validate, SSL operations fail.
        ── Set true if device is Qualcomm-based and exhibits this symptom.
        ── Set N/A on all non-Qualcomm platforms.

  RECOVERY_GRAPHICS_USE_LINELENGTH     = ___________________________  (true/false / N/A)
        ── Required on some Qualcomm devices where the framebuffer stride
        ── (line length) differs from width × bytes-per-pixel. Symptom:
        ── display renders with a horizontal offset or corrupted columns.
        ── Set N/A on all non-Qualcomm platforms.

  ── SAMSUNG-SPECIFIC ─────────────────────────────────────────────────────────

  TW_SAMSUNG_TAR_BACKUP                = ___________________________  (true/false / N/A)
        ── Enables Samsung-style tar-based NANDroid backup format.
        ── Set N/A on all non-Samsung devices.

  ── ANY PLATFORM — ADDITIONAL ────────────────────────────────────────────────

  Additional platform-specific flag 1  = ___________________________
  Additional platform-specific flag 2  = ___________________________
        (document any flag not listed above that the device tree requires,
         with the reason it is required)

└──────────────────────────────────────────────────────────────────────────────┘


████████████████████████████████████████████████████████████████████████████████
█  SECTION 3 — KERNEL
████████████████████████████████████████████████████████████████████████████████

┌─ 3.1  KERNEL BINARY ─────────────────────────────────────────────────────────┐

  Strategy                             = ___________________________  (prebuilt / source)
  Kernel binary path                   = ___________________________
  Kernel image format                  = ___________________________
  DTB strategy                         = ___________________________  (appended / separate / DTBO only)
  DTB path (if separate)               = ___________________________

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 3.2  REQUIRED KERNEL CAPABILITIES ──────────────────────────────────────────┐
│  Every item below must be built-in (=y) or confirmed present in the kernel.  │
│  Modules (=m) are NOT sufficient for items marked built-in required.         │
│  Verify each against /proc/config.gz or the kernel .config used to build.   │
└──────────────────────────────────────────────────────────────────────────────┘

  ── HARDWARE DRIVERS ─────────────────────────────────────────────────────────

  Display panel driver                 [ ] present  name=___________________________
  Framebuffer driver                   [ ] present  name=___________________________
  Touch controller driver              [ ] present  name=___________________________
  USB controller — BUILT-IN NOT MOD   [ ] present  name=___________________________
  Battery / fuel gauge driver          [ ] present  name=___________________________
  Charger IC driver                    [ ] present  name=___________________________
  Platform coprocessor driver          [ ] present / [ ] N/A  name=___________________________
        ── Not all platforms have a coprocessor that requires a driver present
        ── at recovery boot. Fill if applicable. Mark N/A if the platform
        ── has no coprocessor or its driver is not required for recovery.
  Any other platform-specific driver   [ ] present  name=___________________________

  ── FILESYSTEMS ──────────────────────────────────────────────────────────────

  ext4 filesystem                      [ ] present
  f2fs filesystem                      [ ] present  (required if userdata is f2fs)
  SquashFS                             [ ] present  (required if any partition uses squashfs)

  ── DEVICE MAPPER / CRYPTO LAYER ─────────────────────────────────────────────

  Device mapper (dm)                   [ ] present
  dm-crypt                             [ ] present  (required for FDE / FBE crypto layer)
  Overlay filesystem                   [ ] present

  ── FBE KERNEL SUPPORT ───────────────────────────────────────────────────────
  ── All items in this block are required for FBE to function.               ──
  ── A missing item here produces silent /data decrypt failure — not a       ──
  ── kernel panic. The device boots, TWRP loads, and /data fails to mount.  ──

  CONFIG_FS_ENCRYPTION                 [ ] present  (core FBE kernel support — =y mandatory)
  CONFIG_FHANDLE                       [ ] present  (vold file handle operations — =y mandatory)
  CONFIG_KEYS                          [ ] present  (kernel keyring — =y mandatory)
  CONFIG_ENCRYPTED_KEYS                [ ] present  (encrypted keyring type — =y mandatory)

  ── INLINE ENCRYPTION ────────────────────────────────────────────────────────

  CONFIG_BLK_INLINE_ENCRYPTION         [ ] present / [ ] N/A
  CONFIG_BLK_INLINE_ENCRYPTION_FALLBACK [ ] present / [ ] N/A

  ── CRYPTO PRIMITIVES ────────────────────────────────────────────────────────

  CONFIG_CRYPTO_AES                    [ ] present
  CONFIG_CRYPTO_AES_XTS                [ ] present
  CONFIG_CRYPTO_SHA256                 [ ] present
  CONFIG_CRYPTO_SHA512                 [ ] present
  CONFIG_CRYPTO_HMAC                   [ ] present
  FBE crypto cipher in use             [ ] present  cipher=_________________________

  ── SYSTEM SERVICES SUPPORT ──────────────────────────────────────────────────

  configfs (USB gadget)                [ ] present
  SELinux                              [ ] present  mode=___________________________
  CONFIG_CGROUPS                       [ ] present
  CONFIG_NAMESPACES                    [ ] present

└──────────────────────────────────────────────────────────────────────────────┘


████████████████████████████████████████████████████████████████████████████████
█  SECTION 4 — TWRP FSTAB
█
█  TWRP fstab format (5 columns, all required):
█  <block device>  <mount point>  <fs type>  <mount flags>  <twrp flags>
█
█  TWRP flags (column 5) — key ones:
█    display="Name"          label in TWRP UI
█    backup=1 / backup=0     include / exclude from NANDroid
█    wipeingui               show in wipe menu
█    canbewiped              allow wipe operation
█    wipeduringfactoryreset  wipe on factory reset
█    removable               external / removable media
█    logical                 dynamic partition (Android 10+)
█    fileencryption=         FBE type — must match stock fstab exactly
█    forceencrypted=         force encryption flag path — must match stock fstab exactly
█    voldmanaged=            managed external storage
█    fsflags=""              additional filesystem mount flags
█
█  HOW TO DETERMINE WHAT PARTITIONS THIS DEVICE HAS:
█  1. Boot the stock OS. Run: ls -la /dev/block/by-name/
█     This lists every partition the bootloader exposes by name.
█  2. Cross-reference with /proc/partitions for sizes.
█  3. For each partition, determine: does TWRP need to read, write, back up,
█     or mount it? If yes, add a fstab entry. If no, omit it.
█  4. Pull the stock fstab from the running device:
█       adb pull /vendor/etc/fstab.<platform> .
█     This is the authoritative source for fs type, mount flags,
█     fileencryption=, and forceencrypted= values. Copy those values
█     exactly — do not retype them.
█
████████████████████████████████████████████████████████████████████████████████

  BASE BLOCK DEVICE PATH
  ── Two path forms are in use across devices. Confirm which applies.
  ── Form A (older, platform-specific): /dev/block/platform/<soc-id>/by-name/
  ── Form B (newer, Android 10+):       /dev/block/by-name/
  ── Confirm by running: ls /dev/block/platform/ on the device.
  ── If the platform directory exists and contains by-name/, use Form A.
  ── If not, use Form B.

  Block device base path               = ___________________________
        (full prefix up to and including by-name/ — all entries below are suffixed to this)

  ── DYNAMIC LOGICAL PARTITIONS ────────────────────────────────────────────────
  ── These are universally required on Android 10+ devices with dynamic       ──
  ── partitions. The logical flag tells TWRP to use dm-linear to map them.   ──

  /system
    block   = ___________________________
    type    = ext4
    mflags  = ro
    twrp    = logical,display="System",backup=1

  /vendor
    block   = ___________________________
    type    = ext4
    mflags  = ro
    twrp    = logical,display="Vendor",backup=1

  /product
    block   = ___________________________  (confirm present — not all devices have a product partition)
    type    = ext4
    mflags  = ro
    twrp    = logical,display="Product",backup=1

  /odm
    block   = ___________________________  (confirm present — omit entry if this partition does not exist)
    type    = ext4
    mflags  = ro
    twrp    = logical,display="ODM",backup=1

  ── FBE SUPPORT ───────────────────────────────────────────────────────────────
  ── /metadata MUST APPEAR BEFORE /data IN FSTAB — failure to order these    ──
  ── correctly causes FBE metadata init to fail silently.                    ──

  /metadata
    block   = ___________________________
    type    = ext4
    mflags  = ___________________________
    twrp    = display="Metadata",backup=0,wipeduringfactoryreset

  /data
    block   = ___________________________
    type    = ext4
    mflags  = ___________________________
    twrp    = display="Data",wipeingui,forceencrypted=___________________________
    FBE     = fileencryption=___________________________  ← copy exactly from stock fstab
    ── Both forceencrypted= and fileencryption= values must match stock exactly.
    ── See Rules 9 and 9b. Copy from stock — do not retype.

  ── WRITABLE PARTITIONS ───────────────────────────────────────────────────────

  /cache
    block   = ___________________________
    type    = ext4
    mflags  = ___________________________
    twrp    = display="Cache",backup=1,wipeingui,canbewiped
    ── NOTE: /cache does not exist on A/B devices. If partition scheme
    ── (Section 1.5) is A/B, omit this entry entirely. Confirm by checking
    ── ls /dev/block/by-name/ on the device — if no cache entry exists,
    ── this partition is absent and must not appear in the fstab.

  ── CRITICAL PLATFORM PARTITIONS — NEVER WIPE ─────────────────────────────────
  ── Some partitions, if wiped, cause permanent hardware damage that cannot  ──
  ── be recovered without factory service. These must have ro mount flag,    ──
  ── canbewiped=0, and wipeingui=0. The specific partitions vary by device.  ──
  ── Common examples by platform:                                             ──
  ──                                                                          ──
  ──   Samsung:  /efs — contains IMEI and baseband calibration data.         ──
  ──             Wiping /efs permanently bricks the modem. Must be ro,        ──
  ──             canbewiped=0, wipeingui=0. Back up=1 for NANDroid.          ──
  ──                                                                          ──
  ──   MediaTek: /mnt/vendor/protect_f, /mnt/vendor/protect_s — contain     ──
  ──             security certificates and DRM keys. ro, canbewiped=0.       ──
  ──             /mnt/vendor/nvdata, /mnt/vendor/nvcfg — RF calibration      ──
  ──             and NV configuration. ro, canbewiped=0.                     ──
  ──             /nvram — NVRAM partition. ro, canbewiped=0.                 ──
  ──                                                                          ──
  ──   Qualcomm: /persist — contains sensor calibration, DRM licenses,       ──
  ──             and Wi-Fi MAC address on some devices. ro, canbewiped=0.    ──
  ──             /modem — contains baseband firmware. Never wipe. ro.        ──
  ──                                                                          ──
  ── Identify which of these apply to this device by inspecting the stock    ──
  ── fstab and /dev/block/by-name/ output. Add fstab entries for every      ──
  ── partition present that falls into this category.                         ──

  /___________   (critical partition — fill from device)
    block   = ___________________________
    type    = ___________________________
    mflags  = ro
    twrp    = display="___________",backup=1,canbewiped=0,wipeingui=0

  /___________   (critical partition — fill from device)
    block   = ___________________________
    type    = ___________________________
    mflags  = ro
    twrp    = display="___________",backup=1,canbewiped=0,wipeingui=0

  /___________   (critical partition — fill from device)
    block   = ___________________________
    type    = ___________________________
    mflags  = ro
    twrp    = display="___________",backup=1,canbewiped=0,wipeingui=0

  /___________   (critical partition — fill from device)
    block   = ___________________________
    type    = ___________________________
    mflags  = ro
    twrp    = display="___________",backup=1,canbewiped=0,wipeingui=0
        (add additional rows as needed — one per critical partition present on device)

  ── EMMC PARTITIONS (NANDroid targets) ────────────────────────────────────────
  ── These partitions exist as raw emmc regions rather than mounted           ──
  ── filesystems. Confirm each is present via /dev/block/by-name/ before     ──
  ── adding an entry. Record N/A for any partition not present on device.    ──

  /boot          block=___________________________  type=emmc  twrp=display="Boot",backup=1
  /recovery      block=___________________________  type=emmc  twrp=display="Recovery",backup=1
        (omit /recovery on A/B devices — recovery partition does not exist)
  /vbmeta        block=___________________________  type=emmc  twrp=display="VBMeta",backup=1
  /vbmeta_system block=___________________________  type=emmc  twrp=display="VBMeta System",backup=1
        (confirm present — not universal — N/A if absent)
  /vbmeta_vendor block=___________________________  type=emmc  twrp=display="VBMeta Vendor",backup=1
        (confirm present — not universal — N/A if absent)
  /dtbo          block=___________________________  type=emmc  twrp=display="DTBO",backup=1
        (confirm present — N/A if absent)
  /misc          block=___________________________  type=emmc  twrp=display="Misc",backup=1
        (confirm present — N/A if absent)

  ── ADDITIONAL PLATFORM PARTITIONS ────────────────────────────────────────────
  ── Fill from: ls /dev/block/by-name/ on device.                            ──
  ── For each partition not yet listed above that TWRP should back up or     ──
  ── manage, add a row here. Confirm type and flags from stock fstab.        ──

  /___________   block=___________________________  type=_____  twrp=display="___________",backup=___
  /___________   block=___________________________  type=_____  twrp=display="___________",backup=___
  /___________   block=___________________________  type=_____  twrp=display="___________",backup=___
  /___________   block=___________________________  type=_____  twrp=display="___________",backup=___
  /___________   block=___________________________  type=_____  twrp=display="___________",backup=___

  ── EXTERNAL STORAGE ──────────────────────────────────────────────────────────

  SD card
    block   = ___________________________
    mount   = ___________________________
    type    = auto
    twrp    = display="MicroSD",removable,storage,wipeingui,voldmanaged=___________________________

  USB OTG
    block   = ___________________________
    mount   = ___________________________
    type    = auto
    twrp    = display="USB OTG",removable,storage,wipeingui,voldmanaged=___________________________


████████████████████████████████████████████████████████████████████████████████
█  SECTION 5 — PRODUCT MAKEFILE
████████████████████████████████████████████████████████████████████████████████

  PRODUCT_NAME                         = ___________________________
  PRODUCT_DEVICE                       = ___________________________
  PRODUCT_BRAND                        = ___________________________
  PRODUCT_MODEL                        = ___________________________
  PRODUCT_MANUFACTURER                 = ___________________________
  PRODUCT_RELEASE_NAME                 = ___________________________

  Device tree directory                = ___________________________
  vendorsetup.sh lunch combo           = twrp____________________________-eng

  ── BLOB WIRING ───────────────────────────────────────────────────────────────

  PRODUCT_PACKAGES                    += ___________________________
  PRODUCT_COPY_FILES                  += ___________________________
                                         ___________________________
                                         ___________________________
                                         ___________________________
  PRODUCT_SOONG_NAMESPACES            += ___________________________

  ── PROPERTY OVERRIDES ────────────────────────────────────────────────────────

  PRODUCT_PROPERTY_OVERRIDES          += ___________________________
                                         ___________________________
                                         ___________________________
        (N/A if no property overrides required)


████████████████████████████████████████████████████████████████████████████████
█  SECTION 6 — CRYPTO BLOBS
█  What TWRP needs from the device vendor partition to decrypt FBE.
█
█  Section 6 has two parts:
█  6A — the known blob inventory (fill from device)
█  6B — the transitive dependency audit (mandatory procedure — run before build)
█
█  6B is not optional. A missing transitive dependency produces identical
█  symptom to a missing primary blob: silent decrypt failure. The device
█  boots, TWRP loads, /data does not mount. No error is surfaced to the UI.
████████████████████████████████████████████████████████████████████████████████

┌─ 6A  KNOWN BLOB INVENTORY ───────────────────────────────────────────────────┐

  KEYMASTER
  ────────────────────────────────────────────────────────────────────────────
  Keymaster version                    = ___________________________
  Keymaster HAL service binary         = ___________________________
  Keymaster HAL path in vendor         = ___________________________

  REQUIRED LIBRARIES
  ────────────────────────────────────────────────────────────────────────────
  libkeymaster[ver].so                 [ ] confirmed  path=___________________________
  libpuresoftkeymasterdevice.so        [ ] confirmed / [ ] N/A
        (present only on devices with software keymaster fallback — confirm
         by checking vendor/lib64/ on device — N/A if hardware TEE only)
        path=___________________________
  libkeymaster_portable.so             [ ] confirmed  path=___________________________
  libkeymaster_messages.so             [ ] confirmed  path=___________________________
  libcrypto.so                         [ ] confirmed  path=___________________________  ver=_______
  libkeymaster[ver]support.so          [ ] confirmed  path=___________________________

  LINKER STRATEGY
  ────────────────────────────────────────────────────────────────────────────
  How blobs are exposed to TWRP        = ___________________________
  Destination path in build            = ___________________________

  VOLD (if TW_CRYPTO_USE_SYSTEM_VOLD = true)
  ────────────────────────────────────────────────────────────────────────────
  vold binary source path              = ___________________________
  Additional HAL libraries required    = ___________________________

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 6B  TRANSITIVE DEPENDENCY AUDIT ────────────────────────────────────────────┐
│  MANDATORY PROCEDURE — run before build — for every blob in Section 6A.     │
│                                                                              │
│  A keymaster .so may depend on a proprietary OEM library that itself         │
│  depends on another library. The build system does not detect missing        │
│  transitive dependencies. Every leaf of the dependency tree must exist       │
│  in TARGET_RECOVERY_ROOT_OUT or decryption silently fails.                  │
│                                                                              │
│  PROCEDURE:                                                                  │
│  1. For each blob listed in 6A, run:                                         │
│       readelf -d <blob.so> | grep NEEDED                                     │
│     Use the cross-compiled readelf for the correct ABI:                      │
│       aarch64-linux-gnu-readelf  (for arm64 blobs)                           │
│       arm-linux-gnueabihf-readelf (for arm32 blobs)                          │
│  2. Record every NEEDED library returned.                                    │
│  3. For each NEEDED library, check if it is:                                 │
│       (A) a standard Android system library — confirm present in TWRP        │
│       (B) a proprietary OEM library — must be added to blob inventory        │
│  4. For every proprietary library found in step 3(B), repeat from step 1.   │
│  5. Iterate until no new unresolved dependencies are found.                  │
│  6. Every proprietary library in the resolved tree must be:                  │
│       - copied into the build via PRODUCT_COPY_FILES                         │
│       - present in TARGET_RECOVERY_ROOT_OUT before flash                     │
│                                                                              │
│  TOOL NOTE: ldd does not work on cross-architecture binaries.                │
│  readelf -d is the correct tool. Do not substitute.                          │
└──────────────────────────────────────────────────────────────────────────────┘

  ── TRANSITIVE DEPENDENCY TREE (fill as audit proceeds) ──────────────────────

  Blob: ___________________________
    NEEDED: ___________________________  [ ] system lib  [ ] proprietary — resolved path: _______________
    NEEDED: ___________________________  [ ] system lib  [ ] proprietary — resolved path: _______________
    NEEDED: ___________________________  [ ] system lib  [ ] proprietary — resolved path: _______________

  Blob: ___________________________
    NEEDED: ___________________________  [ ] system lib  [ ] proprietary — resolved path: _______________
    NEEDED: ___________________________  [ ] system lib  [ ] proprietary — resolved path: _______________
    NEEDED: ___________________________  [ ] system lib  [ ] proprietary — resolved path: _______________

  Blob: ___________________________
    NEEDED: ___________________________  [ ] system lib  [ ] proprietary — resolved path: _______________
    NEEDED: ___________________________  [ ] system lib  [ ] proprietary — resolved path: _______________
    NEEDED: ___________________________  [ ] system lib  [ ] proprietary — resolved path: _______________

  Blob: ___________________________
    NEEDED: ___________________________  [ ] system lib  [ ] proprietary — resolved path: _______________
    NEEDED: ___________________________  [ ] system lib  [ ] proprietary — resolved path: _______________
    NEEDED: ___________________________  [ ] system lib  [ ] proprietary — resolved path: _______________

  (add additional blob rows as the transitive tree expands)

  ── AUDIT COMPLETION GATE ────────────────────────────────────────────────────

  All proprietary NEEDED libs identified          [ ] YES
  All proprietary NEEDED libs resolved to paths   [ ] YES
  All proprietary NEEDED libs added to PRODUCT_COPY_FILES  [ ] YES
  No unresolved NEEDED entries remain             [ ] YES
  Audit complete — dependency tree is closed      [ ] CONFIRMED

└──────────────────────────────────────────────────────────────────────────────┘


████████████████████████████████████████████████████████████████████████████████
█  SECTION 7 — DEVICE TREE FILE SET
█  Every file that must exist before build.
█
█  Section 7 has two parts:
█  7A — required file inventory
█  7B — init hook parity audit (mandatory procedure)
█
█  7B is not optional. TWRP's generic init sequence does not replicate the
█  OEM hardware bring-up sequence. Missing power regulator sequences,
█  touchscreen wake calls, or firmware mounts produce hardware that is
█  physically present but invisible to TWRP at runtime.
████████████████████████████████████████████████████████████████████████████████

┌─ 7A  REQUIRED FILE INVENTORY ────────────────────────────────────────────────┐

  Android.mk                           [ ] exists
  Android.bp                           [ ] exists / N/A   (required if tree uses Soong)
  AndroidProducts.mk                   [ ] exists
  BoardConfig.mk                       [ ] exists   (Section 1 + Section 2)
  twrp_[device].mk                     [ ] exists   (Section 5)
  vendorsetup.sh                       [ ] exists
  recovery.fstab                       [ ] exists   (Section 4)

  recovery/root/                       [ ] exists   (directory)

  init.recovery.[platform].rc          [ ] exists   ← CRITICAL FOR DECRYPTION
  ────────────────────────────────────────────────────────────────────────────
  │  TEE / mobicore service binary     = ___________________________           │
  │  Keymaster HAL service trigger     = ___________________________           │
  │  Any additional platform service   = ___________________________           │
  └────────────────────────────────────────────────────────────────────────────

  ueventd.[platform].rc                [ ] exists / [ ] N/A
        (required if device needs custom ueventd rules for /dev/block permissions)

└──────────────────────────────────────────────────────────────────────────────┘

┌─ 7B  INIT HOOK PARITY AUDIT ─────────────────────────────────────────────────┐
│  MANDATORY PROCEDURE — run before build.                                     │
│                                                                              │
│  The stock OS brings hardware up through init.rc scripts in vendor/etc/init/ │
│  TWRP's generic init sequence does not replicate these. Any hardware that    │
│  the stock OS activates through a sysfs write, insmod, or daemon launch      │
│  that TWRP does not replicate will be dead at TWRP runtime.                 │
│                                                                              │
│  PROCEDURE:                                                                  │
│  1. Extract the stock recovery or boot ramdisk:                              │
│       mkdir /tmp/stock_rd && cd /tmp/stock_rd                                │
│       lz4 -d <stock_ramdisk.img> stock.cpio                                 │
│       cpio -i -F stock.cpio                                                  │
│  2. Collect all OEM init files:                                              │
│       - /tmp/stock_rd/init.*.rc                                              │
│       - All .rc files from the stock vendor/etc/init/ directory              │
│  3. Diff against your TWRP recovery/root/init.recovery.[platform].rc        │
│  4. For every action in the stock init files not present in TWRP, determine │
│     if that action controls hardware TWRP needs (display, touch, storage,   │
│     USB, crypto). If yes, it must be added.                                 │
│                                                                              │
│  CATEGORIES TO EXTRACT AND REPLICATE:                                        │
│  - Power regulator enable sequences (sysfs writes to regulator nodes)       │
│  - Touchscreen controller wake sequences (sysfs writes, gpio triggers)      │
│  - Firmware mount sequences (mount commands, firmware load triggers)         │
│  - Kernel module loads (insmod / modprobe calls for hardware drivers)        │
│  - Permission grants (chmod / chown on /dev nodes TWRP needs)               │
│  - Directory creation (mkdir for paths TWRP mounts or writes to)            │
│  - Proprietary daemon launches (any daemon that storage or crypto needs)     │
└──────────────────────────────────────────────────────────────────────────────┘

  ── INIT HOOK PARITY CHECKLIST ───────────────────────────────────────────────

  Stock init files reviewed
    [ ] stock ramdisk init.*.rc extracted and reviewed
    [ ] stock vendor/etc/init/ .rc files extracted and reviewed

  Power regulators
    [ ] All sysfs regulator enable sequences identified
    Sequences: ___________________________
    [ ] All sequences present in TWRP init.recovery.[platform].rc

  Touchscreen
    [ ] Touchscreen wake sequence identified (sysfs writes / gpio trigger)
    Sequence: ___________________________
    [ ] Sequence present in TWRP init.recovery.[platform].rc

  Firmware mounts
    [ ] Firmware mount commands identified
    Mounts: ___________________________
    [ ] Mounts present in TWRP init.recovery.[platform].rc

  Kernel module loads
    [ ] All insmod / modprobe calls for hardware identified
    Modules: ___________________________
    [ ] All module loads present in TWRP init.recovery.[platform].rc

  Device node permissions
    [ ] chmod / chown calls for /dev nodes TWRP requires identified
    Nodes: ___________________________
    [ ] All permission grants present in TWRP init.recovery.[platform].rc

  Directory creation
    [ ] mkdir calls for paths TWRP uses identified
    Paths: ___________________________
    [ ] All mkdir calls present in TWRP init.recovery.[platform].rc

  Proprietary daemons
    [ ] All daemons storage or crypto requires identified
    Daemons: ___________________________
    [ ] All daemons launched in TWRP init.recovery.[platform].rc
    [ ] All daemon binaries present in recovery ramdisk

  ── PARITY AUDIT COMPLETION GATE ─────────────────────────────────────────────

  All stock init actions reviewed                 [ ] YES
  All hardware-relevant actions identified        [ ] YES
  All identified actions replicated in TWRP init  [ ] YES
  All required daemon binaries present            [ ] YES
  Init hook parity confirmed                      [ ] CONFIRMED

└──────────────────────────────────────────────────────────────────────────────┘


████████████████████████████████████████████████████████████████████████████████
█  SECTION 8 — HARDWARE RUNTIME VALUES
█  Values TWRP reads at runtime from the running device.
████████████████████████████████████████████████████████████████████████████████

  DISPLAY
  ────────────────────────────────────────────────────────────────────────────
  Framebuffer node                     = ___________________________
  All display device nodes required    = ___________________________
  Backlight sysfs node                 = ___________________________
  Hardware max brightness (sysfs)      = ___________________________

  TOUCH
  ────────────────────────────────────────────────────────────────────────────
  Input event device                   = ___________________________
  Touch X range                        = ___________________________
  Touch Y range                        = ___________________________

  USB
  ────────────────────────────────────────────────────────────────────────────
  UDC sysfs node                       = ___________________________
  USB Vendor ID                        = ___________________________
  USB Product ID                       = ___________________________

  COPROCESSOR / PLATFORM SPECIFIC
  ────────────────────────────────────────────────────────────────────────────
  Platform has coprocessor probe delay = ___________________________  (yes/no)
  Expected probe delay duration (sec)  = ___________________________
        (Document the expected delay duration in seconds. This is NOT a failure.
         Some platforms run a coprocessor or DSP probe sequence before the
         recovery UI renders — the screen may be black or static during this
         period. Common on platforms with dedicated coprocessors or security
         processors (examples: MediaTek coprocessor, Qualcomm ADSP/CDSP probe
         sequences on certain devices). Do not interrupt power during this
         delay. If the UI does not render after 60 seconds, that is a failure,
         not a continuation of the delay.)
  Any platform-specific boot behavior  = ___________________________


████████████████████████████████████████████████████████████████████████████████
█  SECTION 9 — VALIDATION RULES
█
█  These are the mathematical and logical relationships between the values
█  you filled in above. Every rule must pass. A filled slot with a wrong
█  relationship still produces a broken build.
█  Work through every rule. Mark pass or fail. Fix before building.
████████████████████████████████████████████████████████████████████████████████

  ── BOOT IMAGE MATH ──────────────────────────────────────────────────────────

  RULE 1   BOARD_KERNEL_BASE + BOARD_KERNEL_OFFSET = kernel_addr (raw binary)
           _______________ + _______________ = _______________  [ ] PASS  [ ] FAIL

  RULE 2   BOARD_KERNEL_BASE + BOARD_RAMDISK_OFFSET = ramdisk_addr (raw at 0x14-0x17)
           _______________ + _______________ = _______________  [ ] PASS  [ ] FAIL

  RULE 3   BOARD_KERNEL_BASE + BOARD_TAGS_OFFSET = tags_addr (raw at 0x20-0x23)
           _______________ + _______________ = _______________  [ ] PASS  [ ] FAIL

  RULE 4   BOARD_KERNEL_BASE + BOARD_DTB_OFFSET = dtb_addr (raw binary — if used)
           _______________ + _______________ = _______________  [ ] PASS  [ ] FAIL  [ ] N/A

  RULE 5   BOARD_KERNEL_PAGESIZE must be a power of 2
           _______________ is a power of 2               [ ] PASS  [ ] FAIL

  ── PARTITION SIZE ───────────────────────────────────────────────────────────

  RULE 6   Output recovery.img size ≤ BOARD_RECOVERYIMAGE_PARTITION_SIZE
           (check after build — before flash)
           output size _______________ ≤ _______________ [ ] PASS  [ ] FAIL

  RULE 7   BOARD_[GROUP]_SIZE ≤ BOARD_SUPER_PARTITION_SIZE
           _______________ ≤ _______________             [ ] PASS  [ ] FAIL

  RULE 8   BOARD_SUPER_PARTITION_SIZE = physical super partition size exactly
           build value _______________ = hardware value _______________
                                                         [ ] PASS  [ ] FAIL

  ── ENCRYPTION ───────────────────────────────────────────────────────────────

  RULE 9   fileencryption= value in fstab is character-for-character identical
           to the fileencryption= value in the stock vendor fstab
           build fstab value : ___________________________
           stock fstab value : ___________________________
                                                         [ ] PASS  [ ] FAIL

  RULE 9b  forceencrypted= path in fstab is character-for-character identical
           to the forceencrypted= path in the stock vendor fstab
           build fstab value : ___________________________
           stock fstab value : ___________________________
                                                         [ ] PASS  [ ] FAIL

  RULE 10  TW_INCLUDE_CRYPTO_FBE requires TW_INCLUDE_CRYPTO = true
           TW_INCLUDE_CRYPTO = ___________________________[ ] PASS  [ ] FAIL

  RULE 11  TW_INCLUDE_FBE_METADATA_DECRYPT requires TW_INCLUDE_CRYPTO_FBE = true
           TW_INCLUDE_CRYPTO_FBE = _________________________[ ] PASS  [ ] FAIL

  RULE 12  If BOARD_USES_METADATA_PARTITION = true then /metadata entry
           must exist in fstab AND must appear before /data
           /metadata present: [ ] YES    before /data: [ ] YES   [ ] PASS  [ ] FAIL

  ── CRYPTO BLOBS ─────────────────────────────────────────────────────────────

  RULE 13  BOARD_VNDK_VERSION matches the version the keymaster blobs were
           compiled against
           build VNDK: _______________  blob target VNDK: _______________
                                                         [ ] PASS  [ ] FAIL

  RULE 14  All crypto blobs are version-consistent with each other
           libcrypto version: ___________________________
           all keymaster libs link same version: [ ] YES  [ ] PASS  [ ] FAIL

  RULE 15  If TW_FORCE_KEYMASTER_VER is set, it matches actual keymaster HAL ver
           TW_FORCE_KEYMASTER_VER = _____ HAL version = _____
                                                         [ ] PASS  [ ] FAIL  [ ] N/A

  ── DISPLAY ──────────────────────────────────────────────────────────────────

  RULE 16  TARGET_SCREEN_WIDTH and TARGET_SCREEN_HEIGHT match
           the actual framebuffer hardware resolution
           build: _______________ × _______________
           hardware: _______________ × _______________  [ ] PASS  [ ] FAIL

  RULE 17  TARGET_RECOVERY_PIXEL_FORMAT matches actual framebuffer pixel format
           build: ___________________________
           hardware: ___________________________         [ ] PASS  [ ] FAIL

  RULE 18  TW_DEFAULT_BRIGHTNESS ≤ TW_MAX_BRIGHTNESS
           _______________ ≤ _______________             [ ] PASS  [ ] FAIL

  RULE 19  TW_MAX_BRIGHTNESS ≤ hardware sysfs max_brightness value
           _______________ ≤ _______________             [ ] PASS  [ ] FAIL

  RULE 20  TW_THEME is appropriate for screen resolution and density
           Theme: _______________  Resolution: _______________ Density: _____ PPI
           (portrait_hdpi ≥ 240 PPI / portrait_mdpi < 240 PPI)
                                                         [ ] PASS  [ ] FAIL

  ── PLATFORM ─────────────────────────────────────────────────────────────────

  RULE 21  PLATFORM_VERSION matches BOARD_OS_VERSION
           PLATFORM_VERSION = _______________  BOARD_OS_VERSION = _______________
                                                         [ ] PASS  [ ] FAIL

  RULE 22  PLATFORM_SECURITY_PATCH matches BOARD_OS_PATCH_LEVEL
           PLATFORM_SECURITY_PATCH = _______________
           BOARD_OS_PATCH_LEVEL    = _______________     [ ] PASS  [ ] FAIL

  ── FSTAB INTEGRITY ──────────────────────────────────────────────────────────

  RULE 23  Every partition in BOARD_[GROUP]_PARTITION_LIST has
           a corresponding entry in the fstab
           List: ___________________________
           All have fstab entries: [ ] YES               [ ] PASS  [ ] FAIL

  RULE 24  Every block device path in the fstab resolves to an actual
           physical partition on the hardware
           All paths verified against /dev/block/by-name/: [ ] YES
                                                         [ ] PASS  [ ] FAIL

  RULE 25  Every partition declared with ro mount flag in the fstab has the
           ro flag set — with no exceptions. No ro-flagged partition may
           appear in the wipe menu (wipeingui=0) or be wipeable (canbewiped=0).
           List all ro-flagged partitions in this build's fstab:
           ___________________________
           ___________________________
           ___________________________
           ___________________________
           All have ro confirmed in mflags:           [ ] YES
           All have wipeingui=0 or wipeingui absent:  [ ] YES
           All have canbewiped=0 or canbewiped absent: [ ] YES
                                                         [ ] PASS  [ ] FAIL

  ── KERNEL ───────────────────────────────────────────────────────────────────

  RULE 26  USB controller driver is built-in (=y) not a module (=m)
           Confirmed built-in: [ ] YES                   [ ] PASS  [ ] FAIL

  RULE 27  All required drivers listed in Section 3.2 are confirmed present
           All checkboxes in 3.2 checked: [ ] YES        [ ] PASS  [ ] FAIL

  RULE 28  Kernel cmdline androidboot.usbcontroller value matches
           the actual USB controller driver name
           cmdline value: _______________  driver name: _______________
                                                         [ ] PASS  [ ] FAIL

  ── PLATFORM IDENTITY ────────────────────────────────────────────────────────

  RULE 29  androidboot.hardware value in BOARD_KERNEL_CMDLINE matches
           TARGET_BOARD_PLATFORM exactly
           cmdline androidboot.hardware: _______________
           TARGET_BOARD_PLATFORM:        _______________
                                                         [ ] PASS  [ ] FAIL

  ── PARTITION SCHEME CONSISTENCY ─────────────────────────────────────────────

  RULE 30  A/B and A-only flags are internally consistent — not mixed
           If AB_OTA_UPDATER = true:
             BOARD_USES_RECOVERY_AS_BOOT must = true     [ ] confirmed
             TARGET_NO_RECOVERY must = true              [ ] confirmed
             /recovery entry must be ABSENT from fstab   [ ] confirmed
             /cache entry must be ABSENT from fstab      [ ] confirmed
           If AB_OTA_UPDATER = false or omitted:
             BOARD_USES_RECOVERY_AS_BOOT must = false    [ ] confirmed
             BOARD_RECOVERYIMAGE_PARTITION_SIZE filled   [ ] confirmed
           Scheme declared consistent:                   [ ] PASS  [ ] FAIL

  ── FLASH BLOCK SIZE ─────────────────────────────────────────────────────────

  RULE 31  BOARD_FLASH_BLOCK_SIZE (if set) is a power of 2
           AND ≥ BOARD_KERNEL_PAGESIZE
           BOARD_FLASH_BLOCK_SIZE = _______________
           BOARD_KERNEL_PAGESIZE  = _______________
           Is power of 2: [ ] YES  Is ≥ pagesize: [ ] YES
                                                         [ ] PASS  [ ] FAIL  [ ] N/A

  ── ABI / BINDER BITNESS ─────────────────────────────────────────────────────

  RULE 32  TARGET_USES_64_BIT_BINDER matches the ABI bitness of the
           Keymaster HAL binary as confirmed by the file command
           Keymaster HAL bitness (from file command): _______________  (32 / 64)
           TARGET_USES_64_BIT_BINDER:                 _______________  (true=64 / false=32)
           Values consistent:                                          [ ] PASS  [ ] FAIL
           ── A 64-bit kernel with a 32-bit Keymaster HAL requires
           ── TARGET_USES_64_BIT_BINDER = false. Setting it to true
           ── causes the keymaster daemon to segfault at startup with
           ── no UI error — /data decrypt silently fails.


████████████████████████████████████████████████████████████████████████████████
█  SECTION 10 — BUILD GATE
█  Every item must be checked before the make command is executed.
████████████████████████████████████████████████████████████████████████████████

  PRE-BUILD
  ────────────────────────────────────────────────────────────────────────────
  [ ] Section 0 — host environment verified, all packages confirmed including
                   cross-compiled readelf binaries, git identity set
  [ ] Section 1 — all BoardConfig.mk values filled including partition scheme
                   and TARGET_USES_64_BIT_BINDER confirmed against file command
  [ ] Section 2 — all TW_ flags filled or decided, platform-specific flags
                   (Section 2.13) evaluated and filled or marked N/A
  [ ] Section 3 — kernel binary confirmed, all drivers present,
                   all FBE crypto primitives confirmed present in kernel
  [ ] Section 4 — every fstab line filled, all 5 columns present,
                   all ro-flagged partitions confirmed ro, /metadata before /data,
                   device-specific partition inventory complete from by-name/,
                   critical partitions identified and protected
  [ ] Section 5 — product makefile complete including PRODUCT_PACKAGES,
                   PRODUCT_COPY_FILES, and PRODUCT_PROPERTY_OVERRIDES (or N/A)
  [ ] Section 6A — all primary crypto blobs confirmed present and version-consistent
  [ ] Section 6B — transitive dependency audit complete, tree closed,
                    all proprietary dependencies in PRODUCT_COPY_FILES
  [ ] Section 7A — all device tree files exist on disk
  [ ] Section 7B — init hook parity audit complete, all hardware bring-up
                    sequences replicated, all daemon binaries present
  [ ] Section 8  — all hardware runtime values confirmed
  [ ] Section 9  — all 32 validation rules marked PASS
  [ ] Section 1.8 — bootloader gate declared: unlock state confirmed,
                     vbmeta strategy chosen, rollback index verified

  POST-BUILD  (before flash)
  ────────────────────────────────────────────────────────────────────────────
  [ ] Output image size ≤ BOARD_RECOVERYIMAGE_PARTITION_SIZE  (Rule 6)
  [ ] xxd output image offset 0x00 = 414e 4452 4f49 4421  ("ANDROID!")
  [ ] xxd output image offset 0x28 matches BOARD_BOOTIMG_HEADER_VERSION
  [ ] xxd output image offset 0x24 matches BOARD_KERNEL_PAGESIZE
  [ ] xxd output image offset 0x0C matches kernel_addr  (Rule 1 confirmed in output)
  [ ] xxd output image offset 0x14 matches ramdisk_addr  (Rule 2 confirmed in output)
  [ ] xxd output image offset 0x20 matches tags_addr  (Rule 3 confirmed in output)
  [ ] All crypto blobs and transitive dependencies present in output vendor image
  [ ] init.recovery.[platform].rc present in output image ramdisk
        (extract: mkdir /tmp/rd && cd /tmp/rd && lz4 -d <ramdisk.img> ramdisk.cpio
         && cpio -i -F ramdisk.cpio — confirm file at expected path)
  [ ] vbmeta flash strategy executed if Strategy (B) — do not flash TWRP
        before vbmeta is correctly flashed

  POST-FLASH  (hardware verification)
  ────────────────────────────────────────────────────────────────────────────
  [ ] Device boots — no kernel panic
  [ ] If platform has coprocessor probe delay (Section 8): wait for delay to
      complete — screen may be black or static during this period — do not
      interrupt power — wait for TWRP UI to render — if UI absent after
      60 seconds that is a failure, not a delay
  [ ] TWRP UI renders — display functional
  [ ] Touch input responds — UI navigable
  [ ] All partitions mount — no errors in TWRP log
  [ ] /data decrypts — FBE chain functional
  [ ] Battery status correct — fuel gauge functional
  [ ] ADB accessible — USB functional
  [ ] NANDroid backup completes — all images non-zero
  [ ] All ro-flagged partitions NOT in wipe menu — ro confirmed on hardware
  [ ] Every value in this document traceable to physical hardware evidence


████████████████████████████████████████████████████████████████████████████████
█  ALL PRE-BUILD ITEMS CHECKED
█  ALL 32 VALIDATION RULES PASS
█  BOOTLOADER GATE CLEARED
█  BLOB DEPENDENCY TREE CLOSED
█  INIT HOOK PARITY CONFIRMED
█  ABI BITNESS VERIFIED
█  = BUILD IS GUARANTEED
████████████████████████████████████████████████████████████████████████████████

################################################################################
# Built from TWRP source outward. Device-agnostic.
# Fill with verified device values. Pass all rules. Build.
################################################################################
