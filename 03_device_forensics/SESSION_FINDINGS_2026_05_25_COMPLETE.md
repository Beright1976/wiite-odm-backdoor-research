# SESSION FINDINGS — COMPLETE — 2026-05-25
# Topwise/Wiite C17S — Binary Analysis Session
# Researcher: Albert Pittman (beright1976)
# GitHub: https://github.com/Beright1976/wiite-odm-backdoor-research
# CVE Status: Pending assignment
# Session Type: Forensic binary analysis — responsible disclosure documentation

################################################################################
# PART 1 — BOARD LINEAGE — FULLY VERIFIED
################################################################################

All previously unverified board identity items are now confirmed on live device.

## 1.1 Board Identity Table

| Item                | Value                              | Method                      |
|---------------------|------------------------------------|-----------------------------|
| Physical SoC        | MT6762V/WB — hw 0x766, sub 0x8a00 | BROM register — prior session|
| Target Board Name   | WP_C17S_PIX_TFT_D4                | LK binary — live device     |
| ODM Identity        | wiiteer6762_10.0                   | ro.fota.oem — confirmed prior|
| MTK BSP Branch      | alps-mp-q0.mp1-V9.48              | vendor.build.prop — live    |
| Modem Project       | TK_MD_BASIC(LWTG_6177M_R3_6762)   | baseband — confirmed prior  |

Live commands executed this session:
  adb shell su -c "strings /dev/block/by-name/proinfo | grep -i MTK000"
    → MTK000250730160840000565  CONFIRMED
  adb shell su -c "strings /vendor/build.prop | grep -i alps"
    → ro.vendor.mediatek.version.release=alps-mp-q0.mp1-V9.48  CONFIRMED
  adb shell su -c "strings /proc/lk_env | grep -i WP_C17S"
    → WP_C17S_PIX_TFT_D4-a1685973d-20250430100520-20250613073129  CONFIRMED

## 1.2 Manufacturing Timeline — VERIFIED

| Event               | Date            | Source                          |
|---------------------|-----------------|---------------------------------|
| eMMC chip manufacture| June 2018      | Samsung 3V6CMB CID register     |
| LK source commit    | April 30, 2025  | LK binary build string          |
| LK binary build     | June 13, 2025   | LK binary build string          |
| Board assembly      | July 30, 2025   | proinfo.bin MTK factory string  |

Proinfo string parse — MTK000250730160840000565:
  MTK000 = MediaTek factory prefix
  25     = year 2025
  07     = month July
  30     = day 30
  16     = hour 16
  08     = minute 08
  40     = second 40
  000    = millisecond
  00565  = unit serial/batch counter

Gap: 7 years, 1 month between eMMC manufacture (June 2018) and board
assembly (July 30, 2025). Confirmed cost-down ODM supply chain practice.

################################################################################
# PART 2 — libcharon-ss.so BINARY ANALYSIS — VERIFIED
################################################################################

## 2.1 File Identity

  Path:    /vendor/lib64/libcharon-ss.so
  Source:  super_unpacked/vendor/lib64/ (TWRP-extracted partition pull)
  Size:    739,544 bytes
  Date:    May 7, 2025 15:02

## 2.2 ODM Modification Confirmed — Not Stock Strongswan

Strings NOT present in stock Strongswan charon source confirmed in this binary:

  IMEI/IMSI handling:
    build_imei, encode_imei_to_bcd, get_imsi_from_id
    ike_query_device_identity_create
    add DEVICE_IDENTITY payload
    add DEVICE_IDENTITY payload failed: missing IMEI
    IMEI=%s, BCD_encode=%02X%02X%02X%02X%02X%02X%02X%02X%02X

  Non-Strongswan references:
    atcmd_txrx, ssh:, ECN_TUNNEL
    terminate_child_execute, terminate_ike_execute

Exported symbols NOT in stock Strongswan (confirmed readelf):
  Symbol 39:  atcmd_txrx            92 bytes  FUNC GLOBAL  offset 0x2d3f0
  Symbol 99:  notify_wod          3616 bytes  FUNC GLOBAL  offset 0x2c0e0
  Symbol 146: notify_wod_attach_failed 152 bytes FUNC GLOBAL offset 0x2d358
  Symbol 314: notify_wod_ikeesp    440 bytes  FUNC GLOBAL  offset 0x2d44c

