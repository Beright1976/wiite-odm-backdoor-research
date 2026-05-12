# Combined Threat Assessment: The Topwise/Wiite & Adups Attack Chain

**Researcher:** Albert Pittman (Beright1976)  
**Date:** 2026-05-12  
**Architecture Type:** Dual-Agent Command & Control (C2) and Silent Execution Pipeline  
**Privilege Level:** `android.uid.system` (UID 1000) / OS-Framework Level  
**Components:** `AdupsFota.apk` (Delivery & C2) + `Stopwatch.apk` (Execution & Privilege Escalation)  
**Evidence Base:** JADX decompilation of live APKs pulled from physical device via ADB root session  

---

## Executive Summary

Forensic decompilation of the MT6765/MT6762 Topwise/Wiite Board Support Package (BSP) reveals a highly engineered, dual-agent supply chain backdoor. The system relies on two seemingly independent applications that function as a single, coordinated framework for silent remote code execution and data exfiltration.

`AdupsFota` provides the network triggers, C2 communication, and payload delivery pipeline. `Stopwatch` provides the UID 1000 system-level privileges required to execute those payloads and install additional APKs silently. Both agents are intentionally protected by identical, non-standard framework-level hooks injected into the core Android OS to prevent them from being terminated by Android's memory management or discovered by standard security analysis.

This is not a collection of independent vulnerabilities. It is a coordinated, multi-phase attack chain engineered as a complete system.

---

## The Complete Attack Chain: Trigger to Execution

### Phase 1 — The Exfiltration and Check-in Trigger (AdupsFota)

The attack chain initiates automatically without any user interaction on every device boot.

**The Trigger:** `MyApplication.onCreate()` registers a `CONNECTIVITY_CHANGE` broadcast receiver on every boot:

```java
private void h() {
    IntentFilter intentFilter = new IntentFilter("android.net.conn.CONNECTIVITY_CHANGE");
    intentFilter.addAction("android.intent.action.ACTION_POWER_DISCONNECTED");
    intentFilter.addAction("android.intent.action.DATE_CHANGED");
    registerReceiver(new MyReceiver(), intentFilter);
}
```

**The Check-in:** The exact moment the device connects to WiFi or LTE, `MyReceiver` fires and calls `SystemActionService`.

**The Exfiltration:** The device silently contacts `fota5p.adups.com` and hits two endpoints simultaneously:

- `submitReport.do` — submits a device report to Adups servers
- `detectSchedule.do` — checks for remotely staged payloads

The server URLs are deliberately obfuscated in the DEX using character array construction to evade string-based security scanning:

```java
// Not written as a string — constructed character by character to defeat scanners
public static final String f1542a = String.valueOf(
    new char[]{'h','t','t','p','s',':','/','/','f','o','t','a','5','p','.','a','d','u','p','s','.','c','o','m'}
);
```

A secondary endpoint `fota5p.adups.cn` provides failover to a Chinese domestic server unreachable from most western security research environments.

---

### Phase 2 — Remote Payload Delivery (AdupsFota)

When the Adups C2 server decides to target a device, it responds to the `detectSchedule.do` check-in with a configured payload package. The response is parsed by the `VersionBean` class, which defines every field the server controls:

```java
public class VersionBean {
    private String deltaurl;    // Server dictates exactly what to download and from where
    private long filesize;      // Expected payload size
    private int isOldPkg;       // Package type flag
    private int issilent;       // CRITICAL: server-controlled silent install flag
    private String md5sum;      // Integrity verification
    private String sha;         // Secondary integrity verification
    private String versionName; // Version identifier
    private List<PolicyBean> policy;  // Server-defined installation policy
}
```

**Server-Controlled Targeting:** The `deltaurl` field gives the Adups server complete control over what the device downloads and from which location. This URL is not hardcoded — it is set fresh on every check-in response.

**The Silent Flag:** `issilent=1` is a server-set parameter that instructs the OS to bypass all standard Android user prompts, notifications, and UI dialogs. The install happens invisibly. There is no user-facing indicator of any kind.

**Delivery and Verification:** The payload downloads to a remotely configured `download_path_server` directory controlled by `StorageUtil`. MD5 and SHA fields verify payload integrity — ensuring only Adups-signed content executes.

**Storage Staging:** `StorageUtil.f()` creates persistent staging directories on every boot:

```java
new File(context.getFilesDir() + "/adupsfota").mkdirs();
new File(context.getExternalFilesDir(null) + "/adupsfota").mkdirs();
new File(context.getExternalFilesDir(null) + "/fota").mkdirs();
```

These directories persist across reboots and are invisible to standard file managers.

---

### Phase 3 — System-Level Execution (Stopwatch)

Once the payload is verified and staged, execution is handed off to the secondary agent.

**Baseband Exploitation:** If the payload is a modem update, `StorageUtil` manages the following files directly:

```java
// Explicit modem binary management — confirmed in StorageUtil.java
new File(str + "/modem.bin")   // Modem firmware
new File(str + "/nvitem.bin")  // NVRAM — contains IMEI and baseband calibration
new File(str + "/dsp.bin")     // DSP firmware
new File(str + "/vmjaluna.bin") // Virtual machine image
```

`nvitem.bin` is the NVRAM partition — it contains IMEI data and baseband calibration. AdupsFota has the confirmed infrastructure to silently download and manage modem-level firmware. This operates below Android entirely.

**Silent APK Installation:** If the payload is an Android application, execution is handed to `Stopwatch.apk`. Because Stopwatch runs as `android.uid.system` (UID 1000) from the read-only `/vendor/operator/app/` partition, it has maximum OS authority without declaring it in its manifest.

