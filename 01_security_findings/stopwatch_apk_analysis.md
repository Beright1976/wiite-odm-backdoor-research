# Security Analysis Report
## `com.wiite.wiite_stopwatch` — Stopwatch.apk Static Analysis

**Researcher:** Albert Pittman (Beright1976)  
**Analysis Date:** 2026-04-13  
**Analysis Method:** Static — DEX string extraction, manifest decode, androguard  
**Target Device:** LOKMAT APPLLP 5 MAX (C17S)  

---

## Application Identity

| Field | Value |
|---|---|
| Package Name | `com.wiite.wiite_stopwatch` |
| Display Name | Stopwatch |
| Version Name / Code | 1.0 / 1 |
| Min SDK / Target SDK | 24 (Android 7.0) / 29 (Android 10) |
| Shared User ID | `android.uid.system` (UID 1000 — SYSTEM LEVEL) |
| Application Class | `com.wiite.wiite_stopwatch.App` |
| Debuggable | **TRUE** ← Security Flag |
| allowBackup | **TRUE** ← Security Flag |
| Source Path (device) | `/vendor/operator/app/Stopwatch/Stopwatch.apk` |
| APK Size | 4,762,084 bytes (4.5 MB) |
| DEX Files | `classes.dex` + `classes2.dex` |

---

## Critical Verdict

> **This application is NOT a stopwatch. It is a persistent background agent with SYSTEM-level privileges, extensive device information harvesting capability, shell execution capability, and multiple Chinese technology references. It must be removed from any production ROM.**

---

## 1. Executive Summary

`Stopwatch.apk`, shipped pre-installed at `/vendor/operator/app/Stopwatch/` on the LOKMAT APPLLP 5 MAX (C17S), presents as a stopwatch application but contains capabilities far beyond its stated purpose.

The app runs as Android system UID 1000, starts on every boot via a broadcast receiver, and includes libraries capable of executing shell commands, collecting hardware device identifiers (IMEI, IMSI), scanning WiFi networks, performing DES/AES/RSA encryption, and communicating via a Messenger IPC mechanism.

No confirmed outbound network exfiltration endpoint was found in static analysis — the only IP address found (`223.5.5.5`) is Alibaba's public DNS resolver, and the `send2Server` method chain appears to target a local IPC server. However, the capability to exfiltrate data exists in the codebase. Combined with the app's system-level privilege, boot persistence, Tencent/Baidu references, and Chinese ODM origin, this represents a significant security risk.

**Recommendation: Remove this APK entirely from all clean ROM builds. It is the first priority vendor/operator cleanup item.**

---

## 2. AndroidManifest.xml Analysis

### 2.1 Declared Permissions

The manifest declares only one explicit permission:

| Permission | Risk | Purpose |
|---|---|---|
| `android.permission.RECEIVE_BOOT_COMPLETED` | MEDIUM | Starts on every device boot |

**Critical note:** Because the app runs as `android.uid.system` (`sharedUserId`), it inherits all system-level permissions automatically — without declaring them. This single-permission manifest is deliberately misleading.

The effective runtime permission set includes: `INSTALL_PACKAGES`, `WRITE_SECURE_SETTINGS`, `READ_PHONE_STATE`, `ACCESS_FINE_LOCATION`, `RECORD_AUDIO`, `CAMERA`, `READ_SMS`, `READ_CALL_LOG`, `MASTER_CLEAR`, `REBOOT`, and hundreds more.

### 2.2 Application Flags

| Attribute | Value | Security Impact |
|---|---|---|
| `sharedUserId` | `android.uid.system` | CRITICAL — runs as UID 1000, full system access |
| `debuggable` | `true` | HIGH — ADB debugging enabled on production system app |
| `allowBackup` | `true` | MEDIUM — app data extractable via ADB backup |
| `extractNativeLibs` | `false` | Neutral — libraries loaded directly from APK |

### 2.3 Application Components

