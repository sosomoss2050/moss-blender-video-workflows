# libtv 交付路径（Moss 团队二开）

本节使白模工作流**无缝嵌入 libtv 画布全自动生产管线**：用户只需提需求，画布创建、绑定、节点搭建、生成、合成全部由 agent 经 libtv CLI 自动完成。libtv CLI 不可用时（`which libtv` 失败）回退到[视频生成](video-generation.md)原生服务路径。

## 能力边界（先读）

- libtv 的 `video` 节点参考输入是**图片**（`mixed2video` 经 `--left` 挂 image 资源节点），**不接受视频参考**。白模 MP4 按镜头导出关键帧图挂参考（弱约束，交付时声明）。
- 节点 `duration` 由模型 schema 决定：部分模型是连续滑条（如 Seedance 2.0 Mini 为 4–15s 整数步进），部分是离散档位。**用 `libtv model <modelKey>` 查实际 schema 后再定 shot-plan 时长**，不凭经验猜档位。
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

# 2) 关键帧风格化（关键步骤，不可跳过）
#    frames2video/mixed2video 下参考图的外观权重远高于 prompt 文字；
#    直接挂白模灰白帧会生成“白模风”成片。必须先用 image 节点把白模帧转成目标风格：
libtv node create "镜头01-首帧-风格化" -t image \
  --prompt "将参考图中的场景与人物转为写实风格：现代科研实验舱，穿浅蓝实验服的科研人员，深灰色金属控制装置，冷色调环境光。保持构图、人物姿态与空间关系不变。" \
  --set "model=Seedream 4.0" --set ratio=16:9 --left "镜头01-首帧" --run
libtv node create "镜头01-尾帧-风格化" -t image \
  --prompt "同前风格：写实科研实验舱，人物面向装置抬手，装置环形灯带发出蓝色光晕。保持构图与姿态不变。" \
  --set "model=Seedream 4.0" --set ratio=16:9 --left "镜头01-尾帧" --run

# 5) video 节点：prompt 取 prompts.md 该镜头段落，挂风格化后的帧（非白模原始帧），直接生成
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

## 实战经验（2026-10-11 首次端到端实弹验证）

1. **CLI 安装位置**：CLI 装在 `~/.libtv/libtv`，不一定在 PATH 里。检测用 `which libtv || [ -x ~/.libtv/libtv ]`，后者直接用全路径调用。
2. **`project create` 返回结构**：uuid 在 `projectMeta.uuid` 下，不在顶层；workspaceId 同理在 `folders[].id`。解析 stdout JSON 时按实际结构取。
3. **生成等待**：`libtv node --run` 会随任务结束而退出，但进度要另查节点：`libtv node "<名>"` 读 `taskInfo.status`（1=进行中带 progressPercent，2=成功，3=失败）与 `url[]`（成品视频地址在节点顶层 `url` 字段，不在 `results`）。
4. **低成本试水**：样片模式（如 `star-video2.5-draft`，480P）可先低成本验证动作，满意再直出高清。
5. **成片验收方法**：`ffmpeg fps=3` 采样成片 → 逐帧差分看运动曲线（应与人工作业时序吻合、末段归零）→ 逐帧亮蓝像素检测蓝光出现时点。与白模触发时点偏差 1s 内可接受（弱约束）。
6. **外观白模化检测（饱和度检查）**：成片若是灰白素模风，用饱和度指标判定——`整帧平均饱和度<10` 即为白模风污染（真实感画面通常 >40）。根因是参考图外观权重压过 prompt：必须走关键帧风格化步骤，白模原始帧禁止直接挂 video 节点。风格化帧自检阈值：平均饱和度 >40 稳妥，冷色调暗场景可放宽到 >15（需确认非素模灰白）。
7. **尾帧避免带发光状态**：frames2video 会把尾帧中的发光等终态“匀速回放”到全片，导致触发时序失效（如蓝光从 t=0 就亮）。尾帧应选目标事件发生**之前**的邻近帧（如触发前 0.3s），终态发光交给 prompt 时间线描述。
8. **样片模式时序服从度弱**：480P 样片模式（如 star-video2.5-draft）适合低成本验证外观与动作，但“末段定格”“精确触发时点”等时序约束服从度明显 weaker，验收时对样片只判外观与动作结构，时序问题留到正式高清版复验。

## 分镜主导权约定（防两套分镜打架）

**shot-plan.md 是唯一分镜源**。进 libtv 时用 `-u rows` 直接写入 shot-plan 的分镜内容，**不**让 script 节点的文本生成器重新产出一版分镜。`--run` 触发的文本生成仅在需要 libtv 补充旁白/对白等文本内容时使用。

## 生成前自检（每次 --run 前）

1. **提示词回读核对**：`libtv node "<节点名>"` 读回 `params.prompt`，与 prompts.md 对应段落核对一致后才触发生成。
2. **参考图外观检查**：挂给 video 节点的首尾帧必须是风格化后的图；若为灰白素模帧，先走风格化步骤（见上方序列第 2 步），否则成片会复制白模外观。

## 交付与验收

- 生成结果按[视频生成](video-generation.md)「结果」节逐段对比白模，弱约束偏差照实报告。
- 生成任务失败先查原节点状态，不重复 `--run` 造成重复扣费。
- 交付区分：白模 MP4 / .blend / prompts.md / 画布节点与成片，标注各自验证状态；libtv 节点即在线交付物，用户可直接在网页画布查看。
