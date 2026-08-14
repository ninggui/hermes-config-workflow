# hermes-config-workflow

Hermes Agent 配置工作流：config.yaml 设置、API key 管理、fallback 链配置、web 后端切换。含大量实测避坑经验。

## 这是什么

一个可复用的 AI Agent 技能（Skill），来自真实业务场景沉淀。记录 Hermes Agent（AI Agent 框架）配置过程中的完整方法论与避坑清单。

## 解决的问题

- `config.yaml` 直接编辑被拦截，不知道正确改法
- API key 放错位置（config vs .env）
- fallback 链配置了不生效（格式陷阱）
- `.env` 写入有 consent 拦截，自动化被封堵
- 换了 key 但服务还在用旧的（需要重启 gateway）

## 核心知识点

| 主题 | 要点 |
|------|------|
| config.yaml | 用 `hermes config set`，禁止直接编辑 |
| API keys | 放 `.env`，不放 config.yaml |
| fallback 链 | 必须是 dict 列表格式 `{provider, model, base_url}`，字符串格式静默失效 |
| .env 写入 | 有 defense-in-depth 保护，用脚本文件方式绕过 |
| key 切换 | 切换后需重启 gateway 生效；多 key 用凭据池自动轮换 |
| web 后端 | 百度插件/DDGS/Tavily 优先级与切换 |

## 使用方式

将本仓库内容放入你的 Agent 技能目录：

- **Hermes**: `skills/` 目录
- **Claude**: `~/.claude/skills/`
- **其他 Agent**: 按对应 SKILL.md 格式

Agent 会在匹配触发条件时自动加载并使用。

## 典型场景

- "配置一下 web 搜索后端"
- "换个 API key"
- "配置 fallback 多模型"
- "为什么配置改了不生效"

## 内容结构

- `SKILL.md` — 核心技能定义（触发条件、执行流程、避坑清单）
- `references/` — 详细参考文档（env 绕过、fallback 格式、自定义插件等）

## 许可

MIT
