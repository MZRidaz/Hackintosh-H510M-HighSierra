# 黑苹果安装指南：macOS High Sierra 10.13.6

适用硬件（你的机器）：
- CPU：Intel i7-10700F（十代，无核显）
- 主板：MSI H510M BOMBER (MS-7D22)
- 显卡：NVIDIA GTX 1050 Ti
- 硬盘：WD 500GB 机械盘（与 Windows 同盘双系统）
- 内存：32GB

配套文件：`EFI-HighSierra-v2.zip`（我已针对这台机器配好的 OpenCore 引导，含 CPU 伪装、
声卡 ALC897、Intel I219-V 网卡、USB 驱动，另含显卡补丁 WhateverGreen 和
SSDT-EC-USBX。配置已通过官方校验工具 ocvalidate 检查，无问题）

---

## 先说清楚预期

- 这是 2017 年的系统装在 2020 年的硬件上，靠「CPU 伪装成七代酷睿」运行，
  目标是装上、能正常日常使用。
- 大概率不完美的地方：睡眠可能不好用（文末有关闭方法）、隔空投送/接力没有、
  iMessage 不保证能激活。
- **装好之后先升到顶、再停住**：系统装完先不要装显卡驱动，先把安全更新升到
  **2020-006**（系统变成 17G14042，见第 5 步），装好显卡驱动之后**再也不要点
  系统更新**（版本号一变，NVIDIA 显卡驱动就不匹配，会黑屏）。App Store 里记得
  关掉自动更新。
- 机械硬盘会慢：开机和安装都要有耐心，全程预留 2 小时左右。

## 准备清单

- 一个 ≥16GB 的 U 盘（会被格式化，提前备份里面东西）
- 有线键盘、鼠标（安装阶段别用蓝牙和无线接收器不稳的设备）
- **重要数据先备份**：下面要动 Windows 的分区，正常操作不会丢数据，但先备份
- 本指南 + EFI-HighSierra-v2.zip

---

## 第 0 步：确认 Windows 磁盘是 GPT 格式

1. Win + R 输入 `diskpart` 回车
2. 输入 `list disk`，看你的 500GB 硬盘那一行，「GPT」列有没有 `*`
3. 有 `*` → 继续。没有（是 MBR）→ **先停下，把这个情况告诉我**，不要继续

输入 `exit` 退出。

## 第 1 步：在 Windows 里给 macOS 腾地方

1. Win + X → 磁盘管理
2. 右键一个空间富余的分区（比如 C 盘或数据盘）→ 压缩卷 → 压缩出至少
   **120000 MB**（约 120GB，越多越好）
3. 压缩完会出现一块「未分配」→ 右键它 → 新建简单卷 → 一路下一步，
   文件系统选 **exFAT**，卷标填 **MACOS**（这一步只是占位，方便安装时认出它，
   安装时会把它抹掉重格）

## 第 2 步：制作安装 U 盘（在 Windows 上）

> 注：老办法用的 MakeInstall 工具已被作者删除（仓库 404），下面是 OpenCore
> 官方指南现在推荐的做法：U 盘上只放两个文件夹，一个恢复镜像、一个我配的
> EFI。安装时系统会现场从苹果服务器下载，所以**安装全程要插网线**。

1. 安装 Python：打开 https://www.python.org/downloads/ 下载安装，
   **安装时务必勾选 “Add python.exe to PATH”**
2. 下载 High Sierra 的恢复镜像：
   - 下载 OpenCore 官方包：https://github.com/acidanthera/OpenCorePkg/releases
     选一个 RELEASE 版 zip（比如 0.9.8），解压
   - 进入解压出的 `Utilities\macrecovery` 文件夹，在文件夹路径栏输入 `cmd`
     回车，打开命令行窗口
   - 在命令行里粘贴运行这一整行：
     `python macrecovery.py -b Mac-BE088AF1C5EB4FA2 -m 00000000000J80300 download`
   - 跑完后，这个文件夹里会出现 **BaseSystem.dmg** 和 **BaseSystem.chunklist**
     两个文件（约 500MB，很快）。这两个文件留好，下一步要用
3. 准备 U 盘：
   - 插上 U 盘（会被清空），在「磁盘管理」里把 U 盘格式化成一个 **FAT32**
     分区；如果 U 盘大于 16GB、Windows 不给选 FAT32，就用 Rufus
     （https://rufus.ie）：Boot selection 选 Non bootable，文件系统选
     Large FAT32，点 Start
