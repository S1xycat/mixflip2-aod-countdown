# Stage 01 — MIX Flip 2 外屏 AOD 环境调查

**阶段**：Stage 1
**日期**：2026-10-22
**设备**：Xiaomi 2505APX7BC（bixi）/ HyperOS OS3.0.304.0.WOHCNXM / Android 16
**执行者**：DSHA Agent（DeepSeek Harness）
**阶段性质**：只读调查，不改系统、不装模块、不写应用

---

## A. 本阶段目标

在动手开发前，先把「MIX Flip 2 外屏 AOD 到底是怎么实现的」查清楚，为后续开发提供可靠靶点。具体要回答：

1. 本机的准确系统版本与开发环境（Root / LSPosed / 工具链）是否满足开发条件？
2. `com.miui.aod`、`com.miui.fliphome`、`com.android.systemui`、`com.android.deskclock` 四个组件的实际路径、版本、职责是什么？
3. 外屏 AOD 的绘制宿主是谁？由谁负责？
4. 系统时钟的外屏倒计时为什么能保持常亮？它是否走了 AOD？
5. 外屏与内屏是否使用不同的显示控制流程？
6. 已有开源实现 Hook 了哪些类和方法？对当前 HyperOS 3 是否适配？

**本阶段不涉及**：修改系统 APK、修改系统配置、安装 LSPosed 模块、替换 AOD、修改 CPU/屏幕电源状态、创建倒计时应用。

---

## B. 已完成工作

以下均为**实际执行完成**的工作，非计划：

1. ✅ 采集设备基础属性（型号、代号、系统版本、Android 版本、构建时间）
2. ✅ 验证 Root 可用性（KernelSU，`u:r:ksu:s0`）
3. ✅ 检查 LSPosed 安装与运行状态（`lspd` 常驻进程 + 4 个 zygisk companion 进程）
4. ✅ 检查容器内开发工具链（Java / Gradle / Android SDK / jadx / adb）
5. ✅ 提取四个目标组件的 APK 路径、versionCode、versionName、sharedUser、flags
6. ✅ 完整导出 `com.miui.aod` 的 `pm dump`（262 KB，组件表证据）
7. ✅ 确认 `com.miui.aod` 与 `com.android.systemui` 同 UID
8. ✅ 定位 AOD 插件入口 `DozeServicePluginImpl` 与注册 Action `PLUGIN_DOZE`
9. ✅ 在框架层验证 7 个关键符号的存在性
10. ✅ 采集 `dumpsys power` 与 `dumpsys display` 完整输出，确认双屏 displayId 映射
11. ✅ 采集 `ps -A` 确认 AOD 双进程常驻
12. ✅ 克隆并阅读参考项目 `FlipOuterUnlock_262` 的 `AodHook.kt`（405 行）、`Config.kt`、`README.md`、`module.prop`、`scope.list`
13. ✅ 记录 dex 字符串 grep 的失败结果（负结果，证明必须反编译）
14. ✅ 清理设备侧临时副本

**未完成**：由于本机无 GitHub 认证，仓库创建与推送未完成（详见 G 节）。

---

## C. 执行命令

以下为关键命令及实际结果。完整输出见 `logs/stage-01-raw/`。

### C.1 设备基础

```bash
$ adb-shell getprop ro.product.model
2505APX7BC

$ adb-shell getprop ro.product.device
bixi

$ adb-shell getprop ro.mi.os.version.incremental
OS3.0.304.0.WOHCNXM

$ adb-shell getprop ro.build.version.release
16

$ adb-shell getprop ro.build.date
Thu Jun 25 20:24:38 CST 2026
```

### C.2 Root 与框架

