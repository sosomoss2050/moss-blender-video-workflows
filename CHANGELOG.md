# Changelog

所有显著变更记录在本文件。格式基于 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，版本号遵循 SemVer。

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