## 2.3 Ghidra Decompile Results — notify_wod

  Size: 3,616 bytes — main payload dispatch engine
  
  Constructs ODM-defined AT-command-style control strings:
    ipsecattach, ipsecdetach, ipsecpcscf, ipsecdns, ipseckeepalive

  Dispatches via: FUN_0012cf00 — confirmed as socket_local_client("wod_ipsec")

  CRITICAL — Deliberate log censorship:
    Code checks is_ap_load_type() and suppresses buffer content in user builds
    Prints: data:[%s=***] instead of actual payload
    This is conscious evasion of forensic logging — not in stock Strongswan
    Internal alias: wod_tcp_txrx

## 2.4 Ghidra Decompile Results — notify_wod_ikeesp

  Size: 440 bytes — IKE ESP notification handler

  Constructs cryptographic material strings:
    ipsecikedecryptadd, ipsecikedecryptdel
    ipsecespdecryptadd, ipsecespdecryptdel

  Takes IKE/ESP key/policy data (param_3) and formats into strings
  passed to FUN_0012cf00 (wod_ipsec socket)

  SIGNIFICANCE: IKE and ESP decryption key material is externalized
  through the wod_ipsec socket to epdg_wod and then to the modem.

## 2.5 Ghidra Decompile Results — FUN_0012cf00 (wod_tcp_txrx)

  Confirmed implementation:
    __fd = socket_local_client("wod_ipsec", 0, 1);

  Behavior:
    1. Connects to wod_ipsec abstract namespace Unix socket
    2. Sends formatted command strings from notify_wod
    3. Sets SO_RCVTIMEO — waits for response
    4. Checks is_ap_load_type() — censors sensitive values in logs
       Prints: data:[%s=***] in user builds

  This is the confirmed transmission engine for all ODM IPSec commands.

## 2.6 atcmd_txrx — Assembly Analysis

  Confirmed as string sanitization wrapper around FUN_0012cf00.
  Trims null terminators from AT command response buffers.
  Not the transmission engine — notify_wod is.

################################################################################
# PART 3 — epdg_wod BINARY ANALYSIS — VERIFIED
################################################################################

## 3.1 File Identity

  Path:    /vendor/bin/epdg_wod
  Size:    137,200 bytes
  Date:    May 7, 2025 15:02

## 3.2 Strings Analysis — Confirmed

  /data/vendor/ipsec/wo/           — working directory
  /dev/ccci_woa                    — MediaTek CCCI modem interface
  wod_ipsec                        — socket listener (confirmed)
  wod_action                       — additional socket
  wod_sim                          — additional socket
  ipsec_attach_hdl                 — maps to ipsecattach commands
  ipsec_detach_hdl                 — maps to ipsecdetach commands
  ipsec_keepalive_hdl              — maps to ipseckeepalive commands
  ipsec_pcscf_dns_hdl              — maps to ipsecpcscf/ipsecdns
  ipsec_dpd_hdl                    — Dead Peer Detection handler
  leftcustcpimei=%s                — IMEI as tunnel configuration parameter
  leftimei=%s                      — IMEI in tunnel left-side identity
  Derive IMSI from IMPI            — IMSI derivation from IMS private identity
  cust_imei_cp                     — custom IMEI copy function
  wod_ipsec                        — socket name confirmation

## 3.3 Ghidra Decompile Results — Socket Hub (FUN_00105254)

  Initializes three server sockets:
    wo_sock_server_create("wod_action", 0, 4)
    wo_sock_server_create("wod_sim", 0, 4)
    wo_sock_server_create("wod_ipsec", 0, 4)  ← confirmed listener

  Opens CCCI modem interface:
    DAT_00131008 = wo_ccci_open("/dev/ccci_woa")

  Enters read loop: wo_ccci_read — bidirectional modem communication.

## 3.4 Ghidra Decompile Results — Command Dispatcher (FUN_00105fc8)

  Routes commands starting with:
    wodem, wosimeauth, wo, +wo, ipsec

  Routing logic:
    IF /dev/ccci_woa bridge active → wo_ccci_write (direct to modem)
    ELSE → send over wod_* sockets

  All ipsec* commands from libcharon-ss.so route to modem via wo_ccci_write.

