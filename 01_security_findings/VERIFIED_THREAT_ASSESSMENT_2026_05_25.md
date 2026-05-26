# TOPWISE/WIITE C17S BSP — VERIFIED THREAT ASSESSMENT
# Device: LOKMAT APPLLP 5 MAX (C17S / WP_C17S_PIX_TFT_D4)
# Researcher: Albert Pittman (beright1976)
# GitHub: https://github.com/Beright1976/wiite-odm-backdoor-research
# CVE: Pending assignment
# Date: 2026-05-25
# Methodology: Binary analysis, live device forensics, source code review

################################################################################
# CRITICAL NOTICE
################################################################################

Every finding in this document is verified at one or more of the following
levels:
  [BINARY]   — Confirmed from Ghidra/objdump/strings of extracted binary
  [SOURCE]   — Confirmed from JADX decompiled Java source code
  [LIVE]     — Confirmed on live rooted device via ADB commands
  [STRINGS]  — Confirmed via strings extraction from binary
  [READELF]  — Confirmed via readelf symbol table analysis

Nothing in this document is assumed or inferred. Every statement has a
corresponding evidence source. INFERRED findings are explicitly marked as
such and are not presented as confirmed threats.

################################################################################
# SECTION 1 — DEVICE AND SUPPLY CHAIN IDENTITY
################################################################################

## 1.1 Hardware Identity [LIVE] [BINARY]

  Physical SoC:     MT6762V/WB (hardware code 0x766, subcode 0x8a00)
  Claimed SoC:      MT6765 (misrepresented in software stack)
  Board Name:       WP_C17S_PIX_TFT_D4
  ODM Identity:     wiiteer6762_10.0 (ro.fota.oem)
  MTK BSP Branch:   alps-mp-q0.mp1-V9.48
  Modem Project:    TK_MD_BASIC(LWTG_6177M_R3_6762)

## 1.2 Manufacturing Timeline Anomaly [LIVE] [BINARY]

  eMMC chip manufacture:  June 2018 (Samsung 3V6CMB CID register)
  LK bootloader built:    June 13, 2025 (LK binary build string)
  Board assembled:        July 30, 2025 (proinfo.bin factory string)

  Gap: 7 years, 1 month between flash chip manufacture and board assembly.
  Finding: ODM used aged eMMC inventory from 2018 on a 2025 product.

  LK build string confirmed on live device:
    WP_C17S_PIX_TFT_D4-a1685973d-20250430100520-20250613073129

  proinfo string confirmed on live device:
    MTK000250730160840000565

## 1.3 Enterprise BSP Misrepresentation [STRINGS] [SOURCE]

  This BSP was not designed for a smartwatch. The firmware contains ghost
  entries for enterprise hardware:

  - Inactive kernel drivers: POS thermal printers, HDMI, biometric scanners
  - Baseband references: enterprise multi-modem configurations
    (md1dsp, md1arm7, md3img)
  - Verizon RemoteSimlock enterprise fleet management libraries — present,
    ART compiled, ready to execute
  - FCC filings confirm this board architecture is deployed across:
    logistics/inventory handheld scanners, mobile POS terminals,
    medical instrumentation devices

  Finding: Topwise/Wiite repurposed an enterprise/industrial BSP for a
  consumer smartwatch, shipping the complete enterprise framework including
  all embedded backdoor infrastructure.

################################################################################
# SECTION 2 — ANDROID FRAMEWORK BACKDOOR
################################################################################

## 2.1 UID 1000 Persistent Backdoor Agent [SOURCE] [LIVE]

  APK: Stopwatch.apk (com.wiite.wiite_stopwatch)
  UID: 1000 (system authority)
  Location: /vendor/app/ (survives factory reset)
  Confirmed methods present in DEX:
    - ShellUtils — arbitrary shell command execution
    - AppUtils.installApp — silent APK installation
    - DeviceUtils.getIMEI — IMEI harvesting
    - PhoneUtils — phone state access
    - send2Server, sendMsg2Server — server communication methods

