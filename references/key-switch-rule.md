# API Key 切换铁律

## 触发场景

用户提供备用 API Key 或说"费用用完就用这个"时。

## 铁律

**禁止未经验证直接替换主 Key。** 用户意图是配置备用策略，不是立即切换。

## 正确流程

1. **先验证**旧 Key 是否真的耗尽（curl 测试或询问用户）
2. 只是注册备用 Provider + 写入 fallback 链
3. 用户说"切"才切

## 错误案例

2026-08-09：用户说"费用用完了就用这个"，我直接用 sed 把主 Key 替换成备用 Key。用户验证后发现旧 Key 额度还在，严厉纠正："以后不要瞎搞"。

## 正确做法

```bash
# 只是注册备用，不触碰主 Key
hermes config set providers.tokenrhythm-backup.base_url "https://tokenrhythm.studio/v1"
hermes config set providers.tokenrhythm-backup.key_env "TOKENRHYTHM_API_KEY_BACKUP"
hermes config set fallback_providers '["siliconflow", "tokenrhythm-backup"]'

# .env 写入备用 Key（用脚本绕过）
# 主 Key 原封不动
```

## 切换时机

- 用户明确说"切备用"时
- OR 用户说旧 Key 额度确实耗尽时