## 3.5 Live Device Confirmation — Socket Ownership

  Command: adb shell su -c "ss -xlp | grep wod_ipsec"
  Output:
    u_str LISTEN 0 0 @wod_ipsec 9877 * 0
    users:(("epdg_wod",pid=842,fd=6))

  CONFIRMED:
    wod_ipsec active and listening on live device
    @ prefix = abstract namespace — kernel memory only, not on filesystem
    epdg_wod owns socket at PID 842, file descriptor 6
    Running right now on live device

################################################################################
# PART 4 — libwo.so BINARY ANALYSIS — VERIFIED
################################################################################

## 4.1 File Identity

  Path:    /vendor/lib64/libwo.so
  Size:    54,912 bytes
  Date:    May 7, 2025 15:02

## 4.2 Ghidra Decompile Results

  wo_ccci_open:
    Opens /dev/ccci_woa with flags 0x80002 (O_RDWR | O_CLOEXEC)
    Uses ioctl(fd, 0x80044301, &status) — waits for modem ready (status=2)

  wo_ccci_write:
    Aggressive retry loop — exponential backoff
    After 5 failures: exit(2) — daemon restarts to maintain modem connection
    Checks is_ap_load_type() — suppresses buffer content in user builds
    Prints: buf:[%s] censored in production

  CRITICAL: Same is_ap_load_type() log censorship pattern appears in
  BOTH libcharon-ss.so AND libwo.so. Coordinated evasion across two
  separate binaries by the same ODM developer team.

  wo_sock_server_create:
    Binds Unix Domain Sockets (wod_ipsec, wod_action, wod_sim)
    Sets FD_CLOEXEC
    Fallback: uses ANDROID_SOCKET_%s environment variable from init

################################################################################
# PART 5 — VERIFIED END-TO-END CHAIN
################################################################################

Complete pipeline verified across four binaries + live device:

  libcharon-ss.so (IKEv2 daemon)
    Formats: ipsecattach, ipsecikedecryptadd, ipsecespdecryptadd
    Censors own logs: is_ap_load_type() check — data:[%s=***]
    Connects: socket_local_client("wod_ipsec")
    ↓
  @wod_ipsec (abstract namespace Unix socket)
    Confirmed live: epdg_wod PID 842 fd=6
    ↓
  epdg_wod (/vendor/bin/epdg_wod)
    Receives ipsec* command strings
    wo_ccci_open("/dev/ccci_woa", O_RDWR|O_CLOEXEC)
    ioctl wait for modem ready
    wo_ccci_write — aggressive persistence, exit(2) on failure
    Censors own logs: is_ap_load_type() check
    ↓
  /dev/ccci_woa (MediaTek CCCI kernel interface)
    Cross-Core Communication Interface — direct modem kernel channel
    ↓
  MT6762 baseband modem

Two findings that distinguish this from legitimate VoLTE behavior:
  1. Coordinated log censorship in TWO separate binaries
  2. exit(2) on write failure — persistence guarantee for the channel

################################################################################
# PART 6 — MtkCapCtrl ANALYSIS — VERIFIED
################################################################################

## 6.1 Architecture

  Service registered: ServiceManager.addService("capctrl", this)
  Accessible to: Any process that can reach ServiceManager
  AIDL interface: com.mediatek.capctrl.aidl.IMtkCapCtrl

## 6.2 enableCapabaility — Authentication NOT Enforced

  Implementation in MtkCapCtrlInterfaceManager.java:

    public int enableCapabaility(String str, int i) {
        AsyncResult asyncResult = (AsyncResult) sendRequest(4,
            new EnableCapabilityRequestInfo(str, Binder.getCallingUid(), i));
        ...
    }

  Caller UID recorded via Binder.getCallingUid() — but NOT validated.
  No check that routeAuthMessage() was called first.
  No permission check beyond enforceInterface() (descriptor only).
  Authentication flow (routeAuthMessage/routeCertificate) exists but
  is completely separate — enableCapabaility does not gate on it.

## 6.3 Terminal Routing — CapRIL.java

  enableCapability() routes directly to modem radio:

    mtkRadioExProxy.enableCapabaility(
        serial,
        featureName,   // capability string — no validation
        callerId,      // UID — logged only, not enforced
        toActive       // enable/disable
    );

  Via HIDL interface: vendor.mediatek.hardware.mtkradioex@1.0
  HIDL service names: mtkCap1, mtkCap2, mtkCap3, mtkCap4
  Destination: MediaTek Radio HAL → modem firmware