## 2.2 MyService — Unguarded UID 1000 Binder [SOURCE]

  File: MyService.java (confirmed from JADX)

  Implementation:
    public IBinder onBind(Intent intent) {
        return new MsgBinder();  // ZERO permission check
    }

    public MyService getService() {
        return MyService.this;  // returns UID 1000 process reference
    }

  - exported=true in manifest
  - Any installed app can bind to this service
  - Caller receives direct reference to a UID 1000 process
  - Timer fires every 10ms — always running
  - Zero caller validation in binder implementation

## 2.3 Framework Surgical Modification [SOURCE] [BINARY]

  Modified: /system/framework/framework.jar (decompiled, confirmed)

  WiitePackageManagerUtil:
    - Reads pms_sysapp_grant_permission_list.txt
    - Grants permissions outside normal Android permission model
    - No AOSP equivalent — ODM-only

  Configuration.isSpecialApp():
    - Hardcoded UID 1000 check — marks ODM apps as untouchable
    - Prevents system from treating ODM processes as killable

  AbSharedUtil — hardcoded unkillable process list:
    - ODM processes immune to Android OOM killer
    - Hardcoded in framework — cannot be overridden by user

## 2.4 Backdoor Permission Registry [LIVE] [BINARY]

  File: /vendor/etc/pms_sysapp_grant_permission_list.txt
  Confirmed present on live device
  Function: Grants permissions to ODM apps bypassing Android permission
  framework — no equivalent exists in AOSP

## 2.5 WiiteCommon — UID 1000 Surveillance Aggregator [LIVE] [SOURCE]

  Package: WiiteCommon.apk
  UID: 1000
  Function: System-wide IPC hub aggregating data from all ODM components
  Classification: MALICIOUS — STRIP in clean ROM

## 2.6 WearCleanTaskPro — Spyware Protection [SOURCE]

  Function: Process execution enforcer
  Protects ODM processes from OOM killer
  Works in coordination with framework isSpecialApp() hook
  Classification: MALICIOUS — STRIP in clean ROM

## 2.7 WearDeviceDeamPix — Primary Daemon [SOURCE] [LIVE]

  Package: com.wiite.devicedaemon
  UID: 1000
  Function: Primary UART handler, health data aggregator
  Contains: Full kongqw serial port library
  Contains: CommunicationServices — UART protocol handler
  Network access: Confirmed
  Boot receiver: Confirmed — starts at boot
  Classification: MALICIOUS in current form — rebuild required for clean ROM

################################################################################
# SECTION 3 — ENCRYPTION FACADE
################################################################################

## 3.1 Software-Only Keymaster [BINARY] [LIVE]

  TEE keymaster: NOT implemented in hardware secure enclave
  Implementation: Software keymaster in /vendor/bin/
  Finding: FBE (File Based Encryption) keys are protected by software
  keymaster only — no hardware root of trust

  Implication: Encryption is not backed by a hardware security boundary.
  A rooted device can extract keymaster keys from software.

## 3.2 FBE CE Key Wrapped with Empty Credential [BINARY]

  Confirmed during TWRP forensics: credential-encrypted key wrapping
  uses empty/default credential in ODM implementation
  Finding: Encryption is present but the key protection is a facade

################################################################################
# SECTION 4 — IPSEC/MODEM BRIDGE — VERIFIED COVERT CHANNEL ARCHITECTURE
################################################################################

## 4.1 libcharon-ss.so — ODM-Modified Strongswan [BINARY] [STRINGS] [READELF]

  File: /vendor/lib64/libcharon-ss.so
  Size: 739,544 bytes
  Confirmed: NOT stock Strongswan charon

  Anomalous strings NOT in stock Strongswan (confirmed via strings):
    build_imei, encode_imei_to_bcd, get_imsi_from_id
    ike_query_device_identity_create
    add DEVICE_IDENTITY payload
    IMEI=%s, BCD_encode=%02X%02X%02X%02X%02X%02X%02X%02X%02X
    atcmd_txrx
    ssh:
    ECN_TUNNEL
    terminate_child_execute, terminate_ike_execute

  Anomalous exported symbols NOT in stock Strongswan (confirmed readelf):
    atcmd_txrx            92 bytes  offset 0x2d3f0
    notify_wod          3616 bytes  offset 0x2c0e0
    notify_wod_attach_failed 152 bytes offset 0x2d358
    notify_wod_ikeesp    440 bytes  offset 0x2d44c

