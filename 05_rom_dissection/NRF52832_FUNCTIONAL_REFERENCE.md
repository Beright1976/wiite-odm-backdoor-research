# nRF52832 COPROCESSOR — COMPLETE FUNCTIONAL REFERENCE
# Device: LOKMAT APPLLP 5 MAX (C17S / WP_C17S_PIX_TFT_D4)
# Purpose: ROM Development Reference — nRF52832 integration requirements
# Author: Albert Pittman (beright1976)
# Sources: 187_187.bin firmware analysis, live device sysfs, hardware database
# Date: 2026-05-25
# Status: VERIFIED — binary confirmed, live device confirmed

################################################################################
# SECTION 1 — CHIP IDENTITY
################################################################################

Chip:           Nordic Semiconductor nRF52832
Role:           Dedicated sensor coprocessor — always-on health subsystem
Flash:          512KB internal flash (independent of eMMC)
RAM:            64KB
Firmware:       187_187.bin (153,832 bytes) — /system/system/firmware/
Version:        48059 (confirmed live via /sys/class/misc/wiite_corp_ctrl/version)
Architecture:   ARM Cortex-M4F

The nRF52832 operates independently of the MT6762 main SoC. It has its own
firmware, its own clock, and its own power domain. It is always-on even when
Android is in low-power or ECO mode.

################################################################################
# SECTION 2 — DUAL OPERATING MODE ARCHITECTURE
################################################################################

## Mode 1 — Full Android Mode
  MT6762 fully active
  All radios active: LTE, WiFi, BLE
  Full Android stack running
  nRF52832 reports sensor data via UART to Android
  Battery: approximately 1 day

## Mode 2 — ECO/Sport Mode
  MT6762 radios reduced or suspended
  nRF52832 takes primary operational control
  Health tracking continues independently on nRF52832
  BLE remains active for phone companion connection
  Battery: approximately 3 days
  CRITICAL: Full BLE command table accessible in ECO mode

################################################################################
# SECTION 3 — HARDWARE INTERFACE TO MT6762
################################################################################

## Primary Interface — UART
  Port:       ttyS1
  Protocol:   Custom framed packet protocol with CRC validation
  Direction:  Bidirectional
  Library:    com.kongqw.serialportlibrary (embedded in WearDeviceDeamPix)
  Control:    nRF controls UART state via [TWIS] UART ON/OFF commands

## Secondary Interface — I2C (TWIS — Two Wire Interface Slave)
  Role:       nRF operates as I2C slave to MT6762 master
  Purpose:    Low-level hardware command channel
  Confirmed:  [TWIS] CMD_RECOVER OK in firmware strings

## sysfs Control Interface — wiite_corp_ctrl
  Driver:     wiite_corp kernel module
  Path:       /sys/class/misc/wiite_corp_ctrl/
  Attributes: 34 total (full list in hardware database)

  Critical attributes for ROM development:
    version      r    nRF firmware version (current: 48059)
    nrf_sys_mode rw   Operating mode (0=normal)
    upgrade_op   rw   Firmware flash trigger — world writable (-rw-rw-rw-)
    command      rw   Raw command channel
    user_command rw   User-space command channel
    ble_mac      r    BLE MAC address
    nrf_log      r    nRF log output

  SECURITY NOTE: upgrade_op is -rw-rw-rw- — any process can reflash nRF firmware
  For clean ROM: this MUST be restricted to system or root only

################################################################################
# SECTION 4 — OPTICAL SENSOR STACK
################################################################################

Heart Rate / SpO2 Sensor: VC30F (confirmed from firmware init string)
  Init string: [HRS] hrs_sensor_driver_init_start VC30F
  NOT VC32 — Gemini initial assessment corrected by binary evidence

SpO2 Processing:
  store_spo_data — SpO2 data storage
  BLE_SPO2_SWITCH 0/1 — enable/disable
  BLE_SPO_MONITOR ON/OFF interval — monitoring intervals

Heart Rate Processing:
  Three processing paths: 11111hrs_data, 22222hrs_data, 333333hrs_data
  hrs_sensor_driver_init — driver initialization
  SYNC_HRS_HIS_DATA — historical sync to Android

Motion/Step Tracking:
  Accelerometer data in firmware
  CMD_STEP_HISTORY — step history retrieval
  CMD_TODAY_24H_STEP — today's step count
  CMD_ALL_SLEEP_DATA — sleep data (flagged TEST ONLY in firmware)

IR/Temperature:
  COMMAND_IR_DATA on Android side receives: ir_int, ir2_int, ir3_int
  ByteUtil.irConvert() processes raw IR bitstream (0x80 framing)