```bash
$ adb-shell id
uid=0(root) gid=0(root) groups=0(root),1077(aid_1077),3003(inet),9997(everybody) context=u:r:ksu:s0

$ adb-shell "ls /data/adb/modules/"
AdGuardHome
device_faker
h_reboot
playintegrityfix
zygisk_lsposed
zygisksu

$ adb-shell "ps -A" | grep -i -E "xposed|lsp|magisk|zygisk|aod"
root      3707     1 ... zn-zygisk-companion64 zygisk_lsposed
root      5400     1 ... zn-zygisk-companion64 zygisk_lsposed
root      5702     1 ... lspd
root      6163     1 ... zn-zygisk-companion64 zygisk_lsposed
root      8414     1 ... zn-zygisk-companion64 zygisk_lsposed
system   26453  5870 ... com.miui.aod
root     26659     1 ... zn-zygisk-companion64 zygisk_lsposed
system   30821  5870 ... com.miui.aod:keyguardeditor
```

> 结论：LSPosed **确实在运行**，不是仅安装未启用。

### C.3 开发工具链

```bash
$ java -version
openjdk version "21.0.12.1" 2026-08-18
OpenJDK Runtime Environment (build 21.0.12.1+1-1-24.04.4-Ubuntu)

$ gradle -v
bash: gradle: command not found

$ which jadx apktool d2j-dex2jar
（无输出）

$ ls /root/Android/Sdk
（不存在）
```

> 结论：Java 可用，Gradle / SDK / jadx 全部缺失。

### C.4 组件路径与版本

```bash
$ adb-shell pm path com.miui.aod
package:/product/priv-app/MIUIAod/MIUIAod.apk

$ adb-shell pm path com.miui.fliphome
package:/product/priv-app/FlipHome/FlipHome.apk

$ adb-shell pm path com.android.systemui
package:/system_ext/priv-app/MiuiSystemUI/MiuiSystemUI.apk

$ adb-shell pm path com.android.deskclock
package:/product/data-app/MIUIDeskClock/MIUIDeskClock.apk
```

```bash
$ adb-shell dumpsys package com.miui.aod | grep -E "versionName|versionCode|codePath|SharedUser"
    sharedUser=SharedUserSetting{89065f7 android.uid.system/1000}
    codePath=/product/priv-app/MIUIAod
    versionCode=22335121 minSdk=33 targetSdk=35
    versionName=DEV-FLIP-2335.1.2.1-05070145
    flags=[ SYSTEM HAS_CODE ALLOW_CLEAR_USER_DATA ALLOW_BACKUP KILL_AFTER_RESTORE ]
```

```bash
$ adb-shell dumpsys package com.miui.fliphome | grep -E "versionName|versionCode|codePath"
    codePath=/product/priv-app/FlipHome
    versionCode=101041105 minSdk=34 targetSdk=35
    versionName=ALPHA-1.1.4.1105-06162009

$ adb-shell dumpsys package com.android.systemui | grep -E "versionName|versionCode|codePath|SharedUser"
    sharedUser=SharedUserSetting{89065f7 android.uid.system/1000}
    codePath=/system_ext/priv-app/MiuiSystemUI
    versionCode=202501210 minSdk=36 targetSdk=36
    versionName=16.03.251211.r

$ adb-shell dumpsys package com.android.deskclock | grep -E "versionName|versionCode|codePath"
    codePath=/product/data-app/MIUIDeskClock
    versionCode=1305006100 minSdk=26 targetSdk=35
    versionName=17.61.0
```

### C.5 AOD 插件入口定位（核心证据）

```bash
$ adb-shell dumpsys package com.miui.aod | grep -A3 "Service Resolver Table"
Service Resolver Table:
  Non-Data Actions:
      com.android.systemui.action.PLUGIN_DOZE:
        6caa39 com.miui.aod/.doze.DozeServicePluginImpl filter b38087e
```

> **这是本阶段最关键的一条证据**：AOD 通过 SystemUI 插件机制加载。

### C.6 框架符号验证

