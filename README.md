<div align="center">

# hermes-config-workflow

**Hermes Agent 配置改法：别直接改 config.yaml，用 `hermes config set`。**

<p>
  <a href="#"><img src="https://img.shields.io/badge/critical-don%27t--edit--yaml-red" alt="Don't edit YAML" /></a>
</p>

## 关键规则

Hermes 把行为设置存在 `~/.hermes/config.yaml`，密钥在 `~/.hermes/.env`。

**不要直接用 patch/write_file 改 config.yaml**——Hermes 会阻止。正确做法：

```bash
hermes config set <key> <value>
```

## 什么时候用

- 用户要开/关某个功能
- 用户给了 API key 要接上
- 换模型/换搜索源

## License

MIT
