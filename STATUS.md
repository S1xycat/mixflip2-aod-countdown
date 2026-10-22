# 项目状态

**最后更新**：2026-10-22
**当前阶段**：Stage 1 — 外屏 AOD 环境调查（✅ 本地完成 / ❌ 未推送）
**最新 Commit SHA**：`2ceb6fa0ca93c38a7c38fa6703863640251dd819`（仅本地，未推送）
**分支**：`main`

---

## 最近完成的任务

- [x] Stage 1：MIX Flip 2 外屏 AOD 环境调查
  - 设备/系统/Root/LSPosed 环境确认
  - 四组件（aod / fliphome / systemui / deskclock）路径与版本提取
  - 外屏 AOD 实现机制定位（SystemUI PLUGIN_DOZE 插件架构）
  - 框架层 Hook 符号命中验证（7 项）
  - 参考项目 FlipOuterUnlock_262 的 AodHook 源码分析

---

## 已验证技术结论

以下结论均有本机实测证据，详见 `reports/stage-01.md`。

1. **AOD 是 SystemUI 的插件，不是独立应用**
   `com.miui.aod/.doze.DozeServicePluginImpl` 通过 `com.android.systemui.action.PLUGIN_DOZE` 由 SystemUI 加载；两者同属 `android.uid.system`(1000)。

2. **外屏 displayId = 0，内屏 displayId = 1**（与常识相反）
   - 外屏：1208×1392，刘海 `Rect(810,664-1208,1392)`，`installOrientation=ROTATION_180`
   - 内屏：1224×2912，刘海 `Rect(569,0-655,146)`，`installOrientation=ROTATION_0`

3. **框架层外屏 Doze 符号全部存在**
   `updateRearDozeSettings`、`handleRearSandman`、`mRearAlwaysOnEnabled`、`shouldDeviceBeSleep`、`DreamController`、`setDozeScreenState`。

4. **AOD 包版本名带 `DEV-FLIP`**
   `DEV-FLIP-2335.1.2.1-05070145` — 说明该 AOD 包为折叠屏专门编译。

5. **时钟应用无外屏专用 Activity**
   `com.android.deskclock` 组件表中无任何 flip/cover/outer 命名，倒计时仍是通用 `DeskClockTabActivity`。

6. **LSPosed 环境可用**，可直接加载 Hook 模块。

7. **DRM 双连接器独立**：`card0-DSI-1`（内屏）、`card0-DSI-2`（外屏）。

---

## 尚未解决的问题

| # | 问题 | 状态 |
|---|---|---|
| 1 | `powerGroupId=1` 在外屏 AOD 激活时是否真实出现 | 需折叠实测 |
| 2 | `DozeMachine` / `DozeHost` / `FlipLinkageStyleController` 在 HyperOS 3 的真实类名与签名 | dex 字符串加密，grep 失败，**需反编译** |
| 3 | Layer2 对象图遍历路径是否仍可达 | 需反编译 + 实机 |
| 4 | AOD 插件是否使用独立 ClassLoader | 需反编译 |
| 5 | **外屏 AOD 能否承载非系统模板内容（倒计时圆盘能否进 AOD）** | **核心未决问题** |
| 6 | `dumpsys SurfaceFlinger` / `dumpsys dreams` 被设备策略拦截 | 需改用 logcat 或 root 直读 |
| 7 | `updateRearDozeSettings` 精确参数签名 | 需反编译 services.jar |

---

## 阻塞项

- **GitHub 推送未完成**：本机无 SSH 密钥、无 gh CLI、无 Token、无 credential helper。仓库 `mixflip2-aod-countdown` 在 GitHub 上不存在（API 404）。需要用户手动创建空仓库并提供认证方式。
- **开发工具链缺失**：Gradle、Android SDK、jadx 均未安装。Stage 2 若需编译或反编译，必须先搭建。

---

## 是否具备进入下一阶段的条件

**部分具备。**

- ✅ LSPosed 环境就绪，可直接安装 Hook 模块
- ✅ 框架层 Hook 入口已确认
- ❌ 反编译工具缺失（Stage 2 反编译 APK 前必须解决）
- ❌ GitHub 同步未打通（阶段成果无法归档）

建议 Stage 2 优先级：**先补工具链 + 打通 GitHub，再做反编译调查**。

---

## 下一阶段建议

1. 反编译 `MIUIAod.apk`，确认 `DozeMachine` / `DozeHost` / `FlipLinkageStyleController` 的真实签名
2. 反编译 `MiuiSystemUI.apk`，确认 AOD 窗口创建流程与是否允许追加自定义图层（**决定倒计时能否进 AOD 的关键**）
3. 折叠状态实测 `powerGroupId=1` 是否出现