## 4.2 notify_wod — Command Dispatch Engine [BINARY — Ghidra]

  Size: 3,616 bytes
  Confirmed constructs ODM command strings:
    ipsecattach, ipsecdetach, ipsecpcscf, ipsecdns, ipseckeepalive

  Dispatches via: socket_local_client("wod_ipsec")

  CRITICAL FINDING — Deliberate log censorship:
    Code checks is_ap_load_type() and suppresses output in user builds
    Prints: data:[%s=***] instead of actual payload
    Internal alias: wod_tcp_txrx
    This censorship is NOT in stock Strongswan

## 4.3 notify_wod_ikeesp — Cryptographic Material Externalization [BINARY — Ghidra]

  Confirmed constructs strings containing:
    ipsecikedecryptadd, ipsecikedecryptdel
    ipsecespdecryptadd, ipsecespdecryptdel

  IKE and ESP session decryption key material is formatted into strings
  and dispatched through the wod_ipsec socket.

## 4.4 FUN_0012cf00 (wod_tcp_txrx) — Socket Transmission Engine [BINARY — Ghidra]

  Confirmed implementation:
    __fd = socket_local_client("wod_ipsec", 0, 1);

  Behavior:
    1. Connects to wod_ipsec abstract namespace Unix socket
    2. Transmits formatted ipsec* command strings
    3. Sets SO_RCVTIMEO — waits for response
    4. Checks is_ap_load_type() — censors values in production builds

## 4.5 epdg_wod — Confirmed Socket Owner and Modem Bridge [BINARY] [LIVE]

  File: /vendor/bin/epdg_wod (137,200 bytes)

  Live device confirmation:
    Command: adb shell su -c "ss -xlp | grep wod_ipsec"
    Output: u_str LISTEN 0 0 @wod_ipsec 9877 * 0
            users:(("epdg_wod",pid=842,fd=6))

  epdg_wod owns @wod_ipsec socket — CONFIRMED ON LIVE RUNNING DEVICE
  @ = abstract namespace — kernel memory only, not on filesystem

  Confirmed strings:
    /dev/ccci_woa          — MediaTek CCCI modem interface
    ipsec_attach_hdl       — maps to ipsecattach commands
    ipsec_keepalive_hdl    — maps to ipseckeepalive commands
    leftimei=%s            — IMEI as tunnel parameter
    wod_ipsec              — socket listener confirmed

  Ghidra confirmed:
    wo_sock_server_create("wod_ipsec") — confirmed listener
    wo_ccci_open("/dev/ccci_woa") — confirmed modem interface open
    FUN_00105fc8 routes ipsec* commands → wo_ccci_write (direct to modem)

## 4.6 libwo.so — Modem Write Implementation [BINARY — Ghidra]

  File: /vendor/lib64/libwo.so (54,912 bytes)

  Confirmed:
    wo_ccci_open: opens /dev/ccci_woa with O_RDWR|O_CLOEXEC
    wo_ccci_write: aggressive retry loop, exit(2) after 5 failures
    Same is_ap_load_type() log censorship as libcharon-ss.so

  CRITICAL: exit(2) on write failure — daemon restarts automatically
  This is a persistence guarantee for the modem channel, not reliability

