# Changelog

All notable changes to this project are documented here. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/), and this project adheres to [Semantic Versioning](https://semver.org/).

## [1.0.0] - 2026-10-10

### Added
- Full OpenCore 1.0.8 EFI for the ASUS ROG Zephyrus M16 (GU603ZM).
- SMBIOS `MacBookPro16,1` (triplets redacted to placeholders).
- nullmoth NVIDIA driver configuration — RTX 3060 Laptop verified working (Metal 3, internal 2560×1600@165Hz + HDMI).
- CPU model-name fix (`revcpu=1` + RestrictEvents NVRAM keys) → shows "i7-12700H".
- ALC285 audio via AppleALC (codec detected; layout rotation in progress).
- ELAN1010 trackpad support (Alder Lake I2C ID patch + VoodooI2C PlugIns + GPIO dependency).
- GoldenGate graphical boot theme (OpenCanopy).
- ACPI plan: SSDT-EC/USBX/PLUG-ALT/RTCAWAC/HPET/XOSI + DSDT replacement + APIC (14-core alignment).
- Documentation: README (EN) / README.zh-CN, INSTALL (EN) / INSTALL.zh-CN, debug notes (EN/ZH).
- MIT license.

### Changed
- N/A (first release).

### Fixed
- DSDT load failure (26 conditional-scope "guard If" → NoOp) → all 21 ACPI tables load with 0 failures.
- Alder Lake hybrid-core CPU boot (table-level 14-core surgery + CpuTscSync + non-monotonic-time patches).
- Windows dual-boot: `ScanPolicy=0xF0103` so the OC menu no longer shows a Windows entry (boot Windows via BIOS F8).

### Known issues
- Audio: ALC285 codec is detected but the speaker pin-map needs a layout rotation (currently `21`).
- Trackpad: ELAN1010 patch applied but not yet confirmed on a live boot.
- Wi-Fi: itlwm + HeliPort (no native menu-bar icon / AirDrop / Handoff).
- USB-C/Thunderbolt4 port unsupported on macOS.
