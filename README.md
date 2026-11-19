# Android ADB Troubleshooting

Hands-on Android diagnostics and incident reports using Android Debug Bridge (ADB), `logcat`, and `dumpsys`.

The examples document observed behavior, distinguish confirmed findings from hypotheses, and avoid publishing credentials, device identifiers, or real network identifiers.

## Incident reports

- [Pixel 8 Pro — simultaneous Wi-Fi/Bluetooth malfunction (2026-11-19)](incidents/pixel-8-pro-wifi-bluetooth-2026-11-19.md): Wi-Fi enabled but no nearby networks shown in Android Settings; Bluetooth also malfunctioned. Recovered Wi-Fi discovery and connectivity with ADB; on recurrence Wi-Fi initialization failed and a forced complete shutdown restored operation. Root cause remains unconfirmed.

## Runbooks

- [ADB Wi-Fi scan and recovery](runbooks/adb-wifi-recovery.md): restart Wi-Fi, request a scan, inspect results, connect to a network, and preserve diagnostics before recovery.

## Safety

- Never commit Wi-Fi passwords, authentication tokens, private device identifiers, or unsanitized `adb bugreport` archives.
- `adb logcat` and `dumpsys` outputs may contain sensitive details; review before publishing.
- Commands target a locally authorized Android device with USB debugging enabled.
- Recovery commands provide a workaround, not a guaranteed permanent fix.