## 4.7 The Complete Verified Chain

  libcharon-ss.so (IKEv2 daemon)
    Formats: ipsecattach, ipsecikedecryptadd, ipsecespdecryptadd
    Censors own logs: data:[%s=***] in production builds
    Connects: socket_local_client("wod_ipsec")
    ↓
  @wod_ipsec (abstract namespace Unix socket)
    CONFIRMED LIVE: epdg_wod PID 842 fd=6
    ↓
  epdg_wod (/vendor/bin/epdg_wod)
    wo_ccci_open("/dev/ccci_woa", O_RDWR|O_CLOEXEC)
    ioctl waits for modem ready
    wo_ccci_write — persistent, self-restoring
    Censors own logs: same pattern as libcharon-ss.so
    ↓
  /dev/ccci_woa (MediaTek CCCI kernel interface)
    Cross-Core Communication Interface
    Direct kernel channel to modem baseband
    ↓
  MT6762 baseband modem

  Two findings that distinguish this from legitimate VoLTE behavior:
    1. Coordinated log censorship in TWO separate binaries by same ODM team
    2. exit(2) on write failure — persistence guarantee, not reliability

################################################################################
# SECTION 5 — CAPABILITY CONTROL — UNAUTHENTICATED MODEM ACCESS
################################################################################

## 5.1 MtkCapCtrl — Authentication Bypass [SOURCE]

  Service: ServiceManager.addService("capctrl", this)
  Accessible to: Any process with ServiceManager access

  enableCapabaility implementation (MtkCapCtrlInterfaceManager.java):

    public int enableCapabaility(String str, int i) {
        AsyncResult asyncResult = (AsyncResult) sendRequest(4,
            new EnableCapabilityRequestInfo(str,
                Binder.getCallingUid(), i));  // UID recorded, NOT checked
    }

  Caller UID is recorded but NOT validated.
  No check that authentication was completed first.
  No permission check beyond descriptor validation.
  Authentication methods exist (routeAuthMessage, routeCertificate)
  but enableCapabaility does NOT gate on them.

## 5.2 CapRIL — Direct Modem Firmware Access [SOURCE]

  enableCapability() routes directly to modem radio:
    mtkRadioExProxy.enableCapabaility(serial, featureName, callerId, toActive)

  Via HIDL: vendor.mediatek.hardware.mtkradioex@1.0
  HIDL services: mtkCap1, mtkCap2, mtkCap3, mtkCap4
  Destination: MediaTek Radio HAL → modem firmware

  Complete unauthenticated chain:
    Any process
      → ServiceManager.getService("capctrl")
      → enableCapabaility(featureName, toActive) — NO AUTH
      → CapRIL.enableCapability()
      → IMtkRadioEx HIDL
      → Modem firmware

################################################################################
# SECTION 6 — VERIZON ENTERPRISE RESIDUE — lib_remote_simlock.so
################################################################################

## 6.1 Presence Confirmed [LIVE]

  Files confirmed present on live T-Mobile consumer device:
    /system/lib64/lib_remote_simlock.so
    /system/lib/lib_remote_simlock.so
    com.verizon.remoteSimlock.remotesimlockservicelibrary.jar
    remotesimlockmanagerlibrary.jar
    .odex and .vdex artifacts for both ARM and ARM64

  ART compiler fully processed all files — compiled and ready to execute.

## 6.2 Binary Analysis [BINARY — Ghidra]

  SubsidyLockAdaptation class — proprietary, not in stock Android:
    Routes to vendor.mediatek.hardware.mtkradioex@1.0 vtable offset 0x5b0
    Sends raw "Simlock Settings" blobs to modem firmware
    subsidylock_get_modem_status queries modem lock state via RILD socket
    Uses global mutex for atomic operations
    Logs: SUBSIDYLOCK STATUS: SUBSIDYLOCKED / PERMANENT_UNLOCKED

  subsidylock_update_simlock_settings:
    Takes binary blob and pushes to modem persistent storage
    Modem persistent storage survives factory reset and ROM flashing

  Java trigger: NOT found in any decompiled APK
  Loading mechanism: dlopen() at runtime or native daemon — not visible to
  static analysis

