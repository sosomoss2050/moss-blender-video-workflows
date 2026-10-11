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
9. **duration 低于模型下限直接拒单**：各模型 duration 有 min/max（如 star-video2.5-draft 为 4–30s），设 3 秒会直接生成失败（status=3，积分返还）。**建节点前先 `libtv model <modelKey>` 查 duration 范围**，shot-plan 分镜头时长全部落在范围内。
10. **audio 节点选音乐模型必须显式 `--set scene=Music`**：audio 默认场景是 seed-audio（语音类），选 Eleven Music V3 / Mureka 等音乐模型时不指定 scene 会报「场景不一致」拒绝创建。音乐最短 30s，成片不足时长在合成阶段裁剪+尾淡出。
11. **平台侧偶发失败按纪律重试**：failedReason 为「视频生成失败，积分将会返还」属平台临时错误——先查因、再重试原节点（`libtv node "<名>" --run` 即可，参数保留），不删节点重建、不盲目三连扣费。若连续失败再检查参数合法性（常见：duration 越界、参考图违规）。
12. **多镜头批量生成的会话韧性**：批量 `--run` 遇执行会话中断会留下「空壳节点」（有名字无参数），再建同名节点会报重名冲突。处理：`libtv node list` 找 id → `libtv node delete <id>` 清理空壳 → 重建。并行提交（& + wait）可提高批量效率但空壳风险更高，重要任务建议串行。

## 声音三件套（配乐 / 对白 / 音效）—— 2026-10-11 实测铁律

Seedance 2.5 支持**配乐 + 对白 + 音效同出**，但三者都要显式要求，**缺哪块就没有哪块**——不是模型不支持。

1. **总开关 `-s enableSound=on`**：默认就是 `on`，但显式写成 `off` 会全程无声（曾误设一次，生成无声成片）。建节点时**显式写 `on`**做保险。
2. **对白**：prompt 里必须写**台词文本 + 性别 + 语气**，例如：`女子轻声说出台词：「今天风真温柔。」（气声，温柔，中文女声，清晰可辨）`。只写角色名或“她说话”不会出准确对白。
3. **配乐**：prompt 里必须写**乐器 + 风格 + 情绪**，例如：`【背景配乐】全片铺底古筝为主的国风器乐，轻盈流动，春日温柔舒缓的旋律，音量低于对白`。**中式/古风场景用古筝/国风**；只写“背景音乐”通常不出配乐。
4. **音效**：环境音单独列，例如：`【音效】微风风声、庭院鸟鸣、衣料轻响`。
5. **验收**：`ffprobe -show_entries stream=codec_type,codec_name 成片` 必须有 `video(h264)` **且有 `audio(aac)`**；无声版本只有视频轨。有音频轨后再人工试听确认配乐/对白/音效三轨齐全。
6. **prompt 四件套模板**：画面 + 台词 + 配乐 + 音效，见 [`视频生成`](video-generation.md)。

## 成本基准（Advanced 会员）

| 项 | 值 |
|----|-----|
| Advanced 会员（进阶版VIP连续包年） | 月 4600 积分 |
| Seedance 2.5 样片模式 480P 4秒单镜头 | **80 积分/镜头** |
| 月可跑 | ≈ 57 个样片镜头 |
| 5 镜头短片 | ≈ 400 积分（月额度 8.7%） |

生成为**结果交付**：改 prompt 重跑会再次扣费。**先确认方向再跑**，不要为调一个小参数盲跑多版（曾为调声音/配乐连跑 3 版，白烧 160 积分）。

## 分镜表这事（2026-10-11 实测修正）

**Seedance 能自己分镜吗？——能，而且实测可行，优先一次提交。**

三种方案，按场景选：

| 方案 | 做法 | 适用 |
|------|------|------|
| ① **单节点多镜头硬切** ⭐ | 一个 video 节点，prompt 的 `【Storyboard】` 段写 `[Shot 01][Duration] 0–4秒 …` 到 `[Shot 05]`，`duration=总秒数`，`modeType=mixed2video`，一次提交 | **同场景、同角色组的多个镜头**（首选，效率高、免拼接、一致性更好） |
| ② 一镜一节点 | 每镜头一个 video 节点，各自 prompt + 参考图 | 跨场景/跨角色/需要极致单镜控制时 |
| ③ 长视频续接 | `supportVideoContinuation` 上一条尾帧接下一条 | 相邻镜头衔接优化 |

