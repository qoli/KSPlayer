# KSPlayer（KSMEPlayer）升级 FFmpeg 8.0.1 评估笔记

**目的**：评估将 KSPlayer 的 FFmpeg 依赖升级到 8.0.1 后，是否在 KSMEPlayer 场景带来明显收益（性能、稳定性、格式兼容）。

## 1. 现状摘要（基于代码）
- KSPlayer 使用本地 `FFmpegKit`（`/Volumes/Data/Github/KSPlayer/Package.swift`），升级 FFmpegKit 即等于升级 KSPlayer 的 FFmpeg。
- KSMEPlayer 解码路径：
  - 若 `mediaType == video` 且 `hardwareDecode == true` 且 `asynchronousDecompression == true` 且可创建 `DecompressionSession` → 走 `VideoToolboxDecode`（硬解）。
  - 否则走 `FFmpegDecode`（软解）。
- `AVFFmpegExtension.swift` 中 `getFormat()` 会优先选择 `AV_PIX_FMT_VIDEOTOOLBOX` 并创建 `hw_device_ctx`。
- 默认配置：`hardwareDecode = true`、`asynchronousDecompression = false`（`KSOptions`）。

**结论**：多数情况下，KSMEPlayer 仍可能使用 FFmpeg 软解（因为 `asynchronousDecompression` 默认是 `false`）。

## 2. 升级 FFmpeg 8.0.1 可能带来的收益
### 更可能有收益的场景
- **软解/软编** 比例高（不走硬解或硬解失败回退）。
- 处理 **复杂封装/字幕/滤镜** 或容器/码流兼容性问题。
- 现有版本存在 **已知 bug/崩溃** 或性能瓶颈（例如某些 demux/bitstream 解析）。

### 不一定明显收益的场景
- 主要走 **VideoToolbox 硬解**：性能瓶颈更多取决于系统硬件与驱动，FFmpeg 版本提升不一定显著。
- 播放瓶颈来自 **I/O/网络** 而非解码本身。

## 3. 推荐评估方式（A/B）
### 建议指标
- CPU 使用率、解码线程占用
- 掉帧/卡顿次数
- 缓冲耗时、首帧时间（TTFF）
- 码流兼容性问题（特别是边缘格式）

### A/B 建议方法
1. 固定相同输入（同一组样片、同一设备、同一网络环境）。
2. 对比 **FFmpeg 6.x** vs **FFmpeg 8.0.1**。
3. 记录日志：
   - 解码器是否命中 `AV_PIX_FMT_VIDEOTOOLBOX`（硬解还是软解）。
   - `av_version_info()` 版本输出。
4. 以“平均帧处理时间/CPU 峰值/错误率”作为主指标。

## 4. 风险与注意事项
- **兼容性变化**：FFmpeg 8.0.1 可能带来解码器行为变化（边缘格式可能变好或变差）。
- **编译配置变化**：不同配置（启用/禁用库）会影响真实收益。
- **稳定性测试**：建议在实际内容集上做回归，避免上线后出现格式崩溃或字幕异常。

## 5. 建议结论
- 如果当前 **主要走软解**，或存在格式兼容问题 → **升级有潜在收益**。
- 如果绝大部分走硬解 → **性能提升可能不明显**，升级更多是获得稳定性与兼容性的潜在改善。

## 6. 下一步（可选）
- 添加日志：打印硬解是否命中、FFmpeg 版本信息。
- 做一轮 A/B 对照测试并记录结果。

---
本文档用于初步判断与规划，后续根据测试结果再决定是否上线升级。

## 7. MoltenVK 重新编译需求
- 未来需要使用以下分支重新编译 MoltenVK：
  https://github.com/qiudaomao/MoltenVK/tree/avsample_v1.3.0

## 8. 完全对齐 MPVKit 能力的改动清单（含新增库与 BuildMPV 参数）

### 8.1 版本对齐（必须做）
更新 `Plugins/BuildFFmpeg/main.swift` 中 `Library.version`，对齐 MPVKit 版本矩阵：
- FFmpeg: n8.0.1
- libmpv: v0.41.0
- libass: 0.17.4
- libbluray: 1.4.0
- libdav1d: 1.5.2-xcode（若走上游源码，需换成对应官方 tag）
- libplacebo: 7.351.0-2512（MPVKit 私有 tag）
- libshaderc: 2025.5.0
- vulkan: 1.4.1
- lcms2: 2.17.0
- libdovi: 3.3.2
- gnutls: 3.8.11
- openssl: 3.3.5
- libsmbclient: 4.15.13-2512（MPVKit 私有 tag）
- 其他依赖按 MPVKit 版本表同步

