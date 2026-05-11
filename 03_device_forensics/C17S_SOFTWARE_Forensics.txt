################################################################################
# C17S SOFTWARE DIGITAL TWIN — MASTER OUTLINE
# Device:      LOKMAT APPLLP 5 MAX (C17S)
# SoC:         MT6762V/WB (physical) / MT6765 (software platform)
# Author:      beright1976
#
# 
# METHODOLOGY:
#   This document is the forensic software map of the LOKMAT APPLLP 5 MAX (C17S).
#   It is the software equivalent of the hardware forensics database.
#   Nothing is assumed. Nothing is guessed. Nothing is borrowed from similar devices.
#   Every value must be verified against live hardware or extracted binaries.
#   The clean partition pulls are the authority for all file-sourced data.
#   The live device is the authority for all runtime-sourced data.
#   This document is not complete until every field has a [VERIFIED] marker
#   backed by a physical evidence source that can be cited.
# 
# AUTHORITATIVE SOURCES:
#   Clean partition pulls (TWRP-extracted):
#     ~/Downloads/LOKMAT_C17S_Forensics/super_unpacked/system/   — 8,859 files
#     ~/Downloads/LOKMAT_C17S_Forensics/super_unpacked/vendor/   — 1,468 files
#     ~/Downloads/LOKMAT_C17S_Forensics/super_unpacked/product/  — 286 files
#   Live device:
#     Connected via ADB, Magisk root, SELinux permissive
#     TWRP built and functional — use for all partition access
#  
# **WHEN ALL ENTRIES ARE COMPLETELY VERIFIED THEN SECTION MAY BE MARKED VERIFIED**
#      
################################################################################
################################################################################
# SECTION 0 — DOCUMENT CONTROL & PARTITION ARCHITECTURE
################################################################################
# 
# PURPOSE:
#   Establishes the authoritative partition structure, extraction methodology,
#   file counts, and cross-reference map for the entire document.
#   Every other section sources its data from the paths defined here.
# 
# VERIFICATION METHOD:
#   Direct filesystem inspection of clean TWRP pulls.
#   Live device confirmation via ADB for mount points and partition layout.
# 
################################################################################

# 0.1 PARTITION LAYOUT

  Partition scheme          : A-only, single slot, dynamic partitions
  Dynamic partitions native : ro.boot.dynamic_partitions=true
  Dynamic retrofit          : not set (native dynamic — NOT retrofit)
  LP metadata version       : 10.0
  LP metadata slot count    : 2
  LP metadata max size      : 65536 bytes

  Root architecture         : dm-1 mounts at /. /system is a real subdirectory
                              inside dm-1 — NOT system-as-root. /system has no
                              /proc/mounts entry — it is not a separate mount.
                              tune2fs confirms: Last mounted on: /system_root
                              Volume name: /
                              This is a deliberate ODM structure to pass Treble
                              compliance validation on Android 10 while
                              maintaining pre-SAR partition layout.
                              /system/vendor  → /vendor  (symlink)
                              /system/product → /product (symlink)

  ODM overlay               : /odm exists at root as symlink tree → /vendor/odm/
                              /vendor/odm/ is part of vendor logical partition
                              — NOT a separate physical partition
                              odm properties source: ro.product.property_source_order
                              = odm,vendor,product,system

  eMMC device identification:
    Device                  : mmcblk0
    Type                    : MMC
    Manufacturer ID         : 0x000015 (Samsung)
    OEM ID                  : 0x0100
    Product name            : 3V6CMB
    Hardware revision       : 0x0
    Firmware revision       : 0x0300000000000000
    Serial number           : 0x6c4bd632
    CID                     : 150100335636434d42036c4bd6326559
    CSD                     : d02701320f5903fff6dbffef8e40400d
    Manufacture date        : 06/2018
    Total sectors           : 244,277,248
    Total size (blockdev)   : 125,069,950,976 bytes (~116.5 GB)
    Logical block size      : 512 bytes
    Physical block size     : 512 bytes
    Preferred erase size    : 4,194,304 bytes (4 MB)
    Erase size              : 524,288 bytes (512 KB)
    RPMB size multiplier    : 0x80
    Reliable write sectors  : 0x1
    Pre-EOL info            : 0x01 (normal — not worn)
    Life time est type A    : not exposed by kernel
    Life time est type B    : not exposed by kernel
    Enhanced area offset    : not configured (value wraps — no enhanced area)
    mmcblk0 range           : 32 (partition slots)
    mmcblk0 ext_range       : 256

  GPT layout:
    Disk GUID               : 00000000-0000-0000-0000-000000000000
    GPT table capacity      : 128 entries
    Partitions used         : 38 named partitions
    First usable sector     : 34
    Last usable sector      : 244,277,214
    Free space              : 30 sectors (15 KB)
    GPT state               : protective/hybrid MBR + GPT
                              NOTE: sgdisk reports corrupt GPT — disk
                              verification recommended but device boots
                              and operates normally

  /dev/block/by-name/ entry count : 42 total
    38 named partition symlinks
     4 whole-device symlinks (mmcblk0, mmcblk0boot0,
                               mmcblk0boot1, mmcblk0rpmb)

  COMPLETE PARTITION TABLE
  (sources: sgdisk, /dev/block/by-name/, /proc/partitions, /sys/block/mmcblk0/):

  #   Partition        Block Device     Start sector   Size (KB)      Size
  ────────────────────────────────────────────────────────────────────────────
   1  boot_para        mmcblk0p1               64         1,024       1 MB
   2  recovery         mmcblk0p2            2,112        32,768      32 MB
   3  para             mmcblk0p3           67,648           512     512 KB
   4  expdb            mmcblk0p4           68,672        20,480      20 MB
   5  vbmeta_system    mmcblk0p5          109,632         8,192       8 MB
   6  vbmeta_vendor    mmcblk0p6          126,016         8,192       8 MB
   7  frp              mmcblk0p7          142,400         1,024       1 MB
   8  nvcfg            mmcblk0p8          144,448        32,768      32 MB
   9  nvdata           mmcblk0p9          209,984        65,536      64 MB
  10  md_udc           mmcblk0p10         341,056        23,144   ~22.6 MB  ← FBE key material
  11  metadata         mmcblk0p11         387,344        32,768      32 MB  ← EMPTY, NEVER USE
  12  protect1         mmcblk0p12         452,880         8,192       8 MB
  13  protect2         mmcblk0p13         469,264        11,128   ~10.9 MB
  14  seccfg           mmcblk0p14         491,520         8,192       8 MB
  15  sec1             mmcblk0p15         507,904         2,048       2 MB
  16  proinfo          mmcblk0p16         512,000         3,072       3 MB
  17  md1img           mmcblk0p17         518,144       102,400     100 MB
  18  spmfw            mmcblk0p18         722,944         1,024       1 MB
  19  scp1             mmcblk0p19         724,992         1,024       1 MB
  20  scp2             mmcblk0p20         727,040         1,024       1 MB
  21  sspm_1           mmcblk0p21         729,088         1,024       1 MB
  22  sspm_2           mmcblk0p22         731,136         1,024       1 MB
  23  gz1              mmcblk0p23         733,184        16,384      16 MB
  24  gz2              mmcblk0p24         765,952        16,384      16 MB
  25  nvram            mmcblk0p25         798,720        65,536      64 MB
  26  lk               mmcblk0p26         929,792         1,024       1 MB
  27  lk2              mmcblk0p27         931,840         1,024       1 MB
  28  boot             mmcblk0p28         933,888        32,768      32 MB
  29  logo             mmcblk0p29         999,424         8,192       8 MB
  30  dtbo             mmcblk0p30       1,015,808         8,192       8 MB
  31  tee1             mmcblk0p31       1,032,192         5,120       5 MB
  32  tee2             mmcblk0p32       1,042,432        11,264      11 MB
  33  super            mmcblk0p33       1,064,960     4,194,304       4 GB
  34  vbmeta           mmcblk0p34       9,453,568         8,192       8 MB
  35  cache            mmcblk0p35       9,469,952       442,368     432 MB
  36  userdata         mmcblk0p36      10,354,688   116,900,847  ~111.5 GB
  37  otp              mmcblk0p37     244,156,383        44,032      43 MB
  38  flashinfo        mmcblk0p38     244,244,447        16,384      16 MB
  ────────────────────────────────────────────────────────────────────────────
      mmcblk0boot0     (hardware boot ROM partition)   4,096         4 MB
      mmcblk0boot1     (hardware boot ROM partition)   4,096         4 MB
      mmcblk0rpmb      (replay protected memory block) 16,384       16 MB

  PARTITION IDENTITY CONFIRMED (/sys/block/mmcblk0/uevent):
    mmcblk0p10 : PARTNAME=md_udc   ← FBE key material
    mmcblk0p11 : PARTNAME=metadata ← EMPTY, ghost partition, NEVER USE

  FSTAB PARTITION REFERENCES WITH NO PHYSICAL PARTITION:
    md1dsp, md1arm7, md3img, odmdtbo — referenced in fstab.mt6762 and
    fstab.mt6765 but confirmed absent from GPT, by-name, /proc/partitions,
    and raw sector scan. These are MT6765 BSP platform fstab template
    remnants. The ODM used the MTK reference fstab unmodified.

  DYNAMIC PARTITION GROUP (lpdump):
    Group name              : main
    Maximum size            : 4,292,870,144 bytes

    Partition    dm device   Sectors      Actual size            UUID
    ───────────────────────────────────────────────────────────────────────────
    product      dm-0        2,298,328    1,176,743,936 (~1.1 GB) 453968ff-b43c-54b7-a506-a6a88682c128
    system       dm-1        2,516,312    1,288,351,744 (~1.2 GB) 4729639d-b5f2-5cc1-a120-9ac5f788683c
    vendor       dm-2          568,848      291,250,176 (~278 MB) 2b96c597-1e2f-5ee1-9851-c4a9fa9de36e
    ───────────────────────────────────────────────────────────────────────────
    dm-3         com.android.tzdata APEX — 800 KB — not a named partition

  DEVICE MAPPER SYMLINKS (/dev/block/mapper/):
    system                  → /dev/block/dm-1
    vendor                  → /dev/block/dm-2
    product                 → /dev/block/dm-0
    com.android.tzdata@297903000 → /dev/block/dm-3

  DYNAMIC PARTITION FILESYSTEM DETAILS (tune2fs):
    dm-0 (product):
      Volume name           : product
      Last mounted on       : /product
      Block size            : 4096
      Block count           : 282,737
      Inode count           : 432
      Free blocks           : 846
      Lifetime writes       : 1 KB
      Features              : ext_attr resize_inode dir_index filetype extent
                              sparse_super large_file huge_file uninit_bg
                              dir_nlink extra_isize

    dm-1 (system):
      Volume name           : /
      Last mounted on       : /system_root
      Block size            : 4096
      Block count           : 309,554
      Inode count           : 4,320
      Free blocks           : 943
      Mount count           : 573
      Last mount time       : Sun May 3 23:11:34 2026
      Lifetime writes       : 5,477 KB
      Features              : ext_attr resize_inode dir_index filetype extent
                              sparse_super large_file huge_file uninit_bg
                              dir_nlink extra_isize

    dm-2 (vendor):
      Volume name           : vendor
      Last mounted on       : /vendor
      Filesystem state      : not clean
      Block size            : 4096
      Block count           : 69,962
      Inode count           : 1,584
      Free blocks           : 215
      Mount count           : 144
      Last mount time       : Sun May 3 23:11:33 2026
      Lifetime writes       : 569 KB
      Features              : ext_attr resize_inode dir_index filetype extent
                              sparse_super large_file huge_file uninit_bg
                              dir_nlink extra_isize

  SWAP (fstab.enableswap):
    /dev/block/zram0        : swap, zramsize=55%

# 0.2 FSTAB

  Location                  : /vendor/etc/fstab.mt6762
                              /vendor/etc/fstab.mt6765
  NOTE: Both files are identical — same MT6765 BSP template, different name.
  Source template           : vendor/mediatek/proprietary/hardware/fstab/mt6765/fstab.in.mt6765
  Product fstab             : does not exist — product mounted via vendor fstab

  LOGICAL PARTITION MOUNTS:
    system   /system   ext4  ro  wait,avb=vbmeta_system,logical,first_stage_mount,
                                 avb_keys=/avb/q-gsi.avbpubkey:/avb/r-gsi.avbpubkey:/avb/s-gsi.avbpubkey
    vendor   /vendor   ext4  ro  wait,avb,logical,first_stage_mount
    product  /product  ext4  ro  wait,avb,logical,first_stage_mount

  PHYSICAL PARTITION MOUNTS:
    /dev/block/platform/bootdevice/by-name/md_udc      /metadata          ext4
      noatime,nosuid,nodev,discard — wait,formattable,first_stage_mount
    /dev/block/platform/bootdevice/by-name/userdata    /data              ext4
      noatime,nosuid,nodev,noauto_da_alloc,errors=panic —
      wait,check,quota,reservedsize=128M,formattable,resize,
      checkpoint=block,fileencryption=aes-256-xts
    /dev/block/platform/bootdevice/by-name/cache       /cache             ext4
      noatime,nosuid,nodev,noauto_da_alloc,discard — wait,check,formattable
    /dev/block/platform/bootdevice/by-name/protect1    /mnt/vendor/protect_f  ext4
      noatime,nosuid,nodev,noauto_da_alloc,commit=1,nodelalloc —
      wait,check,formattable
    /dev/block/platform/bootdevice/by-name/protect2    /mnt/vendor/protect_s  ext4
      noatime,nosuid,nodev,noauto_da_alloc,commit=1,nodelalloc —
      wait,check,formattable
    /dev/block/platform/bootdevice/by-name/nvdata      /mnt/vendor/nvdata ext4
      noatime,nosuid,nodev,noauto_da_alloc,discard — wait,check,formattable
    /dev/block/platform/bootdevice/by-name/nvcfg       /mnt/vendor/nvcfg  ext4
      noatime,nosuid,nodev,noauto_da_alloc,commit=1,nodelalloc —
      wait,check,formattable

  EMMC RAW MOUNTS (no filesystem — raw binary access):
    /dev/block/platform/bootdevice/by-name/frp         /persistent
    /dev/block/platform/bootdevice/by-name/nvram       /nvram
    /dev/block/platform/bootdevice/by-name/proinfo     /proinfo
    /dev/block/platform/bootdevice/by-name/lk          /bootloader
    /dev/block/platform/bootdevice/by-name/lk2         /bootloader2
    /dev/block/platform/bootdevice/by-name/para        /misc
    /dev/block/platform/bootdevice/by-name/boot        /boot        first_stage_mount,nofail
    /dev/block/platform/bootdevice/by-name/recovery    /recovery    first_stage_mount,nofail
    /dev/block/platform/bootdevice/by-name/vbmeta_vendor  /vbmeta_vendor  first_stage_mount,nofail
    /dev/block/platform/bootdevice/by-name/vbmeta_system  /vbmeta_system  first_stage_mount,nofail,avb=vbmeta
    /dev/block/platform/bootdevice/by-name/logo        /logo
    /dev/block/platform/bootdevice/by-name/expdb       /expdb
    /dev/block/platform/bootdevice/by-name/seccfg      /seccfg
    /dev/block/platform/bootdevice/by-name/tee1        /tee1
    /dev/block/platform/bootdevice/by-name/tee2        /tee2
    /dev/block/platform/bootdevice/by-name/scp1        /scp1
    /dev/block/platform/bootdevice/by-name/scp2        /scp2
    /dev/block/platform/bootdevice/by-name/sspm_1      /sspm_1
    /dev/block/platform/bootdevice/by-name/sspm_2      /sspm_2
    /dev/block/platform/bootdevice/by-name/md1img      /md1img
    /dev/block/platform/bootdevice/by-name/spmfw       /spmfw
    /dev/block/platform/bootdevice/by-name/boot_para   /boot_para
    /dev/block/platform/bootdevice/by-name/dtbo        /dtbo        first_stage_mount,nofail
    /dev/block/platform/bootdevice/by-name/vbmeta      /vbmeta

  EXTERNAL STORAGE:
    /devices/platform/externdevice*  auto  voldmanaged=sdcard1:auto,encryptable=userdata
    /devices/platform/mt_usb*        vfat  voldmanaged=usbotg:auto

  FSTAB GHOST ENTRIES (referenced but partition does not exist on this device):
    /dev/block/platform/bootdevice/by-name/md1dsp   /md1dsp
    /dev/block/platform/bootdevice/by-name/md1arm7  /md1arm7
    /dev/block/platform/bootdevice/by-name/md3img   /md3img
    /dev/block/platform/bootdevice/by-name/odmdtbo  /odmdtbo
    NOTE: Confirmed absent from GPT, by-name, /proc/partitions, and raw
          sector scan. MT6765 BSP platform fstab template remnants.

# 0.3 LIVE MOUNT TABLE (from /proc/mounts)

  /dev/block/dm-1                               /          ext4  ro
  /dev/block/dm-2                               /vendor    ext4  ro
  /dev/block/dm-0                               /product   ext4  ro
  /dev/block/platform/bootdevice/by-name/md_udc /metadata  ext4  rw
    mount options: nosuid,nodev,noatime,discard,data=ordered
  /dev/block/platform/bootdevice/by-name/userdata /data    ext4  rw
    mount options: nosuid,nodev,noatime,noauto_da_alloc,resgid=1065,
                   errors=panic,data=ordered
  /dev/block/platform/bootdevice/by-name/cache  /cache     ext4  rw
  /dev/block/mmcblk0p12  /mnt/vendor/protect_f  ext4  rw
  /dev/block/mmcblk0p13  /mnt/vendor/protect_s  ext4  rw
  /dev/block/mmcblk0p9   /mnt/vendor/nvdata      ext4  rw
  /dev/block/mmcblk0p8   /mnt/vendor/nvcfg       ext4  rw
  /dev/block/mmcblk0p10  /data/local/tmp/meta_mount  ext4  rw
    NOTE: mmcblk0p10 (md_udc) mounted twice — as /metadata AND as
          /data/local/tmp/meta_mount. Second mount is a Magisk/root
          diagnostic artifact, not a stock mount.

  Virtual/special mounts:
    tmpfs      /dev, /mnt, /apex, /storage, /sbin (magisk)
    debugfs    /sys/kernel/debug
    tracefs    /sys/kernel/debug/tracing
    selinuxfs  /sys/fs/selinux
    configfs   /config
    bpf        /sys/fs/bpf
    pstore     /sys/fs/pstore
    cgroup     /dev/cpuctl, /acct, /dev/cpuset, /dev/stune, /dev/cg2_bpf
    functionfs /dev/usb-ffs/adb
    sdcardfs   /data/media mounts (emulated storage, 4 bind points)

  APEX modules mounted (loop devices):
    com.android.tzdata@297903000        dm-3      800 KB
    com.android.media@300900700         loop3     4.3 MB
    com.android.runtime@1               loop4      95 MB
    com.android.apex.cts.shim@1         loop5     232 KB
    com.android.resolv@300900700        loop6     2.3 MB
    com.android.conscrypt@291900801     loop7     4.9 MB
    com.android.media.swcodec@300900700 loop8      18 MB

# 0.4 KERNEL CMDLINE (from /proc/cmdline — live device)

  console=tty0 console=ttyS0,921600n1 vmalloc=400M slub_debug=OFZPU
  page_owner=on swiotlb=noforce androidboot.hardware=mt6765 maxcpus=8
  loop.max_part=7 firmware_class.path=/vendor/firmware has_battery_removed=0
  androidboot.boot_devices=bootdevice,11230000.mmc root=/dev/ram
  androidboot.verifiedbootstate=orange bootopt=64S3,32N2,64N2
  buildvariant=user androidboot.atm=disabled androidboot.meta_log_disable=0
  printk.disable_uart=0 bootprof.pl_t=4671 bootprof.lk_t=8129
  bootprof.logo_t=998 androidboot.serialno=MTK0002507301608400
  androidboot.bootreason=reboot gpt=1 usb2jtag_mode=0 mrdump_ddrsv=yes
  mrdump_cb=0x10e800,0x1400 androidboot.dtb_idx=0 androidboot.dtbo_idx=0

  NOTE: Section 15 BOARD_KERNEL_CMDLINE must match this verbatim.
  Additional params present in live cmdline not in Section 15:
    androidboot.atm=disabled
    androidboot.meta_log_disable=0
    printk.disable_uart=0
    bootprof.pl_t=4671
    bootprof.lk_t=8129
    bootprof.logo_t=998
    androidboot.serialno=MTK0002507301608400
    androidboot.bootreason=reboot
    usb2jtag_mode=0
    androidboot.dtb_idx=0
    androidboot.dtbo_idx=0

# 0.5 SECURITY STATE

  Verified boot state       : orange (ro.boot.verifiedbootstate=orange)
  Bootloader locked         : 0 (ro.boot.flash.locked=0 — UNLOCKED)
  Verity mode               : not set
  Warranty bit              : not set
  AVB                       : disabled
  SELinux                   : PERMISSIVE

# 0.6 CLEAN PULL VERIFICATION

  Extraction method:
    1. Boot device into TWRP (built, functional, decrypting)
    2. adb shell twrp mount system / vendor / product
    3. adb pull /system_root → super_unpacked/system/
    4. adb pull /vendor     → super_unpacked/vendor/
    5. adb pull /product    → super_unpacked/product/

  Verified file counts (from clean TWRP pull 2026-05-08):
    system/  : 8,859 files
    vendor/  : 1,468 files
    product/ : 286 files

  Previous extraction was INCOMPLETE — all data sourced from previous
  extraction must be reverified against current clean pulls.

# 0.7 CROSS-REFERENCE MAP

  Section 1  — Init System          → sources from vendor/etc/init/
  Section 2  — HAL Stack            → sources from vendor/etc/vintf/, vendor/lib64/hw/
  Section 3  — Crypto Stack         → sources from vendor/lib64/, system/lib64/vndk-29/
  Section 4  — Sensor Stack         → sources from vendor/lib64/hw/, live /proc/maps
  Section 5  — Display & GPU        → sources from vendor/lib64/egl/, live /proc/maps
  Section 6  — Camera Stack         → sources from vendor/lib64/, live /proc/maps
  Section 7  — Audio Stack          → sources from vendor/etc/, vendor/lib64/
  Section 8  — Radio & Modem        → sources from vendor/lib64/, vendor/bin/
  Section 9  — Connectivity         → sources from vendor/firmware/, vendor/lib/modules/
  Section 10 — VNDK Inventory       → sources from system/lib64/vndk-29/
  Section 11 — Firmware Blobs       → sources from vendor/firmware/
  Section 12 — APK Inventory        → sources from all three partitions
  Section 13 — ODM Control Layer    → sources from live sysfs + APK decompile
  Section 14 — SELinux Policy       → sources from vendor/etc/selinux/
  Section 15 — Build Specification  → sources from vendor/build.prop + all sections above
  Section 16 — Security Analysis    → sources from APK static analysis + live device
  Section 17 — ROM Build Gate       → derived from all sections above
################################################################################
# SECTION 1 — INIT SYSTEM
################################################################################

# 1.1 RC FILE INVENTORY

  Total RC files : 85
  Location       : /vendor/etc/init/
  Structure:
    Flat (/vendor/etc/init/*.rc)       : 68 files
    Subdirectory (/vendor/etc/init/hw/): 17 files

  NOTE: ls /vendor/etc/init/ returns 69 entries — the 69th entry is the
        hw/ subdirectory itself. Flat RC file count is confirmed 68.

  CORRECTION FROM PREVIOUS DOCUMENT:
    init.wlan_drv.rc is the correct filename.
    init.vlan_drv.rc does NOT exist — was a typo in an earlier ls output.
    Confirmed: ls /vendor/etc/init/ | grep wlan returns init.wlan_drv.rc only.

  COMPLETE FILE LIST — FLAT (68 files):

    android.hardware.audio@5.0-service-mediatek.rc
    android.hardware.bluetooth@1.0-service-mediatek.rc
    android.hardware.cas@1.1-service.rc
    android.hardware.configstore@1.1-service.rc
    android.hardware.drm@1.0-service.rc
    android.hardware.drm@1.2-service.clearkey.rc
    android.hardware.drm@1.2-service.widevine.rc
    android.hardware.gatekeeper@1.0-service.rc
    android.hardware.gnss@2.0-service-mediatek.rc
    android.hardware.graphics.allocator@2.0-service.rc
    android.hardware.graphics.composer@2.1-service.rc
    android.hardware.health@2.0-service.rc
    android.hardware.keymaster@4.0-service.rc
    android.hardware.light@2.0-service-mediatek.rc
    android.hardware.media.omx@1.0-service.rc
    android.hardware.memtrack@1.0-service.rc
    android.hardware.secure_element@1.0-service-mediatek.rc
    android.hardware.sensors@2.0-service-mediatek.rc
    android.hardware.thermal@1.0-service.rc
    android.hardware.usb@1.1-service-mediatek.rc
    android.hardware.vibrator@1.0-service.rc
    android.hardware.wifi@1.0-service-lazy-mediatek.rc
    atcid.rc
    audiocmdservice_atci.rc
    bootperf.rc
    camerahalserver.rc
    em_hidl_user.rc
    fuelgauged_init.rc
    fuelgauged_nvram_init.rc
    gsm0710muxd.rc
    hostapd.android.rc
    init.bip.rc
    init.bt_drv.rc
    init.cccifsd.rc
    init.cccimdinit.rc
    init.cccirpcd.rc
    init_connectivity.rc
    init.gps_drv.rc
    init.md_apps.rc
    init.thermal.rc
    init.thermal_manager.rc
    init.thermalloadalgod.rc
    init.volte_imcb.rc
    init.volte_imsm_93.rc
    init.volte_md_status.rc
    init.volte_stack.rc
    init.volte_ua.rc
    init.vtservice_hidl.rc
    init.wfca.rc
    init.wlan_drv.rc
    init.wmt_drv.rc
    init.wod.rc
    ipsec_mon.rc
    lbs_hidl_service.rc
    mtk_agpsd_p.rc
    mtkrild.rc
    muxreport.rc
    netdagent.rc
    nvram_daemon.rc
    vendor.mediatek.hardware.gpu@1.0-service.rc
    vendor.mediatek.hardware.mms@1.3-service.rc
    vendor.mediatek.hardware.mtkcodecservice@1.1-service.rc
    vendor.mediatek.hardware.mtkpower@1.0-init.rc
    vendor.mediatek.hardware.mtkpower@1.0-service.rc
    vendor.mediatek.hardware.nvram@1.1-sevice.rc   ← ODM TYPO: "sevice" not "service"
    vendor.mediatek.hardware.pq@2.2-service.rc
    vndservicemanager.rc
    wlan_assistant.rc

  COMPLETE FILE LIST — hw/ SUBDIRECTORY (17 files):

    hw/factory_init.connectivity.rc
    hw/factory_init.project.rc
    hw/factory_init.rc
    hw/init.aee.rc
    hw/init.ago.rc
    hw/init.connectivity.rc
    hw/init.modem.rc
    hw/init.mt6762.rc
    hw/init.mt6765.rc
    hw/init.mt6765.usb.rc
    hw/init.project.rc
    hw/init.sensor_1_0.rc
    hw/meta_init.connectivity.rc
    hw/meta_init.modem.rc
    hw/meta_init.project.rc
    hw/meta_init.rc
    hw/multi_init.rc

# 1.2 RC FILE CONTENT — COMPLETE

  ── hw/ SUBDIRECTORY FILES ──────────────────────────────────────────────────

  hw/init.mt6762.rc [1,113 lines — FULL CONTENT DOCUMENTED]

    CRITICAL NOTE: init.mt6762.rc and init.mt6765.rc are BYTE-FOR-BYTE
    IDENTICAL. Same content, different filename. The ODM ships one file
    under both platform names. Confirmed by cat of both files returning
    identical output.

    IMPORT CHAIN (top of file — processed in order):
      import ${ro.vendor.rc}init.connectivity.rc    ← resolves to init_connectivity.rc
      import /vendor/etc/init/hw/init.mt6765.usb.rc
      import /vendor/etc/init/hw/init.project.rc
      import /FWUpgradeInit.rc                      ← not in vendor, ramdisk injection
      import /vendor/etc/init/hw/init.aee.rc
      import /system/etc/init/hw/init.aee.rc
      import ${ro.vendor.rc}init.volte.rc
      import ${ro.vendor.rc}init.mal.rc
      import ${ro.vendor.rc}init.sensor_1_0.rc
      import ${ro.vendor.rc}init.ago.rc
      import ${ro.vendor.rc}init.modem.rc           ← MUST BE LAST — sets vold.post_fs_data_done=1

    on early-init:
      write /proc/bootprof "INIT:early-init"
      write /proc/perfmgr/boost_ctrl/cpu_ctrl/boot_freq "0 0 5 0"
      write /sys/devices/platform/10012000.dvfsrc/helio-dvfsrc/dvfsrc_req_ddr_opp 0
      mount debugfs debugfs /sys/kernel/debug
      chmod 0755 /sys/kernel/debug

    on init:
      mkdir /dev/stune/io (io boost cgroup)
      symlink /sdcard /mnt/sdcard
      mkdir /mnt/vendor/protect_f, protect_s, nvcfg, nvdata
      mkdir /custom
      mkdir /mnt/cd-rom 0000 system system
      chown/chmod /proc/lk_env (root:system 0660)
      write /proc/sys/net/core/wmem_max 8388608
      write /proc/sys/net/core/rmem_max 8388608
      cpuset assignments: foreground/boost 0-7, background 4-7,
                          system-background 2-7, top-app 0-7
      write /dev/cpuctl/bg_non_interactive/cpu.rt_runtime_us 100000
      symlink /dev/block/platform/bootdevice /dev/block/bootdevice

    on late-init:
      write /proc/bootprof "INIT:late-init"
      write /sys/kernel/rcu_expedited 1

    on charger (offline charging mode — separate boot path):
      Disables CPUs 4-7, enables 0-3
      e2fsck on cache partition
      Mounts cache, nvcfg
      Starts fuelgauged, fuelgauged_nvram
      Starts servicemanager, hwservicemanager, light-hal-2-0, health-hal-2-0

    on fs:
      write /proc/bootprof "INIT:Mount_START"
      wait_for_prop hwservicemanager.ready "true"
      start vendor.boot-hal-1-0
      mount_all /vendor/etc/fstab.mt6765        ← mounts all partitions
      write /proc/bootprof "INIT:Mount_END"
      mount iso9660 loop@/vendor/etc/iAmCdRom.iso /mnt/cd-rom ro
        NOTE: ODM mounts a CD-ROM ISO image at /mnt/cd-rom on every boot.
              Purpose unknown. Requires Ghidra/strings analysis of iAmCdRom.iso.

    on post-fs:
      write /proc/bootprof "INIT:post-fs"
      boot perf boost: eas_ctrl boot_boost "0-3 100"
      symlink /sdcard /storage/sdcard0
      chown/chmod /proc/freqhopping/* nodes
      symlink /dev/block/sda /dev/block/mmcblk0boot0
      symlink /dev/block/sdb /dev/block/mmcblk0boot1
      symlink /dev/block/mmcblk0boot0 /dev/block/platform/bootdevice/by-name/preloader_a
      symlink /dev/block/mmcblk0boot1 /dev/block/platform/bootdevice/by-name/preloader_b
        NOTE: A/B preloader symlinks created even though device is A-only.
              ODM BSP template — not functional on this device.
      chown/chmod /proc/bootprof, /proc/mtprof/cputime, /proc/mtprof/reboot_pid
      chown/chmod /mnt/vendor/protect_f, protect_s

    on post-fs-data:
      write /proc/bootprof "INIT:post-fs-data"
      Boot-time IO tuning: read_ahead_kb 2048 on mmcblk0, dm-0 through dm-5
      Filesystem setup: nvcfg/fg, nvdata/media, cct, flash, shading,
                        camera_dump, dipdebug, flicker directories
      restorecon_recursive /mnt/vendor/nvdata
      chmod 0660 /dev/rfkill
      mkdir /dev/radio 0770 radio radio
      GPS: chmod 0775 /vendor/xbin/MPED, chmod 0666 /dev/gps_emi
      SCP/SSPM log node permissions
      radvd config: chmod 444 /etc/radvd/radvd.conf
      Mali GPU: chmod 666 /dev/mali0 [NOTE: Mali driver commented out — PowerVR used]
      Touch panel sysfs permissions (tpd_setting, tpd_debug)
      CMDQ: chmod 0644 /dev/mtk_cmdq (system:system)
      MDP SYNC: chmod 0644 /dev/mdp_sync (system:system)
      Audio IPI, SCP, SmartPA nodes
      Accdet, Fingerprint (/dev/goodix_fp 0666), VideoCodec, MJC
      Display: /dev/graphics/fb0 (system:graphics 0664)
               /dev/mtk_disp_mgr, /dev/mtkfb_vsync, /dev/sw_sync
      FM Radio: chmod 0660 /dev/fm (media:media)
      Device info: chmod 0440 /dev/devmap (root:system)
      Bluetooth: /dev/hid-keyboard (bluetooth:bluetooth 0660)
                 /dev/uinput (system:net_bt_admin)
      Camera nodes: camera-sysram, camera-isp, camera-dpe, camera-tsf,
                    camera-fdvt, camera-pipemgr, kd_camera_hw, flashlight,
                    kd_camera_hw_bus2, FM50AF, CAM_CAL_DRV, MAINAF, MAIN2AF,
                    SUBAF (all system:camera 0660)
      CCU: chmod 0666 /dev/ccu (system:camera)
      Various MTK legacy nodes (MATV, OTG, MDP, SMI, RRC, DFRC, EBC, UIO0)
      Thermal: mkdir /data/vendor/.tp/ 0775 system system
      write /sys/power/pm_async 0

      #nrf ctrl — wiite_corp_ctrl permissions set here:
        chown/chmod 0666: rst_op, upgrade_op, nrf_rtc, btMAC, longterm,
                          command, version, config, uart_op
        NOTE: Only 9 of 34 attributes explicitly permissioned here.
              Remaining attributes inherit default or are write-only.

      #uart1:
        chmod 666 /dev/ttyS1
        chown system system /dev/ttyS1
        NOTE: 0666 is intentional — any process can access nRF52832 UART

    on boot:
      write /proc/bootprof "INIT:boot"
      write /proc/sys/vm/dirty_writeback_centisecs 300
      LCD/LED brightness nodes (system:system 0664)
      CPU freq hotplug nodes
      CPU governor nodes (policy0, policy4)
      Binder debug nodes
      PPM policy nodes (extensive)
      CPU loading nodes
      CPU freq nodes (hevc limit, user limit, power mode)
      BW monitor nodes
      perfmgr nodes (extensive)
      Network: /proc/net/vlan/config, ip tables, nf_conntrack
      WMT proc: /proc/driver/wmt_dbg, wmt_aee (shell:system)
      EMCSMDLOGGER: /dev/ttyGS3 (system:radio 0660)

      #nrf ctrl (additional):
        NOTE: Only rst_op, upgrade_op etc set in post-fs-data above.
              ttyS1 chmod 666 set here in boot stage.

      TCP buffer sizes set for: default, wifi, lte, umts, hspa, hsupa,
                                 hsdpa, hspap, edge, gprs
      setprop persist.sys.strictmode.disable 1

    on property:sys.boot_completed=1:
      write /proc/bootprof "0"  ← stops boot profiling
      IO tune restore: read_ahead_kb 512, nr_requests 128
      EAS+: schedtune.boost 0 on all groups
            cpu eas enable = 2
            cpufreq_sched_disable = 0
            scaling_governor = "schedplus" on policy0 and policy4
      Boot perf restore: boot_boost "0-3 0", sched_stune_task_threshold "-1",
                         boot_freq "-1 -1 -1 -1", dvfsrc_req_ddr_opp "-1"
      write /proc/mtprintk ${persist.vendor.uartconsole.enable}

    on boot (additional):
      insmod /vendor/lib/modules/fpsgo.ko
        NOTE: fpsgo.ko loaded via insmod here, not via module load order.

    SERVICES DEFINED IN init.mt6762.rc:
      bugreport: /system/bin/dumpstate -d -p -B -z (class late_start, disabled, oneshot, keycodes 114 115 116)
      pppoe_wlan0: /vendor/bin/launchpppoe (user root, disabled, oneshot)
      cmmb: /system/bin/cmmbsp (class main, user root, ioprio rt 4) [legacy, likely unused]
      meta_tst: /vendor/bin/meta_tst (class main, user system, NET_RAW NET_ADMIN, disabled, oneshot)
      loghidlsysservice: /system/bin/loghidlsysservice (class main, user root, disabled)
      osi: /system/bin/osi (class main, user root, disabled, oneshot) [VSIM service]

    PROPERTY TRIGGERS IN init.mt6762.rc:
      ro.board.platform=mt6753 → hispeed_freq node (legacy, not this device)
      vold.decrypt=trigger_* → bootprof markers + sys.boot_completed=0
      persist.vendor.mediatek.fg.disable/force25c/log.enable → battery nodes
      sys.boot_completed=1 → stop bootprof, IO tune, EAS+, start meta_tst if atm=enable
      persist.vendor.uartconsole.enable=* → write /proc/mtprintk

  hw/init.mt6765.rc
    IDENTICAL to init.mt6762.rc — confirmed byte-for-byte same content.
    Two files, one implementation. ODM BSP template pattern.

  hw/init.project.rc
    on init:
      mkdir /mnt/media_rw/usbotg 0700 media_rw media_rw
      mkdir /storage/usbotg 0700 root root
    on post-fs-data:
      chmod 0660 /dev/ttyMT2 (system:system) ← GPS UART
      chown gps/chmod 0660: gpsdrv sysfs nodes (pwrctl, suspend, state, pwrsave, status)
      chmod 0660 /dev/ttyMT1 (system:system) ← sensor UART
    on boot:
      chmod 0660 /dev/ttyMT0 (system:system) ← BT UART

  hw/init.ago.rc
    on init:
      write /sys/block/zram0/comp_algorithm lz4
    on post-fs-data:
      write /sys/block/zram0/comp_algorithm lz4
      write /proc/sys/vm/page-cluster 0
      write /proc/sys/vm/swappiness 100
      write /sys/kernel/mm/ksm/pages_to_scan 100
      write /sys/kernel/mm/ksm/sleep_millisecs 500
      write /sys/kernel/mm/ksm/run 1
    NOTE: Document previously stated ZRAM 3.3GB, swappiness 150 — BOTH WRONG.
          Actual: swappiness=100, no size set, KSM enabled (100 pages/500ms scan).
          ZRAM size set by fstab.enableswap: zramsize=55% (of RAM).

  hw/init.aee.rc
    on init:
      write /proc/aed/generate-oops aee0
    on post-fs-data:
      mkdir /data/vendor/aee_exp 0775 system system
      mkdir /data/vendor/reboot 0775 system system

  hw/init.connectivity.rc
    service wmt_loader /vendor/bin/wmt_loader (class core, user system, group system)
    service wmt_launcher /vendor/bin/wmt_launcher -p /vendor/firmware/
      (class core, user system, group system)
    NOTE: This file uses class core and NO -o 1 flag on wmt_launcher.
          Compare with init_connectivity.rc (flat) which uses class early_hal
          and adds -o 1 flag. The flat init_connectivity.rc is the active one
          at runtime (loaded via import ${ro.vendor.rc}init.connectivity.rc).
          This hw/ version appears to be the older/backup template.

  hw/init.modem.rc
    on post-fs-data:
      setprop vold.post_fs_data_done 1
    NOTE: This is the ONLY content. This single property set is why this file
          MUST be the last import in init.mt6762.rc. vold waits for this
          property before proceeding with /data mount and FBE operations.
          If any RC file is imported after this, vold may trigger before
          all post-fs-data work is complete.

  hw/init.mt6765.usb.rc
    on boot:
      setprop sys.usb.configfs 1

  hw/init.sensor_1_0.rc
    on post-fs-data:
      mkdir /data/vendor/sensor 0777 system system

  hw/factory_init.rc
    on early-init: bootprof marker "INIT:factory:early-init"
    on init: bootprof marker "INIT:factory:init"
    on late-init: bootprof marker "INIT:factory:late-init"

  hw/factory_init.connectivity.rc  — EMPTY (header comment only)
  hw/factory_init.project.rc       — EMPTY (header comment only)
  hw/meta_init.rc
    on early-init: bootprof marker "INIT:meta:early-init"
    on init: bootprof marker "INIT:meta:init"
    on late-init: bootprof marker "INIT:meta:late-init"
  hw/meta_init.connectivity.rc     — EMPTY (header comment only)
  hw/meta_init.modem.rc            — EMPTY (header comment only)
  hw/meta_init.project.rc          — EMPTY (header comment only)
  hw/multi_init.rc                 — EMPTY (header comment only)

  ── FLAT RC FILES — COMPLETE SERVICE MAP ────────────────────────────────────

  android.hardware.audio@5.0-service-mediatek.rc
    service vendor.audio-hal-2-0
    binary  : /vendor/bin/hw/android.hardware.audio@5.0-service-mediatek
    class   : hal
    user    : audioserver
    group   : audio camera drmrpc inet media mediadrm net_bt net_bt_admin
              net_bw_acct wakelock system sdcard_rw
    caps    : BLOCK_SUSPEND SYS_NICE
    ioprio  : rt 4
    oneshot
    interface: android.hardware.audio@5.0::IDevicesFactory default
    post-fs-data: mkdir /data/vendor/audiohal 0771 system audio
    NOTE: oneshot — audioflinger restarts the hal via onrestart rule itself,
          not via init restart.

  android.hardware.bluetooth@1.0-service-mediatek.rc
    service bluetooth-1-0
    binary  : /vendor/bin/hw/android.hardware.bluetooth@1.0-service-mediatek
    class   : hal
    user    : bluetooth
    group   : bluetooth
    caps    : SYS_NICE
    NOTE: stopped when vts.native_server.on=1 on userdebug/eng builds only.

  android.hardware.cas@1.1-service.rc
    service vendor.cas-hal-1-1
    binary  : /vendor/bin/hw/android.hardware.cas@1.1-service
    class   : hal
    user    : media
    group   : mediadrm drmrpc
    ioprio  : rt 4

  android.hardware.configstore@1.1-service.rc
    service vendor.configstore-hal
    binary  : /vendor/bin/hw/android.hardware.configstore@1.1-service
    class   : hal animation
    user    : system
    group   : system

  android.hardware.drm@1.0-service.rc
    service vendor.drm-hal-1-0
    binary  : /vendor/bin/hw/android.hardware.drm@1.0-service
    interfaces: android.hardware.drm@1.0::ICryptoFactory default
                android.hardware.drm@1.0::IDrmFactory default
    class   : hal
    user    : media
    group   : mediadrm drmrpc
    ioprio  : rt 4

  android.hardware.drm@1.2-service.clearkey.rc
    service vendor.drm-clearkey-hal-1-2
    binary  : /vendor/bin/hw/android.hardware.drm@1.2-service.clearkey
    interfaces: ICryptoFactory + IDrmFactory clearkey @ 1.0, 1.1, 1.2
    class   : hal
    user    : media
    group   : media mediadrm
    ioprio  : rt 4

  android.hardware.drm@1.2-service.widevine.rc
    service vendor.drm-widevine-hal-1-2
    binary  : /vendor/bin/hw/android.hardware.drm@1.2-service.widevine
    interfaces: ICryptoFactory + IDrmFactory widevine @ 1.0, 1.1, 1.2
                android.hidl.base@1.0::IBase widevine
    class   : hal
    user    : media
    group   : media mediadrm drmrpc system
    ioprio  : rt 4
    service vendor.move_data_sh /system/bin/move_widevine_data.sh
      class late_start, user media, disabled, oneshot
      started on property:init.svc.mediadrm=running

  android.hardware.gatekeeper@1.0-service.rc
    service vendor.gatekeeper-1-0
    binary  : /vendor/bin/hw/android.hardware.gatekeeper@1.0-service
    class   : hal
    user    : system
    group   : system

  android.hardware.gnss@2.0-service-mediatek.rc
    service gnss_service
    binary  : /vendor/bin/hw/android.hardware.gnss@2.0-service-mediatek
    class   : hal
    user    : system
    group   : system gps

  android.hardware.graphics.allocator@2.0-service.rc
    service vendor.gralloc-2-0
    binary  : /vendor/bin/hw/android.hardware.graphics.allocator@2.0-service
    interface: android.hardware.graphics.allocator@2.0::IAllocator default
    class   : hal animation
    user    : system
    group   : graphics drmrpc
    caps    : SYS_NICE
    onrestart: restart surfaceflinger

  android.hardware.graphics.composer@2.1-service.rc
    service vendor.hwcomposer-2-1
    binary  : /vendor/bin/hw/android.hardware.graphics.composer@2.1-service
    class   : hal animation
    user    : system
    group   : graphics drmrpc
    caps    : SYS_NICE
    onrestart: restart surfaceflinger

  android.hardware.health@2.0-service.rc
    service health-hal-2-0
    binary  : /vendor/bin/hw/android.hardware.health@2.0-service
    class   : hal
    user    : system
    group   : system
    caps    : WAKE_ALARM
    file    : /dev/kmsg w

  android.hardware.keymaster@4.0-service.rc
    service vendor.keymaster-4-0
    binary  : /vendor/bin/hw/android.hardware.keymaster@4.0-service
    class   : early_hal
    user    : system
    group   : system drmrpc
    NOTE: early_hal class — starts before vold. Critical for FBE key operations.

  android.hardware.light@2.0-service-mediatek.rc
    service light-hal-2-0
    binary  : /vendor/bin/hw/android.hardware.light@2.0-service-mediatek
    interface: android.hardware.light@2.0::ILight default
    class   : hal
    user    : system
    group   : system
    shutdown: critical

  android.hardware.media.omx@1.0-service.rc
    service mediacodec
    binary  : /vendor/bin/hw/android.hardware.media.omx@1.0-service
    class   : main
    user    : mediacodec
    group   : camera drmrpc mediadrm media_rw sdcard_rw media system audio
              inet net_bt net_bt_admin net_bw_acct sdcard_r
    ioprio  : rt 4

  android.hardware.memtrack@1.0-service.rc
    service vendor.memtrack-hal-1-0
    binary  : /vendor/bin/hw/android.hardware.memtrack@1.0-service
    class   : hal
    user    : system
    group   : system

  android.hardware.secure_element@1.0-service-mediatek.rc
    service mtk_secure_element_hal_service
    binary  : /vendor/bin/hw/android.hardware.secure_element@1.0-service-mediatek
    class   : hal
    user    : secure_element
    group   : secure_element

  android.hardware.sensors@2.0-service-mediatek.rc
    service vendor.sensors-hal-2-0
    binary  : /vendor/bin/hw/android.hardware.sensors@2.0-service-mediatek
    class   : main
    user    : system
    group   : system
    rlimit  : rtprio 10 10

  android.hardware.thermal@1.0-service.rc
    service vendor.thermal-hal-1-0
    binary  : /vendor/bin/hw/android.hardware.thermal@1.0-service
    class   : hal
    user    : system
    group   : system

  android.hardware.usb@1.1-service-mediatek.rc
    service vendor.usb-hal-1-1
    binary  : /vendor/bin/hw/android.hardware.usb@1.1-service-mediatek
    class   : hal
    user    : system
    group   : system
    on boot: chown/chmod dual_role_usb nodes for both type-C and USB2.0 ports:
      /sys/class/dual_role_usb/dual-role-type_c_port0/{mode,power_role,data_role}
      /sys/class/dual_role_usb/dual-role-usb20/{mode,power_role,data_role}

  android.hardware.vibrator@1.0-service.rc
    service vendor.vibrator-1-0
    binary  : /vendor/bin/hw/android.hardware.vibrator@1.0-service
    class   : hal
    user    : system
    group   : system

  android.hardware.wifi@1.0-service-lazy-mediatek.rc
    service vendor.wifi_hal_legacy
    binary  : /vendor/bin/hw/android.hardware.wifi@1.0-service-lazy-mediatek
    interface: android.hardware.wifi@1.0::IWifi default
    oneshot, disabled
    class   : hal
    caps    : NET_ADMIN NET_RAW SYS_MODULE
    user    : wifi
    group   : wifi gps
    NOTE: disabled — started by WiFi framework on demand (lazy HAL).

  atcid.rc
    service atcid-daemon-u
    binary  : /vendor/bin/atcid
    interfaces: vendor.mediatek.hardware.atci@1.0::IAtcid default
    class   : hal + main (dual class declaration)
    socket  : adb_atci_socket stream 660 radio system
    user    : system
    group   : radio system media bluetooth wifi
    disabled, oneshot
    starts on: persist.vendor.service.atci.autostart=1 OR atm_mode=1
    on property:sys.boot_completed=1: setprop vendor.mtk.atci.boot_completed 1

  audiocmdservice_atci.rc
    service audio-daemon
    binary  : /vendor/bin/audiocmdservice_atci
    class   : main
    socket  : atci-audio stream 666 system system
    user    : system
    group   : system audio
    disabled, oneshot

  bootperf.rc
    on early-init && property:ro.boot.boot_trace=1:
      setprop debug.atrace.tags.enableflags 0x1fffffe
      setprop debug.atrace.app_number 1
      setprop debug.atrace.app_0 com.android.systemui
    NOTE: Boot systrace only enabled when ro.boot.boot_trace=1 explicitly set.

  camerahalserver.rc
    service camerahalserver
    binary  : /vendor/bin/hw/camerahalserver
    class   : main
    user    : cameraserver
    group   : audio camera input drmrpc sdcard_rw system media graphics
    ioprio  : rt 4
    caps    : SYS_NICE

  em_hidl_user.rc
    service emhidl_user
    binary  : /vendor/bin/em_hidl
    interface: vendor.mediatek.hardware.engineermode@1.2::IEmd EmHidlServer
    class   : hal
    oneshot, disabled
    user    : system
    group   : bluetooth

  fuelgauged_init.rc
    service fuelgauged
    binary  : /vendor/bin/fuelgauged
    class   : core
    user    : system
    group   : system

  fuelgauged_nvram_init.rc
    service fuelgauged_nvram
    binary  : /vendor/bin/fuelgauged_nvram
    class   : main
    user    : system
    group   : system
    oneshot

  gsm0710muxd.rc
    service vendor.gsm0710muxd
    binary  : /vendor/bin/gsm0710muxd -s /dev/ttyC0 -f 512 -n 8 -m basic
    class   : main
    user    : root
    group   : radio cache inet misc
    disabled, oneshot
    NOTE: args match live process confirmed in ps -A output.

  hostapd.android.rc
    service hostapd
    binary  : /vendor/bin/hw/hostapd
    interfaces: android.hardware.wifi.hostapd@1.0 and @1.1
    class   : main
    caps    : NET_ADMIN NET_RAW
    user    : wifi
    group   : wifi net_raw net_admin
    disabled, oneshot
    post-fs-data: mkdir /data/vendor/wifi/hostapd/sockets 0770 wifi wifi

  init.bip.rc
    service vendor.bip
    binary  : /vendor/bin/bip
    class   : core
    socket  : vendor.bip stream 660 root system
    user    : root
    group   : system log inet radio net_admin root
    oneshot, disabled
    lifecycle: start on vendor.volte_md_status=ready
               stop on vendor.volte_md_status={reset,stop,flightmode,exception}

  init.bt_drv.rc
    on property:vendor.connsys.driver.ready=yes:
      insmod /vendor/lib/modules/bt_drv.ko

  init.cccifsd.rc
    service vendor.ccci_fsd /vendor/bin/ccci_fsd 0
      user radio, group radio system root, class core
      priority -20, ioprio rt 4, oneshot
    service vendor.ccci3_fsd /vendor/bin/ccci_fsd 2
      user radio, group radio system, class core, oneshot
    post-fs-data:
      mkdir /data/vendor/ccci_cfg 0770 system radio
      mkdir /data/vendor/mdlpm 0770 system radio
      mkdir /data/vendor/md3 0770 system radio
      restorecon_recursive /mnt/vendor/protect_f, protect_s

  init.cccimdinit.rc
    service ccci_mdinit /vendor/bin/ccci_mdinit 0
      user system, group radio system, class core, oneshot
    service ccci3_mdinit /vendor/bin/ccci_mdinit 2
      user system, group radio system, class core, oneshot
    post-fs-data:
      mkdir /mnt/vendor/nvdata/md 0770 root system
      mkdir /mnt/vendor/nvdata/md3 0770 root system
      mkdir /data/vendor/md3 0770 root system
      chown radio radio /sys/kernel/ccci/boot
    NOTE: CCCI kernel modules commented out (build-in modified).

  init.cccirpcd.rc
    service vendor.ccci_rpcd /vendor/bin/ccci_rpcd 0
      user radio, group radio system, class core, oneshot

  init_connectivity.rc [ACTIVE connectivity init — loaded at runtime]
    service wmt_loader /vendor/bin/wmt_loader
      class early_hal, user system, group system, oneshot
    service wmt_launcher /vendor/bin/wmt_launcher -p /vendor/firmware/ -o 1
      class early_hal, user system, group system
    service stp_dump /vendor/bin/stp_dump3
      class core, user system, group system sdcard_rw misc, disabled
    post-fs-data:
      mkdir /data/vendor/stp_dump 0755 system system
      mkdir /data/vendor/connsyslog 0755 system system
    property triggers:
      ro.build.type=eng → enable kernel tracing
      persist.vendor.connsys.coredump.mode={1,2} → start stp_dump
      persist.vendor.connsys.coredump.mode=0 → stop stp_dump
      vendor.connsys.driver.ready=yes → write wmt_dbg coredump mode

  init.gps_drv.rc
    on property:vendor.connsys.driver.ready=yes:
      insmod /vendor/lib/modules/gps_drv.ko

  init.md_apps.rc
    on boot:
      setprop ro.vendor.md_apps.support 1
      setprop ro.vendor.md_apps.load_type user
      setprop ro.vendor.md_apps.load_verno "MOLY.LR12A.R3.MP.V107.4"
      setprop ro.vendor.md_apps.load_gencfg "GEN93_USER"
      setprop ro.vendor.md_apps.load_date "2021/01/20 17:32:43 GMT +08:00"

  init.thermal.rc
    service thermal /vendor/bin/thermal
      class main, user system, group system

  init.thermal_manager.rc
    service thermal_manager /vendor/bin/thermal_manager
      class main, user root, group system, oneshot

  init.thermalloadalgod.rc
    service thermalloadalgod /vendor/bin/thermalloadalgod
      class main, user system, group system

  init.volte_imcb.rc
    service vendor.volte_imcb /vendor/bin/volte_imcb
    class   : main
    sockets : volte_imsa1 stream 660 root radio
              volte_imsa2 stream 660 root radio
              volte_ut stream 660 root radio
    user    : root
    group   : system log inet radio
    oneshot, disabled
    NOTE: volte_imsvt1 and volte_imsvt2 sockets COMMENTED OUT in this file.
          volte_imsvt1 instead declared in mtkrild.rc (rild socket).
          This is an ODM socket allocation conflict — documented, not resolved.

  init.volte_imsm_93.rc
    service volte_imsm_93 /vendor/bin/volte_imsm_93
    class   : main
    user    : root
    group   : radio cache inet misc log system wifi
    oneshot, disabled
    lifecycle: on vendor.volte_md_status=ready:
                 restart volte_imsm_93, volte_stack, volte_ua, volte_imcb
               on vendor.volte_md_status={reset,stop,flightmode,exception}:
                 rm /dev/radio/ptty{ims,2ims,3ims,4ims}
                 stop volte_imcb, volte_ua, volte_stack, volte_imsm_93

  init.volte_md_status.rc
    service volte_md_status /system/vendor/bin/volte_md_status
    class   : core
    user    : root
    group   : radio log system
    oneshot, disabled
    NOTE: restarted on init trigger — starts very early.

  init.volte_stack.rc
    service vendor.volte_stack /vendor/bin/volte_stack
    class   : main
    socket  : volte_stack stream 660 root system
    user    : root
    group   : system log inet radio
    oneshot, disabled

  init.volte_ua.rc
    service vendor.volte_ua /vendor/bin/volte_ua
    class   : main
    socket  : volte_ua stream 660 root system
    user    : root
    group   : system log inet radio net_admin root
    oneshot, disabled

  init.vtservice_hidl.rc
    service vtservice_hidl /vendor/bin/hw/vtservice_hidl
    class   : hal
    socket  : volte_vt stream 660 root system
    user    : system
    group   : system camera graphics inet net_admin net_raw media
              sdcard_rw radio sdcard_r media_rw
    ioprio  : rt 4

  init.wfca.rc
    service wfca /vendor/bin/wfca
    class   : core
    sockets : wfca stream 660 root system
              wfca_rds stream 660 root system
    user    : root
    group   : system log inet radio net_admin root

  init.wlan_drv.rc
    on property:vendor.connsys.driver.ready=yes:
      insmod /vendor/lib/modules/wmt_chrdev_wifi.ko
      insmod /vendor/lib/modules/wlan_drv_${ro.vendor.wlan.gen}.ko
      start wlan_assistant
    NOTE: ro.vendor.wlan.gen=gen4m → loads wlan_drv_gen4m.ko

  init.wmt_drv.rc
    on boot:
      insmod /vendor/lib/modules/wmt_drv.ko
    NOTE: wmt_drv.ko is the parent connectivity module — must load first.
          bt_drv, gps_drv, wmt_chrdev_wifi, wlan_drv_gen4m all depend on it.

  init.wod.rc
    service vendor.epdg_wod /vendor/bin/epdg_wod
    class   : late_start
    sockets : wod_action stream 660 root system
              wod_sim stream 660 root system
    user    : root
    group   : root system log inet radio net_admin
    disabled
    post-fs-data:
      mkdir /data/vendor/ipsec 0700 root root
      mkdir /data/vendor/ipsec/wo 0700 root root
      mkdir /dev/vendor/net/ 0700 root root
    lifecycle:
      start on vendor.volte_md_status=ready AND persist.vendor.mtk_wfc_support=1
      stop on vendor.volte_md_status={reset,stop,flightmode,exception}
      on restart: setprop vendor.ril.mux.report.case 2, vendor.ril.muxreport 1

  ipsec_mon.rc
    service ipsec_mon /vendor/bin/ipsec_mon
    class   : main
    user    : system
    group   : net_admin net_raw system misc
    caps    : NET_ADMIN NET_RAW

  lbs_hidl_service.rc
    service lbs_hidl_service /vendor/bin/lbs_hidl_service
    class   : hal
    user    : system
    group   : system gps radio inet sdcard_r sdcard_rw

  mtk_agpsd_p.rc
    service agpsd /vendor/bin/mtk_agpsd
    class   : main
    sockets : agpsd stream 660 gps system
              agpsd2 stream 660 gps inet
              agpsd3 stream 660 gps inet
    user    : gps
    group   : gps radio inet sdcard_r sdcard_rw misc system
    post-fs-data: mkdir /data/vendor/agps_supl 0771 gps system

  mtkrild.rc
    service vendor.ril-daemon-mtk
    binary  : /vendor/bin/hw/mtkfusionrild -l libmtk-ril.so
    class   : main
    sockets : sap_uim_socket1 stream 660 bluetooth bluetooth
              rild-oem stream 660 radio system
              volte_imsvt1 stream 660 root radio
              rild-atci stream 660 root radio
    user    : radio
    group   : radio cache inet misc audio log readproc wakelock
              sdcard_r sdcard_rw system
    caps    : BLOCK_SUSPEND NET_ADMIN NET_RAW
    oneshot, disabled
    starts on: vendor.ril.mtk=1 OR vold.decrypt=trigger_restart_framework
    post-fs-data: mkdir /data/vendor/radio 0771 radio radio

  muxreport.rc
    service vendor.muxreport-daemon /vendor/bin/muxreport
    class   : core
    user    : radio
    group   : radio cache inet misc
    disabled, oneshot
    starts on: vendor.ril.muxreport=1

  netdagent.rc
    service netdagent /vendor/bin/netdagent
    class   : late_start
    user    : system
    group   : net_admin net_raw system misc
    caps    : NET_ADMIN NET_RAW
    on init: setprop ro.vendor.net.upload.benchmark.default blocking
    on property:persist.vendor.operator.optr=OP12:
      write /proc/sys/net/optr 12

  nvram_daemon.rc
    service nvram_daemon /vendor/bin/nvram_daemon
    class   : core
    user    : root
    group   : system radio wifi audio bluetooth media
    oneshot

  vendor.mediatek.hardware.gpu@1.0-service.rc
    service gpu-1-0
    binary  : /vendor/bin/hw/vendor.mediatek.hardware.gpu@1.0-service
    class   : hal
    user    : system
    group   : system graphics

  vendor.mediatek.hardware.mms@1.3-service.rc
    service mms-1-3
    binary  : /vendor/bin/hw/vendor.mediatek.hardware.mms@1.3-service
    class   : main
    user    : system
    group   : system graphics

  vendor.mediatek.hardware.mtkcodecservice@1.1-service.rc
    service mtkcodecservice-1-1
    binary  : /vendor/bin/hw/vendor.mediatek.hardware.mtkcodecservice@1.1-service
    interface: vendor.mediatek.hardware.mtkcodecservice@1.1::IMtkCodecService default
    oneshot, disabled
    class   : hal
    user    : system
    group   : audio media sdcard_r

  vendor.mediatek.hardware.mtkpower@1.0-init.rc
    on post-fs-data (extensive — full list):
      EAS/sched nodes: schedtune.boost, perfserv_uclamp_min/fg/bg/ta,
                       io/schedtune.uclamp_min, io/schedtune.util.min,
                       idle_prefer, top-app/foreground prefer_idle,
                       io/schedtune.boost, sched_boost, background/boost
      perfmgr nodes: perfserv_schedplus_down/up_throttle, sync_flag,
                     cpu_prefer, legacy/perfserv_core/freq, ta/fg/bg_boost,
                     perfserv_core/freq, ext_launch_mon, sched_migrate_cost,
                     big_task_rotation, stune_task_thresh, topo_ctrl nodes,
                     tchbst/usrtch, cfp_enable/polling_ms/up/down_loading/time/opp
      GED/GPU nodes: gx_top_app_pid, gx_3D_benchmark_on, gx_force_cpu_boost,
                     timer_base_dvfs_margin, loading_base_dvfs_step, dvfs_margin_value
      PPM nodes: policy_status, userlimit_boost, hard_userlimit_cpu_freq
      Cache: /sys/module/ca_drv/parameters/enable
      FPSGO: bhr_opp, floor_bound, kmin, fstb_tune_*, fstb_fps_list,
             fstb_fteh_list, eara ai_bench, gpu_block_boost, boost_ta,
             margin_mode/dbnc, eara_thermal/enable, rescue_percent/enhance,
             ultra_rescue, deqtime_bound, short_rescue_ns, min_rescue_percent,
             adjust_loading, loading_adj/debnc/time/th
      VPU/MDLA: /sys/kernel/debug/mdla/power, vpu/power
      NET: /proc/net/wlan/setCAM
      FS: /sys/fs/f2fs/sdc42, /sys/fs/ext4/sdc42, mmcblk0p42
          NOTE: references sdc42 and mmcblk0p42 — these are BSP template
                references, not this device's actual partition numbers.
      BLKDEV: sdc/queue/read_ahead_kb, dm-2/queue/read_ahead_kb,
              mmcblk0/queue/read_ahead_kb

  vendor.mediatek.hardware.mtkpower@1.0-service.rc
    service power-hal-1-0
    binary  : /vendor/bin/hw/vendor.mediatek.hardware.mtkpower@1.0-service
    class   : hal
    caps    : NET_ADMIN NET_RAW SYS_MODULE
    user    : system
    group   : system readproc wifi media media_rw camera audio
    on property:vold.decrypt=trigger_shutdown_framework:
      setprop vendor.powerhal.init 0

  vendor.mediatek.hardware.nvram@1.1-sevice.rc  [ODM TYPO — filename preserved]
    service nvram-hidl-1-1
    binary  : /vendor/bin/hw/vendor.mediatek.hardware.nvram@1.1-service
      NOTE: service binary name is correct — only the RC filename has the typo.
    class   : hal
    user    : root
    group   : system bluetooth media audio radio

  vendor.mediatek.hardware.pq@2.2-service.rc
    service pq-2-2
    binary  : /vendor/bin/hw/vendor.mediatek.hardware.pq@2.2-service
    class   : main
    user    : system
    group   : system graphics

  vndservicemanager.rc
    service vndservicemanager /vendor/bin/vndservicemanager /dev/vndbinder
    class   : core
    user    : system
    group   : system readproc
    writepid: /dev/cpuset/system-background/tasks
    shutdown: critical

  wlan_assistant.rc
    service wlan_assistant /vendor/bin/wlan_assistant
    class   : main
    user    : wifi
    group   : system gps inet

# 1.3 BOOT SEQUENCE MAP

  TIMING SOURCE: /proc/bootprof (live device capture)

  Stage 0 — Preloader
    Duration : 4,671ms
    Source   : /dev/block/mmcblk0boot0 (preloader_a)

  Stage 1 — LK (Little Kernel bootloader)
    Duration : 8,129ms total (logo display: 998ms)
    Source   : /dev/block/by-name/lk (mmcblk0p26)

  Stage 2 — Kernel init
    Timestamp: 34.3ms after kernel handoff (ON marker)
    Key kernel initcalls (from bootprof):
       298ms : init_menu
       333ms : of_platform_default_populate_init
       393ms : ccci_util_init
       940ms : mt6577-uart init (UART driver — 476ms probe)
       963ms : second mt6577-uart probe (21ms)
       999ms : __mt_gpufreq_init
     1,070ms : musb-hdrc init (USB — 40ms probe)
     8,405ms : wiite_corp I2C probe — 7,259ms
               NOTE: nRF52832 takes 7.26 seconds to respond to I2C probe.
               This is the confirmed nRF52832 probe delay. ALL services
               that depend on sensor data must account for this delay.
     8,462ms : mt6370_pmu_charger probe
     8,469ms : mt_i2c_init complete (7,326ms total for all I2C)
     8,637ms : mmcblk probe complete
     9,117ms : mtkfb (display) probe — 584ms
    10,071ms : hyn_ts (Hynitron touchscreen) probe — 150ms
    10,254ms : Kernel_init_done

  Stage 3 — Init process (userspace)
    12,265ms : INIT:early-init
               → perfmgr boot boost, debugfs mount
    12,488ms : INIT:late-init
               → rcu_expedited enable
    12,503ms : INIT:Mount_START
               → wait_for_prop hwservicemanager.ready
               → start vendor.boot-hal-1-0
               → mount_all /vendor/etc/fstab.mt6765
    13,147ms : INIT:Mount_END
               → 644ms for complete partition mount
               → also mounts iAmCdRom.iso at /mnt/cd-rom
    NOTE: INIT:post-fs and INIT:post-fs-data not captured in this
          bootprof run — device was already booted, markers already cleared.

  Stage 4 — Framework
    34,424ms : BOOT_Animation:END
    36,937ms : AP_Init complete (com.android.bluetooth last service)
    36,953ms : bootprof OFF (sys.boot_completed=1 triggered)

  TOTAL BOOT TIME (cold to usable): ~37 seconds

  CRITICAL ORDERING CONSTRAINTS:
    1. hwservicemanager must be ready before mount_all (wait_for_prop)
    2. vendor.keymaster-4-0 (early_hal) must start before vold
    3. vold must start after all post-fs-data work completes
    4. vold.post_fs_data_done=1 set by init.modem.rc (last import)
    5. wiite_corp_ctrl sysfs available after 8,405ms kernel probe
    6. wmt_drv.ko insmod on boot trigger (before connsys.driver.ready)
    7. bt_drv, gps_drv, wlan_drv all insmod AFTER vendor.connsys.driver.ready=yes
    8. wlan_drv loads wlan_drv_gen4m.ko then starts wlan_assistant

# 1.4 LIVE SERVICE STATE (from ps -A — live device capture 2026-05-09)

  KERNEL THREADS (PID 2, kthreadd parent):
    [ksoftirqd/0-7]    — softirq handlers per CPU
    [kworker/N:M]      — kernel work queues (numerous)
    [migration/0-7]    — CPU migration threads
    [cmdq_flushq]×16   — MTK CMDQ display command queue flush workers
    [cmdq_auto_relea]×16 — CMDQ auto-release workers
    [cmdq_resource]    — CMDQ resource manager
    [cmdq_mdp_task]    — CMDQ MDP task handler
    [irq/27-WIITE_CO]  — wiite_corp_ctrl interrupt thread (nRF52832)
    [irq/25-WIITE_CO]  — wiite_con_ctrl interrupt thread
    [irq/30-mt6370_p]  — PMIC interrupt thread
    [ccci_fsm1]        — CCCI modem FSM
    [ccci_rpc_k]       — CCCI RPC kernel thread
    [ccci_udc]         — CCCI UDC thread
    [ccci_ipc_3/6/7/8] — CCCI IPC threads
    [ccci_sys/ctrl]    — CCCI system/control
    [md1_cldma_worker] — CLDMA modem DMA
    [md1_tx0-3_worker] — modem TX workers
    [md1_rx0-7_worker] — modem RX workers
    [cldma_rxq0]       — CLDMA RX queue
    [pvr_timer/defer/wdg/cacheop/misr] — PowerVR GPU kernel threads
    [EEM_CTRL_L/2L/CCI/L_HI/2L_HI]    — CPU voltage EEM controllers
    [hps_main]         — CPU hotplug strategy
    [btif_rxd]         — Bluetooth RX data
    [accel/baro/temp/alti_polling]      — sensor polling threads
    [conn-md-thread]   — connectivity modem thread
    [mtk_stp_psm/btm/wmtd/wmtd_worker] — WMT STP stack
    [jbd2/mmcblk0p10]  — metadata journal
    [jbd2/mmcblk0p36]  — userdata journal
    [jbd2/mmcblk0p35]  — cache journal
    [jbd2/mmcblk0p12/13/9/8] — protect_f, protect_s, nvdata, nvcfg journals
    [mmcqd/0]          — eMMC main queue
    [mmcqd/0boot0/1]   — boot partition queues
    [mmcqd/0rpmb]      — RPMB queue
    [wdtk-0 through 7] — watchdog threads per CPU

  INIT SERVICES — CORE CLASS (start earliest, PID order):
    PID  359  logd              (logd)
    PID  360  servicemanager    (system)
    PID  361  hwservicemanager  (system)
    PID  362  vndservicemanager (system)
    PID  369  volte_md_status   (root)
    PID  372  vold              (root)
    PID  413  vendor.keymaster-4-0  (system) ← early_hal class
    PID  415  wmt_launcher      (system)
    PID  554  fuelgauged        (system)
    PID  556  ccci_fsd [0]      (radio)
    PID  557  ccci_fsd [util]   (radio)
    PID  559  ccci_mdinit       (system)
    PID  561  ccci_rpcd         (radio)
    PID  563  wfca              (root)
    PID  541  nvram-hidl-1-1    (root)

  INIT SERVICES — HAL CLASS:
    PID  412  android.system.suspend@1.0-service  (system)
    PID  515  android.hidl.allocator@1.0-service  (system)
    PID  516  vendor.audio-hal-2-0        (audioserver)
    PID  517  bluetooth-1-0               (bluetooth)
    PID  518  vendor.cas-hal-1-1          (media)
    PID  519  vendor.configstore-hal      (system)
    PID  520  vendor.drm-hal-1-0          (media)
    PID  521  vendor.drm-clearkey-hal-1-2 (media)
    PID  522  vendor.drm-widevine-hal-1-2 (media)
    PID  523  vendor.gatekeeper-1-0       (system)
    PID  524  gnss_service                (system)
    PID  525  vendor.gralloc-2-0          (system)
    PID  526  vendor.hwcomposer-2-1       (system)
    PID  527  health-hal-2-0              (system)
    PID  528  light-hal-2-0               (system)
    PID  529  vendor.memtrack-hal-1-0     (system)
    PID  530  mtk_secure_element_hal_service (secure_element)
    PID  532  vendor.thermal-hal-1-0      (system)
    PID  533  vendor.usb-hal-1-1          (system)
    PID  536  vendor.vibrator-1-0         (system)
    PID  537  vtservice_hidl              (system)
    PID  538  lbs_hidl_service            (system)
    PID  539  vendor.gpu-1-0              (system)
    PID  540  power-hal-1-0               (system)
    PID  669  vendor.sensors-hal-2-0      (system)
    PID 1236  vendor.wifi_hal_legacy      (wifi) ← lazy, started on demand

  INIT SERVICES — MAIN/LATE_START CLASS:
    PID  511  netd                    (root)
    PID  512  zygote64                (root)
    PID  513  zygote                  (root)
    PID  542  ashmemd                 (nobody)
    PID  545  audioserver             (audioserver)
    PID  548  gpuservice              (gpu_service)
    PID  550  lmkd                    (lmkd)
    PID  551  surfaceflinger          (system)
    PID  631  mnld                    (gps)
    PID  632  lbs_dbg                 (system)
    PID  634  cameraserver            (cameraserver)
    PID  635  drmserver               (drm)
    PID  637  incidentd               (incidentd)
    PID  639  thermald                (system)
    PID  640  vtservice               (system)
    PID  641  installd                (root)
    PID  642  keystore                (keystore)
    PID  643  mediadrmserver          (media)
    PID  644  media.extractor         (mediaex)
    PID  645  media.metrics           (media)
    PID  646  mediaserver             (media)
    PID  647  mobile_log_d            (system)
    PID  652  netdiag                 (shell)
    PID  654  statsd                  (statsd)
    PID  659  storaged                (root)
    PID  664  wificond                (wifi)
    PID  667  media.codec             (mediacodec)
    PID  671  camerahalserver         (cameraserver)
    PID  690  thermal                 (system)
    PID  701  thermalloadalgod        (system)
    PID  703  ipsec_mon               (system)
    PID  704  mtk_agpsd               (gps)
    PID  705  vendor.mms-1-3          (system)
    PID  706  vendor.pq-2-2           (system)
    PID  709  wlan_assistant          (wifi)
    PID  710  media.swcodec           (mediacodec)
    PID  713  gatekeeperd             (system)
    PID  715  tombstoned              (tombstoned)
    PID  719  netdagent               (system)
    PID  887  vendor.bip              (root)
    PID  889  volte_imsm_93           (root)
    PID  890  vendor.volte_stack      (root)
    PID  893  vendor.volte_ua         (root)
    PID  903  vendor.volte_imcb       (system)
    PID  905  vendor.epdg_wod         (root)
    PID  924  vendor.gsm0710muxd      (radio)
    PID  995  vendor.ril-daemon-mtk   (radio) [mtkfusionrild]
    PID 1079  system_server           (system) ← spawned from zygote64 PID 512
    PID 1286  wpa_supplicant          (wifi)
    PID 1915  batterywarning          (system)
    PID 1934  adbd                    (shell)
    PID 2898  starter                 (root)
    PID 2907  charon                  (root) ← IPsec/StrongSwan, child of starter

  ODM PROCESSES — ALL RUNNING AT CAPTURE TIME:
    PID 1328  com.wiite.WiiteCommon                  (system UID)
    PID 2106  com.wiite.devicedaemon                 (system UID)
    PID 2157  com.wiite.cleantask                    (system UID)
    PID 2411  com.wiite.devicedaemon:communication   (system UID) ← ttyS1 owner
    PID 2610  com.wiite.wearhealthuart               (system UID)
    PID 2619  com.wiite.wiite_heartrate              (system UID)
    PID 2628  com.wiite.wiite_sleep                  (system UID)
    PID 2649  com.wiite.wiite_bloodoxygen            (system UID)
    PID 4729  com.wiite.power                        (system UID)
    PID 4757  com.wiite.wiitestore                   (system UID)
    PID 2124  com.adups.fota                         (u0_a41)
    NOTE: com.adups.fota running despite being marked hardcoded unkillable
          in AbSharedUtil.java. Confirmed active at capture time.

  MAGISK PROCESSES:
    PID  466  magiskd          (root) ← primary daemon
    PID 22560  magiskd         (root) ← secondary instance
    PID 12731  com.topjohnwu.magisk (u0_a56)
    PID 12773  com.topjohnwu.magisk:root:0 (root)
    PID 12765  libbusybox.so   (root) ← Magisk busybox

  SERVICES NOT RUNNING AT CAPTURE TIME (disabled/oneshot/demand):
    atcid-daemon-u, audiocmdservice_atci, emhidl_user, fuelgauged_nvram,
    hostapd, meta_tst, loghidlsysservice, osi, vendor.move_data_sh,
    vendor.muxreport-daemon, mtkcodecservice-1-1, bugreport, pppoe_wlan0,
    cmmb, stp_dump

# 1.5 NOTABLE FINDINGS

  iAmCdRom.iso:
    Mounted at /mnt/cd-rom (iso9660, read-only) on every boot via init.mt6762.rc.
    File located at /vendor/etc/iAmCdRom.iso.
    Purpose unknown. Requires binwalk + strings analysis.
    Status: documented, unanalyzed.

  init.connectivity.rc vs init_connectivity.rc discrepancy:
    hw/init.connectivity.rc     — class core, no -o 1 flag (older template)
    flat/init_connectivity.rc   — class early_hal, -o 1 flag (active at runtime)
    The flat version is loaded via import ${ro.vendor.rc}init.connectivity.rc.
    The hw/ version is the MTK reference, the flat version is the ODM override.

  volte_imsvt1 socket conflict:
    init.volte_imcb.rc defines volte_imsvt1 as COMMENTED OUT.
    mtkrild.rc defines volte_imsvt1 as an active socket on mtkfusionrild.
    At runtime volte_imsvt1 is owned by mtkfusionrild, not volte_imcb.
    This is an intentional ODM reconfiguration — socket moved to RIL side.

  A/B preloader symlinks on A-only device:
    init.mt6762.rc creates preloader_a → mmcblk0boot0 and preloader_b → mmcblk0boot1.
    Device is A-only. These symlinks are MTK BSP template remnants.
    They exist but serve no function on this device.

  Mali GPU driver commented out:
    init.mt6762.rc has mali.ko insmod COMMENTED OUT.
    /dev/mali0 chmod 666 still runs.
    This device uses PowerVR GE8320 (rgx driver), not Mali.
    Mali node exists as a BSP artifact, device node permissioned but driver absent.

  mtkpower@1.0-init.rc references sdc42 and mmcblk0p42:
    These partition references do not exist on this device (38 partitions max is p38).
    BSP template references for a platform with different partition layout.
    The chown/chmod commands for those nodes will fail silently on boot.

################################################################################
# SECTION 2 — HAL STACK
################################################################################

# 2.1 VINTF MANIFEST

  Location       : /vendor/etc/vintf/manifest.xml
  Manifest type  : device
  Target level   : 4
  SEPolicy       : 29.0

  Source fragments (compiled into manifest.xml — comment header confirms):
    manifest_target_level_q.xml
    manifest_audio.xml
    manifest_ss_se.xml
    manifest_radioconfig.xml
    manifest_em.xml
    manifest_bgservice.xml
    manifest_op_ss.xml
    manifest_wifi.xml
    manifest_videotelephony.xml
    manifest_km4.xml
    manifest_netdagent.xml
    manifest_pq.xml
    manifest_gnss.xml
    manifest.xml

  AOSP STANDARD HALs (from /vendor/etc/vintf/manifest.xml):

  HAL                                   Version   Transport      Interface / Instance
  ──────────────────────────────────────────────────────────────────────────────────────
  android.hardware.audio                5.0       hwbinder       IDevicesFactory/default
  android.hardware.audio.effect         5.0       hwbinder       IEffectsFactory/default
  android.hardware.bluetooth            1.0       hwbinder       IBluetoothHci/default
  android.hardware.bluetooth.audio      2.0       hwbinder       IBluetoothAudioProvidersFactory/default
  android.hardware.camera.provider      2.4       hwbinder       ICameraProvider/internal/0
  android.hardware.configstore          1.1       hwbinder       ISurfaceFlingerConfigs/default
  android.hardware.drm                  1.0/1.2   hwbinder       ICryptoFactory/default
                                                                 IDrmFactory/default
                                                                 ICryptoFactory/clearkey (1.2)
                                                                 ICryptoFactory/widevine (1.2)
                                                                 IDrmFactory/clearkey (1.2)
                                                                 IDrmFactory/widevine (1.2)
  android.hardware.gatekeeper           1.0       hwbinder       IGatekeeper/default
  android.hardware.gnss                 1.1/2.0   hwbinder       IGnss/default
  android.hardware.graphics.allocator   2.0       hwbinder       IAllocator/default
  android.hardware.graphics.composer    2.1       hwbinder       IComposer/default
  android.hardware.graphics.mapper      2.1       passthrough    IMapper/default (arch 32+64)
  android.hardware.health               2.0       hwbinder       IHealth/default
  android.hardware.keymaster            4.0       hwbinder       IKeymasterDevice/default
  android.hardware.light                2.0       hwbinder       ILight/default
  android.hardware.media.omx            1.0       hwbinder       IOmx/default
                                                                 IOmxStore/default
  android.hardware.memtrack             1.0       hwbinder       IMemtrack/default
  android.hardware.power                1.3       hwbinder       IPower/default
  android.hardware.radio                1.2/1.4   hwbinder       override=true
                                                                 IRadio/slot1, imsAospSlot1, se1, em1 (1.2)
                                                                 ISap/slot1 (1.2)
                                                                 IRadio/slot1, imsAospSlot1, se1, em1 (1.4)
  android.hardware.radio.config         1.1       hwbinder       IRadioConfig/default
  android.hardware.secure_element       1.0       hwbinder       override=true
                                                                 ISecureElement/SIM1
  android.hardware.sensors              2.0       hwbinder       ISensors/default
  android.hardware.thermal              1.0       hwbinder       IThermal/default
  android.hardware.usb                  1.1       hwbinder       IUsb/default
  android.hardware.vibrator             1.0       hwbinder       IVibrator/default
  android.hardware.wifi                 1.3       hwbinder       IWifi/default
  android.hardware.wifi.hostapd         1.1       hwbinder       IHostapd/default
  android.hardware.wifi.supplicant      1.2       hwbinder       ISupplicant/default

  MEDIATEK PROPRIETARY HALs (from /vendor/etc/vintf/manifest.xml):

  HAL                                           Version   Interface / Instance
  ──────────────────────────────────────────────────────────────────────────────────────
  vendor.mediatek.hardware.atci                 1.0       IAtcid/default
  vendor.mediatek.hardware.camera.bgservice     1.1       IBGService/internal/0
  vendor.mediatek.hardware.engineermode         1.2       IEmd/EmHidlServer
  vendor.mediatek.hardware.gpu                  1.0       IGraphicExt/default
  vendor.mediatek.hardware.lbs                  1.0       ILbs/ (18 instances)
                                                            AgpsDebugInterface, AgpsInterface,
                                                            mtk_agps2framework, mtk_debugService2mnld,
                                                            mtk_framework2agps, mtk_lbs_log_v2s,
                                                            mtk_lppe_socket_agps, mtk_lppe_socket_bt,
                                                            mtk_lppe_socket_ipaddr, mtk_lppe_socket_lbs,
                                                            mtk_lppe_socket_network, mtk_lppe_socket_sensor,
                                                            mtk_lppe_socket_wlan, mtk_meta2mnld,
                                                            mtk_mnld2debugService, mtk_mnld2mtklogger,
                                                            mtk_mnld2nlputils, mtk_mtklogger2mnld
  vendor.mediatek.hardware.mms                  1.3       IMms/default
  vendor.mediatek.hardware.mtkcodecservice      1.1       IMtkCodecService/default
  vendor.mediatek.hardware.mtkpower            1.0       IMtkPerf/default
                                                            IMtkPower/default
  vendor.mediatek.hardware.mtkradioex          1.5       override=true
                                                            IMtkRadioEx/ (7 instances)
                                                            imsSlot1, mtkAssist1, mtkCap1, mtkEm1,
                                                            mtkRcs1, mtkSe1, mtkSlot1
  vendor.mediatek.hardware.netdagent           1.0       INetdagent/default
  vendor.mediatek.hardware.nvram               1.1       INvram/default
  vendor.mediatek.hardware.power              2.1       IPower/default
  vendor.mediatek.hardware.pq                 2.3       IPictureQuality/default
  vendor.mediatek.hardware.radio_op           2.0       IRadioOp/OpImsRILd1
                                                            IRadioOp/slot1
  vendor.mediatek.hardware.videotelephony     1.0       IVideoTelephony/default
  vendor.mediatek.hardware.wifi.supplicant    2.0       ISupplicant/default

  NOTE: vendor.mediatek.hardware.wifi.supplicant declared as 2.0 in manifest.
        Library vendor.mediatek.hardware.wifi.supplicant@2.1.so also present in
        vendor/lib64/ — 2.1 library ships but only 2.0 is declared.

  NOTE: android.hardware.radio has override=true — ODM overrides AOSP radio HAL.
        android.hardware.secure_element has override=true — ODM overrides SE HAL.
        vendor.mediatek.hardware.mtkradioex has override=true.
        All three are refined further by the active ODM manifest (manifest_ss.xml).

# 2.2 FRAMEWORK VINTF MANIFEST

  Location : /system/etc/vintf/manifest.xml
  Type     : framework

  Framework HALs provided to vendor:
    android.frameworks.displayservice      1.0  hwbinder  IDisplayService/default
    android.frameworks.schedulerservice   1.0  hwbinder  ISchedulingPolicyService/default
    android.frameworks.sensorservice      1.0  hwbinder  ISensorManager/default
    android.hidl.manager                  1.2  hwbinder  IServiceManager/default
    android.hidl.memory                   1.0  passthrough (32+64) IMapper/ashmem
    android.hidl.token                    1.0  hwbinder  ITokenManager/default
    android.system.net.netd               1.1  hwbinder  INetd/default
    android.system.wifi.keystore          1.0  hwbinder  IKeystore/default
    netutils-wrapper                      1.0  native

  Framework manifest subdirectory (/system/etc/vintf/manifest/):
    android.frameworks.stats@1.0-service.xml        → IStats/default
    android.hidl.allocator@1.0-service.xml          → IAllocator/ashmem
    android.system.suspend@1.0-service.xml          → ISystemSuspend/default
    manifest_android.frameworks.cameraservice.service@2.0.xml → ICameraService/default
    manifest_media_c2_software.xml                  → IComponentStore/software (Codec 2.0)

  Framework compatibility matrices (/system/etc/vintf/):
    compatibility_matrix.1.xml   — FCM level 1
    compatibility_matrix.2.xml   — FCM level 2
    compatibility_matrix.3.xml   — FCM level 3
    compatibility_matrix.4.xml   — FCM level 4 (active for this device)
    compatibility_matrix.legacy.xml
    compatibility_matrix.device.xml — type=framework, sepolicy 29.0, avb 1.0

# 2.3 ODM VINTF MANIFESTS

  Location : /vendor/odm/etc/vintf/
  All are type=device with override=true entries only.
  Runtime selector : manifest_ss.xml (Single SIM — active on this device)

  File                        Size    Purpose
  ──────────────────────────────────────────────────────────────
  manifest_ss.xml             1,473   Single SIM — ACTIVE CONFIG
  manifest_ss_se.xml          1,578   Single SIM + SIM SE
  manifest_ss_se_ese.xml      1,616   Single SIM + SIM SE + eSE
  manifest_dsds.xml           2,140   Dual SIM
  manifest_dsds_se.xml        2,283   Dual SIM + SIM SE
  manifest_dsds_se_ese.xml    2,321   Dual SIM + SIM SE + eSE
  manifest_qsqs.xml           3,284   Quad SIM
  manifest_tsts.xml           2,617   Triple SIM

  ACTIVE MANIFEST — manifest_ss.xml:
    android.hardware.radio       1.2/1.4  override=true
      IRadio/slot1, imsAospSlot1, se1, em1
      ISap/slot1
    vendor.mediatek.hardware.mtkradioex  1.5  override=true
      IMtkRadioEx/mtkSlot1, imsSlot1, mtkSe1, mtkEm1,
                  mtkAssist1, mtkRcs1, mtkCap1
    android.hardware.secure_element  override=true — NO version, NO interface
      NOTE: This is an explicit OMAPI disable. Comment in file confirms:
            "OMAPI is declared to be disabled"
      The main vendor manifest declares secure_element@1.0/SIM1 but this
      ODM override blanks it entirely at runtime.
      CONFLICT: lshal shows secure_element@1.0/SIM1 registered — the service
      binary is running despite the ODM override. Runtime state contradicts
      the manifest declaration. Requires further investigation.

# 2.4 VENDOR COMPATIBILITY MATRIX

  Location : /vendor/etc/vintf/compatibility_matrix.xml
  Type     : device (what vendor requires from framework)

  Required framework HALs (all optional=false):
    android.frameworks.schedulerservice   1.0
    android.frameworks.sensorservice      1.0
    android.hidl.allocator                1.0
    android.hidl.manager                  1.0
    android.hidl.memory                   1.0
    android.hidl.token                    1.0
    android.system.wifi.keystore          1.0
  vendor-ndk version : 29
  system-sdk version : 29

  NOTE: /system/etc/vintf/compatibility_matrix.xml does not exist.
        Standard for Android 10 — system matrix is not required device-side.

# 2.5 HAL IMPLEMENTATION BINARIES

  SERVICE BINARIES — /vendor/bin/hw/ (38 files):

  Binary                                                  Size (bytes)
  ──────────────────────────────────────────────────────────────────────
  android.hardware.audio@5.0-service-mediatek             15,472
  android.hardware.bluetooth@1.0-service-mediatek         11,512
  android.hardware.cas@1.1-service                        39,696
  android.hardware.configstore@1.1-service                21,848
  android.hardware.drm@1.0-service                        11,120
  android.hardware.drm@1.2-service.clearkey              152,336
  android.hardware.drm@1.2-service.widevine               24,604
  android.hardware.gatekeeper@1.0-service                 11,600
  android.hardware.gnss@2.0-service-mediatek              11,536
  android.hardware.graphics.allocator@2.0-service         11,584
  android.hardware.graphics.composer@2.1-service          11,672
  android.hardware.health@2.0-service                     70,264
  android.hardware.keymaster@4.0-service                  11,848
  android.hardware.light@2.0-service-mediatek             11,544
  android.hardware.media.omx@1.0-service                  15,620
  android.hardware.memtrack@1.0-service                   11,616
  android.hardware.secure_element@1.0-service-mediatek    74,544
  android.hardware.sensors@2.0-service-mediatek           73,176
  android.hardware.thermal@1.0-service                    11,632
  android.hardware.usb@1.1-service-mediatek               38,832
  android.hardware.vibrator@1.0-service                   11,608
  android.hardware.wifi@1.0-service-lazy-mediatek        427,424
  camerahalserver                                          15,880
  hostapd                                                 940,776
  mtkfusionrild                                            20,168
  power_legacy_test_v_1_0                                  11,800
  power_native_test_v_1_0                                  20,192
  vendor.mediatek.hardware.gpu@1.0-service                 29,976
  vendor.mediatek.hardware.mms@1.3-service                 37,448
  vendor.mediatek.hardware.mtkcodecservice@1.1-service     11,616
  vendor.mediatek.hardware.mtkpower@1.0-service            37,240
  vendor.mediatek.hardware.nvram@1.1-service               37,488
  vendor.mediatek.hardware.pq@2.2-service                  37,448
  vtservice_hidl                                           11,120
  wpa_supplicant                                        2,594,800

  NOTE: vendor.mediatek.hardware.nvram@1.1-service binary name is correct.
        The RC file that launches it has the ODM typo (sevice not service).
        Service binary name and RC filename do not match — documented in Section 1.

  NOTE: vendor.mediatek.hardware.pq@2.2-service is the service binary (2.2)
        but the impl library is vendor.mediatek.hardware.pq@2.3-impl.so.
        Service wrapper version and impl version differ — this is normal.

  IMPLEMENTATION LIBRARIES — /vendor/lib64/hw/ (52 files):

  Library                                                    Size (bytes)
  ──────────────────────────────────────────────────────────────────────
  android.hardware.audio.effect@5.0-impl.so                  302,176
  android.hardware.audio@5.0-impl-mediatek.so                236,056
  android.hardware.bluetooth.audio@2.0-impl-mediatek.so      101,208
  android.hardware.bluetooth@1.0-impl-mediatek.so             79,336
  android.hardware.camera.provider@2.4-impl-mediatek.so      153,200
  android.hardware.gatekeeper@1.0-impl.so                     20,600
  android.hardware.gnss@2.0-impl-mediatek.so                 196,792
  android.hardware.graphics.allocator@2.0-impl.so             24,904
  android.hardware.graphics.composer@2.1-impl.so             104,224
  android.hardware.graphics.mapper@2.0-impl-2.1.so            38,696
  android.hardware.light@2.0-impl-mediatek.so                 25,168
  android.hardware.memtrack@1.0-impl.so                       20,576
  android.hardware.power@1.3-impl-mediatek.so                 55,032
  android.hardware.thermal@1.0-impl.so                        29,024
  android.hardware.vibrator@1.0-impl.so                       20,648
  audio.bluetooth.default.so                                  92,864
  audio.primary.default.so                                    15,584
  audio.primary.mt6765.so                               2,042,304
  audio.r_submix.default.so                                   41,960
  audio.r_submix.mt6765.so                                    50,176
  audio.usb.default.so                                        28,416
  audio.usb.mt6765.so                                         62,968
  audio_policy.stub.so                                        15,424
  flp.default.so                                              36,352
  gatekeeper.default.so → libSoftGatekeeper.so (symlink)
  gps.default.so                                              75,192
  gralloc.default.so                                          23,824
  gralloc.mt6765.so                                          616,632
  hwcomposer.mt6765.so                                       682,224
  libSoftGatekeeper.so                                        32,536
  lights.mt6765.so                                            23,936
  local_time.default.so                                       15,128
  memtrack.mt6765.so                                          15,200
  power.default.so                                            15,120
  power.mt6765.so                                             15,128
  sensors.mt6765.so                                          261,136
  sound_trigger.primary.mt6765.so                             49,512
  thermal.mt6765.so                                           15,480
  vendor.mediatek.hardware.camera.bgservice@1.1-impl.so       46,992
  vendor.mediatek.hardware.camera.lomoeffect@1.0-impl.so      24,736
  vendor.mediatek.hardware.mms@1.3-impl.so                    46,688
  vendor.mediatek.hardware.mtkcodecservice@1.1-impl.so         87,560
  vendor.mediatek.hardware.mtkpower@1.0-impl.so               46,208
  vendor.mediatek.hardware.nvram@1.1-impl.so                  37,896
  vendor.mediatek.hardware.power@2.1-impl.so                  59,784
  vendor.mediatek.hardware.pq@2.3-impl.so                    177,208
  vendor.mediatek.hardware.videotelephony@1.0-impl.so          37,672
  vibrator.default.so                                         15,216
  vulkan.mt6765.so                                           683,024

  IMPLEMENTATION LIBRARIES — /vendor/lib/hw/ (47 files):
  NOTE: 32-bit counterparts of the 64-bit hw/ libraries above.
        Sizes differ from 64-bit versions as expected.
        Notable absent from 32-bit: android.hardware.camera.provider@2.4-impl-mediatek.so
        Camera provider is 64-bit only — confirmed by lshal (PID 671 loaded from lib64/hw/).

        Notable present in 32-bit only:
          android.hardware.drm@1.0-impl.so (74,452) — DRM impl is 32-bit only

# 2.6 GPU LIBRARIES

  Location : /vendor/lib64/egl/ and /vendor/lib/egl/

  64-bit (/vendor/lib64/egl/):
    libEGL_mtk.so          14,000   MTK EGL wrapper
    libGLES_meow.so     1,999,944   PowerVR GE8320 GLES implementation (codename MEOW)
    libGLESv1_CM_mtk.so   706,528   OpenGL ES 1.x CM wrapper
    libGLESv2_mtk.so    1,591,296   OpenGL ES 2.x wrapper
    libMEOW_trace.so      535,640   PowerVR GPU trace library

  32-bit (/vendor/lib/egl/):
    egl.cfg              76           EGL config file (32-bit only)
    libEGL_mtk.so        17,604
    libGLES_meow.so   1,252,340
    libGLESv1_CM_mtk.so  554,392
    libGLESv2_mtk.so   1,246,648
    libMEOW_trace.so     370,644

  NOTE: Codename MEOW confirmed in both library name and lshal.
        libPVROCL.so (34,741,696 bytes in lib64/) is the PowerVR OpenCL implementation.
        libOpenCL.so is the public-facing wrapper — only library in public.libraries.txt.

# 2.7 DRM LIBRARIES

  /vendor/lib64/mediadrm/:
    libdrmclearkeyplugin.so    64,016
    libmockdrmcryptoplugin.so  53,984
    libwvdrmengine.so       3,443,760   Widevine DRM engine

  /vendor/lib/mediadrm/:
    libdrmclearkeyplugin.so    47,212
    libmockdrmcryptoplugin.so  45,368
    libwvdrmengine.so       2,602,656

  /vendor/lib64/mediacas/:
    libclearkeycasplugin.so   100,968

  /vendor/lib/mediacas/:
    libclearkeycasplugin.so    89,664

# 2.8 AUDIO EFFECT LIBRARIES

  /vendor/lib64/soundfx/:
    libaudiopreprocessing.so   37,224
    libbundlewrapper.so        78,768
    libdownmix.so              23,624
    libdynproc.so              66,528
    libeffectproxy.so          19,544
    libldnhncr.so              19,640
    libreverbwrapper.so        36,848
    libvisualizer.so           19,536

  /vendor/lib/soundfx/:
    libaudiopreprocessing.so   24,672
    libbundlewrapper.so        71,660
    libdownmix.so              15,440
    libdynproc.so              46,560
    libeffectproxy.so          15,248
    libldnhncr.so              15,576
    libreverbwrapper.so        36,944
    libvisualizer.so           28,420

# 2.9 CAMERA SUPPORT LIBRARIES

  /vendor/lib64/mtkcam/:
    libmtkcam_streaminfo_plugin-p1stt.so   29,328

  Camera sensor tuning blobs present in vendor/lib64/:
    bf2553L_mipi_raw_IdxMgr.so     15,024   — sensor index manager (64-bit only)
    bf2553L_mipi_raw_tuning.so  2,367,144   — bf2553L ISP tuning data
    gc030a_mipi_raw_IdxMgr.so      15,024   — sensor index manager (64-bit only)
    gc030a_mipi_raw_tuning.so   2,391,712   — GalaxyCore GC030A ISP tuning data
    gc2385_mipi_raw_IdxMgr.so      15,024   — sensor index manager (64-bit only)
    gc2385_mipi_raw_tuning.so   2,453,160   — GalaxyCore GC2385 ISP tuning data

  Camera sensor tuning blobs present in vendor/lib/ (32-bit, no IdxMgr):
    bf2553L_mipi_raw_tuning.so  2,367,776
    gc030a_mipi_raw_tuning.so   2,388,244
    gc2385_mipi_raw_tuning.so   2,453,784

  Sony PDAF libraries present in vendor/lib64/ (BSP platform inclusions):
    libSonyIMX230PdafLibrary.so + Wrapper   — IMX230 (not this device)
    libSonyIMX338PdafLibrary.so + Wrapper   — IMX338 (not this device)
    libSonyIMX386PdafLibrary.so + Wrapper   — IMX386 (not this device)
    libSonyIMX519PdafLibrary.so + Wrapper   — IMX519 (not this device)
    NOTE: MT6765 platform BSP inclusions. Camera sensor hardware truth
          is bf2553L, gc030a, gc2385 — cross-reference Section 6.

# 2.10 KERNEL MODULES

  Location    : /vendor/lib/modules/
  Total       : 6 files (all loaded at runtime — confirmed via /proc/modules)
  All modules : out-of-tree (O flag)

  Module              File size    Loaded size   Dependents   Flags
  ──────────────────────────────────────────────────────────────────
  wmt_drv.ko          2,133,064    1,224,704     5            (O)
  wlan_drv_gen4m.ko   5,158,760    1,994,752     0            (O)
  wmt_chrdev_wifi.ko     57,720       28,672     1            (O)
  gps_drv.ko             72,192       65,536     0            (O)
  bt_drv.ko              52,216       28,672     1            (O)
  fpsgo.ko               34,608      811,008     0            (PO)
  ──────────────────────────────────────────────────────────────────
  NOTE: fpsgo.ko is marked (PO) — out-of-tree AND kernel tainted.
        wmt_drv.ko is the parent connectivity module.
        Dependency chain: wmt_drv → wmt_chrdev_wifi → wlan_drv_gen4m
                          wmt_drv → bt_drv
                          wmt_drv → gps_drv
        Load order enforced by init.wmt_drv.rc (boot), then
        vendor.connsys.driver.ready=yes triggers bt, gps, wlan insmod.

# 2.11 VNDK INVENTORY

  VNDK version : 29
  Locations:
    /system/lib64/vndk-29/     — 219 libraries (64-bit VNDK)
    /system/lib/vndk-29/       — 219 libraries (32-bit VNDK)
    /system/lib64/vndk-sp-29/  — 39 files + hw/ subdirectory (64-bit VNDK-SP)
    /system/lib/vndk-sp-29/    — 39 files + hw/ subdirectory (32-bit VNDK-SP)

  VNDK-SP-29/hw/ (both 32 and 64-bit):
    android.hidl.memory@1.0-impl.so
    NOTE: This is the ashmem memory allocator passthrough implementation.
          Loaded by gralloc (PID 526), pq service (PID 706),
          media.codec (PID 667) — confirmed via lshal.

  ODM-MODIFIED LIBRARIES IN VNDK-SP-29 (critical for crypto isolation):
    The following three libraries in vndk-sp-29 are AOSP stock versions.
    The ODM ships modified versions of these same libraries inside the
    crypto isolation bubble (/crypto_isolation/):
      libcrypto.so    — ODM modified OpenSSL 1.1.1w
      libhidlbase.so  — ODM modified HIDL base
      libutils.so     — ODM modified Android utilities
    The AOSP versions in vndk-sp-29 are NOT the ones used by
    keymaster/gatekeeper. See Section 3 for complete crypto isolation detail.

# 2.12 PUBLIC VENDOR LIBRARIES

  /vendor/etc/public.libraries.txt:
    libOpenCL.so
    NOTE: Only one vendor library is exposed to app processes.
          libOpenCL.so is the public wrapper for libPVROCL.so (PowerVR OpenCL).
          All other vendor libraries are VNDK-restricted.

# 2.13 LIVE HAL REGISTRATION STATE (from lshal — live device capture 2026-05-09)

  Legend: DM = declared in device manifest, FC = found in framework compat matrix,
          FM = declared in framework manifest, X = not in any manifest

  All HALs with DM,FC status are fully compliant — declared and matched.
  Passthrough HALs confirmed loaded (selected):
    audio@5.0 impl                → PID 516 (audioserver)
    bluetooth@1.0 impl            → PID 517
    camera.provider@2.4 impl      → PID 671 (camerahalserver)
    gatekeeper@1.0 impl           → PID 523
    graphics.mapper@2.1 impl      → loaded by PIDs 526 551 671 1079 1244 1328
                                    2106 and many others (system-wide)
    power@1.3 impl                → PID 540 (power-hal)
    vndk-sp-29 hidl.memory impl   → PIDs 526 706 (gralloc, pq)

  HALs registered but marked X (not in manifest — runtime only):
    vendor.mediatek.hardware.mms@1.3           PID 705
    vendor.mediatek.hardware.mtkcodecservice   PID 1393
    vendor.mediatek.hardware.mtkpower         PID 540
    vendor.mediatek.hardware.nvram@1.1        PID 541
    vendor.mediatek.hardware.power@2.1        PID 540
    vendor.mediatek.hardware.pq@2.3           PID 706
    vendor.mediatek.hardware.videotelephony   PID 537

  secure_element CONFLICT:
    Manifest (manifest_ss.xml) : override=true, no interface — OMAPI disabled
    lshal runtime              : secure_element@1.0/SIM1 registered (DM,FC Y)
    The mtk_secure_element_hal_service binary is running (PID 530).
    Runtime state contradicts ODM manifest override. Needs Ghidra analysis
    of the secure element service binary to determine what SIM1 actually is.

# 2.14 DEVICE FEATURE DECLARATIONS (from /vendor/etc/permissions/)

  34 files total — 31 XML feature declarations + 2 .txt lists

  HARDWARE FEATURES DECLARED:
    android.hardware.audio.low_latency
    android.hardware.audio.output          (via handheld_core_hardware.xml)
    android.hardware.bluetooth
    android.hardware.bluetooth_le
    android.hardware.camera
    android.hardware.camera.autofocus
    android.hardware.camera.flash
    android.hardware.camera.front
    android.hardware.camera.level.full
    android.hardware.camera.capability.manual_sensor
    android.hardware.camera.capability.manual_post_processing
    android.hardware.camera.capability.raw
    android.hardware.camera.any
    android.hardware.faketouch
    android.hardware.location
    android.hardware.location.network
    android.hardware.location.gps
    android.hardware.microphone
    android.hardware.opengles.aep
    android.hardware.sensor.accelerometer
    android.hardware.sensor.compass        (via handheld_core_hardware.xml)
    android.hardware.sensor.stepcounter
    android.hardware.sensor.stepdetector
    android.hardware.telephony
    android.hardware.telephony.gsm
    android.hardware.telephony.ims
    android.hardware.touchscreen
    android.hardware.touchscreen.multitouch
    android.hardware.touchscreen.multitouch.distinct
    android.hardware.touchscreen.multitouch.jazzhand
    android.hardware.usb.accessory
    android.hardware.usb.host
    android.hardware.vulkan.compute        version=0
    android.hardware.vulkan.level          version=0
    android.hardware.vulkan.version        version=4198400 (Vulkan 1.1)
    android.hardware.wifi
    android.hardware.wifi.direct
    android.hardware.wifi.passpoint

  SOFTWARE FEATURES DECLARED:
    android.software.activities_on_secondary_displays
    android.software.app_widgets
    android.software.autofill
    android.software.backup
    android.software.cant_save_state
    android.software.companion_device_setup
    android.software.connectionservice
    android.software.cts
    android.software.device_admin
    android.software.home_screen
    android.software.input_methods
    android.software.ipsec_tunnels
    android.software.managed_users
    android.software.midi
    android.software.picture_in_picture
    android.software.print
    android.software.secure_lock_screen
    android.software.verified_boot
    android.software.voice_recognizers

  NOTE: android.hardware.sensor.compass declared in handheld_core_hardware.xml
        but no magnetometer exists on this device. BSP template not cleaned up.
        Will fail CTS compass sensor tests.

  NOTE: android.software.verified_boot declared but AVB is disabled and boot
        state is orange. This declaration was present in stock firmware before
        bootloader unlock. It is an ODM compliance lie — verified boot is not
        active. The BROM dump confirms null vbmeta flags (0x00000003).

  REMOVABLE SYSTEM APPS (/vendor/etc/permissions/pms_sysapp_removable_system_list.txt):
    com.android.quicksearchbox
    com.android.calendar
    com.android.dreams.basic
    com.android.musicfx
    com.android.calculator2
    com.android.email
    com.android.exchange

  REMOVABLE VENDOR APPS (/vendor/etc/permissions/pms_sysapp_removable_vendor_list.txt):
    com.android.vending
    com.google.android.apps.maps
    NOTE: Play Store and Maps are explicitly user-removable at vendor level.
          Indicates ODM GMS agreement allows removal of these two specifically.

# 2.15 SHIM REQUIREMENTS

  Zero shim binaries required.

  NVRAM chain RPATH correction required (not shims):
    libhwm.so + libnvram.so + libfile_op.so — patchelf RPATH correction needed.
    These libraries have hardcoded library search paths that must be corrected
    for ROM build. See Section 15 for build specification details.
################################################################################
# SECTION 3 — CRYPTO & SECURITY STACK
################################################################################

# 3.1 KEYMASTER HAL — LIVE RUNTIME CHAIN (OS)

  Service binary  : /vendor/bin/hw/android.hardware.keymaster@4.0-service
  Binary size     : 11,848 bytes
  HIDL interface  : android.hardware.keymaster@4.0::IKeymasterDevice/default
  HAL transport   : hwbinder
  Class           : early_hal (starts before vold)
  PID at capture  : 413
  Security level  : SOFTWARE — libpuresoftkeymasterdevice.so confirmed
                    (iTrustee TEE present and running but keymaster never
                     calls into it — confirmed by maps: zero TEE client libs)

  COMPLETE LIBRARY MAP (from /proc/413/maps — live device 2026-05-09):

  Source                                    Library
  ──────────────────────────────────────────────────────────────────────────
  /vendor/bin/hw/                           android.hardware.keymaster@4.0-service
  /apex/com.android.runtime/lib64/bionic/  libm.so
  /system/lib64/                            libbase.so
  /system/lib64/                            libdl_android.so
  /system/lib64/                            libc++.so
  /system/lib64/                            libnetd_client.so
  /system/lib64/vndk-sp-29/                 libc++.so
  /system/lib64/vndk-sp-29/                 libbinderthreadstate.so
  /system/lib64/vndk-sp-29/                 libcutils.so
  /system/lib64/vndk-29/                    android.hardware.keymaster@3.0.so
  /system/lib64/vndk-29/                    libcrypto.so          ← AOSP VNDK-29 version
  /system/lib64/vndk-29/                    libpuresoftkeymasterdevice.so
  /system/lib64/vndk-29/                    libkeymaster_messages.so
  /vendor/lib64/                            libkeymaster4.so
  /vendor/lib64/                            libkeymaster4support.so

  CRITICAL NOTE — NO CRYPTO ISOLATION BUBBLE IN OS:
    In the full Android OS, VNDK-29 provides natural linker namespace separation.
    The keymaster HAL loads libcrypto.so from /system/lib64/vndk-29/ — the
    AOSP version — through the standard VNDK linker namespace. The ODM-modified
    library conflict does not occur in the OS because the Android linker namespace
    architecture automatically isolates vendor from system.

    The crypto isolation bubble was TWRP-specific only — required because TWRP
    has no VNDK-29, leaving the ODM blobs and TWRP bionic in the same default
    namespace where they collide fatally (SIGSEGV before keymaster code executes).

    ROM build implication: NO custom isolation bubble needed. Ship vendor blobs
    as-is, let VNDK-29 handle namespace resolution naturally.

  NOTE: android.hardware.keymaster@3.0.so loads into the keymaster@4.0
        process. This is standard MTK Android 10 — @4.0.so has DT_NEEDED
        dependency on @3.0.so. The @3.0 service is NOT registered in
        hwservicemanager at runtime on the OS (unlike TWRP where the
        Keymaster3 getService() blocking loop required a manifest workaround).

# 3.2 KEYSTORE DAEMON — LIVE RUNTIME CHAIN (OS)

  Binary     : /system/bin/keystore
  Size       : 477,632 bytes
  PID        : 642 (confirmed via maps)
  User       : keystore
  Purpose    : Android Keystore system service — manages app key material
               Separate from keymaster HAL — keystore calls keymaster for
               hardware-backed operations

  Library sources confirmed from /proc/642/maps:
    /system/bin/keystore                     (main binary)
    /apex/com.android.runtime/lib64/bionic/ libc.so
    /system/lib64/                           libbase.so
    /system/lib64/                           libbinder.so
    /system/lib64/                           libbinderthreadstate.so
    /system/lib64/                           libcgrouprc.so
    /system/lib64/                           libc++.so
    /system/lib64/                           libdl_android.so
    /system/lib64/                           libhidlbase.so
    /system/lib64/                           libhwbinder.so
    /system/lib64/                           libkeymaster4support.so
    /system/lib64/                           libkeymaster_messages.so
    /system/lib64/                           libkeystore_aidl.so
    /system/lib64/                           libkeystore_parcelables.so
    /system/lib64/                           libnetd_client.so
    /system/lib64/                           libpackagelistparser.so
    /system/lib64/                           libprocessgroup.so
    /system/lib64/                           libprotobuf-cpp-lite.so
    /system/lib64/                           libselinux.so
    /system/lib64/                           libservices.so
    /system/lib64/                           libsoftkeymasterdevice.so
    /system/lib64/                           libutils.so
    /system/lib64/                           libvndksupport.so
    /system/lib64/                           libwifikeystorehal.so
    /system/lib64/                           android.hardware.keymaster@3.0.so
    /system/lib64/                           android.hardware.keymaster@4.0.so
    /system/lib64/                           android.system.wifi.keystore@1.0.so
    /system/lib64/                           libgatekeeper.so

  NOTE: keystore daemon loads libsoftkeymasterdevice.so from /system/lib64/ —
        the AOSP version. This is the fallback software keymaster used by
        keystore itself, separate from the ODM keymaster HAL chain.

# 3.3 GATEKEEPERD — LIVE RUNTIME CHAIN (OS)

  Binary     : /system/bin/gatekeeperd
  Size       : 45,336 bytes
  PID        : 713 (confirmed via maps)
  User       : system
  Purpose    : Manages credential verification (PIN/pattern/password).
               On this device lockscreen is DISABLED — gatekeeperd runs
               but no credential handle is enrolled.

  Library sources confirmed from /proc/713/maps:
    /system/bin/gatekeeperd
    /apex/com.android.runtime/lib64/bionic/ libc.so
    /system/lib64/                           libbase.so
    /system/lib64/                           libbinderthreadstate.so
    /system/lib64/                           libcgrouprc.so  (truncated — full map not returned)
    /system/lib64/                           libdl_android.so
    /system/lib64/                           libgatekeeper.so
    /system/lib64/                           libgsi.so
    /system/lib64/                           libhardware.so
    /system/lib64/                           libhidlbase.so
    /system/lib64/                           libhwbinder.so
    /system/lib64/                           libkeystore_aidl.so
    /system/lib64/                           libnetd_client.so
    /system/lib64/                           libprocessgroup.so
    /system/lib64/                           libselinux.so
    /system/lib64/                           libvndksupport.so

  NOTE: gatekeeperd loads entirely from /system/lib64/ — no vendor libs.
        No ODM-modified library involvement in the gatekeeper chain.

# 3.4 CREDSTORE

  Binary     : /system/bin/credstore — DOES NOT EXIST on this device
  Runtime    : credstore not running — confirmed via pidof + maps attempt
  Impact     : credstore is optional in Android 10. Its absence means no
               Credential Manager API support. No impact on FBE decryption
               or keymaster operation.

# 3.5 ODM-MODIFIED LIBRARIES

  The ODM (Topwise/Wiite) recompiled keymaster/gatekeeper stack against
  their own modified versions of three standard Android libraries.
  Symbol tables confirmed to differ from AOSP via binary analysis.

  Confirmed ODM-modified:
    libcrypto.so   — ODM-modified OpenSSL 1.1.1w
    libhidlbase.so — ODM-modified HIDL base
    libutils.so    — ODM-modified Android utilities

  Discovery method:
    1. Hardcoded dlopen() calls found inside ODM keymaster/gatekeeper blobs
       via binary analysis. Blobs had hardcoded absolute library paths
       pointing to their modified versions.
    2. Standard Android linker found incompatible symbol tables — SIGSEGV
       before keymaster code executes (observed during TWRP build session).
    3. LD_DEBUG tracing during TWRP build confirmed exact library paths
       and symbol resolution failures.

  OS resolution: VNDK-29 linker namespace separation handles this naturally.
  TWRP resolution: Crypto isolation bubble (see 3.6).
  ROM build: No special handling required — VNDK-29 handles it.

  NVRAM chain (separate issue — not crypto isolation):
    libhwm.so + libnvram.so + libfile_op.so require patchelf RPATH correction
    for ROM build. These are not in the keymaster chain — they are in the
    NVRAM/calibration blob chain. See Section 15.

# 3.6 CRYPTO ISOLATION BUBBLE — TWRP IMPLEMENTATION

  NOTE: TWRP-specific only. NOT required for ROM build.
        Required because TWRP has no VNDK-29 linker namespace.

  Problem:
    ODM keymaster/gatekeeper compiled against modified libcrypto,
    libhidlbase, libutils. Without VNDK-29, these blobs and TWRP bionic
    share the default linker namespace. Incompatible symbol tables cause
    SIGSEGV before keymaster code executes.

  Architecture:
    Containment zone : /crypto_isolation/ in TWRP ramdisk
    All ODM blobs    : patchelf --set-rpath /crypto_isolation applied
    VFS bind mount   : mount none /crypto_isolation/hw /vendor/lib64/hw bind
    Bionic           : NOT in bubble — libc/libm/libdl use TWRP bionic directly
    ODM chain        : handles all crypto logic inside the bubble
    Neither side     : aware of the other — complete namespace isolation

  Critical init order (TWRP ramdisk init.recovery.mt6762.rc):
    on fs:
      1. mkdir /crypto_isolation
      2. mount --bind /crypto_isolation/hw /vendor/lib64/hw
      3. start vold  ← MUST come after bind mount

  VFS bind mount rationale:
    gatekeeper uses hw_get_module() which searches /vendor/lib64/hw/.
    Android 11 linker rejects symlinks crossing namespace boundaries.
    Bind mount projects bubble as real vendor directory — linker permits load.

  Keymaster@3.0 blocking loop fix (TWRP-specific):
    Problem: enumerateAvailableDevices() calls getService() for @3.0 which
    blocks indefinitely in a 1-second retry loop — TWRP splash screen hang.
    Root cause: @4.0.so has DT_NEEDED on @3.0.so. @3.0 library loads but
    registerAsService() never called. hwservicemanager retries forever.
    Fix: Declare keymaster@3.0 passthrough in device manifest.
         hwservicemanager attempts dlopen() — returns null in milliseconds.
         No blocking wait. No source modification required.
    Known cosmetic log: "transport == passthrough and arch == is not valid"
    Adding arch="32+64" causes fatal CHECK() crash — warning is correct state.

  Result (verified 2026-05-05):
    twrp.decrypt.done=true
    twrp.all.users.decrypted=true
    /data/system/ fully readable

# 3.7 FBE IMPLEMENTATION

  Type             : File-Based Encryption (FBE)
  Policy version   : v1 — 8-byte key descriptor (SHA512 double hash)
                     NOT v2 (v2 uses 16-byte identifier, kernel 4.14+)
  Contents mode    : aes-256-xts (from stock fstab fileencryption= flag)
  Filenames mode   : aes-256-cts (from ro.crypto.volume.filenames_mode)
  ro.crypto.type   : file (confirmed live)
  ro.crypto.state  : encrypted (confirmed live)

  Kernel FBE path (kernel 4.9.190+):
    FS_IOC_ADD_ENCRYPTION_KEY introduced in kernel 4.14.
    On kernel 4.9 this ioctl returns ENOTTY.
    fscrypt layer detects ENOTTY and falls back to legacy keyctl/add_key.
    Legacy path requires "fscrypt" named keyring in session keyring.
    install_keyring in init.rc creates this keyring before fscrypt init.

  TWRP build flags required:
    TW_INCLUDE_CRYPTO     := true
    TW_INCLUDE_CRYPTO_FBE := true
    TW_USE_FSCRYPT_POLICY := 1    ← forces v1 policy code path
    BOARD_USES_METADATA_PARTITION := true

  Key material location (live device confirmed):
    /data/unencrypted/key/          ← subdirectory, NOT files directly in /data/unencrypted/
    Contents: encrypted_key, keymaster_key_blob, secdiscardable, stretching, version

  /data/unencrypted/ full listing (confirmed live):
    key/          (4096 bytes, root:root, 2026-05-04) ← FBE key material
    magisk/       (4096 bytes, root:root, 2026-05-04) ← Magisk artifact
    mode          (23 bytes)
    per_boot_ref  (8 bytes)
    ref           (8 bytes)

  Vold key management:
    Path confirmed: /data/misc/vold/user_keys/
    NOTE: /data/vold/user_keys/ does NOT exist — previous document had
          wrong path. Correct path is /data/misc/vold/user_keys/.
    DE key : /data/misc/vold/user_keys/de/0/
    CE key : /data/misc/vold/user_keys/ce/0/current/
    Contents: encrypted_key, keymaster_key_blob, secdiscardable,
              stretching, version

  Fstab fscrypt declaration:
    /dev/block/platform/bootdevice/by-name/userdata /data ext4
      fileencryption=aes-256-xts (content mode only in fstab)
    ro.crypto.volume.filenames_mode=aes-256-cts set via product makefile
    NOTE: fileencryption=aes-256-xts only in fstab (no suffix).
          Filename mode declared separately. Required for libfs_mgr parser
          compatibility — 5-column AOSP format throughout.

  Lockscreen state:
    DISABLED — locksettings.db: lockscreen.disabled=1
    /data/system/gatekeeper/ does NOT exist — no credential handle enrolled
    FBE CE key wrapped with empty credential at format time
    TWRP automatic decrypt uses empty credential — no password prompt

# 3.8 METADATA PARTITION — CRITICAL

  TWO partitions exist. Only one contains key material.

  REAL metadata (FBE key storage):
    Name            : md_udc
    Block device    : mmcblk0p10
    By-name symlink : /dev/block/by-name/md_udc → mmcblk0p10
    Mount point     : /metadata
    Filesystem      : ext4
    Fstab entry     : /dev/block/platform/bootdevice/by-name/md_udc /metadata ext4
                      noatime,nosuid,nodev,discard — wait,formattable,first_stage_mount
    Content         : FBE key material (encrypted_key, keymaster_key_blob)
    Status          : LIVE — actively used

  GHOST partition (empty decoy):
    Name            : metadata
    Block device    : mmcblk0p11
    By-name symlink : /dev/block/by-name/metadata → mmcblk0p11
    Content         : ALL ZEROS — confirmed by dd byte scan
    Status          : DEAD — never written, never mounted by stock OS
    Origin          : Generic Android 10 partition template artifact

  TRAP for ROM build:
    Standard Android 10 expects /dev/block/by-name/metadata → /metadata.
    On this device that partition is empty. Using it causes fscrypt to find
    no key material — decryption fails with no diagnostic output.
    ALWAYS use md_udc as the /metadata mount target on this device.

  TWRP recovery.fstab entry (confirmed working):
    /dev/block/platform/bootdevice/by-name/md_udc /metadata ext4
      noatime,nosuid,nodev,discard wait,formattable

# 3.9 AVB & SIGNING

  AVB status         : DISABLED
  vbmeta flags       : 0x00000003 (null vbmeta — confirmed raw hex from BROM dump)
  Verified boot      : orange state
  Bootloader         : UNLOCKED
  Signing            : test-keys (ODM cost-reduction — not release-keys)
  OTA signing cert   : AOSP default testkey (SHA1 RSA2048, valid through 2035)
  SELinux            : PERMISSIVE

  AVB chain (from stock vbmeta partition — BROM dump, not live device):
    Algorithm              : SHA256_RSA2048
    avbtool version        : 1.1.0
    Rollback index         : 0
    Flags                  : 3 (disabled)
    Chain: recovery, vbmeta_system, vbmeta_vendor, boot, dtbo
    dtbo hash              : sha256
                             salt:   f0b811c76a5dd838af23cc5fcd69405d5ec71be8ac01354723f02895b8c8edec
                             digest: 34df441cacdc9d852b72cbe166abe44f22c0eae684c2f64cd3d85f998ffb500f

  NOTE: AVB was disabled by device owner post-purchase during bootloader
        unlock and Magisk root procedure. The android.software.verified_boot
        permission declaration in vendor/etc/permissions/ was present in
        stock firmware — it is an ODM compliance lie predating the owner's
        modifications.

# 3.10 KEYSTORE USER DATA (OS — LIVE DEVICE)

  Path : /data/misc/keystore/user_0/

  Identity keys:
    10046_USRPKEY_wear_device_access_token_key  — Wiite wear device auth token
    10046_USRCERT_wear_device_access_token_key  — matching certificate
    10046_USRCERT_c5635988b7157a3a              — device certificate
    10046_USRPKEY_c5635988b7157a3a              — matching private key
    10077_USRPKEY_USDID                         — hardware device ID handle
    10077_USRCERT_USDID                         — matching certificate
    1010_USRPKEY_MacRandSecret                  — WiFi MAC randomization seed

  Security master keys:
    10067_USRPKEY__androidx_security_master_key_  — AndroidX crypto master key
    10046_USRPKEY__androidx_security_master_key_  — AndroidX crypto master key (app UID 10046)
    10046_USRPKEY_mobstore_encrypt                — mobile store encryption key

  Subscription / auth:
    10046_USRPKEY_OGI+]TS43+]SubscriptionEncryptionKey
    10046_USRPKEY_snet_test_keys                  — SafetyNet verification
    10046_USRCERT_snet_test_keys                  — matching certificate
    10044_USRPKEY_gmphn_pkey                      — Google phone key

  NOTE: All keystore entries are app-layer keys stored by the OS.
        None of these are in the FBE key derivation chain.
        FBE keys are in /data/misc/vold/ and /data/unencrypted/key/ only.

# 3.11 HARDWARE SECURITY COMPONENTS — NOT IN FBE PATH

  iTrustee TEE:
    Present — tee1/tee2 partitions loaded, PSCI active
    Running — confirmed from ATF/PSCI data in hardware database
    NOT used by keymaster — zero TEE client libs in PID 413 maps
    Function: TEE OS execution environment (runs but keymaster ignores it)

  RPMB (Replay Protected Memory Block):
    Present — mmcblk0rpmb, 16MB
    Function: Rollback counter storage ONLY
    RPMB key in live memory: a0920d822eb816ed9c4fcc3be1bf17e73852bf3197797ca29c621b619c471c6d
    NOT in FBE key derivation chain

  Wiite kernel drivers (wiite_con_ctrl, wiite_corp_ctrl):
    Sensor hub interface drivers ONLY
    Zero crypto involvement — confirmed by dmesg analysis
    All activity: GPIO toggling, nRF52832 interrupt polling, sensor events
    IRQ 27 fires every ~22 seconds — sensor data, not crypto operations
    The naming convention is internal Wiite — no security function implied

  FDE key present in live memory — legacy artifact, unused:
    c55728f39fbd80892be02a01a3d2c40c
    This device uses FBE exclusively. FDE key is a BSP initialization
    artifact from the bootloader. Has no operational role.

# 3.12 FSTAB DUAL-PARSER ISSUE (TWRP — RESOLVED)

  Problem (TWRP-specific):
    TWRP 4-column flags= format causes libfs_mgr (used by TWRP 11 fscrypt
    layer) to execute goto err on first flags= line, aborting entire fstab
    parse. fscrypt receives empty fstab. fscrypt_initialize_systemwide_keys
    fails three times consecutively.

  Solution:
    Pure AOSP 5-column format throughout entire recovery.fstab.
    fileencryption=aes-256-xts only (no filename mode suffix in fstab).
    Filename mode set via ro.crypto.volume.filenames_mode in product makefile.

  ReadDefaultFstab() path resolution:
    /system/bin/recovery present on this device → primary path: /etc/recovery.fstab
    Additional deployment to all libfs_mgr search locations required.

# 3.13 SECURITY STATE SUMMARY

  Property                          Value                   Source
  ──────────────────────────────────────────────────────────────────────
  ro.crypto.type                    file                    live device
  ro.crypto.state                   encrypted               live device
  ro.crypto.volume.filenames_mode   aes-256-cts             live device
  ro.crypto.volume.options          not set (OS normal)     live device
  ro.crypto.fs_crypto_blkdev        not set (FBE, not FDE)  live device
  vold.decrypt                      not set (FBE, not FDE)  live device
  vold.post_fs_data_done            1                       live device
  ro.boot.verifiedbootstate         orange                  live device
  ro.boot.flash.locked              0                       live device
  AVB flags                         0x00000003 (disabled)   BROM hex dump
  SELinux                           permissive              live device
  Bootloader                        unlocked                live device
  Magisk                            active                  live device
################################################################################
# SECTION 4 — SENSOR STACK
################################################################################

# 4.1 ARCHITECTURE OVERVIEW

  This device runs two parallel independent sensor stacks operating
  simultaneously. They do not share code, data paths, or failure modes.
  A failure in one stack does not affect the other.

  STACK A — Platform Sensor HAL (Android sensor framework path):
    Service       : android.hardware.sensors@2.0-service-mediatek (PID 669)
    HAL type      : HAL 1.0 passthrough wrapped by HIDL 2.0 service
    Primary impl  : /vendor/lib64/hw/sensors.mt6765.so
    Transport     : wiite_corp_ctrl sysfs → WiiteEventSensor class
    Exposes to OS : 10 sensors via SensorManager API
    Data source   : nRF52832 co-processor via sysfs attributes

  STACK B — Direct MEMS Sensor Access (ODM health application path):
    Library       : /vendor/lib64/libsensor_custom.so (15,024 bytes)
    Role          : Direct hardware access to Pixart PAR2822 optical sensor
    Transport     : Direct I2C ioctl to /dev/i2c-4, address 0x33
    Independence  : Completely bypasses Android sensor framework
    Failure mode  : If no driver bound to 4-0033, ioctl fails — full stack down

  ROM BUILD CRITICAL:
    Stack A works on GSI because it communicates via sysfs and the wiite_corp
    driver IS in the ODM kernel (and must be in the ROM kernel).
    Stack B fails on GSI because the mediatek,heartrate driver is NOT in the
    GSI kernel — the PAR2822 device node exists but sits unbound.
    Both drivers must be present in the ROM kernel for full sensor operation.

# 4.2 ANDROID SENSOR FRAMEWORK — REGISTERED SENSORS

  Source: dumpsys sensorservice (live device 2026-05-10 04:08:12)
  Total hardware sensors : 10
  Total running          : 10
  All vendor string      : MTK

  Handle     Name                   Type (Android)        Mode           Rates              FIFO (max/rsvd)  Wake
  ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
  0x00000001 ACCELEROMETER          accelerometer (1)      continuous     50-200Hz           4500/3000 events  no
  0x00000005 LIGHT                  light (5)              on-change      1Hz min            no batching       no
  0x00000006 PRESSURE               pressure (6)           continuous     1-10Hz             4500/300 events   no
  0x0000000d AMBIENT_TEMPERATURE    ambient_temperature(13) on-change     1Hz min            no batching       no
  0x00000012 STEP_DETECTOR          step_detector (18)     special-trigger                   4500/100 events   no
  0x00000013 STEP_COUNTER           step_counter (19)      on-change      1Hz min            no batching       no
  0x00000015 HEART_RATE             heart_rate (21)        on-change      1Hz min            no batching       no
  0x00000016 TILT_DETECTOR          tilt_detector (22)     special-trigger                   no batching       YES (wakeUp)
  0x0000001f HEART_BEAT             heart_beat (31)        special-trigger                   no batching       no
  0x00000065 STEP_DETECTOR_WAKEUP   step_detector (18)     special-trigger                   4500/100 events   YES (wakeUp)

  Fusion states: all disabled (no gyroscope — 9-axis, game, geomag all off)

  Active sensor connections at capture time:
    com.google.android.location.os.real.RealWakeUpTiltDetector → TILT_DETECTOR
    com.wiite.devicedaemon.CommunicationServices                → LIGHT
    com.google.android.location.collectionlib.BatchSignalCollector → ACCELEROMETER
    com.wiite.wearhealthuart.service.PedometerService           → STEP_COUNTER

  NOTE: HEART_RATE (0x00000015) and HEART_BEAT (0x0000001f) appear in the
        sensor list but were NOT active at capture time. They register
        successfully because the nRF52832 exposes PAR2822 data via sysfs.
        These sensors depend on Stack B (libsensor_custom.so) for raw data
        and Stack A for Android framework delivery.

# 4.3 SENSOR HAL SERVICE — RUNTIME LIBRARY MAP

  NOTE: The /proc/maps command for PID 669 was not successfully captured —
        Gemini returned GNSS service maps instead (command expansion failure).
        Library chain is documented from hardware database binary analysis.
        Confirmed libraries from sensors.mt6765.so readelf DT_NEEDED:

  sensors.mt6765.so dependencies:
    liblog.so, libcutils.so, libutils.so, libhardware.so,
    libstagefright_foundation.so, libksensor.so, libc++.so, libc.so,
    libm.so, libdl.so

  libksensor.so dependencies:
    liblog.so, libc++.so, libdl.so, libc.so, libm.so
    Role: Pure calibration math — no ioctl, no device node access.
          Does not read from hardware directly.

  libsensor_custom.so dependencies:
    Direct I2C access — interfaces with /dev/i2c-4 at address 0x33
    No dependency on sensors.mt6765.so or libksensor.so
    Role: PAR2822 optical sensor raw data acquisition for heart rate and SpO2

# 4.4 I2C BUS INVENTORY (from /sys/bus/i2c/devices/ — live device)

  I2C controllers on this platform (MT6762V/WB):
    i2c-0  : 11007000.i2c0
    i2c-1  : 11008000.i2c1 (no devices enumerated)
    i2c-2  : 11009000.i2c2
    i2c-3  : 1100f000.i2c3 (no devices enumerated)
    i2c-4  : 11011000.i2c4  ← PRIMARY SENSOR BUS
    i2c-5  : 11016000.i2c5
    i2c-6  : 1100d000.i2c6

  COMPLETE DEVICE ENUMERATION:

  Bus    Address  DTS Name       Compatible          Driver Bound    Notes
  ──────────────────────────────────────────────────────────────────────────────────
  0-005d hyn_ts   (touch)        Hynitron touch      hyn_ts          ← touchscreen
  2-000c unknown  (unknown)      (unknown)           (unknown)
  2-0010 unknown  (unknown)      (unknown)           (unknown)
  2-001a unknown  (unknown)      (unknown)            (unknown)
  2-002d unknown  (unknown)      (unknown)            (unknown)
  2-0050 unknown  (unknown)      (unknown)            (unknown)
  2-0052 unknown  (unknown)      (unknown)            (unknown)
  4-0011 unknown  (unknown)      (unknown)            (unknown)
  4-0033 heartrate mediatek,heartrate NONE            ← PAR2822 — NO DRIVER BOUND
  4-0050 wiite_cp wiite,cp       wiite_corp          ← nRF52832 DRIVER BOUND
  5-0034 unknown  (unknown)      (unknown)            (unknown)
  6-000c unknown  (unknown)      (unknown)            (unknown)
  6-002d unknown  (unknown)      (unknown)            (unknown)
  6-0050 unknown  (unknown)      (unknown)            (unknown)

  4-0033 UEVENT (complete):
    OF_NAME=heartrate
    OF_FULLNAME=/i2c4@11011000/heartrate@33
    OF_COMPATIBLE_0=mediatek,heartrate
    OF_COMPATIBLE_N=1
    MODALIAS=i2c:heartrate
    NOTE: NO DRIVER FIELD — nothing bound. Device tree node exists,
          kernel sees the device, but no kernel driver claims it.
          This is the PAR2822 Pixart optical sensor.

  4-0050 UEVENT (complete):
    DRIVER=wiite_corp
    OF_NAME=wiite_cp
    OF_FULLNAME=/i2c4@11011000/wiite_cp@50
    OF_COMPATIBLE_0=wiite,cp
    OF_COMPATIBLE_N=1
    MODALIAS=i2c:cp
    NOTE: DRIVER=wiite_corp confirmed bound. nRF52832 co-processor.

# 4.5 GSI FAILURE ROOT CAUSE — CONFIRMED

  On GSI all sensors EXCEPT PAR2822 work because:
    - wiite_corp driver is in the ODM kernel → bound to 4-0050 → nRF works
    - sensors.mt6765.so reads wiite_corp_ctrl sysfs → Stack A functions

  Pixart PAR2822 fails on GSI because:
    - mediatek,heartrate driver NOT in GSI kernel
    - 4-0033 device node exists in DTS but sits unbound (no DRIVER field)
    - libsensor_custom.so opens /dev/i2c-4 and tries to communicate with 0x33
    - No driver → ioctl returns error → Stack B completely non-functional
    - Heart rate, SpO2, HEART_BEAT sensor fail

  ROM build requirement to fix this:
    The ROM kernel MUST include the mediatek,heartrate driver compiled in.
    Compatible string: mediatek,heartrate
    Device tree node: /i2c4@11011000/heartrate@33 (I2C4 address 0x33)
    Without this driver the PAR2822 fails identically to GSI.

  The wiite_corp driver MUST also be present (already confirmed in ODM kernel).
  Compatible string: wiite,cp
  Device tree node: /i2c4@11011000/wiite_cp@50 (I2C4 address 0x50)

# 4.6 nRF52832 CO-PROCESSOR — COMPLETE IDENTIFICATION

  SILICON IDENTIFICATION:
    Chip ID           : 0x70 (from /sys/class/misc/wiite_corp_ctrl/chip_id)
    Silicon           : Nordic Semiconductor nRF52832
    Firmware version  : 48059 (from /sys/class/misc/wiite_corp_ctrl/version)
    Operating mode    : 0 = normal (from /sys/class/misc/wiite_corp_ctrl/nrf_sys_mode)
    BT MAC            : ca:77:87:c7:16:00 (from wiite_corp_ctrl/btMAC)
    RTC (live)        : 2026-5-9 23:50:0 (from wiite_corp_ctrl/nrf_rtc)
    Long-term storage : NO SUPPORT

  SPECIFICATIONS:
    CPU               : ARM Cortex-M4F (hardware floating point)
    Flash             : 512KB internal — firmware lives HERE, not on eMMC
    RAM               : 64KB
    Bluetooth         : 5.0 BLE
    I2C               : Multiple instances (PAR2822 and other sensors on nRF side)
    UART              : ttyS1 — connects to MT6762 at 115200 8N1, chmod 0666

  KERNEL INTERFACE:
    I2C node          : 4-0050 (I2C4, address 0x50)
    DTS compatible    : wiite,cp
    Kernel driver     : wiite_corp (BOUND — confirmed)
    sysfs interface   : /sys/class/misc/wiite_corp_ctrl/ (34 attributes)
    Platform device   : /sys/devices/platform/wiite_corp
    IRQ               : 27 (irq/27-WIITE_CO kernel thread)
                        fires every ~22 seconds — sensor event interrupt
    IRQ 25            : irq/25-WIITE_CO — secondary interrupt handler

  POWER:
    Regulator         : vmch/wiite_corp-vhrs
    Path              : /sys/kernel/debug/regulator/vmch/wiite_corp-vhrs
                        /sys/devices/platform/mt-pmic/regulator/regulator.27/wiite_corp-vhrs
    NOTE: nRF52832 has dedicated voltage rail from PMIC regulator.27

  wiite_con device (secondary):
    Platform device   : /sys/devices/platform/wiite_con
    DTS symbols       : wiite_con, with wiite,con1 and wiite,con2 sub-nodes
    sysfs             : /sys/class/misc/wiite_con_ctrl
    Role              : Secondary control interface — exact function TBD

  ROM IMPACT:
    nRF52832 firmware lives in internal 512KB flash — completely unaffected
    by any eMMC flash operation, TWRP, or AOSP ROM installation.
    The nRF operates independently regardless of Android state.

# 4.7 wiite_corp_ctrl SYSFS INTERFACE

  Path    : /sys/class/misc/wiite_corp_ctrl/
  Init rc : Attributes permissioned in init.mt6762.rc on boot trigger
            chown/chmod 0666 applied to all listed attributes

  CONFIRMED ATTRIBUTES (from live device):

  Attribute          Access   Confirmed Value      Description
  ────────────────────────────────────────────────────────────────────────
  version            r--      48059                nRF firmware version
  chip_id            r--      0x70                 Silicon chip ID
  nrf_rtc            rw       2026-5-9 23:50:0     Real-time clock
  btMAC              rw       ca:77:87:c7:16:00    Bluetooth MAC address
  longterm           rw       NO SUPPORT           Long-term data storage
  nrf_sys_mode       rw       0                    Operating mode (0=normal)
  config             rw       0-0-0-0-0-0-0-0-0-0-0-0-0-0-1-1-0-0-0-0-0-0-0-1-0-0-0-0-0-1-0-0
                                                   32-bit sensor enable bitmap
  rst_op             rw       (write-only reset)   nRF reset operation
  upgrade_op         rw       (write-only OTA)     Firmware upgrade trigger
  command            rw       (command channel)    Raw command interface
  uart_op            rw       (uart config)        UART operation control

  CONFIG BITMAP (32 bits, current value):
    Bit 14 = 1 : sensor enabled
    Bit 15 = 1 : sensor enabled
    Bit 23 = 1 : sensor enabled
    Bit 29 = 1 : sensor enabled
    All others = 0
    Full interpretation: requires nRF firmware protocol documentation
    Each bit corresponds to a specific sensor/feature enable flag

  Additional attributes documented in hardware database (not read this session):
    acc_dir, baro_enable, baro_value, altitude_value, spo2_simu, step_simu,
    tilt_simu, tempearture_value (ODM typo preserved)

# 4.8 CONFIRMED SENSOR HARDWARE — nRF52832 SIDE

  These sensors are managed by the nRF52832 internally.
  The MT6762 does NOT have direct kernel drivers for any of these.
  All data arrives via wiite_corp_ctrl sysfs or UART (ttyS1).

  Sensor          IC Model      Interface         Status        Notes
  ──────────────────────────────────────────────────────────────────────────────
  Heart Rate      PAR2822       nRF internal I2C  ACTIVE        97 BPM captured
  SpO2            PAR2822       nRF internal I2C  ACTIVE        spo2_simu node
  Barometer       unknown       nRF internal I2C  ACTIVE        baro_value node
  Altitude        derived       nRF internal calc ACTIVE        altitude_value
  Temperature     unknown       nRF internal      ACTIVE        tempearture_value
  Accelerometer   unknown       nRF internal I2C  ACTIVE        acc_dir node
  Step Counter    nRF firmware  nRF internal      ACTIVE        step_simu node
  Tilt/Gesture    nRF firmware  nRF internal      ACTIVE        tilt_simu node

  NOTE: Barometer IC — BMP280 absent from ODM schematics referenced in
        hardware database. Actual model unknown. On nRF internal I2C.
  NOTE: Accelerometer IC — LIS2HH12 absent per hardware database. Unknown
        model. On nRF internal I2C.
  NOTE: Gyroscope — NO evidence of gyroscope hardware on this device.
        dumpsys fusion states all show "disabled" — confirms no gyro.
        9-axis, game, geomag fusion all report 0 clients.

# 4.9 PAR2822 PIXART OPTICAL SENSOR — KERNEL INTERFACE

  Device tree node  : /i2c4@11011000/heartrate@33
  I2C bus           : I2C4 (11011000.i2c4)
  I2C address       : 0x33
  DTS compatible    : mediatek,heartrate
  Kernel driver     : NONE BOUND (confirmed — no DRIVER field in uevent)
  MODALIAS          : i2c:heartrate
  DTS path in sysfs : /sys/firmware/devicetree/base/i2c4@11011000/heartrate@33

  Stack B access path:
    libsensor_custom.so → /dev/i2c-4 → ioctl I2C_RDWR → address 0x33
    Requires: mediatek,heartrate driver bound to claim device
    Without driver: ioctl fails → no data → heart rate/SpO2 dead

  Physical connection:
    PAR2822 is on nRF52832 internal I2C bus (nRF manages the sensor)
    The heartrate@33 DTS entry is the MT6762 side interface to receive
    processed data from the nRF52832, NOT a direct connection to PAR2822.
    The nRF acts as intermediary — MT6762 driver talks to nRF,
    nRF reads PAR2822 directly.

# 4.10 INPUT DEVICES (from /proc/bus/input/devices — live device)

  3 input devices registered:

  event0 — mtk-kpd (MTK keypad)
    Bus     : 0019 (platform)
    Vendor  : 2454
    Product : 6500
    Version : 0010
    Sysfs   : /devices/platform/10010000.kp/input/input0
    Keys    : KEY_A, KEY_VOLUMEUP, KEY_POWER, KEY_BACK, KEY_CAMERA, KEY_RESTART

  event1 — hyn_ts (Hynitron touchscreen)
    Bus     : 0018 (I2C)
    I2C     : 0-005d (I2C0, address 0x5D)
    Sysfs   : /devices/platform/11007000.i2c0/i2c-0/0-005d/input/input2
    ABS     : MT_SLOT (0-4), MT_TOUCH_MAJOR, MT_WIDTH_MAJOR,
              MT_POSITION_X (0-480), MT_POSITION_Y (0-640),
              MT_TRACKING_ID (0-5), MT_PRESSURE
    Note    : 5-finger multitouch, 480x640 resolution

  event2 — mtk-tpd (MTK touch panel driver)
    Bus     : 0000 (virtual)
    Sysfs   : /devices/virtual/input/input1
    ABS     : ABS_X (0-480), ABS_Y (0-640), ABS_PRESSURE,
              MT_TOUCH_MAJOR, MT_TOUCH_MINOR,
              MT_POSITION_X (0-480), MT_POSITION_Y (0-640),
              MT_TRACKING_ID (0-10)
    Note    : Virtual input layer over hyn_ts

  NOTE: /sys/class/sensors/ does NOT exist on this device.
        Sensors are not exposed via the Android sysfs sensor class.
        All sensor data routes through the HIDL sensor HAL only.

# 4.11 MISC DEVICE INVENTORY (sensor-relevant entries)

  sensorlist → /devices/virtual/misc/sensorlist
    MTK sensor list virtual device — used by sensors.mt6765.so
    to enumerate available sensors from the HAL implementation.

  wiite_corp_ctrl → /devices/virtual/misc/wiite_corp_ctrl
    nRF52832 primary control interface. Created 2026-05-07 21:03 (boot time).
    All 34 attributes accessible via this path.

  wiite_con_ctrl → /devices/virtual/misc/wiite_con_ctrl
    Secondary Wiite control interface. Created 2026-05-10 04:08.
    Created later than wiite_corp_ctrl — probe sequence difference.
    Exact role: TBD (not yet fully characterized).

  touch → /devices/virtual/misc/touch
    MTK touch panel misc interface.

# 4.12 ROM BUILD REQUIREMENTS FOR FULL SENSOR OPERATION

  The following MUST be true for all 10 sensors to function:

  1. wiite_corp driver compiled into kernel (or as loadable module)
     Compatible : wiite,cp
     Provides   : /sys/class/misc/wiite_corp_ctrl/ interface
     Enables    : All nRF52832-based sensor data (Stack A)

  2. mediatek,heartrate driver compiled into kernel (or as loadable module)
     Compatible : mediatek,heartrate
     I2C        : bus 4, address 0x33
     Provides   : bound driver for PAR2822 interface node
     Enables    : Stack B (libsensor_custom.so) heart rate + SpO2

  3. /vendor/lib64/hw/sensors.mt6765.so — carry from stock vendor
     Required   : Stack A HAL implementation

  4. /vendor/lib64/libsensor_custom.so — carry from stock vendor
     Required   : Stack B PAR2822 direct access

  5. /vendor/lib64/libksensor.so — carry from stock vendor
     Required   : Calibration math dependency of sensors.mt6765.so

  6. init.recovery.mt6762.rc or init.mt6762.rc wiite_corp_ctrl permissions
     Required   : chmod 0666 on all wiite_corp_ctrl attributes
     Without    : HAL cannot read sensor data from sysfs

  7. ttyS1 permissions: chmod 666 /dev/ttyS1
     Required   : nRF UART communication channel
     Set in     : init.mt6762.rc boot trigger

  GSI result without items 1-7: Accelerometer works (kernel built-in MTK accel),
  everything else fails silently. Items 1 and 2 are the critical kernel drivers.

################################################################################
# SECTION 5 — DISPLAY & GPU STACK
################################################################################

# 5.1 DISPLAY HARDWARE

  Panel IC          : Novatek NT35695B
  Panel mfr         : AUO (AU Optronics)
  Backlight IC      : Richtek RT5081 (PMU charger with backlight controller)
  DSI driver string : nt35695B_fhd_dsi_cmd_auo_rt5081_drv
  Resolution        : 480 × 640 pixels (portrait)
  Physical density  : 146 PPI (from DTBO tpd-resolution-y/pixel-density)
  SF density        : ro.sf.lcd_density=160 (system-reported)
  ODM override dpi  : 186 dpi (framework overlay — inflates UI scale)
  Pixel format      : RGBX_8888 (32-bit, zero padding confirmed)
  Stride            : 1920 bytes (480 pixels × 4 bytes)
  DSI mode          : Command mode (CMD) — display retains frame, no continuous refresh
  FPS               : 60 Hz confirmed (VSYNC period = 16,666,667 ns live)
  Smart 90          : off — no 90Hz mode on this device
  Cutout            : none — rectangular panel
  VRAM              : 0x17bb000 = 25,034,752 bytes (~23.9 MB)
  Framebuffer base  : high word 0x00 (upper 32-bit)

  Build display ID (ro.build.display.id):
    APPLLP_5_MAX_LMT_WTC17S_EN_480_640_V1.5_20250613
    Fields: model=APPLLP_5_MAX, region=EN, res=480_640,
            version=V1.5, date=20250613

  Color capability:
    Wide color display : NO (SurfaceFlinger confirmed: 0)
    Color management   : NO (SurfaceFlinger confirmed: 0)
    Color mode         : ColorMode::NATIVE (0) only — no DCI-P3, no sRGB managed
    Display ID data    : none (no panel EDID equivalent)

  Backlight:
    Node        : /sys/class/leds/lcd-backlight/brightness
    Max value   : 255 (confirmed live)
    Current     : 0 (display off during ADB session — confirmed live)
    PWM channel : 3 (mode 5 = PWM, from DTBO led@6 fragment)
    GPIO        : GPIO 159 = external amplifier enable (separate from backlight)

  Device nodes (all confirmed live):
    /dev/graphics/fb0   — framebuffer, major 29 minor 0,
                          permissions: system:graphics crw-rw-r--
    /dev/mtk_disp_mgr   — MTK display manager, system:graphics, chmod 0664
    /dev/mtkfb_vsync    — display vsync, system:graphics, chmod 0660
    /dev/sw_sync        — software sync, system:graphics, chmod 0660

  VSYNC/Scheduler:
    VSYNC source      : SyntheticVSync (not hardware vsync signal)
    VSYNC period      : 16,666,667 ns = 60.000 Hz (exact)
    SF Scheduler      : enabled
    App phase offset  : 8,300,000 ns (8.3ms)
    SF phase offset   : 8,300,000 ns (8.3ms)
    Present offset    : 0 ns
    Backpressure      : disabled (debug.sf.disable_backpressure=1)
    Present timestamp : enabled (service.sf.present_timestamp=1)
    Latch unsignaled  : enabled (vendor.debug.sf.latch_unsignaled=1)

  SurfaceFlinger build config (confirmed live):
    PRESENT_TIME_OFFSET         : 0
    FORCE_HWC_FOR_RBG_TO_YUV   : 1
    MAX_VIRT_DISPLAY_DIM        : 0
    RUNNING_WITHOUT_SYNC_FRAMEWORK: 0
    NUM_FRAMEBUFFER_SURFACE_BUFFERS: 3

  Display orientation:
    ro.surface_flinger.primary_display_orientation: 0 (portrait native)

  HDR:
    vendor.debug.sf.hdr_enable: 0 — HDR disabled

# 5.2 GPU

  GPU silicon       : PowerVR GE8320 (Rogue architecture — codename MEOW)
  Driver version    : Rogue_DDK_Android rogueddk 1.11@5425693 (release)
  HW version        : 22.87.104.18
  Firmware file     : /vendor/firmware/rgx.fw (118,784 bytes)
  EGL HAL           : ro.hardware.egl=mtk

  API support:
    OpenGL ES       : 3.2 (ro.opengles.version=196610 = 0x30002)
                      NOTE: Draft section stated 3.0 — CORRECTED to 3.2
    Vulkan          : supported (vulkan.mt6765.so present in vendor/lib64/hw/)
    OpenCL          : supported via libOpenCL.so → libPVROCL.so
    HWUI Vulkan     : NOT used (ro.hwui.use_vulkan is empty)
    OpenGL AEP      : declared in vendor/etc/permissions/android.hardware.opengles.aep.xml

  Vulkan feature declarations:
    android.hardware.vulkan.compute  version=0
    android.hardware.vulkan.level    version=0
    android.hardware.vulkan.version  version=4198400 (Vulkan 1.1)

  PowerVR live status (from /sys/kernel/debug/pvr/status — live device):
    Driver Status   : OK
    Firmware Status : OK
    HWR Event Count : 0    — zero hardware resets
    CRR Event Count : 0    — zero context reset requests
    FWF Event Count : 0    — zero firmware faults
    APM Event Count : 125507 — normal GPU power management cycling
    GPU Utilisation : 3%   — idle at capture time

  Services:
    init.svc.gpu      : running (confirmed live)
    init.svc.gpu-1-0  : running (confirmed live)
    ro.boottime.gpu   : 15,650,190,769 ns (~15.6 seconds from boot zero)

# 5.3 COMPOSER PROCESS — RUNTIME LIBRARY MAP

  Service binary : /vendor/bin/hw/android.hardware.graphics.composer@2.1-service
  PID            : 526 (confirmed — vendor.debug.sf.hwc_pid=526)

  Libraries confirmed from /proc/526/maps (live device):

  Source                                Library
  ────────────────────────────────────────────────────────────────────────────
  /vendor/bin/hw/                       android.hardware.graphics.composer@2.1-service
  /system/lib64/vndk-sp-29/hw/          android.hidl.memory@1.0-impl.so
  /vendor/lib64/hw/                     gralloc.mt6765.so
  /vendor/lib64/hw/                     android.hardware.graphics.mapper@2.0-impl-2.1.so
  /vendor/lib64/                        libmpvr.so
  /vendor/lib64/                        libdrm.so

  NOTE: /proc/526/maps was truncated in capture — additional libraries present
        in process memory. Full library inventory from Section 5.4 documents
        the complete display stack.

  Gralloc mapper version:
    android.hardware.graphics.mapper@2.0-impl-2.1.so — mapper 2.0 impl with 2.1
    shim. The HIDL interface is mapper@2.1 (passthrough, confirmed in lshal)
    loaded by the widest client set on the device.

# 5.4 COMPLETE DISPLAY HAL LIBRARY INVENTORY

  HWComposer (HWC2) implementation:
    /vendor/lib64/hw/hwcomposer.mt6765.so         682,224 bytes
    /vendor/lib64/hw/android.hardware.graphics.composer@2.1-impl.so  104,224 bytes

  Gralloc / graphics memory:
    /vendor/lib64/hw/gralloc.mt6765.so            616,632 bytes
    /vendor/lib64/hw/android.hardware.graphics.mapper@2.0-impl-2.1.so 38,696 bytes
    /vendor/lib64/hw/android.hardware.graphics.allocator@2.0-impl.so  24,904 bytes
    /vendor/lib64/libgralloc_extra.so
    /vendor/lib64/libion_mtk.so
    /vendor/lib64/libion_ulit.so

  HWComposer adapters:
    /vendor/lib64/libhwc2on1adapter.so
    /vendor/lib64/libhwc2onfbadapter.so

  PowerVR GPU userspace stack:
    /vendor/lib64/egl/libEGL_mtk.so               14,000 bytes
    /vendor/lib64/egl/libGLES_meow.so          1,999,944 bytes  ← GLES impl
    /vendor/lib64/egl/libGLESv1_CM_mtk.so        706,528 bytes
    /vendor/lib64/egl/libGLESv2_mtk.so         1,591,296 bytes
    /vendor/lib64/egl/libMEOW_trace.so           535,640 bytes
    /vendor/lib64/libIMGegl.so
    /vendor/lib64/libusc.so
    /vendor/lib64/libsrv_um.so
    /vendor/lib64/libglslcompiler.so
    /vendor/lib64/libmpvr.so
    /vendor/lib64/hw/vulkan.mt6765.so            683,024 bytes
    /vendor/lib64/libPVROCL.so               34,741,696 bytes  ← OpenCL impl
    /vendor/lib64/libOpenCL.so                              ← public wrapper

  MTK GPU management:
    /vendor/lib64/libgpu_aux.so
    /vendor/lib64/libgpud.so
    /vendor/lib64/libged.so
    /vendor/lib64/vendor.mediatek.hardware.gpu@1.0.so
    /vendor/lib64/libgpudataproducer.so

  MTK display / picture quality:
    /vendor/lib64/libdpframework.so
    /vendor/lib64/libpq_cust_base.so
    /vendor/lib64/libpq_prot.so
    /vendor/lib64/vendor.mediatek.hardware.pq@2.3-impl.so     177,208 bytes
    /vendor/lib64/vendor.mediatek.hardware.pq@2.0.so
    /vendor/lib64/libladder.so
    /vendor/lib64/libdrm.so

  Power integration:
    /vendor/lib64/vendor.mediatek.hardware.mtkpower@1.0.so
    /vendor/lib64/libmtkperf_client_vendor.so

  32-bit counterparts:
    /vendor/lib/egl/ — complete 32-bit mirror of all EGL/GLES libs
    /vendor/lib/egl/egl.cfg — EGL config file (32-bit path only)
    NOTE: egl.cfg EXISTS at vendor/lib/egl/egl.cfg — confirmed.
          Previously documented as absent — that was incorrect.

# 5.5 EGL CONFIGURATION

  ro.hardware.egl          : mtk
  EGL config file          : /vendor/lib/egl/egl.cfg (32-bit path)
  Sync extensions          : EGL_ANDROID_native_fence_sync, EGL_KHR_wait_sync
                             (confirmed from SurfaceFlinger sync config)
  HDR                      : not supported (vendor.debug.sf.hdr_enable=0)
  Wide color               : not supported (SurfaceFlinger confirmed)

# 5.6 FIRMWARE FILES (/vendor/firmware/ — complete inventory)

  File                                Size (bytes)  Purpose
  ──────────────────────────────────────────────────────────────────────────────
  rgx.fw                               118,784      PowerVR GE8320 GPU firmware
  WIFI_RAM_CODE_soc1_0_1_1.bin         612,652      WiFi coprocessor RAM code
  soc1_0_ram_bt_1_1_hdr.bin            570,765      Bluetooth RAM firmware
  soc1_0_patch_mcu_1_1_hdr.bin          26,350      WMT MCU patch binary
  soc1_0_ram_mcu_1_1_hdr.bin           104,324      WMT MCU RAM binary
  soc1_0_ram_wifi_1_1_hdr.bin          195,228      WiFi RAM secondary binary
  gt1151_default_firmware2.img          63,616      Goodix GT1151 touch firmware
  BT_FW.cfg                                410      Bluetooth firmware config
  WMT_SOC.cfg                              121      WMT SoC config
  MNL.bin                                  128      MNL binary (modem nav)

  NOTE: gt1151_default_firmware2.img — Goodix GT1151 is an alternate touch
        controller IC. Live device has Hynitron hyn_ts at I2C0 0x5D.
        This is an ODM BSP leftover — the board was designed to support either
        Goodix GT1151 or Hynitron as touch controller. Production unit shipped
        with Hynitron. Goodix firmware is present but unused and never loaded.

  ROM BUILD: All files in /vendor/firmware/ must be carried into the ROM.
             rgx.fw is required for GPU operation. WiFi/BT firmware binaries
             are required for connectivity. gt1151 can be carried as-is —
             it will not load if no GT1151 device is probed.

# 5.7 DISPLAY PIPELINE DIAGRAM

  Hardware flow:
    MT6762 display subsystem
      → DSI0 (command mode)
        → NT35695B panel (480×640, AUO)
          → RT5081 backlight controller (PWM channel 3, GPIO PWM)

  Software flow:
    Android SurfaceFlinger
      → HWC2 (hwcomposer.mt6765.so via android.hardware.graphics.composer@2.1)
        → MTK display driver (kernel side)
          → RDMA → OVL → DSI CMD → panel

  GPU path:
    App → GLES 3.2 (libGLES_meow.so / PowerVR GE8320)
      → gralloc buffer (gralloc.mt6765.so)
        → HWC2 composition
          → framebuffer output

  Memory management:
    ION allocator (libion_mtk.so)
      → gralloc_extra metadata
        → mapper@2.1 passthrough
          → DMA-BUF kernel interface
################################################################################
# SECTION 6 — CAMERA STACK
################################################################################

# 6.1 CAMERA ARCHITECTURE

  Framework process : cameraserver (PID 651)
  HAL process       : camerahalserver (PID 692)
  HIDL interface    : android.hardware.camera.provider@2.4 — hwbinder
  HAL type          : Camera2 / HAL3
  HAL impl          : android.hardware.camera.provider@2.4-impl-mediatek.so
  Camera API        : ro.vendor.mtk_camera_app_version=3
  Android devices   : 2 (Device 0="0", Device 1="1" — confirmed dumpsys)
  Active clients    : none at capture time
  Allowed user IDs  : 0

  Boot times:
    cameraserver    : 16,538,523,153 ns (~16.5 seconds from boot zero)
    camerahalserver : 16,742,365,384 ns (~16.7 seconds from boot zero)

  ZSL:
    camera.disable_zsl_mode=1 — ZSL disabled
    ro.vendor.camera3.zsl.default=130

  MDP processing:
    vendor.camera.mdp.cz.enable=1  — Color Zone enabled
    vendor.camera.mdp.dre.enable=0 — Dynamic Range Enhancement disabled

  Shutter sound: ro.camera.sound.forced=0 — not forced

# 6.2 ACTIVE CAMERA SENSORS

  Source: /proc/driver/camera_info (live device), DTBO fragments @18 @41 @42,
          I2C live sysfs, dumpsys media.camera

  ─── CAM_Info[0] — REAR CAMERA ───────────────────────────────────────────────

  Sensor model      : GalaxyCore GC2385
  Android device ID : 0
  Library prefix    : gc2385_mipi_raw
  I2C bus           : I2C2 (11009000.i2c2), address 0x1A
  MCLK GPIO         : GPIO 100, function 1 (MCLK output active)
                      GPIO 100, function 0 (GPIO idle/off)
  RST GPIO          : GPIO 102 — assert=output-low, deassert=output-high
  PWR_DOWN GPIO     : GPIO 98  — low=powered, high=powered-down
  EEPROM            : I2C2 address 0x50
  AF controller     : I2C2 address 0x0C
  Interface         : MIPI serial (Seninf_Type=1)
  MIPI lanes        : 1
  Output format     : Bayer B-first (output_format=0)
  MCLK frequency    : 24 MHz
  Driving current   : 8 mA (Driving_Current=3)
  Preview res       : 1600 × 1200 (2MP, 4:3)
  Capture res       : 1600 × 1200
  Video res         : 1600 × 1200
  HSV/SLV res       : 1600 × 1200
  Grab offset       : x=0, y=0 (full frame, no crop)
  Delay frames      : 2
  PDAF              : NOT supported (PDAF_Support=0)
  HDR               : NOT supported (HDR_Support=0)
  ISP tuning blob   : gc2385_mipi_raw_tuning.so  (2,453,160 bytes)
  Index manager     : gc2385_mipi_raw_IdxMgr.so  (15,024 bytes)

  Power supply:
    vcama-supply    : mt_pmic_vcama_ldo_reg    (camera analog)
    vcamd-supply    : mt_pmic_vcamd_ldo_reg    (camera digital)
    vcamio-supply   : mt_pmic_vcamio_ldo_reg   (camera IO)
    vcamaf-supply   : mt_pmic_vldo28_ldo_reg   (AF motor — shared with touch)

  ─── CAM_Info[1] — FRONT CAMERA ──────────────────────────────────────────────

  Sensor model      : GalaxyCore GC030A
  Android device ID : 1
  Library prefix    : gc030a_mipi_raw
  I2C bus           : I2C2 (11009000.i2c2), address 0x2D
  MCLK GPIO         : GPIO 99, function 1 (MCLK output active)
                      GPIO 99, function 0 (GPIO idle/off)
  RST GPIO          : GPIO 101 — assert=output-low, deassert=output-high
  PWR_DOWN GPIO     : GPIO 97  — low=powered, high=powered-down
  EEPROM            : I2C2 address 0x52
  Interface         : MIPI serial (Seninf_Type=1)
  MIPI lanes        : 1
  Output format     : Bayer B-first (output_format=0)
  MCLK frequency    : 24 MHz
  Driving current   : 6 mA (Driving_Current=2)
  Preview res       : 640 × 480 (VGA, 4:3)
  Capture res       : 640 × 480
  Video res         : 640 × 480
  HSV/SLV res       : 640 × 480
  Grab offset       : x=0, y=0
  Delay frames      : 2
  PDAF              : NOT supported (PDAF_Support=0)
  HDR               : NOT supported (HDR_Support=0)
  ISP tuning blob   : gc030a_mipi_raw_tuning.so  (2,391,712 bytes)
  Index manager     : gc030a_mipi_raw_IdxMgr.so  (15,024 bytes)

  Power supply:
    vcamio_sub-supply : mt_pmic_vcamio_ldo_reg
    vcamaf_sub-supply : mt_pmic_vldo28_ldo_reg

# 6.3 THIRD CAMERA SLOT — BF2553L (BLOBS PRESENT, NOT ACTIVE)

  Sensor model      : BYD Microelectronics BF2553L
  Sensor ID         : 9555 (0x2553)
  DTS slot          : I2C6 (1100d000.i2c6), address 0x2D
                      DTS entry: camera_main_two@2d
  EEPROM            : I2C6 address 0x50
  AF                : I2C6 address 0x0C
  ISP tuning blob   : bf2553L_mipi_raw_tuning.so  (2,367,144 bytes)
  Index manager     : bf2553L_mipi_raw_IdxMgr.so  (15,024 bytes)

  Power supply:
    vcamio_main2-supply : mt_pmic_vcamio_ldo_reg
    vcamaf_main2-supply : mt_pmic_vldo28_ldo_reg

  STATUS: BLOBS PRESENT — NOT ACTIVE AT RUNTIME
    /proc/driver/camera_info shows only CAM_Info[0] and CAM_Info[1].
    dumpsys media.camera confirms exactly 2 Android camera devices.
    bf2553L does NOT appear in the live camera subsystem.
    Physical sensor is either not populated on this production PCB
    or not probing successfully on I2C6.

  CRITICAL ROM NOTE: bf2553L blobs MUST be carried into the ROM.
    The MTK camera HAL loader enumerates all declared sensor libraries
    at startup. Removing these blobs causes HAL initialization failure
    even though bf2553L is not physically active on this device.

# 6.4 DORMANT BSP SENSOR LIBRARIES — SONY PDAF

  8 Sony PDAF libraries present in /vendor/lib64/ — none loaded at runtime.
  MT6765 platform BSP inclusions for Sony-sensor device variants.
  This device uses GalaxyCore sensors only — Sony libs are unused scaffolding.

  Library                              Size (bytes)
  ─────────────────────────────────────────────────────
  libSonyIMX230PdafLibrary.so              14,904
  libSonyIMX230PdafLibraryWrapper.so       35,704
  libSonyIMX338PdafLibrary.so              14,904
  libSonyIMX338PdafLibraryWrapper.so       35,712
  libSonyIMX386PdafLibrary.so              14,904
  libSonyIMX386PdafLibraryWrapper.so       35,712
  libSonyIMX519PdafLibrary.so              14,904
  libSonyIMX519PdafLibraryWrapper.so       35,704

  CRITICAL ROM NOTE: ALL 8 must be carried into the ROM.
    The MTK camera HAL loader requires their presence during initialization.
    Removing any of them causes camera HAL initialization failure regardless
    of whether those sensors are physically present on this device.

# 6.5 LOMOEFFECT HAL

  /vendor/lib64/hw/vendor.mediatek.hardware.camera.lomoeffect@1.0-impl.so
    Size   : 24,736 bytes
  /vendor/lib64/vendor.mediatek.hardware.camera.lomoeffect@1.0.so
    Status : present, not loaded in any running process at capture time

  ROM NOTE: Carry as-is. Required by HAL loader — do not remove.

# 6.6 HAL IMPLEMENTATION LIBRARY INVENTORY

  HAL core (in /vendor/lib64/hw/):
    android.hardware.camera.provider@2.4-impl-mediatek.so   153,200 bytes
    vendor.mediatek.hardware.camera.bgservice@1.1-impl.so    46,992 bytes
    vendor.mediatek.hardware.camera.lomoeffect@1.0-impl.so   24,736 bytes

  MTK camera framework (in /vendor/lib64/):
    libcam.feature_utils.so                                 198,120 bytes

  MTK camera plugin (in /vendor/lib64/mtkcam/):
    libmtkcam_streaminfo_plugin-p1stt.so                     29,328 bytes
    NOTE: This is the only file in the /vendor/lib64/mtkcam/ subdirectory.

  HIDL interfaces:
    vendor.mediatek.hardware.camera.bgservice@1.0.so (vendor/lib64/)
    vendor.mediatek.hardware.camera.bgservice@1.1.so (vendor/lib64/)
    vendor.mediatek.hardware.camera.lomoeffect@1.0.so (vendor/lib64/)

  ISP tuning blobs (all in /vendor/lib64/):
    gc2385_mipi_raw_tuning.so      2,453,160 bytes  active rear sensor
    gc2385_mipi_raw_IdxMgr.so         15,024 bytes  active rear sensor
    gc030a_mipi_raw_tuning.so      2,391,712 bytes  active front sensor
    gc030a_mipi_raw_IdxMgr.so         15,024 bytes  active front sensor
    bf2553L_mipi_raw_tuning.so     2,367,144 bytes  inactive third slot
    bf2553L_mipi_raw_IdxMgr.so        15,024 bytes  inactive third slot

# 6.7 CAMERAHALSERVER RUNTIME LIBRARY MAP

  Service binary : /vendor/bin/hw/camerahalserver (15,880 bytes)
  PID            : 692 (confirmed)

  Confirmed from /proc/692/maps (live device — Gemini truncated output):

  Source                    Library
  ──────────────────────────────────────────────────────────────────────
  /vendor/bin/hw/           camerahalserver
  /vendor/lib64/mtkcam/     libmtkcam_streaminfo_plugin-p1stt.so

  NOTE: Full /proc/692/maps was truncated in capture. The camerahalserver
        process loads extensively from vendor/lib64/ including all ISP
        tuning blobs, bgservice impl, lomoeffect impl, and the provider
        impl. Complete map re-pull deferred — partial capture documented
        as-is. Library chain from binary analysis covers remainder.

# 6.8 CAMERA PIPELINE BUFFER CONFIGURATION

  Source: persist.vendor.camera3.pipeline.bufnum.* (live device)

  Base buffer counts (all streams):
    imgo (raw output)            : 5
    lcso (lens correction stats) : 5
    rrzo (resized raw output)    : 5
    rsso (resolution switch)     : 5

  High RAM minimum buffer counts:
    fdyuv (face detect YUV)      : 8
    imgo                         : 8
    lcso                         : 8
    rrzo                         : 8
    rsso                         : 8

  Low RAM minimum buffer counts:
    fdyuv                        : 6
    imgo                         : 6
    lcso                         : 6
    rrzo                         : 6
    rsso                         : 6

# 6.9 FLASH

  Status            : FUNCTIONAL (confirmed by device owner)
  Declaration       : android.hardware.camera.flash in
                      vendor/etc/permissions/android.hardware.camera.xml
  Force disable     : persist.sys.force.no.flashlight=false
  Shutter sound     : ro.camera.sound.forced=0

# 6.10 FEATURE DECLARATIONS

  Source: vendor/etc/permissions/android.hardware.camera.xml (live device)

  Feature                                              Status
  ────────────────────────────────────────────────────────────────────────
  android.hardware.camera                              ACTIVE — GC2385
  android.hardware.camera.autofocus                    ACTIVE — AF on I2C2 0x0C
  android.hardware.camera.flash                        ACTIVE — confirmed
  android.hardware.camera.front                        ACTIVE — GC030A
  android.hardware.camera.level.full                   declared
  android.hardware.camera.capability.manual_sensor     declared
  android.hardware.camera.capability.manual_post_processing  declared
  android.hardware.camera.capability.raw               declared
  android.hardware.camera.any                          ACTIVE — confirmed

  Additional capabilities (from MTK capability matrix):
    Face detection    : mtk_fd_support=1
    Slow motion       : mtk_slow_motion_support=1

# 6.11 ROM BUILD REQUIREMENTS

  The following vendor blobs are ALL required. Missing any one causes
  camera HAL initialization failure:

  HAL implementation:
    android.hardware.camera.provider@2.4-impl-mediatek.so
    vendor.mediatek.hardware.camera.bgservice@1.1-impl.so
    vendor.mediatek.hardware.camera.lomoeffect@1.0-impl.so
    libcam.feature_utils.so
    libmtkcam_streaminfo_plugin-p1stt.so (path: vendor/lib64/mtkcam/)

  Active sensor tuning:
    gc2385_mipi_raw_tuning.so
    gc2385_mipi_raw_IdxMgr.so
    gc030a_mipi_raw_tuning.so
    gc030a_mipi_raw_IdxMgr.so

  Inactive sensor tuning (required by HAL loader — carry unchanged):
    bf2553L_mipi_raw_tuning.so
    bf2553L_mipi_raw_IdxMgr.so
    libSonyIMX230PdafLibrary.so
    libSonyIMX230PdafLibraryWrapper.so
    libSonyIMX338PdafLibrary.so
    libSonyIMX338PdafLibraryWrapper.so
    libSonyIMX386PdafLibrary.so
    libSonyIMX386PdafLibraryWrapper.so
    libSonyIMX519PdafLibrary.so
    libSonyIMX519PdafLibraryWrapper.so################################################################################
# SECTION 7 — AUDIO STACK
################################################################################

# 7.1 AUDIO ARCHITECTURE

  Primary HAL service : android.hardware.audio@5.0-service-mediatek
  HAL binary          : audio.primary.mt6765.so
  audioserver process : /system/bin/audioserver
  HAL version         : 3.0 (primary module)
  Config file         : /vendor/etc/audio_policy_configuration.xml
  audio_device.xml    : DOES NOT EXIST — confirmed (No such file)
  audio_policy.conf   : DOES NOT EXIST — confirmed (No such file)
  mixer_paths.xml     : DOES NOT EXIST — audio routing compiled into
                        audio.primary.mt6765.so directly (standard MTK)

  Boot times:
    audioserver             : 15,637,206,076 ns (~15.6 sec from boot zero)
    vendor.audio-hal-2-0   : 15,486,781,615 ns (~15.5 sec from boot zero)

  Services running (confirmed live):
    init.svc.audioserver          : running
    init.svc.vendor.audio-hal-2-0 : running

  Global config: speaker_drc_enabled="false"

  Key properties:
    persist.vendor.audiohal.besloudness_state : 1 (BES Loudness enabled)
    persist.vendor.audiohal.firstboot         : 0
    persist.vendor.bluetooth.a2dpstandbytime  : 500 ms
    ro.audio.silent                           : 0
    ro.audio.usb.period_us                    : 16000 (16ms USB period)
    ro.vendor.mtk_audio_alac_support          : 1 (Apple Lossless)
    ro.vendor.mtk_audio_ape_support           : 1 (Monkey's Audio)
    ro.vendor.mtk_audio_tuning_tool_ver       : V2.2
    vendor.audiohal.speech.shm_init           : 1 (speech shared mem init)

# 7.2 AUDIO MODULES

  Five modules defined in audio_policy_configuration.xml:
    1. primary          — primary audio HAL (this file)
    2. a2dp             — legacy A2DP HAL (xi:include)
    3. bluetooth        — BT Audio HAL 2.0 (xi:include)
    4. usb              — USB audio HAL (xi:include)
    5. r_submix         — remote submix (xi:include)

  Volume tables included:
    /vendor/etc/audio_policy_volumes.xml
    /vendor/etc/default_volume_tables.xml

# 7.3 PRIMARY MODULE — MIX PORTS

  Source: /vendor/etc/audio_policy_configuration.xml (full confirmed)

  ─── OUTPUT MIX PORTS ────────────────────────────────────────────────────────

  primary output
    Flags    : AUDIO_OUTPUT_FLAG_PRIMARY | AUDIO_OUTPUT_FLAG_FAST
    PCM 32-bit : 44100, 48000 Hz — AUDIO_CHANNEL_OUT_STEREO
    PCM 16-bit : 44100, 48000 Hz — AUDIO_CHANNEL_OUT_STEREO

  deep_buffer
    Flags    : AUDIO_OUTPUT_FLAG_DEEP_BUFFER
    PCM 32-bit : 44100, 48000 Hz — AUDIO_CHANNEL_OUT_STEREO
    PCM 16-bit : 44100, 48000 Hz — AUDIO_CHANNEL_OUT_STEREO

  incall_music_uplink
    Flags    : AUDIO_OUTPUT_FLAG_INCALL_MUSIC
    PCM 32-bit : 44100, 48000 Hz — AUDIO_CHANNEL_OUT_STEREO
    PCM 16-bit : 44100, 48000 Hz — AUDIO_CHANNEL_OUT_STEREO

  ─── INPUT MIX PORTS ─────────────────────────────────────────────────────────

  primary input
    Role           : sink
    maxOpenCount   : 1
    maxActiveCount : 1
    PCM 16-bit     : 8000, 16000, 32000, 44100, 48000 Hz
    Channels       : MONO, STEREO, VOICE_UPLINK, VOICE_DNLINK

  voice tx
    Role           : sink
    maxOpenCount   : 1
    maxActiveCount : 1
    PCM 16-bit     : 8000, 16000, 32000, 44100, 48000 Hz
    Channels       : MONO, STEREO, VOICE_UPLINK, VOICE_DNLINK

  FM Tuner input
    Role           : sink
    maxOpenCount   : 1
    maxActiveCount : 1
    PCM 16-bit     : 8000, 16000, 32000, 44100, 48000 Hz
    Channels       : MONO, STEREO, VOICE_UPLINK, VOICE_DNLINK

# 7.4 PRIMARY MODULE — DEVICE PORTS

  Source: /vendor/etc/audio_policy_configuration.xml (full confirmed)

  PERMANENTLY ATTACHED DEVICES:
    Speaker, Earpiece, Built-In Mic, Built-In Back Mic,
    FM Tuner In, Voice Call In, Echo Ref In, Telephony Tx

  DEFAULT OUTPUT: Speaker

  ─── OUTPUT DEVICE PORTS ─────────────────────────────────────────────────────

  Earpiece (AUDIO_DEVICE_OUT_EARPIECE)
    PCM 32/16-bit : 44100, 48000 Hz — MONO
    NOTE: No physical earpiece on watch form factor — BSP leftover

  Speaker (AUDIO_DEVICE_OUT_SPEAKER)
    PCM 32/16-bit : 44100, 48000 Hz — STEREO
    PRIMARY active output path on this wrist-worn device

  Wired Headset (AUDIO_DEVICE_OUT_WIRED_HEADSET)
    PCM 32/16-bit : 44100, 48000 Hz — STEREO
    NOTE: No 3.5mm jack on this device — BSP leftover

  Wired Headphones (AUDIO_DEVICE_OUT_WIRED_HEADPHONE)
    PCM 32/16-bit : 44100, 48000 Hz — STEREO
    NOTE: No 3.5mm jack on this device — BSP leftover

  AUX Digital Out (AUDIO_DEVICE_OUT_AUX_DIGITAL)
    PCM 32/16-bit : 44100, 48000 Hz — STEREO

  BT SCO (AUDIO_DEVICE_OUT_BLUETOOTH_SCO)
    PCM 16-bit : 8000, 16000 Hz — MONO
    NOTE: MONO confirmed. Call audio path at narrowband/wideband rates only.

  BT SCO Headset (AUDIO_DEVICE_OUT_BLUETOOTH_SCO_HEADSET)
    PCM 16-bit : 8000, 16000 Hz — MONO

  BT SCO Car Kit (AUDIO_DEVICE_OUT_BLUETOOTH_SCO_CARKIT)
    PCM 16-bit : 8000, 16000 Hz — MONO

  Telephony Tx (AUDIO_DEVICE_OUT_TELEPHONY_TX)
    PCM 16-bit : 8000, 16000 Hz — MONO, STEREO

  ─── INPUT DEVICE PORTS ──────────────────────────────────────────────────────

  Built-In Mic (AUDIO_DEVICE_IN_BUILTIN_MIC)
    PCM 16-bit : 8000, 11025, 12000, 16000, 22050, 24000, 32000, 44100, 48000 Hz
    Channels   : MONO, STEREO, FRONT_BACK
    PRIMARY active input on this device

  Built-In Back Mic (AUDIO_DEVICE_IN_BACK_MIC)
    PCM 16-bit : same rates as Built-In Mic
    Channels   : MONO, STEREO, FRONT_BACK

  Wired Headset Mic (AUDIO_DEVICE_IN_WIRED_HEADSET)
    PCM 16-bit : same rates — BSP leftover, no physical jack

  BT SCO Headset Mic (AUDIO_DEVICE_IN_BLUETOOTH_SCO_HEADSET)
    PCM 16-bit : 8000, 16000 Hz — MONO
    Call audio input via Bluetooth SCO

  AUX Digital In (AUDIO_DEVICE_IN_AUX_DIGITAL)
    PCM 16-bit : full rate set, MONO, STEREO, FRONT_BACK

  FM Tuner In (AUDIO_DEVICE_IN_FM_TUNER)
    PCM 16-bit : full rate set, MONO, STEREO, FRONT_BACK

  Voice Call In (AUDIO_DEVICE_IN_VOICE_CALL)
    PCM 16-bit : full rate set, MONO, STEREO, FRONT_BACK

  Echo Ref In (AUDIO_DEVICE_IN_ECHO_REFERENCE)
    PCM 16-bit : full rate set, MONO, STEREO, FRONT_BACK

# 7.5 PRIMARY MODULE — AUDIO ROUTES

  Source: /vendor/etc/audio_policy_configuration.xml (full confirmed)

  Sink               Sources
  ─────────────────────────────────────────────────────────────────────────────
  Earpiece           primary output, deep_buffer
  Speaker            primary output, deep_buffer
  Wired Headset      primary output, deep_buffer
  Wired Headphones   primary output, deep_buffer
  AUX Digital Out    primary output, deep_buffer
  BT SCO             primary output, deep_buffer
  BT SCO Headset     primary output, deep_buffer
  BT SCO Car Kit     primary output, deep_buffer
  Telephony Tx       incall_music_uplink
  primary input      Built-In Mic, Built-In Back Mic, Wired Headset Mic,
                     BT SCO Headset Mic, AUX Digital In, FM Tuner In,
                     Voice Call In, Echo Ref In
  voice tx           Built-In Mic, Built-In Back Mic, Wired Headset Mic,
                     BT SCO Headset Mic, AUX Digital In, FM Tuner In,
                     Voice Call In, Echo Ref In
  FM Tuner input     Built-In Mic, Built-In Back Mic, Wired Headset Mic,
                     BT SCO Headset Mic, AUX Digital In, FM Tuner In,
                     Voice Call In, Echo Ref In

  NOTE: Voice call audio (modem path) routes through the HAL's internal
        mixer, not through AudioPolicyManager routes. The primary HAL
        handles the modem voice path internally for call audio quality.

# 7.6 A2DP MODULE — LEGACY

  Source: /vendor/etc/a2dp_audio_policy_configuration.xml (full confirmed)
  Module : a2dp, halVersion 2.0

  Mix ports:
    a2dp output : no profile — codec negotiated at runtime
    a2dp input  : PCM 16-bit, 44100 Hz, STEREO

  Device ports:
    BT A2DP Out (AUDIO_DEVICE_OUT_BLUETOOTH_A2DP)
      PCM 16-bit : 44100 Hz — STEREO
    BT A2DP Headphones (AUDIO_DEVICE_OUT_BLUETOOTH_A2DP_HEADPHONES)
      PCM 16-bit : 44100 Hz — STEREO
    BT A2DP Speaker (AUDIO_DEVICE_OUT_BLUETOOTH_A2DP_SPEAKER)
      PCM 16-bit : 44100 Hz — STEREO
    BT A2DP In (AUDIO_DEVICE_IN_BLUETOOTH_A2DP)
      PCM 16-bit : 44100 Hz — STEREO

  Routes:
    BT A2DP Out      ← a2dp output
    BT A2DP Headphones ← a2dp output
    BT A2DP Speaker  ← a2dp output
    a2dp input       ← BT A2DP In

# 7.7 BLUETOOTH AUDIO HAL 2.0 MODULE

  Source: /vendor/etc/bluetooth_audio_policy_configuration.xml (full confirmed)
  Module : bluetooth, halVersion 2.0
  NOTE   : This is the newer BT Audio HAL alongside the legacy a2dp module.
           This module supports higher sample rates and Hearing Aid output.

  Mix ports:
    a2dp output     : no profile — codec negotiated at runtime
    hearing aid output : PCM 16-bit, 16000/24000 Hz — STEREO

  Device ports:
    BT A2DP Out (AUDIO_DEVICE_OUT_BLUETOOTH_A2DP)
      PCM 16-bit : 44100, 48000, 88200, 96000 Hz — STEREO
    BT A2DP Headphones (AUDIO_DEVICE_OUT_BLUETOOTH_A2DP_HEADPHONES)
      PCM 16-bit : 44100, 48000, 88200, 96000 Hz — STEREO
    BT A2DP Speaker (AUDIO_DEVICE_OUT_BLUETOOTH_A2DP_SPEAKER)
      PCM 16-bit : 44100, 48000, 88200, 96000 Hz — STEREO
    BT Hearing Aid Out (AUDIO_DEVICE_OUT_HEARING_AID)
      no profile — capability negotiated at runtime

  Routes:
    BT A2DP Out         ← a2dp output
    BT A2DP Headphones  ← a2dp output
    BT A2DP Speaker     ← a2dp output
    BT Hearing Aid Out  ← hearing aid output

  CRITICAL FOR WRIST-WORN DEVICE:
    This module enables high-quality Bluetooth audio at 88200/96000 Hz
    sample rates — above CD quality. All music and media playback to BT
    headphones routes through this module at runtime. The bluetooth HAL 2.0
    is the primary BT audio path for this device in normal use.

# 7.8 A2DP INPUT MODULE

  Source: /vendor/etc/a2dp_in_audio_policy_configuration.xml (full confirmed)
  Module : a2dp, halVersion 2.0

  Mix ports:
    a2dp input : PCM 16-bit, 44100/48000 Hz — MONO, STEREO

  Device ports:
    BT A2DP In (AUDIO_DEVICE_IN_BLUETOOTH_A2DP)
      PCM 16-bit : 44100, 48000 Hz — MONO, STEREO

  Routes:
    a2dp input ← BT A2DP In

  NOTE: A2DP input supports both mono and stereo at 44100 and 48000 Hz.
        Enables BT microphone input (e.g. BT headset recording).

# 7.9 USB AUDIO MODULE

  Source: /vendor/etc/usb_audio_policy_configuration.xml
  NOTE  : Full file content not captured in this session. USB audio HAL
          is confirmed present via xi:include in master policy config.
          ro.audio.usb.period_us=16000 (16ms USB audio period confirmed).
          Stream volume data confirms USB headset path active in OS:
          STREAM_MUSIC usb_headset volume=2 (device 0x4000000)

# 7.10 AUDIO EFFECTS

  Source: /vendor/etc/audio_effects.xml (full confirmed)
  Config version: 2.0

  LIBRARIES:
    Name                Library file
    ─────────────────────────────────────────────────────────
    bundle              libbundlewrapper.so
    reverb              libreverbwrapper.so
    visualizer          libvisualizer.so
    downmix             libdownmix.so
    loudness_enhancer   libldnhncr.so
    dynamics_processing libdynproc.so
    pre_processing      libaudiopreprocessing.so

  EFFECTS:
    Name                Library             UUID
    ─────────────────────────────────────────────────────────────────────────
    bassboost           bundle              86315140-33a4-11df-a3c2-0002a5d5c51b
    virtualizer         bundle              1d403300-33a4-11df-a5c6-0002a5d5c51b
    equalizer           bundle              ce772f20-33a4-11df-a148-0002a5d5c51b
    volume              bundle              119341a0-8469-11df-81f9-0002a5d5c51b
    reverb_env_aux      reverb              4a387fc0-8ab3-11df-8bad-0002a5d5c51b
    reverb_env_ins      reverb              c7a511a0-a3bb-11df-860e-0002a5d5c51b
    reverb_pre_aux      reverb              f29a1400-a3bb-11df-8ddc-0002a5d5c51b
    reverb_pre_ins      reverb              172cdf00-a3bc-11df-a72f-0002a5d5c51b
    visualizer          visualizer          d069d9e0-8329-11df-9168-0002a5d5c51b
    downmix             downmix             93f04452-e4fe-41cc-91f9-e475b6d1d69f
    loudness_enhancer   loudness_enhancer   fa415329-2034-4bea-b5dc-5b381c8d1e2c
    dynamics_processing dynamics_processing e0e6539b-1781-7261-676f-6d7573696340
    aec                 pre_processing      9f35ed76-0b66-4330-8f79-e39ca266dc7c
    ns                  pre_processing      300abe9f-dfc5-4340-9c4b-79ef1be4e651
    agc                 pre_processing      3387eb70-9896-4338-90f5-b2de883864c9

  PREPROCESSING — voice_communication stream:
    aec (Acoustic Echo Cancellation)
    ns  (Noise Suppression)
    agc (Automatic Gain Control)
    NOTE: All three applied to VoIP calls — critical for BT call quality
          on wrist-worn device where mic is near speaker.

  POSTPROCESSING: commented out — no active post-processing on any stream.

# 7.11 AUDIO EFFECT LIBRARIES (soundfx/)

  Location : /vendor/lib64/soundfx/ and /vendor/lib/soundfx/

  64-bit (/vendor/lib64/soundfx/):
    libaudiopreprocessing.so    37,224 bytes
    libbundlewrapper.so         78,768 bytes
    libdownmix.so               23,624 bytes
    libdynproc.so               66,528 bytes
    libeffectproxy.so           19,544 bytes
    libldnhncr.so               19,640 bytes
    libreverbwrapper.so         36,848 bytes
    libvisualizer.so            19,536 bytes

  32-bit (/vendor/lib/soundfx/):
    libaudiopreprocessing.so    24,672 bytes
    libbundlewrapper.so         71,660 bytes
    libdownmix.so               15,440 bytes
    libdynproc.so               46,560 bytes
    libeffectproxy.so           15,248 bytes
    libldnhncr.so               15,576 bytes
    libreverbwrapper.so         36,944 bytes
    libvisualizer.so            28,420 bytes

  NOTE: libeffectproxy.so present in library directory but not declared
        in audio_effects.xml — loaded on-demand by the effects framework.

# 7.12 AUDIOSERVER RUNTIME LIBRARY MAP

  Service binary : /system/bin/audioserver
  Runtime        : confirmed running (live device)

  Libraries confirmed from /proc/audioserver/maps (partial — thread
  signal stack sections dominated output; named library entries extracted):

  Source                        Library
  ──────────────────────────────────────────────────────────────────────────
  /system/bin/                  audioserver
  /system/lib64/                libstatslog.so
  /system/lib64/                libchrome.so
  /system/lib64/hw/             audio.a2dp.default.so
  /system/lib64/                libevent.so
  /system/lib64/                ashmemd_aidl_interface-cpp.so
  /system/lib64/                libashmemd_client.so
  /system/lib64/                libbesloudness.so
  /system/lib64/                android.hardware.media@1.0.so
  /system/lib64/                android.hardware.configstore@1.0.so
  /vendor/lib64/                libmtkcam_ulog.so
  /vendor/lib64/                libmtkcam_stdutils.so

  NOTE: audio.a2dp.default.so loads from /system/lib64/hw/ — AOSP A2DP
        implementation in system, not vendor. This is correct for Android 10.

  NOTE: libmtkcam_ulog.so and libmtkcam_stdutils.so appear in audioserver
        process memory. These are MTK camera utility libraries present via
        indirect dependency chain. Not functionally significant for audio.

  NOTE: libbesloudness.so — MTK BES (Better Experience Solution) Loudness
        enhancement. Loads from /system/lib64/. Active:
        persist.vendor.audiohal.besloudness_state=1

# 7.13 STREAM VOLUMES (from dumpsys audio — live device 2026-05-08)

  Stream                  Min  Max  Current                    Active device
  ──────────────────────────────────────────────────────────────────────────────
  STREAM_VOICE_CALL        1   15   speaker:5, earpiece:5        speaker
  STREAM_SYSTEM            0   15   speaker:15, default:9        speaker
  STREAM_RING              0   15   speaker:15, default:9        speaker
  STREAM_MUSIC             0   15   speaker:12, bt_a2dp:13,      speaker
                                    usb_headset:2, default:9
  STREAM_ALARM             1   15   speaker:15, default:9        speaker
  STREAM_NOTIFICATION      0   15   speaker:15, default:9        speaker
  STREAM_BLUETOOTH_SCO     0   15   speaker:7, earpiece:7        speaker
  STREAM_SYSTEM_ENFORCED   0   15   speaker:15, default:9        speaker
  STREAM_DTMF              0   15   speaker:15, default:9        speaker
  STREAM_TTS               0   15   speaker:12, bt_a2dp:13,      speaker
                                    usb_headset:2, default:9
  STREAM_ACCESSIBILITY     0   15   speaker:12, bt_a2dp:13,      speaker
                                    usb_headset:2, default:9
  STREAM_ASSISTANT         0   15   speaker:12, bt_a2dp:13,      speaker
                                    usb_headset:2, default:9
  STREAM_REROUTING         0   15   default:12                   speaker
  STREAM_PATCH             0   15   default:12                   speaker

  All streams routing to speaker at capture time — no BT connected.
  bt_a2dp volume level 13 stored (higher than speaker 12) — BT preferred
  for music when connected.

# 7.14 AUDIO PARAMETER DATABASE

  Location : /vendor/etc/audio_param/
  Total    : 99 files (AudioParam + ParamUnitDesc pairs for each category)
  Tool ver : ro.vendor.mtk_audio_tuning_tool_ver=V2.2

  Categories and files:

  Configuration root:
    AudioParamOptions.xml         4,013 bytes

  Bluetooth:
    BtInfo_AudioParam.xml           516 bytes
    BtInfo_ParamUnitDesc.xml        410 bytes

  Headphone impedance (BSP leftover — no headphone jack on this device):
    HpImpedance_AudioParam.xml    3,406 bytes
    HpImpedance_ParamUnitDesc.xml   938 bytes

  Microphone:
    MicInfo_AudioParam.xml        4,200 bytes
    MicInfo_ParamUnitDesc.xml     3,340 bytes

  Playback:
    Playback_AudioParam.xml         114 bytes
    Playback_ParamTreeView.xml    2,566 bytes
    Playback_ParamUnitDesc.xml    1,492 bytes
    PlaybackACF_AudioParam.xml    8,309 bytes     (Advanced Calibration Filter)
    PlaybackACF_ParamUnitDesc.xml 3,646 bytes
    PlaybackDRC_AudioParam.xml   15,965 bytes     (Dynamic Range Control)
    PlaybackDRC_ParamUnitDesc.xml 3,489 bytes
    PlaybackHCF_AudioParam.xml    5,520 bytes     (High-Frequency Compensation)
    PlaybackHCF_ParamUnitDesc.xml 2,835 bytes
    PlaybackVolAna_AudioParam.xml 2,934 bytes     (Analog volume)
    PlaybackVolAna_ParamUnitDesc.xml 3,037 bytes
    PlaybackVolDigi_AudioParam.xml 6,148 bytes    (Digital volume)
    PlaybackVolDigi_ParamUnitDesc.xml 1,428 bytes
    PlaybackVolUI_AudioParam.xml  4,255 bytes     (UI volume steps)
    PlaybackVolUI_ParamUnitDesc.xml 1,944 bytes

  Recording:
    Record_AudioParam.xml        30,383 bytes
    Record_ParamTreeView.xml      2,117 bytes
    Record_ParamUnitDesc.xml     12,933 bytes
    RecordDMNR_AudioParam.xml       998 bytes     (Dual Mic Noise Reduction)
    RecordDMNR_ParamUnitDesc.xml    490 bytes
    RecordFIR_AudioParam.xml      1,359 bytes     (FIR filter coefficients)
    RecordFIR_ParamUnitDesc.xml   2,052 bytes
    RecordUI_AudioParam.xml       5,916 bytes
    RecordUI_ParamUnitDesc.xml    2,134 bytes
    RecordVol_AudioParam.xml      3,368 bytes
    RecordVol_ParamUnitDesc.xml   1,516 bytes
    RecordVolUI_AudioParam.xml    2,589 bytes
    RecordVolUI_ParamUnitDesc.xml 1,648 bytes

  Ringback:
    RingbackVol_AudioParam.xml      698 bytes
    RingbackVol_ParamUnitDesc.xml 1,006 bytes
    RingbackVolUI_AudioParam.xml    612 bytes
    RingbackVolUI_ParamUnitDesc.xml 951 bytes

  Sound enhancement:
    SoundEnhancement_AudioParam.xml    319 bytes
    SoundEnhancement_ParamUnitDesc.xml 324 bytes

  Speech (cellular voice call):
    Speech_AudioParam.xml        46,795 bytes     (largest speech param file)
    Speech_ParamTreeView.xml     13,513 bytes
    Speech_ParamUnitDesc.xml     28,967 bytes
    SpeechDMNR_AudioParam.xml     3,144 bytes     (DMNR for voice calls)
    SpeechDMNR_ParamUnitDesc.xml 25,978 bytes
    SpeechEchoRef_AudioParam.xml    325 bytes
    SpeechEchoRef_ParamUnitDesc.xml 321 bytes
    SpeechGeneral_AudioParam.xml    440 bytes
    SpeechGeneral_ParamUnitDesc.xml 580 bytes
    SpeechMagiClarity_AudioParam.xml    489 bytes (MTK MagiClarity enhancement)
    SpeechMagiClarity_ParamUnitDesc.xml 336 bytes
    SpeechNetwork_AudioParam.xml    322 bytes
    SpeechNetwork_ParamUnitDesc.xml 328 bytes
    SpeechUI_AudioParam.xml       3,696 bytes
    SpeechUI_ParamUnitDesc.xml    3,433 bytes
    SpeechVol_AudioParam.xml      7,280 bytes
    SpeechVol_ParamUnitDesc.xml   1,966 bytes
    SpeechVolUI_AudioParam.xml    1,729 bytes
    SpeechVolUI_ParamUnitDesc.xml 1,170 bytes

  USB call and device:
    USBCall_AudioParam.xml          896 bytes
    USBCall_ParamUnitDesc.xml       523 bytes
    USBDevice_AudioParam.xml      1,253 bytes
    USBDevice_ParamUnitDesc.xml     748 bytes

  Uplink configuration:
    UplinkConfiguration_AudioParam.xml    745 bytes
    UplinkConfiguration_ParamUnitDesc.xml 612 bytes

  Voice on Wakeup:
    VOW_AudioParam.xml            1,638 bytes
    VOW_ParamUnitDesc.xml         1,177 bytes

  VoIP (original):
    VoIP_AudioParam.xml           7,058 bytes
    VoIP_ParamTreeView.xml        4,684 bytes
    VoIP_ParamUnitDesc.xml       15,487 bytes
    VoIPDMNR_AudioParam.xml       1,062 bytes
    VoIPDMNR_ParamUnitDesc.xml      730 bytes
    VoIPGeneral_AudioParam.xml      333 bytes
    VoIPGeneral_ParamUnitDesc.xml   342 bytes
    VoIPUI_AudioParam.xml         1,966 bytes
    VoIPUI_ParamUnitDesc.xml      1,611 bytes
    VoIPVol_AudioParam.xml        1,173 bytes
    VoIPVol_ParamUnitDesc.xml     1,286 bytes
    VoIPVolUI_AudioParam.xml        943 bytes
    VoIPVolUI_ParamUnitDesc.xml   1,088 bytes

  VoIP v2 (enhanced):
    VoIPv2_AudioParam.xml        33,240 bytes
    VoIPv2_ParamTreeView.xml     37,571 bytes
    VoIPv2_ParamUnitDesc.xml     46,806 bytes     (largest single file)
    VoIPv2DMNR_AudioParam.xml     2,488 bytes
    VoIPv2DMNR_ParamUnitDesc.xml 162,277 bytes    (second largest)
    VoIPv2General_AudioParam.xml    440 bytes
    VoIPv2General_ParamUnitDesc.xml 778 bytes
    VoIPv2UI_AudioParam.xml       3,543 bytes
    VoIPv2UI_ParamUnitDesc.xml    2,329 bytes

  Volume gain maps:
    Volume_AudioParam.xml        17,714 bytes
    Volume_ParamUnitDesc.xml      3,442 bytes
    VolumeGainMap_AudioParam.xml  7,691 bytes
    VolumeGainMap_ParamUnitDesc.xml 2,779 bytes
    VolumeGainMapUL_AudioParam.xml  1,965 bytes  (uplink gain map)
    VolumeGainMapUL_ParamUnitDesc.xml 2,126 bytes

# 7.15 AUDIO PATHS — WRIST-WORN DEVICE SUMMARY

  This device has no 3.5mm jack. All audio paths in practice:

  SPEAKER PATH (primary, always present):
    AudioFlinger → primary output / deep_buffer → audio.primary.mt6765.so
    → MT6357 PMIC audio codec → speaker amplifier → speaker driver
    Formats: PCM 32/16-bit, 44100/48000 Hz, stereo
    BES Loudness enhancement active (besloudness_state=1)

  BT SCO PATH (voice calls via Bluetooth):
    AudioFlinger → primary output → audio.primary.mt6765.so
    → BT SCO hardware path → BT controller → remote device
    Format: PCM 16-bit, 8000 Hz (NB) or 16000 Hz (WB) — MONO
    BT SCO mic in: 8000/16000 Hz MONO
    AEC + NS + AGC preprocessing applied to voice_communication stream

  BT A2DP PATH (music/media via Bluetooth):
    AudioFlinger → a2dp output → bluetooth HAL 2.0 or legacy a2dp HAL
    → BT controller (wlan_drv_gen4m.ko / bt_drv.ko) → remote device
    Formats: PCM 16-bit, 44100/48000/88200/96000 Hz, stereo
    Codec negotiated at runtime (SBC, AAC, etc.)
    BT A2DP volume level=13 stored (higher than speaker level=12)

  USB AUDIO PATH:
    AudioFlinger → USB HAL → USB device
    Period: 16ms (ro.audio.usb.period_us=16000)
    usb_headset volume stored=2 (low default)

  CELLULAR VOICE CALL PATH:
    Modem → Telephony Tx → audio.primary.mt6765.so internal voice path
    → Speaker or BT SCO (routed by HAL internally)
    HAL handles voice routing internally — not through AudioPolicyManager

  NOTE: Earpiece, Wired Headset, and Wired Headphone ports are declared
        in the policy config as BSP leftovers. No corresponding physical
        hardware exists on this 2.4-inch watch form factor. Android
        framework is aware of them as declarative ports but they will
        never be selected by AudioPolicyManager in normal use.

# 7.16 ROM BUILD REQUIREMENTS

  All files in /vendor/etc/audio_param/ — carry complete (99 files)
  /vendor/etc/audio_policy_configuration.xml — carry as-is
  /vendor/etc/audio_effects.xml — carry as-is
  /vendor/etc/a2dp_audio_policy_configuration.xml — carry as-is
  /vendor/etc/bluetooth_audio_policy_configuration.xml — carry as-is
  /vendor/etc/a2dp_in_audio_policy_configuration.xml — carry as-is
  /vendor/etc/usb_audio_policy_configuration.xml — carry as-is
  /vendor/etc/r_submix_audio_policy_configuration.xml — carry as-is
  /vendor/etc/audio_policy_volumes.xml — carry as-is
  /vendor/etc/default_volume_tables.xml — carry as-is
  /vendor/etc/audio_em.xml — carry as-is
  /vendor/etc/audio_param/ (entire directory) — carry as-is
  /vendor/etc/aurisys_config.xml — carry as-is
  /vendor/etc/aurisys_param/ (entire directory) — carry as-is
  /vendor/lib64/soundfx/ (all 8 .so files) — carry as-is
  /vendor/lib/soundfx/ (all 8 .so files) — carry as-is
  /vendor/lib64/hw/audio.primary.mt6765.so — carry as-is
  /vendor/lib64/hw/audio.primary.default.so — carry as-is
  /vendor/lib64/hw/audio.bluetooth.default.so — carry as-is
  /vendor/lib64/hw/audio.r_submix.default.so — carry as-is
  /vendor/lib64/hw/audio.r_submix.mt6765.so — carry as-is
  /vendor/lib64/hw/audio.usb.default.so — carry as-is
  /vendor/lib64/hw/audio.usb.mt6765.so — carry as-is

  BES Loudness: persist.vendor.audiohal.besloudness_state=1
    Must be set in product/system properties for ROM or speaker output
    will be noticeably quieter than stock.

  BT A2DP standby time: persist.vendor.bluetooth.a2dpstandbytime=500
    Carry into ROM properties — controls BT A2DP standby timeout in ms.
################################################################################
# SECTION 8 — RADIO & MODEM STACK
################################################################################

# 8.1 RIL ARCHITECTURE

  RIL daemon      : mtkfusionrild
  Binary path     : /vendor/bin/hw/mtkfusionrild
  Binary size     : 20,168 bytes
  RIL library     : vendor.rild.libpath=mtk-ril.so (confirmed live)
  RIL args        : vendor.rild.libargs=-d /dev/ttyC0 (confirmed live)
  RIL mode        : ro.vendor.mtk_ril_mode=c6m_1rild (single RILD)
  RIL impl        : gsm.version.ril-impl=android reference-ril 1.0
  SIM config      : persist.radio.multisim.config=ss (single SIM)
  Max multisim    : ro.vendor.radio.max.multisim=ss

  Service name    : init.svc.vendor.ril-daemon-mtk=running (confirmed live)
  Boot time       : ro.boottime.vendor.ril-daemon-mtk=18,161,139,230 ns
                    (~18.2 seconds from boot zero — last radio service to start)

  Active modem    : vendor.ril.active.md=10 (modem 1 active)
  MTK RIL flag    : vendor.ril.mtk=1

# 8.2 MODEM IDENTIFICATION

  Baseband        : gsm.version.baseband=MOLY.LR12A.R3.MP.V107.4 (confirmed live)
  Load verno      : ro.vendor.md_apps.load_verno=MOLY.LR12A.R3.MP.V107.4
  Project config  : vendor.gsm.project.baseband=TK_MD_BASIC(LWTG_6177M_R3_6762)
  Single bin      : ro.vendor.mtk_single_bin_modem_support=1
  Modem variant   : LWTG (LTE + WCDMA + TDS + GSM — confirmed)
  RAT config      : persist.vendor.radio.mtk_ps2_rat=L/W/G
                    persist.vendor.radio.mtk_ps3_rat=G
  C2K LTE mode    : ro.vendor.mtk_c2k_lte_mode=0 (CDMA2000 disabled)
  LTE support     : ro.vendor.mtk_lte_support=1
  Default network : ro.telephony.default_network=9,9,9,9 (LTE preferred)
  Radio access    : persist.vendor.radio.raf1=213002 (radio access family bitmap)
  World mode      : vendor.ril.nw.worldmode.activemode=1

# 8.3 SIM STATE (live device — 2026-05-08/09)

  SIM count       : ro.telephony.sim.count=1
  SIM state       : gsm.sim.state=LOADED (confirmed live)
  UICC type       : vendor.gsm.ril.uicctype=USIM
  Full UICC type  : vendor.gsm.ril.fulluicctype=USIM
  Phone book      : vendor.gsm.sim.ril.phbready=true
  Test SIM        : vendor.gsm.sim.ril.testsim=0

  Operator:
    Name          : gsm.sim.operator.alpha=T-Mobile
                    vendor.gsm.sim.operator.default-name=T-Mobile
    MCC/MNC       : gsm.sim.operator.numeric=310260
                    vendor.ril.data.gsm_mcc_mnc0=310260
                    vendor.gsm.ril.uicc.mccmnc=310260
    Country       : gsm.sim.operator.iso-country=us
    Roaming       : gsm.operator.isroaming=false

  ICCID (confirmed):
    vendor.ril.iccid.sim1=8901260220751115411F
    gsm.sim.preiccid_0=8901260 (pre-ICCID prefix T-Mobile US)
    CFU ICCID (base64): ODkwMTI2MDIyMDc1MTExNTQxMUY=
    Hashed last ICCID: 7f562cd58bb777374ef5918d86439501

  IMSI (confirmed): vendor.ril.data.gsm_imsi0=310260225xxxxxx
    MCC=310 (USA), MNC=260 (T-Mobile)
    Remaining digits redacted for privacy — full value confirmed present

  PIN state       : gsm.sim.num.pin=3 (3 PIN attempts remaining)
                    gsm.sim.num.pin2=10, puk=10, puk2=10
  IMSI status     : vendor.ril.imsi.status.sim1=1

  Serial number   : vendor.gsm.serial=MTK000250730160840000565

# 8.4 NETWORK STATE (live device — 2026-05-09 capture)

  Active network type : gsm.network.type=IWLAN
    NOTE: Device was on WiFi Calling (IWLAN) at capture time.
          LTE signal was -119 dBm (weak) — automatic handoff to IWLAN.
          This confirms WiFi Calling is fully functional on this device.

  LTE signal      : vendor.ril.nw.signalstrength.lte.1=-119,3
                    (-119 dBm RSRP, SNR=3 — marginal LTE, hence IWLAN)
  Current phone   : gsm.current.phone-type=1 (GSM)
  Data PDN        : vendor.ril.data.pdn_info0=default_2
  PDP active      : gsm.defaultpdpcontext.active=false
  Data roaming    : net.lte.data.roaming.setting1=true
  Preferred data  : vendor.ril.data.preferred_data_mode=1
  Smart data      : persist.vendor.radio.smart.data.switch=1
  Data type       : persist.vendor.radio.mtk_data_type=0
  SSC mode        : vendor.ril.data.ssc.mode=3
  IMS PDN         : vendor.ims.eims.pdn.info=ims_0_0,
  DSBP support    : persist.vendor.radio.mtk_dsbp_support=1
  Allow data slot : vendor.ril.data.allow_data_slot=0

  Fast dormancy:
    fd mode       : vendor.ril.fd.mode=1 (enabled)
    fd counter    : persist.vendor.radio.fd.counter=150
    fd r8 counter : persist.vendor.radio.fd.r8.counter=150
    fd off counter: persist.vendor.radio.fd.off.counter=50

  Emergency numbers: ril.ecclist=911,112

# 8.5 VOLTE & IMS STACK

  All IMS services confirmed RUNNING on live device:

  Service                     State    Boot time (ns)
  ────────────────────────────────────────────────────────────────────────
  init.svc.volte_imsm_93      running  17,875,696,461
  init.svc.vendor.volte_stack running  17,881,792,923
  init.svc.vendor.volte_ua    running  17,887,694,615
  init.svc.vendor.volte_imcb  running  17,917,903,846
  init.svc.volte_md_status    running  12,475,566,076  ← starts at 12.5s
  ────────────────────────────────────────────────────────────────────────

  VoLTE state     : vendor.volte_md_status=ready
  VoLTE PID       : vendor.net.ims.volte.pid=890

  VoLTE capability flags (all confirmed live):
    persist.vendor.volte_support=1
    persist.vendor.mtk.volte.enable=1
    persist.vendor.ims_support=1
    persist.vendor.mims_support=2
    persist.vendor.radio.volte_state=1
    ro.vendor.md_auto_setup_ims=1
    persist.vendor.mtk_dynamic_ims_switch=1
    persist.vendor.mtk_ct_volte_support=1      (China Telecom VoLTE support)

  Video over LTE (ViLTE):
    persist.vendor.vilte_support=1
    persist.vendor.mtk.vilte.enable=1

  WiFi Calling (WFC/IWLAN):
    persist.vendor.radio.wfc_state=1
    ro.telephony.iwlan_operation_mode=default
    vendor.net.ims.ipsec.version=2.0
    gsm.network.type=IWLAN (ACTIVE at capture — confirmed functional)

  SIM supplementary services:
    persist.vendor.radio.terminal-based.clir_m=4
    persist.vendor.radio.terminal-based.clir_n=0
    persist.vendor.radio.terminal-based.cw=disabled_tbcw
    persist.vendor.radio.cfu_over_ims=0

# 8.6 CCCI DEVICE NODES — COMPLETE INVENTORY

  CCCI = Cross-Chip Communication Interface — modem kernel driver
  Major 239 = CCCI character device driver
  Major 240 = CCCI monitor
  Major 250 = CCCI status devices
  All nodes: crw-rw---- radio:radio (audio nodes: audio:audio)
  All created: 2026-05-07 21:03 (boot time)

  Node                  Major  Minor  Owner        Purpose
  ─────────────────────────────────────────────────────────────────────────
  /dev/ccci_aud         239    1      audio:audio  Modem audio channel
  /dev/ccci_bip         239    27     radio        BIP (Bearer Independent Protocol)
  /dev/ccci_c2k_agps    239    25     radio        C2K AGPS channel
  /dev/ccci_c2k_ppp     239    24     radio        C2K PPP data channel
  /dev/ccci_ccb_ctrl    239    154    radio        CCB control channel
  /dev/ccci_ccb_dhl     239    151    radio        CCB data high-level
  /dev/ccci_ccb_md_monitor 239 152    radio        CCB modem monitor
  /dev/ccci_ccb_meta    239    153    radio        CCB meta channel
  /dev/ccci_fs          239    4      radio        Modem filesystem channel
  /dev/ccci_imsa        239    10     radio        IMS A channel
  /dev/ccci_imsc        239    9      radio        IMS C channel
  /dev/ccci_imsdc       239    11     radio        IMS DC channel
  /dev/ccci_imsem       239    21     radio        IMS EM channel
  /dev/ccci_imsm        239    22     radio        IMS M channel
  /dev/ccci_imsv        239    8      radio        IMS V (video) channel
  /dev/ccci_ioctl0      239    12     radio        IOCTL channel 0
  /dev/ccci_ioctl1      239    13     radio        IOCTL channel 1
  /dev/ccci_ioctl2      239    14     radio        IOCTL channel 2
  /dev/ccci_ioctl3      239    15     radio        IOCTL channel 3
  /dev/ccci_ioctl4      239    16     radio        IOCTL channel 4
  /dev/ccci_ipc_1220_0  239    100    radio        IPC port 1220 ch 0
  /dev/ccci_ipc_2       239    102    radio        IPC port 2
  /dev/ccci_ipc_4       239    104    radio        IPC port 4
  /dev/ccci_ipc_5       239    105    radio        IPC port 5
  /dev/ccci_ipc_9       239    109    radio        IPC port 9
  /dev/ccci_it          239    17     radio        IT (internal test) channel
  /dev/ccci_lb_it       239    18     radio        Loopback IT channel
  /dev/ccci_md1_sta     250    1      radio        Modem 1 status
  /dev/ccci_md2_sta     250    2      radio        Modem 2 status
  /dev/ccci_md3_sta     250    3      radio        Modem 3 status
  /dev/ccci_md_log_ctrl 239    2      radio        Modem log control
  /dev/ccci_mdl_monitor 239    19     radio        Modem log monitor
  /dev/ccci_mdx_sta     250    0      radio        Modem X status
  /dev/ccci_monitor     240    0      radio        CCCI monitor
  /dev/ccci_raw_audio   239    159    audio:audio  Raw audio channel
  /dev/ccci_raw_dhl     239    155    radio        Raw DHL channel
  /dev/ccci_raw_mdm     239    156    radio        Raw modem channel
  /dev/ccci_raw_netd    239    157    radio        Raw netd channel
  /dev/ccci_raw_usb     239    158    radio        Raw USB channel
  /dev/ccci_rpc         239    20     radio        RPC channel
  /dev/ccci_ss_xcap     239    26     radio        SS XCAP channel
  /dev/ccci_woa         239    23     radio        WOA (Wake on Alarm) channel

  TOTAL: 42 CCCI nodes

  IMS-specific CCCI channels (critical for VoLTE):
    ccci_imsa  — IMS A: general IMS application channel
    ccci_imsc  — IMS C: IMS call channel
    ccci_imsdc — IMS DC: IMS data channel
    ccci_imsem — IMS EM: IMS emergency channel
    ccci_imsm  — IMS M: IMS management channel
    ccci_imsv  — IMS V: IMS video channel (ViLTE)

  ROM BUILD NOTE: All 42 CCCI nodes are created by the CCCI kernel driver
    at boot. These are kernel-side. No ROM action needed beyond ensuring
    the ccci_fsd, ccci_mdinit, and ccci_rpcd services start correctly.

# 8.7 TTY MODEM CHANNELS

  All crw-rw---- radio:radio, major 239

  Node          Minor  Purpose
  ─────────────────────────────────────────────────────────────────────
  /dev/ttyC0    3      RIL primary AT channel (vendor.rild.libargs=-d /dev/ttyC0)
  /dev/ttyC1    7      AT command channel 1
  /dev/ttyC2    5      AT command channel 2
  /dev/ttyC3    6      AT command channel 3
  /dev/ttyC5    28     Extended AT channel 5
  /dev/ttyC6    29     Extended AT channel 6

# 8.8 MODEM SERVICES — ALL CONFIRMED RUNNING

  Service                  State    Boot time (ns)      Purpose
  ──────────────────────────────────────────────────────────────────────────────
  ccci_mdinit              running  15,698,796,153      Modem 1 init
  ccci3_mdinit             stopped  15,703,258,846      Modem 3 init (ss=unused)
  vendor.ccci_fsd          running  15,686,354,923      CCCI filesystem daemon
  vendor.ccci3_fsd         running  15,690,339,769      CCCI3 filesystem daemon
  vendor.ccci_rpcd         running  15,712,387,846      CCCI RPC daemon
  vendor.gsm0710muxd       running  17,957,765,538      GSM 07.10 UART mux
  vendor.ril-daemon-mtk    running  18,161,139,230      MTK RIL daemon (last up)
  volte_md_status          running  12,475,566,076      VoLTE modem status
  vendor.volte_stack       running  17,881,792,923      VoLTE stack
  vendor.volte_ua          running  17,887,694,615      VoLTE user agent
  vendor.volte_imcb        running  17,917,903,846      VoLTE IMS CB
  volte_imsm_93            running  17,875,696,461      VoLTE IMS manager
  ──────────────────────────────────────────────────────────────────────────────

  gsm0710muxd arguments: -s /dev/ttyC0 -f 512 -n 8 -m basic
    -s /dev/ttyC0  : primary modem UART
    -f 512         : frame size 512 bytes
    -n 8           : 8 virtual channels
    -m basic       : basic framing mode (not UIH)

# 8.9 RIL LIBRARY INVENTORY

  Core RIL:
    libmtk-ril.so           4,131,264 bytes  Primary MTK RIL implementation
    libmtk-rilop.so           341,040 bytes  Operator extension library
    librilfusion.so           845,416 bytes  MTK fusion RIL layer
    libril.so                 325,624 bytes  AOSP RIL base
    libreference-ril.so       104,512 bytes  Reference RIL
    libmtkrilutils.so          74,512 bytes  MTK RIL utilities
    librilutils.so             19,160 bytes  AOSP RIL utilities
    libmtkrillog.so            15,064 bytes  MTK RIL logging
    libccci_util.so            56,800 bytes  CCCI utility library

  Radio feature support:
    libfeature_rss.so          20,208 bytes  Radio Selective Switching

  MTK Radio HIDL — all versions (confirmed in lshal):
    vendor.mediatek.hardware.mtkradioex@1.0.so   2,634,024 bytes
    vendor.mediatek.hardware.mtkradioex@1.1.so     891,800 bytes
    vendor.mediatek.hardware.mtkradioex@1.2.so     910,728 bytes
    vendor.mediatek.hardware.mtkradioex@1.3.so     776,816 bytes
    vendor.mediatek.hardware.mtkradioex@1.4.so     702,544 bytes
    vendor.mediatek.hardware.mtkradioex@1.5.so     970,960 bytes

    NOTE: All 6 versions loaded simultaneously at runtime. None removable.
          Confirmed in lshal — IMtkRadioEx registered for all 6 versions.

  Radio OP HIDL — all versions:
    vendor.mediatek.hardware.radio_op@1.1.so       552,992 bytes
    vendor.mediatek.hardware.radio_op@1.2.so       249,848 bytes
    vendor.mediatek.hardware.radio_op@1.3.so       344,848 bytes
    vendor.mediatek.hardware.radio_op@1.4.so       283,888 bytes
    vendor.mediatek.hardware.radio_op@2.0.so       642,952 bytes

    NOTE: All 5 versions required. radio_op@2.0 registered for
          OpImsRILd1 and slot1 instances in lshal.

  IPSec/WiFi Calling libraries:
    libcharon-ss.so           739,544 bytes  Strongswan IKEv2 charon daemon
    libcrypto-ss.so         1,787,984 bytes  Strongswan OpenSSL crypto
    libssl-ss.so              447,936 bytes  Strongswan SSL
    libcurl-ss.so             280,312 bytes  Strongswan cURL
    libipsec_ims_shr.so        36,344 bytes  IMS IPSec shared library

# 8.10 WIFI CALLING — IPSEC/EPDG STACK

  Status at capture: ACTIVE — gsm.network.type=IWLAN confirmed
  IMS IPSec version: vendor.net.ims.ipsec.version=2.0
  WFC state        : persist.vendor.radio.wfc_state=1

  Implementation:
    Strongswan IKEv2 via charon daemon
    Binary: /vendor/bin/charon (14,240 bytes)
    Binary: /vendor/bin/stroke (18,408 bytes) — stroke control interface
    Binary: /vendor/bin/epdg_wod (137,200 bytes) — ePDG watchdog daemon
    Init  : init.wod.rc

  IPSec configuration:
    /vendor/etc/ipsec/ipsec.conf         218 bytes
    /vendor/etc/ipsec/strongswan.conf   1,282 bytes
    /vendor/etc/ipsec/ssl/ directory
    /vendor/etc/ipsec/ipsec.d/cacerts/  13 CA certificates

  CA CERTIFICATES (complete list — 13 files):
    DigiCertGlobalRootCA.crt              947 bytes
    DigiCertGlobalRootG2.crt              914 bytes
    DigiCertGlobalRootG3.crt              579 bytes
    Entrust.net_Certification_Authority_2048.cer   1,500 bytes
    GeoTrust_PCA_G3_Root.der            1,026 bytes
    GeoTrust_Primary_CA.der               896 bytes
    GeoTrust_Primary_CA_G2_ECC.der        690 bytes
    Root-R3.cer                           867 bytes
    Root_CA_1003.crt                    1,586 bytes
    VeriSignClass3G4.der                  683 bytes
    VeriSignClass3G5.der                1,239 bytes
    VeriSignUniversalRootCertification.der 1,213 bytes
    gold.cer                            1,476 bytes
    thawte.der                          1,060 bytes

    NOTE: The Strongswan libraries use the -ss suffix indicating they are
          statically linked variants compiled independently from the system
          OpenSSL to avoid version conflicts with system crypto.
          Whether these are stock Strongswan or ODM-modified requires
          binary analysis to confirm. Carry as-is — do not substitute.

  EPDG (Evolved Packet Data Gateway) daemon:
    epdg_wod (ePDG watchdog) — 137,200 bytes
    Monitors ePDG tunnel health and handles reconnection
    Managed by init.wod.rc

# 8.11 CARRIER CONFIG

  No external carrier config XMLs present in /vendor/etc/.
  Configuration compiled into libcarrierconfig.so in system partition.

  ECC list: /vendor/etc/ecc_list.xml (17,915 bytes)
  ECC from live device: ril.ecclist=911,112
  Operator ECC XMLs:
    ecc_list_OP01.xml, ecc_list_OP02.xml, ecc_list_OP09.xml,
    ecc_list_OP12.xml, ecc_list_OP17.xml, ecc_list_OP18.xml,
    ecc_list_OP20.xml

# 8.12 AGPS / SUPL CONFIGURATION

  Source: /vendor/etc/slp_conf (confirmed live)
    SUPL_ADDR=supl.google.com
    SUPL_PORT=7276
    SUPL_TLS=0
    SUPL_VERSION=2

  AGPS daemon: /vendor/bin/mtk_agpsd (2,141,904 bytes)
  AGPS config : /vendor/etc/agps_profiles_conf2.xml (27,012 bytes)
  LBS HIDL    : /vendor/bin/lbs_hidl_service (11,384 bytes)

# 8.13 SUPPLEMENTARY SERVICES

  CFU (Call Forwarding Unconditional):
    persist.vendor.radio.cfu.change.0=1
    persist.vendor.radio.cfu.iccid.0=ODkwMTI2MDIyMDc1MTExNTQxMUY= (ICCID)
    persist.vendor.radio.cfu_over_ims=0

  CLIR (Calling Line Identity Restriction):
    persist.vendor.radio.terminal-based.clir_m=4
    persist.vendor.radio.terminal-based.clir_n=0

  Call waiting:
    persist.vendor.radio.terminal-based.cw=disabled_tbcw

  SIM lock:
    vendor.gsm.sim.slot.lock.state=1
    vendor.gsm.sim.slot.lock.policy=0
    ro.vendor.sim_me_lock_mode=0

  GID1 (Group Identifier):
    vendor.ril.data.gid1-0=544DFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF

# 8.14 ROM BUILD REQUIREMENTS

  All radio services must start in correct order:
    volte_md_status (earliest — 12.5s)
    ccci services (15.7s)
    gsm0710muxd (17.9s)
    ril-daemon-mtk (18.2s — last)

  Required vendor binaries:
    mtkfusionrild, ccci_fsd, ccci_mdinit, ccci_rpcd
    gsm0710muxd, mtk_agpsd, lbs_hidl_service
    volte_md_status, volte_imsm_93, volte_imcb, volte_stack, volte_ua
    charon, stroke, epdg_wod
    netdagent, netdc, muxreport

  Required vendor libraries — ALL must be carried:
    libmtk-ril.so, libmtk-rilop.so, librilfusion.so
    libril.so, libreference-ril.so
    libmtkrilutils.so, librilutils.so, libmtkrillog.so
    libccci_util.so, libfeature_rss.so
    All 6 mtkradioex versions (@1.0 through @1.5)
    All 5 radio_op versions (@1.1 through @2.0)
    libcharon-ss.so, libcrypto-ss.so, libssl-ss.so
    libcurl-ss.so, libipsec_ims_shr.so

  Required vendor configs — ALL must be carried:
    /vendor/etc/ipsec/ (complete directory including all 13 CA certs)
    /vendor/etc/ecc_list.xml and all ecc_list_OP*.xml
    /vendor/etc/agps_profiles_conf2.xml
    /vendor/etc/slp_conf
    /vendor/etc/vendor-apns-conf.xml
    /vendor/etc/smsdbvisitor.xml
    /vendor/etc/special_pws_channel.xml
    /vendor/etc/spn-conf-op09.xml
    /vendor/etc/virtual-spn-conf-by-*.xml (4 files)
    /vendor/etc/permissions/android.hardware.telephony.gsm.xml
    /vendor/etc/permissions/android.hardware.telephony.ims.xml

  VoLTE properties required in ROM build:
    persist.vendor.volte_support=1
    persist.vendor.mtk.volte.enable=1
    persist.vendor.vilte_support=1
    persist.vendor.mtk.vilte.enable=1
    persist.vendor.ims_support=1
    persist.vendor.radio.wfc_state=1
    ro.vendor.md_auto_setup_ims=1

  Fast dormancy properties:
    persist.vendor.radio.fd.counter=150
    persist.vendor.radio.fd.r8.counter=150
    persist.vendor.radio.fd.off.counter=50
    persist.vendor.radio.fd.off.r8.counter=50
################################################################################
# SECTION 9 — CONNECTIVITY STACK
################################################################################
#
# PURPOSE:
#   Documents the complete WiFi, Bluetooth, GPS/GNSS, and USB connectivity
#   stack for the C17S. All subsystems route through a single MediaTek combo
#   chip controlled by the wmt_drv kernel module. All values verified from
#   live device ADB session 2026-05-10.
#
# AUTHORITATIVE SOURCES:
#   /proc/modules — live kernel module state
#   /vendor/lib/modules/ — module file inventory
#   /vendor/firmware/ — firmware binary inventory
#   /vendor/firmware/WMT_SOC.cfg — WMT chip configuration
#   /vendor/firmware/BT_FW.cfg — Bluetooth firmware configuration
#   /proc/PID/maps — runtime library maps for all connectivity services
#   getprop — live property values
#   ip link — live interface state
#   /config/usb_gadget/g1/ — USB gadget hardware registers
#   ps -A — live process inventory
#
################################################################################

# 9.1 KERNEL MODULES

  Location    : /vendor/lib/modules/
  Total files : 7
  Loaded      : 6 (met.ko present on disk, not loaded at runtime)
  All modules : out-of-tree (O flag), except fpsgo.ko (PO — platform out-of-tree)

  MODULE INVENTORY — FILE SIZES AND LOADED SIZES:

  Module               File size    Loaded size   Flags  Loaded
  ─────────────────────────────────────────────────────────────
  wmt_drv.ko           2,133,064    1,224,704     (O)    YES
  wlan_drv_gen4m.ko    5,158,760    1,994,752     (O)    YES
  wmt_chrdev_wifi.ko      57,720       28,672     (O)    YES
  gps_drv.ko              72,192       65,536     (O)    YES
  bt_drv.ko               52,216       28,672     (O)    YES
  fpsgo.ko                34,608      811,008     (PO)   YES
  met.ko                 901,856          —       (O)    NO — dormant
  ─────────────────────────────────────────────────────────────

  DEPENDENCY CHAIN (confirmed from /proc/modules use_count field):

    wmt_drv.ko — parent module, 5 dependents:
      wlan_drv_gen4m, wmt_chrdev_wifi, gps_drv, bt_drv
      (all four depend directly on wmt_drv)
    wmt_chrdev_wifi.ko — also depends on wlan_drv_gen4m (1 additional dependency)
    fpsgo.ko — independent, no dependencies
    met.ko — MTK Embedded Trace (MET) profiling module. On disk, never loaded
              at runtime. Dormant BSP inclusion.

  LOAD ORDER (init-enforced):
    Stage 1 — on boot trigger:
      insmod /vendor/lib/modules/wmt_drv.ko
      (init.wmt_drv.rc — fires on boot, before vendor.connsys.driver.ready)

    Stage 2 — on property:vendor.connsys.driver.ready=yes:
      insmod /vendor/lib/modules/bt_drv.ko
      insmod /vendor/lib/modules/gps_drv.ko
      insmod /vendor/lib/modules/wmt_chrdev_wifi.ko
      insmod /vendor/lib/modules/wlan_drv_${ro.vendor.wlan.gen}.ko
        → resolves to wlan_drv_gen4m.ko (ro.vendor.wlan.gen=gen4m confirmed)
      start wlan_assistant

    Stage 3 — on boot trigger (independent):
      insmod /vendor/lib/modules/fpsgo.ko
      (init.mt6762.rc — not connectivity related, documented here for completeness)

  KERNEL THREADS (connectivity — confirmed from ps -A):
    [btif_rxd]          PID 197  — Bluetooth HCI RX data thread
    [mtk_stp_btm]       PID 662  — WMT STP Bluetooth management thread
    [mtk_wmtd]          PID 665  — WMT daemon kernel thread (primary)
    [mtk_wmtd_worker]   PID 668  — WMT daemon worker thread

# 9.2 WMT COMBO CHIP ARCHITECTURE

  All three wireless subsystems — WiFi, Bluetooth, and GPS — are integrated
  into a single MediaTek combo chip controlled by wmt_drv.ko. The chip
  presents three logical interfaces to the OS through separate kernel drivers
  that all depend on the parent WMT driver.

  Combo chip generation : gen4m (MT6625L/MT6631 gen4 combo variant)
  Chip ID property      : vendor.connsys.chipid — NOT SET as live property
                          The 0x6765 value documented in the hardware database
                          is sourced from the WMT SoC config, not a runtime
                          property. No live property exposes this value.
  Driver ready property : vendor.connsys.driver.ready=yes (confirmed live)
  Coredump mode         : persist.vendor.connsys.coredump.mode — NOT SET
                          (stp_dump service disabled — no active coredump)
  Firmware path         : /vendor/firmware/ (firmware_class.path in cmdline)

  WMT SOC CONFIGURATION (/vendor/firmware/WMT_SOC.cfg — 121 bytes, full content):

    coex_wmt_ant_mode=1
    wmt_gps_lna_pin=0
    wmt_gps_lna_enable=0
    co_clock_flag=1
    bt_tssi_from_wifi=3
    bt_tssi_target=3776

  FIELD DECODE:
    coex_wmt_ant_mode=1     : WiFi and Bluetooth share a single antenna
                              (antenna coexistence mode 1 — shared)
    wmt_gps_lna_pin=0       : GPS LNA control pin = 0. Maps to GPIO 91
                              (wmt_gps_lna, confirmed in DTBO fragment@4)
    wmt_gps_lna_enable=0    : GPS LNA disabled by default at WMT init
    co_clock_flag=1         : WiFi, Bluetooth, and GPS share the same
                              reference clock oscillator (co-clock active)
    bt_tssi_from_wifi=3     : Bluetooth Tx power derived from WiFi TSSI
                              calibration, mode 3
    bt_tssi_target=3776     : Bluetooth target TSSI value (0xEC0) — transmit
                              signal strength index target for BT PA calibration

  PROC DRIVER NODES:
    /proc/driver/wmt_dbg    : WMT debug interface (requires prior echo to wmt_psm
                              to populate; idle state returns "no data available")
    /proc/driver/wmt_aee    : WMT AEE (Android Exception Engine) interface
    /proc/driver/wmt_dump_info : WMT dump trigger
    /proc/driver/stp_sdio_own  : STP SDIO ownership register
    /proc/driver/stp_sdio_rxdbg: STP SDIO RX debug
    /proc/driver/stp_sdio_txdbg: STP SDIO TX debug
    /proc/driver/conn_md_dbg   : Connectivity-modem debug interface

  WMT LAUNCHER SERVICE:
    Binary  : /vendor/bin/wmt_launcher -p /vendor/firmware/ -o 1
    PID     : 415
    User    : system
    Class   : early_hal
    Runtime library map (from /proc/415/maps — confirmed):
      /vendor/bin/wmt_launcher
      /system/lib64/libnetd_client.so
      /system/lib64/liblog.so
      /system/lib64/libc++.so
      /system/lib64/vndk-sp-29/libc++.so
      /system/lib64/vndk-sp-29/libbase.so
      /system/lib64/vndk-sp-29/libcutils.so
      /apex/com.android.runtime/lib64/bionic/libc.so
      /apex/com.android.runtime/lib64/bionic/libm.so
      /apex/com.android.runtime/lib64/bionic/libdl.so
      /dev/__properties__/u:object_r:wmt_prop:s0  ← WMT-specific property namespace
    NOTE: wmt_prop sysfs property namespace is unique to wmt_launcher —
          no other connectivity service has a dedicated property namespace.
          Confirms wmt_launcher is the sole owner of WMT chip state.

# 9.3 WIFI

  Chip generation     : gen4m (confirmed: ro.vendor.wlan.gen=gen4m)
  WiFi firmware ver   : t-neptune-main-soc1_0e1-1916-CERVINO_SOC1_0_E1_ASIC-20200326221201
    Decoded:
      Codename        : CERVINO (gen4m firmware codename)
      SOC variant     : soc1_0 E1 ASIC (silicon stepping E1, production ASIC)
      Build date      : 2020-03-26 22:12:01 UTC
      Build string    : t-neptune-main-soc1_0e1-1916
  WiFi HAL version    : android.hardware.wifi@1.3
  Supplicant HAL      : vendor.mediatek.hardware.wifi.supplicant@2.0
                        (2.1 library ships but only 2.0 declared in manifest)

  NETWORK INTERFACES (from ip link — confirmed live):

    wlan0 : UP, DORMANT — primary station interface
      mtu   : 1500
      qdisc : mq (multi-queue)
      qlen  : 3000
      state : UP / mode DORMANT (associated but no data path active at capture)
      MAC   : 66:b1:5b:92:8f:a6
              NOTE: Randomized MAC. True chip MAC is not exposed via ip link.
              WiFi MAC randomization seed stored in keystore as
              1010_USRPKEY_MacRandSecret — confirmed from Section 3.10.

    ap0   : DOWN — softAP interface
      mtu   : 1500
      qdisc : mq
      qlen  : 1000
      state : DOWN / mode DEFAULT
      MAC   : d6:85:1c:89:11:c8 (randomized softAP MAC)

    p2p0  : DOWN — WiFi Direct interface
      mtu   : 1500
      qdisc : mq
      qlen  : 1000
      state : DOWN / mode DORMANT
      MAC   : 32:5d:78:fb:e6:28 (randomized P2P MAC)

  FIRMWARE FILES (/vendor/firmware/):
    WIFI_RAM_CODE_soc1_0_1_1.bin      612,652 bytes  WiFi coprocessor RAM code
    soc1_0_patch_mcu_1_1_hdr.bin       26,350 bytes  WMT MCU patch binary
    soc1_0_ram_mcu_1_1_hdr.bin        104,324 bytes  WMT MCU RAM binary
    soc1_0_ram_wifi_1_1_hdr.bin       195,228 bytes  WiFi RAM secondary binary
    WMT_SOC.cfg                           121 bytes  WMT SoC config (see 9.2)

  WIFI HAL SERVICE:
    Binary  : /vendor/bin/hw/android.hardware.wifi@1.0-service-lazy-mediatek
    PID     : 1236 (lazy HAL — started on-demand by WiFi framework)
    User    : wifi
    Caps    : NET_ADMIN NET_RAW SYS_MODULE

  WIFI HAL RUNTIME LIBRARY MAP (from /proc/1236/maps — confirmed):
    Source                                     Library
    ─────────────────────────────────────────────────────────────────────────
    /vendor/bin/hw/                            android.hardware.wifi@1.0-service-lazy-mediatek
    /vendor/lib64/                             libwifi-hal.so
    /system/lib64/vndk-29/                     android.hardware.wifi@1.0.so
    /system/lib64/vndk-29/                     android.hardware.wifi@1.1.so
    /system/lib64/vndk-29/                     android.hardware.wifi@1.2.so
    /system/lib64/vndk-29/                     android.hardware.wifi@1.3.so
    /system/lib64/vndk-29/                     libwifi-system-iface.so
    /system/lib64/vndk-29/                     libnl.so
    /system/lib64/                             libbase.so
    /system/lib64/                             libvndksupport.so
    /system/lib64/                             libcgrouprc.so
    /system/lib64/                             libdl_android.so
    /system/lib64/                             libc++.so
    /system/lib64/                             liblog.so
    /system/lib64/                             libnetd_client.so
    /system/lib64/vndk-sp-29/                  libbase.so
    /system/lib64/vndk-sp-29/                  libhidlbase.so
    /system/lib64/vndk-sp-29/                  libhidltransport.so
    /system/lib64/vndk-sp-29/                  libc++.so
    /system/lib64/vndk-sp-29/                  libcutils.so
    /system/lib64/vndk-sp-29/                  libprocessgroup.so
    /system/lib64/vndk-sp-29/                  libutils.so
    /system/lib64/vndk-sp-29/                  libhwbinder.so
    /system/lib64/vndk-sp-29/                  libbinderthreadstate.so
    /apex/com.android.runtime/lib64/bionic/    libc.so, libm.so, libdl.so
    /apex/com.android.runtime/bin/             linker64
    NOTE: WiFi HAL loads all four wifi HIDL versions (@1.0 through @1.3)
          simultaneously — standard MTK Android 10 behavior.
    NOTE: WiFi HAL reads two dedicated property namespaces not present in
          any other connectivity service:
            u:object_r:wifi_log_prop:s0
            u:object_r:exported_wifi_prop:s0

  WLAN_ASSISTANT RUNTIME LIBRARY MAP (from /proc/709/maps — confirmed):
    Source                                     Library
    ─────────────────────────────────────────────────────────────────────────
    /vendor/bin/                               wlan_assistant
    /system/lib64/vndk-sp-29/                  libbase.so
    /system/lib64/vndk-sp-29/                  libc++.so
    /system/lib64/vndk-sp-29/                  libcutils.so
    /system/lib64/vndk-29/                     libnl.so
    /system/lib64/                             libc++.so
    /system/lib64/                             liblog.so
    /system/lib64/                             libnetd_client.so
    /apex/com.android.runtime/lib64/bionic/    libc.so, libm.so, libdl.so
    NOTE: wlan_assistant is a minimal binary with a lean dependency set.
          No HIDL libraries — it communicates with the kernel driver
          directly via netlink (libnl.so) after the HAL starts it.

  WIFI VENDOR LIBRARIES (from /vendor/lib64/ grep — confirmed):
    libwifi-hal.so                             Primary WiFi HAL implementation
    libkeystore-engine-wifi-hidl.so            WiFi keystore HIDL engine
    libkeystore-wifi-hidl.so                   WiFi keystore HIDL interface
    libwifitest.so                             WiFi test/diagnostic library
    vendor.mediatek.hardware.wifi.supplicant@2.0.so
    vendor.mediatek.hardware.wifi.supplicant@2.1.so  (ships, not declared in manifest)

  WIFI VENDOR LIBRARIES (from /vendor/lib/ 32-bit grep — confirmed):
    libkeystore-wifi-hidl.so                   (32-bit counterpart)
    NOTE: libkeystore-engine-wifi-hidl.so absent in 32-bit — 64-bit only.
          libwifi-hal.so absent in 32-bit — 64-bit only.
          libwifitest.so absent in 32-bit — 64-bit only.
          vendor.mediatek.hardware.wifi.supplicant absent in 32-bit.

  ADDITIONAL WIFI SERVICES:
    wificond : PID 664, wifi user — WiFi condition daemon
    wpa_supplicant : PID 1286, wifi user, VSS 2,141,600 KB (~2.1 GB)
                     NOTE: wpa_supplicant is the largest VSS process
                     on the device. This is expected — supplicant
                     allocates large virtual buffers for scan data.

# 9.4 BLUETOOTH

  BT device name    : APPLLP_5_MAX (ro.product.bt.name — Section 15)
  BT MAC            : ca:77:87:c7:16:00
                      Source: nRF52832 wiite_corp_ctrl/btMAC sysfs node.
                      This is the nRF52832 Bluetooth 5.0 BLE MAC, not the
                      MT6762 combo chip MAC. The nRF52832 is the BLE radio
                      on this device — the combo chip handles classic BT only.
                      Confirmed in Section 4.6.
  BT HAL version    : android.hardware.bluetooth@1.0

  BLUETOOTH FIRMWARE CONFIG (/vendor/firmware/BT_FW.cfg — 410 bytes, full content):

    # co-exist setting
    coex_bt_ctrl: 0x00
    coex_bt_ctrl_mode: 0x00
    coex_bt_ctrl_rw: 0x00
    coex_opp_time_ratio: 0x00
    coex_opp_time_ratio_bt_slot: 0x00
    coex_opp_time_ratio_wifi_slot: 0x00
    coex_lescan_time_ratio: 0x00
    coex_lescan_time_ratio_bt_slot: 0x00
    coex_lescan_time_ratio_wifi_slot: 0x00

  FIELD DECODE:
    All coexistence parameters set to 0x00. No time-division coexistence
    management is active. Antenna coexistence is handled at the WMT level
    (coex_wmt_ant_mode=1 in WMT_SOC.cfg). The BT_FW.cfg coexistence
    parameters represent an additional software-level arbitration layer
    that is fully disabled — WMT hardware arbitration handles the shared
    antenna entirely.
    NOTE: The commented-out VENDOR_CMD entries at the bottom of the file
          are disabled — no vendor-specific BT firmware commands are being
          injected at init.

  FIRMWARE FILES (/vendor/firmware/):
    soc1_0_ram_bt_1_1_hdr.bin         570,765 bytes  Bluetooth RAM firmware
    BT_FW.cfg                              410 bytes  BT firmware config (above)

  BT HAL SERVICE:
    Binary  : /vendor/bin/hw/android.hardware.bluetooth@1.0-service-mediatek
    PID     : 517
    User    : bluetooth
    Caps    : SYS_NICE

  BT HAL RUNTIME LIBRARY MAP (from /proc/517/maps — confirmed):
    Source                                     Library
    ─────────────────────────────────────────────────────────────────────────
    /vendor/bin/hw/                            android.hardware.bluetooth@1.0-service-mediatek
    /vendor/lib64/hw/                          bluetooth.default.so
    /vendor/lib64/                             libbluetooth_hw_test.so
    /system/lib64/                             android.hardware.bluetooth@1.0.so
    /system/lib64/                             libmtk_symbols.so
    /system/lib64/                             libvndksupport.so
    /system/lib64/                             libcgrouprc.so
    /system/lib64/                             libdl_android.so
    /system/lib64/                             libc++.so
    /system/lib64/                             liblog.so
    /system/lib64/                             libnetd_client.so
    /system/lib64/vndk-sp-29/                  libbase.so
    /system/lib64/vndk-sp-29/                  libhidlbase.so
    /system/lib64/vndk-sp-29/                  libhidltransport.so
    /system/lib64/vndk-sp-29/                  libc++.so
    /system/lib64/vndk-sp-29/                  libcutils.so
    /system/lib64/vndk-sp-29/                  libprocessgroup.so
    /system/lib64/vndk-sp-29/                  libutils.so
    /system/lib64/vndk-sp-29/                  libhwbinder.so
    /apex/com.android.runtime/lib64/bionic/    libc.so, libm.so, libdl.so
    /apex/com.android.runtime/bin/             linker64

  BLUETOOTH VENDOR LIBRARIES (from /vendor/lib64/ grep — confirmed):
    libbluetooth_audio_session_mediatek.so
    libbluetooth_hw_test.so
    libbluetooth_mtk.so
    libbluetooth_mtk_pure.so
    libbluetooth_relayer.so
    libbluetoothem_mtk.so      ← ODM library naming (em = engineer mode/extension)

  BLUETOOTH VENDOR LIBRARIES (from /vendor/lib/ 32-bit grep — confirmed):
    libbluetooth_audio_session_mediatek.so
    libbluetooth_hw_test.so
    libbluetooth_mtk.so
    libbluetooth_mtk_pure.so
    libbluetooth_relayer.so
    libbluetoothem_mtk.so
    NOTE: 32-bit BT library set is complete — all 6 libs mirror 64-bit.

  ADDITIONAL BT SERVICES:
    com.android.bluetooth : PID 1886, bluetooth user, VSS 5,776,856 KB (~5.5 GB)
                            Android Bluetooth stack application. Largest VSS
                            process on device — normal for BT stack Java process.

# 9.5 GPS / GNSS

  HAL              : android.hardware.gnss@2.0-service-mediatek (PID 524, system user)
  HIDL versions    : android.hardware.gnss@1.0, @1.1, @2.0 (all loaded simultaneously)
  MTK LBS HAL      : vendor.mediatek.hardware.lbs@1.0 (18 instances in manifest)

  GNSS HAL RUNTIME LIBRARY MAP (from /proc/524/maps — confirmed):
    Source                                     Library
    ─────────────────────────────────────────────────────────────────────────
    /vendor/bin/hw/                            android.hardware.gnss@2.0-service-mediatek
    /system/lib64/                             android.hardware.gnss@1.0.so
    /system/lib64/                             android.hardware.gnss@1.1.so
    /system/lib64/                             android.hardware.gnss@2.0.so
    /system/lib64/vndk-29/                     libnl.so
    /system/lib64/vndk-sp-29/                  libhidlbase.so
    /system/lib64/vndk-sp-29/                  libhidltransport.so
    /system/lib64/vndk-sp-29/                  libbase.so
    /system/lib64/vndk-sp-29/                  libc++.so
    /system/lib64/vndk-sp-29/                  libcutils.so
    /system/lib64/vndk-sp-29/                  libprocessgroup.so
    /system/lib64/vndk-sp-29/                  libutils.so
    /system/lib64/vndk-sp-29/                  libhwbinder.so
    /system/lib64/                             libvndksupport.so
    /system/lib64/                             libcgrouprc.so
    /system/lib64/                             libdl_android.so
    /system/lib64/                             libc++.so
    /system/lib64/                             liblog.so
    /system/lib64/                             libnetd_client.so
    /apex/com.android.runtime/lib64/bionic/    libc.so, libm.so, libdl.so
    /apex/com.android.runtime/bin/             linker64
    NOTE: GNSS HAL process has 80+ thread signal stack allocations at capture
          time — consistent with a heavy multi-threaded location daemon.
          This is expected behavior for the MT6762 GNSS stack.
    NOTE: libnl.so (netlink) present — GNSS HAL uses netlink for network-based
          location (NLP) and SUPL connectivity.

  MNLD DAEMON RUNTIME LIBRARY MAP (from /proc/631/maps — confirmed):
    Binary  : /vendor/bin/mnld  (mnld = modem navigation daemon)
    User    : gps
    Source                                     Library
    ─────────────────────────────────────────────────────────────────────────
    /vendor/bin/                               mnld
    /system/lib64/                             libbinder.so  ← binder (not HIDL)
    /system/lib64/                             libc++.so
    /system/lib64/                             libcutils.so  (system version)
    /system/lib64/                             libutils.so   (system version)
    /system/lib64/                             libprocessgroup.so (system version)
    /system/lib64/                             libvndksupport.so
    /system/lib64/                             libcgrouprc.so
    /system/lib64/                             libdl_android.so
    /system/lib64/                             liblog.so
    /system/lib64/                             libnetd_client.so
    /apex/com.android.runtime/lib64/bionic/    libc.so, libm.so, libdl.so
    NOTE: mnld uses standard Android binder (libbinder.so), not HIDL binder.
          It uses system/lib64 versions of libcutils, libutils, libprocessgroup
          — not vndk-sp-29. This is correct for a gps-user process that predates
          the HIDL layer and communicates with it via the LBS HIDL service.

  LBS HIDL SERVICE:
    Binary  : /vendor/bin/lbs_hidl_service
    PID     : 538
    User    : system
    Interface: vendor.mediatek.hardware.lbs@1.0 (18 instances — full AGPS stack)
    Vendor libraries (from /vendor/lib64/ grep — confirmed):
      lbs_hidl_service-impl.so      (64-bit only — no 32-bit counterpart)
      vendor.mediatek.hardware.lbs@1.0.so  (64-bit only)
    NOTE: /proc/538/maps output was corrupted by Gemini — it duplicated the
          BT HAL maps (PID 517) verbatim for PID 538. The library chain above
          is sourced from the vendor grep output and confirmed MTK LBS
          architecture, not from the corrupted maps output.

  GPS PROCESSES (all confirmed running — from ps -A):
    gnss_service  : PID 524  system  — GNSS HAL service (gnss@2.0)
    lbs_hidl_service: PID 538 system  — MTK LBS HIDL bridge
    mnld          : PID 631  gps     — Modem navigation daemon (primary)
    lbs_dbg       : PID 632  system  — LBS debug/logging daemon
    mtk_agpsd     : PID 704  gps     — Assisted GPS daemon

  FIRMWARE FILES (/vendor/firmware/):
    MNL.bin                           128 bytes  MNL binary (modem navigation link)

  GPS CONFIGURATION FILES (/vendor/etc/):
    agps_profiles_conf2.xml       27,012 bytes  AGPS profile configuration
    slp_conf                                    SUPL configuration:
      SUPL_ADDR=supl.google.com
      SUPL_PORT=7276
      SUPL_TLS=0
      SUPL_VERSION=2

  GPS HARDWARE PATH:
    GPS antenna control : GPIO 91 — GPS LNA enable
                          (wmt_gps_lna_pin=0 in WMT_SOC.cfg maps to GPIO 91
                          per DTBO fragment@4 GPS LNA pin definition)
    GPS LNA state       : Disabled at WMT init (wmt_gps_lna_enable=0)
                          Enabled dynamically by driver when GNSS session starts

# 9.6 USB

  Controller          : musb-hdrc (confirmed: sys.usb.controller=musb-hdrc)
  Physical address    : 0x11200000 (confirmed from hardware database iomem map)
  USB standard        : USB 2.0 (480 Mbps full-speed)
  VID                 : 0x0E8D (MediaTek) — confirmed at hardware register level
                        /config/usb_gadget/g1/idVendor = 0x0e8d
  PID                 : 0x201D — confirmed at hardware register level
                        /config/usb_gadget/g1/idProduct = 0x201d
                        NOTE: The 0x201C concern noted in the original Section 9
                        draft is RESOLVED. 0x201D confirmed at three independent
                        sources: getprop vendor.usb.pid, configfs idProduct node,
                        and prior /proc/iomem USB enumeration. No 0x201C in use.
  Current config      : mtp,adb (sys.usb.config = mtp,adb)
  Current state       : mtp,adb (sys.usb.state = mtp,adb — active at capture)
  Gadget config string: mtp_adb
                        (/config/usb_gadget/g1/configs/b.1/strings/0x409/configuration)

  USB GADGET FUNCTION INVENTORY (/config/usb_gadget/g1/functions/ — confirmed):
    ffs.adb         — ADB function (Android Debug Bridge)
    mtp.gs0         — MTP (Media Transfer Protocol) — active in current config
    ptp.gs1         — PTP (Picture Transfer Protocol)
    accessory.gs2   — Android Open Accessory (AOA)
    audio_source.gs3— USB audio source
    mass_storage.usb0 — USB mass storage (iAmCdRom.iso bicr path)
    hid.gs0         — HID (Human Interface Device)
    midi.gs5        — USB MIDI
    acm.gs0-gs3     — ACM serial (4 instances — AT command channels to modem)
    via_atc.gs0     — VIA AT command channel
    via_ets.gs0     — VIA ETS channel
    via_modem.gs0   — VIA modem channel
    NOTE: The via_* functions are modem/carrier AT command interfaces.
          The acm.gs0-gs3 functions are additional serial AT channels.
          None of these are active in normal mtp,adb mode.
          mass_storage.usb0 is the iAmCdRom.iso bicr virtual CD-ROM path.
          When persist.vendor.usb.bicr=no (stock default), mass_storage
          carries no image. See Section 13 iAmCdRom.iso documentation.

  USB POWER:
    Current limit   : 500,000 uA = 500 mA (standard USB 2.0 — confirmed Section 13)
    Voltage         : 5V
    Fast charge     : NOT SUPPORTED — device charges at USB 2.0 standard rate only

  USB VID/PID ROM BUILD NOTE:
    PRODUCT_PROPERTY_OVERRIDES must include:
      vendor.usb.vid=0x0E8D
      vendor.usb.pid=0x201D
    These values are required for ADB host recognition and MTP enumeration.
    Host systems cache VID/PID associations — any change breaks existing
    driver bindings on the ADB host.

# 9.7 CONNECTIVITY ROM BUILD REQUIREMENTS

  KERNEL MODULES — all 7 must be carried to /vendor/lib/modules/:
    wmt_drv.ko, wlan_drv_gen4m.ko, wmt_chrdev_wifi.ko
    gps_drv.ko, bt_drv.ko, fpsgo.ko, met.ko
    met.ko carries as dormant — do not attempt to remove it.

  FIRMWARE — all carry from /vendor/firmware/ as-is:
    WIFI_RAM_CODE_soc1_0_1_1.bin
    soc1_0_patch_mcu_1_1_hdr.bin
    soc1_0_ram_mcu_1_1_hdr.bin
    soc1_0_ram_wifi_1_1_hdr.bin
    soc1_0_ram_bt_1_1_hdr.bin
    BT_FW.cfg
    WMT_SOC.cfg
    MNL.bin

  INIT SCRIPTS — all required:
    init.wmt_drv.rc    — loads wmt_drv.ko on boot
    init.bt_drv.rc     — loads bt_drv.ko on connsys.driver.ready
    init.gps_drv.rc    — loads gps_drv.ko on connsys.driver.ready
    init.wlan_drv.rc   — loads wmt_chrdev_wifi.ko and wlan_drv_gen4m.ko
                         on connsys.driver.ready, starts wlan_assistant
    init_connectivity.rc — wmt_loader and wmt_launcher services (class early_hal)

  KEY PROPERTIES:
    ro.vendor.wlan.gen=gen4m
    ro.product.bt.name=APPLLP_5_MAX
    ro.default_wifi_hotspot=APPLLP_5_MAX
    wifi.interface=wlan0
    wifi.direct.interface=p2p0
    wifi.tethering.interface=ap0
    vendor.usb.vid=0x0E8D
    vendor.usb.pid=0x201D 

################################################################################
# SECTION 10 — VNDK-29 & VNDK-SP-29 INVENTORY
################################################################################
#
# PURPOSE:
#   Documents the complete VNDK-29 and VNDK-SP-29 library sets on this device,
#   and maps the three-version crypto/keymaster library architecture that is
#   unique to this ODM. This section is the single source of truth for linker
#   namespace resolution on C17S.
#
# AUTHORITATIVE SOURCES:
#   ls -la /system/lib64/vndk-29/       — confirmed live 2026-05-10
#   ls -la /system/lib64/vndk-sp-29/    — confirmed live 2026-05-10
#   ls -la /system/lib/vndk-29/         — confirmed live 2026-05-10
#   ls -la /system/lib/vndk-sp-29/      — confirmed live 2026-05-10
#   ls -la /system/lib64/vndk-sp-29/hw/ — confirmed live 2026-05-10
#   ls -la /system/lib/vndk-sp-29/hw/   — confirmed live 2026-05-10
#   /proc/PID/maps — runtime verification of which version loads per process
#
################################################################################

# 10.1 THREE-VERSION CRYPTO ARCHITECTURE — DEVICE-UNIQUE

  This device carries three independent, simultaneously present versions of
  libcrypto.so and libssl.so, each in a different location and serving a
  different subsystem. This is the direct result of ODM binary compilation
  decisions combined with MTK's VNDK-29 linker namespace design. All three
  versions are active at runtime.

  VERSION 1 — VNDK-29 NAMESPACE (Keymaster/Gatekeeper chain):
    /system/lib64/vndk-29/libcrypto.so     1,149,976 bytes  (64-bit)
    /system/lib/vndk-29/libcrypto.so         959,964 bytes  (32-bit)
    /system/lib64/vndk-29/libssl.so          338,912 bytes  (64-bit)
    /system/lib/vndk-29/libssl.so            343,932 bytes  (32-bit)
    Users    : android.hardware.keymaster@4.0-service (via VNDK-29 namespace)
               android.hardware.gatekeeper@1.0-service (via VNDK-29 namespace)
    SHA256   : libcrypto.so (64-bit): 91254cfc5b1b3a93dd43b1d65674293da1b13309c1bb8d169431ab34d51f95ea
               (confirmed from Section 3 crypto chain verification)
    Nature   : AOSP OpenSSL 1.1.1. Natural VNDK-29 linker namespace separation
               isolates this version from the system and vendor versions.
               The keymaster HAL process maps this version exclusively —
               confirmed via /proc/413/maps in Section 3.

  VERSION 2 — SYSTEM NAMESPACE (Keystore daemon and system processes):
    /system/lib64/libcrypto.so             1,264,696 bytes  (64-bit)
    /system/lib/libcrypto.so                 899,404 bytes  (32-bit)
    /system/lib64/libssl.so                  355,328 bytes  (64-bit)
    /system/lib/libssl.so                    242,376 bytes  (32-bit)
    Users    : keystore daemon (PID 642), gatekeeperd (PID 713),
               system_server and other system UID processes
    Nature   : AOSP OpenSSL, system partition. Different binary from VNDK-29
               version — confirmed by size difference (1,264,696 vs 1,149,976).
               System processes that link against /system/lib64/ directly
               (not via VNDK-29 namespace) load this version.

  VERSION 3 — VENDOR STRONGSWAN NAMESPACE (IPsec/WiFi Calling):
    /vendor/lib64/libcrypto-ss.so          1,787,984 bytes  (64-bit)
    /vendor/lib/libcrypto-ss.so            1,150,736 bytes  (32-bit)
    /vendor/lib64/libssl-ss.so               447,936 bytes  (64-bit)
    /vendor/lib/libssl-ss.so                 302,832 bytes  (32-bit)
    Users    : charon (Strongswan IKEv2 daemon), epdg_wod (ePDG watchdog)
               libcharon-ss.so, libipsec_ims_shr.so — WiFi Calling stack
    Nature   : ODM-compiled Strongswan-specific OpenSSL. The -ss suffix
               (Strongswan) is the ODM's own namespace isolation mechanism
               for the IPsec/WiFi Calling stack. These are statically distinct
               from both the VNDK-29 and system versions. Whether ODM-modified
               or stock Strongswan build requires Ghidra analysis — documented
               in Section 16. The -ss libs are larger than both other versions,
               consistent with a Strongswan-specific static link model or
               additional symbol inclusion.

  THE HARDCODED RPATH ARCHITECTURE:
    The ODM compiled their keymaster/gatekeeper vendor blobs (libkeymaster4.so,
    libkeymaster4support.so) with hardcoded DT_RPATH entries pointing to
    specific library search paths. This is the mechanism by which the ODM
    "controls" which version of libcrypto loads into which process.

    In the Android OS: VNDK-29 linker namespace separation handles everything
    automatically. The hardcoded RPATHs are effectively overridden by the
    namespace linker rules — the correct VNDK-29 version loads without conflict.

    In TWRP (no VNDK-29): The hardcoded RPATHs caused the default linker
    namespace collision documented in Section 3 — ODM blobs and TWRP bionic
    sharing one namespace, incompatible symbol tables, SIGSEGV before keymaster
    code executes. Solution: /crypto_isolation/ bubble with bind mount.

    This design is not a mistake — it is an ODM cost-reduction pattern where
    the vendor avoids maintaining proper Android namespace declarations and
    instead relies on path-based isolation. It works in the OS because VNDK-29
    happens to provide the isolation they depend on.

  NOTABLE ASYMMETRY — libkeymaster4support.so:
    /vendor/lib64/libkeymaster4support.so    165,776 bytes  (vendor)
    /system/lib64/libkeymaster4support.so    165,776 bytes  (system)
    /system/lib/libkeymaster4support.so      127,464 bytes  (system 32-bit)
    Both vendor and system 64-bit versions are byte-for-byte identical size.
    This is the ODM shipping their vendor build artifact into the system
    partition as well. The keystore daemon (PID 642) loads from /system/lib64/,
    the keymaster HAL (PID 413) loads from vendor via VNDK-29 namespace.

  NOTABLE ASYMMETRY — android.hardware.gatekeeper@1.0.so:
    Present in /system/lib64/ (92,856 bytes) — confirmed.
    Absent from /system/lib/ grep output — no 32-bit system counterpart found.
    Present in /system/lib64/vndk-29/ (92,856 bytes — identical size to system).
    Gatekeeper is 64-bit only at the system level on this device.

# 10.2 VNDK-29 — 64-BIT

  Location    : /system/lib64/vndk-29/
  Total files : 216
  Role        : Vendor Native Development Kit version 29 — 64-bit.
                Provides a stable, version-locked set of libraries that vendor
                blobs may link against without coupling to the system partition's
                evolving library ABIs. The VNDK-29 linker namespace prevents
                vendor processes from loading system versions and vice versa.

  CRYPTO CHAIN MEMBERS CONFIRMED [VERIFIED — live /proc/maps + ls]:
    libcrypto.so                       1,149,976  OpenSSL 1.1.1 (AOSP VNDK-29)
    libssl.so                            338,912  OpenSSL TLS (AOSP VNDK-29)
    libpuresoftkeymasterdevice.so         45,944  Pure-software keymaster impl
    libkeymaster_messages.so              84,024  Keymaster message structures
    libkeymaster_portable.so             248,912  Portable keymaster operations
    libgatekeeper.so                      28,072  Gatekeeper core
    libsoftkeymasterdevice.so            181,264  Software keymaster device
    android.hardware.keymaster@3.0.so    140,880  Keymaster 3.0 HIDL
    android.hardware.keymaster@4.0.so    158,504  Keymaster 4.0 HIDL
    android.hardware.gatekeeper@1.0.so    92,856  Gatekeeper HIDL

  COMPLETE FILE INVENTORY — /system/lib64/vndk-29/ (216 files):

  HIDL Interface Libraries — android.frameworks.*:
    android.frameworks.cameraservice.common@2.0.so      15,064
    android.frameworks.cameraservice.device@2.0.so     195,400
    android.frameworks.cameraservice.service@2.0.so    147,744
    android.frameworks.displayservice@1.0.so            167,336
    android.frameworks.schedulerservice@1.0.so           83,688
    android.frameworks.sensorservice@1.0.so             223,328
    android.frameworks.stats@1.0.so                     118,728
    android.frameworks.vr.composer@1.0.so               120,704

  HIDL Interface Libraries — android.hardware.*:
    android.hardware.atrace@1.0.so                       88,136
    android.hardware.audio.common@2.0.so                 15,040
    android.hardware.audio.common@4.0.so                 15,040
    android.hardware.audio.common@5.0.so                 27,560
    android.hardware.audio.effect@2.0.so              1,283,480
    android.hardware.audio.effect@4.0.so              1,275,160
    android.hardware.audio.effect@5.0.so              1,275,216
    android.hardware.audio@2.0.so                       625,832
    android.hardware.audio@4.0.so                       664,704
    android.hardware.audio@5.0.so                       673,048
    android.hardware.authsecret@1.0.so                   79,616
    android.hardware.automotive.audiocontrol@1.0.so      84,072
    android.hardware.automotive.evs@1.0.so              247,768
    android.hardware.automotive.vehicle@2.0.so          156,376
    android.hardware.biometrics.face@1.0.so             186,096
    android.hardware.biometrics.fingerprint@2.1.so      168,288
    android.hardware.bluetooth.a2dp@1.0.so              134,616
    android.hardware.bluetooth.audio@2.0.so             222,280
    android.hardware.bluetooth@1.0.so                   129,784
    android.hardware.boot@1.0.so                        101,376
    android.hardware.broadcastradio@1.0.so              253,360
    android.hardware.broadcastradio@1.1.so              286,904
    android.hardware.broadcastradio@2.0.so              321,000
    android.hardware.camera.common@1.0.so                15,048
    android.hardware.camera.device@1.0.so               278,416
    android.hardware.camera.device@3.2.so               202,952
    android.hardware.camera.device@3.3.so                98,088
    android.hardware.camera.device@3.4.so               162,112
    android.hardware.camera.device@3.5.so               244,992
    android.hardware.camera.metadata@3.2.so              14,984
    android.hardware.camera.metadata@3.3.so              15,000
    android.hardware.camera.metadata@3.4.so              15,016
    android.hardware.camera.provider@2.4.so             147,928
    android.hardware.camera.provider@2.5.so              93,784
    android.hardware.cas.native@1.0.so                   93,184
    android.hardware.cas@1.0.so                         234,616
    android.hardware.cas@1.1.so                         186,448
    android.hardware.configstore-utils.so                15,512
    android.hardware.configstore@1.0.so                 123,632
    android.hardware.configstore@1.1.so                 102,576
    android.hardware.confirmationui-support-lib.so       14,904
    android.hardware.confirmationui@1.0.so              125,448
    android.hardware.contexthub@1.0.so                  151,120
    android.hardware.drm@1.0.so                         368,568
    android.hardware.drm@1.1.so                         253,584
    android.hardware.drm@1.2.so                         383,896
    android.hardware.dumpstate@1.0.so                    75,376
    android.hardware.fastboot@1.0.so                     92,856
    android.hardware.gatekeeper@1.0.so                   92,856
    android.hardware.gnss.measurement_corrections@1.0.so 125,352
    android.hardware.gnss.visibility_control@1.0.so     125,384
    android.hardware.gnss@1.0.so                        962,664
    android.hardware.gnss@1.1.so                        326,056
    android.hardware.gnss@2.0.so                        657,168
    android.hardware.graphics.allocator@2.0.so           84,328
    android.hardware.graphics.allocator@3.0.so           84,368
    android.hardware.graphics.bufferqueue@1.0.so        204,312
    android.hardware.graphics.bufferqueue@2.0.so        182,632
    android.hardware.graphics.composer@2.1.so           255,424
    android.hardware.graphics.composer@2.2.so           211,480
    android.hardware.graphics.composer@2.3.so           265,232
    android.hardware.health.storage@1.0.so              117,256
    android.hardware.health@1.0.so                       88,784
    android.hardware.health@2.0.so                      161,464
    android.hardware.input.classifier@1.0.so             83,784
    android.hardware.input.common@1.0.so                 15,024
    android.hardware.ir@1.0.so                           79,936
    android.hardware.keymaster@3.0.so                   140,880
    android.hardware.keymaster@4.0.so                   158,504
    android.hardware.light@2.0.so                        79,808
    android.hardware.media.bufferpool@1.0.so            163,824
    android.hardware.media.bufferpool@2.0.so            201,544
    android.hardware.media.c2@1.0.so                    530,696
    android.hardware.media.omx@1.0.so                   401,992
    android.hardware.media@1.0.so                        15,024
    android.hardware.memtrack@1.0.so                     79,888
    android.hardware.neuralnetworks@1.0.so              213,584
    android.hardware.neuralnetworks@1.1.so              101,960
    android.hardware.neuralnetworks@1.2.so              398,280
    android.hardware.nfc@1.0.so                         133,856
    android.hardware.nfc@1.1.so                         144,056
    android.hardware.nfc@1.2.so                         102,576
    android.hardware.oemlock@1.0.so                      92,968
    android.hardware.power.stats@1.0.so                 102,056
    android.hardware.power@1.0.so                        88,376
    android.hardware.power@1.1.so                        93,480
    android.hardware.power@1.2.so                        94,056
    android.hardware.power@1.3.so                        99,016
    android.hardware.radio.config@1.0.so                171,936
    android.hardware.radio.config@1.1.so                204,048
    android.hardware.radio.config@1.2.so                158,208
    android.hardware.radio.deprecated@1.0.so            171,912
    android.hardware.radio@1.0.so                     1,359,632
    android.hardware.radio@1.1.so                       969,856
    android.hardware.radio@1.2.so                     1,076,416
    android.hardware.radio@1.3.so                     1,019,400
    android.hardware.radio@1.4.so                     1,189,328
    android.hardware.secure_element@1.0.so              139,080
    android.hardware.secure_element@1.1.so              139,760
    android.hardware.sensors@1.0.so                     105,984
    android.hardware.sensors@2.0.so                     155,984
    android.hardware.soundtrigger@2.0-core.so            36,808
    android.hardware.soundtrigger@2.0.so                151,432
    android.hardware.soundtrigger@2.1.so                165,528
    android.hardware.soundtrigger@2.2.so                102,680
    android.hardware.tetheroffload.config@1.0.so         79,464
    android.hardware.tetheroffload.control@1.0.so       160,896
    android.hardware.thermal@1.0.so                      88,688
    android.hardware.thermal@1.1.so                     126,496
    android.hardware.thermal@2.0.so                     148,592
    android.hardware.tv.cec@1.0.so                      151,704
    android.hardware.tv.cec@2.0.so                      164,440
    android.hardware.tv.input@1.0.so                    130,232
    android.hardware.usb.gadget@1.0.so                  121,768
    android.hardware.usb@1.0.so                         126,272
    android.hardware.usb@1.1.so                         131,784
    android.hardware.usb@1.2.so                         150,040
    android.hardware.vibrator@1.0.so                     88,368
    android.hardware.vibrator@1.1.so                     89,152
    android.hardware.vibrator@1.2.so                     94,032
    android.hardware.vibrator@1.3.so                    103,160
    android.hardware.vr@1.0.so                           79,464
    android.hardware.weaver@1.0.so                       84,336
    android.hardware.wifi.hostapd@1.0.so                 88,496
    android.hardware.wifi.hostapd@1.1.so                135,344
    android.hardware.wifi.offload@1.0.so                143,256
    android.hardware.wifi.supplicant@1.0.so           1,357,600
    android.hardware.wifi.supplicant@1.1.so             502,056
    android.hardware.wifi.supplicant@1.2.so             747,280
    android.hardware.wifi@1.0.so                      1,056,352
    android.hardware.wifi@1.1.so                        199,048
    android.hardware.wifi@1.2.so                        566,192
    android.hardware.wifi@1.3.so                        333,560

  HIDL Interface Libraries — android.hidl.*, android.system.*:
    android.hidl.allocator@1.0.so                        80,176
    android.hidl.memory.block@1.0.so                     15,008
    android.hidl.token@1.0-utils.so                      20,512
    android.hidl.token@1.0.so                            88,160
    android.system.net.netd@1.0.so                       79,928
    android.system.net.netd@1.1.so                       97,248
    android.system.suspend@1.0.so                       117,104
    android.system.wifi.keystore@1.0.so                  88,656

  Native Libraries — lib*:
    libadf.so                                            23,576
    libaudioroute.so                                     23,680
    libaudioutils.so                                     87,376
    libbinder.so                                        497,936
    libcamera_metadata.so                                41,448
    libcap.so                                            27,824
    libclang_rt.scudo-aarch64-android.so                669,376
    libclang_rt.scudo_minimal-aarch64-android.so        325,720
    libclang_rt.ubsan_standalone-aarch64-android.so   1,048,024
    libcn-cbor.so                                        19,176
    libcodec2.so                                         14,896
    libcrypto.so                                      1,149,976  ← VNDK-29 crypto (VERSION 1)
    libcrypto_utils.so                                   15,056
    libcurl.so                                          386,592
    libdiskconfig.so                                     23,640
    libdumpstateutil.so                                  36,840
    libevent.so                                         312,208
    libexif.so                                          228,272
    libexpat.so                                         151,504
    libfmq.so                                            19,704
    libgatekeeper.so                                     28,072  ← VNDK-29 gatekeeper
    libgui.so                                           943,096
    libhardware_legacy.so                                24,256
    libhidlallocatorutils.so                             15,144
    libhidlcache.so                                      50,008
    libjpeg.so                                          323,432
    libkeymaster_messages.so                             84,024  ← VNDK-29 keymaster msgs
    libkeymaster_portable.so                            248,912  ← VNDK-29 keymaster portable
    libldacBT_abr.so                                     15,048
    libldacBT_enc.so                                     52,736
    liblz4.so                                           130,512
    libmedia_helper.so                                  100,688
    libmedia_omx.so                                     239,776
    libmemtrack.so                                       16,072
    libminijail.so                                      133,000
    libmkbootimg_abi_check.so                            14,888
    libnetutils.so                                       36,496
    libnl.so                                            137,688
    libpcre2.so                                         315,832
    libpiex.so                                          104,600
    libpng.so                                           214,832
    libpower.so                                          24,168
    libprocinfo.so                                       19,848
    libprotobuf-cpp-full.so                           1,910,072
    libprotobuf-cpp-lite.so                             279,768
    libpuresoftkeymasterdevice.so                        45,944  ← VNDK-29 soft keymaster
    libradio_metadata.so                                 15,488
    libselinux.so                                        62,992
    libsoftkeymasterdevice.so                           181,264  ← VNDK-29 soft keymaster device
    libspeexresampler.so                                 24,208
    libsqlite.so                                      1,146,440
    libssl.so                                           338,912  ← VNDK-29 TLS (VERSION 1)
    libstagefright_bufferpool@2.0.so                    178,968
    libstagefright_bufferqueue_helper.so                560,992
    libstagefright_foundation.so                        180,504
    libstagefright_omx.so                               269,152
    libstagefright_omx_utils.so                          23,976
    libstagefright_xmlparser.so                         103,472
    libsysutils.so                                       53,512
    libtinyalsa.so                                       36,512
    libtinyxml2.so                                       79,760
    libui.so                                            173,936
    libusbhost.so                                        23,776
    libwifi-system-iface.so                              24,064
    libxml2.so                                        1,320,184
    libyuv.so                                           299,536
    libziparchive.so                                     54,104

# 10.3 VNDK-SP-29 — 64-BIT

  Location    : /system/lib64/vndk-sp-29/
  Total items : 35 .so files + 1 hw/ subdirectory
  Role        : VNDK Same-Process libraries — version 29, 64-bit.
                These libraries are loaded into vendor HAL processes directly
                (same process, not cross-process). They are the most
                performance-critical VNDK libraries — binder, hidlbase, utils,
                processgroup — loaded by every HAL service on the device.
                Confirmed loaded by: keymaster (PID 413), bluetooth (PID 517),
                GNSS (PID 524), WiFi HAL (PID 1236), wlan_assistant (PID 709),
                and every other HAL-class process.

  hw/ SUBDIRECTORY:
    Location : /system/lib64/vndk-sp-29/hw/
    Contents : 1 file
    android.hidl.memory@1.0-impl.so     25,824
    Role     : Ashmem memory allocator passthrough implementation.
               Loaded by gralloc (PID 526), pq service (PID 706),
               media.codec (PID 667) — confirmed via lshal Section 2.

  COMPLETE FILE INVENTORY — /system/lib64/vndk-sp-29/ (35 files):

  HIDL Interface Libraries:
    android.hardware.graphics.common@1.0.so      14,984
    android.hardware.graphics.common@1.1.so      15,000
    android.hardware.graphics.common@1.2.so      15,040
    android.hardware.graphics.mapper@2.0.so      93,184
    android.hardware.graphics.mapper@2.1.so      98,216
    android.hardware.graphics.mapper@3.0.so     102,200
    android.hardware.renderscript@1.0.so        325,888
    android.hidl.memory.token@1.0.so             79,744
    android.hidl.memory@1.0.so                  125,624
    android.hidl.safe_union@1.0.so               14,984

  Native Libraries — lib*:
    libRSCpuRef.so                              500,200
    libRSDriver.so                              211,296
    libRS_internal.so                           322,744
    libbacktrace.so                             109,648
    libbase.so                                   83,552  ← used by all HAL processes
    libbcinfo.so                              1,154,472
    libbinderthreadstate.so                      23,896  ← used by all HAL processes
    libblas.so                                1,185,264
    libc++.so                                   845,936  ← C++ runtime for vendor processes
    libcompiler_rt.so                           111,152
    libcutils.so                                 73,704  ← used by all HAL processes
    libhardware.so                               15,256
    libhidlbase.so                              692,184  ← HIDL binder, all HAL processes
    libhidlmemory.so                             29,680
    libhidltransport.so                          14,896  ← used by all HAL processes
    libhwbinder.so                               14,896  ← hwbinder, all HAL processes
    libhwbinder_noltopgo.so                     166,664
    libion.so                                    15,184
    libjsoncpp.so                               132,320
    liblzma.so                                  152,672
    libprocessgroup.so                          251,208  ← used by all HAL processes
    libunwindstack.so                           373,464
    libutils.so                                 117,416  ← used by all HAL processes
    libutilscallstack.so                         28,520
    libz.so                                     102,344

  NOTE: libhidlbase.so (692,184 bytes) is one of the three ODM-modified
        libraries confirmed from Section 3. The version in vndk-sp-29 is the
        AOSP stock version. The ODM-modified version lives in the crypto
        isolation bubble (/crypto_isolation/) and is used only in TWRP.
        In the OS, VNDK-29 namespace separation ensures the stock version
        loads for HAL processes — the ODM-modified version is never invoked
        during normal Android operation.

# 10.4 VNDK-29 — 32-BIT

  Location    : /system/lib/vndk-29/
  Total files : 216
  Role        : 32-bit counterpart of the 64-bit VNDK-29 set.
                All 216 libraries present — complete mirror with 32-bit binaries.
                This device runs zygote64_32 (ro.zygote=zygote64_32) — both
                64-bit and 32-bit app processes are supported. The 32-bit VNDK
                is required for 32-bit vendor process compatibility.

  CRYPTO CHAIN MEMBERS (32-bit sizes):
    libcrypto.so                           959,964  ← VNDK-29 crypto 32-bit (VERSION 1)
    libssl.so                              343,932  ← VNDK-29 TLS 32-bit (VERSION 1)
    libpuresoftkeymasterdevice.so           37,312
    libkeymaster_messages.so                63,848
    libkeymaster_portable.so               184,856
    libgatekeeper.so                        23,776
    libsoftkeymasterdevice.so              138,724
    android.hardware.keymaster@3.0.so      105,620
    android.hardware.keymaster@4.0.so      118,980
    android.hardware.gatekeeper@1.0.so      70,268

  NOTE: All 216 library names are identical to the 64-bit set. File sizes
        differ as expected for 32-bit ARM compilation. The 32-bit libcrypto.so
        at 959,964 bytes vs 64-bit 1,149,976 bytes confirms independent
        compilation for each ABI — not a fat binary.

  COMPLETE FILE INVENTORY — /system/lib/vndk-29/ (216 files):
  [All 216 filenames identical to 10.2 64-bit list — sizes differ per ABI.
   Key size reference for 32-bit crypto chain documented above.
   Full size listing available from raw ls output captured 2026-05-10.]

# 10.5 VNDK-SP-29 — 32-BIT

  Location    : /system/lib/vndk-sp-29/
  Total items : 35 .so files + 1 hw/ subdirectory
  Role        : 32-bit counterpart of the 64-bit VNDK-SP-29 set.
                Complete mirror — all 35 library names identical to 64-bit.
                Required for 32-bit HAL process support.

  hw/ SUBDIRECTORY:
    Location : /system/lib/vndk-sp-29/hw/
    Contents : 1 file
    android.hidl.memory@1.0-impl.so     24,772  (32-bit — vs 25,824 in 64-bit)

  KEY 32-BIT SIZES:
    libbase.so                            63,220
    libhidlbase.so                       491,488
    libhidltransport.so                   14,868
    libhwbinder.so                        14,868
    libcutils.so                          57,872
    libprocessgroup.so                   186,404
    libutils.so                           89,284
    libc++.so                            580,768

  NOTE: All 35 library names are identical to the 64-bit set.
        Full size listing from raw ls output captured 2026-05-10.

# 10.6 VNDK VERSION DECLARATION

  VNDK version        : 29
  Declared in         : ro.vndk.version=29 (confirmed live)
  Extra versions      : PRODUCT_EXTRA_VNDK_VERSIONS := 29
  vendor-ndk version  : 29 (vendor compatibility matrix confirmed Section 2.4)
  system-sdk version  : 29
  VNDK snapshot path  : /system/lib64/vndk-29/ and /system/lib/vndk-29/
  VNDK-SP path        : /system/lib64/vndk-sp-29/ and /system/lib/vndk-sp-29/

# 10.7 CRYPTO LIBRARY LOCATION SUMMARY

  All confirmed from live device 2026-05-10. Three-version architecture.

  Library               Location                         Size (64-bit)   User
  ─────────────────────────────────────────────────────────────────────────────
  libcrypto.so          /system/lib64/vndk-29/           1,149,976       keymaster HAL
  libcrypto.so          /system/lib64/                   1,264,696       keystore, system
  libcrypto-ss.so       /vendor/lib64/                   1,787,984       IPsec/WiFi Calling
  libssl.so             /system/lib64/vndk-29/             338,912       keymaster HAL
  libssl.so             /system/lib64/                     355,328       keystore, system
  libssl-ss.so          /vendor/lib64/                     447,936       IPsec/WiFi Calling
  libkeymaster4.so      /vendor/lib64/                      91,032       keymaster HAL (dirty link bridge)
  libkeymaster4support  /vendor/lib64/                     165,776       keymaster HAL
  libkeymaster4support  /system/lib64/                     165,776       keystore daemon
  libpuresoftdevice     /system/lib64/vndk-29/              45,944       keymaster HAL (VNDK namespace)
  libgatekeeper.so      /system/lib64/vndk-29/              28,072       gatekeeper HAL (VNDK namespace)
  ─────────────────────────────────────────────────────────────────────────────
  NOTE: libpuresoftkeymasterdevice.so is VNDK-29 ONLY — not present in
        /system/lib64/ directly and not present in /vendor/lib64/.
        The VNDK-29 namespace is the sole delivery mechanism for the
        pure-software keymaster implementation on this device.

################################################################################
# SECTION 11 — FIRMWARE BLOBS
################################################################################
#
# PURPOSE:
#   Documents the complete firmware blob inventory for the C17S. Covers all
#   binary firmware images loaded at runtime by kernel drivers and HAL services.
#   Two locations: /vendor/firmware/ (standard firmware path declared in kernel
#   cmdline as firmware_class.path=/vendor/firmware) and /vendor/bin/ (CCU
#   camera ISP firmware, deployed as executables by ODM convention).
#
# AUTHORITATIVE SOURCES:
#   ls -la /vendor/firmware/                   — confirmed live 2026-05-10
#   ls -la /vendor/bin/ | grep firmware/ccu    — confirmed live 2026-05-10
#   SHA256 hashes                              — from hardware database
#                                                forensic session 2026-04-14
#   Hardware identification                    — from hardware database and
#                                                init script analysis
#
# ROM BUILD RULE:
#   Every file in this section must be carried into the ROM unchanged.
#   No firmware binary may be substituted, modified, or omitted.
#   SHA256 hashes are the integrity reference — any deviation from these
#   values indicates a wrong-version or corrupted file.
#
################################################################################

# 11.1 /vendor/firmware/ — COMPLETE INVENTORY

  Location            : /vendor/firmware/
  Total files         : 10
  Firmware path decl  : firmware_class.path=/vendor/firmware (kernel cmdline)
  All permissions     : -rw-r--r-- root:root

  File                              Size      SHA256
  ─────────────────────────────────────────────────────────────────────────────
  BT_FW.cfg                           410     e018d7b8153af454d7e5c51626d584582904ffc5ba2bd8edb5d2a1ed27124b67
  MNL.bin                             128     38723a2e5e8a17aa7950dc008209944e898f69a7bd10a23c839d341e935fd5ca
  WIFI_RAM_CODE_soc1_0_1_1.bin    612,652     bffe8f712bf16666b5ccec1d7eb1149334e6b8f66652a68e4ebfb63347634566
  WMT_SOC.cfg                         121     6845e0f30219d5aedaf3791b8318eaaf7ae22d0087702268a815c8724243e2e8
  gt1151_default_firmware2.img     63,616     d2a6e1c2e1e350d940fad41991900fcfc4c81b3e786068560fe4f554602c3688
  rgx.fw                          118,784     af317093b7f374d9b223eb69a3fb22db894a324e09e1391aa2c07ed1cea968f0
  soc1_0_patch_mcu_1_1_hdr.bin     26,350     f90816944375a023801af68bb5cf1cb163c76d0cb3a52b79b34084cc8aaee91e
  soc1_0_ram_bt_1_1_hdr.bin       570,765     36fcbd3e272fbd496c330f31f90ba31c07ee5a9d48888fefa2d27c352966ed19
  soc1_0_ram_mcu_1_1_hdr.bin      104,324     47a349e4ddae156404fa4f65c824ad3da91d1027c9cbdfe0bb3ef16d47ca1600
  soc1_0_ram_wifi_1_1_hdr.bin     195,228     f99d7a212e4c4d00bf33f8241ff5bd4b8b93315558646e448c25a05762cc9191
  ─────────────────────────────────────────────────────────────────────────────

# 11.2 FIRMWARE FILE IDENTIFICATION

  ── BLUETOOTH ────────────────────────────────────────────────────────────────

  soc1_0_ram_bt_1_1_hdr.bin    570,765 bytes
    Hardware     : MediaTek gen4m combo chip — Bluetooth subsystem
    Type         : Bluetooth RAM firmware image with header
    Header format: MTK WMT binary header (hdr suffix — headered format)
    Loaded by    : wmt_drv.ko + bt_drv.ko during Bluetooth initialization
    Trigger      : vendor.connsys.driver.ready=yes property fires bt_drv insmod,
                   wmt_launcher loads this image into BT coprocessor RAM
    Largest of all connectivity firmware images — contains the complete
    Bluetooth stack firmware for the gen4m combo chip's BT core.

  BT_FW.cfg    410 bytes
    Hardware     : MediaTek gen4m combo chip — Bluetooth coexistence config
    Type         : Text configuration file (documented fully in Section 9.3)
    Content      : Coexistence parameters — all set to 0x00 (disabled).
                   WMT hardware coexistence handles antenna arbitration.
    Loaded by    : wmt_launcher on startup — parsed before BT firmware load

  ── WIFI ─────────────────────────────────────────────────────────────────────

  WIFI_RAM_CODE_soc1_0_1_1.bin    612,652 bytes
    Hardware     : MediaTek gen4m combo chip — WiFi subsystem
    Type         : WiFi coprocessor RAM code — primary WiFi firmware image
    Firmware ver : t-neptune-main-soc1_0e1-1916-CERVINO_SOC1_0_E1_ASIC-20200326221201
                   (confirmed live from vendor.wlan.firmware.version — Section 9.3)
    Loaded by    : wmt_drv.ko + wlan_drv_gen4m.ko during WiFi initialization
    Naming decode: WIFI_RAM_CODE = RAM code type, soc1_0 = SoC variant 1.0,
                   1_1 = firmware version 1.1

  soc1_0_ram_wifi_1_1_hdr.bin    195,228 bytes
    Hardware     : MediaTek gen4m combo chip — WiFi subsystem secondary
    Type         : WiFi RAM secondary binary with header
    Role         : Supplementary WiFi firmware loaded alongside the primary
                   WIFI_RAM_CODE image. Contains additional WiFi subsystem
                   code (PHY, calibration tables, or RF front-end control).
    Loaded by    : wmt_launcher -p /vendor/firmware/ scans and loads all
                   matching soc1_0 prefixed images in dependency order.

  soc1_0_patch_mcu_1_1_hdr.bin    26,350 bytes
    Hardware     : MediaTek gen4m combo chip — WMT MCU subsystem
    Type         : WMT MCU patch binary
    Role         : Patch overlay applied to the WMT combo chip MCU firmware
                   at initialization. The smallest firmware image — a delta
                   patch, not a complete firmware image.
    Loaded by    : wmt_launcher before RAM images — patch applies first

  soc1_0_ram_mcu_1_1_hdr.bin    104,324 bytes
    Hardware     : MediaTek gen4m combo chip — WMT MCU subsystem
    Type         : WMT MCU RAM binary (complete MCU firmware image)
    Role         : Main WMT MCU firmware. The MCU is the management core of
                   the combo chip — it arbitrates between WiFi, Bluetooth,
                   and GPS subsystems and handles coexistence at hardware level.
    Load order   : Loaded after patch, before WiFi and BT subsystem images.

  WMT_SOC.cfg    121 bytes
    Hardware     : MediaTek gen4m combo chip — WMT system configuration
    Type         : Text configuration file (documented fully in Section 9.2)
    Content      : Antenna mode, GPS LNA, clock sharing, BT TSSI parameters.
    Loaded by    : wmt_launcher on startup — first file read before any
                   firmware images load. Defines hardware topology for the
                   entire combo chip initialization sequence.

  ── GPS / GNSS ───────────────────────────────────────────────────────────────

  MNL.bin    128 bytes
    Hardware     : MT6762 GNSS subsystem — modem navigation link
    Type         : MNL (Modem Navigation Link) binary
    Role         : Communication interface binary between the GNSS hardware
                   block and the mnld daemon. At 128 bytes this is the smallest
                   binary in the firmware directory — it is an interface
                   descriptor or stub, not a full firmware image. The actual
                   GNSS firmware is embedded in the modem image (md1img partition).
    Loaded by    : mnld daemon (GPS user, PID 631) at GPS session start
    Note         : GPS LNA control via GPIO 91 (wmt_gps_lna_pin=0 in WMT_SOC.cfg)
                   is managed by wmt_drv.ko — not by MNL.bin directly.

  ── GPU ──────────────────────────────────────────────────────────────────────

  rgx.fw    118,784 bytes
    Hardware     : PowerVR GE8320 (Rogue architecture — codename MEOW)
    Type         : PowerVR GPU microkernel firmware
    Driver ver   : Rogue_DDK_Android rogueddk 1.11@5425693 (release)
    HW version   : 22.87.104.18 (confirmed from dumpsys — Section 5.2)
    Loaded by    : PowerVR kernel driver (pvr_timer, pvr_defer, pvr_wdg,
                   pvr_cacheop, pvr_misr kernel threads active — Section 1.4)
    Load trigger : GPU device initialization at kernel boot — confirms by
                   ro.boottime.gpu = 15,650,190,769 ns (~15.6 seconds from zero)
    Content      : PowerVR Rogue microkernel — the firmware that runs inside
                   the GPU's internal processor to manage command submission,
                   power state transitions, and hardware reset recovery.
    ROM critical : Without rgx.fw, GPU driver initialization fails, display
                   composition falls back to software rendering, and all GLES
                   operations fail. This is a hard dependency for the device
                   to function as a usable Android device.

  ── TOUCH CONTROLLER (BSP LEFTOVER) ─────────────────────────────────────────

  gt1151_default_firmware2.img    63,616 bytes
    Hardware     : Goodix GT1151 capacitive touch controller
    Status       : BSP LEFTOVER — NOT LOADED ON THIS UNIT
    Actual touch : Hynitron hyn_ts at I2C0 address 0x5D (confirmed Section 4)
    Explanation  : The production PCB was designed to support either Goodix
                   GT1151 or Hynitron as the touch controller. This unit
                   shipped with Hynitron. The Goodix firmware is present in
                   vendor/firmware/ because the ODM shipped the same vendor
                   partition image for all hardware variants rather than
                   creating separate images per touch controller variant.
    Load behavior: The Goodix driver probes at boot. When no GT1151 device
                   is detected on I2C (because hyn_ts owns I2C0 0x5D instead),
                   the Goodix driver skips firmware load silently. The file
                   consumes 63,616 bytes of vendor partition space with no
                   functional effect on this unit.
    ROM rule     : CARRY AS-IS. The HAL loader scans /vendor/firmware/
                   at init time. Removing this file does not affect operation
                   on this unit, but carry it to maintain stock vendor parity
                   and avoid any unexpected probe-time errors.

# 11.3 /vendor/bin/ CCU FIRMWARE — CAMERA ISP

  Location        : /vendor/bin/
  Files           : lib3a.ccu.dm, lib3a.ccu.pm
  Permissions     : -rwxr-xr-x root:shell (executable — ODM deployment method)
  Note            : These are camera ISP firmware binaries deployed to
                    /vendor/bin/ rather than /vendor/firmware/. The ODM
                    placed them here by convention — they are loaded by the
                    camera HAL at runtime, not by the kernel firmware loader.
                    firmware_class.path=/vendor/firmware does not cover them.

  lib3a.ccu.dm    6,676 bytes
    Hardware     : MT6762 CCU (Concurrent Control Unit) — camera ISP
    Type         : CCU data memory firmware image
    Role         : Data memory content loaded into the CCU coprocessor when
                   the camera ISP initializes. The CCU is a dedicated ARM
                   Cortex-M4 microcontroller inside the MT6762 ISP block that
                   handles 3A (Auto-exposure, Auto-focus, Auto-white-balance)
                   control loops in real time, offloading the main CPU.
                   .dm = data memory — the data segment of the CCU firmware.
    Loaded by    : Camera HAL (camerahalserver PID 671/692) via MTK ISP HAL
                   library chain during camera session initialization.

  lib3a.ccu.pm    29,628 bytes
    Hardware     : MT6762 CCU (Concurrent Control Unit) — camera ISP
    Type         : CCU program memory firmware image
    Role         : Program memory content loaded into the CCU coprocessor.
                   .pm = program memory — the code segment of the CCU firmware.
                   The PM image is the executable portion; the DM image is the
                   accompanying data. Both must load together for the CCU to
                   operate. Without them the 3A control loops fall back to
                   software execution on the main CPU, degrading camera
                   performance significantly.
    Loaded by    : Same camera HAL chain as lib3a.ccu.dm. Both images are
                   loaded in sequence: PM first (code), then DM (data).
    Size ratio   : PM (29,628) is ~4.4x larger than DM (6,676) — consistent
                   with a firmware where the algorithm code dominates over
                   data tables.

# 11.4 FIRMWARE LOAD SEQUENCE

  The complete runtime firmware load sequence across all subsystems:

  Stage          Time          Firmware                    Loader
  ─────────────────────────────────────────────────────────────────────────────
  Kernel boot    ~15.5s        rgx.fw                      PowerVR kernel driver
  Kernel boot    ~18s          WMT_SOC.cfg                 wmt_launcher
  Kernel boot    ~18s          soc1_0_patch_mcu_1_1_hdr.bin wmt_launcher
  Kernel boot    ~18s          soc1_0_ram_mcu_1_1_hdr.bin  wmt_launcher
  Kernel boot    ~18s          BT_FW.cfg                   wmt_launcher
  Kernel boot    ~18s          soc1_0_ram_bt_1_1_hdr.bin   wmt_launcher
  Kernel boot    ~19s          WIFI_RAM_CODE_soc1_0_1_1.bin wlan_drv_gen4m.ko
  Kernel boot    ~19s          soc1_0_ram_wifi_1_1_hdr.bin  wlan_drv_gen4m.ko
  Camera open    on-demand     lib3a.ccu.pm                camerahalserver
  Camera open    on-demand     lib3a.ccu.dm                camerahalserver
  GPS start      on-demand     MNL.bin                     mnld
  ─────────────────────────────────────────────────────────────────────────────
  gt1151_default_firmware2.img — never loaded on this unit (no GT1151 hardware)

# 11.5 ROM BUILD REQUIREMENTS

  All files carry unchanged. SHA256 hashes in 11.1 are the integrity reference.

  /vendor/firmware/ — all 10 files:
    PRODUCT_COPY_FILES declaration required for each file.
    Destination: $(TARGET_COPY_OUT_VENDOR)/firmware/

  /vendor/bin/ CCU firmware — both files:
    lib3a.ccu.dm and lib3a.ccu.pm deploy to vendor/bin/.
    Permissions must be preserved as executable (-rwxr-xr-x).
    If using PRODUCT_COPY_FILES, ensure execute bit is not stripped.
    Alternatively declare via PRODUCT_PACKAGES if wrapped in a prebuilt module.
    These two files are NOT covered by the standard firmware/ path —
    they require explicit deployment to vendor/bin/.

  CRITICAL: rgx.fw is the single highest-priority firmware file.
    GPU initialization failure = no display composition = black screen on boot.
    Verify this file's SHA256 against the database value before any ROM flash.
        
        
################################################################################
# SECTION 12 — APK INVENTORY SUPPLEMENT
# (Insert above existing COMPLETE ABSOLUTE IN-PARITY MAPPED APK INVENTORY)
################################################################################

# 12.A APK FILE SIZES — COMPLETE BY PARTITION

  Source: find -exec ls -la (live device 2026-05-10)

  VENDOR PARTITION (/vendor/app/ + /vendor/operator/):

  APK                                      Size (bytes)   Location
  ──────────────────────────────────────────────────────────────────────────
  SensorHub.apk                             2,209,203     vendor/app/
  Stopwatch.apk                             4,762,084     vendor/operator/app/

  PRODUCT PARTITION — /product/app/ (40 APKs):

  APK                                      Size (bytes)
  ──────────────────────────────────────────────────────
  Maps.apk                               145,520,241     ← 145MB Google Maps
  webview.apk                            133,486,789     ← 133MB WebView
  WearFitnessUart.apk                     30,633,532
  WearGallerySquare.apk                   32,246,830
  WearHomeSportDM101.apk                  16,031,362
  WearLatinIME.apk                        19,039,754
  WearWeather.apk                         10,807,771
  WearAgingTest.apk                       13,972,104
  WearReliableTest.apk                    13,997,525
  WearDevicemsg.apk                        8,172,512
  WearSpoUart.apk                          9,537,511
  WearScreenRecord.apk                     7,788,206
  WearAppDialStore.apk                     7,270,858
  WearClock.apk                            6,430,980
  WearMusic.apk                            5,899,791
  WearCamera.apk                           5,737,265
  WearHeartRateUart.apk                    4,569,178
  WearBreath.apk                           4,586,499
  WearBrowserLightning.apk                 4,471,292
  WearSleepUart.apk                        4,396,687
  WearSdkUart.apk                          3,882,167
  WearRecorder.apk                         3,690,129
  WearDialerContacts.apk                   3,844,760
  WearBluetoothSettings.apk               3,538,095
  WearMms.apk                              3,306,049
  WearFileManager.apk                      3,372,482
  WearAppFreeze.apk                        3,277,552
  WearHomeSettings.apk                     3,645,831
  WearSystemTool.apk                       2,873,531
  WearCalendar.apk                         2,508,709
  WearThirdPartAppAdapter.apk              2,124,193
  WearCleanTaskPro.apk                     2,370,797
  FaceUnlockLauncher.apk                   1,995,763
  WearInChargingUi.apk                     1,820,317
  WearSyncContact.apk                      1,703,839
  WearConfig.apk                           1,794,095
  WearLanguageSettings.apk                 4,542,897
  WearHealthStepUart.apk                     389,950
  GoogleLocationHistory.apk                   45,904
  ModuleMetadataGooglePrebuilt.apk             25,167

  PRODUCT PARTITION — /product/priv-app/ (20 APKs):

  APK                                      Size (bytes)
  ──────────────────────────────────────────────────────
  Velvet.apk                             190,375,551     ← 190MB Google Assistant
  GmsCore.apk                            139,766,888     ← 139MB GMS Core
  GmsCore/m/independent/AndroidPlatformServices.apk
                                          11,299,778
  Phonesky.apk                           106,363,988     ← 106MB Play Store
  HeilsFaceUnlockDM101.apk (see note)   103,222,085     ← see Section 16
  WearSettings.apk                        40,647,473
  WearSystemUIWithLegacyRecents.apk       17,389,102
  WearMtkDialerIncallUi.apk              13,467,992
  WearMtkContacts.apk                    10,483,073
  ImsService.apk (system/priv-app)        6,524,791
  WearTelecom.apk (system/priv-app)       6,430,525
  WearEngineerMode.apk (system/priv-app)  6,536,845
  GmsCore (system — see above)
  CarrierServices.apk                     7,936,509
  AdupsFota.apk                           7,367,820     ← see Section 16
  StorageManager.apk                      5,546,124
  WearOOBE.apk                            3,465,898
  WearCommon.apk                          2,134,345
  WearDeviceDeamPix.apk                   3,088,919     ← hardware-critical
  WearSystemMode.apk                         37,533
  GoogleOneTimeInitializer.apk              163,696
  MtkCarrierConfig.apk                      258,566
  Omacp.apk                                 553,444
  ConfigUpdater.apk                       5,395,748
  GoogleServicesFramework.apk             4,296,499

  PRODUCT PARTITION — /product/overlay/ (9 APKs):

  APK                                      Size (bytes)
  ──────────────────────────────────────────────────────
  HomeThemeDM101.apk                       2,496,722
  WearSdkIconOverlay.apk                      74,518
  WearSdkIconOverlay_Wiiwear.apk              74,518
  SdkDM101.apk                                94,913
  framework-res__auto_generated_rro_product.apk  115,109
  GmsConfigOverlay.apk                        20,911
  GmsGoogleDocumentsUIOverlay.apk             12,638
  GmsGsaConfigOverlay.apk                     12,638
  GoogleExtServicesConfigOverlay.apk          12,638

  SYSTEM PARTITION — /system/app/ (19 APKs):

  APK                                      Size (bytes)
  ──────────────────────────────────────────────────────
  WearBluetooth.apk                        4,023,020
  WearFactoryMode.apk                      4,059,274
  WearCalculator.apk                       9,073,086
  WearScreenLock.apk                       3,048,349
  WearEngineerMode.apk                     6,536,845
  WearPinyinIme.apk                        1,549,722
  LocationEM2.apk                            714,786
  DebugLoggerUI.apk                          627,285
  CaptivePortalLogin.apk                     339,148
  SimAppDialog.apk                           298,516
  BatteryWarning.apk                         144,381
  KeyChain.apk                               115,326
  YGPS.apk                                   278,988
  CarrierDefaultApp.apk                      119,543
  AutoDialer.apk                              78,735
  Gba.apk                                     94,703
  SecureElement.apk                          156,062
  WearHTMLViewer.apk                          29,151
  PacProcessor.apk                            16,798
  GoogleExtShared.apk                         12,695

  SYSTEM PARTITION — /system/priv-app/ (32 APKs):

  APK                                      Size (bytes)
  ──────────────────────────────────────────────────────
  HeilsFaceUnlockDM101.apk               103,222,085     ← see Section 16
  MtkTeleService.apk                      10,318,535
  WearMtkMms.apk                          10,887,720
  ImsService.apk                           6,524,791
  WearEngineerMode.apk                     6,536,845
  WearTelecom.apk                          6,430,525
  WearDocumentsUI.apk                      4,717,562
  WearPermissionController.apk             5,295,213
  MtkDownloadProvider.apk                  5,499,815
  GoogleNetworkStack.apk                   2,006,288
  WearPackageInstaller.apk                 1,658,007
  LPPeService.apk                          1,257,508
  MtkContactsProvider.apk                 1,067,481
  WearMediaProvider.apk                      403,498
  MtkMmsAppService.apk                       311,146
  WearSettingsProvider.apk                   304,073
  WapiCertManager.apk                        296,287
  MtkTelephonyProvider.apk                   570,583
  MtkDownloadProviderUi.apk                  287,785
  Shell.apk                                  340,145
  VpnDialogs.apk                             127,684
  GoogleExtServicesPrebuilt.apk              115,395
  GoogleNetworkPermissionConfig.apk           45,602
  StatementService.apk                        65,950
  FusedLocation.apk                           33,182
  MtkMmsService.apk                           98,786
  MtkTelephonyAssist.apk                      53,662
  MtkCapCtrl.apk                              57,758
  InputDevices.apk                           204,534
  ProxyHandler.apk                            24,990
  UserDictionaryProvider.apk                  37,278
  DynamicSystemInstallationService.apk        37,371
  ExternalStorageProvider.apk                 53,662

# 12.B NATIVE LIBRARIES EMBEDDED IN APKs

  Source: find /product/app /product/priv-app /system/app /system/priv-app
          -name "*.so" (live device)

  APK                        Library                        Arch
  ──────────────────────────────────────────────────────────────────────────
  WearLatinIME.apk           libjni_latinime.so             arm64
  WearBluetooth.apk          libmtkbluetooth_jni.so         arm64
  WearPinyinIme.apk          libjni_pinyinime.so            arm (32-bit only)
  PacProcessor.apk           libjni_pacprocessor.so         arm64
  WapiCertManager.apk        libwapi_cert_jni.so            arm + arm64 (both)

  WearDeviceDeamPix.apk (from Section 12.5 analysis):
    libSerialPort.so                                         arm64 + armeabi-v7a

  NOTE: These native libs are bundled inside the APK and extracted to
        /data/app/<package>/lib/ at install time by PackageManager.
        They are NOT in vendor/lib64/ or system/lib64/.
        ROM carry-forward: APK carries its own .so — no separate action needed.

  NOTE: WearPinyinIme uses 32-bit only arm lib. PacProcessor and
        WearBluetooth are arm64 only. WapiCertManager ships both arches.

# 12.C RUNTIME PROCESS STATE (live device 2026-05-10)

  Source: ps -A -o PID,USER,NAME (live device)

  WIITE PROCESSES — ALL CONFIRMED RUNNING:

  PID    User    Package                          Role
  ──────────────────────────────────────────────────────────────────────────
  1328   system  com.wiite.WiiteCommon            Common services layer
  2157   system  com.wiite.cleantask              Task/memory management
  2106   system  com.wiite.devicedaemon           nRF52832 bridge daemon
  2411   system  com.wiite.devicedaemon:communication  UART comm thread
  15119  system  com.wiite.power                  Power management
  2610   system  com.wiite.wearhealthuart         Health UART interface
  2649   system  com.wiite.wiite_bloodoxygen      SpO2 application
  2619   system  com.wiite.wiite_heartrate        Heart rate application
  2628   system  com.wiite.wiite_sleep            Sleep tracking
  4757   system  com.wiite.wiitestore             App store

  NOTE: com.wiite.devicedaemon:communication (PID 2411) confirmed with
        FD 52 = /dev/ttyS1 open — live UART connection to nRF52832.
        This is the process that owns the nRF52832 co-processor bridge.

  GOOGLE PROCESSES — RUNNING:

  PID    User       Package
  ──────────────────────────────────────────────────────────────────────────
  3039   u0_a46     com.google.android.gms
  17743  u0_a46     com.google.android.gms.persistent
  8980   u0_a46     com.google.process.gservices
  1650   u0_a13     com.google.android.ext.services
  17706  u0_a55     com.google.android.apps.maps
  17370  u0_a62     com.google.android.apps.photos
  16090  u0_a38     com.google.android.googlequicksearchbox:googleapp
  27438  u0_a38     com.google.android.googlequicksearchbox:interactor
  17999  u0_a38     com.google.android.googlequicksearchbox:search

  SYSTEM PROCESSES — RUNNING:

  PID    User           Package/Process
  ──────────────────────────────────────────────────────────────────────────
  1244   u0_a45         com.android.systemui
  1539   radio          com.android.phone
  1886   bluetooth      com.android.bluetooth
  2172   secure_element com.android.se
  17520  system         com.android.settings
  1342   network_stack  com.android.networkstack.process
  19310  u0_a20         com.android.permissioncontroller
  17276  u0_a18         android.process.media

  KERNEL THREADS — HARDWARE RELATED:

  PID    Thread name          Purpose
  ──────────────────────────────────────────────────────────────────────────
  249    irq/27-WIITE_CO      nRF52832 wake IRQ (GPIO 8)
  250    irq/25-WIITE_CO      nRF52832 no-wake IRQ (GPIO 6)
  304    mtk-tpd              MTK touch panel driver thread
  306    mtk-tpd              MTK touch panel driver thread 2
  233    mtk_io_boost         MTK I/O boost thread
  662    mtk_stp_btm          MTK BT/WiFi/GPS stack BTM thread
  660    mtk_stp_psm          MTK BT/WiFi/GPS power save thread
  198    mtk_vibrator         MTK vibrator kernel thread
  665    mtk_wmtd             WMT daemon kernel thread
  668    mtk_wmtd_worker      WMT daemon worker thread

# 12.D APK ROM DISPOSITION TABLE

  Legend: KEEP = required for function / STRIP = safe to remove /
          OPTIONAL = present but non-critical / FLAG = security review
          required (see Section 16)

  ─── HARDWARE-CRITICAL — NEVER REMOVE ────────────────────────────────────

  APK                        Partition    Reason
  ──────────────────────────────────────────────────────────────────────────
  WearDeviceDeamPix.apk      product/priv nRF52832 bridge — DFU + UART owner
  SensorHub.apk              vendor/app   Sensor hub HAL support
  WearBluetooth.apk          system/app   libmtkbluetooth_jni.so — BT JNI
  ImsService.apk             system/priv  VoLTE IMS service
  WearTelecom.apk            system/priv  Telephony framework
  MtkTeleService.apk         system/priv  MTK telephony service
  MtkContactsProvider.apk    system/priv  Contacts provider
  MtkTelephonyProvider.apk   system/priv  Telephony data provider
  WearSettingsProvider.apk   system/priv  Settings provider
  WearMediaProvider.apk      system/priv  Media provider
  FusedLocation.apk          system/priv  Fused location
  WearPackageInstaller.apk   system/priv  Package installer
  WearPermissionController.apk system/priv Permission management
  GoogleNetworkStack.apk     system/priv  Network stack
  MtkDownloadProvider.apk    system/priv  Download provider
  LPPeService.apk            system/priv  LPPe GPS service
  WearMtkMms.apk             system/priv  MMS
  GmsCore.apk                product/priv Google Play Services
  GoogleServicesFramework.apk product/priv GMS framework
  Phonesky.apk               product/priv Play Store
  CarrierServices.apk        product/priv Carrier services
  MtkCarrierConfig.apk       product/priv Carrier config
  WearCommon.apk             product/priv ODM common library
  WearSystemUIWithLegacyRecents product/priv System UI
  WearSettings.apk           product/priv Settings app
  WearMtkContacts.apk        product/priv Contacts
  WearMtkDialerIncallUi.apk  product/priv Dialer/in-call UI
  WearSystemMode.apk         product/priv System mode control
  StorageManager.apk         product/priv Storage management
  ConfigUpdater.apk          product/priv GMS config update
  webview.apk                product/app  WebView engine
  HomeThemeDM101.apk         product/overlay ODM theme
  WearSdkIconOverlay.apk     product/overlay SDK icons
  SdkDM101.apk               product/overlay SDK overlay

  ─── WIITE HEALTH STACK — KEEP FOR ROM, REPLACE IN PHASE 2 ───────────────

  APK                        Role
  ──────────────────────────────────────────────────────────────────────────
  WearFitnessUart.apk        Fitness tracking via nRF UART
  WearHeartRateUart.apk      Heart rate via nRF UART
  WearSpoUart.apk            SpO2 via nRF UART
  WearSleepUart.apk          Sleep tracking via nRF UART
  WearHealthStepUart.apk     Step counter via nRF UART
  WearSdkUart.apk            SDK UART interface
  WearBreath.apk             Breathing training
  com.wiite.wearhealthuart   Running — health UART bridge
  com.wiite.wiite_heartrate  Running — heart rate app
  com.wiite.wiite_bloodoxygen Running — SpO2 app
  com.wiite.wiite_sleep      Running — sleep app

  NOTE: Phase 2 — wiite_bridge_daemon (custom C++ daemon) will replace
        these ODM apps for clean nRF52832 communication. Keep all for
        initial ROM to ensure hardware functional from day one.

  ─── OPTIONAL — NON-CRITICAL FOR CORE FUNCTION ───────────────────────────

  APK                        Size         Notes
  ──────────────────────────────────────────────────────────────────────────
  Maps.apk                   145MB        Large — keep if space permits
  Velvet.apk                 190MB        Google Assistant — large
  WearGallerySquare.apk      32MB         Gallery
  WearHomeSportDM101.apk     16MB         Sport tracking UI
  WearLatinIME.apk           19MB         English keyboard
  WearPinyinIme.apk          1.5MB        Chinese input — removable for EN
  WearWeather.apk            10MB         Weather app
  WearCamera.apk             5.7MB        Camera app
  WearMusic.apk              5.9MB        Music player
  WearScreenRecord.apk       7.8MB        Screen recorder
  WearBrowserLightning.apk   4.5MB        Browser
  WearFileManager.apk        3.4MB        File manager
  WearMms.apk                3.3MB        MMS UI
  WearRecorder.apk           3.7MB        Audio recorder
  WearCalendar.apk           2.5MB        Calendar
  WearClock.apk              6.4MB        Clock/alarm
  WearDevicemsg.apk          8.2MB        Device messaging
  WearDialerContacts.apk     3.8MB        Dialer contacts
  WearAppFreeze.apk          3.3MB        App freeze utility
  WearCleanTaskPro.apk       2.4MB        Task cleaner
  WearBluetoothSettings.apk  3.5MB        BT settings
  WearSyncContact.apk        1.7MB        Contact sync
  WearLanguageSettings.apk   4.5MB        Language settings
  WearConfig.apk             1.8MB        Device config
  WearHomeSettings.apk       3.6MB        Home settings
  WearSystemTool.apk         2.9MB        System tools
  WearThirdPartAppAdapter.apk 2.1MB       Third party adapter
  WearInChargingUi.apk       1.8MB        Charging screen UI
  WearAppDialStore.apk       7.3MB        App store UI
  WearAgingTest.apk          14MB         Factory aging test
  WearReliableTest.apk       14MB         Factory reliability test
  YGPS.apk                   279KB        GPS test utility
  DebugLoggerUI.apk          627KB        Debug logging UI
  LocationEM2.apk            715KB        Location engineer mode
  WearEngineerMode.apk       6.5MB        Engineer mode
  WearFactoryMode.apk        4MB          Factory mode
  WearCalculator.apk         9MB          Calculator
  WearPinyinIme.apk          1.5MB        Chinese keyboard
  WearHTMLViewer.apk         29KB         HTML viewer stub
  AutoDialer.apk             79KB         Auto dialer
  WearOOBE.apk               3.5MB        First-run setup
  GoogleLocationHistory.apk  46KB         Location history
  FaceUnlockLauncher.apk     2MB          Face unlock UI wrapper
  WearScreenLock.apk         3MB          Screen lock UI

  ─── SECURITY FLAG — SEE SECTION 16 ──────────────────────────────────────

  APK                        Size         Flag reason
  ──────────────────────────────────────────────────────────────────────────
  AdupsFota.apk              7.4MB        OTA update client — data exfil risk
  HeilsFaceUnlockDM101.apk   103MB        Large, proprietary, face data access
  Stopwatch.apk              4.8MB        vendor/operator path — unusual location
  WapiCertManager.apk        296KB        WAPI cert management — dual arch .so
  Omacp.apk                  553KB        OMA Client Provisioning — carrier push

  ─── ALREADY USER-REMOVABLE (ODM declared) ───────────────────────────────

  Source: pms_sysapp_removable_vendor_list.txt (Section 2.14)
    com.android.vending (Phonesky) — user removable per ODM
    com.google.android.apps.maps (Maps) — user removable per ODM       
################################################################################
# COMPLETE ABSOLUTE IN-PARITY MAPPED APK INVENTORY
################################################################################


# PARTITION: /data
================================================================================
PACKAGE: com.android.chrome
PATH:    /data/app/com.android.chrome-P8LXGQAP56je6M4sIoGsEA
UID:     10065
VERSION: 147.0.7727.137
PERMISSIONS:
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_ADSERVICES_ATTRIBUTION
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.BLUETOOTH_CONNECT
  - android.permission.BLUETOOTH_ADVERTISE
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_SCAN
  - android.permission.READ_CONTACTS
  - android.permission.REORDER_TASKS
  - android.permission.REQUEST_INSTALL_PACKAGES
  - android.permission.CAMERA
  - android.permission.CAPTURE_KEYBOARD
  - android.permission.DOWNLOAD_WITHOUT_NOTIFICATION
  - android.permission.FOREGROUND_SERVICE
  - android.permission.FOREGROUND_SERVICE_CAMERA
  - android.permission.FOREGROUND_SERVICE_MICROPHONE
  - android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION
  - android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK
  - android.permission.FOREGROUND_SERVICE_DATA_SYNC
  - android.permission.RUN_USER_INITIATED_JOBS
  - android.permission.GET_ACCOUNTS
  - android.permission.INTERNET
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.NFC
  - android.permission.POST_NOTIFICATIONS
  - android.permission.QUERY_ALL_PACKAGES
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_MEDIA_AUDIO
  - android.permission.READ_MEDIA_IMAGES
  - android.permission.READ_MEDIA_VIDEO
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.RECORD_AUDIO
  - android.permission.REPOSITION_SELF_WINDOWS
  - android.permission.USE_CREDENTIALS
  - android.permission.USE_BIOMETRIC
  - android.permission.USE_FINGERPRINT
  - android.permission.VIBRATE
  - android.permission.WAKE_LOCK
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.CREDENTIAL_MANAGER_QUERY_CANDIDATE_CREDENTIALS
  - android.permission.CREDENTIAL_MANAGER_SET_ALLOWED_PROVIDERS
  - android.permission.CREDENTIAL_MANAGER_SET_ORIGIN
  - android.permission.QUERY_ADVANCED_PROTECTION_MODE
  - android.permission.USE_LOOPBACK_INTERFACE
  - android.permission.SCENE_UNDERSTANDING_FINE
  - android.permission.HAND_TRACKING
  - android.permission.USE_PINNED_WINDOWING_LAYER
--------------------------------------------------------------------------------
PACKAGE: com.android.vending
PATH:    /data/app/com.android.vending-lLHJs6eXxLsgCXqIoSjmOg
UID:     10044
VERSION: 43.0.18-23 [0] [PR] 679685942
PERMISSIONS:
  - android.permission.READ_DEVICE_CONFIG
  - android.permission.WRITE_DEVICE_CONFIG
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.BLUETOOTH_SCAN
  - android.permission.BLUETOOTH_ADVERTISE
  - android.permission.BLUETOOTH_CONNECT
  - android.permission.NEARBY_WIFI_DEVICES
  - android.permission.HIDE_NON_SYSTEM_OVERLAY_WINDOWS
  - android.permission.ACCESS_ADSERVICES_AD_ID
  - android.permission.ACCESS_ADSERVICES_ATTRIBUTION
  - android.permission.RECORD_AUDIO
  - android.permission.SEND_SMS: restricted=true
  - android.permission.SEND_SMS_NO_CONFIRMATION
  - android.permission.RECEIVE_SMS: restricted=true
  - android.permission.READ_SMS: restricted=true
  - android.permission.READ_PHONE_STATE
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.READ_CONTACTS
  - android.permission.FOREGROUND_SERVICE_MICROPHONE
  - android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK
  - android.permission.ACCESS_NOTIFICATION_POLICY
  - android.permission.GET_TASKS
  - android.permission.REAL_GET_TASKS
  - android.permission.INTERNET
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.CLEAR_APP_CACHE
  - android.permission.CHANGE_COMPONENT_ENABLED_STATE
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.READ_PRIVILEGED_PHONE_STATE
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.ACCESS_DOWNLOAD_MANAGER
  - android.permission.ACCESS_DOWNLOAD_MANAGER_ADVANCED
  - android.permission.SEND_DOWNLOAD_COMPLETED_INTENTS
  - android.permission.INSTALL_PACKAGES
  - android.permission.DELETE_PACKAGES
  - android.permission.NFC
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.STATUS_BAR
  - android.permission.WAKE_LOCK
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.USE_FINGERPRINT
  - android.permission.USE_BIOMETRIC
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.AUTHENTICATE_ACCOUNTS
  - android.permission.USE_CREDENTIALS
  - android.permission.GET_ACCOUNTS_PRIVILEGED
  - android.permission.SEND_SAFETY_CENTER_UPDATE
  - android.permission.INSTALL_GRANT_RUNTIME_PERMISSIONS
  - android.permission.GRANT_RUNTIME_PERMISSIONS
  - android.permission.REVOKE_RUNTIME_PERMISSIONS
  - android.permission.PACKAGE_USAGE_STATS
  - android.permission.CHANGE_DEVICE_IDLE_TEMP_WHITELIST
  - android.permission.BATTERY_STATS
  - android.permission.BACKUP
  - android.permission.CONNECTIVITY_INTERNAL
  - android.permission.DUMP
  - android.permission.ALLOCATE_AGGRESSIVE
  - android.permission.GET_APP_OPS_STATS
  - android.permission.SET_PREFERRED_APPLICATIONS
  - android.permission.ACCESS_INSTANT_APPS
  - android.permission.UPDATE_DEVICE_STATS
  - android.permission.MANAGE_APP_OPS_MODES
  - android.permission.MANAGE_ROLLBACKS
  - android.permission.PACKAGE_VERIFICATION_AGENT
  - android.permission.SET_HARMFUL_APP_WARNINGS
  - android.permission.MANAGE_USERS
  - android.permission.INTERACT_ACROSS_USERS
  - android.permission.WRITE_SETTINGS
  - android.permission.WRITE_SECURE_SETTINGS
  - android.permission.FORCE_STOP_PACKAGES
  - android.permission.LOADER_USAGE_STATS
  - android.permission.GET_PACKAGE_SIZE
  - android.permission.READ_APP_BADGE
  - android.permission.SUBSTITUTE_NOTIFICATION_APP_NAME
  - android.permission.READ_RUNTIME_PROFILES
  - android.permission.FOREGROUND_SERVICE
  - android.permission.REBOOT
  - android.permission.START_ACTIVITIES_FROM_BACKGROUND
  - android.permission.VIBRATE
  - android.permission.CHANGE_OVERLAY_PACKAGES
  - android.permission.SEND_DEVICE_CUSTOMIZATION_READY
  - android.permission.QUERY_ALL_PACKAGES
  - android.permission.OVERRIDE_COMPAT_CHANGE_CONFIG_ON_RELEASE_BUILD
  - android.permission.ENFORCE_UPDATE_OWNERSHIP
  - android.permission.LAUNCH_TWO_PANE_SETTINGS_DEEP_LINK
  - android.permission.POST_NOTIFICATIONS
  - android.permission.FOREGROUND_SERVICE_SYSTEM_EXEMPTED
  - android.permission.READ_SYSTEM_GRAMMATICAL_GENDER
  - android.permission.QUERY_ADVANCED_PROTECTION_MODE
  - android.permission.GET_APP_METADATA
  - android.permission.HIDE_STATUS_BAR_NOTIFICATION
  - android.permission.READ_DEVICE_CONFIG
  - android.permission.WRITE_DEVICE_CONFIG
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.BLUETOOTH_SCAN
  - android.permission.BLUETOOTH_ADVERTISE
  - android.permission.BLUETOOTH_CONNECT
  - android.permission.NEARBY_WIFI_DEVICES
  - android.permission.HIDE_NON_SYSTEM_OVERLAY_WINDOWS
  - android.permission.MANAGE_CLOUDSEARCH
  - android.permission.ACCESS_ADSERVICES_AD_ID
  - android.permission.ACCESS_ADSERVICES_ATTRIBUTION
  - android.permission.SEND_SMS: restricted=true
  - android.permission.SEND_SMS_NO_CONFIRMATION
  - android.permission.RECEIVE_SMS: restricted=true
  - android.permission.READ_SMS: restricted=true
  - android.permission.READ_PHONE_STATE
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.READ_CONTACTS
  - android.permission.GET_TASKS
  - android.permission.REAL_GET_TASKS
  - android.permission.INTERNET
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.CLEAR_APP_CACHE
  - android.permission.CHANGE_COMPONENT_ENABLED_STATE
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.READ_PRIVILEGED_PHONE_STATE
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.ACCESS_DOWNLOAD_MANAGER
  - android.permission.ACCESS_DOWNLOAD_MANAGER_ADVANCED
  - android.permission.SEND_DOWNLOAD_COMPLETED_INTENTS
  - android.permission.INSTALL_PACKAGES
  - android.permission.DELETE_PACKAGES
  - android.permission.NFC
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.STATUS_BAR
  - android.permission.WAKE_LOCK
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.USE_FINGERPRINT
  - android.permission.USE_BIOMETRIC
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.AUTHENTICATE_ACCOUNTS
  - android.permission.USE_CREDENTIALS
  - android.permission.GET_ACCOUNTS_PRIVILEGED
  - android.permission.SEND_SAFETY_CENTER_UPDATE
  - android.permission.INSTALL_GRANT_RUNTIME_PERMISSIONS
  - android.permission.GRANT_RUNTIME_PERMISSIONS
  - android.permission.REVOKE_RUNTIME_PERMISSIONS
  - android.permission.PACKAGE_USAGE_STATS
  - android.permission.CHANGE_DEVICE_IDLE_TEMP_WHITELIST
  - android.permission.BATTERY_STATS
  - android.permission.BACKUP
  - android.permission.CONNECTIVITY_INTERNAL
  - android.permission.DUMP
  - android.permission.ALLOCATE_AGGRESSIVE
  - android.permission.GET_APP_OPS_STATS
  - android.permission.SET_PREFERRED_APPLICATIONS
  - android.permission.ACCESS_INSTANT_APPS
  - android.permission.UPDATE_DEVICE_STATS
  - android.permission.MANAGE_APP_OPS_MODES
  - android.permission.MANAGE_ROLLBACKS
  - android.permission.PACKAGE_VERIFICATION_AGENT
  - android.permission.SET_HARMFUL_APP_WARNINGS
  - android.permission.MANAGE_USERS
  - android.permission.INTERACT_ACROSS_USERS
  - android.permission.WRITE_SETTINGS
  - android.permission.WRITE_SECURE_SETTINGS
  - android.permission.FORCE_STOP_PACKAGES
  - android.permission.LOADER_USAGE_STATS
  - android.permission.GET_PACKAGE_SIZE
  - android.permission.READ_APP_BADGE
  - android.permission.SUBSTITUTE_NOTIFICATION_APP_NAME
  - android.permission.READ_RUNTIME_PROFILES
  - android.permission.FOREGROUND_SERVICE
  - android.permission.REBOOT
  - android.permission.START_ACTIVITIES_FROM_BACKGROUND
  - android.permission.VIBRATE
  - android.permission.CHANGE_OVERLAY_PACKAGES
  - android.permission.SEND_DEVICE_CUSTOMIZATION_READY
  - android.permission.QUERY_ALL_PACKAGES
  - android.permission.OVERRIDE_COMPAT_CHANGE_CONFIG_ON_RELEASE_BUILD
  - android.permission.ENFORCE_UPDATE_OWNERSHIP
  - android.permission.LAUNCH_TWO_PANE_SETTINGS_DEEP_LINK
  - android.permission.POST_NOTIFICATIONS
  - android.permission.FOREGROUND_SERVICE_SYSTEM_EXEMPTED
  - android.permission.READ_SYSTEM_GRAMMATICAL_GENDER
--------------------------------------------------------------------------------
PACKAGE: com.anthropic.claude
PATH:    /data/app/com.anthropic.claude-rcx7P28JNrp5WOFab064mQ
UID:     10091
VERSION: 1.260430.10
PERMISSIONS:
  - android.permission.INTERNET
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.VIBRATE
  - android.permission.RECORD_AUDIO
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.READ_CALENDAR
  - android.permission.WRITE_CALENDAR
  - android.permission.DETECT_SCREEN_CAPTURE
  - android.permission.health.READ_STEPS
  - android.permission.health.READ_DISTANCE
  - android.permission.health.READ_ACTIVE_CALORIES_BURNED
  - android.permission.health.READ_TOTAL_CALORIES_BURNED
  - android.permission.health.READ_FLOORS_CLIMBED
  - android.permission.health.READ_ELEVATION_GAINED
  - android.permission.health.READ_WHEELCHAIR_PUSHES
  - android.permission.health.READ_SPEED
  - android.permission.health.READ_STEPS_CADENCE
  - android.permission.health.READ_CYCLING_PEDALING_CADENCE
  - android.permission.health.READ_POWER
  - android.permission.health.READ_EXERCISE
  - android.permission.health.READ_WEIGHT
  - android.permission.health.READ_HEIGHT
  - android.permission.health.READ_BODY_FAT
  - android.permission.health.READ_LEAN_BODY_MASS
  - android.permission.health.READ_BONE_MASS
  - android.permission.health.READ_BODY_WATER_MASS
  - android.permission.health.READ_BASAL_METABOLIC_RATE
  - android.permission.health.READ_HEART_RATE
  - android.permission.health.READ_RESTING_HEART_RATE
  - android.permission.health.READ_HEART_RATE_VARIABILITY
  - android.permission.health.READ_BLOOD_PRESSURE
  - android.permission.health.READ_BLOOD_GLUCOSE
  - android.permission.health.READ_OXYGEN_SATURATION
  - android.permission.health.READ_RESPIRATORY_RATE
  - android.permission.health.READ_BODY_TEMPERATURE
  - android.permission.health.READ_BASAL_BODY_TEMPERATURE
  - android.permission.health.READ_VO2_MAX
  - android.permission.health.READ_SKIN_TEMPERATURE
  - android.permission.health.READ_HYDRATION
  - android.permission.health.READ_NUTRITION
  - android.permission.health.READ_SLEEP
  - android.permission.health.READ_CERVICAL_MUCUS
  - android.permission.health.READ_MENSTRUATION
  - android.permission.health.READ_OVULATION_TEST
  - android.permission.health.READ_INTERMENSTRUAL_BLEEDING
  - android.permission.health.READ_SEXUAL_ACTIVITY
  - android.permission.health.READ_MINDFULNESS
  - android.permission.health.READ_PLANNED_EXERCISE
  - android.permission.health.READ_HEALTH_DATA_HISTORY
  - android.permission.FOREGROUND_SERVICE
  - android.permission.FOREGROUND_SERVICE_MICROPHONE
  - android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK
  - android.permission.POST_NOTIFICATIONS
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.USE_BIOMETRIC
  - android.permission.USE_FINGERPRINT
  - android.permission.WAKE_LOCK
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.ACCESS_ADSERVICES_ATTRIBUTION
--------------------------------------------------------------------------------
PACKAGE: com.arla.bodycamapp
PATH:    /data/app/com.arla.bodycamapp-LkyAFR8XkmGs1zcEFpU3FA
UID:     10086
VERSION: 1.0.0
PERMISSIONS:
  - android.permission.CAMERA
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_MEDIA_IMAGES
  - android.permission.READ_MEDIA_VIDEO
  - android.permission.ACCESS_MEDIA_LOCATION
  - android.permission.RECORD_AUDIO
  - android.permission.WAKE_LOCK
  - android.permission.INTERNET
  - android.permission.ACCESS_NETWORK_STATE
--------------------------------------------------------------------------------
PACKAGE: com.arumcomm.settingsshortcut
PATH:    /data/app/com.arumcomm.settingsshortcut-a5_s2-sGCedyfjfUz_tKLg
UID:     10081
VERSION: 3.1.1
PERMISSIONS:
  - android.permission.POST_NOTIFICATIONS
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.INTERNET
  - android.permission.WAKE_LOCK
  - android.permission.ACCESS_ADSERVICES_AD_ID
  - android.permission.ACCESS_ADSERVICES_ATTRIBUTION
  - android.permission.ACCESS_ADSERVICES_TOPICS
  - android.permission.FOREGROUND_SERVICE
--------------------------------------------------------------------------------
PACKAGE: com.brave.browser
PATH:    /data/app/com.brave.browser-I7Xg1ufIkWkBnOPWr3M9_w
UID:     10057
VERSION: 1.89.145
PERMISSIONS:
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_ADSERVICES_ATTRIBUTION
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.BLUETOOTH_CONNECT
  - android.permission.BLUETOOTH_ADVERTISE
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_SCAN
  - android.permission.REORDER_TASKS
  - android.permission.REQUEST_INSTALL_PACKAGES
  - android.permission.CAMERA
  - android.permission.CAPTURE_KEYBOARD
  - android.permission.DOWNLOAD_WITHOUT_NOTIFICATION
  - android.permission.FOREGROUND_SERVICE
  - android.permission.FOREGROUND_SERVICE_MICROPHONE
  - android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK
  - android.permission.FOREGROUND_SERVICE_DATA_SYNC
  - android.permission.RUN_USER_INITIATED_JOBS
  - android.permission.INTERNET
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.POST_NOTIFICATIONS
  - android.permission.QUERY_ALL_PACKAGES
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_MEDIA_AUDIO
  - android.permission.READ_MEDIA_IMAGES
  - android.permission.READ_MEDIA_VIDEO
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.RECORD_AUDIO
  - android.permission.REPOSITION_SELF_WINDOWS
  - android.permission.USE_BIOMETRIC
  - android.permission.USE_FINGERPRINT
  - android.permission.VIBRATE
  - android.permission.WAKE_LOCK
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.CREDENTIAL_MANAGER_QUERY_CANDIDATE_CREDENTIALS
  - android.permission.CREDENTIAL_MANAGER_SET_ALLOWED_PROVIDERS
  - android.permission.CREDENTIAL_MANAGER_SET_ORIGIN
  - android.permission.QUERY_ADVANCED_PROTECTION_MODE
  - android.permission.USE_LOOPBACK_INTERFACE
  - android.permission.SCENE_UNDERSTANDING_FINE
  - android.permission.HAND_TRACKING
  - android.permission.USE_PINNED_WINDOWING_LAYER
--------------------------------------------------------------------------------
PACKAGE: com.facebook.katana
PATH:    /data/app/com.facebook.katana-LHnrqh9b0H1DK_cVpCn0zQ
UID:     10077
VERSION: 560.0.0.57.63
PERMISSIONS:
  - android.permission.ACCESS_MEDIA_LOCATION
  - android.permission.READ_MEDIA_VISUAL_USER_SELECTED
  - android.permission.READ_MEDIA_VIDEO
  - android.permission.READ_MEDIA_IMAGES
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.CALL_PHONE
  - android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.WAKE_LOCK
  - android.permission.VIBRATE
  - android.permission.READ_CONTACTS
  - android.permission.WRITE_CONTACTS
  - android.permission.GET_ACCOUNTS
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.AUTHENTICATE_ACCOUNTS
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.WRITE_SYNC_SETTINGS
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.BROADCAST_STICKY
  - android.permission.DOWNLOAD_WITHOUT_NOTIFICATION
  - android.permission.NFC
  - android.permission.CAMERA
  - android.permission.POST_NOTIFICATIONS
  - android.permission.RECORD_AUDIO
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_PHONE_NUMBERS
  - android.permission.READ_PHONE_STATE
  - android.permission.READ_BASIC_PHONE_STATE
  - android.permission.READ_CALENDAR
  - android.permission.WRITE_CALENDAR
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.READ_PROFILE
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.GET_TASKS
  - android.permission.BLUETOOTH_CONNECT
  - android.permission.BLUETOOTH
  - android.permission.ACCESS_ADSERVICES_ATTRIBUTION
  - android.permission.ACCESS_ADSERVICES_AD_ID
  - android.permission.ACCESS_ADSERVICES_CUSTOM_AUDIENCE
  - android.permission.ACCESS_ADSERVICES_TOPICS
  - android.permission.CREDENTIAL_MANAGER_SET_ALLOWED_PROVIDERS
  - android.permission.INTERNET
  - android.permission.FOREGROUND_SERVICE
  - android.permission.FOREGROUND_SERVICE_CONNECTED_DEVICE
  - android.permission.DETECT_SCREEN_CAPTURE
  - android.permission.DETECT_SCREEN_RECORDING
  - android.permission.ANSWER_PHONE_CALLS
  - android.permission.READ_CALL_LOG: restricted=true
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.BATTERY_STATS
  - android.permission.USE_FULL_SCREEN_INTENT
  - android.permission.CAPTURE_VIDEO_OUTPUT
  - android.permission.FOREGROUND_SERVICE_PHONE_CALL
  - android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION
  - android.permission.FOREGROUND_SERVICE_CAMERA
  - android.permission.FOREGROUND_SERVICE_MICROPHONE
  - android.permission.REORDER_TASKS
  - android.permission.MANAGE_OWN_CALLS
  - android.permission.FOREGROUND_SERVICE_DATA_SYNC
  - android.permission.RUN_USER_INITIATED_JOBS
  - android.permission.USE_BIOMETRIC
  - android.permission.USE_FINGERPRINT
  - android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK
--------------------------------------------------------------------------------
PACKAGE: com.facebook.orca
PATH:    /data/app/com.facebook.orca-Y3ytsBYxyJ9fH3YQ1xy7-Q
UID:     10078
VERSION: 560.0.0.55.69
PERMISSIONS:
  - android.permission.READ_MEDIA_AUDIO
  - android.permission.READ_MEDIA_VIDEO
  - android.permission.READ_MEDIA_IMAGES
  - android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS
  - android.permission.POST_NOTIFICATIONS
  - android.permission.SCHEDULE_EXACT_ALARM
  - android.permission.INTERNET
  - android.permission.GET_ACCOUNTS
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.WAKE_LOCK
  - android.permission.VIBRATE
  - android.permission.READ_CONTACTS
  - android.permission.READ_PROFILE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_PHONE_NUMBERS
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.RECORD_AUDIO
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.CALL_PHONE
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.DOWNLOAD_WITHOUT_NOTIFICATION
  - android.permission.AUTHENTICATE_ACCOUNTS
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.WRITE_CONTACTS
  - android.permission.WRITE_SYNC_SETTINGS
  - android.permission.READ_MEDIA_VISUAL_USER_SELECTED
  - android.permission.BLUETOOTH
  - android.permission.NFC
  - android.permission.CAMERA
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.USE_FULL_SCREEN_INTENT
  - android.permission.READ_CALENDAR
  - android.permission.WRITE_CALENDAR
  - android.permission.FOREGROUND_SERVICE_SPECIAL_USE
  - android.permission.FOREGROUND_SERVICE_MICROPHONE
  - android.permission.CREDENTIAL_MANAGER_SET_ALLOWED_PROVIDERS
  - android.permission.READ_PHONE_STATE
  - android.permission.READ_BASIC_PHONE_STATE
  - android.permission.FOREGROUND_SERVICE
  - android.permission.FOREGROUND_SERVICE_PHONE_CALL
  - android.permission.BLUETOOTH_CONNECT
  - android.permission.FOREGROUND_SERVICE_CONNECTED_DEVICE
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.USE_BIOMETRIC
  - android.permission.FOREGROUND_SERVICE_LOCATION
  - android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION
  - android.permission.FOREGROUND_SERVICE_CAMERA
  - android.permission.MANAGE_OWN_CALLS
  - android.permission.DETECT_SCREEN_CAPTURE
  - android.permission.DETECT_SCREEN_RECORDING
  - android.permission.USE_FINGERPRINT
  - android.permission.CAPTURE_VIDEO_OUTPUT
--------------------------------------------------------------------------------
PACKAGE: com.google.android.apps.bard
PATH:    /data/app/com.google.android.apps.bard-9sPTNdm5gDOT1mbCkQnBPw
UID:     10070
VERSION: 1.0.882555348
PERMISSIONS:
  - android.permission.WAKE_LOCK
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.GET_PACKAGE_SIZE
--------------------------------------------------------------------------------
PACKAGE: com.google.android.apps.docs
PATH:    /data/app/com.google.android.apps.docs-0IiC2MFtX5U6ntduH_qzKQ
UID:     10069
VERSION: 2.26.177.0.all.alldpi
PERMISSIONS:
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_MEDIA_AUDIO
  - android.permission.CAMERA
  - android.permission.REQUEST_INSTALL_PACKAGES
  - android.permission.GET_PACKAGE_SIZE
  - android.permission.DOWNLOAD_WITHOUT_NOTIFICATION
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.WAKE_LOCK
  - android.permission.INTERNET
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.GET_ACCOUNTS
  - android.permission.USE_CREDENTIALS
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.READ_SYNC_STATS
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.WRITE_SYNC_SETTINGS
  - android.permission.READ_CONTACTS
  - android.permission.SUBSCRIBED_FEEDS_READ
  - android.permission.SUBSCRIBED_FEEDS_WRITE
  - android.permission.VIBRATE
  - android.permission.FOREGROUND_SERVICE
  - android.permission.FOREGROUND_SERVICE_DATA_SYNC
  - android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK
  - android.permission.RUN_USER_INITIATED_JOBS
  - android.permission.POST_NOTIFICATIONS
  - android.permission.NFC
  - android.permission.RECEIVE_BOOT_COMPLETED
--------------------------------------------------------------------------------
PACKAGE: com.google.android.apps.docs.editors.docs
PATH:    /data/app/com.google.android.apps.docs.editors.docs-4dLxw9Rer456bdJqIufKUQ
UID:     10067
VERSION: 1.26.161.04.90
PERMISSIONS:
  - android.permission.INTERNET
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.WAKE_LOCK
  - android.permission.GET_ACCOUNTS
  - android.permission.RECORD_AUDIO
  - android.permission.GET_PACKAGE_SIZE
  - android.permission.USE_CREDENTIALS
  - android.permission.READ_CONTACTS
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.WRITE_SYNC_SETTINGS
  - android.permission.SUBSCRIBED_FEEDS_READ
  - android.permission.SUBSCRIBED_FEEDS_WRITE
  - android.permission.VIBRATE
  - android.permission.FOREGROUND_SERVICE
  - android.permission.FOREGROUND_SERVICE_DATA_SYNC
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.READ_SYNC_STATS
  - android.permission.POST_NOTIFICATIONS
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.RUN_USER_INITIATED_JOBS
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.google.android.apps.magazines
PATH:    /data/app/com.google.android.apps.magazines-DKZcE_XWtl5a_uSM871Q1A
UID:     10068
VERSION: 5.157.0.901298157
PERMISSIONS:
  - android.permission.INTERNET
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.WRITE_SYNC_SETTINGS
  - android.permission.READ_SYNC_STATS
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.POST_NOTIFICATIONS
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.WAKE_LOCK
  - android.permission.FOREGROUND_SERVICE
  - android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.GET_PACKAGE_SIZE
  - android.permission.VIBRATE
--------------------------------------------------------------------------------
PACKAGE: com.google.android.apps.maps
PATH:    /data/app/com.google.android.apps.maps-kPZM08MIiZo3IZwgFoV_Cw
UID:     10055
VERSION: 11.16.3
PERMISSIONS:
  - android.permission.INTERNET
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_MEDIA_IMAGES
  - android.permission.READ_MEDIA_VIDEO
  - android.permission.READ_MEDIA_VISUAL_USER_SELECTED
  - android.permission.ACCESS_MEDIA_LOCATION
  - android.permission.ACCESS_BACKGROUND_LOCATION: restricted=true
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACTIVITY_RECOGNITION
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.USE_CREDENTIALS
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.WRITE_SYNC_SETTINGS
  - android.permission.DISABLE_KEYGUARD
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.DOWNLOAD_WITHOUT_NOTIFICATION
  - android.permission.VIBRATE
  - android.permission.NFC
  - android.permission.READ_CONTACTS
  - android.permission.FOREGROUND_SERVICE
  - android.permission.FOREGROUND_SERVICE_DATA_SYNC
  - android.permission.FOREGROUND_SERVICE_MEDIA_PROCESSING
  - android.permission.RUN_USER_INITIATED_JOBS
  - android.permission.FOREGROUND_SERVICE_LOCATION
  - android.permission.POST_NOTIFICATIONS
  - android.permission.DETECT_SCREEN_CAPTURE
  - android.permission.WAKE_LOCK
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.BLUETOOTH_CONNECT
  - android.permission.BLUETOOTH_SCAN
  - android.permission.BROADCAST_STICKY
  - android.permission.GET_PACKAGE_SIZE
  - android.permission.POST_PROMOTED_NOTIFICATIONS
  - android.permission.SCHEDULE_EXACT_ALARM
  - android.permission.ACCESS_ADSERVICES_ATTRIBUTION
  - android.permission.CAMERA
  - android.permission.RECORD_AUDIO
  - android.permission.INTERNET
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.ACCESS_MEDIA_LOCATION
  - android.permission.ACCESS_BACKGROUND_LOCATION: restricted=true
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACTIVITY_RECOGNITION
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.USE_CREDENTIALS
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.WRITE_SYNC_SETTINGS
  - android.permission.DISABLE_KEYGUARD
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.DOWNLOAD_WITHOUT_NOTIFICATION
  - android.permission.VIBRATE
  - android.permission.NFC
  - android.permission.READ_CONTACTS
  - android.permission.CAMERA
  - android.permission.RECORD_AUDIO
  - android.permission.FOREGROUND_SERVICE
  - android.permission.WAKE_LOCK
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.BLUETOOTH_CONNECT
  - android.permission.BLUETOOTH_SCAN
  - android.permission.BROADCAST_STICKY
  - android.permission.GET_PACKAGE_SIZE
  - android.permission.SCHEDULE_EXACT_ALARM
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.google.android.apps.nbu.files
PATH:    /data/app/com.google.android.apps.nbu.files-4zusdPicSCntz0GArqbrIw
UID:     10073
VERSION: 1.11051.902106514.0-release
PERMISSIONS:
  - android.permission.WAKE_LOCK
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.ACCESS_MEDIA_LOCATION
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.INTERNET
  - android.permission.WRITE_SETTINGS
  - android.permission.FOREGROUND_SERVICE
  - android.permission.FOREGROUND_SERVICE_DATA_SYNC
  - android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK
  - android.permission.PACKAGE_USAGE_STATS
  - android.permission.GET_PACKAGE_SIZE
  - android.permission.REQUEST_DELETE_PACKAGES
  - android.permission.MANAGE_EXTERNAL_STORAGE
  - android.permission.QUERY_ALL_PACKAGES
  - android.permission.POST_NOTIFICATIONS
  - android.permission.USE_BIOMETRIC
  - android.permission.REQUEST_INSTALL_PACKAGES
--------------------------------------------------------------------------------
PACKAGE: com.google.android.apps.photos
PATH:    /data/app/com.google.android.apps.photos-lLsR5Xp2FG5E5nV6VXYepw
UID:     10062
VERSION: 7.74.0.906046556
PERMISSIONS:
  - android.permission.NFC
  - android.permission.INTERNET
  - android.permission.BLUETOOTH
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.READ_CONTACTS
  - android.permission.READ_PROFILE
  - android.permission.READ_PHONE_STATE
  - android.permission.SUBSCRIBED_FEEDS_READ
  - android.permission.SUBSCRIBED_FEEDS_WRITE
  - android.permission.WRITE_SYNC_SETTINGS
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.USE_CREDENTIALS
  - android.permission.DOWNLOAD_WITHOUT_NOTIFICATION
  - android.permission.GET_ACCOUNTS
  - android.permission.FOREGROUND_SERVICE_DATA_SYNC
  - android.permission.READ_CALENDAR
  - android.permission.WRITE_CALENDAR
  - android.permission.CAMERA
  - android.permission.USE_BIOMETRIC
  - android.permission.USE_FINGERPRINT
  - android.permission.ACCESS_MEDIA_LOCATION
  - android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS
  - android.permission.POST_NOTIFICATIONS
  - android.permission.VIBRATE
  - android.permission.GET_PACKAGE_SIZE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.SET_WALLPAPER
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.WAKE_LOCK
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.MANAGE_MEDIA
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.WRITE_SETTINGS
  - android.permission.READ_MEDIA_IMAGES
  - android.permission.READ_MEDIA_VIDEO
  - android.permission.READ_MEDIA_AUDIO
  - android.permission.READ_MEDIA_VISUAL_USER_SELECTED
  - android.permission.FOREGROUND_SERVICE
  - android.permission.FOREGROUND_SERVICE_CONNECTED_DEVICE
  - android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK
  - android.permission.RECORD_AUDIO
  - android.permission.RUN_USER_INITIATED_JOBS
  - android.permission.WRITE_MEDIA_STORAGE
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.google.android.apps.subscriptions.red
PATH:    /data/app/com.google.android.apps.subscriptions.red-TxunaRY3q3-wQ1K6dwTx6g
UID:     10072
VERSION: 1.308.907040899
PERMISSIONS:
  - android.permission.INTERNET
  - android.permission.WAKE_LOCK
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.GET_PACKAGE_SIZE
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.QUERY_ALL_PACKAGES
  - android.permission.POST_NOTIFICATIONS
--------------------------------------------------------------------------------
PACKAGE: com.google.android.apps.translate
PATH:    /data/app/com.google.android.apps.translate-q8jKoeXkSbYJJ2DBw4ZAJw
UID:     10066
VERSION: 10.16.65.908820900.3-release
PERMISSIONS:
  - android.permission.INTERNET
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.RECORD_AUDIO
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.FOREGROUND_SERVICE
  - android.permission.FOREGROUND_SERVICE_MICROPHONE
  - android.permission.POST_NOTIFICATIONS
  - android.permission.FOREGROUND_SERVICE_SPECIAL_USE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.NFC
  - android.permission.GET_PACKAGE_SIZE
  - android.permission.WAKE_LOCK
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.google.android.apps.youtube.music
PATH:    /data/app/com.google.android.apps.youtube.music-iIASUQiDmmXgPkJ2vIWg8A
UID:     10074
VERSION: 9.17.51
PERMISSIONS:
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.BLUETOOTH_SCAN
  - android.permission.BLUETOOTH_ADVERTISE
  - android.permission.BLUETOOTH_CONNECT
  - android.permission.INTERNET
  - android.permission.NFC
  - android.permission.READ_MEDIA_IMAGES
  - android.permission.CAMERA
  - android.permission.READ_MEDIA_AUDIO
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.VIBRATE
  - android.permission.WAKE_LOCK
  - android.permission.FOREGROUND_SERVICE
  - android.permission.FOREGROUND_SERVICE_DATA_SYNC
  - android.permission.RUN_USER_INITIATED_JOBS
  - android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK
  - android.permission.POST_NOTIFICATIONS
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.REQUEST_COMPANION_START_FOREGROUND_SERVICES_FROM_BACKGROUND
  - android.permission.FOREGROUND_SERVICE_CONNECTED_DEVICE
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.RECORD_AUDIO
  - android.permission.ACCESS_COARSE_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.google.android.contactkeys
PATH:    /data/app/com.google.android.contactkeys-v1fxmNMcxj04Q8DfKP9PJQ
UID:     10059
VERSION: 1.355.899514882
PERMISSIONS:
  - android.permission.QUERY_ALL_PACKAGES
  - android.permission.INTERNET
--------------------------------------------------------------------------------
PACKAGE: com.google.android.contacts
PATH:    /data/app/com.google.android.contacts-oZd7a25J0MNxfdPV2iIzgg
UID:     10064
VERSION: 4.78.26.904564657
PERMISSIONS:
  - android.permission.READ_CONTACTS
  - android.permission.WRITE_CONTACTS
  - android.permission.GET_ACCOUNTS
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.CALL_PHONE
  - android.permission.INTERNET
  - android.permission.NFC
  - android.permission.READ_PHONE_NUMBERS
  - android.permission.READ_PHONE_STATE
  - android.permission.WAKE_LOCK
  - android.permission.WRITE_SETTINGS
  - android.permission.USE_CREDENTIALS
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.READ_CALL_LOG: restricted=true
  - android.permission.READ_SMS: restricted=true
  - android.permission.WRITE_SYNC_SETTINGS
  - android.permission.READ_SYNC_STATS
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.FOREGROUND_SERVICE
  - android.permission.HIDE_OVERLAY_WINDOWS
  - android.permission.SET_DEFAULT_ACCOUNT_FOR_CONTACTS
  - android.permission.ACCESS_RCS_USER_CAPABILITY_EXCHANGE
  - android.permission.POST_NOTIFICATIONS
  - android.permission.FOREGROUND_SERVICE_DATA_SYNC
  - android.permission.VIBRATE
--------------------------------------------------------------------------------
PACKAGE: com.google.android.gm
PATH:    /data/app/com.google.android.gm-RffNRaSeW9R-irj3Z-S9-w
UID:     10063
VERSION: 2026.04.27.910758475.Release
PERMISSIONS:
  - android.permission.AUTHENTICATE_ACCOUNTS
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.FOREGROUND_SERVICE
  - android.permission.FOREGROUND_SERVICE_DATA_SYNC
  - android.permission.GET_ACCOUNTS
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.INTERNET
  - android.permission.READ_CONTACTS
  - android.permission.WRITE_CONTACTS
  - android.permission.READ_CALENDAR
  - android.permission.WRITE_CALENDAR
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.READ_SYNC_STATS
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.SUBSCRIBED_FEEDS_READ
  - android.permission.SUBSCRIBED_FEEDS_WRITE
  - android.permission.USE_CREDENTIALS
  - android.permission.VIBRATE
  - android.permission.WAKE_LOCK
  - android.permission.WRITE_SYNC_SETTINGS
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.DOWNLOAD_WITHOUT_NOTIFICATION
  - android.permission.READ_PROFILE
  - android.permission.REQUEST_INSTALL_PACKAGES
  - android.permission.GET_PACKAGE_SIZE
  - android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS
  - android.permission.REQUEST_PASSWORD_COMPLEXITY
  - android.permission.POST_NOTIFICATIONS
  - android.permission.ACCESS_ADSERVICES_AD_ID
  - android.permission.ACCESS_ADSERVICES_ATTRIBUTION
  - android.permission.READ_PHONE_STATE
  - android.permission.ACCESS_NOTIFICATION_POLICY
  - android.permission.CAMERA
  - android.permission.RECORD_AUDIO
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_CONNECT
  - android.permission.ANSWER_PHONE_CALLS
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.CALL_PHONE
  - android.permission.MANAGE_OWN_CALLS
  - android.permission.USE_FULL_SCREEN_INTENT
  - android.permission.FOREGROUND_SERVICE_MICROPHONE
  - android.permission.FOREGROUND_SERVICE_CAMERA
  - android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION
  - android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK
  - android.permission.FOREGROUND_SERVICE_PHONE_CALL
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.google.android.gms
PATH:    /data/app/com.google.android.gms-u4G9IS6E01NedZFpfH4XAg
UID:     10046
VERSION: 21.48.57 (100400-424705839)
PERMISSIONS:
  - android.permission.GET_ACCOUNTS
  - android.permission.INTERNET
  - android.permission.USE_CREDENTIALS
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.OBSERVE_GRANT_REVOKE_PERMISSIONS
  - android.permission.GET_PACKAGE_SIZE
  - android.permission.WRITE_EMBEDDED_SUBSCRIPTIONS
  - android.permission.MANAGE_SOUND_TRIGGER
  - android.permission.FOREGROUND_SERVICE
  - android.permission.START_ACTIVITIES_FROM_BACKGROUND
  - android.permission.QUERY_ALL_PACKAGES
  - android.permission.RENOUNCE_PERMISSIONS
  - android.permission.POST_NOTIFICATIONS
  - android.permission.READ_SYSTEM_GRAMMATICAL_GENDER
  - android.permission.SETUP_FSVERITY
  - android.permission.SUBSCRIBED_FEEDS_READ
  - android.permission.SUBSCRIBED_FEEDS_WRITE
  - android.permission.CAPTURE_VIDEO_OUTPUT
  - android.permission.CAPTURE_AUDIO_OUTPUT
  - android.permission.REMOTE_DISPLAY_PROVIDER
  - android.permission.KILL_BACKGROUND_PROCESSES
  - android.permission.FORCE_STOP_PACKAGES
  - android.permission.ACCESS_FPS_COUNTER
  - android.permission.MANAGE_GAME_ACTIVITY
  - android.permission.MANAGE_GAME_MODE
  - android.permission.OBSERVE_ROLE_HOLDERS
  - android.permission.MODIFY_ADSERVICES_STATE
  - android.permission.ACCESS_ADSERVICES_STATE
  - android.permission.MODIFY_ADSERVICES_STATE_COMPAT
  - android.permission.ACCESS_ADSERVICES_STATE_COMPAT
  - android.permission.UPDATE_PRIVILEGED_AD_ID
  - android.permission.UPDATE_PRIVILEGED_AD_ID_COMPAT
  - android.permission.READ_DROPBOX_DATA
  - android.permission.RECOVERY
  - android.permission.VIBRATE
  - android.permission.READ_DREAM_STATE
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.RECEIVE_DATA_ACTIVITY_CHANGE
  - android.permission.WRITE_SETTINGS
  - android.permission.WRITE_SYNC_SETTINGS
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.AUTHENTICATE_ACCOUNTS
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.NFC
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_PHONE_STATE
  - android.permission.PROVIDE_TRUST_AGENT
  - android.permission.WAKE_LOCK
  - android.permission.DISPATCH_PROVISIONING_MESSAGE
  - android.permission.RECOVER_KEYSTORE
  - android.permission.PRE_FACTORY_RESET
  - android.permission.MANAGE_FACTORY_RESET_PROTECTION
  - android.permission.CHECK_REMOTE_LOCKSCREEN
  - android.permission.CONFIGURE_FACTORY_RESET_PROTECTION
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.RECEIVE_SMS: restricted=true
  - android.permission.READ_PRIVILEGED_PHONE_STATE
  - android.permission.CREDENTIAL_MANAGER_SET_ORIGIN
  - android.permission.SET_BIOMETRIC_DIALOG_ADVANCED
  - android.permission.CREDENTIAL_MANAGER_QUERY_CANDIDATE_CREDENTIALS
  - android.permission.CONNECTIVITY_USE_RESTRICTED_NETWORKS
  - android.permission.ALLOCATE_AGGRESSIVE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_CONTACTS
  - android.permission.USE_FINGERPRINT
  - android.permission.USE_BIOMETRIC
  - android.permission.SEND_SMS_NO_CONFIRMATION
  - android.permission.HIDE_OVERLAY_WINDOWS
  - android.permission.CAMERA
  - android.permission.FLASHLIGHT
  - android.permission.PREPARE_FACTORY_RESET
  - android.permission.PACKAGE_USAGE_STATS
  - android.permission.RECEIVE_MMS: restricted=true
  - android.permission.ACCESS_BACKGROUND_LOCATION: restricted=true
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_CONTEXT_HUB
  - android.permission.ACCESS_LOCATION_EXTRA_COMMANDS
  - android.permission.ACCESS_VIBRATOR_STATE
  - android.permission.LOCATION_BYPASS
  - android.permission.LOCATION_HARDWARE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.GET_APP_OPS_STATS
  - android.permission.TURN_SCREEN_ON
  - android.permission.UPDATE_APP_OPS_STATS
  - android.permission.BROADCAST_CLOSE_SYSTEM_DIALOGS
  - android.permission.INSTALL_LOCATION_TIME_ZONE_PROVIDER_SERVICE
  - android.permission.ACCESS_NOTIFICATION_POLICY
  - android.permission.PROCESS_OUTGOING_CALLS: restricted=true
  - android.permission.SEND_SMS: restricted=true
  - android.permission.BLUETOOTH_CONNECT
  - android.permission.BLUETOOTH_SCAN
  - android.permission.BLUETOOTH_ADVERTISE
  - android.permission.MODIFY_DEFAULT_AUDIO_EFFECTS
  - android.permission.SUBSTITUTE_NOTIFICATION_APP_NAME
  - android.permission.SUBSTITUTE_SHARE_TARGET_APP_NAME_AND_ICON
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.CAPTURE_AUDIO_HOTWORD
  - android.permission.MANAGE_VOICE_KEYPHRASES
  - android.permission.CHANGE_WIFI_MULTICAST_STATE
  - android.permission.COMPANION_APPROVE_WIFI_CONNECTIONS
  - android.permission.UWB
  - android.permission.MANAGE_WIFI_INTERFACES
  - android.permission.MANAGE_WIFI_NETWORK_SELECTION
  - android.permission.NEARBY_WIFI_DEVICES
  - android.permission.USE_FULL_SCREEN_INTENT
  - android.permission.READ_MEDIA_VISUAL_USER_SELECTED
  - android.permission.UWB_PRIVILEGED
  - android.permission.UWB_RANGING
  - android.permission.REQUEST_COMPANION_PROFILE_COMPUTER
  - android.permission.REQUEST_COMPANION_RUN_IN_BACKGROUND
  - android.permission.REQUEST_COMPANION_SELF_MANAGED
  - android.permission.TETHER_PRIVILEGED
  - android.permission.CONTROL_DISPLAY_SATURATION
  - android.permission.GET_ROLE_HOLDERS
  - android.permission.SET_TIME_ZONE
  - android.permission.GET_TASKS
  - android.permission.REAL_GET_TASKS
  - android.permission.READ_CALL_LOG: restricted=true
  - android.permission.READ_SMS: restricted=true
  - android.permission.SET_DEFAULT_ACCOUNT_FOR_CONTACTS
  - android.permission.WRITE_CONTACTS
  - android.permission.READ_PROFILE
  - android.permission.WRITE_PROFILE
  - android.permission.MANAGE_DEVICE_ADMINS
  - android.permission.READ_OEM_UNLOCK_STATE
  - android.permission.ACCESS_NETWORK_CONDITIONS
  - android.permission.SCORE_NETWORKS
  - android.permission.OVERRIDE_WIFI_CONFIG
  - android.permission.MODIFY_PHONE_STATE
  - android.permission.CONTROL_INCALL_EXPERIENCE
  - android.permission.MANAGE_USB
  - android.permission.CALL_PRIVILEGED
  - android.permission.BLUETOOTH_PRIVILEGED
  - android.permission.DISABLE_KEYGUARD
  - android.permission.CALL_PHONE
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.USER_ACTIVITY
  - android.permission.MODIFY_AUDIO_ROUTING
  - android.permission.MODIFY_DAY_NIGHT_MODE
  - android.permission.START_TASKS_FROM_RECENTS
  - android.permission.MANAGE_ACTIVITY_STACKS
  - android.permission.REORDER_TASKS
  - android.permission.ACTIVITY_EMBEDDING
  - android.permission.CAPTURE_SECURE_VIDEO_OUTPUT
  - android.permission.RECORD_AUDIO
  - android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION
  - android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK
  - android.permission.PROVIDE_RESOLVER_RANKER_SERVICE
  - android.permission.READ_LOGS
  - android.permission.DOWNLOAD_WITHOUT_NOTIFICATION
  - android.permission.MODIFY_NETWORK_ACCOUNTING
  - android.permission.READ_MEDIA_AUDIO
  - android.permission.READ_MEDIA_IMAGES
  - android.permission.READ_MEDIA_VIDEO
  - android.permission.ACCESS_MEDIA_LOCATION
  - android.permission.FOREGROUND_SERVICE_DATA_SYNC
  - android.permission.INTENT_FILTER_VERIFICATION_AGENT
  - android.permission.DOMAIN_VERIFICATION_AGENT
  - android.permission.LOCAL_MAC_ADDRESS
  - android.permission.ACCESS_BROADCAST_RESPONSE_STATS
  - android.permission.CHANGE_DEVICE_IDLE_TEMP_WHITELIST
  - android.permission.INTERACT_ACROSS_USERS
  - android.permission.PACKET_KEEPALIVE_OFFLOAD
  - android.permission.SET_LOW_POWER_STANDBY_PORTS
  - android.permission.SCHEDULE_PRIORITIZED_ALARM
  - android.permission.LOG_COMPAT_CHANGE
  - android.permission.READ_COMPAT_CHANGE_CONFIG
  - android.permission.ACCESS_LOCAL_NETWORK
  - android.permission.DETECT_SCREEN_CAPTURE
  - android.permission.READ_WIFI_CREDENTIAL
  - android.permission.FOREGROUND_SERVICE_CONNECTED_DEVICE
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.WRITE_CALL_LOG: restricted=true
  - android.permission.ACTIVITY_RECOGNITION
  - android.permission.MANAGE_SUBSCRIPTION_PLANS
  - android.permission.READ_DEVICE_CONFIG
  - android.permission.WRITE_DEVICE_CONFIG
  - android.permission.MANAGE_TIME_AND_ZONE_DETECTION
  - android.permission.EMERGENCY_INSTALL_PACKAGES
  - android.permission.MANAGE_ROLLBACKS
  - android.permission.REQUEST_DELETE_PACKAGES
  - android.permission.RANGING
  - android.permission.UPDATE_FONTS
  - android.permission.LAUNCH_MULTI_PANE_SETTINGS_DEEP_LINK
  - android.permission.MEDIA_CONTENT_CONTROL
  - android.permission.SEND_SAFETY_CENTER_UPDATE
  - android.permission.FOREGROUND_SERVICE_SYSTEM_EXEMPTED
  - android.permission.FOREGROUND_SERVICE_MICROPHONE
  - android.permission.CALL_AUDIO_INTERCEPTION
  - android.permission.THREAD_NETWORK_PRIVILEGED
  - android.permission.WRITE_VERIFICATION_STATE_E2EE_CONTACT_KEYS
  - android.permission.SATELLITE_COMMUNICATION
  - android.permission.DRAW_MOTION_CUES
  - android.permission.QUERY_ADVANCED_PROTECTION_MODE
  - android.permission.MANAGE_ADVANCED_PROTECTION_MODE
  - android.permission.SUBSCRIBE_TO_KEYGUARD_LOCKED_STATE
  - android.permission.READ_INTRUSION_DETECTION_STATE
  - android.permission.MANAGE_INTRUSION_DETECTION_STATE
  - android.permission.PERFORM_GESTURE_EXCHANGE
  - android.permission.ACCESS_CELL_BROADCAST
  - android.permission.health.READ_HEALTH_DATA_IN_BACKGROUND
  - android.permission.health.READ_HEART_RATE
  - android.permission.BODY_SENSORS
  - android.permission.READ_NEARBY_STREAMING_POLICY
  - android.permission.REGISTER_CALL_PROVIDER
  - android.permission.HIGH_SAMPLING_RATE_SENSORS
  - android.permission.PROVIDE_DEFAULT_ENABLED_CREDENTIAL_SERVICE
  - android.permission.PROVIDE_REMOTE_CREDENTIALS
  - android.permission.BACKUP
  - android.permission.NOTIFY_PENDING_SYSTEM_UPDATE
  - android.permission.RESET_PASSWORD
  - android.permission.LOCK_DEVICE
  - android.permission.CONTROL_KEYGUARD_SECURE_NOTIFICATIONS
  - android.permission.MANAGE_SECURE_LOCK_DEVICE
  - android.permission.HIDE_NON_SYSTEM_OVERLAY_WINDOWS
  - android.permission.GET_ACCOUNTS
  - android.permission.INTERNET
  - android.permission.USE_CREDENTIALS
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.OBSERVE_GRANT_REVOKE_PERMISSIONS
  - android.permission.GET_PACKAGE_SIZE
  - android.permission.MANAGE_SOUND_TRIGGER
  - android.permission.FOREGROUND_SERVICE
  - android.permission.START_ACTIVITIES_FROM_BACKGROUND
  - android.permission.QUERY_ALL_PACKAGES
  - android.permission.RENOUNCE_PERMISSIONS
  - android.permission.INTERACT_ACROSS_USERS
  - android.permission.SCHEDULE_PRIORITIZED_ALARM
  - android.permission.SUBSCRIBED_FEEDS_READ
  - android.permission.SUBSCRIBED_FEEDS_WRITE
  - android.permission.CAPTURE_VIDEO_OUTPUT
  - android.permission.CAPTURE_AUDIO_OUTPUT
  - android.permission.REMOTE_DISPLAY_PROVIDER
  - android.permission.KILL_BACKGROUND_PROCESSES
  - android.permission.RECOVERY
  - android.permission.VIBRATE
  - android.permission.READ_DREAM_STATE
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.RECEIVE_DATA_ACTIVITY_CHANGE
  - android.permission.WRITE_SETTINGS
  - android.permission.WRITE_SYNC_SETTINGS
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.AUTHENTICATE_ACCOUNTS
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.NFC
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_PHONE_STATE
  - android.permission.PROVIDE_TRUST_AGENT
  - android.permission.WAKE_LOCK
  - android.permission.DISPATCH_PROVISIONING_MESSAGE
  - android.permission.RECOVER_KEYSTORE
  - android.permission.PRE_FACTORY_RESET
  - android.permission.MANAGE_FACTORY_RESET_PROTECTION
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.RECEIVE_SMS: restricted=true
  - android.permission.READ_PRIVILEGED_PHONE_STATE
  - android.permission.CONNECTIVITY_USE_RESTRICTED_NETWORKS
  - android.permission.ALLOCATE_AGGRESSIVE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_CONTACTS
  - android.permission.USE_FINGERPRINT
  - android.permission.USE_BIOMETRIC
  - android.permission.SEND_SMS_NO_CONFIRMATION
  - android.permission.CAMERA
  - android.permission.FLASHLIGHT
  - android.permission.PACKAGE_USAGE_STATS
  - android.permission.RECEIVE_MMS: restricted=true
  - android.permission.ACCESS_BACKGROUND_LOCATION: restricted=true
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_CONTEXT_HUB
  - android.permission.LOCATION_HARDWARE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.GET_APP_OPS_STATS
  - android.permission.UPDATE_APP_OPS_STATS
  - android.permission.BROADCAST_CLOSE_SYSTEM_DIALOGS
  - android.permission.INSTALL_LOCATION_TIME_ZONE_PROVIDER_SERVICE
  - android.permission.ACCESS_NOTIFICATION_POLICY
  - android.permission.PROCESS_OUTGOING_CALLS: restricted=true
  - android.permission.SEND_SMS: restricted=true
  - android.permission.BLUETOOTH_CONNECT
  - android.permission.BLUETOOTH_SCAN
  - android.permission.BLUETOOTH_ADVERTISE
  - android.permission.SUBSTITUTE_NOTIFICATION_APP_NAME
  - android.permission.SUBSTITUTE_SHARE_TARGET_APP_NAME_AND_ICON
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.CAPTURE_AUDIO_HOTWORD
  - android.permission.MANAGE_VOICE_KEYPHRASES
  - android.permission.CHANGE_WIFI_MULTICAST_STATE
  - android.permission.COMPANION_APPROVE_WIFI_CONNECTIONS
  - android.permission.UWB
  - android.permission.NEARBY_WIFI_DEVICES
  - android.permission.USE_FULL_SCREEN_INTENT
  - android.permission.UWB_PRIVILEGED
  - android.permission.UWB_RANGING
  - android.permission.TETHER_PRIVILEGED
  - android.permission.CONTROL_DISPLAY_SATURATION
  - android.permission.SET_TIME_ZONE
  - android.permission.GET_TASKS
  - android.permission.REAL_GET_TASKS
  - android.permission.READ_CALL_LOG: restricted=true
  - android.permission.READ_SMS: restricted=true
  - android.permission.WRITE_CONTACTS
  - android.permission.READ_PROFILE
  - android.permission.WRITE_PROFILE
  - android.permission.MANAGE_DEVICE_ADMINS
  - android.permission.READ_OEM_UNLOCK_STATE
  - android.permission.ACCESS_NETWORK_CONDITIONS
  - android.permission.SCORE_NETWORKS
  - android.permission.OVERRIDE_WIFI_CONFIG
  - android.permission.MODIFY_PHONE_STATE
  - android.permission.CONTROL_INCALL_EXPERIENCE
  - android.permission.MANAGE_USB
  - android.permission.CALL_PRIVILEGED
  - android.permission.BLUETOOTH_PRIVILEGED
  - android.permission.DISABLE_KEYGUARD
  - android.permission.CALL_PHONE
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.USER_ACTIVITY
  - android.permission.MODIFY_AUDIO_ROUTING
  - android.permission.MODIFY_DAY_NIGHT_MODE
  - android.permission.START_TASKS_FROM_RECENTS
  - android.permission.MANAGE_ACTIVITY_STACKS
  - android.permission.REORDER_TASKS
  - android.permission.ACTIVITY_EMBEDDING
  - android.permission.CAPTURE_SECURE_VIDEO_OUTPUT
  - android.permission.RECORD_AUDIO
  - android.permission.PROVIDE_RESOLVER_RANKER_SERVICE
  - android.permission.READ_LOGS
  - android.permission.DOWNLOAD_WITHOUT_NOTIFICATION
  - android.permission.MODIFY_NETWORK_ACCOUNTING
  - android.permission.READ_MEDIA_AUDIO
  - android.permission.READ_MEDIA_IMAGES
  - android.permission.READ_MEDIA_VIDEO
  - android.permission.ACCESS_MEDIA_LOCATION
  - android.permission.INTENT_FILTER_VERIFICATION_AGENT
  - android.permission.DOMAIN_VERIFICATION_AGENT
  - android.permission.LOCAL_MAC_ADDRESS
  - android.permission.CHANGE_DEVICE_IDLE_TEMP_WHITELIST
  - android.permission.READ_WIFI_CREDENTIAL
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.WRITE_CALL_LOG: restricted=true
  - android.permission.ACTIVITY_RECOGNITION
  - android.permission.MANAGE_SUBSCRIPTION_PLANS
  - android.permission.READ_DEVICE_CONFIG
  - android.permission.WRITE_DEVICE_CONFIG
  - android.permission.MANAGE_TIME_AND_ZONE_DETECTION
  - android.permission.MANAGE_ROLLBACKS
  - android.permission.REQUEST_DELETE_PACKAGES
  - android.permission.UPDATE_FONTS
  - android.permission.LAUNCH_MULTI_PANE_SETTINGS_DEEP_LINK
  - android.permission.READ_NEARBY_STREAMING_POLICY
  - android.permission.BACKUP
  - android.permission.REGISTER_CALL_PROVIDER
  - android.permission.NOTIFY_PENDING_SYSTEM_UPDATE
  - android.permission.BODY_SENSORS
  - android.permission.HIGH_SAMPLING_RATE_SENSORS
  - android.permission.RESET_PASSWORD
  - android.permission.LOCK_DEVICE
  - android.permission.CONTROL_KEYGUARD_SECURE_NOTIFICATIONS
  - android.permission.HIDE_NON_SYSTEM_OVERLAY_WINDOWS
--------------------------------------------------------------------------------
PACKAGE: com.google.android.googlequicksearchbox
PATH:    /data/app/com.google.android.googlequicksearchbox-iFP93pWPWxhR3O46emu-Pw
UID:     10038
VERSION: 11.21.9.21.arm64
PERMISSIONS:
  - android.permission.READ_CONTACTS
  - android.permission.WRITE_CONTACTS
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.INTERNET
  - android.permission.USE_CREDENTIALS
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.WRITE_SETTINGS
  - android.permission.BROADCAST_STICKY
  - android.permission.MEDIA_CONTENT_CONTROL
  - android.permission.INTERACT_ACROSS_USERS
  - android.permission.INTERACT_ACROSS_PROFILES
  - android.permission.FOREGROUND_SERVICE
  - android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK
  - android.permission.FOREGROUND_SERVICE_DATA_SYNC
  - android.permission.FOREGROUND_SERVICE_CONNECTED_DEVICE
  - android.permission.FOREGROUND_SERVICE_MICROPHONE
  - android.permission.FOREGROUND_SERVICE_CAMERA
  - android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION
  - android.permission.GET_PACKAGE_SIZE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.ACCESS_NOTIFICATION_POLICY
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_MEDIA_VISUAL_USER_SELECTED
  - android.permission.READ_MEDIA_IMAGES
  - android.permission.READ_MEDIA_VIDEO
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.WAKE_LOCK
  - android.permission.READ_PROFILE
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_BACKGROUND_LOCATION: restricted=true
  - android.permission.BLUETOOTH
  - android.permission.CALL_PHONE
  - android.permission.CALL_PRIVILEGED
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.NEARBY_WIFI_DEVICES
  - android.permission.READ_ASSISTANT_APP_SEARCH_DATA
  - android.permission.READ_CALENDAR
  - android.permission.READ_CALL_LOG: restricted=true
  - android.permission.READ_SMS: restricted=true
  - android.permission.READ_PHONE_STATE
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.RECORD_AUDIO
  - android.permission.SEND_SMS: restricted=true
  - android.permission.VIBRATE
  - android.permission.GET_TASKS
  - android.permission.REAL_GET_TASKS
  - android.permission.WRITE_CALENDAR
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_SMS
  - android.permission.CAMERA
  - android.permission.FLASHLIGHT
  - android.permission.DOWNLOAD_WITHOUT_NOTIFICATION
  - android.permission.MANAGE_VOICE_KEYPHRASES
  - android.permission.MANAGE_SOUND_TRIGGER
  - android.permission.SOUND_TRIGGER_RUN_IN_BATTERY_SAVER
  - android.permission.CAPTURE_AUDIO_HOTWORD
  - android.permission.CAPTURE_AUDIO_OUTPUT
  - android.permission.MANAGE_HOTWORD_DETECTION
  - android.permission.STOP_APP_SWITCHES
  - android.permission.SET_WALLPAPER
  - android.permission.SET_WALLPAPER_HINTS
  - android.permission.BIND_APPWIDGET
  - android.permission.USE_FINGERPRINT
  - android.permission.CHANGE_WIFI_MULTICAST_STATE
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.SYSTEM_APPLICATION_OVERLAY
  - android.permission.SET_MEDIA_KEY_LISTENER
  - android.permission.SET_VOLUME_KEY_LONG_PRESS_LISTENER
  - android.permission.MANAGE_USB
  - android.permission.PACKAGE_USAGE_STATS
  - android.permission.START_ACTIVITIES_FROM_BACKGROUND
  - android.permission.ANSWER_PHONE_CALLS
  - android.permission.NFC
  - android.permission.EXPAND_STATUS_BAR
  - android.permission.QUERY_ALL_PACKAGES
  - android.permission.CONTROL_INCALL_EXPERIENCE
  - android.permission.ENTER_CAR_MODE_PRIORITIZED
  - android.permission.WRITE_APN_SETTINGS
  - android.permission.SCHEDULE_EXACT_ALARM
  - android.permission.BLUETOOTH_CONNECT
  - android.permission.BLUETOOTH_SCAN
  - android.permission.SUBSTITUTE_SHARE_TARGET_APP_NAME_AND_ICON
  - android.permission.POST_NOTIFICATIONS
  - android.permission.SUBSCRIBE_TO_KEYGUARD_LOCKED_STATE
  - android.permission.ACCESS_ADSERVICES_ATTRIBUTION
  - android.permission.ACCESS_ADSERVICES_AD_ID
  - android.permission.EXECUTE_APP_ACTION
  - android.permission.EXECUTE_APP_FUNCTIONS
  - android.permission.CAPTURE_MEDIA_OUTPUT
  - android.permission.MODIFY_AUDIO_ROUTING
  - android.permission.DETECT_SCREEN_CAPTURE
  - android.permission.ACCESS_COMPUTER_CONTROL
  - android.permission.MANAGE_ASSISTANT_AUDIO
  - android.permission.POST_PROMOTED_NOTIFICATIONS
  - android.permission.REORDER_TASKS
  - android.permission.GLOBAL_SEARCH
  - android.permission.READ_CONTACTS
  - android.permission.WRITE_CONTACTS
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.INTERNET
  - android.permission.USE_CREDENTIALS
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.WRITE_SETTINGS
  - android.permission.BROADCAST_STICKY
  - android.permission.MEDIA_CONTENT_CONTROL
  - android.permission.INTERACT_ACROSS_USERS
  - android.permission.INTERACT_ACROSS_PROFILES
  - android.permission.FOREGROUND_SERVICE
  - android.permission.GET_PACKAGE_SIZE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.ACCESS_NOTIFICATION_POLICY
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.BLUETOOTH_PRIVILEGED
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.WAKE_LOCK
  - android.permission.READ_PROFILE
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_BACKGROUND_LOCATION: restricted=true
  - android.permission.BLUETOOTH
  - android.permission.CALL_PHONE
  - android.permission.CALL_PRIVILEGED
  - android.permission.GET_ACCOUNTS
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.READ_CALENDAR
  - android.permission.READ_CALL_LOG: restricted=true
  - android.permission.READ_SMS: restricted=true
  - android.permission.READ_PHONE_STATE
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.RECORD_AUDIO
  - android.permission.SEND_SMS: restricted=true
  - android.permission.VIBRATE
  - android.permission.GET_TASKS
  - android.permission.REAL_GET_TASKS
  - android.permission.WRITE_CALENDAR
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_SMS
  - android.permission.CAMERA
  - android.permission.FLASHLIGHT
  - android.permission.DOWNLOAD_WITHOUT_NOTIFICATION
  - android.permission.MANAGE_VOICE_KEYPHRASES
  - android.permission.MANAGE_SOUND_TRIGGER
  - android.permission.CAPTURE_AUDIO_HOTWORD
  - android.permission.STOP_APP_SWITCHES
  - android.permission.SET_WALLPAPER
  - android.permission.SET_WALLPAPER_HINTS
  - android.permission.BIND_APPWIDGET
  - android.permission.USE_FINGERPRINT
  - android.permission.CHANGE_WIFI_MULTICAST_STATE
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.SET_MEDIA_KEY_LISTENER
  - android.permission.SET_VOLUME_KEY_LONG_PRESS_LISTENER
  - android.permission.MANAGE_USB
  - android.permission.USE_FULL_SCREEN_INTENT
  - android.permission.PACKAGE_USAGE_STATS
  - android.permission.START_ACTIVITIES_FROM_BACKGROUND
  - android.permission.ANSWER_PHONE_CALLS
  - android.permission.NFC
  - android.permission.EXPAND_STATUS_BAR
  - android.permission.QUERY_ALL_PACKAGES
  - android.permission.CONTROL_INCALL_EXPERIENCE
  - android.permission.ENTER_CAR_MODE_PRIORITIZED
  - android.permission.WRITE_APN_SETTINGS
--------------------------------------------------------------------------------
PACKAGE: com.google.android.ims
PATH:    /data/app/com.google.android.ims-gxnIY5c11Tjaam0N13UKLQ
UID:     10043
VERSION: 92.0.426919243-carrierservices_20220207_RC00
PERMISSIONS:
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.INTERNET
  - android.permission.WAKE_LOCK
  - android.permission.READ_PRIVILEGED_PHONE_STATE
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.CALL_PHONE
  - android.permission.READ_CALL_LOG: restricted=true
  - android.permission.READ_CONTACTS
  - android.permission.WRITE_CALL_LOG: restricted=true
  - android.permission.ADD_VOICEMAIL
  - android.permission.PROCESS_OUTGOING_CALLS: restricted=true
  - android.permission.RECORD_AUDIO
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.CAMERA
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.PACKET_KEEPALIVE_OFFLOAD
  - android.permission.CONNECTIVITY_USE_RESTRICTED_NETWORKS
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.INTERNET
  - android.permission.WAKE_LOCK
  - android.permission.READ_PRIVILEGED_PHONE_STATE
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.CALL_PHONE
  - android.permission.READ_CALL_LOG: restricted=true
  - android.permission.READ_CONTACTS
  - android.permission.WRITE_CALL_LOG: restricted=true
  - android.permission.ADD_VOICEMAIL
  - android.permission.PROCESS_OUTGOING_CALLS: restricted=true
  - android.permission.RECORD_AUDIO
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.CAMERA
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.PACKET_KEEPALIVE_OFFLOAD
  - android.permission.CONNECTIVITY_USE_RESTRICTED_NETWORKS
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.google.android.inputmethod.latin
PATH:    /data/app/com.google.android.inputmethod.latin-ZjCKbfEaMJn4JfRv-WE78g
UID:     10082
VERSION: 17.2.3.895242737-release-arm64-v8a
PERMISSIONS:
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.GET_ACCOUNTS
  - android.permission.INTERNET
  - android.permission.READ_CONTACTS
  - android.permission.READ_USER_DICTIONARY
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.VIBRATE
  - android.permission.WRITE_USER_DICTIONARY
  - android.permission.GET_PACKAGE_SIZE
  - android.permission.WAKE_LOCK
  - android.permission.READ_MEDIA_IMAGES
  - android.permission.READ_MEDIA_VISUAL_USER_SELECTED
  - android.permission.RECORD_AUDIO
  - android.permission.FOREGROUND_SERVICE
  - android.permission.INTERACT_ACROSS_PROFILES
  - android.permission.CAMERA
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.INJECT_KEY_EVENTS
--------------------------------------------------------------------------------
PACKAGE: com.google.android.safetycore
PATH:    /data/app/com.google.android.safetycore-dF6ypEkNtuAEKFR_kwYe5g
UID:     10058
VERSION: 1.0.895829843
PERMISSIONS:
  - android.permission.INTERNET
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.WAKE_LOCK
  - android.permission.RECEIVE_BOOT_COMPLETED
--------------------------------------------------------------------------------
PACKAGE: com.google.android.youtube
PATH:    /data/app/com.google.android.youtube-DHWmzBMOxY8l1xR-71LyCw
UID:     10061
VERSION: 21.18.164
PERMISSIONS:
  - android.permission.READ_MEDIA_VISUAL_USER_SELECTED
  - android.permission.INTERNET
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_MEDIA_IMAGES
  - android.permission.READ_MEDIA_VIDEO
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.MANAGE_DOCUMENTS
  - android.permission.GET_ACCOUNTS
  - android.permission.WAKE_LOCK
  - android.permission.NFC
  - android.permission.CAMERA
  - android.permission.FOREGROUND_SERVICE_CAMERA
  - android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION
  - android.permission.FOREGROUND_SERVICE_MICROPHONE
  - android.permission.VIBRATE
  - android.permission.GET_PACKAGE_SIZE
  - android.permission.FOREGROUND_SERVICE
  - android.permission.FOREGROUND_SERVICE_DATA_SYNC
  - android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK
  - android.permission.RUN_USER_INITIATED_JOBS
  - android.permission.USE_FINGERPRINT
  - android.permission.USE_BIOMETRIC
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.READ_CONTACTS
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.RECORD_AUDIO
  - android.permission.READ_PHONE_STATE
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.NEARBY_WIFI_DEVICES
  - android.permission.HIGH_SAMPLING_RATE_SENSORS
  - android.permission.POST_NOTIFICATIONS
  - android.permission.ACCESS_ADSERVICES_ATTRIBUTION
  - android.permission.ACCESS_ADSERVICES_AD_ID
  - android.permission.DETECT_SCREEN_CAPTURE
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.lonelycatgames.Xplore
PATH:    /data/app/com.lonelycatgames.Xplore-0pOEyKLCEAGJpPgU3JJrhA
UID:     10084
VERSION: 4.47.25
PERMISSIONS:
  - android.permission.INTERNET
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.CHANGE_WIFI_MULTICAST_STATE
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.USE_FINGERPRINT
  - android.permission.USE_BIOMETRIC
  - android.permission.REQUEST_DELETE_PACKAGES
  - android.permission.VIBRATE
  - android.permission.WAKE_LOCK
  - android.permission.REQUEST_INSTALL_PACKAGES
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.MANAGE_EXTERNAL_STORAGE
  - android.permission.QUERY_ALL_PACKAGES
  - android.permission.POST_NOTIFICATIONS
  - android.permission.FOREGROUND_SERVICE
  - android.permission.FOREGROUND_SERVICE_DATA_SYNC
  - android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK
  - android.permission.ACCESS_ADSERVICES_AD_ID
  - android.permission.ACCESS_ADSERVICES_ATTRIBUTION
  - android.permission.ACCESS_ADSERVICES_TOPICS
--------------------------------------------------------------------------------
PACKAGE: com.osborntech.supercycle
PATH:    /data/app/com.osborntech.supercycle-KbL3UU7TW-jI9U7d3SQuAA
UID:     10090
VERSION: 3.1.15
PERMISSIONS:
  - android.permission.INTERNET
  - android.permission.FOREGROUND_SERVICE
  - android.permission.FOREGROUND_SERVICE_LOCATION
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.BLUETOOTH_CONNECT
  - android.permission.BLUETOOTH_SCAN
  - android.permission.ACTIVITY_RECOGNITION
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.WAKE_LOCK
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.ACCESS_ADSERVICES_ATTRIBUTION
  - android.permission.ACCESS_ADSERVICES_AD_ID
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.SCHEDULE_EXACT_ALARM
  - android.permission.POST_NOTIFICATIONS
--------------------------------------------------------------------------------
PACKAGE: com.squareup.cash
PATH:    /data/app/com.squareup.cash-Wspamf0LMRRigjo5LHNP_g
UID:     10076
VERSION: 5.49.0
PERMISSIONS:
  - android.permission.ACCESS_BACKGROUND_LOCATION: restricted=true
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.BLUETOOTH_ADVERTISE
  - android.permission.BLUETOOTH_CONNECT
  - android.permission.BLUETOOTH_SCAN
  - android.permission.CAMERA
  - android.permission.DETECT_SCREEN_CAPTURE
  - android.permission.INTERNET
  - android.permission.POST_NOTIFICATIONS
  - android.permission.READ_CONTACTS
  - android.permission.READ_PHONE_STATE
  - android.permission.READ_PHONE_NUMBERS
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.NFC
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.USE_FINGERPRINT
  - android.permission.VIBRATE
  - android.permission.GET_ACCOUNTS
  - android.permission.SEND_DOWNLOAD_COMPLETED_INTENTS
  - android.permission.DETECT_SCREEN_RECORDING
  - android.permission.USE_BIOMETRIC
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.WAKE_LOCK
  - android.permission.ACCESS_ADSERVICES_ATTRIBUTION
  - android.permission.ACCESS_ADSERVICES_AD_ID
  - android.permission.FOREGROUND_SERVICE
--------------------------------------------------------------------------------
PACKAGE: com.termux
PATH:    /data/app/com.termux-G-or7QYLTCPKL9GNB9ZKgw
UID:     10080
VERSION: 0.119.0-beta.3
PERMISSIONS:
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.INTERNET
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.MANAGE_EXTERNAL_STORAGE
  - android.permission.WAKE_LOCK
  - android.permission.VIBRATE
  - android.permission.FOREGROUND_SERVICE
  - android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.READ_LOGS
  - android.permission.DUMP
  - android.permission.WRITE_SECURE_SETTINGS
  - android.permission.REQUEST_INSTALL_PACKAGES
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.PACKAGE_USAGE_STATS
  - android.permission.ACCESS_MEDIA_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.teslacoilsw.launcher
PATH:    /data/app/com.teslacoilsw.launcher-m2I_GjM71RXql5RlZsOOjw
UID:     10060
VERSION: 81039 (8.4.7)
PERMISSIONS:
  - android.permission.POST_NOTIFICATIONS
  - android.permission.KILL_BACKGROUND_PROCESSES
  - android.permission.INTERNET
  - android.permission.EXPAND_STATUS_BAR
  - android.permission.WRITE_SECURE_SETTINGS
  - android.permission.STATUS_BAR
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.WRITE_SETTINGS
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_MEDIA_AUDIO
  - android.permission.READ_CONTACTS
  - android.permission.READ_CALENDAR
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_CONNECT
  - android.permission.CALL_PHONE
  - android.permission.SET_WALLPAPER
  - android.permission.VIBRATE
  - android.permission.SET_WALLPAPER_HINTS
  - android.permission.BIND_APPWIDGET
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.REQUEST_DELETE_PACKAGES
  - android.permission.QUERY_ALL_PACKAGES
  - android.permission.READ_BASIC_PHONE_STATE
  - android.permission.WAKE_LOCK
  - android.permission.FOREGROUND_SERVICE
  - android.permission.PACKAGE_USAGE_STATS
--------------------------------------------------------------------------------
PACKAGE: com.teslacoilsw.launcherclientproxy
PATH:    /data/app/com.teslacoilsw.launcherclientproxy-7T-oqtYQSRc2MU0Z-wwVsA
UID:     10079
VERSION: 1.1
PERMISSIONS:
  - android.permission.INTERNET
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.ACCESS_MEDIA_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.tmobile.tuesdays
PATH:    /data/app/com.tmobile.tuesdays-itRiuouvjIlTHwiWu3z8Jw
UID:     10071
VERSION: 10.1.2
PERMISSIONS:
  - android.permission.INTERNET
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.RECORD_AUDIO
  - android.permission.WAKE_LOCK
  - android.permission.DISABLE_KEYGUARD
  - android.permission.USE_FULL_SCREEN_INTENT
  - android.permission.READ_CONTACTS
  - android.permission.WRITE_CONTACTS
  - android.permission.READ_PHONE_STATE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.CAMERA
  - android.permission.FLASHLIGHT
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.BLUETOOTH_SCAN
  - android.permission.POST_NOTIFICATIONS
  - android.permission.READ_PHONE_NUMBERS
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.FOREGROUND_SERVICE
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.FOREGROUND_SERVICE_DATA_SYNC
  - android.permission.VIBRATE
  - android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION
  - android.permission.SCHEDULE_EXACT_ALARM
  - android.permission.READ_PRIVILEGED_PHONE_STATE
  - android.permission.USE_FINGERPRINT
  - android.permission.USE_BIOMETRIC
  - android.permission.DETECT_SCREEN_CAPTURE
  - android.permission.WRITE_EMBEDDED_SUBSCRIPTIONS
  - android.permission.ACCESS_ADSERVICES_ATTRIBUTION
  - android.permission.ACCESS_ADSERVICES_AD_ID
  - android.permission.ACCESS_ADSERVICES_TOPICS
  - android.permission.REORDER_TASKS
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.topjohnwu.magisk
PATH:    /data/app/com.topjohnwu.magisk-bOilnQ2GUMTk0mDNusgT9w
UID:     10056
VERSION: 30.7
PERMISSIONS:
  - android.permission.INTERNET
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.REQUEST_INSTALL_PACKAGES
  - android.permission.HIDE_OVERLAY_WINDOWS
  - android.permission.UPDATE_PACKAGES_WITHOUT_USER_ACTION
  - android.permission.POST_NOTIFICATIONS
  - android.permission.RUN_USER_INITIATED_JOBS
  - android.permission.FOREGROUND_SERVICE
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.QUERY_ALL_PACKAGES
--------------------------------------------------------------------------------
PACKAGE: com.touchtype.swiftkey
PATH:    /data/app/com.touchtype.swiftkey-8lNyyFIAUUnwCqQK1XoAHA
UID:     10083
VERSION: 9.12.27.4
PERMISSIONS:
  - android.permission.WAKE_LOCK
  - android.permission.VIBRATE
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.POST_NOTIFICATIONS
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.INTERNET
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.RECORD_AUDIO
  - android.permission.INTERACT_ACROSS_PROFILES
  - android.permission.READ_MEDIA_IMAGES
  - android.permission.READ_MEDIA_VISUAL_USER_SELECTED
  - android.permission.NFC
  - android.permission.HIGH_SAMPLING_RATE_SENSORS
  - android.permission.CAMERA
--------------------------------------------------------------------------------
PACKAGE: com.weawow
PATH:    /data/app/com.weawow-srf3yaULLIM5LjLuyB9rYQ
UID:     10085
VERSION: 7.1.3
PERMISSIONS:
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_BACKGROUND_LOCATION: restricted=true
  - android.permission.INTERNET
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.FOREGROUND_SERVICE
  - android.permission.SCHEDULE_EXACT_ALARM
  - android.permission.POST_NOTIFICATIONS
  - android.permission.WAKE_LOCK
--------------------------------------------------------------------------------
PACKAGE: org.videolan.vlc
PATH:    /data/app/org.videolan.vlc-fqMyZW9-wN0tdR4rUgm20w
UID:     10075
VERSION: 3.7.0
PERMISSIONS:
  - android.permission.FOREGROUND_SERVICE_SPECIAL_USE
  - android.permission.BLUETOOTH_CONNECT
  - android.permission.MANAGE_EXTERNAL_STORAGE
  - android.permission.VIBRATE
  - android.permission.WRITE_SETTINGS
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.INTERNET
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.RECORD_AUDIO
  - android.permission.FOREGROUND_SERVICE
  - android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK
  - android.permission.FOREGROUND_SERVICE_DATA_SYNC
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.READ_MEDIA_AUDIO
  - android.permission.READ_MEDIA_VIDEO
  - android.permission.READ_MEDIA_IMAGES
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.WAKE_LOCK
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.BLUETOOTH
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.POST_NOTIFICATIONS
--------------------------------------------------------------------------------

# PARTITION: /product
================================================================================
PACKAGE: com.wiite.faceunlocklauncher
PATH:    /product/app/FaceUnlockLauncher/FaceUnlockLauncher.apk
UID:     10052
VERSION: 1.0
--------------------------------------------------------------------------------
PACKAGE: com.google.android.gms.location.history
PATH:    /product/app/GoogleLocationHistory/GoogleLocationHistory.apk
UID:     10050
VERSION: 1.0
PERMISSIONS:
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_BACKGROUND_LOCATION: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.google.android.modulemetadata
PATH:    /product/app/ModuleMetadataGooglePrebuilt/ModuleMetadataGooglePrebuilt.apk
UID:     10051
VERSION: 2020-09-01
--------------------------------------------------------------------------------
PACKAGE: com.wiite.agingtest
PATH:    /product/app/WearAgingTest/WearAgingTest.apk
UID:     1000
VERSION: 1.0
PERMISSIONS:
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.RECORD_AUDIO
  - android.permission.VIBRATE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.FLASHLIGHT
  - android.permission.CAMERA
  - android.permission.ACCESS_MEDIA_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.wiite.wiitestore
PATH:    /product/app/WearAppDialStore/WearAppDialStore.apk
UID:     1000
VERSION: 1.0
PERMISSIONS:
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.INTERNET
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.READ_PHONE_STATE
  - android.permission.WAKE_LOCK
  - android.permission.INSTALL_PACKAGES
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.FOREGROUND_SERVICE
  - android.permission.ACCESS_MEDIA_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.wiite.freezeapp
PATH:    /product/app/WearAppFreeze/WearAppFreeze.apk
UID:     1000
VERSION: 1.0.1
--------------------------------------------------------------------------------
PACKAGE: com.wiite.bluetoothsettnigs
PATH:    /product/app/WearBluetoothSettings/WearBluetoothSettings.apk
UID:     10047
VERSION: 1.0
--------------------------------------------------------------------------------
PACKAGE: com.wiite.breathtraining
PATH:    /product/app/WearBreath/WearBreath.apk
UID:     1000
VERSION: 1.0.2
PERMISSIONS:
  - android.permission.VIBRATE
  - android.permission.BODY_SENSORS
--------------------------------------------------------------------------------
PACKAGE: com.wiite.browser
PATH:    /product/app/WearBrowserLightning/WearBrowserLightning.apk
UID:     10049
VERSION: 4.5.1
PERMISSIONS:
  - android.permission.INTERNET
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.WAKE_LOCK
  - android.permission.REQUEST_INSTALL_PACKAGES
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_BACKGROUND_LOCATION: restricted=true
  - android.permission.ACCESS_MEDIA_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.wiite.mycalender
PATH:    /product/app/WearCalendar/WearCalendar.apk
UID:     1000
VERSION: 1.0
--------------------------------------------------------------------------------
PACKAGE: com.wiite.camera
PATH:    /product/app/WearCamera/WearCamera.apk
UID:     1000
VERSION: 1.0.2
PERMISSIONS:
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.BIND_WALLPAPER
  - android.permission.CAMERA
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.GET_ACCOUNTS
  - android.permission.INTERNET
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.NFC
  - android.permission.READ_CONTACTS
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.RECORD_AUDIO
  - android.permission.SET_WALLPAPER
  - android.permission.SUBSCRIBED_FEEDS_READ
  - android.permission.SUBSCRIBED_FEEDS_WRITE
  - android.permission.USE_CREDENTIALS
  - android.permission.VIBRATE
  - android.permission.WAKE_LOCK
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_SETTINGS
  - android.permission.WRITE_SYNC_SETTINGS
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.ACCESS_BACKGROUND_LOCATION: restricted=true
  - android.permission.ACCESS_MEDIA_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.wiite.cleantask
PATH:    /product/app/WearCleanTaskPro/WearCleanTaskPro.apk
UID:     1000
VERSION: 1.0.3
PERMISSIONS:
  - android.permission.WAKE_LOCK
  - android.permission.GET_TASKS
  - android.permission.FORCE_STOP_PACKAGES
  - android.permission.RECEIVE_BOOT_COMPLETED
--------------------------------------------------------------------------------
PACKAGE: com.android.deskclock
PATH:    /product/app/WearClock/WearClock.apk
UID:     1000
VERSION: 5.0.0
PERMISSIONS:
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.WAKE_LOCK
  - android.permission.VIBRATE
  - android.permission.DISABLE_KEYGUARD
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.CRYPT_KEEPER
  - android.permission.STORAGE_INTERNAL
  - android.permission.USE_FULL_SCREEN_INTENT
--------------------------------------------------------------------------------
PACKAGE: com.wiite.dev
PATH:    /product/app/WearConfig/WearConfig.apk
UID:     1000
VERSION: 1.0
--------------------------------------------------------------------------------
PACKAGE: com.wiite.devicemsg
PATH:    /product/app/WearDevicemsg/WearDevicemsg.apk
UID:     1000
VERSION: 1.0
PERMISSIONS:
  - android.permission.INTERNET
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.READ_PHONE_STATE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.wiite.dbf.androidp_contacts
PATH:    /product/app/WearDialerContacts/WearDialerContacts.apk
UID:     1000
VERSION: 1.0.3
PERMISSIONS:
  - android.permission.READ_PHONE_STATE
  - android.permission.READ_CONTACTS
  - android.permission.READ_CALL_LOG: restricted=true
  - android.permission.CALL_PHONE
  - android.permission.VIBRATE
  - android.permission.WRITE_CALL_LOG: restricted=true
  - android.permission.WRITE_CONTACTS
  - android.permission.MODIFY_PHONE_STATE
  - android.permission.WRITE_SETTINGS
--------------------------------------------------------------------------------
PACKAGE: com.wiite.filemanager
PATH:    /product/app/WearFileManager/WearFileManager.apk
UID:     1000
VERSION: 1.0
PERMISSIONS:
  - android.permission.VIBRATE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_MEDIA_STORAGE
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.DISABLE_KEYGUARD
  - android.permission.MOUNT_UNMOUNT_FILESYSTEMS
  - android.permission.REQUEST_INSTALL_PACKAGES
  - android.permission.WAKE_LOCK
  - android.permission.ACCESS_MEDIA_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.sprots.wiiteer.fitness
PATH:    /product/app/WearFitnessUart/WearFitnessUart.apk
UID:     1000
VERSION: 1.6
PERMISSIONS:
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.INTERNET
  - android.permission.READ_PHONE_STATE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.ACCESS_LOCATION_EXTRA_COMMANDS
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.BODY_SENSORS
  - android.permission.VIBRATE
  - android.permission.WAKE_LOCK
  - android.permission.DISABLE_KEYGUARD
  - android.permission.FOREGROUND_SERVICE
  - android.permission.KILL_BACKGROUND_PROCESSES
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.android.gallery3d
PATH:    /product/app/WearGallerySquare/WearGallerySquare.apk
UID:     1000
VERSION: 6.28.1
PERMISSIONS:
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.SET_WALLPAPER
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.ACCESS_MEDIA_LOCATION
  - android.permission.READ_MEDIA_IMAGES
  - android.permission.READ_MEDIA_VIDEO
  - android.permission.POST_NOTIFICATIONS
  - android.permission.MANAGE_EXTERNAL_STORAGE
  - android.permission.MANAGE_MEDIA
  - android.permission.USE_FINGERPRINT
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.wiite.wearhealthuart
PATH:    /product/app/WearHealthStepUart/WearHealthStepUart.apk
UID:     1000
VERSION: null
PERMISSIONS:
  - android.permission.WRITE_SETTINGS
  - android.permission.RECEIVE_BOOT_COMPLETED
--------------------------------------------------------------------------------
PACKAGE: com.wiite.wiite_heartrate
PATH:    /product/app/WearHeartRateUart/WearHeartRateUart.apk
UID:     1000
VERSION: 1.0.2
PERMISSIONS:
  - android.permission.BODY_SENSORS
  - android.permission.VIBRATE
--------------------------------------------------------------------------------
PACKAGE: com.wiite.dbf.wiitedesktopsettings
PATH:    /product/app/WearHomeSettings/WearHomeSettings.apk
UID:     1000
VERSION: 1.0.2
--------------------------------------------------------------------------------
PACKAGE: com.wiite.home
PATH:    /product/app/WearHomeSportDM101/WearHomeSportDM101.apk
UID:     1000
VERSION: 10
PERMISSIONS:
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.INTERNET
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.READ_CALL_LOG: restricted=true
  - android.permission.VIBRATE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.MOUNT_UNMOUNT_FILESYSTEMS
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.SET_WALLPAPER
  - android.permission.CHANGE_CONFIGURATION
  - android.permission.WRITE_SETTINGS
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.WRITE_SECURE_SETTINGS
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.BIND_APPWIDGET
  - android.permission.ADVANCED_WIDGET_API
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.WAKE_LOCK
  - android.permission.DEVICE_POWER
  - android.permission.INSTALL_PACKAGES
  - android.permission.DELETE_PACKAGES
  - android.permission.READ_PHONE_STATE
--------------------------------------------------------------------------------
PACKAGE: com.wiite.power
PATH:    /product/app/WearInChargingUi/WearInChargingUi.apk
UID:     1000
VERSION: 1.3
PERMISSIONS:
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.SYSTEM_ALERT_WINDOW
--------------------------------------------------------------------------------
PACKAGE: com.wiite.wearlanguagesettings
PATH:    /product/app/WearLanguageSettings/WearLanguageSettings.apk
UID:     1000
VERSION: 1.0
--------------------------------------------------------------------------------
PACKAGE: com.android.inputmethod.latin
PATH:    /product/app/WearLatinIME/WearLatinIME.apk
UID:     10054
VERSION: 10
PERMISSIONS:
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.DOWNLOAD_WITHOUT_NOTIFICATION
  - android.permission.GET_ACCOUNTS
  - android.permission.READ_CONTACTS
  - android.permission.READ_PROFILE
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.READ_SYNC_STATS
  - android.permission.READ_USER_DICTIONARY
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.USE_CREDENTIALS
  - android.permission.VIBRATE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_SYNC_SETTINGS
  - android.permission.WRITE_USER_DICTIONARY
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.ACCESS_MEDIA_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.wiite.dbf.wiitemms
PATH:    /product/app/WearMms/WearMms.apk
UID:     1000
VERSION: 1.0.2
PERMISSIONS:
  - android.permission.RECEIVE_SMS: restricted=true
  - android.permission.READ_SMS: restricted=true
  - android.permission.SEND_SMS: restricted=true
  - android.permission.WRITE_SMS
  - android.permission.CALL_PHONE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_PHONE_STATE
  - android.permission.READ_PHONE_NUMBERS
  - android.permission.READ_CONTACTS
  - android.permission.ACCESS_MEDIA_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.android.music
PATH:    /product/app/WearMusic/WearMusic.apk
UID:     1000
VERSION: 1.2
PERMISSIONS:
  - android.permission.WRITE_SETTINGS
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.WAKE_LOCK
  - android.permission.INTERNET
  - android.permission.READ_PHONE_STATE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.BROADCAST_STICKY
  - android.permission.FOREGROUND_SERVICE
--------------------------------------------------------------------------------
PACKAGE: com.wiite.wear.dy.recorder
PATH:    /product/app/WearRecorder/WearRecorder.apk
UID:     1000
VERSION: 1.0.10
PERMISSIONS:
  - android.permission.RECORD_AUDIO
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.ACCESS_MEDIA_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.wiite.reliabletest
PATH:    /product/app/WearReliableTest/WearReliableTest.apk
UID:     1000
VERSION: 1.3
PERMISSIONS:
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_PRIVILEGED_PHONE_STATE
  - android.permission.VIBRATE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.FLASHLIGHT
  - android.permission.CAMERA
  - android.permission.DISABLE_KEYGUARD
  - android.permission.WAKE_LOCK
  - android.permission.ACCESS_MEDIA_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.wiite.screenrecorder
PATH:    /product/app/WearScreenRecord/WearScreenRecord.apk
UID:     1000
VERSION: 1.0
PERMISSIONS:
  - android.permission.WRITE_INTERNAL_STORAGE
  - android.permission.RECORD_AUDIO
  - android.permission.FOREGROUND_SERVICE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.CAPTURE_AUDIO_OUTPUT
  - android.permission.CAMERA
  - android.permission.INTERNET
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.wiite.wearuart
PATH:    /product/app/WearSdkUart/WearSdkUart.apk
UID:     1000
VERSION: 1.0.2
PERMISSIONS:
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.READ_PHONE_STATE
  - android.permission.READ_SETTINGS
  - android.permission.VIBRATE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.MOUNT_UNMOUNT_FILESYSTEMS
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.LOCAL_MAC_ADDRESS
  - android.permission.DISABLE_KEYGUARD
  - android.permission.WAKE_LOCK
  - android.permission.DEVICE_POWER
  - android.permission.WRITE_SETTINGS
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.ACCESS_BACKGROUND_LOCATION: restricted=true
  - android.permission.ACCESS_MEDIA_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.wiite.wiite_sleep
PATH:    /product/app/WearSleepUart/WearSleepUart.apk
UID:     1000
VERSION: 1.0.2
--------------------------------------------------------------------------------
PACKAGE: com.wiite.wiite_bloodoxygen
PATH:    /product/app/WearSpoUart/WearSpoUart.apk
UID:     1000
VERSION: 1.0.3
PERMISSIONS:
  - android.permission.BODY_SENSORS
  - android.permission.VIBRATE
--------------------------------------------------------------------------------
PACKAGE: com.wiite.bluetoothvcard
PATH:    /product/app/WearSyncContact/WearSyncContact.apk
UID:     1000
VERSION: 1.0
PERMISSIONS:
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.ACCESS_MEDIA_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.wiite.systemtool
PATH:    /product/app/WearSystemTool/WearSystemTool.apk
UID:     10048
VERSION: 1.0.2
--------------------------------------------------------------------------------
PACKAGE: com.wiite.wearthirdappadater
PATH:    /product/app/WearThirdPartAppAdapter/WearThirdPartAppAdapter.apk
UID:     1000
VERSION: 1.0.2
PERMISSIONS:
  - android.permission.GET_TASKS
  - android.permission.FORCE_STOP_PACKAGES
--------------------------------------------------------------------------------
PACKAGE: com.android.watchweather
PATH:    /product/app/WearWeather/WearWeather.apk
UID:     1000
VERSION: 1.1
PERMISSIONS:
  - android.permission.ACCESS_BACKGROUND_LOCATION: restricted=true
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.READ_PHONE_STATE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.INTERNET
  - android.permission.MOUNT_UNMOUNT_FILESYSTEMS
  - android.permission.READ_LOGS
  - android.permission.VIBRATE
  - android.permission.WAKE_LOCK
  - android.permission.WRITE_SETTINGS
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.android.webview
PATH:    /product/app/webview/webview.apk
UID:     10053
VERSION: 74.0.3729.186
PERMISSIONS:
  - android.permission.INTERNET
  - android.permission.ACCESS_NETWORK_STATE
--------------------------------------------------------------------------------
PACKAGE: com.google.android.overlay.gmsconfig
PATH:    /product/overlay/GmsConfigOverlay.apk
UID:     10007
VERSION: 1.0
--------------------------------------------------------------------------------
PACKAGE: com.google.android.overlay.gms.documentsui
PATH:    /product/overlay/GmsGoogleDocumentsUIOverlay.apk
UID:     10006
VERSION: 1.0
--------------------------------------------------------------------------------
PACKAGE: com.google.android.overlay.gmsgsaconfig
PATH:    /product/overlay/GmsGsaConfigOverlay.apk
UID:     10004
VERSION: 1.0
--------------------------------------------------------------------------------
PACKAGE: com.google.android.overlay.modules.ext.services
PATH:    /product/overlay/GoogleExtServicesConfigOverlay.apk
UID:     10010
VERSION: 1.0
--------------------------------------------------------------------------------
PACKAGE: com.wiite.home.theme.dm101
PATH:    /product/overlay/HomeThemeDM101/HomeThemeDM101.apk
UID:     10008
VERSION: 1.0
--------------------------------------------------------------------------------
PACKAGE: com.wiite.theme.a.sdk.dm101
PATH:    /product/overlay/SdkDM101/SdkDM101.apk
UID:     10002
VERSION: 1.0
--------------------------------------------------------------------------------
PACKAGE: com.wiitetech.WiiWatchPro.overlay
PATH:    /product/overlay/WearSdkIconOverlay/WearSdkIconOverlay.apk
UID:     10005
VERSION: 1.0
--------------------------------------------------------------------------------
PACKAGE: com.wiiteer.wear.overlay
PATH:    /product/overlay/WearSdkIconOverlay_Wiiwear/WearSdkIconOverlay_Wiiwear.apk
UID:     10003
VERSION: 1.0
--------------------------------------------------------------------------------
PACKAGE: android.auto_generated_rro_product__
PATH:    /product/overlay/framework-res__auto_generated_rro_product.apk
UID:     10009
VERSION: 1.0
--------------------------------------------------------------------------------
PACKAGE: com.adups.fota
PATH:    /product/priv-app/AdupsFota/AdupsFota.apk
UID:     10041
VERSION: 5.28
PERMISSIONS:
  - android.permission.VIBRATE
  - android.permission.INTERNET
  - android.permission.WAKE_LOCK
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.FOREGROUND_SERVICE
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.REBOOT
  - android.permission.RECOVERY
  - android.permission.DELETE_CACHE_FILES
  - android.permission.WRITE_SECURE_SETTINGS
  - android.permission.ACCESS_CACHE_FILESYSTEM
  - android.permission.CHANGE_OVERLAY_PACKAGES
  - android.permission.READ_PRIVILEGED_PHONE_STATE
  - android.permission.ACCESS_MEDIA_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.google.android.configupdater
PATH:    /product/priv-app/ConfigUpdater/ConfigUpdater.apk
UID:     10035
VERSION: 10
PERMISSIONS:
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.INTERNET
  - android.permission.DOWNLOAD_WITHOUT_NOTIFICATION
  - android.permission.UPDATE_CONFIG
--------------------------------------------------------------------------------
PACKAGE: com.google.android.onetimeinitializer
PATH:    /product/priv-app/GoogleOneTimeInitializer/GoogleOneTimeInitializer.apk
UID:     10040
VERSION: 10
PERMISSIONS:
  - android.permission.CHANGE_COMPONENT_ENABLED_STATE
  - android.permission.RECEIVE_BOOT_COMPLETED
--------------------------------------------------------------------------------
PACKAGE: com.google.android.gsf
PATH:    /product/priv-app/GoogleServicesFramework/GoogleServicesFramework.apk
UID:     10046
VERSION: 10-6475670
PERMISSIONS:
  - android.permission.ACCESS_CACHE_FILESYSTEM
  - android.permission.ACCESS_DOWNLOAD_MANAGER
  - android.permission.ACCESS_DOWNLOAD_MANAGER_ADVANCED
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.AUTHENTICATE_ACCOUNTS
  - android.permission.BACKUP
  - android.permission.BROADCAST_STICKY
  - android.permission.CHANGE_COMPONENT_ENABLED_STATE
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.DOWNLOAD_WITHOUT_NOTIFICATION
  - android.permission.DUMP
  - android.permission.GET_ACCOUNTS
  - android.permission.INTERNET
  - android.permission.INVOKE_CARRIER_SETUP
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.MASTER_CLEAR
  - android.permission.PERSISTENT_ACTIVITY
  - android.permission.READ_CONTACTS
  - android.permission.READ_DREAM_STATE
  - android.permission.READ_LOGS
  - android.permission.READ_NETWORK_USAGE_HISTORY
  - android.permission.READ_PHONE_STATE
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.READ_SYNC_STATS
  - android.permission.REBOOT
  - android.permission.RECOVERY
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.RECEIVE_DATA_ACTIVITY_CHANGE
  - android.permission.SET_TIME
  - android.permission.STATUS_BAR
  - android.permission.UPDATE_DEVICE_STATS
  - android.permission.USE_CREDENTIALS
  - android.permission.VIBRATE
  - android.permission.WAKE_LOCK
  - android.permission.WRITE_CONTACTS
  - android.permission.WRITE_GSERVICES
  - android.permission.WRITE_SECURE_SETTINGS
  - android.permission.WRITE_SETTINGS
  - android.permission.WRITE_SYNC_SETTINGS
  - android.permission.INTERACT_ACROSS_USERS
  - android.permission.MANAGE_USERS
--------------------------------------------------------------------------------
PACKAGE: com.android.carrierconfig
PATH:    /product/priv-app/MtkCarrierConfig/MtkCarrierConfig.apk
UID:     10037
VERSION: 1.0.0
PERMISSIONS:
  - android.permission.READ_PRIVILEGED_PHONE_STATE
--------------------------------------------------------------------------------
PACKAGE: com.mediatek.omacp
PATH:    /product/priv-app/Omacp/Omacp.apk
UID:     1000
VERSION: 10
PERMISSIONS:
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.VIBRATE
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.READ_PHONE_STATE
  - android.permission.WAKE_LOCK
  - android.permission.RECEIVE_WAP_PUSH: restricted=true
  - android.permission.FOREGROUND_SERVICE
--------------------------------------------------------------------------------
PACKAGE: com.android.storagemanager
PATH:    /product/priv-app/StorageManager/StorageManager.apk
UID:     10042
VERSION: 10
PERMISSIONS:
  - android.permission.PACKAGE_USAGE_STATS
  - android.permission.GET_PACKAGE_SIZE
  - android.permission.DELETE_PACKAGES
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.MANAGE_USERS
  - android.permission.WRITE_SECURE_SETTINGS
  - android.permission.INTERACT_ACROSS_USERS
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.USE_RESERVED_DISK
--------------------------------------------------------------------------------
PACKAGE: com.wiite.WiiteCommon
PATH:    /product/priv-app/WearCommon/WearCommon.apk
UID:     1000
VERSION: 1.1
PERMISSIONS:
  - android.permission.WRITE_SETTINGS
  - android.permission.REBOOT
  - android.permission.GET_TASKS
  - android.permission.FORCE_STOP_PACKAGES
--------------------------------------------------------------------------------
PACKAGE: com.wiite.devicedaemon
PATH:    /product/priv-app/WearDeviceDeamPix/WearDeviceDeamPix.apk
UID:     1000
VERSION: 1.0.2
PERMISSIONS:
  - android.permission.FOREGROUND_SERVICE
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.READ_PHONE_STATE
  - android.permission.READ_SETTINGS
  - android.permission.VIBRATE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.DISABLE_KEYGUARD
  - android.permission.WAKE_LOCK
  - android.permission.DEVICE_POWER
  - android.permission.WRITE_SETTINGS
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.SHUTDOWN
  - android.permission.CHANGE_CONFIGURATION
  - android.permission.PEERS_MAC_ADDRESS
  - android.permission.ACCESS_BACKGROUND_LOCATION: restricted=true
  - android.permission.ACCESS_MEDIA_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.android.contacts
PATH:    /product/priv-app/WearMtkContacts/WearMtkContacts.apk
UID:     10036
VERSION: 1.7.31
PERMISSIONS:
  - android.permission.READ_CONTACTS
  - android.permission.WRITE_CONTACTS
  - android.permission.GET_ACCOUNTS
  - android.permission.GET_ACCOUNTS_PRIVILEGED
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.CALL_PHONE
  - android.permission.READ_PROFILE
  - android.permission.WRITE_PROFILE
  - android.permission.INTERNET
  - android.permission.NFC
  - android.permission.READ_PHONE_STATE
  - android.permission.WAKE_LOCK
  - android.permission.WRITE_SETTINGS
  - android.permission.USE_CREDENTIALS
  - android.permission.VIBRATE
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_SYNC_SETTINGS
  - android.permission.READ_SYNC_STATS
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.FOREGROUND_SERVICE
  - android.permission.CALL_PRIVILEGED
  - android.permission.BLUETOOTH
  - android.permission.MODIFY_PHONE_STATE
  - android.permission.PROCESS_OUTGOING_CALLS: restricted=true
  - android.permission.ACCESS_MEDIA_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.android.dialer
PATH:    /product/priv-app/WearMtkDialerIncallUi/WearMtkDialerIncallUi.apk
UID:     10039
VERSION: 23.0
PERMISSIONS:
  - android.permission.CALL_PHONE
  - android.permission.READ_CONTACTS
  - android.permission.WRITE_CONTACTS
  - android.permission.READ_CALL_LOG: restricted=true
  - android.permission.WRITE_CALL_LOG: restricted=true
  - android.permission.READ_PROFILE
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.GET_ACCOUNTS
  - android.permission.GET_ACCOUNTS_PRIVILEGED
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.INTERNET
  - android.permission.PROCESS_OUTGOING_CALLS: restricted=true
  - android.permission.NFC
  - android.permission.READ_PHONE_STATE
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.MODIFY_PHONE_STATE
  - android.permission.WAKE_LOCK
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_SETTINGS
  - android.permission.USE_CREDENTIALS
  - android.permission.VIBRATE
  - android.permission.BLUETOOTH
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.ALLOW_ANY_CODEC_FOR_PLAYBACK
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.BROADCAST_STICKY
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.SEND_SMS: restricted=true
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.CONNECTIVITY_USE_RESTRICTED_NETWORKS
  - android.permission.USE_FULL_SCREEN_INTENT
  - android.permission.READ_PRIVILEGED_PHONE_STATE
  - android.permission.CONTROL_INCALL_EXPERIENCE
  - android.permission.STATUS_BAR
  - android.permission.CAMERA
  - android.permission.STOP_APP_SWITCHES
  - android.permission.READ_APP_BADGE
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_SPEED_DIAL
  - android.permission.MANAGE_USERS
  - android.permission.CREATE_USERS
  - android.permission.FOREGROUND_SERVICE
  - android.permission.RECORD_AUDIO
  - android.permission.INTERACT_ACROSS_USERS
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_BACKGROUND_LOCATION: restricted=true
  - android.permission.ACCESS_MEDIA_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.ss.oobe
PATH:    /product/priv-app/WearOOBE/WearOOBE.apk
UID:     1000
VERSION: 1.0
PERMISSIONS:
  - android.permission.WRITE_SETTINGS
  - android.permission.WRITE_SECURE_SETTINGS
  - android.permission.CHANGE_CONFIGURATION
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.READ_CONTACTS
  - android.permission.WRITE_CONTACTS
  - android.permission.WAKE_LOCK
  - android.permission.DISABLE_KEYGUARD
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.READ_SYNC_STATS
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.WRITE_SYNC_SETTINGS
  - android.permission.GET_ACCOUNTS
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.WRITE_APN_SETTINGS
  - android.permission.READ_PHONE_STATE
  - android.permission.SET_TIME_ZONE
  - android.permission.STATUS_BAR
  - android.permission.SET_ORIENTATION
  - android.permission.REBOOT
  - android.permission.DEVICE_POWER
--------------------------------------------------------------------------------
PACKAGE: com.android.settings
PATH:    /product/priv-app/WearSettings/WearSettings.apk
UID:     1000
VERSION: 10
PERMISSIONS:
  - android.permission.REQUEST_NETWORK_SCORES
  - android.permission.WRITE_MEDIA_STORAGE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_SETTINGS
  - android.permission.WRITE_SECURE_SETTINGS
  - android.permission.DEVICE_POWER
  - android.permission.CHANGE_CONFIGURATION
  - android.permission.MOUNT_UNMOUNT_FILESYSTEMS
  - android.permission.VIBRATE
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.BLUETOOTH_PRIVILEGED
  - android.permission.NFC
  - android.permission.HARDWARE_TEST
  - android.permission.CALL_PHONE
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.MASTER_CLEAR
  - android.permission.ACCESS_DOWNLOAD_MANAGER
  - android.permission.READ_CONTACTS
  - android.permission.WRITE_CONTACTS
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.LOCAL_MAC_ADDRESS
  - android.permission.ACCESS_WIMAX_STATE
  - android.permission.CHANGE_WIMAX_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.TETHER_PRIVILEGED
  - android.permission.FOREGROUND_SERVICE
  - android.permission.INTERNET
  - android.permission.CLEAR_APP_USER_DATA
  - android.permission.READ_PHONE_STATE
  - android.permission.MODIFY_PHONE_STATE
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.WRITE_APN_SETTINGS
  - android.permission.ACCESS_CHECKIN_PROPERTIES
  - android.permission.READ_USER_DICTIONARY
  - android.permission.WRITE_USER_DICTIONARY
  - android.permission.FORCE_STOP_PACKAGES
  - android.permission.PACKAGE_USAGE_STATS
  - android.permission.BATTERY_STATS
  - android.permission.MOVE_PACKAGE
  - android.permission.USE_CREDENTIALS
  - android.permission.BACKUP
  - android.permission.READ_SYNC_STATS
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.WRITE_SYNC_SETTINGS
  - android.permission.READ_DEVICE_CONFIG
  - android.permission.STATUS_BAR
  - android.permission.MANAGE_USB
  - android.permission.MANAGE_DEBUGGING
  - android.permission.SET_POINTER_SPEED
  - android.permission.SET_KEYBOARD_LAYOUT
  - android.permission.INTERACT_ACROSS_USERS_FULL
  - android.permission.COPY_PROTECTED_DATA
  - android.permission.MANAGE_USERS
  - android.permission.MANAGE_PROFILE_AND_DEVICE_OWNERS
  - android.permission.READ_PROFILE
  - android.permission.CONFIGURE_WIFI_DISPLAY
  - android.permission.CONFIGURE_DISPLAY_COLOR_MODE
  - android.permission.CONTROL_DISPLAY_COLOR_TRANSFORMS
  - android.permission.SET_TIME
  - android.permission.ACCESS_NOTIFICATIONS
  - android.permission.REBOOT
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.MANAGE_DEVICE_ADMINS
  - android.permission.READ_SEARCH_INDEXABLES
  - android.permission.BIND_SETTINGS_SUGGESTIONS_SERVICE
  - android.permission.OEM_UNLOCK_STATE
  - android.permission.MANAGE_USER_OEM_UNLOCK_STATE
  - android.permission.OVERRIDE_WIFI_CONFIG
  - android.permission.USE_FINGERPRINT
  - android.permission.MANAGE_FINGERPRINT
  - android.permission.USE_BIOMETRIC
  - android.permission.USE_BIOMETRIC_INTERNAL
  - android.permission.USER_ACTIVITY
  - android.permission.CHANGE_APP_IDLE_STATE
  - android.permission.PEERS_MAC_ADDRESS
  - android.permission.MANAGE_NOTIFICATIONS
  - android.permission.DELETE_PACKAGES
  - android.permission.REQUEST_DELETE_PACKAGES
  - android.permission.MANAGE_APP_OPS_RESTRICTIONS
  - android.permission.MANAGE_APP_OPS_MODES
  - android.permission.HIDE_NON_SYSTEM_OVERLAY_WINDOWS
  - android.permission.READ_PRINT_SERVICES
  - android.permission.NETWORK_SETTINGS
  - android.permission.TEST_BLACKLISTED_PASSWORD
  - android.permission.USE_RESERVED_DISK
  - android.permission.MANAGE_SCOPED_ACCESS_DIRECTORY_PERMISSIONS
  - android.permission.CAMERA
  - android.permission.MEDIA_CONTENT_CONTROL
  - android.permission.ACCESS_COARSE_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.wiite.systemmode
PATH:    /product/priv-app/WearSystemMode/WearSystemMode.apk
UID:     1000
VERSION: 10
--------------------------------------------------------------------------------
PACKAGE: com.android.systemui
PATH:    /product/priv-app/WearSystemUIWithLegacyRecents/WearSystemUIWithLegacyRecents.apk
UID:     10045
VERSION: 10
PERMISSIONS:
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_MEDIA_STORAGE
  - android.permission.WAKE_LOCK
  - android.permission.INJECT_EVENTS
  - android.permission.DUMP
  - android.permission.WRITE_SETTINGS
  - android.permission.READ_DEVICE_CONFIG
  - android.permission.STATUS_BAR_SERVICE
  - android.permission.STATUS_BAR
  - android.permission.EXPAND_STATUS_BAR
  - android.permission.REMOTE_AUDIO_PLAYBACK
  - android.permission.MANAGE_USERS
  - android.permission.READ_PROFILE
  - android.permission.READ_CONTACTS
  - android.permission.CONFIGURE_WIFI_DISPLAY
  - android.permission.WRITE_SECURE_SETTINGS
  - android.permission.GET_APP_OPS_STATS
  - android.permission.USE_RESERVED_DISK
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.BLUETOOTH_PRIVILEGED
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.READ_PRIVILEGED_PHONE_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.OVERRIDE_WIFI_CONFIG
  - android.permission.MANAGE_NETWORK_POLICY
  - android.permission.CONNECTIVITY_INTERNAL
  - android.permission.NETWORK_SETTINGS
  - android.permission.TETHER_PRIVILEGED
  - android.permission.READ_NETWORK_USAGE_HISTORY
  - android.permission.REQUEST_NETWORK_SCORES
  - android.permission.CONTROL_VPN
  - android.permission.PEERS_MAC_ADDRESS
  - android.permission.MANAGE_USB
  - android.permission.CONTROL_DISPLAY_BRIGHTNESS
  - android.permission.DEVICE_POWER
  - android.permission.MOUNT_UNMOUNT_FILESYSTEMS
  - android.permission.MASTER_CLEAR
  - android.permission.VIBRATE
  - android.permission.MANAGE_SENSOR_PRIVACY
  - android.permission.REAL_GET_TASKS
  - android.permission.GET_DETAILED_TASKS
  - android.permission.REORDER_TASKS
  - android.permission.REMOVE_TASKS
  - android.permission.STOP_APP_SWITCHES
  - android.permission.SET_SCREEN_COMPATIBILITY
  - android.permission.START_ANY_ACTIVITY
  - android.permission.START_ACTIVITIES_FROM_BACKGROUND
  - android.permission.INTERACT_ACROSS_USERS
  - android.permission.INTERACT_ACROSS_USERS_FULL
  - android.permission.GET_TOP_ACTIVITY_INFO
  - android.permission.MANAGE_ACTIVITY_STACKS
  - android.permission.START_ACTIVITY_AS_CALLER
  - android.permission.START_TASKS_FROM_RECENTS
  - android.permission.GET_INTENT_SENDER_INTENT
  - android.permission.INTERNAL_SYSTEM_WINDOW
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.READ_FRAME_BUFFER
  - android.permission.MANAGE_APP_TOKENS
  - android.permission.REGISTER_WINDOW_MANAGER_LISTENERS
  - android.permission.SET_ORIENTATION
  - android.permission.DISABLE_KEYGUARD
  - android.permission.MONITOR_INPUT
  - android.permission.READ_DREAM_STATE
  - android.permission.WRITE_DREAM_STATE
  - android.permission.WRITE_EMBEDDED_SUBSCRIPTIONS
  - android.permission.CONTROL_KEYGUARD
  - android.permission.MODIFY_PHONE_STATE
  - android.permission.GET_ACCOUNTS
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.BIND_DEVICE_ADMIN
  - android.permission.CHANGE_COMPONENT_ENABLED_STATE
  - android.permission.MEDIA_CONTENT_CONTROL
  - android.permission.ACCESS_KEYGUARD_SECURE_STORAGE
  - android.permission.TRUST_LISTENER
  - android.permission.USE_BIOMETRIC_INTERNAL
  - android.permission.USE_FINGERPRINT
  - android.permission.RESET_FINGERPRINT_LOCKOUT
  - android.permission.MANAGE_BIOMETRIC
  - android.permission.MANAGE_SLICE_PERMISSIONS
  - android.permission.CONTROL_KEYGUARD_SECURE_NOTIFICATIONS
  - android.permission.GET_RUNTIME_PERMISSIONS
  - android.permission.SET_WALLPAPER
  - android.permission.CAMERA
  - android.permission.MANAGE_MEDIA_PROJECTION
  - android.permission.FOREGROUND_SERVICE
  - android.permission.RECORD_AUDIO
  - android.permission.ACCESS_VOICE_INTERACTION_SERVICE
  - android.permission.CHANGE_DEVICE_IDLE_TEMP_WHITELIST
  - android.permission.TABLET_MODE
  - android.permission.BIND_QUICK_SETTINGS_TILE
  - android.permission.MODIFY_DAY_NIGHT_MODE
  - android.permission.UPDATE_APP_OPS_STATS
  - android.permission.BATTERY_STATS
  - android.permission.MANAGE_PROFILE_AND_DEVICE_OWNERS
  - android.permission.RECEIVE_MEDIA_RESOURCE_USAGE
  - android.permission.MANAGE_NOTIFICATIONS
  - android.permission.ACCESS_VR_MANAGER
  - android.permission.SUBSTITUTE_NOTIFICATION_APP_NAME
  - android.permission.RESET_SHORTCUT_MANAGER_THROTTLING
  - android.permission.MODIFY_THEME_OVERLAY
  - android.permission.MODIFY_ACCESSIBILITY_DATA
  - android.permission.CHANGE_ACCESSIBILITY_VOLUME
  - android.permission.BIND_RESOLVER_RANKER_SERVICE
  - android.permission.ACCESS_INSTANT_APPS
  - android.permission.CONTROL_REMOTE_APP_TRANSITION_ANIMATIONS
  - android.permission.CHANGE_OVERLAY_PACKAGES
  - android.permission.WATCH_APPOPS
  - android.permission.MANAGE_DEBUGGING
  - android.permission.HIDE_NON_SYSTEM_OVERLAY_WINDOWS
  - android.permission.CONTROL_DISPLAY_COLOR_TRANSFORMS
  - android.permission.MANAGE_VOICE_KEYPHRASES
  - android.permission.REBOOT
  - android.permission.READ_CALL_LOG: restricted=true
--------------------------------------------------------------------------------

# PARTITION: /system
================================================================================
PACKAGE: com.mediatek.autodialer
PATH:    /system/app/AutoDialer/AutoDialer.apk
UID:     1001
VERSION: 1.0
PERMISSIONS:
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_LOCATION_EXTRA_COMMANDS
  - android.permission.CALL_PRIVILEGED
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.READ_PHONE_STATE
  - android.permission.FOREGROUND_SERVICE
--------------------------------------------------------------------------------
PACKAGE: com.mediatek.batterywarning
PATH:    /system/app/BatteryWarning/BatteryWarning.apk
UID:     1000
VERSION: 10
PERMISSIONS:
  - android.permission.RECEIVE_BOOT_COMPLETED
--------------------------------------------------------------------------------
PACKAGE: com.android.captiveportallogin
PATH:    /system/app/CaptivePortalLogin/CaptivePortalLogin.apk
UID:     10027
VERSION: 2019-09
PERMISSIONS:
  - android.permission.INTERNET
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.CONNECTIVITY_USE_RESTRICTED_NETWORKS
  - android.permission.NETWORK_BYPASS_PRIVATE_DNS
  - android.permission.MAINLINE_NETWORK_STACK
  - android.permission.ACCESS_COARSE_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.android.carrierdefaultapp
PATH:    /system/app/CarrierDefaultApp/CarrierDefaultApp.apk
UID:     10026
VERSION: 10
PERMISSIONS:
  - android.permission.INTERNET
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.CONNECTIVITY_INTERNAL
  - android.permission.MODIFY_PHONE_STATE
  - android.permission.READ_PRIVILEGED_PHONE_STATE
  - android.permission.CONNECTIVITY_USE_RESTRICTED_NETWORKS
  - android.permission.NETWORK_BYPASS_PRIVATE_DNS
  - android.permission.SUBSTITUTE_NOTIFICATION_APP_NAME
--------------------------------------------------------------------------------
PACKAGE: com.debug.loggerui
PATH:    /system/app/DebugLoggerUI/DebugLoggerUI.apk
UID:     10030
VERSION: 6.0.0
PERMISSIONS:
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.VIBRATE
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.GET_TASKS
  - android.permission.READ_FRAME_BUFFER
  - android.permission.MANAGE_USERS
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_MEDIA_STORAGE
  - android.permission.RAISED_THREAD_PRIORITY
  - android.permission.DUMP
  - android.permission.BLUETOOTH
  - android.permission.FOREGROUND_SERVICE
  - android.permission.STATUS_BAR_SERVICE
  - android.permission.READ_LOGS
  - android.permission.ACCESS_MEDIA_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.mediatek.gba
PATH:    /system/app/Gba/Gba.apk
UID:     1001
VERSION: 10
PERMISSIONS:
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.READ_PRIVILEGED_PHONE_STATE
--------------------------------------------------------------------------------
PACKAGE: com.google.android.ext.shared
PATH:    /system/app/GoogleExtShared/GoogleExtShared.apk
UID:     10028
VERSION: 1
--------------------------------------------------------------------------------
PACKAGE: com.android.keychain
PATH:    /system/app/KeyChain/KeyChain.apk
UID:     1000
VERSION: 10
--------------------------------------------------------------------------------
PACKAGE: com.mediatek.lbs.em2.ui
PATH:    /system/app/LocationEM2/LocationEM2.apk
UID:     10031
VERSION: 1.0
PERMISSIONS:
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_LOCATION_EXTRA_COMMANDS
  - android.permission.WAKE_LOCK
  - android.permission.INTERNET
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_MEDIA_STORAGE
  - android.permission.READ_PHONE_STATE
  - android.permission.LOCATION_HARDWARE
  - android.permission.FOREGROUND_SERVICE
  - android.permission.WRITE_SECURE_SETTINGS
  - android.permission.ACCESS_BACKGROUND_LOCATION: restricted=true
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.android.pacprocessor
PATH:    /system/app/PacProcessor/PacProcessor.apk
UID:     10029
VERSION: 10
PERMISSIONS:
  - android.permission.INTERNET
--------------------------------------------------------------------------------
PACKAGE: com.android.se
PATH:    /system/app/SecureElement/SecureElement.apk
UID:     1068
VERSION: 10
--------------------------------------------------------------------------------
PACKAGE: com.android.simappdialog
PATH:    /system/app/SimAppDialog/SimAppDialog.apk
UID:     10024
VERSION: 10
--------------------------------------------------------------------------------
PACKAGE: com.android.bluetooth
PATH:    /system/app/WearBluetooth/WearBluetooth.apk
UID:     1002
VERSION: 10
PERMISSIONS:
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.ACCESS_BLUETOOTH_SHARE
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.INTERNET
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.BLUETOOTH_PRIVILEGED
  - android.permission.BLUETOOTH_MAP
  - android.permission.DUMP
  - android.permission.WAKE_LOCK
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_CONTACTS
  - android.permission.WRITE_CONTACTS
  - android.permission.READ_CALL_LOG: restricted=true
  - android.permission.WRITE_CALL_LOG: restricted=true
  - android.permission.READ_PRIVILEGED_PHONE_STATE
  - android.permission.WRITE_SETTINGS
  - android.permission.NFC_HANDOVER_STATUS
  - android.permission.WRITE_SECURE_SETTINGS
  - android.permission.WRITE_APN_SETTINGS
  - android.permission.NET_ADMIN
  - android.permission.CALL_PRIVILEGED
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.NET_TUNNELING
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.CONNECTIVITY_INTERNAL
  - android.permission.TETHER_PRIVILEGED
  - android.permission.MODIFY_PHONE_STATE
  - android.permission.INTERACT_ACROSS_USERS_FULL
  - android.permission.BLUETOOTH_STACK
  - android.permission.INTERACT_ACROSS_USERS
  - android.permission.MANAGE_USERS
  - android.permission.GET_ACCOUNTS
  - android.permission.RECEIVE_SMS: restricted=true
  - android.permission.SEND_SMS: restricted=true
  - android.permission.READ_SMS: restricted=true
  - android.permission.WRITE_SMS
  - android.permission.MEDIA_CONTENT_CONTROL
  - android.permission.UPDATE_APP_OPS_STATS
  - android.permission.MANAGE_APP_OPS_MODES
  - android.permission.VIBRATE
  - android.permission.DEVICE_POWER
  - android.permission.REAL_GET_TASKS
  - android.permission.MODIFY_AUDIO_ROUTING
  - android.permission.UPDATE_DEVICE_STATS
  - android.permission.PACKAGE_USAGE_STATS
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.GET_RUNTIME_PERMISSIONS
  - android.permission.REQUEST_INSTALL_PACKAGES
  - android.permission.WRITE_MEDIA_STORAGE
  - android.permission.READ_PROFILE
--------------------------------------------------------------------------------
PACKAGE: com.miui.calculator
PATH:    /system/app/WearCalculator/WearCalculator.apk
UID:     10033
VERSION: 15.1.9
PERMISSIONS:
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.INTERNET
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.GET_TASKS
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.SYSTEM_OVERLAY_WINDOW
  - android.permission.FOREGROUND_SERVICE
  - android.permission.INTERACT_ACROSS_USERS_FULL
  - android.permission.WRITE_SECURE_SETTINGS
  - android.permission.VIBRATE
--------------------------------------------------------------------------------
PACKAGE: com.signal.main
PATH:    /system/app/WearFactoryMode/WearFactoryMode.apk
UID:     1000
VERSION: 10
PERMISSIONS:
  - android.permission.ACCESS_FM_RADIO
  - android.permission.CAMERA
  - android.permission.FLASHLIGHT
  - android.permission.HARDWARE_TEST
  - android.permission.DISABLE_KEYGUARD
  - android.permission.GET_TASKS
  - android.permission.SET_PREFERRED_APPLICATIONS
  - android.permission.DEVICE_POWER
  - android.permission.WAKE_LOCK
  - android.permission.READ_CONTACTS
  - android.permission.WRITE_SETTINGS
  - android.permission.RECORD_AUDIO
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.MOUNT_UNMOUNT_FILESYSTEMS
  - android.permission.VIBRATE
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.WRITE_SECURE_SETTINGS
  - android.permission.READ_PHONE_STATE
  - android.permission.INTERNAL_SYSTEM_WINDOW
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.HOTKNOT
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.android.htmlviewer
PATH:    /system/app/WearHTMLViewer/WearHTMLViewer.apk
UID:     10023
VERSION: 10
--------------------------------------------------------------------------------
PACKAGE: com.wiite.inputmethod.pinyin
PATH:    /system/app/WearPinyinIme/WearPinyinIme.apk
UID:     10025
VERSION: 1.0.0
PERMISSIONS:
  - android.permission.VIBRATE
  - android.permission.INTERNET
  - android.permission.READ_USER_DICTIONARY
  - android.permission.WRITE_USER_DICTIONARY
  - android.permission.READ_CONTACTS
--------------------------------------------------------------------------------
PACKAGE: fdy.wiitter.com
PATH:    /system/app/WearScreenLock/WearScreenLock.apk
UID:     1000
VERSION: 1.0
PERMISSIONS:
  - android.permission.VIBRATE
  - android.permission.FORCE_STOP_PACKAGES
--------------------------------------------------------------------------------
PACKAGE: com.mediatek.ygps
PATH:    /system/app/YGPS/YGPS.apk
UID:     10032
VERSION: 1.1
PERMISSIONS:
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_LOCATION_EXTRA_COMMANDS
  - android.permission.WAKE_LOCK
  - android.permission.FOREGROUND_SERVICE
  - android.permission.INTERNET
  - android.permission.ACCESS_COARSE_LOCATION
--------------------------------------------------------------------------------
PACKAGE: android
PATH:    /system/framework/framework-res.apk
UID:     1000
VERSION: 10
PERMISSIONS:
  - android.permission.LOCATION_HARDWARE
  - android.permission.CONNECTIVITY_USE_RESTRICTED_NETWORKS
  - android.permission.GET_ACCOUNTS
  - android.permission.SEND_SHOW_SUSPENDED_APP_DETAILS
  - android.permission.BIND_JOB_SERVICE
  - android.permission.TRIGGER_TIME_ZONE_RULES_CHECK
  - android.permission.BIND_NETWORK_RECOMMENDATION_SERVICE
  - android.permission.BIND_ATTENTION_SERVICE
  - android.permission.CONTROL_VPN
  - android.permission.PACKAGE_USAGE_STATS
  - android.permission.LOCAL_MAC_ADDRESS
  - android.permission.CONFIRM_FULL_BACKUP
  - android.permission.ACCESS_INSTANT_APPS
--------------------------------------------------------------------------------
PACKAGE: com.mediatek
PATH:    /system/framework/mediatek-res.apk
UID:     1000
VERSION: 10
--------------------------------------------------------------------------------
PACKAGE: com.android.dynsystem
PATH:    /system/priv-app/DynamicSystemInstallationService/DynamicSystemInstallationService.apk
UID:     1000
VERSION: 10
PERMISSIONS:
  - android.permission.FOREGROUND_SERVICE
  - android.permission.INTERNET
  - android.permission.MANAGE_DYNAMIC_SYSTEM
  - android.permission.REBOOT
  - android.permission.RECEIVE_BOOT_COMPLETED
--------------------------------------------------------------------------------
PACKAGE: com.android.externalstorage
PATH:    /system/priv-app/ExternalStorageProvider/ExternalStorageProvider.apk
UID:     10019
VERSION: 10
PERMISSIONS:
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_MEDIA_STORAGE
  - android.permission.MOUNT_UNMOUNT_FILESYSTEMS
--------------------------------------------------------------------------------
PACKAGE: com.android.location.fused
PATH:    /system/priv-app/FusedLocation/FusedLocation.apk
UID:     1000
VERSION: 10
PERMISSIONS:
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.INSTALL_LOCATION_PROVIDER
  - android.permission.INTERACT_ACROSS_USERS_FULL
--------------------------------------------------------------------------------
PACKAGE: com.google.android.ext.services
PATH:    /system/priv-app/GoogleExtServicesPrebuilt/GoogleExtServicesPrebuilt.apk
UID:     10013
VERSION: q_pr1-release_aml_291900801
PERMISSIONS:
  - android.permission.PROVIDE_RESOLVER_RANKER_SERVICE
  - android.permission.READ_DEVICE_CONFIG
  - android.permission.MONITOR_DEFAULT_SMS_PACKAGE
  - android.permission.REQUEST_NOTIFICATION_ASSISTANT_SERVICE
  - android.permission.ACCESS_NETWORK_STATE
--------------------------------------------------------------------------------
PACKAGE: com.google.android.networkstack.permissionconfig
PATH:    /system/priv-app/GoogleNetworkPermissionConfig/GoogleNetworkPermissionConfig.apk
UID:     1073
VERSION: q_pr1-release_aml_291900801
--------------------------------------------------------------------------------
PACKAGE: com.google.android.networkstack
PATH:    /system/priv-app/GoogleNetworkStack/GoogleNetworkStack.apk
UID:     1073
VERSION: r_aml_300900700
PERMISSIONS:
  - android.permission.INTERNET
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.CONNECTIVITY_INTERNAL
  - android.permission.INTERACT_ACROSS_USERS
  - android.permission.WAKE_LOCK
  - android.permission.READ_PRIVILEGED_PHONE_STATE
  - android.permission.READ_DEVICE_CONFIG
  - android.permission.MAINLINE_NETWORK_STACK
--------------------------------------------------------------------------------
PACKAGE: cn.heils.faceunlock
PATH:    /system/priv-app/HeilsFaceUnlockDM101/HeilsFaceUnlockDM101.apk
UID:     10021
VERSION: 1.7
PERMISSIONS:
  - android.permission.CAMERA
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.DISABLE_KEYGUARD
  - android.permission.VIBRATE
  - android.permission.WRITE_SETTINGS
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.READ_PHONE_STATE
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.GET_TASKS
  - android.permission.ACCESS_MEDIA_LOCATION
--------------------------------------------------------------------------------
PACKAGE: com.mediatek.ims
PATH:    /system/priv-app/ImsService/ImsService.apk
UID:     1001
VERSION: 10
PERMISSIONS:
  - android.permission.READ_PRECISE_PHONE_STATE
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.WAKE_LOCK
  - android.permission.CAMERA
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.READ_NETWORK_USAGE_HISTORY
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.WRITE_SETTINGS
  - android.permission.READ_PHONE_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.PACKET_KEEPALIVE_OFFLOAD
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.NETWORK_SIGNAL_STRENGTH_WAKEUP
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.android.inputdevices
PATH:    /system/priv-app/InputDevices/InputDevices.apk
UID:     1000
VERSION: 10
--------------------------------------------------------------------------------
PACKAGE: com.mediatek.location.lppe.main
PATH:    /system/priv-app/LPPeService/LPPeService.apk
UID:     1000
VERSION: 1.0
PERMISSIONS:
  - android.permission.LOCATION_HARDWARE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.READ_SMS: restricted=true
  - android.permission.RECEIVE_SMS: restricted=true
  - android.permission.RECEIVE_WAP_PUSH: restricted=true
  - android.permission.WAKE_LOCK
  - android.permission.PROCESS_OUTGOING_CALLS: restricted=true
  - android.permission.READ_PRIVILEGED_PHONE_STATE
--------------------------------------------------------------------------------
PACKAGE: com.mediatek.capctrl.service
PATH:    /system/priv-app/MtkCapCtrl/MtkCapCtrl.apk
UID:     1001
VERSION: 10
PERMISSIONS:
  - android.permission.RECEIVE_BOOT_COMPLETED
--------------------------------------------------------------------------------
PACKAGE: com.android.providers.contacts
PATH:    /system/priv-app/MtkContactsProvider/MtkContactsProvider.apk
UID:     10012
VERSION: 10
PERMISSIONS:
  - android.permission.BIND_DIRECTORY_SEARCH
  - android.permission.GET_ACCOUNTS
  - android.permission.GET_ACCOUNTS_PRIVILEGED
  - android.permission.INTERACT_ACROSS_USERS
  - android.permission.MANAGE_USERS
  - android.permission.PROCESS_PHONE_ACCOUNT_REGISTRATION
  - android.permission.READ_CONTACTS
  - android.permission.READ_PHONE_STATE
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.SEND_CALL_LOG_CHANGE
  - android.permission.UPDATE_APP_OPS_STATS
  - android.permission.WRITE_CONTACTS
  - android.permission.USE_RESERVED_DISK
  - android.permission.READ_DEVICE_CONFIG
--------------------------------------------------------------------------------
PACKAGE: com.android.providers.downloads
PATH:    /system/priv-app/MtkDownloadProvider/MtkDownloadProvider.apk
UID:     10018
VERSION: 10
PERMISSIONS:
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.START_ACTIVITIES_FROM_BACKGROUND
  - android.permission.ACCESS_DOWNLOAD_MANAGER
  - android.permission.ACCESS_CACHE_FILESYSTEM
  - android.permission.SEND_DOWNLOAD_COMPLETED_INTENTS
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.INTERNET
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_MEDIA_STORAGE
  - android.permission.ACCESS_ALL_DOWNLOADS
  - android.permission.UPDATE_DEVICE_STATS
  - android.permission.CONNECTIVITY_INTERNAL
  - android.permission.MODIFY_NETWORK_ACCOUNTING
  - android.permission.CLEAR_APP_CACHE
  - android.permission.WAKE_LOCK
  - android.permission.UPDATE_APP_OPS_STATS
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.android.providers.downloads.ui
PATH:    /system/priv-app/MtkDownloadProviderUi/MtkDownloadProviderUi.apk
UID:     10018
VERSION: 10
PERMISSIONS:
  - android.permission.SEND_DOWNLOAD_COMPLETED_INTENTS
  - android.permission.ACCESS_ALL_DOWNLOADS
  - android.permission.START_ACTIVITIES_FROM_BACKGROUND
--------------------------------------------------------------------------------
PACKAGE: com.mediatek.mms.appservice
PATH:    /system/priv-app/MtkMmsAppService/MtkMmsAppService.apk
UID:     1001
VERSION: 10
PERMISSIONS:
  - android.permission.RECEIVE_SMS: restricted=true
  - android.permission.RECEIVE_MMS: restricted=true
  - android.permission.SEND_SMS: restricted=true
  - android.permission.INTERNET
  - android.permission.READ_SMS: restricted=true
  - android.permission.WRITE_SMS
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.READ_PHONE_STATE
  - android.permission.WAKE_LOCK
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_WAPPUSH
  - android.permission.WRITE_WAPPUSH
  - android.permission.RECEIVE_WAP_PUSH: restricted=true
  - android.permission.BROADCAST_STICKY
  - android.permission.ACCESS_WIFI_STATE
--------------------------------------------------------------------------------
PACKAGE: com.android.mms.service
PATH:    /system/priv-app/MtkMmsService/MtkMmsService.apk
UID:     1001
VERSION: 10
PERMISSIONS:
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.READ_SMS: restricted=true
  - android.permission.WRITE_SMS
  - android.permission.BROADCAST_WAP_PUSH
  - android.permission.BIND_CARRIER_SERVICES
  - android.permission.BIND_CARRIER_MESSAGING_SERVICE
  - android.permission.INTERACT_ACROSS_USERS
  - android.permission.INTERNET
--------------------------------------------------------------------------------
PACKAGE: com.android.phone
PATH:    /system/priv-app/MtkTeleService/MtkTeleService.apk
UID:     1001
VERSION: 10
PERMISSIONS:
  - android.permission.BROADCAST_STICKY
  - android.permission.CALL_PHONE
  - android.permission.CALL_PRIVILEGED
  - android.permission.CONTROL_INCALL_EXPERIENCE
  - android.permission.DOWNLOAD_WITHOUT_NOTIFICATION
  - android.permission.WRITE_SETTINGS
  - android.permission.WRITE_SECURE_SETTINGS
  - android.permission.READ_CONTACTS
  - android.permission.READ_CALL_LOG: restricted=true
  - android.permission.WRITE_CONTACTS
  - android.permission.WRITE_CALL_LOG: restricted=true
  - android.permission.SYSTEM_ALERT_WINDOW
  - android.permission.INTERNAL_SYSTEM_WINDOW
  - android.permission.VIBRATE
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.REORDER_TASKS
  - android.permission.CHANGE_CONFIGURATION
  - android.permission.WAKE_LOCK
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.STATUS_BAR
  - android.permission.STATUS_BAR_SERVICE
  - android.permission.RECEIVE_SMS: restricted=true
  - android.permission.READ_SMS: restricted=true
  - android.permission.WRITE_SMS
  - android.permission.SEND_SMS: restricted=true
  - android.permission.SEND_RESPOND_VIA_MESSAGE
  - android.permission.SET_TIME
  - android.permission.SET_TIME_ZONE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.READ_PRIVILEGED_PHONE_STATE
  - android.permission.MODIFY_PHONE_STATE
  - android.permission.ACCESS_IMS_CALL_SERVICE
  - android.permission.DEVICE_POWER
  - android.permission.DISABLE_KEYGUARD
  - android.permission.INTERNET
  - android.permission.PROCESS_OUTGOING_CALLS: restricted=true
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.WRITE_APN_SETTINGS
  - android.permission.BROADCAST_SMS
  - android.permission.BROADCAST_WAP_PUSH
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.SHUTDOWN
  - android.permission.RECORD_AUDIO
  - android.permission.PERFORM_CDMA_PROVISIONING
  - android.permission.USE_SIP
  - android.permission.REBOOT
  - android.permission.UPDATE_LOCK
  - android.permission.INTERACT_ACROSS_USERS
  - android.permission.INTERACT_ACROSS_USERS_FULL
  - android.permission.MANAGE_USERS
  - android.permission.UPDATE_APP_OPS_STATS
  - android.permission.MANAGE_APP_OPS_MODES
  - android.permission.CONNECTIVITY_INTERNAL
  - android.permission.SET_PREFERRED_APPLICATIONS
  - android.permission.READ_SEARCH_INDEXABLES
  - android.permission.DUMP
  - android.permission.REGISTER_CALL_PROVIDER
  - android.permission.REGISTER_SIM_SUBSCRIPTION
  - android.permission.BIND_IMS_SERVICE
  - android.permission.BIND_CARRIER_SERVICES
  - android.permission.BIND_CARRIER_MESSAGING_SERVICE
  - android.permission.BIND_EUICC_SERVICE
  - android.permission.WRITE_EMBEDDED_SUBSCRIPTIONS
  - android.permission.READ_SYNC_SETTINGS
  - android.permission.WRITE_SYNC_SETTINGS
  - android.permission.AUTHENTICATE_ACCOUNTS
  - android.permission.MANAGE_ACCOUNTS
  - android.permission.GET_ACCOUNTS
  - android.permission.BIND_VISUAL_VOICEMAIL_SERVICE
  - android.permission.LOCAL_MAC_ADDRESS
  - android.permission.CHANGE_COMPONENT_ENABLED_STATE
  - android.permission.CHANGE_DEVICE_IDLE_TEMP_WHITELIST
  - android.permission.READ_BLOCKED_NUMBERS
  - android.permission.WRITE_BLOCKED_NUMBERS
  - android.permission.NETWORK_SETTINGS
  - android.permission.STOP_APP_SWITCHES
  - android.permission.UPDATE_DEVICE_STATS
  - android.permission.MANAGE_NETWORK_POLICY
  - android.permission.READ_NETWORK_USAGE_HISTORY
  - android.permission.BIND_TELEPHONY_DATA_SERVICE
  - android.permission.PACKAGE_USAGE_STATS
  - android.permission.READ_PRECISE_PHONE_STATE
  - android.permission.MANAGE_ROLE_HOLDERS
  - android.permission.TETHER_PRIVILEGED
--------------------------------------------------------------------------------
PACKAGE: com.mediatek.telephony
PATH:    /system/priv-app/MtkTelephonyAssist/MtkTelephonyAssist.apk
UID:     1001
VERSION: 10
PERMISSIONS:
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.READ_PRIVILEGED_PHONE_STATE
--------------------------------------------------------------------------------
PACKAGE: com.android.providers.telephony
PATH:    /system/priv-app/MtkTelephonyProvider/MtkTelephonyProvider.apk
UID:     1001
VERSION: 10
PERMISSIONS:
  - android.permission.RECEIVE_SMS: restricted=true
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_DEVICE_CONFIG
  - android.permission.INTERACT_ACROSS_USERS
  - android.permission.UPDATE_APP_OPS_STATS
  - android.permission.READ_CONTACTS
  - android.permission.WRITE_SETTINGS
  - android.permission.INTERACT_ACROSS_USERS_FULL
  - android.permission.USE_RESERVED_DISK
  - android.permission.MODIFY_PHONE_STATE
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.android.proxyhandler
PATH:    /system/priv-app/ProxyHandler/ProxyHandler.apk
UID:     10014
VERSION: 10
PERMISSIONS:
  - android.permission.INTERNET
--------------------------------------------------------------------------------
PACKAGE: com.android.shell
PATH:    /system/priv-app/Shell/Shell.apk
UID:     2000
VERSION: 10
PERMISSIONS:
  - android.permission.GET_RUNTIME_PERMISSIONS
  - android.permission.SEND_SMS: restricted=true
  - android.permission.READ_SMS: restricted=true
  - android.permission.CALL_PHONE
  - android.permission.READ_PHONE_STATE
  - android.permission.READ_PRECISE_PHONE_STATE
  - android.permission.READ_PRIVILEGED_PHONE_STATE
  - android.permission.READ_CONTACTS
  - android.permission.WRITE_CONTACTS
  - android.permission.READ_CALENDAR
  - android.permission.WRITE_CALENDAR
  - android.permission.READ_USER_DICTIONARY
  - android.permission.WRITE_USER_DICTIONARY
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.ACCESS_LOCATION_EXTRA_COMMANDS
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.BLUETOOTH
  - android.permission.LOCAL_MAC_ADDRESS
  - android.permission.EXPAND_STATUS_BAR
  - android.permission.DISABLE_KEYGUARD
  - android.permission.MANAGE_NETWORK_POLICY
  - android.permission.MANAGE_USB
  - android.permission.USE_RESERVED_DISK
  - android.permission.FOREGROUND_SERVICE
  - android.permission.REAL_GET_TASKS
  - android.permission.CHANGE_CONFIGURATION
  - android.permission.REORDER_TASKS
  - android.permission.REMOVE_TASKS
  - android.permission.SET_ANIMATION_SCALE
  - android.permission.SET_PREFERRED_APPLICATIONS
  - android.permission.WRITE_SETTINGS
  - android.permission.WRITE_SECURE_SETTINGS
  - android.permission.READ_DEVICE_CONFIG
  - android.permission.WRITE_DEVICE_CONFIG
  - android.permission.BROADCAST_STICKY
  - android.permission.MANAGE_ACCESSIBILITY
  - android.permission.SET_DEBUG_APP
  - android.permission.SET_PROCESS_LIMIT
  - android.permission.SET_ALWAYS_FINISH
  - android.permission.DUMP
  - android.permission.SIGNAL_PERSISTENT_PROCESSES
  - android.permission.KILL_BACKGROUND_PROCESSES
  - android.permission.FORCE_BACK
  - android.permission.BATTERY_STATS
  - android.permission.PACKAGE_USAGE_STATS
  - android.permission.INTERNAL_SYSTEM_WINDOW
  - android.permission.INJECT_EVENTS
  - android.permission.RETRIEVE_WINDOW_CONTENT
  - android.permission.SET_ACTIVITY_WATCHER
  - android.permission.READ_INPUT_STATE
  - android.permission.SET_ORIENTATION
  - android.permission.INSTALL_PACKAGES
  - android.permission.MOVE_PACKAGE
  - android.permission.CLEAR_APP_USER_DATA
  - android.permission.CLEAR_APP_CACHE
  - android.permission.ACCESS_INSTANT_APPS
  - android.permission.DELETE_CACHE_FILES
  - android.permission.DELETE_PACKAGES
  - android.permission.MANAGE_ROLLBACKS
  - android.permission.TEST_MANAGE_ROLLBACKS
  - android.permission.ACCESS_SURFACE_FLINGER
  - android.permission.READ_FRAME_BUFFER
  - android.permission.DEVICE_POWER
  - android.permission.POWER_SAVER
  - android.permission.INSTALL_LOCATION_PROVIDER
  - android.permission.BACKUP
  - android.permission.FORCE_STOP_PACKAGES
  - android.permission.STOP_APP_SWITCHES
  - android.permission.ACCESS_CONTENT_PROVIDERS_EXTERNALLY
  - android.permission.GRANT_RUNTIME_PERMISSIONS
  - android.permission.REVOKE_RUNTIME_PERMISSIONS
  - android.permission.INSTALL_GRANT_RUNTIME_PERMISSIONS
  - android.permission.WHITELIST_RESTRICTED_PERMISSIONS
  - android.permission.SET_KEYBOARD_LAYOUT
  - android.permission.GET_DETAILED_TASKS
  - android.permission.SET_SCREEN_COMPATIBILITY
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_MEDIA_STORAGE
  - android.permission.INTERACT_ACROSS_USERS
  - android.permission.INTERACT_ACROSS_USERS_FULL
  - android.permission.CREATE_USERS
  - android.permission.MANAGE_DEVICE_ADMINS
  - android.permission.ACCESS_LOWPAN_STATE
  - android.permission.CHANGE_LOWPAN_STATE
  - android.permission.READ_LOWPAN_CREDENTIAL
  - android.permission.BLUETOOTH_STACK
  - android.permission.GET_ACCOUNTS
  - android.permission.RETRIEVE_WINDOW_TOKEN
  - android.permission.FRAME_STATS
  - android.permission.BIND_APPWIDGET
  - android.permission.UPDATE_APP_OPS_STATS
  - android.permission.MODIFY_APPWIDGET_BIND_PERMISSIONS
  - android.permission.CHANGE_APP_IDLE_STATE
  - android.permission.CHANGE_DEVICE_IDLE_TEMP_WHITELIST
  - android.permission.MOUNT_UNMOUNT_FILESYSTEMS
  - android.permission.MOUNT_FORMAT_FILESYSTEMS
  - android.permission.MODIFY_PHONE_STATE
  - android.permission.NETWORK_SCAN
  - android.permission.REGISTER_CALL_PROVIDER
  - android.permission.REGISTER_CONNECTION_MANAGER
  - android.permission.REGISTER_SIM_SUBSCRIPTION
  - android.permission.GET_APP_OPS_STATS
  - android.permission.MANAGE_APP_OPS_MODES
  - android.permission.VIBRATE
  - android.permission.MANAGE_ACTIVITY_STACKS
  - android.permission.START_TASKS_FROM_RECENTS
  - android.permission.START_ACTIVITIES_FROM_BACKGROUND
  - android.permission.ACTIVITY_EMBEDDING
  - android.permission.CONNECTIVITY_INTERNAL
  - android.permission.CHANGE_COMPONENT_ENABLED_STATE
  - android.permission.MANAGE_AUTO_FILL
  - android.permission.MANAGE_CONTENT_CAPTURE
  - android.permission.MANAGE_CONTENT_SUGGESTIONS
  - android.permission.MANAGE_APP_PREDICTIONS
  - android.permission.NETWORK_SETTINGS
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.SET_TIME
  - android.permission.SET_TIME_ZONE
  - android.permission.DISABLE_HIDDEN_API_CHECKS
  - android.permission.MANAGE_ROLE_HOLDERS
  - android.permission.OBSERVE_ROLE_HOLDERS
  - android.permission.STATUS_BAR_SERVICE
  - android.permission.STATUS_BAR
  - android.permission.SUBSTITUTE_NOTIFICATION_APP_NAME
  - android.permission.REQUEST_NOTIFICATION_ASSISTANT_SERVICE
  - android.permission.WAKE_LOCK
  - android.permission.CHANGE_OVERLAY_PACKAGES
  - android.permission.RESTRICTED_VR_ACCESS
  - android.permission.MANAGE_BIND_INSTANT_SERVICE
  - android.permission.SET_HARMFUL_APP_WARNINGS
  - android.permission.MANAGE_SENSORS
  - android.permission.MANAGE_AUDIO_POLICY
  - android.permission.MANAGE_CAMERA
  - android.permission.MANAGE_BLUETOOTH_WHEN_WIRELESS_CONSENT_REQUIRED
  - android.permission.MANAGE_WIFI_WHEN_WIRELESS_CONSENT_REQUIRED
  - android.permission.INSTALL_DYNAMIC_SYSTEM
  - android.permission.CONTROL_KEYGUARD
  - android.permission.SUSPEND_APPS
  - android.permission.OBSERVE_APP_USAGE
  - android.permission.READ_CLIPBOARD_IN_BACKGROUND
  - android.permission.ENABLE_TEST_HARNESS_MODE
  - android.permission.MANAGE_APPOPS
  - android.permission.MANAGE_TEST_NETWORKS
  - android.permission.PACKET_KEEPALIVE_OFFLOAD
  - android.permission.CONTROL_KEYGUARD_SECURE_NOTIFICATIONS
  - android.permission.SET_WALLPAPER
  - android.permission.SET_WALLPAPER_COMPONENT
  - android.permission.CACHE_CONTENT
  - android.permission.BIND_EXPLICIT_HEALTH_CHECK_SERVICE
  - android.permission.INTERACT_ACROSS_PROFILES
--------------------------------------------------------------------------------
PACKAGE: com.android.statementservice
PATH:    /system/priv-app/StatementService/StatementService.apk
UID:     10016
VERSION: 1.0
PERMISSIONS:
  - android.permission.INTERNET
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.INTENT_FILTER_VERIFICATION_AGENT
--------------------------------------------------------------------------------
PACKAGE: com.android.providers.userdictionary
PATH:    /system/priv-app/UserDictionaryProvider/UserDictionaryProvider.apk
UID:     10012
VERSION: 10
--------------------------------------------------------------------------------
PACKAGE: com.android.vpndialogs
PATH:    /system/priv-app/VpnDialogs/VpnDialogs.apk
UID:     10011
VERSION: 10
PERMISSIONS:
  - android.permission.CONTROL_VPN
  - android.permission.CONTROL_ALWAYS_ON_VPN
  - android.permission.CONNECTIVITY_INTERNAL
  - android.permission.HIDE_NON_SYSTEM_OVERLAY_WINDOWS
--------------------------------------------------------------------------------
PACKAGE: com.wapi.wapicertmanager
PATH:    /system/priv-app/WapiCertManager/WapiCertManager.apk
UID:     1000
VERSION: 10
PERMISSIONS:
  - android.permission.USE_CREDENTIALS
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.android.documentsui
PATH:    /system/priv-app/WearDocumentsUI/WearDocumentsUI.apk
UID:     10022
VERSION: 10
PERMISSIONS:
  - android.permission.MANAGE_DOCUMENTS
  - android.permission.REMOVE_TASKS
  - android.permission.FOREGROUND_SERVICE
  - android.permission.WAKE_LOCK
  - android.permission.CACHE_CONTENT
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.CHANGE_OVERLAY_PACKAGES
--------------------------------------------------------------------------------
PACKAGE: com.mediatek.engineermode
PATH:    /system/priv-app/WearEngineerMode/WearEngineerMode.apk
UID:     1001
VERSION: 1.0
PERMISSIONS:
  - android.permission.WRITE_SETTINGS
  - android.permission.READ_SETTINGS
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.CALL_PHONE
  - android.permission.CAMERA
  - android.permission.ACCESS_WIFI_STATE
  - android.permission.CHANGE_WIFI_STATE
  - android.permission.ACCESS_BLUETOOTH_SHARE
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.VIBRATE
  - android.permission.BLUETOOTH_STACK
  - android.permission.NFC
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.READ_CONTACTS
  - android.permission.WRITE_CONTACTS
  - android.permission.READ_PHONE_STATE
  - android.permission.ACCESS_COARSE_LOCATION
  - android.permission.WRITE_APN_SETTINGS
  - android.permission.MANAGE_USB
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.WAKE_LOCK
  - android.permission.DEVICE_POWER
  - android.permission.DISABLE_KEYGUARD
  - android.permission.SET_WALLPAPER
  - android.permission.ACCESS_FINE_LOCATION
  - android.permission.READ_PRIVILEGED_PHONE_STATE
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.NETWORK_SETTINGS
  - android.permission.FORCE_STOP_PACKAGES
  - android.permission.INJECT_EVENTS
  - android.permission.MASTER_CLEAR
  - android.permission.ACCESS_SURFACE_FLINGER
  - android.permission.REAL_GET_TASKS
  - android.permission.FOREGROUND_SERVICE
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.android.providers.media
PATH:    /system/priv-app/WearMediaProvider/WearMediaProvider.apk
UID:     10018
VERSION: 10
PERMISSIONS:
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.RECEIVE_DEVICE_CUSTOMIZATION_READY
  - android.permission.WRITE_SETTINGS
  - android.permission.WAKE_LOCK
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_MEDIA_STORAGE
  - android.permission.ACCESS_MTP
  - android.permission.MANAGE_USERS
  - android.permission.INTERACT_ACROSS_USERS
  - android.permission.USE_RESERVED_DISK
  - android.permission.WATCH_APPOPS
  - android.permission.UPDATE_DEVICE_STATS
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
--------------------------------------------------------------------------------
PACKAGE: com.android.mms
PATH:    /system/priv-app/WearMtkMms/WearMtkMms.apk
UID:     10015
VERSION: 10
PERMISSIONS:
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.CALL_PHONE
  - android.permission.READ_CONTACTS
  - android.permission.WRITE_CONTACTS
  - android.permission.READ_PROFILE
  - android.permission.RECEIVE_SMS: restricted=true
  - android.permission.RECEIVE_EMERGENCY_BROADCAST
  - android.permission.RECEIVE_MMS: restricted=true
  - android.permission.SEND_SMS: restricted=true
  - android.permission.VIBRATE
  - android.permission.INTERNET
  - android.permission.READ_SMS: restricted=true
  - android.permission.WRITE_SMS
  - android.permission.ACCESS_NETWORK_STATE
  - android.permission.CHANGE_NETWORK_STATE
  - android.permission.READ_PHONE_STATE
  - android.permission.WAKE_LOCK
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.WRITE_MEDIA_STORAGE
  - android.permission.INSTALL_DRM
  - android.permission.UPDATE_DEVICE_STATS
  - android.permission.READ_WAPPUSH
  - android.permission.WRITE_WAPPUSH
  - android.permission.RECEIVE_WAP_PUSH: restricted=true
  - android.permission.WRITE_SETTINGS
  - android.permission.WRITE_SECURE_SETTINGS
  - android.permission.BROADCAST_STICKY
  - android.permission.GET_TASKS
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.REORDER_TASKS
  - android.permission.WRITE_APN_SETTINGS
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.MMS_SEND_OUTBOX_MSG
  - android.permission.INTERACT_ACROSS_USERS
  - android.permission.INTERACT_ACROSS_USERS_FULL
  - android.permission.MANAGE_USERS
  - android.permission.READ_PRIVILEGED_PHONE_STATE
  - android.permission.MODIFY_PHONE_STATE
  - android.permission.ACCESS_WIFI_STATE
--------------------------------------------------------------------------------
PACKAGE: com.android.packageinstaller
PATH:    /system/priv-app/WearPackageInstaller/WearPackageInstaller.apk
UID:     10017
VERSION: 10
PERMISSIONS:
  - android.permission.MANAGE_USERS
  - android.permission.INSTALL_PACKAGES
  - android.permission.DELETE_PACKAGES
  - android.permission.READ_INSTALL_SESSIONS
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.HIDE_NON_SYSTEM_OVERLAY_WINDOWS
  - android.permission.USE_RESERVED_DISK
  - android.permission.UPDATE_APP_OPS_STATS
  - android.permission.MANAGE_APP_OPS_MODES
  - android.permission.INTERACT_ACROSS_USERS_FULL
  - android.permission.SUBSTITUTE_NOTIFICATION_APP_NAME
  - android.permission.PACKAGE_USAGE_STATS
--------------------------------------------------------------------------------
PACKAGE: com.android.permissioncontroller
PATH:    /system/priv-app/WearPermissionController/WearPermissionController.apk
UID:     10020
VERSION: 29 system image
PERMISSIONS:
  - android.permission.MANAGE_USERS
  - android.permission.GRANT_RUNTIME_PERMISSIONS
  - android.permission.REVOKE_RUNTIME_PERMISSIONS
  - android.permission.ADJUST_RUNTIME_PERMISSIONS_POLICY
  - android.permission.WHITELIST_RESTRICTED_PERMISSIONS
  - android.permission.INTERACT_ACROSS_USERS_FULL
  - android.permission.OBSERVE_GRANT_REVOKE_PERMISSIONS
  - android.permission.UPDATE_APP_OPS_STATS
  - android.permission.MANAGE_APP_OPS_MODES
  - android.permission.GET_APP_OPS_STATS
  - android.permission.KILL_UID
  - android.permission.MANAGE_APP_OPS_RESTRICTIONS
  - android.permission.RECEIVE_BOOT_COMPLETED
  - android.permission.HIDE_NON_SYSTEM_OVERLAY_WINDOWS
  - android.permission.MANAGE_ROLE_HOLDERS
  - android.permission.OBSERVE_ROLE_HOLDERS
  - android.permission.SET_PREFERRED_APPLICATIONS
  - android.permission.ACCESS_SHARED_LIBRARIES
  - android.permission.ACCESS_INSTANT_APPS
  - android.permission.REQUEST_INCIDENT_REPORT_APPROVAL
  - android.permission.APPROVE_INCIDENT_REPORTS
  - android.permission.READ_DEVICE_CONFIG
  - android.permission.OPEN_ACCESSIBILITY_DETAILS_SETTINGS
  - android.permission.READ_PRIVILEGED_PHONE_STATE
  - android.permission.SUBSTITUTE_NOTIFICATION_APP_NAME
--------------------------------------------------------------------------------
PACKAGE: com.android.providers.settings
PATH:    /system/priv-app/WearSettingsProvider/WearSettingsProvider.apk
UID:     1000
VERSION: 10
--------------------------------------------------------------------------------
PACKAGE: com.android.server.telecom
PATH:    /system/priv-app/WearTelecom/WearTelecom.apk
UID:     1000
VERSION: 10
PERMISSIONS:
  - android.permission.BIND_CONNECTION_SERVICE
  - android.permission.BIND_INCALL_SERVICE
  - android.permission.BLUETOOTH
  - android.permission.BLUETOOTH_ADMIN
  - android.permission.BROADCAST_CALLLOG_INFO
  - android.permission.BROADCAST_PHONE_ACCOUNT_REGISTRATION
  - android.permission.CALL_PRIVILEGED
  - android.permission.HANDLE_CALL_INTENT
  - android.permission.INTERACT_ACROSS_USERS
  - android.permission.INTERACT_ACROSS_USERS_FULL
  - android.permission.MANAGE_USERS
  - android.permission.MANAGE_ROLE_HOLDERS
  - android.permission.MODIFY_AUDIO_SETTINGS
  - android.permission.MODIFY_AUDIO_ROUTING
  - android.permission.MODIFY_PHONE_STATE
  - android.permission.READ_CALL_LOG: restricted=true
  - android.permission.READ_DEVICE_CONFIG
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.SEND_SMS: restricted=true
  - android.permission.STOP_APP_SWITCHES
  - android.permission.VIBRATE
  - android.permission.WRITE_CALL_LOG: restricted=true
  - android.permission.WAKE_LOCK
  - android.permission.READ_BLOCKED_NUMBERS
  - android.permission.WRITE_BLOCKED_NUMBERS
  - android.permission.SUBSTITUTE_NOTIFICATION_APP_NAME
  - android.permission.PROCESS_CALL_RECORD
  - android.permission.SYSTEM_ALERT_WINDOW
--------------------------------------------------------------------------------

# PARTITION: /vendor
================================================================================
PACKAGE: com.mediatek.sensorhub.ui
PATH:    /vendor/app/SensorHub/SensorHub.apk
UID:     10034
VERSION: 2.1
PERMISSIONS:
  - android.permission.WRITE_MEDIA_STORAGE
  - android.permission.WRITE_EXTERNAL_STORAGE: restricted=true
  - android.permission.READ_EXTERNAL_STORAGE: restricted=true
  - android.permission.DEVICE_POWER
  - android.permission.WAKE_LOCK
  - android.permission.FOREGROUND_SERVICE
  - android.permission.ACTIVITY_RECOGNITION
--------------------------------------------------------------------------------
PACKAGE: com.wiite.wiite_stopwatch
PATH:    /vendor/operator/app/Stopwatch/Stopwatch.apk
UID:     1000
VERSION: 1.0
PERMISSIONS:
  - android.permission.RECEIVE_BOOT_COMPLETED
--------------------------------------------------------------------------------
PACKAGE: com.mediatek.frameworkresoverlay
PATH:    /vendor/overlay/FrameworkResOverlay/FrameworkResOverlay.apk
UID:     10001
VERSION: 10
--------------------------------------------------------------------------------
PACKAGE: android.auto_generated_rro_vendor__
PATH:    /vendor/overlay/framework-res__auto_generated_rro_vendor.apk
UID:     10000
VERSION: 1.0
--------------------------------------------------------------------------------




✦ adb shell "su -c 'ls -la /vendor/etc/ipsec/ipsec.d/cacerts/'"

    1 total 64
    2 drwxr-xr-x 2 root shell 4096 2008-12-31 18:00 .
    3 drwxr-xr-x 3 root shell 4096 2008-12-31 18:00 ..
    4 -rw-r--r-- 1 root root   947 2008-12-31 18:00 DigiCertGlobalRootCA.crt
    5 -rw-r--r-- 1 root root   914 2008-12-31 18:00 DigiCertGlobalRootG2.crt
    6 -rw-r--r-- 1 root root   579 2008-12-31 18:00 DigiCertGlobalRootG3.crt
    7 -rw-r--r-- 1 root root  1500 2008-12-31 18:00 Entrust.net_Certification_Authority_2048.cer
    8 -rw-r--r-- 1 root root  1026 2008-12-31 18:00 GeoTrust_PCA_G3_Root.der
    9 -rw-r--r-- 1 root root   896 2008-12-31 18:00 GeoTrust_Primary_CA.der
   10 -rw-r--r-- 1 root root   690 2008-12-31 18:00 GeoTrust_Primary_CA_G2_ECC.der
   11 -rw-r--r-- 1 root root   867 2008-12-31 18:00 Root-R3.cer
   12 -rw-r--r-- 1 root root  1586 2008-12-31 18:00 Root_CA_1003.crt
   13 -rw-r--r-- 1 root root   683 2008-12-31 18:00 VeriSignClass3G4.der
   14 -rw-r--r-- 1 root root  1239 2008-12-31 18:00 VeriSignClass3G5.der
   15 -rw-r--r-- 1 root root  1213 2008-12-31 18:00 VeriSignUniversalRootCertification.der
   16 -rw-r--r-- 1 root root  1476 2008-12-31 18:00 gold.cer
   17 -rw-r--r-- 1 root root  1060 2008-12-31 18:00 thawte.der

  ---

  adb shell "su -c 'ls -la /vendor/lib64/ | grep -iE \"ril|radio|ims|volte|ccci|mtkril|ss\\.so|mtk-ril|fusion\"'"`

    1 -rw-r--r--  1 root root     56800 2008-12-31 18:00 libccci_util.so
    2 -rw-r--r--  1 root root    739544 2008-12-31 18:00 libcharon-ss.so
    3 -rw-r--r--  1 root root   1787984 2008-12-31 18:00 libcrypto-ss.so
    4 -rw-r--r--  1 root root    280312 2008-12-31 18:00 libcurl-ss.so
    5 -rw-r--r--  1 root root     20208 2008-12-31 18:00 libfeature_rss.so
    6 -rw-r--r--  1 root root     36344 2008-12-31 18:00 libipsec_ims_shr.so
    7 -rw-r--r--  1 root root   4131264 2008-12-31 18:00 libmtk-ril.so
    8 -rw-r--r--  1 root root    341040 2008-12-31 18:00 libmtk-rilop.so
    9 -rw-r--r--  1 root root     15064 2008-12-31 18:00 libmtkrillog.so
   10 -rw-r--r--  1 root root     74512 2008-12-31 18:00 libmtkrilutils.so
   11 -rw-r--r--  1 root root    104512 2008-12-31 18:00 libreference-ril.so
   12 -rw-r--r--  1 root root    325624 2008-12-31 18:00 libril.so
   13 -rw-r--r--  1 root root    845416 2008-12-31 18:00 librilfusion.so
   14 -rw-r--r--  1 root root     19160 2008-12-31 18:00 librilutils.so
   15 -rw-r--r--  1 root root    447936 2008-12-31 18:00 libssl-ss.so
   16 -rw-r--r--  1 root root     23400 2008-12-31 18:00 libtinycompress.so
   17 -rw-r--r--  1 root root   2634024 2008-12-31 18:00 vendor.mediatek.hardware.mtkradioex@1.0.so
   18 -rw-r--r--  1 root root    891800 2008-12-31 18:00 vendor.mediatek.hardware.mtkradioex@1.1.so
   19 -rw-r--r--  1 root root    910728 2008-12-31 18:00 vendor.mediatek.hardware.mtkradioex@1.2.so
   20 -rw-r--r--  1 root root    776816 2008-12-31 18:00 vendor.mediatek.hardware.mtkradioex@1.3.so
   21 -rw-r--r--  1 root root    702544 2008-12-31 18:00 vendor.mediatek.hardware.mtkradioex@1.4.so
   22 -rw-r--r--  1 root root    970960 2008-12-31 18:00 vendor.mediatek.hardware.mtkradioex@1.5.so
   23 -rw-r--r--  1 root root    552992 2008-12-31 18:00 vendor.mediatek.hardware.radio_op@1.1.so
   24 -rw-r--r--  1 root root    249848 2008-12-31 18:00 vendor.mediatek.hardware.radio_op@1.2.so
   25 -rw-r--r--  1 root root    344848 2008-12-31 18:00 vendor.mediatek.hardware.radio_op@1.3.so
   26 -rw-r--r--  1 root root    283888 2008-12-31 18:00 vendor.mediatek.hardware.radio_op@1.4.so
   27 -rw-r--r--  1 root root    642952 2008-12-31 18:00 vendor.mediatek.hardware.radio_op@2.0.so

  product/app/ — GMS components:
    FaceUnlockLauncher, GoogleLocationHistory, Maps,
    ModuleMetadataGooglePrebuilt, webview

  product/priv-app/ — Privileged:
    GmsCore, GoogleServicesFramework, Phonesky, Velvet — GMS stack
    WearSettings, WearSystemUIWithLegacyRecents, WearCommon — ODM system UI
    WearDeviceDeamPix 
    WearMtkContacts, WearMtkDialerIncallUi, WearTelecom — telephony
    WearOOBE, WearSystemMode — system management
    AdupsFota 
    MtkCarrierConfig, Omacp — carrier config
    CarrierServices, ConfigUpdater, GoogleOneTimeInitializer
    StorageManager

  product/overlay/:
    HomeThemeDM101, SdkDM101, WearSdkIconOverlay,
    WearSdkIconOverlay_Wiiwear — ODM theme overlays
    GmsConfigOverlay, GmsGoogleDocumentsUIOverlay, GmsGsaConfigOverlay,
    GoogleExtServicesConfigOverlay — GMS overlays

  

#SYSTEM APKs (54) 

  system/app/:
    WearBluetooth, WearScreenLock, WearFactoryMode, WearEngineerMode
    YGPS — GPS test utility
    DebugLoggerUI, LocationEM2 — MTK debug apps
    KeyChain, SecureElement — system security
    AutoDialer, BatteryWarning, WearCalculator, WearHTMLViewer,
    WearPinyinIme, CaptivePortalLogin, CarrierDefaultApp, Gba,
    GoogleExtShared, PacProcessor, SimAppDialog
    framework-res.apk, mediatek-res.apk — framework resources

  system/priv-app/:
    WearSystemUIWithLegacyRecents, WearPermissionController
    WearPackageInstaller, WearSettingsProvider, WearMediaProvider
    WearDocumentsUI, WearTelecom, WearEngineerMode, WearMtkMms
    MtkTelephonyProvider, MtkTeleService, MtkMmsService, MtkMmsAppService
    MtkDownloadProvider, MtkDownloadProviderUi, MtkTelephonyAssist
    MtkCapCtrl, MtkContactsProvider
    HeilsFaceUnlockDM101 — proprietary face unlock 
    ImsService, LPPeService — IMS/VoLTE
    WapiCertManager, VpnDialogs, ProxyHandler, StatementService
    UserDictionaryProvider, Shell, InputDevices
    GoogleNetworkStack, GoogleNetworkPermissionConfig
    GoogleExtServicesPrebuilt
    DynamicSystemInstallationService, ExternalStorageProvider
    FusedLocation

  Full catalogue with package names, UIDs, permissions:
  [requires pm dump from live device for each APK]
  WearDeviceDeamPix — DETAILED ANALYSIS 

  Location    : product/priv-app/WearDeviceDeamPix/WearDeviceDeamPix.apk
  Role          : Primary nRF52832 co-processor bridge daemon
  Status      : KEEP — hardware-critical

  Native libraries confirmed [VERIFIED — unzip -l]:
    lib/arm64-v8a/libSerialPort.so (10,072 bytes)
    lib/armeabi-v7a/libSerialPort.so (18,004 bytes)

  Layout files confirmed:
    activity_feature_dfu.xml — Device Firmware Update activity
    activity_main.xml
    incall_ui.xml
    activity_data_test.xml

  CRITICAL FINDING: This APK is not just a sensor bridge — it contains
  DFU (Device Firmware Update) capability. Combined with libSerialPort.so
  it can flash new firmware to the nRF52832 over ttyS1. This is the
  definitive "owner" of the nRF52832 co-processor.

  ROM disposition: KEEP in initial ROM for hardware functionality.
  wiite_bridge_daemon Phase 2 replaces this. (See Section 13.8)

  Live process: com.wiite.devicedaemon:communication (PID 2411)
    FD 52 = /dev/ttyS1 (confirmed open, live UART connection) 
    5 active sockets: FDs 3, 45, 46, 47, 58 (IPC to other watch apps)

################################################################################
# SECTION 13 — ODM CONTROL LAYER
################################################################################

# 13.1 PRIMARY CONTROLLER 

  Device: /sys/class/misc/wiite_corp_ctrl/
  Driver: wiite_corp (built into kernel, char 10:49)

WIITE_CORP_CTRL — COMPLETE 34-ATTRIBUTE INTERFACE MAP
══════════════════════════════════════════════════════
Attribute          Access    Value / State
──────────────────────────────────────────────────────
acc_dir            R         Sensor Direction: 0x0 (default axis mapping)
altitude_value     R         Altitude value: 0 (idle)
baro_enable        R/W       DISABLED
baro_value         R         Baro value: 0 (idle)
bprs_simu          W-ONLY    Blood pressure simulation trigger
btMAC              R         ca:77:87:c7:16:00
chip_id            R         0x70 (nRF52832 confirmed)
command            W-ONLY    Primary command interface
config             R         Capability bitmap [32-bit]:
                             pos 14,15,23,29 = 1, all others = 0
delay              R         pix_delay: 50ms polling interval
dev                —         Kernel device node reference
event_simu         W-ONLY    Sensor event simulation trigger
hrs_simu           W-ONLY    Heart rate simulation trigger
i2c_log            R         i2c_log_en: 0 (logging disabled)
i2c_test           R         (not yet read)
lcm_type           R         Unsupport
local_rtc          R         Raw encoded: 279172874156--464502511--5 -171311752:127:-1
longterm           R         NO SUPPORT
nrf_rtc            R         Live RTC synchronized (confirmed running)
nrf_sys_mode       R         0 (normal operation)
power              —         Kernel sysfs directory — not a file
rst_op             W-ONLY    nRF52832 hardware reset trigger
spo2_simu          W-ONLY    SpO2 simulation trigger
step_simu          W-ONLY    Step counter simulation trigger
subsystem          —         Kernel symlink — not readable
tempearture_value  R         Temperature value: 0 (idle)
tilt_simu          W-ONLY    Tilt/gesture simulation trigger
uart_op            R         uart enable: 0 (UART debug bridge disabled)
uevent             —         Kernel uevent node
upgrade_op         W-ONLY    nRF firmware upgrade trigger
user_command       W-ONLY    User-level command interface
vdd18_con          W-ONLY    1.8V rail control (I/O error on read — write-only)
version            R         48059 (nRF firmware version)
vibr_test          R         NO SUPPORT — no vibration motor hardware
i2c_test           W-ONLY    I2C bus test trigger (I/O error on read — write-only)

    

  Permissions: all nodes 0666 (system:system)

  SECURITY NOTE: 0666 permissions mean any process on the device including
  any installed APK can write directly to the nRF52832 via ttyS1.
  There is zero hardware-level isolation for the watch's core health functions.

# 13.2 CONNECTION CONTROLLER 

  Device: /sys/class/misc/wiite_con_ctrl/
  Driver: wiite_con (built into kernel, char 10:51)
    pin_set : RW — manual GPIO/IRQ pin control
    irq_test: RW — interrupt line verification

# 13.3 SYSFS INIT SEQUENCE 

  STEP 1: write uart_op = "1"
  STEP 2: openUart() sequence:
    write uart_op = "1"
    write command = "7"   → readback must = "1"
    write command = "8"   → readback must = "1"
    write command = "243" → NRF_READ_PUT_ON_STATE
    write command = "12"  → read tilt last value
    write command = "17"  → read silent mode value
    write command = "13"  → read zen mode value
  STEP 3: write nrf_rtc = datetime string
  STEP 4: write longterm = "9", "1", DFU_PACKAGE_VERSION
  STEP 5: write btMAC = bandAddress
  STEP 6: write command = "1,height,weight,20,sex,target"
  STEP 7: write command = "14,iTime"
  STEP 8: write command = "15,metric,temper"

# 13.4 WIRE PROTOCOL 

  Frame: [00 FF] [SEQ] [LEN_MSB] [LEN_LSB] [COMMAND] [DATA...] [CRC16_MSB] [CRC16_LSB]
  CRC16: CCITT-Standard over [COMMAND + DATA]
  
  Heart rate interrupt raw bytes 
    0x00 0x00 0x02 0x01

 

# 13.5 UART CONFIGURATION 

  Port    : /dev/ttyS1
  Baud    : 115200
  Format  : 8N1
  Flow    : None
  Mode    : Raw binary (-icanon, -echo, -opost)
  Min read: 1 byte
  Permissions: 0666 — set intentionally by init.mt6762.rc

# 13.6 SQLITE DATABASE SCHEMA 

  File    : /data/data/com.wiite.wearhealthuart/databases/database.db
  Version : 5
  Table   : health

  CREATE TABLE health (
    _id         INTEGER PRIMARY KEY AUTOINCREMENT,
    steps       INTEGER NOT NULL,
    date        DATE NOT NULL,
    diatance    INTEGER NOT NULL,    ← ODM TYPO: "diatance" MUST be preserved
    kcal        INTEGER NOT NULL,
    long_time   INTEGER NOT NULL,
    targetsteps INTEGER NOT NULL
  );

  NOTE: Consumer APKs expect schema version 5 with the typo.
        Any replacement daemon must reproduce this exact schema.

# 13.7 IPC ARCHITECTURE 

  com.wiite.devicedaemon:communication (PID 2411, ttyS1 owner)
    → FD 52 = /dev/ttyS1 (live UART connection to nRF52832)
    → 5 active sockets: IPC to other watch apps (not external network)
    → publishes via Unix domain sockets to devicedaemon (parent process)
    → devicedaemon coordinates via Android binder
    
    Socket inodes [VERIFIED — ss output live device]:
    26422, 22183, 22184, 22191, 32034

  wearhealthuart
    → reads from database.db via WalkProvider ContentProvider
    → content://com.wiite.wearhealthuart.provider/insert

  AppWhitelistProvider [VERIFIED — Gemini live analysis]:
    URI: content://com.wiite.home.provider.AppWhitelistProvider/user
    Role: Dynamic whitelist for WearCleanTask process killer

# 13.8 wiite_bridge_daemon SPECIFICATION [VERIFIED — Appendix F]

  Name        : wiite_bridge_daemon
  Replaces    : com.wiite.devicedaemon:communication (PID 2411)
  Run as      : ROOT
  Language    : C++
  Phase 1     : Replace devicedaemon:communication — own ttyS1
  Phase 2     : Replace consumer APKs with clean open-source UI

  SELinux domain required:
    Domain: wiite_bridge
    Required rules:
      allow wiite_bridge ttyS_device:chr_file { open read write ioctl };
      allow wiite_bridge wiite_corp_ctrl_sysfs:file { open read write getattr };
      allow wiite_bridge self:capability { sys_rawio };
      allow wiite_bridge app_data_file:dir { write add_name };
      allow wiite_bridge app_data_file:file { create write open };
    File context:
      /vendor/bin/wiite_bridge_daemon u:object_r:wiite_bridge_exec:s0

# 13.9 FRAMEWORK INTEGRATION 

  CRITICAL FINDING: Topwise has forked AOSP at the framework level.
  "wiite" strings confirmed baked into:
    framework.jar
    services.jar
    mediatek-services.jar

  This means the ODM integration is not limited to APKs and HALs —
  it is injected into the Android OS core system services.

  Confirmed framework patch:
    android.content.res.Configuration class contains non-standard method:
      Configuration.isSpecialApp(String pkgName)
    This method is NOT part of AOSP. It is an ODM injection.
    It is called from AbSharedUtil.java isInsideApp() method.
    It returns true for apps on a hidden protected list.
    This hidden list does not appear in any Android settings UI.

  ROM implication: A clean LineageOS 18.1 framework.jar will NOT have
  these hooks. The sensor stack and hardware functions must be verified
  to work without them, or the hooks must be replicated cleanly.
  This is a critical pre-ROM-build verification requirement.

 
  WiitePackageManagerUtil 
    Class: com/android/server/pm/WiitePackageManagerUtil
    Method: slientGrantRuntimePermission
    Logic: Reads registry file and auto-grants ALL requested permissions
           including INSTALL_PACKAGES to listed apps without user interaction.
    Registry: /system/etc/permissions/pms_sysapp_grant_permission_list.txt

  WiiteSystemCommonService 
    Class: com/wiite/WiiteCommon/WiiteSystemCommonService
    Logic: Non-standard system service running inside system_server.
           Provides private IPC hub for ODM apps to bypass standard
           Android security boundaries.

  Power key hijack [VERIFIED — strings services.jar]:
    Strings: com.wiite.power.doubleressed
             com.wiite.power.longpressed
    Logic: PhoneWindowManager patched at OS level to intercept physical
           button events and trigger custom ODM activities.

  CustomPropInterface 
    Class: com/mediatek/custom/CustomPropInterface
    Logic: Uses reflection to spoof Browser User Agent and Build Release Date.

# 13.10 ODM PROCESS PROTECTION SYSTEM 

  Source: AbSharedUtil.java — isInsideApp() and isUnClearApp() methods
  Used by: qs.wiite.cleantask (WearCleanTaskPro) — the system task killer

  HARDCODED UNKILLABLE APPS (never killed regardless of memory pressure):
    com.adups.fota          — OTA/spyware hub 
    com.android.stk        — SIM Tool Kit
    qs.wiite.cleantask     — the killer itself (self-protecting)
    com.tiamaes.watch.activation — device activation/tracking 

  FRAMEWORK HOOK:
    if (Configuration.isSpecialApp(pkgName)) return true; // Protected
    This hidden list is not user-configurable.

  SOCIAL APP PASS (isUnClearApp):
    Tencent QQ, WeChat, Twitter, Facebook, WhatsApp — explicitly protected
    These cannot be killed by WearCleanTask regardless of RAM state.

  ROM implication: These protections live in the framework and in the
  WearCleanTaskPro APK. A clean ROM strips WearCleanTaskPro entirely
  and does not replicate the framework hook.

################################################################################
# SECTION 14 — SELinux POLICY
################################################################################

# 14.1 CURRENT SELINUX STATE [VERIFIED]

  Mode     : PERMISSIVE 
  Target   : ENFORCING in final ROM (post-boot, post-verification)
  Version  : 29.0 

# 14.2 STOCK POLICY FILES 

  vendor/etc/selinux/:
    vendor_sepolicy.cil               
    plat_pub_versioned.cil             
    plat_sepolicy_vers.txt            
    precompiled_sepolicy              
    vendor_file_contexts              
    vendor_hwservice_contexts          
    vendor_mac_permissions.xml         
    vendor_property_contexts          
    vendor_seapp_contexts              
    vendor_sepolicy.cil               
    vndservice_contexts            

  Ramdisk policy files 
    sepolicy (compiled binary)
    vendor_file_contexts
    vendor_property_contexts
    plat_file_contexts
    plat_property_contexts

# 14.3 CONFIRMED DOMAIN REQUIREMENTS

  wiite_corp_ctrl sysfs nodes:
    AVC denial confirmed: system_app → wiite_corp_ctrl/command
    Required: allow system_app wiite_corp_ctrl_sysfs:file rw

  wiite_bridge_daemon — new domain 

  Full domain map:

# 14.4 PRECOMPILED POLICY ANALYSIS

  vendor/etc/selinux/precompiled_sepolicy — requires decompile via sesearch
  or checkpolicy to produce human-readable .te files.
  Domain transition graph — how init transitions to all domains, any
  domains with broad sys_rawio access beyond wiite_bridge.
  Status: 

# 14.5 ROM POLICY BUILD METHODOLOGY 

  Phase 1: Boot ROM permissive — mandatory
  Phase 2: Capture AVC denials: adb shell su -c "dmesg | grep 'avc.*denied'"
  Phase 3: Convert: audit2allow -i denials.txt > device_generated.te
  Phase 4: Review — remove overly broad allows
  Phase 5: Rebuild — repeat until clean boot, zero denials
  Phase 6: Set BOARD_USE_ENFORCING_SELINUX := true — rebuild — verify

################################################################################
# SECTION 15 — BUILD SYSTEM SPECIFICATION
################################################################################

# 15.1 BOARDCONFIG.MK 

  # Architecture 
  TARGET_BOARD_PLATFORM              := mt6765
  TARGET_BOOTLOADER_BOARD_NAME       := WP_C17S_PIX_TFT_D4
  TARGET_CPU_ABI                     := arm64-v8a
  TARGET_CPU_ABI2                    := armeabi-v7a
  TARGET_ARCH                        := arm64
  TARGET_ARCH_VARIANT                := armv8-a
  TARGET_CPU_VARIANT                 := cortex-a53
  TARGET_2ND_ARCH                    := arm
  TARGET_2ND_ARCH_VARIANT            := armv8-a
  TARGET_2ND_CPU_ABI                 := armeabi-v7a
  TARGET_2ND_CPU_ABI2                := armeabi
  TARGET_2ND_CPU_VARIANT             := cortex-a53

  # Kernel — prebuilt 
  TARGET_PREBUILT_KERNEL             := device/wiite/c17s/prebuilt/Image.gz
  BOARD_KERNEL_IMAGE_NAME            := Image.gz
  BOARD_PREBUILT_DTBOIMAGE           := device/wiite/c17s/prebuilt/dtbo.img


  BOARD_KERNEL_BASE          := 0x40000000
  BOARD_KERNEL_OFFSET        := 0x00080000
  BOARD_RAMDISK_OFFSET       := 0x11b00000
  BOARD_KERNEL_TAGS_OFFSET   := 0x07880000
  BOARD_DTB_OFFSET           := 0x07880000
  BOARD_KERNEL_PAGESIZE      := 2048
  BOARD_BOOTIMG_HEADER_VERSION := 2
  BOARD_FLASH_BLOCK_SIZE     := 131072
  BOARD_KERNEL_IMAGE_NAME    := kernel
  # Kernel cmdline 
  BOARD_KERNEL_CMDLINE := console=tty0 console=ttyS0,921600n1 \
      vmalloc=400M slub_debug=OFZPU page_owner=on swiotlb=noforce \
      androidboot.hardware=mt6765 maxcpus=8 loop.max_part=7 \
      firmware_class.path=/vendor/firmware has_battery_removed=0 \
      androidboot.boot_devices=bootdevice,11230000.mmc root=/dev/ram \
      androidboot.verifiedbootstate=orange bootopt=64S3,32N2,64N2 \
      buildvariant=user gpt=1 mrdump_ddrsv=yes \
      mrdump_cb=0x10e800,0x1400

  # AVB — disabled
  BOARD_AVB_ENABLE                   := false

  # Dynamic partitions 
  BOARD_SUPER_PARTITION_SIZE         := 4294967296
  BOARD_SUPER_PARTITION_GROUPS       := wiite_dynamic_partitions
  BOARD_WIITE_DYNAMIC_PARTITIONS_PARTITION_LIST := system vendor product
  BOARD_WIITE_DYNAMIC_PARTITIONS_SIZE := 4292870144

  # Partition filesystems 
  BOARD_SYSTEMIMAGE_FILE_SYSTEM_TYPE          := ext4
  BOARD_VENDORIMAGE_FILE_SYSTEM_TYPE          := ext4
  BOARD_PRODUCTIMAGE_FILE_SYSTEM_TYPE         := ext4
  BOARD_SYSTEMIMAGE_PARTITION_RESERVED_SIZE   := 33554432
  BOARD_VENDORIMAGE_PARTITION_RESERVED_SIZE   := 33554432
  BOARD_PRODUCTIMAGE_PARTITION_RESERVED_SIZE  := 33554432

  # Display 
  TARGET_SCREEN_DENSITY              := 160

  # Filesystem 
  BOARD_USES_METADATA_PARTITION      := true
  TARGET_USERIMAGES_USE_EXT4         := true
  TARGET_USERIMAGES_USE_F2FS         := true

  # Treble 
  BOARD_VNDK_VERSION                 := current

  # SELinux 
  BOARD_SEPOLICY_VERS                := 29.0
  BOARD_PLAT_PRIVATE_SEPOLICY_DIR    := device/wiite/c17s/sepolicy/private
  BOARD_PLAT_PUBLIC_SEPOLICY_DIR     := device/wiite/c17s/sepolicy/public

  # Output paths
  TARGET_COPY_OUT_VENDOR             := vendor
  TARGET_COPY_OUT_PRODUCT            := product

  # DTBO 
  BOARD_DTBOIMG_PARTITION_SIZE       := 8388608

# 15.2 DEVICE_C17S.MK — CORE [VERIFIED]

  PRODUCT_NAME         := lineage_c17s
  PRODUCT_DEVICE       := c17s
  PRODUCT_BRAND        := Wiite
  PRODUCT_MODEL        := LOKMAT APPLLP 5 MAX
  PRODUCT_MANUFACTURER := SPRD

  # Treble 
  PRODUCT_FULL_TREBLE_OVERRIDE        := true
  PRODUCT_VNDK_VERSION                := 29
  PRODUCT_EXTRA_VNDK_VERSIONS         := 29

# 15.3 PRODUCT_PROPERTY_OVERRIDES 

  ro.hardware=mt6762
  ro.board.platform=mt6765
  ro.vendor.mediatek.platform=MT6765
  ro.vndk.version=29
  ro.product.first_api_level=29
  ro.build.version.sdk=29
  ro.build.version.release=10
  ro.treble.enabled=true
  ro.zygote=zygote64_32
  ro.sf.lcd_density=160
  qemu.hw.mainkeys=0
  ro.vendor.wlan.gen=gen4m
  wifi.interface=wlan0
  wifi.direct.interface=p2p0
  wifi.tethering.interface=ap0
  ro.product.bt.name=APPLLP_5_MAX
  ro.default_wifi_hotspot=APPLLP_5_MAX
  vendor.rild.libpath=mtk-ril.so
  vendor.rild.libargs=-d /dev/ttyC0
  ro.opengles.version=196610
  ro.hardware.egl=mtk
  ro.crypto.state=encrypted
  ro.crypto.type=file
  ro.crypto.volume.filenames_mode=aes-256-cts
  persist.vendor.connsys.chipid=0x6765
  persist.fingerprint.gms=Xiaomi/olive/olive:10/QKQ1.191014.001/\
    1749802324:user/release-keys__2020-06-01
  ro.build.version.security_patch=2024-03-05
  ro.vendor.build.security_patch=2020-04-05
  ro.com.wiite.dcpu=true
  ro.device.qr.id=9
  persist.wiite.band.home.support=true
  persist.wiite.home.gesture.icon=true
  persist.wiite.band.home.custom.made=true
  ro.frp.pst=/dev/block/platform/bootdevice/by-name/frp
  ro.product.locale=en-US
  ro.build.display.id=APPLLP_5_MAX_LMT_WTC17S_EN_480_640_V1.5_20250613
  [Additional properties: requires full getprop dump cross-reference]

# 15.4 FRAMEWORK OVERLAYS

  config_density_override            = 186
  config_showNavigationBar           = true (qemu.hw.mainkeys=0)
  config_defaultNotificationSoundUri = pixiedust.ogg
  config_defaultAlarmAlertUri        = Alarm_Classic.ogg
  config_defaultRingtoneUri          = World.ogg
  config_batteryCapacity             = 2946 [VERIFIED — bbattery_history.txt]

# 15.5 PRODUCT_COPY_FILES 


################################################################################
# SECTION 16 — SECURITY ANALYSIS
################################################################################
#
# PURPOSE:
#   Forensic security analysis of all ODM applications and suspicious
#   framework patches. Every system-UID APK must be analyzed.
#   No app clears without evidence. No lib clears based on name alone.
#
################################################################################

# 16.1 STOPWATCH.APK 

  Location    : vendor/operator/app/Stopwatch/Stopwatch.apk
  Verdict     : NOT A STOPWATCH. Persistent system-level surveillance agent.
  sharedUserId: android.uid.system (UID 1000 — full system access)
  debuggable  : true (production app)

  Capabilities confirmed (static analysis):
    IMEI/IMSI/Device ID collection (DeviceUtils, PhoneUtils)
    Arbitrary shell command execution (ShellUtils)
    WiFi network scanning (SSID, BSSID, gateway, IP, MAC)
    Silent APK installation (AppUtils — CRITICAL)
    RSA/AES/DES encryption capability
    Boot persistence (BOOT_COMPLETED receiver with HOME category)
    Vendor partition installation (survives factory reset)
    Exported service without permission guard (privilege escalation surface)

  External references in DEX:
    223.5.5.5 (Alibaba DNS), www.baidu.com, com.tencent.mtt.fileprovider

  Outbound network exfiltration: UNCONFIRMED — requires tcpdump capture
  ROM disposition: STRIP — first priority vendor cleanup
  Full analysis: Stopwatch_APK_Security_Analysis.pdf

# 16.2 AdupsFota.APK 

  Location    : product/priv-app/AdupsFota/AdupsFota.apk
  Company     : ADUPS Technology — Chinese OTA update provider
  History     : 2016 FTC case — BLU devices found transmitting IMEI, IMSI,
                full SMS content, and contact lists to Chinese servers silently.
  Status on this device: Hardcoded as UNKILLABLE in AbSharedUtil.java.
                WearCleanTask will never kill this process regardless of
                memory pressure. This is not accidental.
  Priority    : HIGH — full static analysis required
  ROM disposition: STRIP — pending dynamic analysis confirmation
  Dynamic analysis: tcpdump on boot to confirm/deny outbound connections
  
  Additional permissions confirmed [VERIFIED — dumpsys package]:
    WRITE_SECURE_SETTINGS, REBOOT
    FcmService — persistent push channel (always-on remote command receiver)
    

# 16.3 com.tiamaes.watch.activation

  Location    : product/ (exact path requires pm dump)
  Company     : Tiamaes — Chinese watch ODM closely related to Topwise/Wiite
                Supplies same hardware platform to LOKMAT and other brands.
  Role        : Device activation and registration service
  Behavior    : Phones home to Tiamaes servers on first boot to register
                **for what**
  Status on this device: Hardcoded as UNKILLABLE in AbSharedUtil.java.
  Legal exposure: GDPR violation if device sold to EU customers without
                  explicit consent for IMEI transmission.
                  Device is sold on Amazon globally.
  ROM disposition: STRIP — pending analysis to confirm payload
  Analysis required: APK decompile + tcpdump network capture on first boot

# 16.4 IPSEC/STRONGSWAN LIBRARIES

  Files:
    /vendor/lib64/libcrypto-ss.so
    /vendor/lib64/libssl-ss.so
    /vendor/lib64/libcurl-ss.so
    /vendor/lib64/libcharon-ss.so

  Known: Library names match Strongswan IKEv2 IPsec components used
         for VoLTE WiFi calling (ePDG) functionality. This is a
         legitimate use case on Android devices.

  Unknown: Whether these are stock Strongswan or ODM-modified with
           additional functionality. Given that this device has confirmed
           system UID apps with IMEI collection capability, no ODM library
           receives a free pass based on name recognition alone.

  Status: 
  Required: Symbol table comparison against stock Strongswan, dlopen
            call analysis, dynamic network capture during WiFi calling.

# 16.5 CONFIGURATION.isSpecialApp() FRAMEWORK PATCH

  Type        : AOSP framework modification — hidden method injection
  Class       : android.content.res.Configuration
  Method      : isSpecialApp(String pkgName) — NON-STANDARD, NOT IN AOSP
  Location    : framework.jar (system partition)

  Behavior: Returns true for apps on a hidden ODM-defined protected list.
            Called from AbSharedUtil.isInsideApp() in WearCleanTask.
            Protected apps cannot be killed by system task management.
            This list is not visible to the user in any Android settings UI.

  Additional framework patches confirmed in:
    services.jar — "wiite" strings confirmed present
    mediatek-services.jar — "wiite" strings confirmed present

  ROM implication: A clean LineageOS 18.1 framework.jar will not have
  this method. The WearCleanTaskPro APK is stripped in clean ROM, so the
  missing method will not be called. However, if any retained APK calls
  Configuration.isSpecialApp() directly, it will crash on clean ROM.
  All retained APKs must be audited for this call. [BLOCKED — requires
  JADX decompile of all KEEP APKs to verify no references]

# 16.6 HeilsFaceUnlockDM101.APK 

  Location    : system/priv-app/HeilsFaceUnlockDM101/
  Role        : Proprietary face unlock UI layer
  Note        : No credential enrolled on this device — unused at runtime
  Analysis    : What camera data does it access? What does it store?
                What does it transmit?
  Status      

# 16.7 ADDITIONAL SYSTEM-UID APKs [FLAGGED]

  All APKs running as android.uid.system require analysis.
  Full list requires: adb shell pm list packages -f | xargs pm dump
  to identify all system UID packages.
  Status: 
  
# 16.9 HASHED ICCID PERSISTENCE 

  Property : persist.vendor.radio.ss.hashed_last_iccid1
  Value    : 7f562cd58bb777374ef5918d86439501
  Location : NVRAM (persists across factory reset)
  Logic    : Persists a hashed unique identifier of the user's SIM card
             to NVRAM. Creates a permanent device-to-user link that
             survives factory reset and SIM removal.
  Risk     : Allows re-identification of a device after factory reset
             by any process that can read persist.* properties.

################################################################################
# SECTION 17 — ROM BUILD GATE
################################################################################

# 17.1 SECTION COMPLETION GATE

  [ ] Section 0  — Partition architecture verified, file counts confirmed
  [ ] Section 1  — All 85 RC files catalogued with deep content analysis
  [ ] Section 2  — All HALs documented, VINTF verified from clean pull
  [ ] Section 3  — Crypto chain verified, OS bubble strategy finalized
  [ ] Section 4  — Full sensor stack documented, both stacks complete
  [ ] Section 5  — Display and GPU stack complete, egl.cfg content verified
  [ ] Section 6  — Camera stack complete, all blobs catalogued
  [ ] Section 7  — Audio stack complete, all XMLs catalogued
  [ ] Section 8  — Radio stack complete, all daemons and deps documented
  [ ] Section 9  — Connectivity complete, all firmware files verified
  [ ] Section 10 — Complete VNDK-29 and VNDK-SP-29 inventory
  [ ] Section 11 — All firmware blobs catalogued with hardware identification
  [ ] Section 12 — All 127 APKs catalogued with full metadata and UID
  [ ] Section 13 — ODM control layer complete — all 34 attributes, full protocol,
                   framework integration mapped
  [ ] Section 14 — SELinux policy mapped, precompiled_sepolicy decompiled,
                   ROM methodology confirmed
  [ ] Section 15 — BoardConfig and device.mk complete (COPY_FILES last)
  [ ] Section 16 — All items resolved:
                     AdupsFota analyzed
                     tiamaes.activation analyzed
                     libcrypto-ss et al Ghidra complete
                     Configuration.isSpecialApp() impact on retained APKs confirmed
                     HeilsFaceUnlock analyzed
                     All system-UID APKs analyzed
  [ ] Section 17 — All gate items checked

# 17.2 BINARY AUDIT GATE

  [ ] Complete vendor blob dependency map — all 1,468 vendor files
  [ ] All blobs linking ODM-modified libs identified (libcrypto, libhidlbase, libutils)
  [ ] All dlopen call targets confirmed (stripped blobs via Ghidra)
  [ ] Crypto isolation OS strategy finalized and documented
  [ ] patchelf target list complete
  [ ] libcrypto-ss, libssl-ss, libcurl-ss, libcharon-ss cleared or flagged

# 17.3 FRAMEWORK AUDIT GATE

  [ ] services.jar decompiled — exact wiite class list documented
  [ ] framework.jar decompiled — Configuration.isSpecialApp() implementation documented
  [ ] All KEEP APKs verified — no calls to Configuration.isSpecialApp()
  [ ] ROM framework strategy confirmed — replicate or eliminate hooks

# 17.4 HARDWARE PARITY GATE

  [ ] All values in Section 15 traceable to physical hardware evidence
  [ ] No value borrowed from similar device
  [ ] Kernel cmdline carries verbatim from verified live device capture
  [ ] fstab md_udc metadata mount confirmed (not mmcblk0p11)
  [ ] FBE confirmed fscrypt (not dm-crypt, not LUKS)

# 17.5 APP DISPOSITION GATE

  [ ] All 127 APKs have ROM disposition (KEEP / STRIP / REPLACE)
  [ ] All system-UID APKs analyzed for security risk
  [ ] nRF52832 sensor bridge app set confirmed (KEEP list complete)
  [ ] Security APKs confirmed stripped: Stopwatch, AdupsFota, tiamaes.activation
  [ ] No app carried forward without documented verification of its role

# 17.6 PRE-BUILD CHECKLIST

  [ ] All section completion items above checked
  [ ] All binary audit items above checked
  [ ] All framework audit items above checked
  [ ] All hardware parity items above checked
  [ ] All app disposition items above checked
  [ ] PRODUCT_COPY_FILES generated from verified data
  [ ] Build environment confirmed clean

# 17.7 POST-BUILD VERIFICATION

  [ ] Output image size within partition limits
  [ ] Boot image header values match verified raw binary constants
  [ ] All crypto blobs present in vendor output
  [ ] VINTF manifest matches device implementation
  [ ] First boot: permissive SELinux, AVC capture running
  [ ] All HALs registered — vndservicemanager lists all declared interfaces
  [ ] nRF52832 probe completes (7.26s delay) — no watchdog abort
  [ ] /data decrypts on first boot (md_udc key material found)
  [ ] WiFi, Bluetooth, GPS operational
  [ ] Sensors operational — heart rate, SpO2, barometer live
  [ ] Camera operational — gc030a, gc2385, bf2553L initialized
  [ ] USB ADB accessible (VID 0x0E8D / PID 0x201D)
  [ ] Battery fuel gauge accurate (2946 mAh)
  [ ] GMS fingerprint spoof accepted (Xiaomi olive)
  [ ] Zero kernel panics on first 10 boots
  [ ] tcpdump confirms no outbound connections from stripped packages

################################################################################
# END OF MASTER OUTLINE v2.0

# Status: ACTIVE — sections being populated from verified data.
# Nothing marked [VERIFIED] without physical evidence and source citation.
################################################################################