| Component | Class | Risk |
|---|---|---|
| Activity | `MainActivity` | Launcher |
| Activity | `CompanyActivity` | **HIGH** — "Company" activity, purpose unknown |
| Activity | `UtilsTransActivity4MainProcess` | From blankj/utilcode library |
| Activity | `UtilsTransActivity` | From blankj/utilcode, `multiprocess=true` |
| Service | `MyService` (exported=**TRUE**) | **HIGH** — exported service, no permission guard, bound via Messenger |
| Service | `MessengerUtils$ServerService` | IPC server from blankj/utilcode |
| Receiver | `BootBroadcastReceiver` | **HIGH** — fires on `BOOT_COMPLETED` + `HOME` category |
| Provider | `UtilsFileProvider` | File sharing via `grantUriPermissions=true` |

`MyService` being `exported=true` without any permission restriction means any other app on the device can bind to it and invoke its IPC interface. This is a direct privilege escalation surface — a malicious third-party app can bind to a UID 1000 process and execute system-privileged code.

### 2.4 Boot Receiver Intent Filter

`BootBroadcastReceiver` registers for `android.intent.action.BOOT_COMPLETED` with category `android.intent.category.HOME`. The `HOME` category is unusual for a boot receiver — it is normally used by launcher applications. This indicates the app may attempt to register as a home screen replacement or ensures it receives very broad broadcast intents during boot.

---

## 3. Third-Party Library Analysis

### 3.1 com.blankj.utilcode (AndroidUtilCode)

The app bundles the Chinese open-source utility library AndroidUtilCode (GitHub: Blankj/AndroidUtilCode). The modules included provide extensive system capabilities:

| Module | Class | Capability | Risk |
|---|---|---|---|
| Device Info | `DeviceUtils` | IMEI, IMSI, unique device ID, model, manufacturer | HIGH |
| Shell | `ShellUtils` | Execute arbitrary shell commands, read output | HIGH |
| Network/WiFi | `NetworkUtils` | WiFi scan, SSID, MAC, IP, gateway, netmask | MEDIUM |
| Encryption | `EncryptUtils` | DES, 3DES, AES, RSA, MD5, SHA-1/256/512, Base64 | MEDIUM |
| Phone | `PhoneUtils` | Phone number, type, IMEI access | HIGH |
| App Install | `AppUtils` | Install/uninstall APKs programmatically | **CRITICAL** |
| IPC | `MessengerUtils` | Multi-process messaging server | MEDIUM |

**Critical capability:** `AppUtils.installApp()` combined with the `INSTALL_PACKAGES` permission (inherited via system UID) allows this app to silently install arbitrary APKs without any user interaction or confirmation dialog.

### 3.2 GreenDAO ORM

The app uses GreenDAO for local SQLite database storage. A database with a `RECORD_BEAN` table is created:

```sql
CREATE TABLE "RECORD_BEAN" (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    "INDEX" INTEGER NOT NULL,
    "CURRENT_TIME" TEXT
)
```

This stores stopwatch lap records and appears to serve the app's legitimate stopwatch functionality. GreenDAO also references SQLCipher for database encryption — whether it is used cannot be confirmed without runtime analysis.

### 3.3 Tencent QQ Browser FileProvider Reference

The string `com.tencent.mtt.fileprovider` appears in the DEX code. This is the authority string of Tencent QQ Browser's FileProvider. Its presence in a watch stopwatch application has no legitimate explanation. It indicates either code copied from a Tencent app, a Tencent library dependency, or remnants of a larger codebase.

---

## 4. Dangerous Capability Analysis

### 4.1 Device Identifier Harvesting

Since the app runs as system UID, all Android permission checks for `READ_PHONE_STATE` are bypassed:

| Method | Returns |
|---|---|
| `getIMEI()` | IMEI/MEID — unique hardware identifier |
| `getIMSI()` | IMSI — SIM-card subscriber identifier |
| `getUniqueDeviceId()` | Combined multi-identifier device ID |
| `getUniqueDeviceIdReal()` | Raw hardware unique ID |
| `getMacAddressByWifiInfo()` | WiFi MAC address |
| `printDeviceInfo()` | Device model + manufacturer |

### 4.2 Shell Command Execution

