# 安装与调试指南 —— ROG 幻16 (GU603ZM) × macOS Sequoia 15 × NVIDIA 驱动

[English](INSTALL.md) | **简体中文**

> 配套文件：《README.md》（方案说明 / 配置详解 / 版本清单）
> 本指南假设你从 **Windows 11** 起步（当前状态）。全程约需 1.5~3 小时（含下载时间）。

---

## 0. 准备工作

| 项目 | 要求 |
|---|---|
| U 盘 | ≥16GB，会被**全部清空**（桌面上这支 aigo U356 64G 即可） |
| 目标安装盘 | 建议用第二块空盘（你的 YMTC 256G）或在 Micron 上预留的分区；**绝对不要动 Windows 分区** |
| 网络 | 安装阶段需联网（macOS 恢复环境会下载系统主体）；有线网口在本 EFI 中已可用 |
| 备份 | ⚠️ 分区操作前先备份重要数据；建议先跑一次 Windows 完整备份 |
| GPU 模式 | **Windows 下 Armoury Crate → GPU 模式 → Ultimate**，重启一次让固件记住（macOS 侧无法更改此设置） |
| 已完成 | 本 EFI 包（含两份 config、6 SSDT、17 kext、工具） |

---

## 1. 制作安装 U 盘

### 方案 A（推荐）：macrecovery 手工制作

**A1. 格式化 U 盘为 GPT + FAT32 单分区**

