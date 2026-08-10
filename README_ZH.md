# pi-skill-evolution

给 Pi Agent 的 Hermes Agent 风格自动技能进化扩展。

Pi Agent 已经有很好的技能系统和扩展 API，但缺少"自动创建/修改/进化技能"的闭环能力——这正是 Hermes Agent 通过 `background_review` 和 `skill_manage` 实现的核心特性。本仓库在 Pi 的 event-driven 架构上补齐这块拼图。

## 安装

```bash
# 扩展
cp extensions/skill-evolution.ts ~/.pi/agent/extensions/

# 技能（写作指导，非必需）
mkdir -p ~/.pi/agent/skills/skill-authoring
cp skill-authoring/SKILL.md ~/.pi/agent/skills/skill-authoring/
```

Pi 会自动发现并加载，无需设置变更。

## 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `PI_SKILL_EVOLUTION_DIR` | `~/.pi/agent/skills/` | 技能存储目录 |

更多细节见 [README.md](README.md)。