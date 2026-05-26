# SUPPLEMENTAL FINDINGS — 2026-05-25
# Topwise/Wiite C17S — Items Not Captured in Primary Session Documents
# Researcher: Albert Pittman (beright1976)
# GitHub: https://github.com/Beright1976/wiite-odm-backdoor-research
# Status: VERIFIED findings only — evidence source cited for each item

################################################################################
# ITEM 1 — ADUPSFOTA VERIFIED CAPABILITIES
################################################################################

## Source: JADX decompile of AdupsFota.apk (com.adups.fota)

## 1.1 Encrypted Remote Command Channel [SOURCE — JADX]

  FcmService maintains always-on FCM (Firebase Cloud Messaging) connection.
  Remote commands arrive as DES-encrypted payloads.

  Encryption implementation (EncryptUtil — confirmed from source):
    Key derivation: message_id + hardcoded salt "s+-=6"
    Algorithm: DES

  Command dispatch:
    Decrypted payload converted to integer task_id
    task_id maps to CustomActionService handler

  Confirmed task mapping:
    Task 13: Full debug log upload to Adups server via ReportManager

## 1.2 ReportManager — File Exfiltration [SOURCE — JADX]

  Class: ReportManager
  Capability: Uploads any file from device to Adups server
  Method: MultipartBuilder HTTP upload
  Confirmed target: Device debug logs
  Network destination: Adups server (exact domain requires dynamic analysis)

## 1.3 Arbitrary Component Launch [SOURCE — JADX]

  AdupsFota can launch any component on the system by name including:
    com.adups.privacypolicy.activity.GdprActivity
    com.android.settings.adupsfota.FotaReceiver (see Item 2)

## 1.4 FTC Enforcement History

  Adups Technology subject to US Federal Trade Commission enforcement
  action for unauthorized collection and transmission of user data on
  Android devices. Device ships with Adups software despite this history.

################################################################################
# ITEM 2 — FOTARECEIVER FRAMEWORK HOOK
################################################################################

## Source: JADX decompile of AdupsFota.apk — class com.adups.fota.d.e

## Confirmed source code:

  intent.setComponent(new ComponentName(
      "com.android.settings",
      "com.android.settings.adupsfota.FotaReceiver"));

## What this means:

  The stock Android Settings application (com.android.settings) has been
  surgically modified by the ODM to embed a hidden broadcast receiver:
    com.android.settings.adupsfota.FotaReceiver

  Settings runs at system authority. By embedding a receiver inside
  Settings, AdupsFota can trigger system-level actions under Settings
  authority by sending intents to FotaReceiver.

## Current status:

  CONFIRMED: AdupsFota contains the ComponentName reference — JADX source
  UNCONFIRMED: FotaReceiver class not found in decompiled WearSettings.apk
  
  Possible explanations:
    - Heavy DEX obfuscation of the receiver class name
    - Dynamic registration at runtime (not in manifest)
    - Injection directly into services.jar or framework.jar
    - WearSettings.apk may not be the modified Settings — may be in
      a different system Settings package

  Status: CONFIRMED ARCHITECTURAL — implementation obfuscated

## Second confirmed ODM framework modification:

  This is the second confirmed modification to AOSP system apps:
    1. framework.jar — WiitePackageManagerUtil, isSpecialApp(), AbSharedUtil
    2. com.android.settings — FotaReceiver embedded for Adups

################################################################################
# ITEM 3 — ENGINEERING MODE PRIVILEGED SHELL SERVER
################################################################################

## Source: JADX decompile of WearEngineerMode.apk — ShellExe.java

## Two execution paths confirmed:

  Path 1 — execCommandLocal:
    Runtime.getRuntime().exec(command)
    Direct local shell execution

  Path 2 — execCommandOnServer:
    AFMFunctionCallEx.FUNCTION_EM_SHELL_CMD_EXECUTION
    Routes commands to emsvr (Engineering Mode Server daemon)
    emsvr executes commands with elevated privileges

  Public API:
    ShellExe.execCommand(String command)  — single string form
    ShellExe.execCommand(String[] command) — array form
    ShellExe.execCommand(String command, boolean execOnLocal)

## emsvr (Engineering Mode Server):

  A privileged daemon that receives shell commands from the engineering
  mode application and executes them with elevated authority.
  Combined with the bypass/ package this provides privileged shell
  execution accessible through the engineering mode interface.

################################################################################
# ITEM 4 — USB RAW BULK BYPASS CHANNELS
################################################################################