- 图形方式：用 [Rufus](https://rufus.ie/) 选择该 U 盘 → 分区类型 **GPT** → 目标系统 **UEFI (非 CSM)** → 文件系统 **FAT32** → 开始。
  （注意：Windows 资源管理器自带的"格式化"对 64GB U 盘不提供 FAT32 选项，所以必须用 Rufus 或下面的 diskpart。）
- 命令行方式（管理员 CMD）：
  ```
  diskpart
  list disk                 ← 找到你的 U 盘编号（按容量辨认，务必别选错！）
  select disk N
  clean
  convert gpt
  create partition primary
  format fs=fat32 quick
  assign
  exit
  ```
  ⚠️ `clean` 会清空所选磁盘，选错会毁掉你的硬盘，操作前反复确认。

**A2. 拷贝 EFI**

把本包的 `EFI` 文件夹**整体**复制到 U 盘根目录（即 `U盘\EFI\BOOT\BOOTx64.efi`）。

**A3. 下载 macOS 恢复镜像**

打开 CMD/PowerShell，cd 到本包 `工具\macrecovery` 目录，执行：

```
python macrecovery.py -b Mac-7BA5B2D9E42DDD94 -m 00000000000000000 -os latest download
```

- 没装 Python 的话：系统里有 Python 用 `macrecovery.bat` 双击也行；或先装 python.org 的 Python 3。
- 参数说明：`-b` 是 iMacPro1,1 的 Board ID（与本 EFI 的 SMBIOS 配套），`-os latest` 拉最新版（即最新 Sequoia 15.x）。
- 完成后同目录会出现 `com.apple.recovery.boot` 文件夹（内含 `BaseSystem.dmg` 和 `BaseSystem.chunklist`，约 700MB~1GB）。

**A4. 拷贝恢复目录**

把 `com.apple.recovery.boot` 文件夹复制到 U 盘根目录。最终结构：

```
U盘\
├─ EFI\                        ← 本包的 EFI（OpenCore + 我们的 config）
└─ com.apple.recovery.boot\    ← 恢复镜像
   ├─ BaseSystem.dmg
   └─ BaseSystem.chunklist
```

安全弹出，完成。

### 方案 B（懒人替代）：1401 Windows 版

1. 下载：https://github.com/nullmoth/1401/releases/download/v1.6.0/1401-Windows-1.6.0.zip
2. 解压后运行 `1401.exe`（会请求管理员权限）→ 按界面：Check this PC → Build the Mac setup → 读 BIOS 步骤 → Create the macOS stick。
3. 它会自动生成**它自己的** OpenCore 和安装盘（不是本包的 EFI）。
   - 如果你只是想快速跑通，可以全程用它的流程；
   - 如果你想用**本包精心配置的 EFI**（推荐，配置项可审计、带完整注释文档），请用方案 A，或装完系统后按 §4 方案 A 用 1401.app 只做"装驱动"这一步。

---

## 2. BIOS 设置

开机连按 **F2** 进入 BIOS（GUI 版），按 `F7` 进 Advanced Mode（如需）。目标设置：

| 项目 | 设置 | 说明 |
|---|---|---|
| Secure Boot | **Disabled** | 本机当前已是关闭状态 |
| CSM | **Disabled** | 本机当前已是关闭状态 |
| Fast Boot | **Disabled** | 避免跳过 USB 引导 |
| Above 4G Decoding | **Enabled** | 保持开启（当前开） |
| Resizable BAR | **Enabled** | 保持开启；具体 BAR 大小由 OC 的 quirks 控制（nullmoth 验证配置即 ReBAR 开启） |
| VT-d（如有） | **Disabled**（或保持开，由 `DisableIoMapper=true` 兜底） | 两者其一即可 |
| VMD / Intel RST（如有） | **Disabled** | 确保 NVMe 直通；本机 BIOS 若不显示此项则跳过 |
| 独显（若有 dGPU 开关） | **保持开启** | 这是唯一可用显卡，绝不能选 Eco/禁用 |
| 启动 | 保存退出后开机连按 **Esc** 呼出启动菜单，选择 `UEFI: <你的U盘名>` | 优先选 U 盘的 UEFI 项 |

> 提示：GPU 模式（Ultimate）是在 Windows 的 Armoury Crate 里改的，BIOS 里没有；改完重启一次后，U 盘启动时屏幕就走独显了。

---

## 3. 安装 macOS

1. 从 U 盘启动 → 出现 OpenCore 选择器（纯文本简单界面）→ 选 **macOS Base System**（有 `-v` 详细模式滚动，属正常）。
2. **卡在 "PCI configuration end" 几分钟属正常**（尤其以后装了驱动，最长约 100 秒）；若超过 5 分钟见 §6。
3. 进入恢复环境 → 打开 **磁盘工具**：
   - 左侧选"显示所有设备"（Cmd+2？此处指左上角显示菜单）→ 选中你的**目标安装盘** → 抹掉：格式 **APFS**，方案 **GUID 分区图**。
   - ⚠️ 只操作你准备好的那块盘/分区；带 Windows 的磁盘一律不碰。
4. 退出磁盘工具 → **重新安装 macOS** → 选择刚抹好的分区 → 安装。
   - 中途会联网下载系统文件（保持有线或 Wi-Fi 连接——若 Wi-Fi 需要先联网，用有线网口最稳）。
   - 屏幕画面卡顿/无加速是**预期内**的（还没有 NVIDIA 驱动，画面来自固件帧缓冲）。
5. 多次自动重启均从 U 盘引导（OpenCore 会记住上次选择；也可以每次手动选）→ 完成系统设置向导 → 进入 Sequoia 桌面。

> 此时系统**尚无图形驱动**：能操作但不流畅，别急着装软件，先完成 §4。

---

## 4. 安装 NVIDIA 驱动（核心步骤）

前提：已进入 Sequoia 桌面（画面来自固件帧缓冲属正常）。

### 方案 A（推荐）：1401.app 一键装

1. **Windows 侧提前下载**：https://github.com/nullmoth/nvidia-macos-driver/releases/download/v1.6.0/1401-Mac-1.6.0.dmg
   拷到 U 盘或任何 macOS 能读的位置（驱动安装包就在 dmg 里，macOS 侧无需再下载）。
2. macOS 里双击打开 dmg → 运行 **1401** → 按向导走：
   - 它会自动找到"启动本机的 OpenCore"（即 U 盘上的本 EFI，保持 U 盘插着）；
   - 安装前显示所有改动（并备份 config.plist）；
   - 自动完成：装 4 个 kext 到 `/Library/Extensions` → 重建 Auxiliary Kernel Collection → 切换 BAR 设置（= 把我们提供的 `config-postinstall-NVIDIA.plist` 的那 3 处差异自动改好）→ 添加"1401: Remove NVIDIA driver"恢复入口；
   - 首次可能提示需要重启一次再继续（若 SIP 尚未降级；本 EFI 已预置 `430A0000`，通常一步到位）。
3. **重启（保持 U 盘插入）**。首次启动可能在 "PCI configuration end" 停约 1 分钟，等待即可。
4. 验证：
   ```bash
   kmutil showloaded --list-only | grep nullmoth        # 应输出 4 行
   system_profiler SPDisplaysDataType | head -20        # 应显示 GeForce RTX 3060 / Metal: Supported
   ```
   到这一步，内屏/HDMI 应已由 RTX 3060 驱动，Metal 加速可用。

### 方案 B（手动，等效）

1. Windows 侧下载：https://github.com/nullmoth/nvidia-macos-driver/releases/download/v1.6.0/nullmoth-nvidia-1.6.0.tar.gz（拷进 macOS）。
2. macOS 终端：
   ```bash
   cd ~/Downloads
   tar -xzf nullmoth-nvidia-1.6.0.tar.gz && cd pkgroot
   shasum -a 256 -c SHA256SUMS     # 全部 OK 再继续
   sudo ./install.sh               # 安装 + 重建内核集合
   ```
3. 切换配置：把本包 `EFI/OC/config-postinstall-NVIDIA.plist` **重命名/替换** 为 `EFI/OC/config.plist`（先备份原文件）。
   （或手动改那 3 处，见 README §4.1；改完可用 ProperTree 检查。）
4. 重启 → 验证同方案 A。

### 翻车恢复

| 情况 | 处理 |
|---|---|
| 装完驱动无法开机 / 黑屏 | 在 **Windows** 里挂载 U 盘，用 ProperTree（或记事本）编辑 `EFI/OC/config.plist`，在 `boot-args` 最前面加 `-nvoff`（驱动会跳过加载），保存后重试；能进系统后再排查 |
| 想彻底卸载驱动 | macOS 终端运行你解压的 `pkgroot/uninstall.sh`；或使用 1401 安装时加的 "1401: Remove NVIDIA driver" 引导入口（若装过 1401） |
| 切错 config 导致黑屏 | 换回备份 config；或用 U 盘里两份 config 互相替换试 |

---

## 5. 收尾完善（按需）

### 5.1 USB 端口映射（推荐做，一次就好）

1. Windows 里解压 `工具\USBToolBox-Windows.zip` → 运行 `USBToolBox.exe`；
2. 选 **D**（Discovery on Windows）→ 按提示用一个 USB2 设备（如鼠标）+ 一个 USB3 设备（如 U 盘）**逐个插入每个端口**并标记类型：
   - 蓝牙端口（内屏附近那个内部口，参考：本机 BT 在 XHCI 的 HS10）类型选 **255 / Internal**；
   - 其余按物理口类型选 USB2/USB3/Type-C；
3. 完成后选 **macOS 端导出** → 得到 `UTBMap.kext`；
4. 把它放进 U 盘的 `EFI/OC/Kexts\`，并编辑 `config.plist` 的 `Kernel > Add` 增加一条（放在 `USBToolBox.kext` 之后）：
   ```
   BundlePath     = UTBMap.kext
   ExecutablePath = （留空）
   PlistPath      = Contents/Info.plist
   Enabled        = true
   MinKernel/MaxKernel = （留空）
   ```
   （用 ProperTree 打开 config：右键 Kernel > Add → New child… 参考 USBToolBox.kext 那条的样子填。）
5. 重启验证所有端口 + 蓝牙。

### 5.2 Wi-Fi：安装 HeliPort

把 `工具\HeliPort.dmg` 拷进 macOS → 打开 → 把 HeliPort 拖进"应用程序" → 运行 → 选择 Wi-Fi 连接（建议：右键 Dock 图标 → 选项 → 在登录时打开）。

### 5.3 音频检查

默认 layout-id=11。若：
- 完全无声 → 依次试 21 / 31 / 61 / 71（改 config `DeviceProperties > PciRoot(0x0)/Pci(0x1F,0x3) > layout-id`，改完重启；也可临时在 boot-args 加 `alcid=21` 快速试验，但测完请删除以避免与注入冲突）；
- 确认 SSDT-HPET 与两条 HPET 补丁处于启用状态（缺一不可）。

### 5.4 SIP / 安全模型

保持 `csr-active-config=430A0000`、`SecureBootModel=Disabled`（驱动要求，勿随意改成全开）。系统更新一般不受影响，但注意：**大版本（如 26）暂不支持本驱动，不要升跨代**。

### 5.5 Apple 服务（可选）

SMBIOS 为生成的 MacBookPro16,1。发布版已把序列号/MLB/UUID/ROM 替换为占位符（`CHANGEME-SERIAL` / `CHANGEME-MLB` / 全零 UUID / `00:11:22:33:44:55`），**使用前请自行用 GenSMBIOS 生成一套三码并替换**（config 里 `PlatformInfo → Generic` 四个字段）。登录 iCloud/iMessage 前：
- 可选自检：https://checkcoverage.apple.com 输入序列号，正常应显示"请激活/无法验证"这类无效状态；
- 风险自担，建议用小号先试。

### 5.6 把 EFI 挪到内置硬盘（稳定后再做，可选）

U 盘引导完全可用（1401 的恢复入口也依赖它）。若想免 U 盘：
1. 在 macOS 用 `diskutil list` 找到内置盘 EFI 分区 → `sudo diskutil mount diskXs1` → 把 U 盘 `EFI` 文件夹整个拷进去（若已有旧 EFI，先改名备份）；
2. 重启进 BIOS 启动项把 "OpenCore" 或该盘的 UEFI 项调到第一；
3. 之后建议重跑一次 1401.app 让它确认配置（或手动维持两份 config 的切换规则）。

---

## 6. 故障排除速查

| 现象 | 可能原因 | 处理 |
|---|---|---|
| 启动直接进 Windows | 启动项没选 U 盘 / Secure Boot 没关净 | 开机 Esc 选 `UEFI: U盘`；BIOS 关闭 Secure Boot |
| OpenCore 能进，选 macOS 后卡 `EXITBS:START` | 内存映射类 quirk 与固件不匹配（华硕机型高发） | 直接用包内 `调试-config\` 里的备用配置按顺序试（先 `备用1-SetupVirtualMap关闭`），详见该文件夹内《说明.txt》 |
| 卡 `PCI configuration end` 很久 | 装驱动后首次启动正常（≤100 秒）；或 BAR 设置有误 | 等；若 >5 分钟：确认当前 config 与阶段匹配（§README 4.1），或加 `-nvoff` 救援 |
| 装好驱动后黑屏/无信号 | GPU 没在 Ultimate / config 未切换到 postinstall / boot-args 丢失 | ① Armoury Crate 设 Ultimate；② 换 config-postinstall 那份；③ NVRAM 重置后重启（boot-args 会自动重写） |
| 安装期能看到界面但只能"固件画面" | 正常现象（无驱动） | 继续安装，装完驱动即正常 |
| `still waiting for root device` | USB 端口/映射问题 | 换 **USB-A** 口插安装盘；做 §5.1 映射；确认 XHCI-unsupported.kext 已加载 |
| 个别 USB 口不工作 | 未做端口映射（默认端口表不完整） | §5.1 生成 UTBMap.kext |
| 没声音 | layout 不对 / HPET 补丁缺失 | 按 §5.3 依次换 layout；确认补丁与 SSDT-HPET 成组启用 |
| Wi-Fi 无 | 没装 HeliPort / itlwm 未加载 | 安装 HeliPort；`kmutil showloaded | grep itlwm` 检查 |
| 蓝牙无 | BT 端口被裁 / kext 未加载 | 映射时保留 BT 口（Internal）；确认三个 BT kext 顺序（Firmware→BTPatcher→BlueToolFixup） |
| 触控板无反应 | **已知限制**：ADL I2C 控制器无驱动 | 用 USB 鼠标；等社区更新 VoodooI2C（本包已预置 kext，未来更新版本即可） |
| 外接 USB-C 显示器无信号 | 该口走 iGPU/雷电，macOS 不支持 | 改用 **HDMI** 口（直连 dGPU） |
| 睡眠即醒 / 唤醒异常 | 默认未做 GPRW 定制 | 可选：参考社区"GPRW to XPRW"补丁方案（旧 EFI 曾用过）自行加装；或先关闭自动睡眠 |
| 想回 Windows 直启 | 默认启动项被旧 OC 占用 | BIOS 启动顺序改回 `Windows Boot Manager`；不需要的旧引导项可在 BIOS 启动菜单里清理 |
| 系统更新后驱动失效 | 跨大版本 / KC 需重建 | 只留在 15.x 内更新；异常时重跑 1401.app 或 `install.sh` 重建 |

---

## 7. 附录

### 7.1 关键下载地址

| 内容 | 地址 |
|---|---|
| 1401 Windows 工具（v1.6.0） | https://github.com/nullmoth/1401/releases/download/v1.6.0/1401-Windows-1.6.0.zip |
| 1401 Mac 驱动安装器（v1.6.0 dmg） | https://github.com/nullmoth/nvidia-macos-driver/releases/download/v1.6.0/1401-Mac-1.6.0.dmg |
| NVIDIA 驱动手动安装包（tar.gz） | https://github.com/nullmoth/nvidia-macos-driver/releases/download/v1.6.0/nullmoth-nvidia-1.6.0.tar.gz |
| 驱动项目主页 | https://github.com/nullmoth/nvidia-macos-driver |
| OpenCore 官方 | https://github.com/acidanthera/OpenCorePkg/releases |
| Dortania 安装指南 | https://dortania.github.io/OpenCore-Install-Guide/ |
| 国光 Hackintosh 教程（中文） | https://github.com/sqlsec/Hackintosh |

### 7.2 boot-args 逐项说明

| 参数 | 作用 | 可删？ |
|---|---|---|
| `-v` | 详细启动日志（调试期建议保留） | 稳定后建议删 |
| `keepsyms=1` | 崩溃时显示符号 | 可删 |
| `debug=0x100` | panic 后不自动重启（便于拍屏） | 可删 |
| `nvfb=1 nvaccel=1 nvfbheads=4` | **nullmoth：帧缓冲/加速/4 显示头** | ❌ 勿删 |
| `-nvkmsnosmooth` | **nullmoth：关闭平滑扫描** | ❌ 勿删 |
| `amfi_get_out_of_my_way=0x1 amfi=0x80` | **nullmoth：允许 WindowServer 加载驱动 bundle** | ❌ 勿删 |
| `revpatch=cpuname,memtab` `revcpuname=i7-12700H` | RestrictEvents：CPU 名称/内存页显示美化 | 可删 |
| `-nvoff`（救援用） | 临时**禁用** NVIDIA 驱动（用于黑屏救援） | 正常时不要加 |

### 7.3 一键复原清单（新机重装流程顺序）

```
Windows：Armoury Crate → Ultimate → 重启
Windows：做 U 盘（§1）→ 拷 EFI + recovery → 重启进 BIOS（§2）→ 装系统（§3）
macOS：装驱动（§4）→ 验证 4+1 行输出 → 收尾（§5）
```

祝折腾顺利。有任何一步卡住，把 `-v` 启动日志（或 OpenCore 写到 ESP 的 `opencore-*.txt`）发出来即可定位。
