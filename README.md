# Dell Latitude 7490 Hackintosh EFI
# macOS 12.7.3 Monterey with Intel Wireless via AirportItlwm, updated to resolve random kernel panic via wifi mbuf issue

Dell 14" Latidude 7490 i5-8650U OpenCore configuration
Has BootChime enabled, debug modes turned off, non verbose

Thanks to all those that allowed me to find working configs (boy the trackpad was a fun one)

## Tested macOS version:
macOS 12.7.3 Using latest OpenCore as of Feb 6, 2024

## System configuration
- Intel Core i5-8650U
- 32GB DDR4-2400MHz RAM (2x16Gb)
- Intel UHD 620 Graphics
- Idk some generic NVMe 256Gb drive
- Intel Dual-Band Wireless-AC 8265

## Confirm working
1) Loudspeaker
2) 3.5mm headphone jack
3) Battery Management
4) Backlight Control (Fn+B & Fn+S)
5) Touchpad with Gestures
6) WiFi (2.4/5GHz)
7) Bluetooth
8) USB-C Display Output
9) Barrel Plug for Charging (Dell)
10) USB-C for Charging
11) iGPU Acceleration
12) Native hotkey support with Fn keys

^_3.5mm headphone jack might not work properly after system wakes from sleep, reboot will resolve the issue._

## Not working
1) HDMI output

## Unknown (not tested)
1) Intel TurboBoost (Unable to monitor)
2) Ethernet
3) Handoff and Airdrop (Likely not working due to AppleItlwm Limitations)
4) USB-C Data Transfer
5) WWAN
6) SD Card Reader
7) ThunderBolt and Touchscreen (SKU did not include)8
8) SideCar over WiFi

## Credits
6) https://raw.githubusercontent.com/lohseng97/Dell-Latitude-7490-EFI-macOS-BigSur
1) Swung0x48/Dell-Latitude-7390-7490-Hackintosh-EFI: https://github.com/Swung0x48/Dell-Latitude-7390-7490-Hackintosh-EFI/tree/Catalina-OpenCore
2) niiknow/Hackintosh-Latitude-7390: https://github.com/niiknow/Hackintosh-Latitude-7390
3) dortania/OpenCore-Install-Guide: https://dortania.github.io/OpenCore-Install-Guide/
4) OpenIntelWireless/itlwm: https://github.com/OpenIntelWireless/itlwm/
5) xzhih/one-key-hidpi：https://github.com/xzhih/one-key-hidpi


## Wi-Fi: patched AirportItlwm 2.3.0 (fixes mbuf_watchdog kernel panic)

The Intel 8265 Wi-Fi driver shipped here is **AirportItlwm 2.3.0 (Monterey build) plus a one-line fix**
for a receive-path memory leak present in upstream itlwm 2.2.0, 2.3.0 and master.

**Symptom:** after ~10 days of uptime the machine panics with
`mbuf_watchdog: 1 waiters stuck for N secs` and the 4 KB mbuf cluster pool ~100% in use
(e.g. 57769/57858). Wi-Fi DNS/ARP fail a couple of minutes before the panic.
Upstream reports: OpenIntelWireless/itlwm #1010, #975, #728.

**Cause:** in `itl80211/openbsd/net80211/ieee80211_input.c`, `ieee80211_amsdu_decap()` splits an
A-MSDU into subframe `m` and remainder `n`; when `ieee80211_amsdu_decap_validate()` rejects the
subframe only `m` is freed and `n` leaks. OpenBSD frees both. Received frames live in 4 KB
RX clusters (`IWM_RBUF_SIZE 4096`), so each hit pins a 4 KB cluster forever.

**Fix:** `Patches/AirportItlwm-2.3.0-amsdu-mbuf-leak.patch` (adds `mbuf_freem(n)`), applied to
upstream tag `v2.3.0`, built with Xcode 14.2 + MacKernelSDK, target `AirportItlwm-Monterey`, Release.
Binary SHA-256: `0324a70a7b60cfea04e5a699bb99573090042d85ca3f745a4e65c5450903b399`.
No `config.plist` change is required; copy the kext over `EFI/OC/Kexts/AirportItlwm.kext`.

**Check:** `netstat -m | grep 4KB` — "4KB clusters in use" should stay in the low hundreds.
