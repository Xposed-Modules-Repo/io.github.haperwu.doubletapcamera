# 双击电源键直开相机

把 ColorOS / 一加（OnePlus）上 **「双击电源键」的默认目标从「钱包」改成「相机」** 的 LSPosed 模块。

> 实测机型：**一加 Ace 5 Pro（PKR110，ColorOS 16 / Android 16，SDK 36）**

## 它解决什么问题

双击电源键弹出的是「钱包 / NFC 刷卡页」而不是相机 —— 这**不是设置错了**，也不是默认应用、无障碍或手势服务被改写。

根因是 ColorOS 厂商框架主动去问钱包 App「你允许被双击唤起吗」：

```
PhoneWindowManagerExtImpl.oplusInterceptPowerKeyUp   (system_server)
  └─ OplusWalletController.isQuickLaunchSupport / sendCheckWalletMessage
        └─ content://heytap.tas.quickstart.flag.provider.open/CARD_BAG_FLAG
              SUPPORT_QUICK_START=true, HAS_ACTIVE_CARD=true
        └─ OplusWalletController.startWallet(boolean)
              └─ startActivity → com.heytap.tas/...NfcConsumeActivity
```

而**系统设置页写的是另一个键**：`设置 → 连按两次电源按钮` 写的是 AOSP 的
`Settings.Secure double_tap_power_button_gesture`（0=相机 / 1=钱包），厂商框架在自己的两个 vendor dex 里对它 0 命中
⇒ **「设置里明明是相机，双击却开钱包」由此而来**。

本机另外两条相机路径都被封死：AOSP `GestureLauncherService`（ROM 没有 QuickAccessWallet 服务 ⇒ ROM 层关闭）、
ColorOS `OplusQuickAppController`（要求 `persist.vendor.fingerprint.sensor_type == 'side'` ⇒ 光学指纹机型恒 false）。
⇒ **唯一能跑通的目标就是钱包**，而它由钱包 App 自己的开关控制。

## 模块做了什么

在 `system_server` 里 hook `com.android.server.policy.OplusWalletController`：

| hook 点 | 作用 |
| --- | --- |
| `getWalletSwitchState()` | 恒返回 1（否则厂商框架以「switch close」直接放弃钱包分支） |
| `startWallet(boolean)` | 不执行原方法，改为拉起 `com.oplus.camera`（照抄厂商 `OplusQuickAppController.startCamera` 的做法） |

另有两个可选兜底：

- **钱包进程内兜底**（`Instrumentation.callActivityOnResume`）—— 默认关闭，避免把正常打开钱包也改成跳相机；
- **锁屏音量键快门兜底** —— 锁屏时 ColorOS 相机进入「锁屏快捷启动」模式会禁用「音量键 = 快门」，模块在相机进程内对快门计数做自校准，快门没被原生接管时补点一次。

## 锁屏也能直接唤起相机

锁屏（含密码）时相机默认会落在 keyguard 后面。模块在锁屏时会自动切到 secure 入口
（`android.media.action.STILL_IMAGE_CAMERA_SECURE`）并补齐厂商判断「正规快捷启动」所需的两个 intent extra：

- `android_camera_launch_type` = `quick_launch_from_framework`
- `extra_key_caller_package_name` = `com.oplus.gesture`

于是相机日志变成 `mbSecureStartFromFramework: true` → `setShowWhenLocked: true` → **直接盖在锁屏上显示取景界面**。

## 安装

1. 下载并安装 APK（本仓库 Release 里的 `doubletap-camera.apk`）
2. 打开 **LSPosed** → 启用「双击电源键直开相机」→ 作用域勾选 **系统框架 / 系统卡包 / Android 系统**
3. **重启手机**（system_server 注入发生在开机阶段）

> ⚠️ v1.1.0 起包名为 `io.github.haperwu.doubletapcamera`（旧版是 `io.github.doubletapcamera`）。
> 两个包名不同、可并存，请先卸载旧版再装新版。

## 开关（adb）

```bash
adb shell settings put system doubletap_camera_mode <0|1|2|3>
# 0 = 全关        1 = 只在钱包进程拦截
# 2 = 只在 system_server 拦截（默认，推荐）
# 3 = 两者都开

adb shell settings put system doubletap_camera_lock_act 1   # 锁屏时改走「直拍页」CapturePhotoActivity
adb shell settings put system doubletap_camera_secure 2    # 禁用 secure 入口（回退普通入口）
adb shell settings put system doubletap_camera_volkey 0    # 不要音量键快门兜底（默认 1 = 开）
```

## 验证

```bash
adb shell su -c '/data/adb/modules/zygisk_lsposed/lspctl module show io.github.haperwu.doubletapcamera'
#   期望：API: min=100 target=102 / Type: modern / User 0: enabled

adb shell "input keyevent 26 26"      # 双击电源键（必须同一个进程连发两次！）
sleep 3
adb shell "dumpsys activity activities | grep topResumedActivity"
#   期望：topResumedActivity=ActivityRecord{... com.oplus.camera/.Camera}
```

## 要求

- LSPosed 2.x（现代 API，`minApiVersion=100` / `targetApiVersion=102`）+ root
- ColorOS / 一加（实测 ColorOS 16）；其它 ROM 没有「钱包分支」，本模块不适用

## 源码

<https://github.com/HaperWu/doubletapcamera> —— 含完整排查手册（600+ 行：`docs/double-tap-power-wallet-hijack.md`）与模块源码。

## License

MIT