4. 往 U 盘里放两个文件夹（都在 U 盘根目录，并排）：
   - 新建文件夹，命名为 **com.apple.recovery.boot**，把第 2 步的
     BaseSystem.dmg 和 BaseSystem.chunklist 两个文件放进去
   - 解压 `EFI-HighSierra-v2.zip`，把解压出的 **EFI** 文件夹放进 U 盘根目录
   - 完成标志：U 盘根目录就这两样 — `EFI` 文件夹和
     `com.apple.recovery.boot` 文件夹

## 第 3 步：BIOS 设置

开机按 **Del** 进 BIOS，按 F7 切到高级模式（MSI Click BIOS 5）：

| 设置项 | 位置（大致） | 改成 |
|---|---|---|
| Secure Boot | Settings → Security → Secure Boot | **Disabled** |
| Fast Boot | Settings → Advanced → Windows OS Configuration | **Disabled** |
| MSI Fast Boot | 同上附近 | **Disabled** |
| SATA Mode | Settings → Advanced → Integrated Peripherals | **AHCI** |
| XHCI Hand-off | Settings → Advanced → USB Configuration | **Enabled** |
| Initiate Graphic Adapter | Settings → Advanced → Integrated Peripherals / Graphics | **PEG**（独显） |
| BIOS UEFI/CSM Mode | Settings → Advanced → Windows OS Configuration | **UEFI** |
| CFG Lock | OC → CPU Features（如有） | Disabled（找不到就跳过，配置里已绕过） |

F10 保存退出。显示器线确认插在**显卡**上（不是主板接口）。

## 第 4 步：安装系统

1. 插上 U 盘，开机连续按 **F11**，选带 **UEFI** 前缀的 U 盘启动
2. 进入 OpenCore 菜单（黑底白字列表），选带 **BaseSystem / Recovery**
   字样的那一项（这是恢复安装入口）
   - 之后会滚很多代码，这是我故意开的详细日志模式，正常现象
   - 进入恢复界面可能要等一会儿，它要先加载恢复系统
3. 进入安装界面后，先打开顶部菜单的「实用工具 → 磁盘工具」：
   - 菜单「显示 → 显示所有设备」
   - 找到第 1 步建的那个 **MACOS** 卷（exFAT 格式那个），选中它点「抹掉」：
     名称 `Macintosh HD`，格式 **Mac OS Extended（日志式）**，方案 **GUID 分区图**
   - ⚠️ 只抹这一个卷！NTFS 的是 Windows，千万别碰，也不要抹整块物理硬盘
4. 退出磁盘工具 → 「重新安装 macOS」→ 选 Macintosh HD → 继续
   - 这一步会先联网从苹果下载完整系统（约 5GB），网线别拔
   - 下载加安装在机械盘上大约 40–60 分钟，中途会自动重启
5. 每次重启都按 F11 从 U 盘进 OpenCore 菜单：
   - 菜单里出现 **Boot macOS Install from Macintosh HD** 就选它（继续第二阶段），
     可能要重复 1–2 次
   - 最后选 **Macintosh HD** 就是进系统了
6. 初始设置：地区选中国，键盘默认，**不要登录 Apple ID**（选稍后设置），
   创建本地用户名密码，一路继续到桌面

## 第 5 步：先升系统到 17G14042，再装 NVIDIA 显卡驱动（关键一步）

⚠️ 之前版本的指南这里写错了：**不要在 17G65/17G66 上装 40.113 驱动**
（40.113 对应的是 17G4015，且旧版驱动的签名大多已过期，装不上）。
唯一能正常装的，是 NVIDIA 重新签署过的最后一版驱动 **40.140**，它只对应
最终系统版本 **17G14042**。所以顺序必须是：先升系统，再装驱动。

刚进桌面时分辨率低、界面卡顿是正常的，因为显卡驱动还没装。

1. 先升系统：打开 App Store → 更新，把 **Security Update 2020-005**
   装上，重启；再把 **Security Update 2020-006** 装上，重启
   - 升完确认版本：左上角  → 关于本机，应显示 **macOS High Sierra
     10.13.6 (17G14042)**
   - 升的过程中每次重启都按 F11 从 U 盘进 OpenCore 菜单，选 Macintosh HD