```bash
$ adb-shell "grep -a -o -m1 updateRearDozeSettings /system/framework/services.jar"
updateRearDozeSettings

$ adb-shell "grep -a -o -m1 handleRearSandman /system/framework/services.jar"
handleRearSandman

$ adb-shell "grep -a -o -m1 mRearAlwaysOnEnabled /system/framework/services.jar"
mRearAlwaysOnEnabled

$ adb-shell "grep -a -o -m1 shouldDeviceBeSleep /system/framework/services.jar"
shouldDeviceBeSleep

$ adb-shell "grep -a -o -m1 DreamController /system/framework/services.jar"
DreamController

$ adb-shell "grep -a -o -m1 setDozeScreenState /system/framework/framework.jar"
setDozeScreenState

$ adb-shell "grep -a -o -m1 PowerManagerService /system/framework/services.jar"
PowerManagerService
```

> 7 项全部命中。注意：`MiuiFlipPolicy` 未命中（返回空）。

### C.7 显示与电源状态

```bash
$ adb-shell dumpsys display | grep -E "mDisplayId|state=" 
# 外屏 displayId=0，1208x1392，state OFF / committedState ON
# 内屏 displayId=1，1224x2912，state OFF / committedState OFF

$ adb-shell dumpsys power | grep -i "Doze\|Wakefulness"
  mWakefulness=Awake
  mDozeAfterScreenOff=true
  mDozeScreenStateOverrideFromDreamManager=1
  mDozeScreenBrightnessOverrideFromDreamManager=255
  mUseNormalBrightnessForDoze=false
```

### C.8 dex 字符串验证（失败，负结果）

```bash
$ adb-shell "grep -a -o -m1 DozeMachine /data/local/tmp/aod.apk"
（无输出，exit 1）

$ adb-shell "grep -a -o -m1 com/miui/aod/doze/DozeMachine /product/priv-app/MIUIAod/MIUIAod.apk"
（无输出，exit 1）

$ adb-shell "grep -a -o -m1 FlipLinkageStyleController /data/local/tmp/aod.apk"
（无输出，exit 1）
```

> **负结果同样有价值**：证明 AOD 的 dex 字符串经过压缩/加密，无法用 grep 验证类名，**必须反编译**。这直接决定了 Stage 2 的工作项。

### C.9 设备策略限制

```bash
$ adb-shell dumpsys SurfaceFlinger
[POLICY_BLOCKED] 不认识的命令不执行

$ adb-shell dumpsys dreams
[POLICY_BLOCKED] 未识别的 dumpsys 参数；禁止用诊断命令改设备状态

$ adb-shell dumpsys activity services com.miui.aod
[POLICY_BLOCKED] 未识别的 dumpsys 参数
```

> 这些命令被设备 shell 策略拦截，属于环境限制而非项目问题。

### C.10 GitHub 能力检查（本阶段新增）

```bash
$ which git gh ssh
/usr/bin/git
（gh、ssh 未找到）

$ ls /root/.ssh/
（空目录，无密钥）

$ ls /root/.config/gh/
ls: cannot access '/root/.config/gh/': No such file or directory

$ env | grep -i -E "^(GH_|GITHUB_)"
（无输出）

$ git config --global --list
fatal: unable to read config file '/root/.gitconfig': No such file or directory

$ curl -s -o /dev/null -w "%{http_code}" https://api.github.com
200   # 网络可达

$ curl -s https://api.github.com/repos/237389749/mixflip2-aod-countdown
{"message": "Not Found", ...}   # 404

$ curl -s "https://api.github.com/search/repositories?q=mixflip2-aod-countdown"
total: 0   # 全站无同名仓库
```

> 结论：**网络通，但无任何认证凭据**。无法创建仓库，无法推送。

---

## D. 技术发现

每项按可靠性标注。

### D.1 AOD 是 SystemUI 的插件，不是独立应用

- **状态：已实机验证**
- 证据：`pm dump com.miui.aod` 的 Service Resolver Table

```
com.android.systemui.action.PLUGIN_DOZE:
    com.miui.aod/.doze.DozeServicePluginImpl
```

