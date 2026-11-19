# Pixel 8 Pro: intermittent Wi-Fi and Bluetooth failure

**Date:** 2026-11-19  
**Device:** Google Pixel 8 Pro  
**Environment:** Windows PC, Android Platform Tools (ADB), USB debugging  
**Outcome:** Wi-Fi networks became visible and connection succeeded after ADB-based recovery. Bluetooth recovery was **not independently verified**. Root cause remains **unconfirmed**.

## Impact and symptoms

- Wi-Fi toggle could be enabled, but nearby networks were initially absent from the phone's Internet settings and the phone could not connect.
- Bluetooth was also malfunctioning during the incident.
- A forced restart / complete shutdown did not resolve the immediate symptom.
- This was a recurrence of a prior wireless issue, during which Bluetooth HAL and coexistence-related messages had been observed.

## Investigation

### 1. Confirmed ADB access and Wi-Fi state

```cmd
adb devices -l
adb shell cmd wifi status
adb shell cmd wifi list-networks
```

ADB communication worked. Android reported Wi-Fi enabled but disconnected. `list-networks` reported no configured networks; that command shows **saved/configured networks**, not all access points in range.

An attempt to run `adb shell ip link show` returned `Permission denied` under Android's shell restrictions; this does not prove a broken interface.

### 2. Examined scanning

```cmd
adb shell cmd wifi start-scan
adb shell cmd wifi list-scan-results
```

Scanning returned multiple nearby 2.4 GHz and 5 GHz access points. The printed `Age(sec)` values used a decimal comma (for example `35,152` = about 35.152 seconds, **not** 35,152 seconds). This was not evidence of hours-old scan cache.

From `dumpsys wifi`:
- Primary Wi-Fi interface: `wlan0`, with `mIfaceIsUp: true`.
- Scan counters included `SCAN_SUCCESS: 22`, `mWifiLogProto.numNonEmptyScanResults=20`, and `mWifiLogProto.numEmptyScanResults=0`.
- Saved-network count: `mWifiLogProto.numSavedNetworks=0`.
- Android reported disconnected, with no associated BSSID or IP address.

These observations established that the Wi-Fi radio and scanning path could operate during testing, even while the UI had failed to show networks.

### 3. Observed a Wi-Fi subsystem recovery

The `dumpsys wifi` history contained:

```text
2026-11-19 01:39:58  CMD_RECOVERY_RESTART_WIFI  ...  Subsystem Restart
2026-11-19 01:40:02  CMD_RECOVERY_RESTART_WIFI_CONTINUE
```

Subsequent logs included `WifiNative: Scan result ready event` and `WifiScanner: onFullResults`.

The earlier, separate wireless incident had reported Bluetooth HAL crashes and coexistence-related errors. Those historical observations are relevant, but they do not establish the exact cause of this incident.

### 4. Location and connection settings

```cmd
adb shell cmd location is-location-enabled
adb shell cmd wifi list-networks
```

Location returned `true` when checked. No saved Wi-Fi networks were reported.

## Recovery applied

A Wi-Fi service toggle was used, followed by a fresh scan:

```cmd
adb shell svc wifi disable
timeout /t 5
adb shell svc wifi enable
timeout /t 10
adb shell cmd wifi start-scan
timeout /t 10
adb shell cmd wifi list-scan-results
adb shell cmd wifi status
```

After this procedure, nearby networks appeared in Android's Internet settings again.

The phone was then successfully connected to the selected WPA2 network using ADB (credentials intentionally omitted):

```cmd
adb shell cmd wifi connect-network "<SSID>" wpa2 "<PASSWORD>"
adb shell cmd wifi status
```

**Confirmed result:** Wi-Fi network visibility returned and the user reported that the ADB connection succeeded.

**Not confirmed:** Whether Bluetooth recovered, whether the issue will recur, or whether any specific driver/firmware/hardware component was defective.

## Working hypothesis — not a proven root cause

The simultaneous Wi-Fi/Bluetooth symptoms, together with the recorded Wi-Fi subsystem recovery and historical Bluetooth HAL/coexistence messages, make an intermittent shared wireless-stack, firmware, driver, or hardware issue plausible. However, successful scanning and connection mean a *permanent* Wi-Fi radio failure was not demonstrated. A UI/network-selection-layer problem also cannot be excluded.

Do **not** label this as a confirmed hardware defect or a confirmed Android bug without more evidence.

## If the problem returns

1. Before toggling Wi-Fi, capture evidence:
   ```cmd
   adb shell cmd wifi status
   adb shell cmd wifi list-scan-results
   adb shell dumpsys wifi > wifi_diagnostic.txt
   adb shell dumpsys bluetooth_manager > bluetooth_diagnostic.txt
   adb logcat -d -b all -v time > wireless_logcat.txt
   adb bugreport pixel-wireless-incident.zip
   ```
2. Inspect logs for subsystem restarts, Bluetooth HAL failures, scan failures, firmware errors, and timestamps.
3. Run the [Wi-Fi recovery runbook](../runbooks/adb-wifi-recovery.md).
4. Test Wi-Fi browsing **and** Bluetooth discovery/pairing separately.
5. If it recurs, check Android updates; consider Android's **Reset Bluetooth & Wi-Fi** (which deletes saved networks and pairings) before considering more disruptive steps.
6. Redact sensitive material before sharing logs or uploading them to GitHub.

## Lessons learned

- Separate **saved networks** (`cmd wifi list-networks`) from **visible access points** (`cmd wifi list-scan-results`).
- Interpret locale-formatted decimals in scan ages correctly.
- Check whether `wlan0` is up and whether scans actually complete before blaming hardware.
- Document the recovery **and** the unresolved underlying cause.
- ADB is useful even when the Android network-selection UI is malfunctioning.