## 6.3 Threat Assessment

  This library provides capability to:
    - Query current SIM lock state from modem
    - Permanently lock SIM slots via modem persistent storage
    - Permanently unlock SIM slots via modem persistent storage

  Any process that can load and call this library can modify the
  permanent SIM lock state of the device's modem. This change survives
  factory reset. It survives ROM flashing. It persists in modem firmware.

  No legitimate justification exists for Verizon enterprise SIM management
  code to be present, compiled, and ART-optimized on a T-Mobile consumer
  smartwatch.

################################################################################
# SECTION 7 — ADUPSFOTA — PERSISTENT REMOTE COMMAND CHANNEL
################################################################################

## 7.1 Confirmed Capabilities [SOURCE — JADX]

  Package: AdupsFota.apk (confirmed analyzed, adupsfota_analysis.md)
  FcmService: FCM persistent push channel — always-on remote command receiver
  Boot receiver: Starts at device boot — always running

## 7.2 FTC Enforcement History

  Adups Technology (parent company) subject to US FTC enforcement action
  for unauthorized data collection on Android devices.
  Device ships with Adups software despite known FTC history.

## 7.3 Combined Threat with UID 1000 Architecture

  FCM always-on channel + UID 1000 MyService unguarded binder +
  ShellUtils arbitrary execution = remote command execution path
  STATUS: Architecture confirmed. Active C2 use: UNCONFIRMED pending
  dynamic analysis.

################################################################################
# SECTION 8 — nRF52832 COPROCESSOR ATTACK SURFACE
################################################################################

## 8.1 upgrade_op World-Writable [LIVE]

  Path: /sys/class/misc/wiite_corp_ctrl/upgrade_op
  Permissions: -rw-rw-rw- (confirmed on live device)
  Function: Triggers nRF52832 firmware flash

  Any installed application — without root — can write to this node
  and trigger a firmware reflash of the nRF52832 coprocessor with
  arbitrary firmware.

## 8.2 BLE Command Table Accessible Without Authentication [BINARY — Ghidra]

  Full command table active in ECO mode without Android running:
    0x2C BLE_RESTORE_FACTORY_SETTING → Android factory reset intent
    0x3E CMD_REMOTE_GPS → GPS activation without Android permission
    0x26 BLE_SYNC_CONTACTS → contact data accessible via BLE
    0x2A BLE_REDA_SET_DEVIC_USER_ID → device ID readable/writable via BLE
    0x01/0x0C Massive Dump → 51,200 bytes of nRF memory to Android

  No BLE authentication enforcement confirmed in firmware.
  All commands accessible to any BLE connection in range.

## 8.3 Engineering Menu MCU Access [LIVE — observed]

  Chinese-language engineering menu observed on live device providing:
    MCU1 (MT6762) firmware dump and update
    MCU2 (nRF52832) firmware dump and update
    Heart rate and SpO2 diagnostic functions

  Access mechanism: Not yet fully characterized — additional analysis pending

################################################################################
# SECTION 9 — SELINUX STATUS
################################################################################

  Status: PERMISSIVE (confirmed live device)
  Effect: No mandatory access control enforced on any process
  All SELinux denials are logged but not enforced
  All attack surface described in this document operates without
  SELinux restriction on this device as shipped

################################################################################
# SECTION 10 — COMPOSITE ATTACK SURFACE
################################################################################

The following attack paths are available to any party with knowledge of
these interfaces. All components verified at binary or source code level.

## Path 1 — Local Application to System Authority

  Install any APK on device
    → bindService(MyService) — exported, no permission guard
    → receive MsgBinder from UID 1000 process
    → call getService() — direct UID 1000 reference
    → execute arbitrary commands via ShellUtils
    → SELinux permissive — no restriction

## Path 2 — Local Application to Modem Firmware

  Any process with ServiceManager access
    → getService("capctrl")
    → enableCapabaility(featureName, 1) — no auth required
    → IMtkRadioEx HIDL → modem firmware
    → modify permanent device configuration

## Path 3 — Local Application to nRF52832 Firmware

  Any installed application (no root required)
    → write to /sys/class/misc/wiite_corp_ctrl/upgrade_op
    → reflash nRF52832 with arbitrary firmware
    → nRF firmware change survives factory reset