- 并且 `com.miui.aod` 与 `com.android.systemui` 的 `sharedUser` 均为 `SharedUserSetting{89065f7 android.uid.system/1000}`。
- **含义**：AOD 的绘制发生在 SystemUI 进程内，`com.miui.aod` 只是被加载的插件代码。第三方应用无法「注册」进 AOD，因为这不是公开扩展点。

### D.2 外屏 displayId = 0，内屏 displayId = 1

- **状态：已实机验证**
- 证据：`dumpsys display` 的 `DisplayInfo`

| 屏幕 | displayId | 分辨率 | 刘海 | installOrientation |
|---|---|---|---|---|
| 外屏 | **0** | 1208×1392 | `Rect(810,664-1208,1392)` | ROTATION_180 |
| 内屏 | **1** | 1224×2912 | `Rect(569,0-655,146)` | ROTATION_0 |

- **含义**：与「主屏=0、副屏=1」的常识相反。后续所有涉及外屏的 Hook 都必须用 displayId=0，这是最容易踩的坑。

### D.3 框架层外屏 Doze 符号存在

- **状态：已实机验证**（符号存在性）
- 命中清单：

| 符号 | 所在文件 | 推测含义 |
|---|---|---|
| `PowerManagerService` | services.jar | 电源管理服务 |
| `updateRearDozeSettings` | services.jar | 设置外屏 doze 参数 |
| `handleRearSandman` | services.jar | 外屏沙人（休眠/AOD 入口） |
| `mRearAlwaysOnEnabled` | services.jar | 外屏 AOD 总开关字段 |
| `shouldDeviceBeSleep` | services.jar | 显示层睡眠判定 |
| `DreamController` | services.jar | dream 控制器 |
| `setDozeScreenState` | framework.jar | 设置 doze 屏幕态 |

- **未命中**：`MiuiFlipPolicy`（grep 返回空，可能类名不同或已被混淆）。
- **注意**：符号存在 ≠ 方法签名正确。精确签名需反编译确认（见 D.7）。

### D.4 AOD 包版本名带 `DEV-FLIP`

- **状态：已实机验证**
- 证据：`versionName=DEV-FLIP-2335.1.2.1-05070145`
- **含义**：该 AOD 包是折叠屏专用编译版，与项目的折叠外屏 AOD 目标一致，是个好信号。

### D.5 时钟应用无外屏专用 Activity

- **状态：已实机验证**
- 证据：`dumpsys package com.android.deskclock` 的 Activity 表中，全部组件为：
  - `DeskClockTabActivity`、`TimezoneSearchActivity`、`WorldClockEditActivity`、`SettingsActivity`、`LifePostSettingActivity`、`HandleSetAlarmActivity`、`ShortcutTrampolineActivity`、`HandleActivityIntent`、`MyAiActionActivity`、`AlarmRingtonePickerActivity`
  - 用 `grep -i "flip|cover|outer|rear"` 搜索：**无任何命中**
- **含义**：时钟的外屏「只有闹钟和倒计时」是 fliphome 在 Launcher 层做的分组过滤，不是应用自身的多形态适配。倒计时仍运行通用 Activity。

### D.6 时钟倒计时「常亮」不走 AOD

- **状态：推测**（部分有代码证据支持）
- 推理链：
  1. 时钟无 AOD 相关组件（D.5 已验证）
  2. AOD 由 SystemUI 插件独占（D.1 已验证）
  3. `com.miui.fliphome/.FlipLauncher` 处于 `mLastPausedActivity`，说明外屏桌面在承载内容
  4. 参考项目注释表明「Flip1 上外屏需主动 hook 才能启用 AOD」→ 反推外屏默认不具备 AOD
- **结论**：倒计时常亮极可能是「外屏保持 ON + 禁止睡眠」的普通亮屏，而非 DOZE_AOD 低功耗态。这解释了它为什么费电。

### D.7 外屏独立 power group（groupId=1）

