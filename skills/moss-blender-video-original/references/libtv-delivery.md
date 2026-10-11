# libtv 交付路径（Moss 团队二开）

当本机可用 `libtv` CLI（检测：`which libtv`，或参考 libtv-cli skill）时，视频风格化生成本节路径执行；libtv 不可用时回退到[视频生成](video-generation.md)的原生服务路径。

## 能力边界（先读）

- libtv 的 `video` 节点参考输入是**图片**（`mixed2video` 经 `--left` 挂 image 资源节点），**不接受视频参考**。白模 MP4 不能直接作为运动参考上传。
- 因此标准做法：白模按镜头**导出关键帧图**（弱约束），prompts.md 的动作时间线写入 prompt 文本。这是弱约束，不能保证完整动作复刻，交付时必须声明。
- 节点 `duration` 由模型 schema 决定（如 Seedance 2.0 支持 5/10 秒档）。**shot-plan 的时长必须落在可选档位上**：8 秒创意建议按 5 秒或 10 秒重排动作，不能靠白模 8 秒硬配 5 秒节点。

## 标准命令序列

前置：画布已绑定（`libtv project use`），工作目录含 shot-plan.md、scene.blend、白模 MP4、prompts.md。

```bash
# 1) 每个镜头从白模导出关键帧（首/中/尾 3 帧，FFmpeg）
ffmpeg -ss 0.0 -i whitebox_shot01.mp4 -frames:v 1 refs/shot01_first.png
ffmpeg -ss 4.0 -i whitebox_shot01.mp4 -frames:v 1 refs/shot01_mid.png
ffmpeg -ss 7.8 -i whitebox_shot01.mp4 -frames:v 1 refs/shot01_last.png

# 2) 关键帧挂为 image 资源节点
libtv upload "镜头01-首帧" -t image --resource refs/shot01_first.png
libtv upload "镜头01-尾帧" -t image --resource refs/shot01_last.png

# 3) 建 video 节点：prompt 取 prompts.md 该镜头段落，参数对齐 shot-plan
libtv node "镜头01" -t video \
  --prompt "<目标外观+动作时间线+镜头约束，见 prompts.md>" \
  --set "model=Seedance 2.0" \
  --set modeType=mixed2video \
  --set count=1 --set ratio=16:9 --set resolution=1080p \
  --set duration=5 \
  --left "镜头01-首帧" --left "镜头01-尾帧" \
  --run

# 4)（可选）整片合成
libtv node "成片合成" -t video-clip
libtv node "成片合成" -u clipTimelineData='{"videoSourceNodeIds":["..."],"audioSourceNodeIds":[],"clips":[],"totalDuration":15,"currentTime":0,"scale":1,"scrollX":0}'
libtv node "成片合成" --run
```

参数以 `libtv model search --type video` 与 `libtv model <modelKey>` 的实际 schema 为准，不凭名称猜档位。

## 提示词映射规则

| prompts.md 段落 | libtv 落点 |
| --- | --- |
| 目标外观 | `--prompt` 主体描述部分 |
| 动作时间线 | `--prompt` 时间线部分（节点 duration 档位内重述） |
| 镜头约束 | `--prompt` + `ratio` 参数 |
| 结构/负面约束 | `--prompt` 文字表达（libtv 无独立 negative prompt 字段，不要编造参数） |
| 白模关键帧 | `--left` 参考图（首尾帧定构图与结束状态） |

## 验收

- 生成结果仍按[视频生成](video-generation.md)「结果」节逐段对比白模，弱约束偏差照实报告。
- 任务失败先查原任务（`libtv node` 查询），不重复 `--run` 造成重复扣费。
- 交付区分：白模 MP4 / .blend / prompts.md / libtv 节点与成片，标注各自验证状态。
