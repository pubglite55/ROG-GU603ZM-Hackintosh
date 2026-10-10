# Installation & Troubleshooting Guide — ROG Zephyrus M16 (GU603ZM) × macOS Sequoia 15 × NVIDIA driver

> Companion document: [README.md](README.md) (plan / config details / version list)  
> This guide assumes you start from **Windows 11**. Total time: ~1.5–3 hours (including downloads).

---

## 0. Preparation

| Item        | Requirement                                                                                                                           |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| USB drive   | ≥16 GB, will be **fully wiped**                                                                                                       |
| Target disk | A second empty disk or a reserved partition; **never touch the Windows partition**                                                    |
| Network     | Internet needed during install (the macOS recovery environment downloads the system); the wired Ethernet already works with this EFI  |
| Backup      | ⚠️ Back up important data before any partitioning; ideally do a full Windows backup first                                             |
| GPU mode    | **In Windows: Armoury Crate → GPU mode → Ultimate**, reboot once so the firmware remembers it (cannot be changed from the macOS side) |
| Ready       | This EFI package (config, 8 ACPI tables, 21 kexts, tools)                                                                             |

---

## 1. Make the installer USB

### Option A (recommended): manual with macrecovery

**A1. Format the USB as GPT + FAT32 (single partition)**

- GUI: use [Rufus](https://rufus.ie/) → select the USB → partition scheme **GPT** → target system **UEFI (non-CSM)** → filesystem **FAT32** → Start.  
  (Note: Windows Explorer's built-in "Format" does not offer FAT32 for 64 GB drives, so you must use Rufus or `diskpart`.)
- Command line (admin CMD):
  ```
  diskpart
  list disk                 ← identify your USB by size (do NOT pick the wrong disk!)
  select disk N
  clean
  convert gpt
  create partition primary
  format fs=fat32 quick
  assign
  exit
  ```
  ⚠️ `clean` wipes the selected disk. Double-check before proceeding.

**A2. Copy the EFI**

Copy this package's `EFI` folder **as a whole** to the USB root (i.e. `USB\EFI\BOOT\BOOTx64.efi`).

**A3. Download the macOS recovery image**

Open CMD/PowerShell, `cd` into the `macrecovery` directory (from OpenCorePkg `Utilities/macrecovery`), then run:

```
python macrecovery.py -b Mac-7BA5B2D9E42DDD94 -m 00000000000000000 -os latest download
```

- `-b` is the Board ID; the SMBIOS here is **MacBookPro16,1**, so use its board id. `-os latest` pulls the latest Sequoia 15.x.
- When done, a `com.apple.recovery.boot` folder appears (contains `BaseSystem.dmg` + `BaseSystem.chunklist`, ~700 MB–1 GB).

**A4. Copy the recovery folder**

Copy `com.apple.recovery.boot` to the USB root. Final layout:

```
USB\
├─ EFI\                        ← this package's EFI (OpenCore + our config)
└─ com.apple.recovery.boot\    ← recovery image
   ├─ BaseSystem.dmg
   └─ BaseSystem.chunklist
```

Safely eject — done.

### Option B (lazy alternative): 1401 Windows edition

1. Download: <https://github.com/nullmoth/1401/releases/download/v1.6.0/1401-Windows-1.6.0.zip>
2. Extract and run `1401.exe` (asks for admin) → follow: Check this PC → Build the Mac setup → read the BIOS steps → Create the macOS stick.
3. It generates **its own** OpenCore and installer (not this package's EFI).
   - If you just want to get running quickly, you can follow its flow end-to-end;
   - If you want **this carefully-tuned EFI** (recommended: auditable config + fully commented docs), use Option A, or after installing the OS use 1401.app to only do the "install driver" step (§4 Option A).

---

## 2. BIOS settings

Press **F2** at boot to enter BIOS (GUI), press `F7` for Advanced Mode if needed. Targets:

| Item                         | Setting                                                              | Note                                                                                            |
| ---------------------------- | -------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| Secure Boot                  | **Disabled**                                                         | already off on this machine                                                                     |
| CSM                          | **Disabled**                                                         | already off on this machine                                                                     |
| Fast Boot                    | **Disabled**                                                         | avoid skipping USB boot                                                                         |
| Above 4G Decoding            | **Enabled**                                                          | keep on                                                                                         |
| Resizable BAR                | **Enabled**                                                          | keep on; the actual BAR size is controlled by OC quirks (nullmoth's verified config = ReBAR on) |
| VT-d (if present)            | **Disabled** (or leave on, with `DisableIoMapper=true` as fallback)  | either works                                                                                    |
| VMD / Intel RST (if present) | **Disabled**                                                         | ensure NVMe passthrough; skip if not shown in this BIOS                                         |
| dGPU (if a toggle exists)    | **keep enabled**                                                     | this is the only usable GPU — never Eco/disable                                                 |
| Boot                         | after save & exit, press **Esc** at boot and pick `UEFI: <your USB>` | prefer the USB's UEFI entry                                                                     |

> Tip: GPU mode (Ultimate) is set in Windows Armoury Crate, not in BIOS; after changing it and rebooting, the display routes through the dGPU.

---

## 3. Install macOS

1. Boot from USB → OpenCore picker appears (graphical GoldenGate theme) → select **macOS Base System**.
2. Stalling at **"PCI configuration end" for a few minutes is normal** (especially after the driver is installed, up to ~100 s); if >5 min, see §6.
3. Enter the recovery environment → open **Disk Utility**:
   - On the left choose "Show All Devices" → select your **target disk** → Erase: format **APFS**, scheme **GUID Partition Map**.
   - ⚠️ Only operate on the disk/partition you prepared; never touch the disk with Windows.
4. Exit Disk Utility → **Reinstall macOS** → select the erased partition → install.
   - It downloads system files mid-way (keep wired or Wi-Fi connected — wired Ethernet is most reliable).
   - Laggy / unaccelerated display is **expected** (no NVIDIA driver yet; picture comes from the firmware framebuffer).
5. Several automatic reboots all boot from the USB (OpenCore remembers the last selection) → complete setup → enter the Sequoia desktop.

> The system has **no graphics driver yet**: usable but not smooth — don't install apps yet, do §4 first.

---

## 4. Install the NVIDIA driver (core step)

Prerequisite: you are on the Sequoia desktop (firmware-framebuffer picture is normal).

### Option A (recommended): 1401.app one-click

1. **Download ahead of time in Windows**: <https://github.com/nullmoth/nvidia-macos-driver/releases/download/v1.6.0/1401-Mac-1.6.0.dmg>  
   Copy it to the USB or any macOS-readable location (the driver installer is inside the dmg).
2. In macOS, open the dmg → run **1401** → follow the wizard:
   - It auto-finds "the OpenCore that boots this machine" (this EFI on the USB; keep the USB plugged in);
   - Shows all changes before applying (and backs up config.plist);
   - Automatically: installs 4 kexts to `/Library/Extensions` → rebuilds the Auxiliary Kernel Collection → switches BAR settings → adds a "1401: Remove NVIDIA driver" recovery entry.
3. **Reboot (keep the USB inserted)**. First boot may stall at "PCI configuration end" for ~1 minute — wait.
4. Verify:
   ```bash
   kmutil showloaded --list-only | grep nullmoth        # should print 4 lines
   system_profiler SPDisplaysDataType | head -20        # should show GeForce RTX 3060 / Metal: Supported
   ```
   At this point the internal panel/HDMI are driven by the RTX 3060 with Metal acceleration.

### Option B (manual, equivalent)

1. Download in Windows: <https://github.com/nullmoth/nvidia-macos-driver/releases/download/v1.6.0/nullmoth-nvidia-1.6.0.tar.gz> (copy into macOS).
2. macOS terminal:
   ```bash
   cd ~/Downloads
   tar -xzf nullmoth-nvidia-1.6.0.tar.gz && cd pkgroot
   shasum -a 256 -c SHA256SUMS     # continue only if all OK
   sudo ./install.sh               # install + rebuild the kernel collection
   ```
3. The BAR settings are already correct in this EFI (see README §6) — no config swap needed.
4. Reboot → verify as in Option A.

### Recovery from a bad install

| Situation                                      | Action                                                                                                                                                          |
| ---------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Won't boot / black screen after driver install | In **Windows**, mount the USB, edit `EFI/OC/config.plist` and prepend `-nvoff` to `boot-args` (driver skips loading), save and retry; troubleshoot once back in |
| Want to fully uninstall                        | Run the extracted `pkgroot/uninstall.sh` in macOS; or use the "1401: Remove NVIDIA driver" boot entry added by 1401                                             |
| Wrong config → black screen                    | Restore the backup config; or try the rescue configs in `调试-config/`                                                                                            |

---

## 5. Post-install polish (optional)

### 5.1 USB port mapping (recommended, once)

1. In Windows extract `USBToolBox-Windows.zip` → run `USBToolBox.exe`;
2. Choose **D** (Discovery on Windows) → plug a USB2 device (e.g. mouse) + a USB3 device (e.g. USB drive) into **each port** one by one and mark types:
   - The Bluetooth port (internal port near the panel; on this machine BT is on XHCI HS10) → type **255 / Internal**;
   - Others → USB2/USB3/Type-C by physical type;
3. Choose macOS-side export → get `UTBMap.kext`;
4. Put it in the USB's `EFI/OC/Kexts/` and add an entry in `config.plist` `Kernel > Add` (after `USBToolBox.kext`):
   ```
   BundlePath     = UTBMap.kext
   ExecutablePath = (empty)
   PlistPath      = Contents/Info.plist
   Enabled        = true
   ```
5. Reboot and verify all ports + Bluetooth.

### 5.2 Wi-Fi: install HeliPort

Download [HeliPort.dmg](https://github.com/OpenIntelWireless/HeliPort/releases/latest/download/HeliPort.dmg) → open → drag HeliPort into Applications → run → connect (tip: right-click Dock icon → Options → Open at Login).

### 5.3 Audio check

Default `layout-id=21`. If:

- No sound at all → try `11 → 88 → 31 → 71 → 33 → 52 → 61 → 66` (edit config `DeviceProperties > PciRoot(0x0)/Pci(0x1F,0x3) > layout-id`, reboot; or temporarily add `alcid=N` to boot-args for a quick test, then remove it);
- Verify SSDT-HPET and the two HPET patches are enabled (both required).

### 5.4 SIP / SecureBootModel

Keep `csr-active-config=430A0000` and `SecureBootModel=Disabled` (driver requirement). Minor system updates are fine, but **major upgrades (e.g. 26) are unsupported by this driver — do not upgrade across generations**.

### 5.5 Apple services (optional)

SMBIOS is **MacBookPro16,1**. The published config has Serial/MLB/UUID/ROM redacted to placeholders (`CHANGEME-SERIAL` / `CHANGEME-MLB` / zero UUID / `00:11:22:33:44:55`) — **generate your own triplets with GenSMBIOS and replace them** before use (config `PlatformInfo → Generic`). Before signing into iCloud/iMessage:

- Optional check: enter the serial at <https://checkcoverage.apple.com> — a normal invalid serial shows "not activated / cannot verify";
- Proceed at your own risk; try with a secondary account first.

### 5.6 Move the EFI to the internal disk (after stable, optional)

Booting from USB works fine (1401's recovery entry also depends on it). To boot without USB:

1. In macOS use `diskutil list` to find the internal disk's EFI partition → `sudo diskutil mount diskXs1` → copy the USB's `EFI` folder in (back up any old EFI first);
2. Reboot into BIOS and put "OpenCore" (or that disk's UEFI entry) first in boot order;
3. Re-run 1401.app once so it re-confirms the config.


## 7. Appendix

### 7.1 Key download links

| Item                                   | URL                                                                                                     |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| 1401 Windows tool (v1.6.0)             | <https://github.com/nullmoth/1401/releases/download/v1.6.0/1401-Windows-1.6.0.zip>                      |
| 1401 Mac driver installer (v1.6.0 dmg) | <https://github.com/nullmoth/nvidia-macos-driver/releases/download/v1.6.0/1401-Mac-1.6.0.dmg>           |
| NVIDIA driver manual package (tar.gz)  | <https://github.com/nullmoth/nvidia-macos-driver/releases/download/v1.6.0/nullmoth-nvidia-1.6.0.tar.gz> |
| Driver project                         | <https://github.com/nullmoth/nvidia-macos-driver>                                                       |
| OpenCore                               | <https://github.com/acidanthera/OpenCorePkg/releases>                                                   |
| Dortania install guide                 | <https://dortania.github.io/OpenCore-Install-Guide/>                                                    |

### 7.2 boot-args explained

| Arg                                                     | Purpose                                                | Removable?                           |
| ------------------------------------------------------- | ------------------------------------------------------ | ------------------------------------ |
| `keepsyms=1`                                            | show symbols on crash                                  | yes                                  |
| `debug=0x100`                                           | don't auto-reboot on panic                             | yes                                  |
| `nvfb=1 nvaccel=1 nvfbheads=4`                          | **nullmoth: framebuffer/accel/4 heads**                | ❌ keep                               |
| `-nvkmsnosmooth`                                        | **nullmoth: disable smooth scan**                      | ❌ keep                               |
| `amfi_get_out_of_my_way=0x1 amfi=0x80`                  | **nullmoth: allow WindowServer to load driver bundle** | ❌ keep                               |
| `revcpu=1 revpatch=cpuname,memtab revcpuname=i7-12700H` | RestrictEvents: CPU name / memory tabs                 | yes                                  |
| `-vi2c-force-polling`                                   | VoodooI2C: polling mode (trackpad)                     | keep if trackpad works               |
| `-wegnoigpu`                                            | hide driverless Iris Xe                                | keep                                 |
| `-nvrmnoflip`                                           | **nullmoth: internal panel flip (laptop required)**    | ❌ keep                               |
| `-nvoff` (rescue)                                       | temporarily **disable** the NVIDIA driver              | only for rescue, do not add normally |

### 7.3 Quick restore checklist (fresh install order)

```
Windows: Armoury Crate → Ultimate → reboot
Windows: make USB (§1) → copy EFI + recovery → reboot into BIOS (§2) → install (§3)
macOS: install driver (§4) → verify output → polish (§5)
```

Good luck. If any step stalls, send the `-v` boot log (or the `opencore-*.txt` OpenCore writes to the ESP) and the exact symptom.