- **状态：推测**（符号已验证，行为未验证）
- 依据：参考项目 `AodHook.kt` 源码注释 + `updateRearDozeSettings` 等方法签名暗示存在 `groupId` 参数
- **本机实测**：`dumpsys power` 当前**只显示 `powerGroupId: 0`**，未见 groupId=1；`dumpsys display` 两屏 `displayGroupId` 均为 0
- **推测原因**：groupId=1 仅在外屏 AOD 激活时动态出现，当前外屏 `state OFF` 故不可见
- **验证方法**：折叠状态下激活外屏 AOD 后复测 `dumpsys power`

### D.8 DRM 双连接器独立

- **状态：已实机验证**
- 证据：

```
$ adb-shell ls /sys/class/drm/
card0                    card0-DSI-1        card0-DSI-2
card0-Virtual-1          card0-Virtual-2
card0-sde-crtc-0 ... card0-sde-crtc-7
renderD128               sde-conn-2-DSI-1   sde-conn-3-DSI-2
```

- 两个连接器的 sysfs 节点均为标准集合（`connector_id/dpms/edid/enabled/modes/power/status`），**无法**直接控制自刷新。
- **含义**：内外屏共用一个 `card0` DPU，外屏是第二个 DSI 连接器。第三方无法通过 sysfs 让外屏进入低功耗自刷新。

### D.9 参考项目的 Hook 入口清单

- **状态：有代码证据，尚未在本机实机验证**
- 来源：`FlipOuterUnlock_262` 的 `AodHook.kt`（405 行）

**框架侧（system_server）**

| # | 目标类.方法 | 动作 | 风险 |
|---|---|---|---|
| 1 | `com.android.server.power.PowerManagerService.updateRearDozeSettings(int, boolean, boolean)` | groupId==1 时强制 `(1, true, true)` | 参数签名需核对 |
| 2 | `com.android.server.dreams.DreamController.stopDream(boolean, String)` | 拦截 groupId==1 的 slow-to-connect/finish 超时 | 反射读 `mGroupId` 字段 |

**应用侧（SystemUI / com.miui.aod）**

| # | 目标 | 动作 | 风险 |
|---|---|---|---|
| 5 | `android.view.Display.getCutout()` | 仅 AOD 调用栈返回空 cutout | 修 `DisplayUtils.getCutoutPosition()` NPE |
| 3 | `android.service.dreams.DreamService.setDozeScreenState(int)` | `{0,1,3}`→4 (DOZE_AOD) | 状态语义需核对 |
| 4 | `DreamService.onDreamingStarted()` | 触发 Layer2 | 一次性 |
| L2-a | `com.miui.aod.doze.DozeMachine.requestState(State)` | DOZE/DOZE_SUSPEND/FINISH → DOZE_AOD | **类名未验证** |
| L2-b | `com.miui.aod.doze.DozeService.setDozeScreenState(int)` | `{0,1,3}`→4 | **类名未验证** |
| L2-c | `com.miui.aod.DozeHost.isFullAod()` | → false | **类名未验证** |
| L2-d | `com.miui.aod.flip.FlipLinkageStyleController.isFlipped()/isUsingFlip(Context)` | → false / true | **类名未验证** |

- `targetPackages = listOf("android", "com.android.systemui", "com.miui.aod")`
- LSPosed scope：`system`、`com.android.systemui`、`com.miui.aod`、`com.android.camera`、`com.miui.fliphome`、`com.sohu.inputmethod.sogou.xiaomi`、`com.miui.home`

### D.10 参考项目的适配边界

- **状态：已实机验证**（README 原文）
- 原话：**「flip1（ruyi）专用基版，活跃开发。flip2 问题走 FlipOuterUnlock2（基线 90833c4）」**
- **含义**：该仓库**不是**为 Flip2 / HyperOS 3 准备的。直接套用风险高。

### D.11 参考项目不解决「倒计时进 AOD」

