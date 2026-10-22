# MIX Flip 2 外屏 AOD 倒计时

在小米 MIX Flip 2（bixi / HyperOS 3 / Android 16）折叠状态下，让外屏在「倒计时」场景呈现类似 AOD 的低功耗息屏效果——不显示多余内容，只显示中间的倒计时圆盘。

## 背景

MIX Flip 2 外屏由 `com.miui.fliphome` 承载，系统「时钟」应用（`com.android.deskclock`）对外屏有独立适配，仅提供**闹钟**和**倒计时**两个功能。用户希望在倒计时运行时有类似息屏显示（AOD）的效果以降低耗电。

## 核心问题

1. 系统原生 AOD 是否可以被外屏使用？（**Stage 1 已验证：可以**，需 Hook 启用）
2. 能否把时钟的倒计时圆盘放进原生 AOD？（**尚未解决**，AOD 内容由系统模板决定）

## 仓库结构

```
mixflip2-aod-countdown/
├── README.md                 本文件
├── STATUS.md                 项目状态（每阶段更新）
├── .gitignore
├── docs/
│   └── architecture.md       系统架构调查笔记
├── reports/
│   └── stage-01.md           第一阶段：环境调查
└── logs/
    └── stage-01-raw/         第一阶段原始证据（脱敏后）
```

## 设备与环境

| 项 | 值 |
|---|---|
| 设备 | Xiaomi 2505APX7BC |
| 代号 | bixi |
| 系统 | HyperOS OS3.0.304.0.WOHCNXM |
| Android | 16 / SDK 36 |
| Root | KernelSU |
| LSPosed | 已安装并在运行 |

## 阶段进度

| 阶段 | 内容 | 状态 |
|---|---|---|
| Stage 1 | 外屏 AOD 环境调查 | ✅ 本地完成，**待推送** |
| Stage 2 | 待定 | ⏸ 等待指令 |

## 参考项目

- [FlipOuterUnlock_262](https://github.com/237389749/FlipOuterUnlock_262) — LSPosed 模块，含外屏 AOD Hook 实现（项目自述为 flip1 专用基版）
- [MixFlipMod](https://github.com/parallelcc/MixFlipMod) — 同作者的参考实现

## 安全声明

本仓库不包含：Token、密码、SSH 私钥、用户个人数据、未经脱敏的设备敏感日志、不可再分发的系统 APK。

所有日志均经过脱敏处理，仅保留技术结论所需的组件信息。
