# Backup Configs & Debug Notes (English)

> Chinese version: [NOTES.zh-CN.txt](NOTES.zh-CN.txt)
> This directory holds the full debug history of getting macOS to boot on this machine.
> The `备用*.plist` files are drop-in `config.plist` replacements for specific rescue scenarios.

---

## EXITBS:START stall — backup configs

Symptom: OpenCore freezes at the instant it exits (last log line `[EB|LOG:EXITBS:START]`), screen frozen. This is a "memory-mapping quirk vs firmware mismatch", common on ASUS machines.

Four backup configs (based on the install-stage config, only Booter memory items changed), try in order, one file at a time:

| Config | Change |
|---|---|
| 备用1 | disable `SetupVirtualMap` — try first (most common) |
| 备用2 | 备用1 + disable `DevirtualiseMmio` |
| 备用3 | 备用2 + disable `ProtectMemoryRegions` / `ProtectUefiServices` |
| 备用4 | disable `SetupVirtualMap` + `RebuildAppleMemoryMap` + `SyncRuntimePermissions`, enable `EnableWriteUnprotector` (no-MAT firmware) |

Usage (assumes this package's EFI on a USB with a visible EFI partition):
1. Open `USB\EFI\OC\`, back up the current `config.plist` as `config.plist.bak`.
2. Copy the chosen 备用N file to `EFI\OC\` and rename it `config.plist` (overwrite).
3. Reboot and retry macOS.
4. Only ever swap `config.plist` — never touch Kexts / ACPI / Drivers.

---

## The two root causes that finally made macOS boot (key conclusions)

After a long campaign, two fundamental fixes unlocked the boot:

### Root cause 1 — Alder Lake hybrid-core CPU stack
`x86_validate_topology` panic ("14 threads but 20 registered from MADT") and the frozen
`AppleACPICPU: ProcessorId=2` line were caused by a missing complete CPU stack for Alder Lake hybrid cores. Final fix (mirrors 1401's verified 12th-gen policy):
- `Cpuid1Data` spoof (Comet Lake) + `ProvideCurrentCpuInfo=true`
- `CpuTopologyRebuild.kext` (v2.0.2) + boot-arg `ctrsmt=full`
- 3× "force HT enabled" kernel patches (for Sequoia, the `MinKernel=24.0.0` entry)
- Table-level CPU surgery (`sel14b`): per-P-core keep only the first thread (LAPIC 0/8/16/24/32/40), drop all second threads (1/9/17/25/33/41), keep all 8 E-cores (48/50/52/54/56/58/60/62) → 14 CPUs = 14 cores = 14 enabled MADT entries.
- `CpuTscSync.kext` + 2 "Remove non-monotonic time panic" kernel patches (TSC sync for the newly-enabled cores).

### Root cause 2 — DSDT load failure (deleted namespace)
Error chain: `ACPI Error: [\_SB.PC00.XHCI.RHUB.HS01] Namespace lookup failure` → `(DSDT) table load failure` → `AcpiNsDeleteNamespaceByOwner` removes the whole DSDT namespace → 10 table cascading failures.
Cause: the firmware creates USB port devices (HS01-14/SS01-10) conditionally with `If (CondRefOf(PU2C/PU3C))`. Apple's old ACPICA (20160930) does not execute module-level conditional code, so the ports don't exist when the later guards reference them → DSDT fails (Windows/newer ACPICA skip the branches, so only macOS hits this).
Fix: byte-level replace the 26 "guard If" headers with equal-length NoOp, making the USB declarations unconditional → ship `DSDT.aml` replaced via `ACPI→Add` (OpenCore semantics: a DSDT table = replace, not append). 26 spots = 2 outer (PU2C/PU3C) + 14 HS01-14 + 10 SS01-10. Also baked the three renames (`HPET _STA→XSTA`, `_CRS→XCRS`, `_OSI→XOSI`) directly into DSDT (config ACPI→Patch only runs on firmware tables, before Add, so the replaced DSDT wouldn't receive them). Rebuild script: `dsdt-fix/make_dsdt.py`.

Final result: `21 ACPI AML tables successfully acquired and loaded` (0 failures).

---

## Post-install fixes (audio / network / trackpad / GPU)

- **Network**: use itlwm.kext + HeliPort (OCLP-Mod's Wi-Fi patch conflicted with itlwm). Wired RTL8125 works out of the box.
- **Audio**: ALC285; rotate `layout-id` 11 → 88 → 31 → 71 → 33 → 52 → 61 → 66 (layout 21 was already tried).
- **Trackpad**: ELAN1010 on Alder Lake I2C (8086-51E9). Two fixes: append `51E8/51E9/51EA/51EB/51C8` to `VoodooI2CPCIController.IOPCIMatch`, and add VoodooI2C's 3 PlugIns (`VoodooGPIO`/`VoodooI2CServices`/`VoodooInput`) to config. Removed the RapidEFI `SSDT-GPI0.aml` leftover (referenced nonexistent `_SB.GPI0.XSTA`).
- **GPU (nullmoth)**: verified lit (Metal 3). Key learnings:
  - Do NOT add `-v` (starves the driver's auto-go thread past the `settle+40s` cap).
  - Use `-nvrmnoflip` (laptop requirement for the internal panel).
  - BAR: `UEFI ResizeGpuBars=13` + `Booter ResizeAppleGpuBars=-1`.
  - First driver boot stalls at `[ PCI configuration begin ]` — normal, wait up to ~3 min.

---

## Rescue configs

| File | Purpose |
|---|---|
| 备用20-救援-屏蔽驱动.plist | Block the 4 nullmoth kexts + restore pre-driver display (firmware framebuffer) |
| 备用21-驱动齐活版-nvrmsettle+wegnoigpu.plist | Full driver stack + `nvrmsettle=10000` + `-wegnoigpu` |
| 备用24-应急-固件屏优先-把桌面弄回来.plist | Firmware-framebuffer-first rescue to get the desktop back |
| 声卡-layoutNN.plist (×9) | Audio layout rotation variants |
| 备用1…备用19 | Historical boot-debug variants (see NOTES.zh-CN.txt for the full story) |
