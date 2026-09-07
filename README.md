# Hermes 配置工作流

![GitHub stars](https://img.shields.io/github/stars/ninggui/hermes-config-workflow)
![License](https://img.shields.io/github/license/ninggui/hermes-config-workflow)
[![SkillHub](https://img.shields.io/badge/SkillHub-在线安装-blue)](https://skillhub.cn/skills/hermes-config-workflow)

config.yaml / API key / fallback 链 / web 后端配置实操。

## 这是什么

一个可复用的 AI Agent 技能（Skill），来自真实业务场景沉淀，含完整执行流程、避坑清单与验证步骤。

## 快速使用

将本仓库放入 Agent 技能目录后，用对应触发词调用（见 SKILL.md），Agent 会自动加载并执行完整流程。

## 核心能力

| 能力 | 说明 |
|------|------|
| config.yaml 安全修改通道 |
| 凭据池与 .env 管理 |
| fallback 链配置与验证 |
| web 后端切换 |

## 使用方式（安装）

- **Hermes**: 放入 `skills/` 目录
- **Claude**: 放入 `~/.claude/skills/`
- **其他 Agent**: 按对应 SKILL.md 格式放入技能目录
- **SkillHub 一键安装**: https://skillhub.cn/skills/hermes-config-workflow

## 优势

- 全部实测（含 402/401/429 处置）
- key 切换不删除的铁律
- 升级后配置审计清单

## 内容结构

- `SKILL.md` — 核心技能定义（触发条件、执行流程、避坑清单）
- `references/` — 可选参考文件

## 许可

MIT