The payload is passed to `AppUtils.installApp()` from the bundled AndroidUtilCode library. Because Stopwatch inherits `INSTALL_PACKAGES` via system UID, the new application installs silently in the background — no confirmation dialog, no notification, no user awareness.

**Remote Command Execution:** The `FcmService` Firebase push channel provides a parallel execution path. Adups can push remote commands at any time:

| Command Type | Order | Action |
|---|---|---|
| Type 2 | Order 2 | Silent cache clear — `d.e(context)` executes immediately |
| Type 2 | Order 3 | OTA install flow triggered |
| Type 2 | Order 4/5 | Analysis commands |
| Type 1 | — | Push notification to user |
| Any | button==3 | Launch undisclosed component — **purpose unknown, under investigation** |

---

### Phase 4 — OS-Level Evasion and Persistence

To ensure the attack chain cannot be disrupted, the ODM implemented coordinated anti-analysis and persistence mechanisms across every layer of the Android stack.

**Framework Protection — Process Immunity:**  
`services.jar` was modified to inject `Configuration.isSpecialApp()` into `ActivityManagerService` and `DisplayPolicy`. This non-standard method hardcodes both agents as permanently exempt from:
- Background process killing
- Battery optimization enforcement  
- App standby bucketing
- Task management by any system component

`WearCleanTaskPro` — the ODM's own task killer — reads a hardcoded whitelist via `AbSharedUtil` and is physically blocked from killing AdupsFota or Stopwatch regardless of memory pressure.

**Framework Protection — Permission Bypass:**  
`WiitePackageManagerUtil` is injected into `PackageManagerService`. It reads `/system/etc/permissions/pms_sysapp_grant_permission_list.txt` — a purpose-built backdoor registry — and silently grants all requested permissions to listed packages, bypassing Android's standard permission validation entirely.

**Active Tamper Detection:**  
On every boot AdupsFota verifies its own C2 URL and silently resets it if changed:

```java
String strD = o.d(this, "check_url");
if (!TextUtils.isEmpty(strD) && !strD.equals(com.adups.fota.b.b.f1542a)) {
    // Researcher redirected the URL — silently reset it back
    o.b(f1498a, "check_url", com.adups.fota.b.b.f1542a);
}
```

This specifically defeats the standard researcher technique of redirecting traffic to a capture proxy for analysis.

**Factory Reset Survival:**  
Both agents are stored in read-only partitions:
- `Stopwatch.apk` — `/vendor/operator/app/` — vendor partition, survives factory reset
- `AdupsFota.apk` — `/product/priv-app/` — product partition, survives factory reset

Firmware replacement is the only remediation path.

---

### Phase 5 — Network-Level Evasion

**User Agent Spoofing:**  
`CustomPropInterface` is injected into the Android framework and uses Java reflection to actively spoof the Browser User Agent string and Build Release Date on all outbound traffic. Network monitoring tools see a different device identity than the physical hardware. This defeats network fingerprinting and makes traffic attribution to the specific device impossible without deep packet inspection.

**URL Obfuscation:**  
All server endpoints are constructed via character arrays — a deliberate technique to defeat automated string extraction tools used by app stores, antivirus engines, and security researchers. A standard `strings` scan of the DEX returns no readable server URLs.

**Dual Endpoint Architecture:**  
Primary server `fota5p.adups.com` handles international traffic. Failover server `fota5p.adups.cn` handles Chinese domestic traffic and is unreachable from most western security analysis environments, providing a blind spot in any network-level investigation.

**Version Spoofing:**  
`PackageUtils.java` hardcodes a consistent version string regardless of the actual installed version:

```java
// Always reports "5.28" to the server — actual version is masked
return versionName.equalsIgnoreCase("5.28") ? versionName : "5.28";
```

This ensures the server always sees a consistent device identity that cannot be used to fingerprint individual installations.

---

## Confirmed Open Items

Two components in the attack chain have not been fully analyzed and require further investigation:

| Item | Location | Status |
|---|---|---|
| `button==3` component launch | `FcmService.java` — remote command handler | Target class is obfuscated — payload and purpose unknown |
| `CompanyActivity` | `Stopwatch.apk` | Layout loads but `initView()` and `initData()` are empty — initialization may occur via reflection or native code |

Both represent unknown execution paths in an already confirmed malicious system.

---

## Conclusion

This is not an OTA update mechanism. It is a remote silent install and exfiltration pipeline controlled entirely from servers operated by Adups Technology in China.

The evidence is unambiguous:
- `issilent=1` is a server-controlled flag — legitimate OTA systems do not silently install without user consent
- `CONNECTIVITY_CHANGE` as the exfiltration trigger — legitimate OTA systems check on a schedule, not on every network connection
- Character array URL obfuscation — legitimate software does not hide its server addresses from string scanners
- Active tamper detection resetting the C2 URL — legitimate software does not detect and reverse researcher analysis attempts
- Modem firmware management infrastructure — legitimate OTA systems do not manage `nvitem.bin` (IMEI/NVRAM) files
- Dual-agent architecture with coordinated framework protection — legitimate software components do not require OS-level hooks to prevent being killed

Whatever Adups chooses to exfiltrate or install, they have the complete infrastructure to do it silently, persistently, and with cryptographic verification — on every device running this BSP globally. The scope extends beyond consumer watches to POS terminals, logistics handheld scanners, and industrial devices handling financial transactions and inventory management worldwide.

---

*All findings forensically verified via JADX decompilation of APKs pulled directly from the live device. Source code references are exact. No claims are inferred or assumed.*  
*beright1976 | 2026*
