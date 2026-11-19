# ADB Wi-Fi scan and recovery (Windows CMD)

Use this when Android reports Wi-Fi enabled but nearby access points are not visible or connection is failing.

**Prerequisites:** Android Platform Tools installed; USB debugging enabled; phone unlocked and authorized in ADB.

## 1. Check the device and capture initial state

```cmd
adb devices -l
adb shell cmd wifi status
adb shell cmd wifi list-networks
adb shell cmd wifi list-scan-results
```

`list-networks` lists configured networks; `list-scan-results` lists detected access points. An empty configured-network list does **not** mean scanning failed.

## 2. Preserve evidence before attempting recovery (recommended for recurring failures)

```cmd
adb shell dumpsys wifi > wifi_diagnostic.txt
adb shell dumpsys bluetooth_manager > bluetooth_diagnostic.txt
adb logcat -d -b all -v time > wireless_logcat.txt
```

For a comprehensive package:

```cmd
adb bugreport pixel-wireless-incident.zip
```

**Privacy:** These artifacts may contain private network names, MAC addresses, and other identifying information. Never upload unreviewed reports to a public repository.

## 3. Restart Wi-Fi and force a scan

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

`timeout /t` is Windows CMD syntax. In PowerShell use `Start-Sleep -Seconds 5` instead.

Check that networks appear in **Settings → Network & internet → Internet**. The `Age(sec)` column may use a locale-specific decimal comma, so `19,730` may mean about 19.73 seconds rather than 19,730 seconds.

## 4. Connect to a network (optional)

```cmd
adb shell cmd wifi connect-network "<SSID>" wpa2 "<PASSWORD>"
adb shell cmd wifi status
```

Use the security mode appropriate for the access point. **Do not hardcode credentials in a GitHub repository.** A password supplied on a command line may also remain in terminal history.

## 5. Diagnose if recovery fails

```cmd
adb shell dumpsys wifi > wifi_diagnostic.txt
adb shell dumpsys bluetooth_manager > bluetooth_diagnostic.txt
adb logcat -d -b all -v time > wireless_logcat.txt
```

Search for messages including `CMD_RECOVERY_RESTART_WIFI`, `WifiScanner`, `WifiNative`, `WifiHAL`, `coex`, `firmware`, and Bluetooth HAL errors.

If the Wi-Fi UI fails but `list-scan-results` contains fresh entries, distinguish display/network-selection issues from radio scanning. If Wi-Fi and Bluetooth fail together, investigate their shared subsystem without prematurely declaring a hardware fault.

## 6. If Wi-Fi cannot initialize even after ADB enable

On a recurring Pixel 8 Pro incident, `settings get global wifi_on` returned `1` while `cmd wifi status` still reported `Wifi is disabled`. Diagnostics showed repeated `CMD_STA_START_FAILURE` / `WifiNative Failure`. Wi-Fi toggles did not recover it.

After saving diagnostics, **fully power off / force shutdown the phone and power it back on**, then recheck:

```cmd
adb shell cmd wifi status
adb shell cmd wifi start-scan
adb shell cmd wifi list-scan-results
```

A complete forced shutdown **restored Wi-Fi on that recurrence**, but a previous shutdown had not helped during an earlier phase. Do not treat this as a permanent fix or as proof of either hardware or software failure. If the fault keeps recurring, preserve logs for vendor support and test Bluetooth separately.

## Validation checklist

- Networks are visible on the phone.
- Wi-Fi connects to the intended SSID.
- Internet traffic works.
- Bluetooth can discover and connect to a test device.
- Problem does not immediately recur.

**Caution:** This is a workaround, not a permanent repair. Do not factory-reset solely because one scan fails.
