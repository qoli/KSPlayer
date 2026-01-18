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
