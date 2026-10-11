# libtv 交付路径（Moss 团队二开）

本节使白模工作流**无缝嵌入 libtv 画布全自动生产管线**：用户只需提需求，画布创建、绑定、节点搭建、生成、合成全部由 agent 经 libtv CLI 自动完成。libtv CLI 不可用时（`which libtv` 失败）回退到[视频生成](video-generation.md)原生服务路径。

## 能力边界（先读）

- libtv 的 `video` 节点参考输入是**图片**（`mixed2video` 经 `--left` 挂 image 资源节点），**不接受视频参考**。白模 MP4 按镜头导出关键帧图挂参考（弱约束，交付时声明）。
- 节点 `duration` 由模型 schema 决定（如 Seedance 2.0 的 5/10 秒档）。**shot-plan 阶段就把每镜头时长设计在可选档位上**，不事后凑。
- 提示词映射：prompts.md 的目标外观/动作时间线/镜头约束写进 `--prompt` 文本；libtv 无独立 negative prompt 字段，负面约束用文字表达，不编造参数。

## 全自动序列（用户零手动操作）

前置：已登录（`libtv account list` 可查）；工作目录即本次任务输出目录。

```bash
set -euo pipefail

# 0) 画布自动就绪（已绑定时跳过）
if [ ! -f .libtv/project.json ]; then
  WS=$(libtv workspace create "<任务名>" | jq -r '.workspaceId')
  libtv workspace use "$WS"
  PC=$(libtv project create "<任务名>-画布" | jq -r '.uuid')
  libtv project use "$PC"
fi

# 1) 白模关键帧导出（每镜头首/尾帧，FFmpeg）
ffmpeg -ss 0.0 -i whitebox_shot01.mp4 -frames:v 1 refs/shot01_first.png
ffmpeg -ss 9.8 -i whitebox_shot01.mp4 -frames:v 1 refs/shot01_last.png

# 2) script 节点：shot-plan.md 的分镜表直接写入 rows（白模分镜为主导，不让 libtv 再生成一遍）
libtv node "分镜表" -t script \
  --prompt "<目标风格概述>" \
  --set "model=GVLM 3.1"
libtv node "分镜表" -u rows='[
  {"shotNumber":1,"plotDescription":"0-3s 科研人员从左侧入口走入，走向中央装置"},
  {"shotNumber":2,"plotDescription":"3-7s 减速站稳，抬手触碰控制面板，装置蓝光渐亮"},
  {"shotNumber":3,"plotDescription":"7-10s 镜头缓推定格，主体保持"}]'
libtv node "分镜表" --run

# 3) 分镜图组：按 rows 自动派生 image 节点并逐张生成
libtv script storyboard "分镜表" -s aspectRatio=16:9

# 4) 关键帧挂为 image 资源节点
libtv upload "镜头01-首帧" -t image --resource refs/shot01_first.png
libtv upload "镜头01-尾帧" -t image --resource refs/shot01_last.png

# 5) video 节点：prompt 取 prompts.md 该镜头段落，参考图连入，直接生成
libtv node create "镜头01" -t video \
  --prompt "<prompts.md 镜头01段落：外观+时间线+镜头约束>" \
  --set "model=Seedance 2.0" \
  --set modeType=mixed2video \
  --set count=1 --set ratio=16:9 --set resolution=1080p \
  --set duration=5 \
  --left "镜头01-首帧" --left "镜头01-尾帧" \
  --run

# 6) 多镜头时合成成片
libtv node create "成片合成" -t video-clip
libtv node "成片合成" -u clipTimelineData='{"videoSourceNodeIds":["<镜头01key>","<镜头02key>"],"audioSourceNodeIds":[],"clips":[],"totalDuration":10,"currentTime":0,"scale":1,"scrollX":0}'
libtv node "成片合成" --run
```

参数以 `libtv model search --type video` 与 `libtv model <modelKey>` 实际 schema 为准，不凭名称猜档位。

## 分镜主导权约定（防两套分镜打架）

**shot-plan.md 是唯一分镜源**。进 libtv 时用 `-u rows` 直接写入 shot-plan 的分镜内容，**不**让 script 节点的文本生成器重新产出一版分镜。`--run` 触发的文本生成仅在需要 libtv 补充旁白/对白等文本内容时使用。

## 交付与验收

- 生成结果按[视频生成](video-generation.md)「结果」节逐段对比白模，弱约束偏差照实报告。
- 生成任务失败先查原节点状态，不重复 `--run` 造成重复扣费。
- 交付区分：白模 MP4 / .blend / prompts.md / 画布节点与成片，标注各自验证状态；libtv 节点即在线交付物，用户可直接在网页画布查看。
