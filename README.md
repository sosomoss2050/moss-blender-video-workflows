# moss-blender-video-workflows

Moss 团队二开的古装/写实短视频生产工作流：**Blender 白模预演编排动作与镜头 → libtv 画布全自动生成带对白/音效的成片 → 配乐合成**。双 skill 结构，覆盖「参考视频复刻」与「文字创意原创」两类入口。

> Fork from [modengsir/blender-video-workflows](https://github.com/modengsir/blender-video-workflows) (MIT)。Moss 二开：skill 更名 `moss-*` 前缀，新增 libtv 交付路径集成、端到端生产 playbook 与 12 条实战护栏。

## Skills

| Skill | 输入 | 用途 |
| --- | --- | --- |
| `skills/moss-blender-video-recreate` | 参考视频 | 复刻原片构图/动作/运镜 → 白模 → 成片 |
| `skills/moss-blender-video-original` | 文字创意 | 原创分镜 → 白模 → 成片 |

两个 skill 共享同一套 references：Blender 制作验收、视频生成、**libtv 交付路径**；并自带 `templates/` 模板体系（镜头表、角色定妆照、七段式分镜、镜头拆解、复刻比对）。

## 核心流程

```
shot-plan.md（台词时长对齐分镜）
→ Blender 白模（竖屏/横屏，低清预览 + 逐段像素验收）
→ 白模关键帧导出 → 关键帧风格化（图生图，防白模外观污染）
→ libtv 画布自动创建绑定 → video 节点（音画同步，对白入 prompt）
→ audio 节点配乐（scene=Music）→ video-clip / ffmpeg 合成
```

## 安装

将 `skills/moss-blender-video-recreate`、`skills/moss-blender-video-original` 复制到 OpenClaw 的 skills 目录（通常 `~/.openclaw/skills/`），重载 skills 即可被自动发现。

## 依赖

| 依赖 | 用途 | 备注 |
| --- | --- | --- |
| Blender ≥ 4.x（验证于 5.2 LTS） | 白模建模/渲染 | `blender -b --python` 无头批处理 |
| FFmpeg / ffprobe | 关键帧导出、编码、合成 | 8.x 验证 |
| libtv CLI ≥ 1.1 | 画布全自动操作 | 安装见 `~/.libtv`；不在 PATH 时用全路径 |
| libtv 账号 | 画布云服务 | 视频生成按积分计价，样片模式低成本 |

## 使用

一句话任务（参数可全部自动推进）：

> 使用 $moss-blender-video-original，做 10 秒实验舱视频：科研人员走入、站稳、抬手，装置蓝光亮起。16:9、720p。

批量短剧模式（多镜头+对白+配乐）见 [PLAYBOOK.md](PLAYBOOK.md)，含五阶段质检表与命令序列。

## 重要护栏（实战沉淀，详见 references/libtv-delivery.md）

1. 白模原始帧**禁止**直接挂 video 节点——参考图外观权重压过 prompt，必须先风格化（饱和度自检 >40）
2. duration 各模型上下限不同，建节点前先 `libtv model <modelKey>` 查 schema
3. audio 节点选音乐模型必须 `--set scene=Music`
4. 样片模式（480P）时序服从度弱，只验外观与动作结构
5. 尾帧避免带发光终态，防止触发时序被"匀速回放"带偏

## 验证状态

- 端到端实弹验证 ×2：实验舱 10s 单镜头（720p 成片）、雨中归坊 22s 五镜头带对白配乐（样片版成片）
- 验收方法与记录样例见交付目录 ACCEPTANCE.md 模板

## License

MIT（继承上游）；参考视频、模型服务遵循各自许可。
