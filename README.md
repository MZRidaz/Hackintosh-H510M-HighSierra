# Hackintosh High Sierra：微星 H510M BOMBER + i7-10700F + GTX 1050 Ti

一套**实机验证通过**的黑苹果方案：Intel 10 代平台 + NVIDIA Pascal 显卡，安装 **macOS High Sierra 10.13.6（最终版本 17G14042）**，显卡 Web Driver 全加速，可从硬盘 EFI 分区直接启动（免 U 盘）。

> 背景：GTX 1050 Ti（Pascal 架构）的 NVIDIA 官方 Web Driver 只支持到 macOS 10.13，且 i7-10700F 无核显、没有退路，所以这台机器的上限就是 High Sierra。本方案基于 OpenCore 0.9.8 完成，并把安装过程中踩过的每一个坑都记录在下面。

## 硬件配置

| 部件 | 型号 |
|---|---|
| CPU | Intel Core i7-10700F（Comet Lake，无核显，CPUID 伪装为 Kaby Lake 0x906E9） |
| 主板 | 微星 MSI H510M BOMBER（MS-7D22） |
| 内存 | 32 GB DDR4 |
| 显卡 | NVIDIA GeForce GTX 1050 Ti 4 GB |
| 网卡 | Intel I219-V（板载） |
| 声卡 | Realtek ALC897（AppleALC，alcid=11） |
| 硬盘 | 系统盘 NVMe SSD（Windows）+ WD 500 GB HDD（macOS，本方案装在这块盘上） |
| 机型伪装 | iMac18,3（Retina 5K, 27-inch, 2017） |

## 当前状态（均已实机验证）

- ✅ macOS 10.13.6（17G14042，最终安全更新 2020-006）
- ✅ GTX 1050 Ti 显卡全加速（Web Driver 387.10.10.10.40.140）
- ✅ 有线网卡、声卡、USB 键鼠、4K 显示输出正常
- ✅ EFI 已搬入硬盘 ESP 分区，开机直进 OpenCore（Windows / macOS 双系统菜单）
- ⚠️ 装好显卡驱动后**永远不要再点系统更新**，版本号一变驱动立即失效

## 仓库内容

| 路径 | 说明 |
|---|---|
| `EFI/OC/config.plist` | 完整可用的 OpenCore 配置模板（**三码已脱敏**，见下文） |
| `SSDT/SSDT-AWAC.dsl`、`SSDT/SSDT-PLUG.dsl` | 本方案用到的 SSDT 源码（SSDT-EC-USBX 使用 Dortania 预编译版） |
| `docs/install-guide-zh.md` | 完整的中文安装指南（分步操作版） |

> 本仓库不包含 OpenCore 本体与 kext 二进制文件，请按下方版本表从官方仓库下载对应版本，放入 `EFI` 相应目录后与本配置配合使用。

## 依赖版本（与 config.plist 配套）

