# Custom Web Search Provider 插件（百度 qianfan + ddgs 双引擎）

Verified 2026-08-11. 当 Tavily key 432（额度耗尽）时，用百度 AI 搜索做主引擎 + ddgs 兜底。

## 背景

- Tavily 432 = 账户额度问题（非查询问题），换查询词无意义
- 百度 AI 搜索（qianfan）100次/天免费，国内稳定，中文源质量高
- ddgs（DuckDuckGo Python 包）免费无 key，但国内连接不稳定（间歇 30s 超时）
- Hermes `web.backend` 是单值，**无原生 search fallback 链** → 双引擎必须在 provider `search()` 内部实现

## 关键路径

| 项 | 值 |
|---|---|
| HERMES_HOME | `/path/to/data`（用 `get_hermes_home()` 验证，勿假设 ~/.hermes） |
| 用户插件目录 | `/path/to/data/plugins/` || 百度 API | `https://qianfan.baidubce.com/v2/ai_search/chat/completions` |
| 百度 key 默认值 | 见 brand-monitor-search-backend skill（bce-v3/ALTAK-...） |

## 插件文件结构

```
/path/to/data/plugins/baidu/
  __init__.py   # register(ctx)
  provider.py   # BaiduWebSearchProvider
  plugin.yaml   # kind: backend
```

**⚠️ 目录层级坑（2026-08-11 实测）**：插件必须直接放在 `plugins/<name>/` 一层，**不能** `plugins/web/<name>/`（built-in 插件是 web/ 子目录，用户插件扫描只看一层 `plugins/<name>/plugin.yaml`）。`_scan_directory_level` 深度上限 2。另外 `__init__.py` 内部用相对导入 `from .provider import ...`（若写 `from plugins.web.baidu.provider import` 会因目录不同 ImportError）。

`__init__.py`：
```python
from __future__ import annotations
from .provider import BaiduWebSearchProvider

def register(ctx) -> None:
    ctx.register_web_search_provider(BaiduWebSearchProvider())
```

`plugin.yaml`：
```yaml
name: baidu
description: Baidu AI Search with automatic DuckDuckGo fallback
version: 1.0.0
kind: backend
```

## provider.py 核心（search 双引擎）

```python
from agent.web_search_provider import WebSearchProvider

class BaiduWebSearchProvider(WebSearchProvider):
    name = "baidu"
    display_name = "Baidu AI Search (with ddgs fallback)"

    def is_available(self) -> bool:
        return bool(self._get_baidu_key())

    def supports_search(self) -> bool:
        return True

    def supports_extract(self) -> bool:
        return False

    def search(self, query: str, limit: int = 5) -> dict:
        # 1) 百度优先
        try:
            results = self._search_baidu(query, limit)
            if results:
                return {"success": True, "data": {"web": results}}
        except Exception as exc:
            pass  # fall through to ddgs
        # 2) ddgs 兜底
        return self._search_ddgs(query, limit)

    def _search_baidu(self, query, limit):
        # POST https://qianfan.baidubce.com/v2/ai_search/chat/completions
        # json: {messages:[{role:user,content:query}], search_source:"baidu_search_v1",
        #        resource_type_filter:[{type:"web",top_k:limit}], stream:False}
        # headers: Authorization: Bearer <key>
        # 返回 [{title,url,description,position}]，url 从 references[].url
```

注意：provider 用 `urllib.request`（无 requests 依赖更稳），timeout 25s。

## 启用与生效

```bash
export PATH=$PATH:/opt/hermes/bin
hermes plugins list | grep baidu    # source=user
hermes plugins enable baidu          # enabled
hermes config set web.backend baidu
hermes config set web.search_backend baidu
```

**重启 gateway 生效**（关键！）：
- 当前持久主会话保持旧 backend 直到重启（"当前会话仍走 Tavily 432" 是正常现象，不是配置没改）
- 新 CLI 会话（`hermes chat -Q`）会先读新配置，但 provider 插件注册需要 gateway 启动时加载
- 重启方法：`docker restart hermes` 或 delegate_task 子代理执行（子代理独立进程，重启不影响它汇报）
- 验证：`web_search("测试")` 返回成功且无 432；`hermes chat -q "用web_search搜X" -Q` 走新引擎

## 排障

| 症状 | 原因 | 处理 |
|---|---|---|
| `hermes plugins list` 无 baidu | 插件目录不在 HERMES_HOME | `get_hermes_home()` 确认路径，迁移 |
| 显示 not enabled | 未启用 | `hermes plugins enable baidu` |
| 主会话仍走 Tavily 432 | gateway 未重启 / 持久会话缓存 | 重启 gateway |
| ddgs 间歇超时 | DDG 国内连接不稳定 | 属预期，百度为主时少见 |

## 百度 API 请求体（参考 brand-monitor-search-backend skill 亦验证）

```json
{
  "messages": [{"role": "user", "content": "武汉楼市新政 2026"}],
  "search_source": "baidu_search_v1",
  "resource_type_filter": [{"type": "web", "top_k": 5}],
  "stream": false
}
```
响应 `references[]` 含 title/url/content/snippet/date，质量高（政府官网+官方媒体优先返回）。
