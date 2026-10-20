# Dell Vostro 5370 Hackintosh EFI

macOS 12.7.6 (Monterey) / OpenCore 引导，SMBIOS 为 MacBookPro15,1。

## 硬件规格

| 部件 | 型号 |
|---|---|
| CPU | Intel Core i5-8250U (Kaby Lake-R) |
| 核显 | Intel UHD Graphics 620 |
| 无线+蓝牙 | BCM94352Z (DW1550, PCI 14e4:43b1 / BT 0489:e07a) |
| 声卡 | Realtek (AppleALC layout-id 13) |
| 触控板 | I2C HID (VoodooI2C + GPI0/TPD0 SSDT) |
| 键盘 | PS2 (VoodooPS2Controller + SSDT-PS2K 自定义映射) |
| 读卡器 | USB 内置 (GenericCardReaderFriend) |

## Kexts（Kernel → Add 顺序即加载顺序）

| Kext | 版本 | 用途 |
|---|---|---|
| Lilu | 1.6.9 | 补丁引擎 |
| VirtualSMC | — | SMC 仿真 |
| WhateverGreen | — | 核显补丁 |
| AppleALC | — | 声卡（layout-id 13） |
| AirportBrcmFixup (+ AirPortBrcmNIC_Injector) | 2.2.0 | Wi-Fi 驱动（AirPortBrcm4360_Injector 已禁用） |
| BrcmFirmwareData → BrcmPatchRAM3 → BlueToolFixup | 2.7.2 | 蓝牙固件上传 + Monterey 用户态蓝牙栈修复 |
| SMCBatteryManager / SMCDellSensors / SMCProcessor | — | VirtualSMC 插件 |
| GenericCardReaderFriend | — | 读卡器识别 |
| VoodooI2C + VoodooI2CHID | — | 触控板 |
| VoodooPS2Controller | — | 键盘 |

## ACPI（SSDT）

`SSDT-3-xh_OEMBD`（USB 定制）、`SSDT-EC-USBX`、`SSDT-GPI0`/`SSDT-TPD0`（I2C 触控板）、`SSDT-GPRW`（睡眠修复，配合 GPRW→XPRW rename）、`SSDT-HPET-DISABLE`、`SSDT-PLUG`（XCPM）、`SSDT-PNLF`（背光）、`SSDT-PS2K`（键盘映射）、`SSDT-SBUS-MCHC`。

## 正常工作

Wi-Fi、蓝牙（含音频设备）、AirDrop 接收（iPhone→Mac）、核显加速、亮度调节与持久化、睡眠/唤醒、电池、触控板手势、键盘、声卡、读卡器。

## 已知问题与重要结论（不要再回头踩坑）

### 1. AirDrop Mac→iPhone 发送失败 —— 硬件固有限制，软件无解

- **症状**：能搜到 iPhone，点击发送无反应、面板静默关闭。
- **根因**：sharingd 日志 `foundRapport: true, foundBonjour: false` —— iPhone 只通过 BLE 被发现，从未通过 AWDL 上的 Bonjour (`_airdrop._tcp`) 被发现。82 秒窗口内 awdl0 上零 mDNS 记录，AWDL peer 已同步但 iPhone 不应答。
- **结论**：BCM94352 单向 Continuity 缺陷是**固件 + 旧 AirPort_BrcmNIC 驱动的固有限制**（acidanthera bugtracker #1821，2021 年至今未解）。已排除：brcmfx-country（CN/US 均无效）、0x43A0 设备伪装、BrcmBluetoothInjector、iPhone 侧设置。
- **唯一稳定解**：换 BCM94360NG（M.2 2230 A+E key 直插、免驱、AirDrop 双向正常）。

### 2. 蓝牙不要加 BrcmBluetoothInjector.kext

该 kext 仅适用 macOS 10.11–11。Monterey 的组合是 **BrcmFirmwareData + BrcmPatchRAM3 + BlueToolFixup**（BlueToolFixup 要求 macOS 12+，MinKernel 20.0.0）。误加 Injector 会导致蓝牙完全失效。另注：Monterey 上 `system_profiler SPBluetoothDataType` 显示 controllerInfo == nil 是该卡型的正常表现，不代表蓝牙故障。

### 3. 亮度调节

DeviceProperties 中**不能**硬编码 `AAPL,backlight-control` / `@0,backlight-control` / `applbkl`（会导致重启后亮度锁死），也**不能**加 `enable-backlight-registers-fix`（会导致调节失效）。只保留 `enable-backlight-smoother` 及其参数。

### 4. 睡眠设置（系统侧）

`sudo pmset hibernatemode 0 standby 0 autopoweroff 0 tcpkeepalive 0` —— 关深度休眠，防止 USB 断电丢蓝牙固件。`proximitywake` 已设为 0（排查 SSV 弹窗遗留，保留无妨）。

### 5. "宗卷哈希值不匹配" 弹窗

`Misc → Security → SecureBootModel = Disabled` 后短睡眠唤醒不再弹窗；隔夜长睡眠偶发弹窗，与 SSV/APFS seal 校验有关，未完全解决，不影响使用。

### 6. boot-args 当前为空

历史上所有调试参数（brcmfx-country、brcmfx-aspm 等）均已清除，无线网卡以原生参数运行。

## BIOS 设置与 GRUB 变量

### BIOS 界面设置

- Secure Boot → **Disabled**
- Fastboot → **Thorough**
- UEFI Boot Path Security → **Never**

### setup_var（通过 modGRUBShell.efi，已在 OC/Tools 中）

```
setup_var 0x527 0x00
setup_var 0x7E8 0x02
setup_var 0x7E9 0x03
setup_var 0x5DC 0x00
setup_var 0xF7A 0x00
reboot
```

## 系统侧配套

- **EFI 自动挂载**：`/Library/LaunchDaemons/com.ldz.mountefi.plist`（disk0s1, nobrowse，免密码）
- **FileVault 已关闭**（用于自动登录）
- SIP 部分关闭（csr-active-config = AAAAAA==）

## 备份文件

`config.plist.bak-brightness`（亮度修复前）、`config.plist.bak-43A0`（Wi-Fi 伪装实验）、`config.plist.bak-btinjector`（蓝牙注入器误加前，**当前基线**）。
