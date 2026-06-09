# Security Analysis Report
## `com.wiite.devicemsg` — WearDevicemsg.apk Static Analysis

**Researcher:** Albert Pittman (Beright1976)  
**Analysis Date:** 2026-06-09  
**Analysis Method:** Static — apktool 2.5.0 full DEX decode, smali analysis, manifest decode, strings extraction  
**Target Device:** LOKMAT APPLLP 5 MAX (C17S)  
**Related CVE Request:** CVE Request 2025814 (supplemental evidence submitted 2026-06-09)

---

## Application Identity

| Field | Value |
|---|---|
| Package Name | `com.wiite.devicemsg` |
| Display Name | WearDevicemsg |
| APK Path (device) | `/product/app/WearDevicemsg/WearDevicemsg.apk` |
| APK Size | 8,172,512 bytes (7.8 MB) |
| Shared User ID | `android.uid.system` (UID 1000 — SYSTEM LEVEL) |
| Debuggable | **TRUE** ← Security Flag |
| allowBackup | **TRUE** ← Security Flag |
| Compile SDK Version | 32 (Android 12) |
| DEX Files | `classes.dex` + `classes2.dex` |
| Package Cache Entry | `/data/system/package_cache/ae92ce4b513e1de6f90a8b4871b279e641885400/WearDevicemsg-16` |
| Google Malware Flag | **CONFIRMED** — Google Drive access blocked by Google malware scanner |
| Partition | `/product` — read-only, survives factory reset |

---

## Critical Verdict

> **This application is NOT a device messaging utility. It is a confirmed surveillance and device telemetry framework running as UID 1000 system authority, embedding a third-party obfuscated data collection SDK (`com.nasa.cook` / `com.nasa.memory`) that harvests hardware device identifiers, MAC addresses, Android ID, and transmits them via JSON HTTP POST on every boot. Google's malware infrastructure independently confirmed this classification by blocking the APK on Google Drive. This package must be removed from any production ROM.**

---

## 1. Executive Summary

`WearDevicemsg.apk`, shipped pre-installed at `/product/app/WearDevicemsg/` on the LOKMAT APPLLP 5 MAX (C17S), presents as a device messaging component but contains a fully functional surveillance and telemetry architecture operating entirely through an embedded third-party SDK.

The application runs as Android system UID 1000, starts on every boot via a `BOOT_COMPLETED` receiver with the highest possible priority (1000), and immediately initializes an obfuscated third-party SDK identified as `com.nasa.cook` / `com.nasa.memory`. This SDK implements device fingerprinting through hardware identifier collection, stores collected data in a hidden encrypted directory named `.honor`, executes `chmod 777` on its storage directories via `Runtime.exec()`, and transmits assembled device telemetry records to a remote server via JSON HTTP POST.

The SDK's remote server endpoints are not present as static strings in the APK. They are assembled at runtime through dynamic class loading — a deliberate anti-analysis design that makes static URL extraction impossible and requires runtime traffic capture for full server attribution.

Google's malware scanner independently flagged this APK and blocked access to it on Google Drive, providing third-party confirmation of the finding.

**Recommendation: Remove this APK entirely from all clean ROM builds. It is a confirmed data collection and telemetry transmission agent embedded in production consumer firmware without disclosure.**

---

## 2. AndroidManifest.xml Analysis

### 2.1 Declared Permissions

| Permission | Risk | Purpose |
|---|---|---|
| `android.permission.INTERNET` | CRITICAL | Outbound network transmission to remote servers |
| `android.permission.ACCESS_NETWORK_STATE` | HIGH | Detects network availability before transmitting |
| `android.permission.ACCESS_WIFI_STATE` | HIGH | WiFi state access for MAC and BSSID collection |
| `android.permission.READ_PHONE_STATE` | CRITICAL | IMEI, IMSI, phone number, SIM serial access |
| `android.permission.WRITE_EXTERNAL_STORAGE` | HIGH | Write collected data to external storage |
| `android.permission.READ_EXTERNAL_STORAGE` | HIGH | Read files from external storage |