**实测证据（2026-10-11，20 秒 5 镜头一次提交）：**
- ffmpeg 场景切换检测到的切点：**4.67 / 8.50 / 12.04 / 15.38 秒**，与分镜设计的 4.0 / 8.5 / 12.5 / 15.3 秒几乎吻合（样片模式最大偏差 0.67s）。
- 三角色跨镜头形象一致，古筝配乐 + 音效正常。
- 参数：`duration=20`、`modeType=mixed2video`、挂 4 张参考图（角色图被自动转 `assetId`）、`enableSound=on`，约 400 积分。

**结论：同场景多镜头优先用方案 ①。** `lens.镜头 maxCount=1` 指的是画布上的镜头节点数量，**不限制** prompt 内部按 Shot 段落硬切。

## ⚠️ 多镜头连续性（实测踩坑）

**某角色在某 Shot 的 prompt 里没被提到，模型就不会让她出现**；下一 Shot 又写她在场 → 突然冒出，**穿帮**。

> 实例：Shot 01 只写“庭院空镜→女主入画”，没提侍女；Shot 02 写“侍女已在秋千旁等候” → 成片里侍女在 Shot 02 凭空出现。

**修正**：在【角色在场表】或每个 Shot 里写明**所有应在场角色及其位置**（哪怕不是主体）。全局声明“三位角色全片在场”，逐镜头标注谁是当镜主体。

**人数要明确**：模型可能**多生成人物**（实测：要求女配C+女配D 两人旁观，成片出现 3 人）。在场表写明“后方围观者共 2 人（女配C + 女配D）”，负向约束加“无多余人物、围观者严格 N 人”，并给每个配角**指定站位**（如 C 站左后、D 站右后）。

## ⚠️ 越轴（180 度线）（实测踩坑）

多角色对峙/交互场景（如掰手腕），**局部特写镜头可能越轴**，导致左右关系翻转、看着穿帮。

> 实例：镜头05 中景 A 左 B 右，镜头06 手臂特写跳到轴线另一侧。

**修正**：在【空间逻辑】段加越轴约束：“全片保持同一侧 180 度轴线，主体始终在画面同侧（如 A 左 B 右），**禁止越轴**”；或对特写镜头明确机位：“从与前一镜**同一侧**拍摄，保持 A 在画左”。

## 参考图外观污染（防白模风）

参考图的外观权重**远高于 prompt 文字**：
- **白模原始帧禁止直接挂 video 节点**会生成灰白素模风成片。若用白模作构图参考，先经 image 节点（Seedream 4.5）**转成目标写实风格**再挂。
- 自检阈值：成片整帧平均**饱和度 <10** 即白模风污染（真实感画面通常 >40）。

## 验收方法（眼见为实 + 客观检测）

1. **肉眼**：`view_image` 查看关键帧，确认画面有内容、构图符合、外观非白模风、人物到位。**光看亮度/饱和度数值会误判**（曾靠数值判断“画面正常”，实际相机朝天全白）。
2. **客观（多镜头时）**：`ffmpeg` 场景检测验证切点是否落在分镜设计时刻：
   ```bash
   ffmpeg -i out.mp4 -filter:v "select='gt(scene,0.25)',showinfo" -f null - 2>&1 | grep -o "pts_time:[0-9.]*"
   ```
3. **音轨**：`ffprobe -show_entries stream=codec_type,codec_name` 须有 `video(h264)` **且有 `audio(aac)`**；再试听确认配乐/对白/音效三轨。
4. 图像模型超时时，客观检测（场景切换/音轨/时长）可独立完成验收，不必依赖看图。

## 分镜主导权约定（防两套分镜打架）

**shot-plan.md 是唯一分镜源**。进 libtv 时用 `-u rows` 直接写入 shot-plan 的分镜内容，**不**让 script 节点的文本生成器重新产出一版分镜。`--run` 触发的文本生成仅在需要 libtv 补充旁白/对白等文本内容时使用。

## 生成前自检（每次 --run 前）

1. **提示词回读核对**：`libtv node "<节点名>"` 读回 `params.prompt`，与 prompts.md 对应段落核对一致后才触发生成。
2. **参考图外观检查**：挂给 video 节点的首尾帧必须是风格化后的图；若为灰白素模帧，先走风格化步骤（见上方序列第 2 步），否则成片会复制白模外观。

## 交付与验收

- 生成结果按[视频生成](video-generation.md)「结果」节逐段对比白模，弱约束偏差照实报告。
- 生成任务失败先查原节点状态，不重复 `--run` 造成重复扣费。
- 交付区分：白模 MP4 / .blend / prompts.md / 画布节点与成片，标注各自验证状态；libtv 节点即在线交付物，用户可直接在网页画布查看。