## 6.4 Complete Capability Control Chain

  Any process with ServiceManager access
    → ServiceManager.getService("capctrl")
    → enableCapabaility(featureName, toActive)
    → NO authentication enforcement
    → CapRIL.enableCapability()
    → IMtkRadioEx HIDL ("mtkCap1")
    → MediaTek Radio HAL
    → Modem firmware

## 6.5 Engineering Menu Context

  Four engineering menus confirmed on live device:
    1. MTK engineering mode (standard MediaTek)
    2. Wiite control panel — English (Keep/Prevent/Freeze functions)
    3. Chinese engineering menu — MCU1/MCU2 firmware operations
    4. Fourth menu (not yet characterized)

  Chinese engineering menu MCU distinctions:
    MCU1 = MT6762 main SoC
    MCU2 = nRF52832 sensor coprocessor

  Functions observed (Gemini live translation):
    Dump firmware (MCU1/MCU2 separately)
    Update firmware (MCU1/MCU2 separately)
    Heart rate and SpO2 diagnostic functions

  Assessment: MCU2 firmware dump via engineering menu = nRF52832 internal
  512KB flash dump. Currently the only path to nRF firmware binary.
  MCU2 firmware update = direct nRF52832 reflash capability.

################################################################################
# PART 7 — STOPWATCH JADX ANALYSIS
################################################################################

## 7.1 CompanyActivity.java — R-18 RESOLVED

  Status: Empty stub. Three override methods — all empty bodies.
  Loads layout activity_company, does nothing.
  R-18 in vulnerability report updated: NOT the payload. Empty placeholder.

## 7.2 MyService.java — R-06 CONFIRMED AT SOURCE LEVEL

  Timer: fires every 10ms (100 times/second) — always running
  MsgBinder extends Binder — onBind() returns with ZERO permission check
  exported=true — any app can bind and receive MsgBinder
  getService() returns direct reference to running UID 1000 instance

  Any third-party app that binds gets UID 1000 process reference.
  R-06 previously confirmed via manifest analysis — now confirmed at
  implementation level. Zero permission validation in binder implementation.

################################################################################
# PART 8 — COMPOSITE THREAT ARCHITECTURE
################################################################################

Complete attack surface available to any process with knowledge of interfaces.
All layers verified at binary level this session or prior sessions.

  Layer 1 — Android Framework:
    MyService (UID 1000, exported, no permission guard)
    ShellUtils arbitrary command execution
    Framework hooks: WiitePackageManagerUtil, Configuration.isSpecialApp()
    Backdoor registry: pms_sysapp_grant_permission_list.txt
    SELinux: PERMISSIVE — no mandatory access control

  Layer 2 — Capability Control:
    ServiceManager.getService("capctrl") — no auth enforcement
    enableCapabaility() → modem firmware via IMtkRadioEx HIDL
    Engineering mode functions accessible without authentication

  Layer 3 — IPSec/Modem Bridge:
    wod_ipsec abstract socket → epdg_wod → /dev/ccci_woa → modem
    IKE/ESP cryptographic material flows to modem baseband
    Coordinated log censorship in both chain components

  Layer 4 — nRF52832 Coprocessor:
    wiite_corp_ctrl/command — raw command channel (34 attributes, chmod 0666)
    wiite_corp_ctrl/upgrade_op — nRF firmware reflash capability
    wiite_corp_ctrl/nrf_sys_mode — operating mode control
    Engineering menu MCU2 functions map to these sysfs nodes

  Layer 5 — Persistence:
    Vendor partition — survives factory reset
    AbSharedUtil hardcoded unkillable process list
    WearCleanTaskPro protects spyware from OOM killer
    epdg_wod exit(2) on modem write failure — self-restoring
    FCM persistent push channel in AdupsFota — always-on

################################################################################
# PART 9 — DOCTRINE — VERIFIED / INFERRED / UNPROVEN
################################################################################

VERIFIED (binary evidence, live device, or source code):
  libcharon-ss.so is ODM-modified Strongswan — confirmed
  atcmd_txrx, notify_wod family — not in stock Strongswan — confirmed
  wod_ipsec socket chain to modem — confirmed end to end
  Coordinated log censorship in libcharon-ss.so and libwo.so — confirmed
  epdg_wod PID 842 owns wod_ipsec socket on live device — confirmed
  IKE/ESP decryption material formatted and sent to wod_ipsec — confirmed
  enableCapabaility() bypasses authentication enforcement — confirmed
  MtkCapCtrl routes directly to modem via HIDL — confirmed
  Engineering menus exist with MCU1/MCU2 firmware operations — confirmed
  CompanyActivity is empty stub — confirmed
  MyService privilege escalation confirmed at source level — confirmed