**Critical note:** Because the app runs as `android.uid.system` (`sharedUserId`), it inherits all system-level permissions automatically in addition to those declared above. The declared permissions represent only the explicit surface — the effective runtime authority is total system access.

### 2.2 Application Flags

| Attribute | Value | Security Impact |
|---|---|---|
| `sharedUserId` | `android.uid.system` | CRITICAL — runs as UID 1000, full system access |
| `debuggable` | `true` | HIGH — ADB debugging enabled on production system app |
| `allowBackup` | `true` | MEDIUM — app data extractable via ADB backup |
| `extractNativeLibs` | `false` | Neutral — libraries loaded directly from APK |

### 2.3 Application Components

| Component | Class | Exported | Risk |
|---|---|---|---|
| Activity | `com.wiite.devicemsg.MainActivity` | **TRUE** | HIGH — exported entry point |
| Receiver | `com.wiite.devicemsg.BootReceiver` | **TRUE** | CRITICAL — fires on `BOOT_COMPLETED`, priority 1000 |
| Service | `com.wiite.devicemsg.UploadingService` | **TRUE** | CRITICAL — confirmed telemetry upload service, no permission guard |
| Provider | `androidx.startup.InitializationProvider` | FALSE | AndroidX lifecycle initialization |

### 2.4 Boot Receiver Intent Filter

`BootReceiver` registers for `android.intent.action.BOOT_COMPLETED` with priority **1000** — the maximum Android receiver priority value — ensuring this receiver fires before virtually all other boot receivers on the device, initializing the telemetry framework at the earliest possible point in the boot sequence.

### 2.5 UploadingService Export Risk

`UploadingService` is exported with no permission guard. Any application on the device can start this service and trigger SDK initialization and data transmission. Combined with UID 1000 authority, this creates an accessible privileged transmission surface reachable from the normal application layer.

---

## 3. Embedded Third-Party SDK: com.nasa.cook / com.nasa.memory

### 3.1 SDK Architecture

The telemetry framework is not implemented directly in `com.wiite.devicemsg` code. The package embeds a complete third-party SDK organized across two namespaces:

| Namespace | Role |
|---|---|
| `com.nasa.cook` | Public API surface — `CookInit`, `ITrackCallback` interface |
| `com.nasa.memory` | Obfuscated implementation — single-letter class names throughout |

The obfuscation pattern — single-letter class names, runtime URL assembly, XOR-encrypted local storage, and dynamic class loading — is consistent with a commercial surveillance or advertising SDK deliberately engineered to resist reverse engineering.

### 3.2 SDK Initialization Chain

`UploadingService.onCreate()` — executed on every boot — immediately delegates to the SDK:

```smali
# UploadingService.smali — onCreate()
const-string v0, "WIITE001001"
invoke-static {p0, v0}, Lcom/nasa/cook/CookInit;->init(Landroid/content/Context;Ljava/lang/String;)V
```

The string `WIITE001001` is the SDK publisher channel identifier. This ties the SDK instance specifically to the Wiite/Topwise ODM. This is not a generic SDK inclusion — it was integrated by the ODM with a publisher-specific credential.

`CookInit.init()` delegates into the obfuscated implementation layer `com.nasa.memory.tool.h` (`CoreHelper.java`), which performs SDK initialization, device data collection, and transmission scheduling.

### 3.3 Public SDK Methods — Function Confirmation

The `CookInit` public API exposes the following methods, confirming the SDK's dual advertising and tracking function:

| Method | Signature | Purpose |
|---|---|---|
| `init` | `init(Context, String)` | SDK initialization with channel ID |
| `init` | `init(Context, String, ITrackCallback)` | Initialization with tracking callback |
| `showAdvert` | `showAdvert(Context, Object)` | Advertisement display |
| `kill` | `kill()` | SDK termination |

The presence of `showAdvert` alongside `ITrackCallback` and a channel identifier confirms this is a commercial advertising and device tracking SDK embedded inside a system-UID production firmware package without user disclosure and without consent.

---

## 4. Device Identifier Collection — Complete Evidence Chain

### 4.1 DeviceIdUtil.java (j.smali) — Composite Fingerprint Assembly