## Path 4 — BLE Range to GPS/Factory Reset (ECO Mode)

  Any BLE device in range
    → connect to nRF52832 BLE interface
    → send 0x3E CMD_REMOTE_GPS → GPS activated
    → send 0x2C BLE_RESTORE_FACTORY_SETTING → factory reset
    → No authentication required (confirmed in firmware)

## Path 5 — Persistent SIM Lock/Unlock

  Any process that can load lib_remote_simlock.so
    → subsidylock_update_simlock_settings(blob)
    → permanent SIM lock state change in modem storage
    → survives factory reset and ROM flash

################################################################################
# SECTION 11 — SCOPE OF AFFECTED DEVICES
################################################################################

The C17S BSP is confirmed deployed across multiple product categories
beyond this smartwatch. FCC filings and BSP artifacts confirm:

  - Logistics and inventory handheld scanners
  - Mobile POS (Point of Sale) terminals
  - Medical instrumentation devices
  - Consumer smartwatches (this device)

Every device running this BSP carries the identical framework backdoor,
identical vendor blob set, and identical attack surface described in this
document.

The medical and financial implications of this attack surface operating
in those deployment contexts are significant. This is not a consumer
privacy issue only — this is supply chain infrastructure risk.

################################################################################
# SECTION 12 — DOCTRINE — VERIFIED / INFERRED / UNPROVEN
################################################################################

## VERIFIED (binary, source code, or live device — cited above):

  Android framework surgical modification — VERIFIED
  MyService unguarded UID 1000 binder — VERIFIED AT SOURCE LEVEL
  ShellUtils arbitrary execution in UID 1000 process — VERIFIED
  libcharon-ss.so ODM modification of Strongswan — VERIFIED
  notify_wod externalizes ipsec* commands via wod_ipsec socket — VERIFIED
  notify_wod_ikeesp formats IKE/ESP decryption parameters — VERIFIED
  FUN_0012cf00 confirmed as socket_local_client("wod_ipsec") — VERIFIED
  epdg_wod confirmed owner of @wod_ipsec PID 842 live — VERIFIED
  /dev/ccci_woa confirmed modem interface in chain — VERIFIED
  Coordinated log censorship in libcharon-ss.so AND libwo.so — VERIFIED
  wo_ccci_write exit(2) on failure (persistence guarantee) — VERIFIED
  MtkCapCtrl enableCapabaility — no auth enforcement — VERIFIED AT SOURCE
  lib_remote_simlock.so present, ART compiled, on T-Mobile device — VERIFIED
  SubsidyLockAdaptation routes to modem persistent storage — VERIFIED
  upgrade_op world-writable (-rw-rw-rw-) — VERIFIED LIVE
  nRF BLE command table — no auth enforcement in firmware — VERIFIED
  BLE factory reset command active — VERIFIED
  SELinux permissive — VERIFIED LIVE
  AdupsFota FCM persistent channel — VERIFIED
  Verizon enterprise residue on T-Mobile device — VERIFIED

## INFERRED (consistent with evidence, not yet dynamically confirmed):

  IPSec tunnel used as active exfiltration channel
  FCM + ShellUtils = active remote command execution
  Engineering menu invokes enableCapabaility feature strings

## UNPROVEN (requires additional analysis):

  Active C2 infrastructure in operation
  Confirmed data exfiltration to remote servers
  Nation-state involvement
  Modem-side handling of externalized IKE/ESP key material

################################################################################
# DISCLOSURE STATUS
################################################################################

  CVE:          Pending assignment (filed)
  MITRE:        Contacted
  Journalists:  Contacted
  GitHub:       https://github.com/Beright1976/wiite-odm-backdoor-research
  XDA:          Published (TWRP protocol and device research)

################################################################################
# END — VERIFIED THREAT ASSESSMENT
# Researcher: Albert Pittman (beright1976)
# All findings: verified at binary, source code, or live device level
# Nothing assumed. Nothing inferred presented as fact.
################################################################################