- **状态：有代码证据**
- 该模块做的是「**让外屏能进入 AOD 状态**」，即折叠时外屏不再直接黑屏。
- 但 AOD 上显示什么，仍由 `com.miui.aod` 的模板决定（`constant_lockscreen_info` 里的 `templateId: classic_plus`）。
- **结论**：GPT 说的「可以做到」，若指「外屏能开 AOD」→ 成立；若指「倒计时圆盘进原生 AOD」→ **该模块不解决，仍属未决问题**。

### D.12 dex 字符串不可直接 grep

- **状态：验证失败**（这是结论）
- 全部关键类名 grep 均返回空，`exit 1`。
- **含义**：必须用 jadx 反编译才能确认类名与签名。这是 Stage 2 的硬性前置条件。

---

## E. 代码修改

本阶段为**纯调查阶段**，未修改任何代码或系统文件。

仓库内新建的文件：

| 文件 | 作用 |
|---|---|
| `README.md` | 项目说明、背景、阶段进度 |
| `STATUS.md` | 项目状态跟踪 |
| `.gitignore` | 排除构建产物、凭据、系统 APK |
| `reports/stage-01.md` | 本报告 |
| `logs/stage-01-raw/*` | 第一阶段原始证据（32 个文件，1.3 MB） |

**未修改**：
- ❌ 系统 APK（一个字节都没动）
- ❌ 系统配置（未执行 `settings put` / `setprop` / `mount`）
- ❌ 未安装任何 LSPosed 模块
- ❌ 未创建倒计时应用

---

## F. 测试结果

| 项目 | 结果 | 证据 |
|---|---|---|
| 设备属性读取 | ✅ 通过 | `logs/stage-01-raw/01~04` |
| Root 可用性 | ✅ 通过 | `raw/05_root_check.txt`：`uid=0 u:r:ksu:s0` |
| LSPosed 运行状态 | ✅ 通过 | `ps -A`：`lspd` pid 5702 + 4 个 companion |
| Java 可用 | ✅ 通过 | OpenJDK 21.0.12.1 |
| Gradle 可用 | ❌ 失败 | `command not found` |
| Android SDK 可用 | ❌ 失败 | 目录不存在 |
| jadx 可用 | ❌ 失败 | `which` 无输出 |
| 四组件路径提取 | ✅ 通过 | `raw/10~13` |
| AOD 插件入口定位 | ✅ 通过 | `raw/14_aod_fulldump.txt` |
| UID 共享确认 | ✅ 通过 | 两包 `sharedUser=android.uid.system/1000` |
| 框架符号命中（7 项） | ✅ 7/7 通过 | `raw/20~25` |
| `MiuiFlipPolicy` 符号 | ❌ 未命中 | grep 返回空 |
| 双屏 displayId 映射 | ✅ 通过 | `raw/31_display.txt` |
| AOD 双进程常驻 | ✅ 通过 | `ps -A` 实测 |
| dex 类名验证 | ❌ 失败 | `raw/40_dex_string_check.txt` |
| `dumpsys SurfaceFlinger` | ⚠️ 未测试（被策略拦截） | `[POLICY_BLOCKED]` |
| `dumpsys dreams` | ⚠️ 未测试（被策略拦截） | `[POLICY_BLOCKED]` |
| 折叠状态 powerGroupId 复测 | ⚠️ 未测试 | 需人工折叠设备 |
| 外屏 AOD 实际激活测试 | ⚠️ 未测试 | 本阶段禁止改系统 |
| GitHub 仓库创建 | ❌ 失败 | 无认证 |
| GitHub 推送 | ❌ 失败 | 无认证 |

---

## G. 当前阻碍

### G.1 GitHub 推送完全阻塞（最高优先级）

**现象**：

```bash
$ which gh ssh
（未找到 gh、ssh）
$ ls /root/.ssh/
（空）
$ env | grep -i "^GH_\|^GITHUB_"
（无输出）
$ git config --global --list
fatal: unable to read config file '/root/.gitconfig'
```

**结论**：本机**没有任何 GitHub 认证凭据**。同时 `mixflip2-aod-countdown` 仓库在 GitHub 上不存在（API 404，全站搜索 0 结果）。