### 8.2 新增/补齐库（必须做）
MPVKit 有而 FFmpegKit 当前缺失或不完整的库：
- libuchardet
- libunibreak
- libluajit（macOS）
- libuavs3d
- libdovi（对齐版本）
- MoltenVK / shaderc / vulkan（对齐版本）

对应修改点：
- `Plugins/BuildFFmpeg/main.swift`：扩展 `Library` enum 与版本/URL
- `Plugins/BuildFFmpeg`：新增/补齐库的 build 脚本
- `Package.swift`：新增 `.binaryTarget` 并补依赖
- `Sources/`：新增对应 `.xcframework`

### 8.3 BuildMPV 参数对齐（必须做）
更新 `Plugins/BuildFFmpeg/BuildMPV.swift`，对齐 MPVKit 的 mpv Meson 参数：
- 新增：
  - `-Duchardet=enabled`
  - `-Dvulkan=enabled`
  - `-Dmoltenvk=enabled`
  - `-Dvideotoolbox-pl=enabled`
- 禁用：
  - `-Djavascript=disabled`
  - `-Dzimg=disabled`
  - `-Djpeg=disabled`
  - `-Dvapoursynth=disabled`
  - `-Drubberband=disabled`
- macOS 专用：
  - `-Davfoundation=enabled`
  - `-Dlua=luajit`
- 非 macOS：
  - `-Davfoundation=disabled`
  - `-Dlua=disabled`

### 8.4 FFmpeg 配置对齐（必须做）
检查 `Plugins/BuildFFmpeg/BuildFFMPEG.swift` 的配置参数，确保启用：
- `--enable-libplacebo` / `--enable-libshaderc` / `--enable-vulkan`
- `--enable-libdav1d` / `--enable-libdovi` / `--enable-libuavs3d`
- `--enable-libass` + `libfreetype`/`libfribidi`/`libharfbuzz`
- `--enable-gnutls` 或 `--enable-openssl`
- `--enable-libbluray`
- `--enable-libsmbclient`（GPL 相关）

### 8.5 产物与依赖清单对齐（必须做）
- `Package.swift`：
  - 使用预编译产物（URL/Checksum）或本地 `.xcframework`，与新增库一致
- `*.podspec`：新增库需要同步依赖

### 8.6 平台基线（可能要做）
MPVKit 的最低平台：macOS 11 / iOS 14 / tvOS 14。
如需“完全对齐”，需评估是否同步提升。

### 8.7 回归验证（必须做）
- tvOS：VideoToolbox 编解码路径
- HDR/DV、字幕、协议、硬解 fallback、音频重采样
- CLI 与 Xcode 双通路一致性

### 8.8 备注
- MPVKit 的部分版本是私有 tag（如 7.351.0-2512、4.15.13-2512），若继续使用上游源码构建需改为可用的官方 tag 或切换到 MPVKit 产物源。

## 9. MoltenVK（avsample_v1.3.0）合并与重编译要点

### 9.1 当前分支状态
- 分支：`avsample_v1.3.0`
- 最新提交：`1feebde4`（fix hdr on iOS）
- 主要改动：
  - 支持 `AVSampleBufferDisplayLayer` 作为渲染目标
  - HDR metadata 相关修正
  - leak/zombie/macos NSScreen 等稳定性修复
  - 新增 AV demo（`Demos/Cube/AVDemo`）

### 9.2 关键改动位置
- `MoltenVK/MoltenVK/GPUObjects/`：`MVKSurface` / `MVKSwapchain` / `MVKImage` / `MVKQueue`
- `MoltenVK/MoltenVK/OS/AVSampleBufferDisplayLayer+MoltenVK.*`
- `Demos/Cube/AVDemo/*`（演示用）

### 9.3 依赖与编译流程
- 外部依赖版本固定在 `ExternalRevisions/*_repo_revision`
- 拉取依赖：`./fetchDependencies --ios --tvos --iossim --tvossim ...`
- 产物打包脚本：
  - `Scripts/package_moltenvk_xcframework.sh`
  - `Scripts/package_all.sh`
- 产物输出目录：
  - `Package/<Config>/MoltenVK/static/MoltenVK.xcframework`
  - `Package/<Config>/MoltenVK/dynamic/MoltenVK.xcframework`

### 9.4 接入 FFmpegKit 的方式
- 直接用该分支重新编出 `MoltenVK.xcframework`
- 替换 `FFmpegKit` 内的 `Sources/MoltenVK.xcframework`
- 记录来源分支（建议在构建说明中注明 `avsample_v1.3.0`）

### 9.5 回归关注点
- AVSampleBufferDisplayLayer 路径（present / pixel buffer 转换）
- HDR10 / Dolby Vision 等 HDR 片源
- tvOS / iOS AVSampleBufferDisplayLayer + PiP 相关场景