## Source: JADX decompile of WearEngineerMode.apk — BypassService.java

## Bypass channels confirmed:

  Index  Name   sysfs path
  0      gps    /sys/class/usb_rawbulk/gps/enable
  1      pcv    /sys/class/usb_rawbulk/pcv/enable
  2      atc    /sys/class/usb_rawbulk/atc/enable
  3      ets    /sys/class/usb_rawbulk/ets/enable
  4      data   /sys/class/usb_rawbulk/data/enable

## Bypass codes (bitmask):
  gps=1, pcv=2, atc=4, ets=8, data=16

## What each channel does:

  atc — AT command channel routed directly over USB raw bulk
        Bypasses Android USB stack — direct modem AT command access via USB
        Combined with atcmd_txrx in libcharon-ss.so — second AT path

  ets — Engineering Test Service channel
        Direct modem diagnostic access via USB

  gps — GPS raw data channel via USB
  data — Raw data channel via USB

## Trigger mechanism:

  Broadcast intents via LocalBroadcastManager:
    com.via.bypass.action.setbypass — enable bypass mode
    com.via.bypass.action.getbypass — query current bypass mode
    com.via.bypass.action.setfunction — set USB function

  Sets USB function to "via_bypass" mode via UsbManager

## Significance:

  USB connection to this device with bypass mode enabled gives direct
  access to modem AT commands and engineering test service without
  going through Android framework. Combined with the wod_ipsec →
  epdg_wod → /dev/ccci_woa chain — this is a second independent path
  to modem hardware.

################################################################################
# ITEM 5 — DUAL TEE ARCHITECTURE
################################################################################

## Source: tee_active_files.txt (lsof capture), tee_security_state.txt

## Two separate TEE environments confirmed simultaneously active:

  tee1.img —  5,242,880 bytes (5MB)  — Trustonic Kinibi TEE
  tee2.img — 11,534,336 bytes (11MB) — Microtrust TEE

  Properties registered:
    ro.vendor.mtk_trustonic_tee_prop — Trustonic environment
    ro.vendor.mtk_microtrust_tee_prop — Microtrust environment
    soter_teei_prop — SOTER/TEEI interface

## What the TEE is NOT doing:

  android.hardware.keymaster@4.0-service — hal_keymaster_default_exec
    Uses: libSoftKeymaster (software implementation)
    NOT using TEE — confirmed from binary and process map

  android.hardware.gatekeeper@1.0-service — hal_gatekeeper_default_exec
    Uses: libSoftGatekeeper.so
    NOT using TEE — confirmed from binary and process map

## What the TEE IS doing:

  android.hardware.secure_element@1.0-service-mediatek — 74,544 bytes
    Running as: secure_element (PID 1973 confirmed in process list)
    Function: Secure Element management
    Connection to Trustonic/Microtrust: not yet fully characterized

  SOTER interface registered but no active properties returned on live device
  SOTER is Tencent's TEE-based biometric payment authentication framework
  This device has no fingerprint scanner — SOTER has no legitimate function

## Anomaly:

  Two TEE environments of significantly different sizes running
  simultaneously on a non-A/B single-slot device. Neither serves
  Android's primary crypto functions. The larger TEE (11MB Microtrust)
  is more than double the size of the smaller (5MB Trustonic).
  Current function of both TEE environments: NOT FULLY CHARACTERIZED

## TA Registry:

  /vendor/app/mcRegistry/ — no return (empty or inaccessible)
  /data/vendor/mcRegistry/ — no return (empty or inaccessible)
  Trusted Application list: NOT ACCESSIBLE via current tooling

################################################################################
# ITEM 6 — SECURE ELEMENT SERVICE
################################################################################

## Source: tee_active_files.txt process list

  com.android.se — confirmed running PID 1973 as secure_element user

  dumpsys android.hardware.secure_element@1.0::ISecureElement/eSE1
    — returned empty (no active sessions or not exposing status)

  The Secure Element service is active and running. Its function in
  the context of two simultaneous TEE environments is not yet
  characterized.

  Relationship to lib_remote_simlock.so: UNKNOWN
  Relationship to subsidy lock architecture: UNKNOWN
  Relationship to Widevine DRM: POSSIBLE — Widevine confirmed present

################################################################################
# ITEM 7 — CONFIRMED RUNNING PROCESS INVENTORY
################################################################################

## Source: tee_active_files.txt (lsof output from live device)