ANCS (Apple Notification Center Service):
  [ANCS] DISCOVERY_PRIMARY_ALL_FINISH_ANCS
  Full iOS companion app integration confirmed in firmware

################################################################################
# SECTION 5 — BLE COMMAND TABLE (COMPLETE)
################################################################################

Source: Ghidra decompile of 187_187.bin dispatcher (FUN_10009968)
All hex codes are BLE message IDs received by nRF from companion phone/SoC

  0x26  BLE_SYNC_CONTACTS         Contact list synchronization
  0x28  BLE_UUID_HIS_DATA         Historical data synchronization
  0x29  BLE_METRIC_SYSTEM         Unit system toggle (Metric/Imperial)
  0x2A  BLE_REDA_SET_DEVIC_USER_ID Device/User ID operations
                                   Sub-cmd 0x01: Set
                                   Sub-cmd 0x03: Get
                                   Sub-cmd 0x04: Clear
  0x2B  BLE_SYNCHRONIZE_FILES     General file transfer bridge
  0x2C  BLE_RESTORE_FACTORY_SETTING Factory reset trigger → FUN_10008d1a(0xb)
  0x2F  BLE_TURN_WRIST            Wrist-wake accelerometer configuration
  0x30  BLE_SLEEP_OTHER           Sleep monitoring sub-command
  0x31  BLE_ORTHER_AUTO_UPDATA    Firmware update state check
  0x32  BLE_SYNC_WEATHER          Weather data push to watch
  0x33  BLE_ORTHER_AUTO_UPDATA    Triggers FUN_10006b68(1)
  0x34  BLE_ORTHER_AUTO_UPDATA    Triggers FUN_10006b68(0)
  0x35  BLE_TEST_TEMPERATURE      Temperature sensor diagnostic
  0x37  BLE_SYNC_DESKTOP_INFO     Launcher/UI state sync
  0x38  BLED_READ_DATA_PACK       Raw data packet read from chip
  0x3A  BLE_SPO2_SWITCH           Blood oxygen sensor toggle
  0x3E  CMD_REMOTE_GPS            Remote GPS trigger → UART code 0x39
  0x80  Proprietary/PXI           Calls FUN_1000aaec

## Hidden Command Prefix — 0x80 (FUN_1000dc34)
  Sub-cmd 0x01/0x0B: Sends UART 0x50, calls FUN_10008d1a(0xb)
  Sub-cmd 0x01/0x0C: MASSIVE DUMP — 200-cycle loop, 256 bytes/iteration
                     Total: 51,200 bytes of nRF internal memory to MT6762

################################################################################
# SECTION 6 — UART PROTOCOL TABLE (COMPLETE)
################################################################################

Source: 187_187.bin firmware strings + Ghidra analysis of FUN_1000abae callers

## nRF → MT6762 (Outbound from nRF)

  UART Code  Label                   Description
  0x14       Health Monitor State    4-byte state packet, HRS/SpO2 monitoring
  0x21       Weather                 Weather data acknowledgment
  0x22       Contact Sync            Contact data to Android
  0x26       Historical Data Pack    Data pack / dump trigger
  0x39       GPS Control             GPS activation command to MT6762
  0x50       Hidden System Trigger   Low-level SDK reset/trigger
  0x80       IR/Biometric Bitstream  Raw PPG/IR data in ByteUtil.irConvert format

## MT6762 → nRF (Inbound to nRF via UART)

  Label                   Description
  CMD_ANDROID_INFO        Android system info pushed to nRF
  CMD_DELETE_DEV_ID       Delete device ID from nRF storage
  CMD_GET_DEVICE_ID       Retrieve device ID from nRF
  CMD_SYNC_PHONE_BOOK     Phonebook push to nRF storage
  CMD_PHONE_INCALL        Incoming call notification
  CMD_24H_HEART           24hr heart rate command
  CMD_24H_SPO             24hr SpO2 command
  CMD_SLEEP_DATA          Sleep data retrieval
  CMD_STEP_HISTORY        Step history retrieval
  CMD_TODAY_24H_STEP      Today step data
  CMD_LOG_INFO            Log retrieval
  CMD_TEST_DATA           Diagnostic test data
  CMD_ALL_SLEEP_DATA      Full sleep data (TEST ONLY)
  IOS CMD_REMOVE          iOS companion app removal
  UART_READ_DATA          Raw data read

## Packet Structure
  CRC validation: confirmed ([UART] CRC Check Success / Fail)
  Multi-packet: FUN_1000ac22 breaks data > 256 bytes into 180-byte chunks

################################################################################
# SECTION 7 — DATA STORAGE ON nRF52832
################################################################################

