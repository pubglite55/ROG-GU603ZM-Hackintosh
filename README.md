# ROG 幻16 (GU603ZM) OpenCore EFI —— macOS Sequoia 15 × NVIDIA 方案

> 更新日期：2026-10-10 ｜ OpenCore 1.0.8 ｜ 目标系统：**macOS Sequoia 15.x**
> 依据 Dortania OpenCore-Install-Guide 与 sqlsec/Hackintosh（国光）规范构建，
> 独显方案基于 [nullmoth/nvidia-macos-driver](https://github.com/nullmoth/nvidia-macos-driver)（macOS 15 专用社区 NVIDIA 驱动）。

---

## ⚠️ 使用前必读：三个前提 + 一个安全事项

1. **目标系统只能是 macOS 15 (Sequoia)** —— nullmoth 驱动仅支持 15.x（实测 15.7/15.8）。不要尝试 Sonoma / 26。
2. **BIOS / GPU 模式**：Windows 下用 Armoury Crate 把 GPU 模式切到 **Ultimate（MUX 独显直连）**，并关闭核显输出——本机 Iris Xe 在 macOS 无驱动，只有内屏/HDMI 物理直连 RTX 3060 才有显示。
3. **不要混用旧 EFI**：本机曾被 RapidEFI 风格的旧 EFI 引导（注入 14 张 SSDT + 打 `XSTA/XCRS/XOSI` 改名）。本包 ACPI 是对**原厂表还原后**重新生成的，请整体替换、勿叠加旧配置。
4. **⚠️ 三码已脱敏**：本仓库的 `config.plist` 中序列号/MLB/UUID/ROM 均为占位符（`CHANGEME-*` / 全零 UUID / `00:11:22:33:44:55`）。**使用前必须用 GenSMBIOS 重新生成一套三码并替换**，否则无法登录 iCloud/iMessage，且多人共用同一三码会被 Apple 封号。

---

## 1. 硬件支持矩阵

| 部件 | 型号 / ID | 状态 | 说明 |
|---|---|---|---|
| CPU | i7-12700H (Alder Lake-H) | ✅ | SSDT-PLUG-ALT + `ProvideCurrentCpuInfo`；机型名已修正显示为 i7-12700H |
| 核显 | Iris Xe 8086-46A6 | ❌ | Apple 从未支持 11 代后 Intel 核显，无解（不参与输出） |
| 独显 | RTX 3060 Laptop 10DE-2520 (GA106M) | ✅ **实测点亮** | nullmoth 驱动，Metal 3 加速、内屏 2560×1600@165Hz + 外接 HDMI 双屏正常 |
| 内屏 | AUO 2560×1600 165Hz | ✅ | 经 MUX 直连 dGPU，nullmoth 驱动点亮 |
| HDMI 2.0b | 直连 dGPU | ✅ | 同上 |
| USB-C(DP)/雷电4 | 8086-461E | ❌/⚠️ | 雷电/USB4 控制器 macOS 无支持，该口 USB/视频大概率不可用；安装盘插 USB-A |
| 音频 | Realtek ALC285 10EC-0285 | ⚠️ 调优中 | codec 已正确识别，扬声器引脚映射 layout 需轮换（见 §3.1） |
| 有线网口 | Realtek RTL8125 2.5GbE | ✅ | LucyRTL8125Ethernet，免配置 |
| Wi-Fi | Intel AX211 (8086-51F0) | ⚠️ | itlwm.kext + HeliPort.app（Sequoia 无 AirportItlwm） |
| 蓝牙 | Intel AX211 (8087-0033) | ⚠️ | IntelBluetoothFirmware + IntelBTPatcher + BlueToolFixup |
| 键盘 | ATK PS/2 (ATK3001) | ✅ | VoodooPS2Controller |
| 触控板 | ELAN1010 @ I2C1 (8086-51E9) | ⚠️ 待验证 | 已补 Alder Lake I2C ID + GPIO 依赖，见 §3.6 |
| SD 读卡器 | Genesys GL9755 | ❌ | 无 macOS 驱动 |
| 摄像头 | UVC (13D3-56EB) | ✅ | 免驱 |

---

## 2. NVIDIA 独显方案（核心）

### 2.1 nullmoth 驱动

[nullmoth/nvidia-macos-driver](https://github.com/nullmoth/nvidia-macos-driver) 是 2026-10-08 发布的社区开源驱动：

- 架构：`Metal 应用 → NVMTLDriver.bundle（Apple AIR→SPIR-V 翻译）→ NVK（Mesa Vulkan + NAK）→ NVRM/NVAccel/NVRMFB/NVRMAGDC → GSP 固件 (610.57.04) → GPU`
- 设备表覆盖 Turing / Ampere / Ada / Blackwell（GTX 16、RTX 20/30/40/50，含笔记本卡）。
- **本机 RTX 3060 Laptop (GA106M) 已在设备表，且本项目已实机验证点亮**（Metal 3）。

### 2.2 关键原则：不要屏蔽独显

| 项目 | 做法 |
|---|---|
| `-wegnoegpu` / `nv_disable=1` | ❌ 禁用（会关掉唯一可用显卡） |
| WhateverGreen | ❌ 已剔除（其 NVIDIA 处理与本驱动冲突） |
| GPU 模式 Ultimate | ✅ 必须 |
| nullmoth boot-args | ✅ 已预置 `nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth amfi_get_out_of_my_way=0x1 amfi=0x80` |
| 笔记本必需参数 | ✅ `-nvrmnoflip`（否则内屏不翻转点亮） |

### 2.3 实测验证结论（本项目）

- **不要加 `-v`**：verbose 会饿死驱动的 auto-go 线程，超过 `settle+40s` 上限后驱动放弃点亮（详见 `调试-config/` 与对话记录）。
- BAR 用官方推荐组：`UEFI ResizeGpuBars=13` + `Booter ResizeAppleGpuBars=-1`（驱动会自动把 BAR1 扩到 8GB）。
- 首次启动停在 `[ PCI configuration begin ]` 是正常等待点；最多等约 3 分钟。
- 装好驱动后启动日志关键字：`NVRM-xnu: PASS 2 REACHED` / `nvrm-boot-hold = "display armed"`。

### 2.4 为什么不走 OCLP

- OCLP 的 NVIDIA 支持只到 Kepler/Maxwell/Pascal（最高 Ventura）；**Ampere（RTX 30 系）OCLP 无解**。
- nullmoth 不依赖 OCLP：kext 装 `/Library/Extensions` + `kmutil` 重建 AuxKC，只需 `csr-active-config=430A0000` + `SecureBootModel=Disabled`（均已预置）。

---

## 3. 其他驱动要点

### 3.1 音频（ALC285，调优中）
- AppleALC 注入 `layout-id` 到 `PciRoot(0x0)/Pci(0x1F,0x3)`。
- 当前 codec 已正确识别（`IOHDACodecVendorID=0x10EC0285`），但 layout 21 在本机引脚映射不对（会回退到 generic）。
- **轮换顺序建议**：`11 → 88 → 31 → 71 → 33 → 52 → 61 → 66`（`调试-config/声卡-layoutNN.plist` 已备好，每换一个重启试声）。
- SSDT-HPET + 两条 HPET 重命名补丁已就位（音频 IRQ 必需）。

### 3.2 有线网口
- RTL8125 2.5GbE → LucyRTL8125Ethernet.kext，免配置。

### 3.3 Wi-Fi
- Sequoia 起无 AirportItlwm → 用 **itlwm.kext + HeliPort.app**（工具目录已附 HeliPort.dmg）。代价：无原生 Wi-Fi 菜单图标、AirDrop/接力不可用。

### 3.4 蓝牙
- IntelBluetoothFirmware + IntelBTPatcher + BlueToolFixup。注意 USB 端口映射时保留蓝牙所在 HS10（类型 Internal/255）。

### 3.5 USB
- XHCI (8086-51ED) 不在 Apple 原生表 → 已含 `XHCI-unsupported.kext`（daliansky 版）。
- 尚未做端口映射，首装用 USB-A 口；装好后用 USBToolBox 生成 UTBMap.kext。

### 3.6 触控板（ELAN1010，已修复待验证）
- 本机触控板 ELAN1010 挂 `_SB.PC00.I2C1.ETPD`（`_CID=PNP0C50`），I2C 主控 8086-51E9（Alder Lake），GPIO 主控 INTC1055。
- 上游 VoodooI2C 2.9.1 匹配表**不含 Alder Lake**，本项目做了两处修复：
  1. 给 `VoodooI2C.kext/Contents/Info.plist` 追加 `51E8/51E9/51EA/51EB/51C8` 到 `VoodooI2CPCIController.IOPCIMatch`；
  2. config 补上 VoodooI2C 内部 3 个 PlugIn（`VoodooGPIO` / `VoodooI2CServices` / `VoodooInput`）作为独立条目（上游漏配会导致主 kext 依赖解析失败、整体不加载）。
- `SSDT-GPI0.aml`（RapidEFI 残留，引用不存在的 `_SB.GPI0.XSTA`）已**移除**——本机 GPI0 原始 `_STA` 在 macOS 下本就直接返回启用。

---

## 4. EFI 目录结构

```
EFI-GU603ZM-Sequoia-OC1.0.8/
├─ EFI/
│  ├─ BOOT/BOOTx64.efi                 # OpenCore 引导入口
│  └─ OC/
│     ├─ OpenCore.efi                  # OpenCore 1.0.8
│     ├─ config.plist                  # ★ 当前生效配置（安装+装后通用）
│     ├─ config-postinstall-NVIDIA.plist  # 装驱动后可选替换（BAR 微调）
│     ├─ ACPI/                          # SSDT + 原厂表替换（见 §5）
│     ├─ Drivers/                       # OpenRuntime / OpenHfsPlus / ResetNvramEntry
│     ├─ Kexts/                         # 21 个驱动（见 §7）
│     └─ Tools/
├─ 工具/                               # USBToolBox / HeliPort.dmg / macrecovery
├─ 参考资料/                            # SSDT 源码 / SSDTTime 补丁 / 版本记录
└─ 调试-config/                         # 各类备用/救援/声卡 layout 变体（见该目录说明）
```

---

## 5. ACPI 方案

### 5.1 SSDT（SSDTTime 生成）
| SSDT | 作用 |
|---|---|
| SSDT-EC | 假 EC（不干扰真实 EC0_） |
| SSDT-USBX | USB 供电属性 |
| SSDT-PLUG-ALT | CPU 电源管理 plugin-type=1 |
| SSDT-RTCAWAC | AWAC→RTC 切换 |
| SSDT-HPET | HPET `_CRS/_STA` 重建（音频 IRQ 必需） |
| SSDT-XOSI | `_OSI→XOSI` 配合 |

另有 **DSDT.aml**（整体替换，修 Alder Lake 条件式 USB 端口声明导致的 DSDT 加载失败）与 **APIC.aml**（MADT 14 核对齐）。

### 5.2 ACPI 补丁（三条，与 SSDT 成对依赖）
`HPET _STA→XSTA`、`HPET _CRS→XCRS`、`_OSI→XOSI`。

> ⚠️ 三条补丁与对应 SSDT 成组启用/禁用，勿单独拆。

---

## 6. config.plist 关键项

### 6.1 Booter / Kernel 关键 quirks
- `AppleXcpmCfgLock=true`（ROG BIOS 无 CFG Lock 开关）
- `ProvideCurrentCpuInfo=true`（Alder Lake 混合核必需）
- `DisableIoMapper=true`（兜底 VT-d）
- `AvoidRuntimeDefrag` / `RebuildAppleMemoryMap` / `SyncRuntimePermissions` 等现代必需项
- `Kernel → Block: IONDRVSupport (Exclude)`：装驱动后启用（防固件帧缓冲抢 display 0）

### 6.2 NVRAM / 启动参数
| 键 | 值 |
|---|---|
| boot-args | `keepsyms=1 debug=0x100 nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth amfi_get_out_of_my_way=0x1 amfi=0x80 revcpu=1 revpatch=cpuname,memtab revcpuname=i7-12700H -vi2c-force-polling -wegnoigpu -nvrmnoflip` |
| csr-active-config | `430A0000` |
| prev-lang:kbd | `zh-Hans:252` |

> 说明：`revcpu/revpatch/revcpuname` 同时写在 NVRAM `4D1FDA02-...` GUID 下（RestrictEvents 的真实读取位置），`-wegnoigpu` 用于隐藏无驱动的 Iris Xe。

### 6.3 SMBIOS（MacBookPro16,1）
- 机型：`MacBookPro16,1`（2019 款 16 寸，dGPU 机型，符合本方案）
- **三码已脱敏为占位符**，使用前请用 GenSMBIOS 重新生成并替换（序列号 / MLB / SystemUUID / ROM）。

### 6.4 关于 Windows 双系统
- OpenCore 的 ACPI 注入是全局的（改过的 DSDT/APIC 会一起交给 Windows，导致 `ACPI_BIOS_ERROR 0xA5`）。
- 本 EFI 已设 `ScanPolicy=0xF0103`，OC 菜单**不显示 Windows 入口**。
- **进 Windows 请用 BIOS 的 F8 启动菜单选 "Windows Boot Manager"**（拿到原厂干净 ACPI，正常启动）。

---

## 7. 版本清单

| 组件 | 版本 |
|---|---|
| OpenCore | 1.0.8 |
| Lilu | 1.7.2 |
| VirtualSMC / SMCProcessor / SMCBatteryManager | 1.3.8 |
| AppleALC | 1.9.8 |
| RestrictEvents | 1.1.6 |
| NVMeFix | 1.1.3 |
| BlueToolFixup | 2.7.2 |
| IntelBluetoothFirmware / IntelBTPatcher | v2.4.0 |
| itlwm | v2.3.0 |
| LucyRTL8125Ethernet | 1.2.300 |
| ECEnabler | 1.0.6 |
| VoodooPS2Controller | 2.3.7 |
| VoodooI2C / VoodooI2CHID | 2.9.1（含本项目 Alder Lake ID 补丁） |
| USBToolBox | 1.2.0 |
| XHCI-unsupported | daliansky 维护版 |

---

## 8. 安装与验证

1. BIOS 设置：Above 4G Decoding / Resizable BAR 开启，VT-d 可关，Secure Boot 关闭，GPU 模式 Ultimate。
2. 生成三码替换占位符 → 写盘 → 插 USB-A 口启动。
3. 详细安装步骤、BIOS 设置、故障排查见 **《安装与调试指南.md》**。
4. 两份 config 均通过 ocvalidate 1.0.8 校验（0 error）。

---

## 9. 致谢

Dortania OpenCore-Install-Guide、sqlsec/Hackintosh（国光）、Acidanthera、CorpNewt（SSDTTime）、OpenIntelWireless、nullmoth（驱动/1401）、daliansky、Mieze、VoodooI2C 等社区项目。

## 10. 许可

本项目 EFI 配置文件与文档以 **MIT License** 发布（见 `LICENSE`）。各 Kext / 驱动 / SSDT 的版权归其各自作者所有，遵循其原始许可证。