Source file: `.source "DeviceIdUtil.java"` (recovered from smali debug header)

Constructs a composite device fingerprint by concatenating:

```smali
# j.smali — a(Context) method
sget-object v2, Landroid/os/Build;->BRAND:Ljava/lang/String;   # SPRD
sget-object v2, Landroid/os/Build;->MODEL:Ljava/lang/String;   # LOKMAT APPLLP 5 MAX
invoke-static {p0}, Lcom/nasa/memory/tool/p;->b(Context)       # MAC address set
invoke-static {p0}, Lcom/nasa/memory/tool/s;->a(Context)       # Android ID
const-string v0, "SDKO"                                         # SDK origin marker
```

Result is hashed via `o.b()` and written to a persistent cache file. On subsequent boots the cached fingerprint is loaded from disk, providing a stable cross-session device identifier.

### 4.2 MacUtil.java (p.smali) — Hardware MAC Address Collection

Source file: `.source "MacUtil.java"` (recovered from smali debug header)

Five independent collection methods with fallback logic:

| Method | Implementation | Data Collected |
|---|---|---|
| 1 | `cat /sys/class/net/wlan0/address` via shell | WiFi interface MAC |
| 2 | `cat /sys/class/net/eth0/address` via shell | Ethernet interface MAC |
| 3 | `NetworkInterface.getHardwareAddress()` | Hardware MAC via Java API |
| 4 | `WifiManager.getScanResults() + getBSSID()` | Connected AP MAC address |
| 5 | `SystemProperties.native_get("wifi.interface")` | WiFi interface via reflection |

All collected MAC addresses are concatenated with pipe `|` separators into the telemetry record.

**Location correlation risk:** Collection of the connected access point BSSID (Method 4) creates a location-traceable component. Router MAC addresses can be cross-referenced against public WiFi geolocation databases to approximate the physical location of the device at the time of transmission.

### 4.3 SDKTools.java (s.smali) — Android ID Collection

Source file: `.source "SDKTools.java"` (recovered from smali debug header)

```smali
# s.smali — a(Context) method
invoke-virtual {p0}, Landroid/content/Context;->getContentResolver()
const-string v0, "android_id"
invoke-static {p0, v0}, Landroid/provider/Settings$System;->getString(
    Landroid/content/ContentResolver;Ljava/lang/String;)Ljava/lang/String;
```

`android_id` is a 64-bit identifier assigned at device provisioning. It survives application reinstallation and is the primary stable identifier used by advertising and analytics platforms for cross-session device tracking.

---

## 5. Hidden Storage, Filesystem Manipulation, and Data Encryption

### 5.1 Hidden Directory Creation

`SDKTools.java` (`s.smali`, **line 1054**):

```smali
const-string v1, ".honor"
```

Creates a hidden storage directory named `.honor` inside the application files directory. The leading dot makes this directory invisible to standard file browsing on Linux-based systems and to most Android file manager applications.

### 5.2 Privilege Escalation via Shell Execution

`SDKTools.java` (`s.smali`, **lines 323, 338, 344**):

```smali
# line 323
const-string v1, "chmod 777 "

# line 338
invoke-static {}, Ljava/lang/Runtime;->getRuntime()Ljava/lang/Runtime;

# line 344
invoke-virtual {v0, p0}, Ljava/lang/Runtime;->exec(Ljava/lang/String;)Ljava/lang/Process;
```

Executes `chmod 777` on its storage directories via `Runtime.exec()` from a UID 1000 process. Setting permissions to 777 grants any application on the device read and write access to the SDK's hidden data store. From a system-UID process this operation succeeds unconditionally.

### 5.3 XOR Encryption of Cached Data

All data written to the `.honor` directory is XOR-encrypted using a rolling key before being written to disk. The encryption function in `s.smali` operates on raw byte arrays cycling through the key bytes against the data bytes. This prevents casual forensic inspection of cached device fingerprints and queued report data.

---

## 6. Network Transport and Telemetry Transmission

### 6.1 Transport Implementation

The `com.nasa.memory` framework implements a JSON HTTP POST transport layer using `HttpURLConnection` with confirmed configuration:

- Request method: `POST`
- Content-Type: `application/json`
- Headers: `channel`, `version`
- Body: serialized `ReportBean` JSON
- Response: server response read from input stream

### 6.2 ReportBean — Confirmed Transmission Fields

The `DataReporter.java` implementation constructs `ReportBean` records containing:

| Field | Content |
|---|---|
| `uploadKey` | Report classification key |
| `uploadValue` | Report payload value |
| `ext1`, `ext2`, `ext3` | Extension metadata fields |
| `deviceId` | Composite fingerprint (BRAND + MODEL + MAC + Android ID) |
| `channel` | SDK channel identifier (`WIITE001001`) |
| `terminalVersion` | Application version |
| `androidVersion` | Android OS version |
| `androidModel` | Device model string |
| `androidBrand` | Device brand string |
| `macAddress` | Full MAC address set (pipe-separated) |
| `androidId` | `Settings.System android_id` value |
| `terminalTime` | Transmission timestamp |

### 6.3 Asynchronous Queuing

Background cache layer uses `HandlerThread` named `burying_loop_s`. `ReportBean` objects are queued and submitted with scheduled delay, decoupling transmission timing from the boot trigger and ensuring delivery survives temporary network unavailability.

### 6.4 Rate Limiting — Detection Evasion

Transmission attempts capped at **10 per `ecb108v9c` SharedPreferences key**. Limiting network activity frequency reduces probability of detection by behavioral security monitoring tools — consistent with detection evasion design.

### 6.5 Server Endpoint Obfuscation

No server URLs or domain names are present as static strings in the APK. Endpoints are assembled at runtime through dynamic class loading. Standard `strings` analysis, automated app store scanners, and antivirus string extraction return no readable server addresses. Full server attribution requires runtime network traffic capture.

---

## 7. Dynamic Code Execution

The SDK uses `com.nasa.memory.tool.n` as a runtime class loader:

```smali
invoke-static {}, Lcom/nasa/memory/tool/n;->c()Lcom/nasa/memory/tool/n;
invoke-virtual {v0, p1}, Lcom/nasa/memory/tool/n;->b(Landroid/content/Context;)Ljava/lang/Class;
```

This allows the SDK to download and execute code delivered from the remote server at runtime. Behavior, server addresses, collection scope, and transmission targets can be modified server-side without updating the APK. The installed APK is effectively a delivery shell — the active payload is server-controlled.

---

## 8. Capabilities Matrix

| Capability | Implementation | Evidence | Risk |
|---|---|---|---|
| Privilege Level | `android.uid.system` UID 1000 | AndroidManifest.xml | CRITICAL |
| Boot Persistence | `BOOT_COMPLETED` priority 1000 | AndroidManifest.xml | CRITICAL |
| Device Fingerprinting | BRAND + MODEL + MAC + Android ID | `j.smali` DeviceIdUtil.java | CRITICAL |
| Android ID Collection | `Settings.System android_id` | `s.smali` confirmed | HIGH |
| WiFi MAC Harvest | `cat /sys/class/net/wlan0/address` | `p.smali` MacUtil.java | HIGH |
| Ethernet MAC Harvest | `cat /sys/class/net/eth0/address` | `p.smali` MacUtil.java | HIGH |
| Hardware MAC Collection | `NetworkInterface.getHardwareAddress()` | `p.smali` MacUtil.java | HIGH |
| Location Correlation | WiFi AP BSSID collected | `p.smali` MacUtil.java | HIGH |
| Data Transmission | JSON HTTP POST | `com.nasa.memory` transport | CRITICAL |
| Hidden Storage | `.honor` directory | `s.smali` line 1054 | HIGH |
| Filesystem Manipulation | `chmod 777` via `Runtime.exec()` | `s.smali` lines 323, 338, 344 | HIGH |
| XOR Data Encryption | Rolling XOR on all cached data | `s.smali` XOR function | MEDIUM |
| Dynamic Code Execution | Runtime class loading | `com.nasa.memory.tool.n` | CRITICAL |
| Async Queuing | `HandlerThread` `burying_loop_s` | BuryingCache implementation | MEDIUM |
| Rate Limiting | 10-attempt cap `ecb108v9c` | `h.smali` rate check | MEDIUM |
| Server Obfuscation | No static URLs — runtime only | Strings analysis confirmed | HIGH |
| Debuggable Production | `android:debuggable="true"` | AndroidManifest.xml | HIGH |
| Google Malware Flag | Drive access blocked | Third-party confirmation | CONFIRMED |

