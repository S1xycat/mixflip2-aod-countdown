# Stage 01 原始证据

本目录保存第一阶段（环境调查）的原始证据。所有内容已脱敏：
- 已剔除冗长的系统权限清单（763 行）
- 无 Token / 密码 / SSH 私钥
- 无 IMEI / MEID / 序列号 / MAC / Android ID
- 无系统 APK / JAR 二进制（`.gitignore` 双重排除）

| 文件 | 内容 |
|---|---|
| `01~04_*.txt` | 设备型号、代号、HyperOS 版本、Android 版本 |
| `05_root_check.txt` | Root 身份（uid=0, u:r:ksu:s0） |
| `06_root_modules.txt` | KernelSU 模块清单（含 zygisk_lsposed） |
| `10~13_*_path.txt` | 四组件的 APK 路径 |
| `14_aod_componentinfo.txt` | `pm dump com.miui.aod` 精简版（组件表 + UID + 版本） |
| `20~25_fw_*.txt` | 框架符号 grep 结果（updateRearDozeSettings 等） |
| `30_power_key.txt` | `dumpsys power` 关键字段 |
| `31_display_key.txt` | `dumpsys display` 关键字段（双屏 displayId 映射） |
| `32_pms_class.txt` | PowerManagerService 符号命中 |
| `40_dex_string_check.txt` | dex 字符串 grep 失败记录（负结果，证明需反编译） |
| `AodHook.kt` | 参考项目 AodHook.kt 原件（405 行） |
| `REF_Config.kt` | 参考项目功能开关配置 |
| `REF_module.prop` | 参考项目 LSPosed 模块清单 |
| `REF_FlipOuterUnlock_262_README.md` | 参考项目 README（含兼容性声明） |
| `REF_repo_tree.json` | 参考项目文件树（GitHub API） |
