# Lenovo V14-IML (82NA) Hackintosh EFI — OpenCore

联想 V14-IML 黑苹果 OpenCore 引导配置(本人自做、实测完全可用)。

## 电脑配置

| 部件 | 型号 |
|---|---|
| 机型 | Lenovo V14-IML (82NA) 笔记本 |
| CPU | Intel Core i3-10110U (Comet Lake, 2C4T) |
| 核显 | Intel UHD Graphics 620 (GT2) |
| 内存 | 12GB DDR4 2667MHz |
| 硬盘 | 三星 MZALQ256HBJD (PM991) 256GB NVMe |
| 无线 | Intel Wireless-AC 9560 (AirportItlwm 驱动, WiFi 可用) |
| 蓝牙 | Intel (IntelBluetoothFirmware + IntelBTPatcher) |
| 声卡 | Realtek ALC (AppleALC, alcid=11) |
| 触控板 | I2C HID (VoodooI2C + VoodooI2CHID) |
| 键盘 | PS/2 (VoodooPS2Controller) |
| 屏幕 | 1366x768 内置屏 |
| SMBIOS | MacBookPro16,2 |

> ⚠️ 本机无有线网卡 kext(未配置/未使用有线),Kernel/Block 排除了 AppleEthernetRL。

## 系统与 OpenCore

- macOS: Ventura / Sonoma 代系统(原生 AppleIntelCFLGraphicsFramebuffer 可用)
- OpenCore: 较新版本(2025-11-20 构建,自备,非官方 oc107)
- EFI 结构: OC/ 成品版(22 kext / 26 Kernel-Add 条目 / 5 drivers / 4 SSDT) + BOOT/

## 目录结构

```
EFI/
├── BOOT/
│   ├── BOOTx64.efi        # 新版 OpenCore bootstrap(24KB,含 .contentFlavour)
│   ├── .contentFlavour    # OpenCore
│   └── .contentVisibility # Disabled
└── OC/
    ├── config.plist       # 成品配置
    ├── oldConfig.plist    # 留档备份(改前版本)
    ├── OpenCore.efi
    ├── ACPI/              # SSDT-PLUG / SSDT-EC-USBX / SSDT-PNLF / SSDT-AWAC-DISABLE
    ├── Drivers/           # OpenHfsPlus / OpenRuntime / OpenCanopy / ResetNvramEntry / ToggleSipEntry
    ├── Kexts/             # 22 个(见下方清单)
    ├── Tools/             # OpenShell / CleanNvram / ToggleSipEntry
    └── Resources/         # OpenCanopy 图形主题
```

## Kexts 清单(22 个)

**核心**: Lilu 1.7.3, VirtualSMC 1.3.8(+SMCProcessor/SMCSuperIO/SMCBatteryManager/SMCLightSensor),
WhateverGreen 1.7.1, AppleALC 1.9.8, NVMeFix 1.1.4, RestrictEvents 1.1.7, ECEnabler 1.0.6,
USBMap 1.1(定制 18 端口映射), VoodooPS2Controller 2.3.8

**I2C 触控板**: VoodooI2C 2.9.1(含 VoodooInput/VoodooI2CServices/VoodooGPIO), VoodooI2CHID 1.0

**WiFi/蓝牙**: IOSkywalkFamily 1.0, IO80211FamilyLegacy 12.0, AirportItlwm_Ventura 2.3.0,
AMFIPass 1.4.1, BlueToolFixup 2.7.2, IntelBluetoothFirmware 2.5.0, IntelBTPatcher 2.5.0

## 关键配置

- boot-args: `-v amfi=0x80 ipc_control_port_options=0 alcid=11`(极简,生产可用)
- csr-active-config: `ff0f0000`(SIP 全关)
- ScanPolicy: 983299 (0xF0103, 只扫 APFS/SATA/NVMe, 排除 Windows)
- SecureBootModel: Disabled(MacBookPro16,2 为 T2 机型必需)
- HideAuxiliary: True(工具按空格显示)
- LauncherOption: Full(注册固件启动项,直接开机走 OpenCore)
- 核显注入: ig-platform-id `0900a53e` + device-id `9b3e0000` + framebuffer-fbmem/stolenmem/patch-enable 全套补丁(实测可用)
- Kernel/Block: 排除 com.apple.driver.AppleEthernetRL + com.apple.iokit.IOSkywalkFamily

## BIOS 设置(开机按 F2)

1. Security → Secure Boot → **Disabled**
2. Security → Intel Platform Trust Technology (PTT) → **Disabled**(等效 TPM)
3. Startup → UEFI/Legacy Boot → **UEFI Only**
4. (可选) Config → USB → USB Always On 关闭
5. 无需改 CFG Lock(EFI 已开 AppleXcpmCfgLock 软件解锁)

## 使用说明

1. 下载本仓库,把 `EFI` 文件夹复制到 U 盘或内置盘 EFI 分区(ESP)
2. 开机按 F12 选 U 盘(UEFI: USB)引导
3. 装完系统后把 EFI 复制进内置盘 ESP,执行
   `bless --mount /Volumes/EFI --setBoot --file /Volumes/EFI/EFI/BOOT/BOOTx64.efi` 常驻内置盘

## ⚠️ 重要:SMBIOS 已清理,使用前必须自行生成

本仓库 config.plist 的 **序列号/MLB/UUID 已清空为占位符**(`00000000...`),
防止公开复用导致 Apple 服务(如 iMessage/FaceTime)被误锁。

**使用前请用 GenSMBIOS(OpenCore 官方工具)生成你自己的三码**并填入
`PlatformInfo → Generic`:

- SystemSerialNumber
- MLB
- SystemUUID
- ROM(可用网卡 MAC 或随机)

否则 macOS 部分服务(如 iMessage/App Store 登录)不可用。

## 已知要点

- 本配置实测 **Ventura 13** 代系统完全可用(含 WiFi/蓝牙/声卡/亮度/电池/键盘触控板)
- `igfxonln=1` / `-igfxmlr` 参数会导致 AppleIntelCFLGraphicsFramebuffer
  SetupDPTimings 除零 panic —— **不要添加**
- 睡眠: 如异常可 boot-args 加 `hibernatemode=0` 并 pmset 关闭休眠
- 声卡 layout-id 用 11;若换机型按 codec 查 AppleALC 支持列表

## License

仅作个人学习交流使用。OpenCore 与各 kext 版权归原作者所有。