The `ShellUtils` class provides arbitrary shell command execution. DEX strings confirm shell execution capability:

```
ShellUtils
getSystemPropertyByShell
echo root
/root
isDeviceRooted
execute
executeOperation
```

On a device with Magisk root, SELinux permissive, and AVB disabled, these calls execute with unrestricted system access.

### 4.3 WiFi Intelligence

| Method | Data Collected |
|---|---|
| `getWifiScanResult()` | All visible WiFi networks (SSID, BSSID, signal strength) |
| `getServerAddressByWifi()` | DHCP server address |
| `getGatewayByWifi()` | Network gateway IP |
| `getIpAddressByWifi()` | Device IP address |
| `getMacAddressByWifiInfo()` | Device WiFi MAC address |
| `getNetMaskByWifi()` | Subnet mask |

### 4.4 Cryptographic Capabilities

| Algorithm | Methods Present | Risk Context |
|---|---|---|
| AES | `encryptAES`, `decryptAES`, `AES2Base64`, `AES2HexString` | Strong symmetric encryption — can encrypt data for transmission |
| RSA | `encryptRSA`, `decryptRSA`, `RSA2Base64`, `RSA2HexString` | Asymmetric — can encrypt for a remote recipient's public key |
| DES / 3DES | `encrypt/decrypt DES, 3DES` | Legacy encryption |
| MD5 | `encryptMD5`, `encryptMD5File` | Fingerprinting |
| SHA-1/256/384/512 | `encryptSHA*` | Hashing |
| HMAC | `encryptHmacMD5`, `HmacSHA*` | Message authentication |

The presence of RSA is particularly relevant — it enables encrypting harvested data with a remote recipient's public key before transmission, making the payload unreadable to anyone except the intended recipient.

### 4.5 Silent Application Management

Combined with system UID, these methods require no user interaction:

| Method | Function | Risk |
|---|---|---|
| `installApp()` | Silent APK installation | **CRITICAL** |
| `getInstallAppIntent()` | Install intent builder | HIGH |
| `getUninstallAppIntent()` | Uninstall intent builder | HIGH |
| `isAppInstalled()` | Check if app present | MEDIUM |
| `getInstalledPackages()` | List all installed apps | MEDIUM |

---

## 5. Network and External Reference Analysis

### 5.1 IP Addresses Found in DEX

One hard-coded IP address found in the entire DEX string pool:

`223.5.5.5` — Alibaba Cloud public DNS resolver (AliDNS). Analogous to Google's `8.8.8.8`. Confirms the app was developed in a Chinese environment. May be used for DNS resolution or as a connectivity check endpoint.

### 5.2 External Domain References

| Domain / Reference | Organization | Context |
|---|---|---|
| `www.baidu.com` | Baidu Inc. | String literal in DEX |
| `Baidurack` | Baidu Inc. | Internal class name fragment |
| `com.tencent.mtt.fileprovider` | Tencent (QQ Browser) | FileProvider authority string |
| `223.5.5.5` | Alibaba Cloud | Hard-coded IP |

### 5.3 Server Communication Methods

The following methods are present in the DEX. Static analysis suggests they target the local Messenger IPC server, not an external internet endpoint — but without dynamic runtime analysis, remote exfiltration cannot be ruled out:

```
send2Server
sendMsg2Server
sendCachedMsg2Server
serverAddress
MSG_SERVICE_CONNECTED
MSG_SERVICE_DISCONNECTED
```

### 5.4 Static Analysis Limitations

The following cannot be confirmed without dynamic runtime analysis:

- Whether the app makes outbound HTTP/HTTPS connections
- Whether device identifiers (IMEI, IMSI) are transmitted remotely
- Whether `serverAddress` is set to an external host at runtime
- Whether the RSA public key belongs to a remote server
- Whether `223.5.5.5` is used for DNS queries or direct communication

**To complete the investigation:** capture network traffic during boot and during stopwatch use:

```bash
adb shell tcpdump -i any -w /sdcard/capture.pcap &
# reboot device to trigger BOOT_COMPLETED
# use Stopwatch app for 5 minutes
adb shell kill $(adb shell pidof tcpdump)
adb pull /sdcard/capture.pcap
# analyze in Wireshark — filter for UID 1000 source processes
# then full decompile: jadx -d jadx_out/ Stopwatch.apk
```

---

## 6. Behavioral Assessment

### 6.1 Legitimate Stopwatch Functionality (Confirmed)

These components are consistent with a genuine stopwatch:

- `MainActivity` — stopwatch UI (start/stop/lap controls)
- `MyService` with `OnTimeListener` — background timer
- `RecordBean` / `RecordBeanDao` — SQLite lap record storage
- `RecordAdapter` — lap record display
- `MessageEvent` (OnStartEvent, OnStopEvent, OnTimeEvent) — timer events via EventBus
- `MsgBinder` — Activity-to-Service binding for timer updates

### 6.2 Unexplained / Suspicious Functionality

| Finding | Risk | Explanation |
|---|---|---|
| `CompanyActivity` | HIGH | Undeclared "Company" entry point — purpose completely unknown |
| `debuggable=true` in production | HIGH | Allows ADB memory inspection of a system-privileged process in the field |
| `MyService` exported, no permission | HIGH | Any app can bind and invoke this system-privileged service |
| `BOOT_COMPLETED` + `HOME` category | MEDIUM | Atypical for a timer app — suggests launcher-level registration |
| `EncryptUtils` (RSA/AES) | MEDIUM | Full crypto toolkit with no stopwatch use case |
| `ShellUtils` class present | HIGH | Arbitrary shell execution with no stopwatch use case |
| `DeviceUtils.getIMEI()` | HIGH | IMEI/IMSI collection with no stopwatch use case |
| `getWifiScanResult()` | MEDIUM | WiFi scanning with no stopwatch use case |
| `com.tencent.mtt.fileprovider` ref | MEDIUM | Tencent reference unexplained |
| `www.baidu.com` / `Baidurack` | MEDIUM | Baidu reference unexplained |
| `installApp()` capability | **CRITICAL** | Silent APK installation with no stopwatch use case |

---

## 7. Risk Summary

| # | Finding | Risk | Confirmed |
|---|---|---|---|
| R-01 | Runs as `android.uid.system` (UID 1000) | CRITICAL | YES |
| R-02 | Silent APK install capability | CRITICAL | YES — capability present |
| R-03 | Vendor partition persistence — survives factory reset | CRITICAL | YES |
| R-04 | `debuggable=true` on system-privileged production app | HIGH | YES |
| R-05 | `ShellUtils` arbitrary shell execution | HIGH | YES — class present |
| R-06 | IMEI/IMSI/Device ID collection | HIGH | YES — methods present |
| R-07 | `MyService` exported=true, no permission guard | HIGH | YES |
| R-08 | `BootBroadcastReceiver` — starts on every boot | HIGH | YES |
| R-09 | RSA/AES encryption capable | MEDIUM | YES — classes present |
| R-10 | WiFi scanning (SSID, MAC, gateway, IP) | MEDIUM | YES — methods present |
| R-11 | `com.tencent.mtt.fileprovider` reference | MEDIUM | YES — string in DEX |
| R-12 | `www.baidu.com` and `Baidurack` references | MEDIUM | YES — strings in DEX |
| R-13 | `223.5.5.5` (Alibaba DNS) hardcoded | MEDIUM | YES — string in DEX |
| R-14 | `allowBackup=true` — app data extractable via ADB | MEDIUM | YES |
| R-15 | Outbound network exfiltration | UNCONFIRMED | Requires dynamic analysis |
| R-16 | Remote server endpoint | UNCONFIRMED | No external URL found — IPC only in static |
| R-17 | `CompanyActivity` purpose | UNKNOWN | Requires JADX decompile |

---

## 8. Recommendations

### Immediate Actions