The nRF52832 maintains its own persistent data store independent of Android:

  Device ID storage    — CMD_GET_DEVICE_ID / CMD_DELETE_DEV_ID
  User ID storage      — BLE_REDA_SET_DEVIC_USER_ID (Get/Set/Clear)
  Phonebook/Contacts   — CMD_SYNC_PHONE_BOOK → BLE_SYNC_CONTACTS
  Historical HRS data  — SYNC_HRS_HIS_DATA
  Historical SpO2 data — BLE_HIS_SPO
  Step history         — CMD_STEP_HISTORY
  Sleep history        — CMD_SLEEP_DATA
  Calibration data     — FUN_1000db72 (192 bytes from _DAT_1000df7c)

CRITICAL for ROM development: All this data persists on nRF52832 internal
flash and survives Android factory reset. Clearing nRF storage requires
explicit CMD_DELETE_DEV_ID and factory reset command via BLE/sysfs.

################################################################################
# SECTION 8 — ANDROID-SIDE HANDLER (WearDeviceDeamPix)
################################################################################

Primary UART handler: CommunicationServices.java (com.wiite.devicedaemon)
Serial library: com.kongqw.serialportlibrary

## Android UART Command Handlers (confirmed in CommunicationServices.java)

  Decimal  Constant                    Action
  6        COMMAND_STEP_HISTORY_DATA   Broadcasts BROADCAST_STEP_HISTORY_DATA
  7        COMMAND_NOTIFICATION        ANCS notification parsing
  14       COMMAND_TIME_ZONE           Sets system timezone (SET_TIME_ZONE)
  21       COMMAND_RECOVER_WATCH       Triggers FACTORY_RESET system intent
  22       COMMAND_WATCH_FACE          Downloads/installs watch faces via token
  25       COMMAND_DEVICE_ID           Receives device identifiers from nRF
  26       COMMAND_DELETE_DEVICE_ID    Clears identifiers on nRF
  29       COMMAND_IR_DATA             Receives IR/biometric data
  34       Contact Sync (0x22)         Contact data from nRF
  38       Historical Data (0x26)      Historical data from nRF

################################################################################
# SECTION 9 — ROM DEVELOPMENT REQUIREMENTS
################################################################################

## What MUST be carried from vendor for nRF operation:

  Kernel drivers:
    wiite_corp.ko           — sysfs interface driver (mandatory)
    mediatek,heartrate      — device tree node for nRF interrupt
    
  HAL/Libraries:
    sensors.mt6765.so       — sensor HAL (handles nRF data path)
    libksensor.so           — sensor abstraction
    libsensor_custom.so     — custom sensor implementation
    libem_sensor_jni.so     — JNI bridge for sensor data
    libem_support_jni.so    — support JNI layer
    libSerialPort.so        — UART serial port access
    
  APK (functionality only — ODM backdoor stripped):
    WearDeviceDeamPix       — UART handler (must be rebuilt clean)
                              Strip: ShellUtils, AppUtils, PhoneUtils
                              Strip: WiiteSystemCommonService IPC
                              Keep: CommunicationServices UART protocol
                              Keep: kongqw serial library

  Firmware:
    187_187.bin             — ship in /system/firmware/
                              Delivered to nRF via upgrade_op on first boot

## What MUST be fixed for security in clean ROM:

  upgrade_op permissions:
    Current: -rw-rw-rw- (world writable — any app can reflash nRF)
    Required: -rw-r----- (system:system only)

  nrf_sys_mode permissions:
    Current: needs verification
    Required: root only for write

  command attribute:
    Current: AVC denial captured — system_app attempting access
    Required: Explicit SELinux policy — deny all except wiite_corp service

## What the GSI test confirmed does NOT work without vendor drivers:
  Both sensor stacks fail — wiite_corp driver absent from GSI kernel
  Fix: Carry wiite_corp.ko in vendor/lib/modules/

################################################################################
# SECTION 10 — ECO MODE SECURITY IMPLICATIONS FOR ROM
################################################################################

In ECO mode the full BLE command table is active on the nRF52832
regardless of Android state. This means:

  CMD_REMOTE_GPS (0x3E) — GPS can be triggered via BLE without Android
  BLE_RESTORE_FACTORY_SETTING (0x2C) — factory reset via BLE
  BLE_SYNC_CONTACTS — contact data accessible via BLE
  BLE_REDA_SET_DEVIC_USER_ID — device ID readable via BLE

For clean ROM security:
  BLE pairing must be enforced before any command is accepted
  0x2C factory reset via BLE should require explicit user confirmation
  CMD_REMOTE_GPS should require authenticated BLE session

################################################################################
# END — nRF52832 FUNCTIONAL REFERENCE
# All data: verified from 187_187.bin binary analysis and live device
# No assumptions — binary confirmed or live device confirmed only
################################################################################