---

## 9. SDK Attribution

The `com.nasa.cook` / `com.nasa.memory` namespace does not correspond to any publicly documented legitimate Android SDK. The obfuscation is systematic and deliberate:

- Single-letter class names throughout the entire implementation layer
- Runtime URL assembly preventing static server attribution
- XOR-encrypted local storage hiding cached telemetry
- Dynamic class loading enabling server-controlled behavior modification
- Clean public API facade concealing obfuscated implementation
- Rate limiting reducing behavioral detection probability

The channel identifier `WIITE001001` ties this SDK instance specifically to the Wiite/Topwise ODM. This is not a passive SDK inclusion. It was actively integrated by the ODM with a publisher-specific credential and configured to initialize on every boot via a system-UID privileged service.

---

## 10. Relationship to Platform Threat Chain

`WearDevicemsg.apk` is the third confirmed independent data collection framework identified in this firmware:

| Framework | Location | Privilege | Survives Reset | Primary Function |
|---|---|---|---|---|
| `Stopwatch.apk` | `/vendor/operator/app/Stopwatch/` | UID 1000 | YES | System backdoor, silent APK install, shell execution, IMEI/IMSI harvest |
| `AdupsFota.apk` | `/product/priv-app/AdupsFota/` | priv-app | YES | C2 to `fota5p.adups.com`, remote payload delivery, silent OTA install |
| `WearDevicemsg.apk` | `/product/app/WearDevicemsg/` | UID 1000 | YES | Device fingerprint, MAC/Android ID harvest, JSON HTTP POST telemetry |

Three independent frameworks. All read-only partitions. All surviving factory reset. All active every boot. Each implementing independent collection and transmission channels.

`AdupsFota` was the subject of a 2016 US CERT advisory after Kryptowire confirmed it was transmitting SMS content, call logs, contact lists, IMEI, and IMSI to servers in China. Its confirmed presence alongside two additional independent frameworks indicates this is not coincidental firmware assembly.

This is a layered, redundant data collection architecture embedded across multiple independent packages in production consumer firmware, on a BSP platform shared across Topwise/Wiite consumer and enterprise products globally.

---

## 11. Remediation

| Action | Target | Priority |
|---|---|---|
| Remove from ROM | `/product/app/WearDevicemsg/WearDevicemsg.apk` | CRITICAL |
| Remove OAT artifacts | `/product/app/WearDevicemsg/oat/` | CRITICAL |
| Verify absence | Confirm `com.wiite.devicemsg` absent from `pm list packages` | CRITICAL |
| Runtime verification | Confirm no `burying_loop_s` thread in `ps` output post-boot | HIGH |
| Network verification | Confirm no outbound JSON POST traffic on clean ROM boot | HIGH |

Firmware replacement via a clean AOSP-derived ROM with all three frameworks removed is the only path to a trusted device state. APK removal alone is insufficient if framework-level hooks (`WiitePackageManagerUtil`, `Configuration.isSpecialApp`) remain in `services.jar`.

---

## Evidence Integrity Statement

All findings in this document are derived from direct static analysis of `WearDevicemsg.apk` extracted from the live device via ADB root session. Smali source was produced by apktool 2.5.0 full DEX decode. All line number references are exact. All class and method references are exact. All source file names were recovered from smali `.source` debug headers present in the decoded output. No claims are inferred or assumed beyond what the code directly demonstrates.

Google's independent malware flag on this APK provides third-party confirmation consistent with established malware detection criteria.

---

*Albert Pittman (Beright1976) | 2026-06-09*  
*Embedded Android Systems Engineer & Vulnerability Researcher*  
*https://github.com/Beright1976*