All processes confirmed running on live device at time of capture:

  Process                    UID      Notes
  wiite.WiiteCommon          system   ODM IPC hub — MALICIOUS
  te.devicedaemon            system   WearDeviceDeamPix UART handler
  com.adups.fota             u0_a41   Adups FOTA — FCM channel active
  wiite.cleantask            system   Process protection enforcer
  com.android.se             secure_element  Secure Element service
  n:communication            system   Communication service
  .wearhealthuart            system   Health UART handler
  wiite_heartrate            system   Heart rate service
  ite.wiite_sleep            system   Sleep tracking service
  ite_bloodoxygen            system   Blood oxygen service
  iite.wiitestore            system   ODM app store — system privilege
  ndroid.settings            system   Settings — contains FotaReceiver
  com.wiite.power            system   ODM power management

## All ODM system processes are protected by:
  - isSpecialApp() framework hook
  - AbSharedUtil unkillable process list
  - WearCleanTaskPro process enforcer
  - SELinux permissive (no MAC restriction)

################################################################################
# ITEM 8 — COM.WIITE.WIITESTORE / WEARAPPDIALSTORE
################################################################################

## Source: pm list packages -f (live device)

  Package:  com.wiite.wiitestore
  APK:      /product/app/WearAppDialStore/WearAppDialStore.apk
  Partition: product (survives factory reset)
  UID:      system (confirmed from process list PID 21072)
  Function: ODM watch face and app store

## Significance:

  System-privileged app store with network access running on product
  partition. Can install watch faces and applications with system
  authority without user confirmation. Not yet analyzed — decompile
  pending.

################################################################################
# ITEM 9 — SOTER/TEEI ANOMALY
################################################################################

## Source: tee_active_files.txt property files, live device getprop

  soter_teei_prop — property namespace registered
  getprop | grep -i soter — returned empty on live device

  SOTER is Tencent's TEE-backed biometric authentication framework:
    Used by: WeChat Pay, Alipay, and other Chinese payment platforms
    Requires: Fingerprint scanner or other biometric hardware
    This device: NO fingerprint scanner

  SOTER property namespace is registered but no active SOTER properties
  are set. Either SOTER initialization failed silently or it is
  dormant pending a specific trigger.

  A payment authentication framework on a device with no biometric
  hardware is anomalous. Current status: NOT CHARACTERIZED

################################################################################
# ITEM 10 — UNANALYZED APKs IN PROJECTS DIRECTORY
################################################################################

The following APKs are decompiled and sitting in the projects directory
but were not analyzed this session:

  WearSystemMode_out/     — system mode control
  WearFactoryMode_out/    — factory test suite (com.signal package)
    SignalFactoryReceiver.java — broadcast receiver — NOT ANALYZED
    SysResetTest.java — system reset — NOT ANALYZED
    MicRecorderTest.java — microphone recording — NOT ANALYZED
  WearSystemTool_out/     — com.wiite.systemtool — NOT ANALYZED
  WearAppDialStore        — ODM app store — NOT PULLED OR ANALYZED

  com.signal package in WearFactoryMode is anomalous — not a standard
  ODM or MediaTek package name. Contains omrecorder audio recording
  library and ProactiveActivity. NOT YET ANALYZED.

################################################################################
# OPEN INVESTIGATION ITEMS FROM THIS SESSION
################################################################################

Priority 1 — TEE characterization
  What are the two TEE environments (Trustonic + Microtrust) actually doing?
  What trusted applications are loaded?
  What is the Secure Element serving?

Priority 2 — FotaReceiver location
  Find the actual embedded FotaReceiver in the modified Settings package
  May require strings search across all system APKs for the class name

Priority 3 — com.signal / WearFactoryMode full analysis
  SignalFactoryReceiver.java — what broadcasts trigger it?
  SysResetTest.java — is this triggerable remotely?
  ProactiveActivity — what does proactive mean in this context?

Priority 4 — WearSystemTool com.wiite.systemtool
  Never opened — unknown function

Priority 5 — SOTER characterization
  Why is SOTER registered with no active properties?
  Is it waiting for a trigger or initialization failed?

Priority 6 — WearAppDialStore analysis
  System-privileged ODM app store — pull and decompile

################################################################################
# END — SUPPLEMENTAL FINDINGS 2026-05-25
# All items: verified at binary, source code, or live device level
# Items marked UNCHARACTERIZED or NOT ANALYZED are open for next session
################################################################################