2. 确认是 17G14042 后，再下载并安装显卡驱动：
   https://us.download.nvidia.com/Mac/Quadro_Certified/387.10.10.10.40.140/WebDriver-387.10.10.10.40.140.pkg
   - 安装时若提示被安全策略拦截：系统偏好设置 → 安全性与隐私 → 点「仍要打开/允许」
3. 装完重启（还是从 U 盘进 OpenCore → 选 Macintosh HD）
4. 成功标志：分辨率正常、菜单栏右上角出现 NVIDIA 图标、
   关于本机里显卡显示为 NVIDIA GeForce GTX 1050 Ti
5. **到这一步之后就停住**：不要再点任何系统更新，App Store 里关掉自动更新
   （第 7 步有具体位置）

## 第 6 步：把引导装进硬盘（以后不用插 U 盘）

1. 在 macOS 里打开「终端」，输入 `diskutil list` 找到硬盘的 EFI 分区编号
   （一般是 disk0s1），挂载它：`sudo diskutil mount disk0s1`
   （编号以你实际为准，别照抄）
2. 打开挂载好的 EFI 分区，把 U 盘 EFI 里的 `EFI/OC` 文件夹和
   `EFI/BOOT/BOOTX64.efi` 复制过去
   - ⚠️ 这个 EFI 分区同时是 Windows 的引导分区：里面的 **Microsoft 文件夹
     绝对不要删**
   - 复制 BOOTX64.efi 前，先把原有的同名文件改名成 `BOOTX64.efi.bak` 备份
3. 重启进 BIOS，把 Boot Option #1 设为这块硬盘（OpenCore 项），
   或开机 F11 选硬盘。OpenCore 菜单里同样能选 Windows 进原系统

## 第 7 步：收尾设置

- **关自动更新**：系统偏好设置 → App Store → 取消勾选所有自动更新项
- **关睡眠**（十代平台在老系统上睡眠大概率有问题）：系统偏好设置 → 节能 →
  「电脑睡眠」拉到永不
- **Windows 时间错乱修复**（双系统通病）：回 Windows 后如果时间差了 8 小时，
  管理员身份打开 CMD 执行：
  `reg add "HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /t REG_QWORD /d 1 /f`

## 故障速查（先拍照发我再处理）

| 现象 | 可能原因/处理 |
|---|---|
| OpenCore 菜单里没有 BaseSystem/Recovery 项 | U 盘根目录两个文件夹没放对，回第 2 步第 4 点检查（EFI 和 com.apple.recovery.boot 都要在根目录） |
| 卡在 ++++++++++ 一行不动 | 内存引导问题，把卡住的照片发我，我改配置 |
| 滚代码后黑屏/重启 | 多半是显卡或 USB，拍下最后画面发我 |
| 安装界面没鼠标键盘 | 换主板后面板的 USB 2.0 黑色接口；还不行试 PS/2 口键盘（配置里带了 PS/2 驱动） |
| 装完驱动重启黑屏 | 驱动版本不对或没装上：进 Windows 把 EFI 里 config.plist 的 boot-args 末尾加 ` nv_disable=1`，能进桌面后重装驱动，再删掉这个参数 |
| 没声音 | 改 config.plist 里 boot-args 的 `alcid=11`，依次试 66、13、23、69（改完重启生效） |
| 没网络 | 确认网线插好；还不行把现象告诉我 |
| 进不了 Windows | OpenCore 菜单里选 Windows 项；或 BIOS 里选 Windows Boot Manager，都能回去，Windows 本身没被动过 |

---

配置说明（备查）：OpenCore 0.9.8；机型伪装 iMac18,3；CPU 伪装 Kaby Lake
（Cpuid1Data = 906E9）；kexts：Lilu 1.5.4 / WhateverGreen 1.5.6 /
VirtualSMC 1.2.5 / AppleALC 1.6.3（ALC897，layout 11）/ IntelMausi 1.0.7 /
USBInjectAll 0.7.6 / VoodooPS2Controller 2.2.8；SSDT-EC-USBX + SSDT-AWAC +
SSDT-PLUG 已编译内置；Booter Quirks 按 Comet Lake 要求设为
SetupVirtualMap=NO、ProtectUefiServices=YES，安装阶段 XhciPortLimit=YES
（装好做完 USB 定制后可再关回）。