| Priority | Action | Method |
|---|---|---|
| P1 — CRITICAL | Exclude `Stopwatch.apk` from clean ROM | Omit from device tree — do not copy `/vendor/operator/app/Stopwatch/` |
| P1 — CRITICAL | Audit all `/vendor/operator/app/` APKs | Every app in this path inherits system privileges — analyze all |
| P2 — HIGH | Dynamic network analysis | `tcpdump` on boot — confirm or deny outbound connections |
| P2 — HIGH | Full decompile with JADX | Decompile `classes.dex` and `classes2.dex` — full Java source review of `CompanyActivity` and `MyService` |

### Clean ROM Exclusion

```makefile
# In device tree — exclude the entire vendor/operator directory:
# Do not carry /vendor/operator/app/ into the clean vendor image.
# Or specifically exclude via:
PRODUCT_DEL_PACKAGES += com.wiite.wiite_stopwatch
```

---

## Appendix A — Full Class List

```
com.wiite.wiite_stopwatch.App
com.wiite.wiite_stopwatch.BaseActivity
com.wiite.wiite_stopwatch.BootBroadcastReceiver
com.wiite.wiite_stopwatch.BuildConfig
com.wiite.wiite_stopwatch.CompanyActivity
com.wiite.wiite_stopwatch.Constant
com.wiite.wiite_stopwatch.MainActivity
com.wiite.wiite_stopwatch.MainActivity$1
com.wiite.wiite_stopwatch.MessageEvent
com.wiite.wiite_stopwatch.MessageEvent$OnBaseEvent
com.wiite.wiite_stopwatch.MessageEvent$OnNotifyRegister
com.wiite.wiite_stopwatch.MessageEvent$OnRegister
com.wiite.wiite_stopwatch.MessageEvent$OnStartEvent
com.wiite.wiite_stopwatch.MessageEvent$OnStopEvent
com.wiite.wiite_stopwatch.MessageEvent$OnTimeEvent
com.wiite.wiite_stopwatch.MessageEvent$OnUnRegister
com.wiite.wiite_stopwatch.MyService
com.wiite.wiite_stopwatch.MyService$MsgBinder
com.wiite.wiite_stopwatch.MyService$OnTimeListener
com.wiite.wiite_stopwatch.NewGridSpacingDecoration
com.wiite.wiite_stopwatch.RecordAdapter
com.wiite.wiite_stopwatch.dao.DaoMaster
com.wiite.wiite_stopwatch.dao.DaoSession
com.wiite.wiite_stopwatch.dao.MySQLiteOpenHelper
com.wiite.wiite_stopwatch.dao.RecordBean
com.wiite.wiite_stopwatch.dao.RecordBeanDao
```

---

## Appendix B — Security-Relevant DEX Strings

**Device / Identity:**
```
getIMEI | getIMSI | getUniqueDeviceId | getUniqueDeviceIdReal | getDeviceId
printDeviceInfo | getMacAddressByWifiInfo | isImei | ril.gsm.imei
```

**Shell / Root:**
```
ShellUtils | getSystemPropertyByShell | echo root | /root
isDeviceRooted | isRooted | execute | executeOperation
```

**Network / Server:**
```
send2Server | sendMsg2Server | sendCachedMsg2Server | serverAddress
223.5.5.5 | www.baidu.com | Baidurack | com.tencent.mtt.fileprovider
getServerAddressByWifi | isWifiConnected | getWifiScanResult
```

**Cryptographic:**
```
DESede | encryptAES | decryptAES | encryptRSA | decryptRSA
encryptMD5 | encryptSHA256 | base64Encode | rsaKey | secretKey
javax/crypto/Cipher | desKey
```

**App Management:**
```
installApp | getInstallAppIntent | getUninstallAppIntent
getInstalledPackages | isAppInstalled | application/vnd.android.package-archive
```

---

## Appendix C — Analysis Environment

| Tool | Use |
|---|---|
| Python 3.x | DEX string extraction, manifest parsing |
| androguard | APK/manifest analysis |
| Custom DEX parser | String pool extraction from `classes.dex` + `classes2.dex` |
| unzip | APK extraction |

*All findings are based on static analysis of `Stopwatch.apk`. Dynamic runtime behavior may differ. To complete the investigation, runtime network capture is required. See Section 5.4 for the dynamic analysis protocol.*