**影响**：按规范「如果 GitHub 推送失败，不得声称阶段已经同步完成」，本阶段**无法声明已同步**。所有成果目前只在本地。

**需要用户操作**（三选一）：

1. **手动创建仓库** + 提供 Personal Access Token（需 `repo` 权限），我再配置 remote 推送
2. **手动创建仓库** + 生成 SSH 密钥让我把公钥给你，你加到 GitHub
3. **手动创建仓库** + 你自己把本地内容推上去

推荐方案 1（最直接，且 GITHUB_TOKEN 可只放环境变量不外泄）。

### G.2 开发工具链缺失

- Gradle、Android SDK、jadx 全部未安装。
- **影响**：Stage 2 若需要反编译 APK 或编译 Hook 模块，会直接卡住。
- **建议**：Stage 2 开始前先安装。jadx 可以通过下载 release jar 解决（不需 SDK）；Android SDK + Gradle 用于编译模块，也可以用 GitHub Actions CI 编译绕过（参考项目就是这么做的）。

### G.3 设备 shell 策略限制

- `dumpsys SurfaceFlinger`、`dumpsys dreams`、`dumpsys activity services <pkg>` 被拦截。
- **影响**：无法直接观测 AOD 窗口的图层结构与合成顺序——这恰恰是判断「能否追加自定义图层」的关键证据。
- **绕行方案**：改用 `logcat` 抓日志，或通过 root 直读 `/d/` 节点。需 Stage 2 验证。

### G.4 dex 字符串加密

- 关键类名无法 grep 验证。
- **影响**：D.9 表格里 Layer2 的 4 个类名全部处于「有代码证据但未实机验证」状态。
- **绕行方案**：必须反编译（jadx）。

### G.5 折叠状态无法远程切换

- 当前设备展开状态，外屏熄灭，无法观测 `powerGroupId=1`。
- **影响**：D.7 只能停留在推测。
- **需要用户配合**：折叠手机后让我复测，或提供当时的 `dumpsys power` 输出。

### G.6 潜在风险（需后续阶段警惕）

1. **SystemUI 崩溃环风险**：参考项目注释明确记录，`DisplayUtils.getCutoutPosition()` 存在 NPE，若处理不当会导致 SystemUI 7 秒级崩溃重启循环。这是最高危的风险点。
2. **Hook Flip2 可能完全无效**：项目自述为 flip1 基版，HyperOS 3 的方法签名、类名、ClassLoader 结构都可能已经变了。
3. **AOD 内容不可替换**：即使外屏 AOD 成功开启，显示的仍是系统模板（时钟/通知），不是倒计时圆盘。「倒计时进 AOD」需要额外方案，目前没有任何证据表明可行。
4. **改系统设置的风险**：`persist.sys.multi_display_type` 之类的属性会全局改变设备行为，误改可能导致外屏功能异常。

---

## H. 下一阶段建议

按依赖关系排序：

### 优先级 1：解除阻塞（必须最先做）

1. **打通 GitHub**：用户创建仓库并提供认证方式，把本地提交推上去
2. **安装 jadx**：下载 release，用于反编译 APK（不需要完整 SDK）
3. **评估是否走 CI 编译**：若做 LSPosed 模块，可参考项目用 GitHub Actions 编译，避免本地装 SDK

### 优先级 2：反编译调查（Stage 2 主体）

1. **MIUIAod.apk**（64 MB，最优先）
   - 确认 `DozeServicePluginImpl` 的加载流程与插件生命周期
   - 确认 `DozeMachine` / `DozeHost` / `FlipLinkageStyleController` 的真实类名与签名
   - 确认 ClassLoader 隔离方式（决定 LSPosed 注入策略）

2. **MiuiSystemUI.apk**（次优先，但决定核心问题）
   - 确认 `DozeServicePluginImpl` 如何被 connect
   - 确认 AOD 窗口如何创建、图层如何组织
   - **关键判断：能否向 AOD 追加自定义图层** —— 这直接决定「倒计时圆盘能否进 AOD」