INFERRED (plausible, consistent with evidence, not yet binary-proven):
  modem acts on IKE/ESP key material for exfiltration purposes
  engineering menu invokes enableCapabaility with specific feature strings
  wod_ipsec channel used for unauthorized data routing beyond VoLTE

UNPROVEN (requires additional analysis):
  Active C2 infrastructure in operation
  Confirmed data exfiltration to remote servers
  Nation-state involvement
  nRF52832 firmware contains additional backdoor functionality
  What specific capability strings unlock engineering mode functions

################################################################################
# PART 10 — OPEN INVESTIGATION ITEMS (PRIORITY ORDER)
################################################################################

Priority 1 — nRF52832 firmware extraction via MCU2 engineering menu
  Dump nRF52832 internal 512KB flash — currently the only unknown firmware
  Method: Identify which APK launches Chinese engineering menu, trigger MCU2 dump

Priority 2 — Identify capability strings for engineering mode
  What feature name strings does enableCapabaility accept?
  Source: CapRIL logs at RIL_REQUEST_ENABLE_CAPABILITY level

Priority 3 — lib_remote_simlock.so Ghidra analysis
  File: /system/lib64/lib_remote_simlock.so (confirmed present live)
  Verizon enterprise SIM management code — ODM modification unknown

Priority 4 — com.tiamaes.watch.activation network analysis
  What does this agent transmit on first boot/registration?
  Method: tcpdump during first-boot sequence on clean flash

Priority 5 — WearAppFreeze/WearCleanTaskPro JADX
  Decompiled in projects/ — not yet analyzed this session
  WearAppFreeze_out/, WearCleanTaskPro_out/

Priority 6 — Fourth engineering menu characterization
  Three of four menus identified — fourth not yet characterized

################################################################################
# PART 11 — NEW FINDINGS ADDED TO RECORD TODAY
################################################################################

1. Board assembly date July 30, 2025 — NEWLY VERIFIED
2. LK build date June 13, 2025 — NEWLY VERIFIED
3. LK git commit hash a1685973d — NEWLY VERIFIED
4. MTK BSP branch alps-mp-q0.mp1-V9.48 — NEWLY VERIFIED
5. WP_C17S_PIX_TFT_D4 board name — NEWLY VERIFIED
6. libcharon-ss.so ODM modification confirmed — NEW FINDING
7. atcmd_txrx exported symbol in IPSec library — NEW FINDING
8. IMEI/IMSI encoding inside IKEv2 daemon — NEW FINDING
9. notify_wod constructs ipsecattach/ipsecdetach/ipseckeepalive — NEW FINDING
10. notify_wod_ikeesp externalizes IKE/ESP decryption material — NEW FINDING
11. FUN_0012cf00 confirmed as socket_local_client("wod_ipsec") — NEW FINDING
12. Deliberate log censorship in libcharon-ss.so (is_ap_load_type) — NEW FINDING
13. epdg_wod confirmed as wod_ipsec listener PID 842 live — NEW FINDING
14. /dev/ccci_woa confirmed as modem interface in chain — NEW FINDING
15. Deliberate log censorship in libwo.so (same pattern) — NEW FINDING
16. wo_ccci_write exit(2) on failure — persistence guarantee — NEW FINDING
17. MtkCapCtrl enableCapabaility — no auth enforcement — NEW FINDING
18. CapRIL routes to modem via IMtkRadioEx HIDL — NEW FINDING
19. Engineering menus: 4 confirmed, MCU1/MCU2 distinction found — NEW FINDING
20. CompanyActivity confirmed empty stub — R-18 RESOLVED
21. MyService privilege escalation confirmed at source level — R-06 UPGRADED

################################################################################
# END — SESSION_FINDINGS_2026_05_25_COMPLETE.md
# Status: ACTIVE INVESTIGATION
# All findings: forensically verified at binary or live device level
# Attribution: Albert Pittman (beright1976)
# Repo: https://github.com/Beright1976/wiite-odm-backdoor-research
################################################################################
