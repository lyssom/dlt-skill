# dlt-skill

大乐透 (Super Lotto) 多策略预测工具技能 for Claude Code

## 功能

- 运行预测 — 15 个策略，每策略 3 注，共 45 注预测
- 反馈开奖结果 — 录入实际开奖号码，分析命中率
- 添加新策略 — 遵循策略契约，开发新策略
- 回测分析 — 30 期滑动窗口回测，综合排名

## 文件结构

```
dlt-skill/
├── SKILL.md              # 主技能文件
└── references/
    ├── strategies.md       # 15 个策略详情
    ├── confidence-system.md # 置信度与回测体系
    └── external-sources.md # 17 种外部研究来源
```

## 安装

将 `dlt-skill/` 目录复制到 Claude Code 的 skills 目录即可。

## 评分

- **skill-validator 评分**: Good (89/100)
- **验证时间**: 2026-05-12

## 许可证

MIT License
