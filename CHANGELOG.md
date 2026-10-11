# Changelog

所有显著变更记录在本文件。格式基于 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，版本号遵循 SemVer。

## [1.1.1] - 2026-10-11

### Added
- SKILL.md 新增 Step 0「项目目录初始化」（original/recreate 双 skill）：新建项目必须先建 `~/workspace-<agent>/projects/<项目名>/{tmp,outputs,refs}` 并 cd 进入，后续 Blender/libtv/ffmpeg 全部在该目录内执行；中间产物进 `tmp/`、成品进 `outputs/`；`MEDIA:` 只投 outputs 绝对路径并发后自查出站队列；收尾按前缀精确清理 tmp（禁全目录删除）
- 背景：媒体产物散落 workspace 根目录、MEDIA 指令格式不达标导致投递静默失败（2026-10-11 排查，大壮2号定位）

### Changed
- 安装方式改为软链：`~/.openclaw/skills/moss-blender-video-*` → 本仓库 skills/ 目录，仓库即生效，无需双处同步

## [1.1.0] - 2026-10-11


### Added
- **模板体系**（templates/）：shot-list 镜头表、character-lock 角色定妆照、storyboard-7seg 七段式分镜（original）；shot-decompose 镜头拆解、comparison 复刻比对（recreate）
- SKILL.md 工作流前置化：镜头表先确认再动手、角色定妆照逐张确认、多镜头单节点一次提交
- libtv-delivery.md 实测铁律：声音三件套（enableSound 显式 on / 台词+性别语气 / 配乐乐器+风格，缺哪块就没哪块）、Advanced 会员成本基准（80 积分/样片镜头）、单节点多镜头方案（实测切点偏差 ≤0.67s）、角色在场表防穿帮、越轴约束、view_image 肉眼验收补位

### Removed
- Codex 专用残留 agents/openai.yaml ×2（引用旧 skill 名，OpenClaw 不读取）
- .DS_Store 并加入 gitignore

### 来源
- 本版增量来自胡图图（内容制作部）实测反馈，首次双角色协作迭代

## [1.0.0] - 2026-10-11

### Added
- 双 skill：`moss-blender-video-recreate`（参考视频复刻）/ `moss-blender-video-original`（文字创意原创）
- libtv 交付路径集成：画布自动创建绑定、白模关键帧风格化、音画同步视频节点、audio 配乐（scene=Music）、video-clip 合成
- PLAYBOOK.md：内容制作部五阶段生产流程（shot-plan → 白模 → 提示词 → 画布生成 → 对比验收）与常见坑
- 12 条实战护栏（references/libtv-delivery.md）：白模外观污染防护（饱和度检测）、duration schema 前置查询、尾帧终态回放防护、样片模式能力边界、平台侧失败重试纪律、空壳节点清理等
- 端到端实弹验证 ×2：实验舱 10s 单镜头（720p）、雨中归坊 22s 五镜头带对白配乐（样片版）

### Changed
- 相对上游（modengsir/blender-video-workflows）：skill 更名 `moss-*` 前缀；移除 Codex 专用 agents/openai.yaml；SKILL.md 第5步增加 libtv CLI 检测分支

### Security
- 隐私脱敏扫描通过（0 条敏感信息）
