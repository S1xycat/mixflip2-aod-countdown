# 外屏 AOD 架构调查笔记

> 本文件记录 Stage 1 过程中整理的系统架构认知，供后续阶段参考。
> 标注：【已实机验证】/【代码证据】/【推测】/【验证失败】

## 1. 显示拓扑

```
                  ┌─────────────────────────────┐
                  │      card0 (同一 DPU)        │
                  └───────┬─────────────┬───────┘
                          │             │
                  card0-DSI-1     card0-DSI-2
                    (内屏)          (外屏)
                  displayId=1    displayId=0  ← 注意：外屏是 0
                  1224×2912      1208×1392
                  ROTATION_0     ROTATION_180
```

【已实机验证】`/sys/class/drm/` 节点列表：

```
card0  card0-DSI-1  card0-DSI-2  card0-Virtual-1  card0-Virtual-2
card0-sde-crtc-{0..7}  renderD128  sde-conn-2-DSI-1  sde-conn-3-DSI-2
```

【已实机验证】每个 `card0-DSI-*` 下只有标准节点：
`connector_id / device / dpms / edid / enabled / modes / power / status / subsystem / uevent`

→ 没有可编程的自刷新 / AOD 控制节点，第三方无法通过 sysfs 控制面板低功耗态。

## 2. AOD 架构

```
┌──────────────── com.android.systemui (uid 1000) ────────────────┐
│                                                                  │
│   ┌────────────────────────────────────────────────────────┐    │
│   │  PLUGIN_DOZE 插件机制                                    │    │
│   │  Action: com.android.systemui.action.PLUGIN_DOZE         │    │
│   │                                                          │    │
│   │  ┌──────────────────────────────────────────┐           │    │
│   │  │  com.miui.aod/.doze.DozeServicePluginImpl │           │    │
│   │  │  （MIUIAod.apk 提供，同 UID 插件）         │           │    │
│   │  └──────────────────────────────────────────┘           │    │
│   └────────────────────────────────────────────────────────┘    │
│                                                                  │
│   AOD 窗口在此进程内绘制，内容来自 com.miui.aod 的模板            │
└──────────────────────────────────────────────────────────────────┘
                              │
                    DreamService / DozeMachine
                              │
┌───────────────── system_server ─────────────────┐
│  PowerManagerService                            │
│    ├─ updateRearDozeSettings(groupId, ...)      │
│    ├─ handleRearSandman(groupId=1)              │
│    └─ mRearAlwaysOnEnabled                      │
│  DreamController.stopDream(force, reason)       │
│  DisplayManagerServiceImpl.shouldDeviceBeSleep() │
└─────────────────────────────────────────────────┘
```

【已实机验证】插件注册证据：

```
$ pm dump com.miui.aod | grep -A3 "Service Resolver Table"
Service Resolver Table:
  Non-Data Actions:
      com.android.systemui.action.PLUGIN_DOZE:
        6caa39 com.miui.aod/.doze.DozeServicePluginImpl filter b38087e
```

【已实机验证】同 UID：

```
com.miui.aod        → sharedUser=SharedUserSetting{89065f7 android.uid.system/1000}
com.android.systemui → sharedUser=SharedUserSetting{89065f7 android.uid.system/1000}
```

## 3. Doze 屏幕状态码

【代码证据】来自 framework 的 `DreamService.setDozeScreenState(int)` 与参考项目实现：

| 值 | 常量 | 含义 |
|---|---|---|
| 0 | OFF | 屏幕关闭 |
| 1 | ON_SUSPEND / DOZE_SUSPEND | 挂起式亮屏 |
| 2 | ON | 正常亮屏 |
| 3 | DOZE | doze |
| 4 | DOZE_AOD | **AOD 低功耗态**（目标） |

参考项目的核心映射逻辑：`{0, 1, 3}` → `4`，让 `{2, 4}` 透传。

## 4. 外屏组件职责

| 组件 | 职责 | 状态 |
|---|---|---|
| `com.miui.fliphome` | 外屏 Launcher（`.FlipLauncher`），承载外屏分组与页面 | 已实机验证 |
| `com.miui.aod` | AOD 插件实现（`.doze.DozeServicePluginImpl`） | 已实机验证 |
| `com.android.systemui` | AOD 实际绘制宿主 | 已实机验证 |
| `com.android.deskclock` | 时钟应用，无外屏专用 Activity | 已实机验证 |

### 外屏倒计时为什么能常亮

【推测】综合推理：

1. 时钟无 AOD 组件 → 不可能是应用层 AOD
2. AOD 由 SystemUI 插件独占 → 第三方无法参与
3. `FlipLauncher` 处于 `mLastPausedActivity` → 外屏桌面在承载
4. 参考项目需 hook 才能启用外屏 AOD → 外屏默认无 AOD

→ 结论：外屏倒计时的「常亮」是**普通亮屏保持**（可能通过 `FLAG_KEEP_SCREEN_ON` 或系统私有接口），**不是** DOZE_AOD 低功耗态。这就是它耗电的原因。

## 5. 关键类引用

| 类/方法 | 用途 | 验证状态 |
|---|---|---|
| `com.miui.aod.doze.DozeMachine` | doze 状态机 | ❌ dex 加密，未验证 |
| `com.miui.aod.DozeHost` | doze 宿主 | ❌ 未验证 |
| `com.miui.aod.flip.FlipLinkageStyleController` | flip 联动样式控制 | ❌ 未验证 |
| `com.miui.aod.util.DisplayUtils` | 显示工具（有 NPE 风险） | ❌ 未验证 |
| `com.android.server.power.PowerManagerService` | 电源管理 | ✅ 符号存在 |
| `com.android.server.dreams.DreamController` | dream 控制器 | ✅ 符号存在 |
| `android.service.dreams.DreamService` | dream 服务基类 | ✅ 符号存在 |

## 6. 已知坑

1. **SystemUI NPE 崩溃环**：`DisplayUtils.getCutoutPosition()` 对 `display.getCutout()` 无空判断。若移除挖孔导致返回 null，SystemUI 会 7 秒循环崩溃。
2. **外屏 displayId=0**：反直觉，极易写错。
3. **AOD 的独立 ClassLoader**：参考项目注释称 `com.miui.aod.*` 由独立 ClassLoader 加载，`onPackageReady` 的 ClassLoader 看不到。
4. **flip1 vs flip2**：参考项目是 flip1 基版，不能直接套用。
5. **dex 字符串加密**：无法 grep，必须反编译。