- **OpenCore 0.9.8**（[acidanthera/OpenCorePkg](https://github.com/acidanthera/OpenCorePkg)）
- 驱动（EFI/OC/Drivers）：OpenHfsPlus.efi、OpenRuntime.efi、ResetNvramEntry.efi（随 OpenCore 发行包）

| Kext | 版本 | 来源 |
|---|---|---|
| Lilu | 1.5.4 | acidanthera/Lilu |
| VirtualSMC（含 SMCProcessor、SMCSuperIO） | 1.2.5 | acidanthera/VirtualSMC |
| WhateverGreen | 1.5.6 | acidanthera/WhateverGreen |
| AppleALC | 1.6.3 | acidanthera/AppleALC |
| IntelMausi | 1.0.7 | acidanthera/IntelMausi |
| USBToolBox + UTBDefault | 1.2.0 | USBToolBox/kext |
| XHCI-unsupported | 0.9.2（手工加入 500 系控制器 0x43ED 的匹配） | RehabMan/OS-X-USB-Inject-All |
| VoodooPS2Controller | 2.2.8 | acidanthera/VoodooPS2Controller |

## 安装要点（详细步骤见 docs/install-guide-zh.md）

1. **BIOS**：关闭 Secure Boot / Fast Boot，SATA 设为 AHCI，开启 XHCI Hand-off，UEFI 模式，核显选项选 PEG。
2. **做 U 盘**：OpenCorePkg 的 macrecovery 下载 High Sierra 恢复镜像，U 盘根目录放 `com.apple.recovery.boot`（镜像）和 `EFI`（本方案）两个文件夹；安装全程保持有线网络连接（在线恢复安装）。
3. **装系统**：OpenCore 菜单里选**带 `(dmg)` 的那一项**（见坑 #1）；目标分区提前在 Windows 里缩出 ≥120 GB 并建占位分区，只格式化那一个分区，**绝不动 Windows 的分区**。
4. **升到最终版本**：先别装显卡驱动，进系统后在 App Store 里依次装安全更新 2020-005、2020-006，把版本升到 **17G14042**。
5. **装显卡驱动**：安装与 17G14042 严格对应的 **WebDriver 387.10.10.10.40.140**，重启后「关于本机」里显卡应显示 GTX 1050 Ti 4095 MB。
6. **EFI 搬入硬盘**：把 EFI 复制进 macOS 所在硬盘的 ESP 分区，BIOS 里把该盘的 BBS 优先权排第一、启动选项 #1 设为 UEFI Hard Disk，即可脱离 U 盘启动。

## ⚠️ 实机踩坑记录（本方案最有价值的部分）

1. **OpenCore 菜单选错会"假死"**：菜单第 1 项其实是 OpenCore 自己的 BOOTX64.efi，选了会报 `Already started` 然后卡死，看起来像键盘失灵。**真正的安装项是带 `(dmg)` 的那一项。**
2. **恢复界面键鼠失灵**：500 系主板（XHCI 控制器 0x43ED）+ High Sierra，必须用 **USBToolBox + UTBDefault** 这套 USB 方案（外加 XHCI-unsupported 补上 500 系 ID）；USBInjectAll 方案在本机无效。
3. **"未能与恢复服务器取得联系"的真因不是网络**：High Sierra 的 SSL 校验自 2023 年前后已损坏，HTTPS 拉取软件更新目录必然失败（真 Mac 同样中招）。解法是给 NVRAM 写入 `IASUCatalogURL` 指向同一目录的 **http** 地址：
   `http://swscan.apple.com/content/catalogs/others/index-10.13-10.12-10.11-10.10-10.9-mountainlion-lion-snowleopard-leopard.merged-1.sucatalog`
   本仓库的 config 已移除该项（仅安装阶段临时需要，装完应删除）。
4. **目标盘没有 ESP 时安装器拒绝安装**（"无法使用这个磁盘来启动电脑"）：老 Windows 数据盘常没有 EFI 分区，可在磁盘尾部空闲扇区用 `gpt add` 新建一个 256 MB 的 ESP 再格式化，不要动已有分区。
5. **NVIDIA 驱动必须和系统 build 号严格对应**：40.140 只配 17G14042；顺序必须是"先把安全更新升满 → 再装驱动 → 之后永不更新"，搞反了就得重来。
6. **ESP 分区类型被改掉后主板不认启动**：若 ESP 的分区类型 GUID 被工具改成普通数据分区（EBD0A0A2…），BIOS 里不会出现它的启动项。在 Windows 里用 diskpart 执行 `set id=C12A7328-F81F-11D2-BA4B-00A0C93EC93B` 改回即可。
7. **ESP 在 macOS 里挂载失败的两个细节**：用 `newfs_msdos` 格式化时簇数必须 ≥ 65525 才满足 FAT32（267 MB 分区需 `-c 2`）；系统启动后 `diskutil mount` 会拒绝当前启动盘的 ESP，改用 `sudo mount -t msdos /dev/diskXsY /Volumes/EFI` 手动挂载。

## 关于三码（必读）

`config.plist` 里的 `SystemSerialNumber`、`MLB`、`ROM`、`SystemUUID` 已全部替换为占位符。**使用前必须生成你自己的三码**（例如用 OpenCore 发行包里的 macserial 针对 iMac18,3 生成），不要沿用任何现成序列号，否则可能与他人冲突、被苹果拉黑。

## 免责声明

本仓库仅供学习与研究使用。macOS 的安装与使用请遵守苹果的许可协议；Hackintosh 在非苹果硬件上运行不受苹果官方支持，操作前请备份重要数据。