3. **services.jar**
   - 核对 `updateRearDozeSettings` / `handleRearSandman` 的精确签名
   - 确认 `mRearAlwaysOnEnabled` 的读写路径

4. **FlipHome.apk**
   - 确认外屏倒计时页的常亮实现（`FLAG_KEEP_SCREEN_ON` 还是系统私有接口）

### 优先级 3：实机验证

1. 折叠设备，复测 `dumpsys power` 确认 `powerGroupId=1`
2. 用 `logcat` 替代 `dumpsys SurfaceFlinger` 采集 AOD 期间的行为
3. 评估参考项目直接移植到 Flip2 的可行性（预期需大量改动）

### 优先级 4：方案决策（需用户拍板）

在阶段 2 完成后，需要明确一个关键分歧：

- **方案 A**：只求「外屏能开 AOD」（参考项目已实现，改造成本中等，但 AOD 上看到的仍是系统时钟，不是倒计时）
- **方案 B**：求「倒计时圆盘进 AOD」（需要改 SystemUI 或 AOD 模板，难度高，可行性未证实）
- **方案 C**：放弃 AOD，做「全黑背景 + 倒计时圆盘 + 低亮度常亮」的独立应用（可行但省电有限）

**本阶段不推进任何开发，等待用户与 ChatGPT 确认 Stage 2 指令。**

---

## I. GitHub 信息

| 项 | 值 |
|---|---|
| 仓库 | `mixflip2-aod-countdown`（**GitHub 上尚未创建**） |
| 分支 | `main` |
| 最新 Commit SHA | `2ceb6fa0ca93c38a7c38fa6703863640251dd819`（本地提交成功） |
| 本阶段新增文件 | 见下 |
| 是否推送成功 | ❌ **否** — 认证缺失（详见 G.1） |

### 本地文件清单（待推送）

```
mixflip2-aod-countdown/
├── .gitignore
├── README.md
├── STATUS.md
├── reports/
│   └── stage-01.md            ← 本报告
└── logs/
    └── stage-01-raw/          ← 32 个证据文件，1.3 MB
        ├── 01_device_model.txt
        ├── 02_device_codename.txt
        ├── 03_hyperos_version.txt
        ├── 04_android_version.txt
        ├── 05_root_check.txt
        ├── 06_root_modules.txt
        ├── 10_aod_path.txt
        ├── 11_fliphome_path.txt
        ├── 12_systemui_path.txt
        ├── 13_deskclock_path.txt
        ├── 14_aod_fulldump.txt
        ├── 15_systemui_uid.txt
        ├── 20~25_fw_*.txt
        ├── 30_power.txt
        ├── 31_display.txt
        ├── 32_pms_class.txt
        ├── 33_aod_plugin.txt
        ├── 40_dex_string_check.txt
        └── （参考项目源码与 API 响应）
```

### 推送阻塞说明

**本阶段成果未同步到 GitHub。** 原因是本机无任何认证凭据，且仓库不存在。

按规范要求「如果 GitHub 推送失败，不得声称阶段已经同步完成」——此处如实声明：**未同步**。

待用户提供认证后，将执行：

```bash
git add .
git commit -m "stage-01: MIX Flip 2 外屏 AOD 环境调查完成"
git remote add origin <repo-url>
git push -u origin main
```

---

## 阶段结论

**Stage 1 调查目标基本达成**：外屏 AOD 的实现路径已定位到 SystemUI 插件层，框架 Hook 入口已确认 7 个符号，参考项目源码已完整分析。

**但有一个关键问题仍未解答**：AOD 的内容是否可被替换/追加。这个问题的答案（在 `MiuiSystemUI.apk` 与 `MIUIAod.apk` 的反编译结果里）将直接决定项目走哪条技术路线。

**本阶段不擅自进入下一阶段。等待指令。**
