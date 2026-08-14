# API Key 凭据池切换：往返验证完整记录（2026-08-14）

> 场景：用户要求从 key2 切到 key3（付费模型）。此前承诺修复但未验证，用户严重不满。
> 本文件记录**已验证的正确流程**，含假修复根因。

## 假修复根因（为什么"切了没生效"）

1. **改错地方**：只改了 `/path/to/data/.env` 的 `TOKENRHYTHM_API_KEY`，但 Hermes 运行时走**凭据池**（`hermes auth add` 注册的），`.env` 只在池为空时才读。
2. **脚本不去重**：`switch_key3.py` 是"遍历替换"逻辑，每个匹配行都替换成新 key，但**不删除多余重复行** → .env 里累积 3 行同名 `TOKENRHYTHM_API_KEY`（27/28/33行），加载器取哪行不确定。
3. **没做往返验证**：只验证"写入成功"，没验证"运行时生效"。

## 正确切换流程（已验证）

### Step 1: 确认目标 key 有效（curl 健康检查）

```bash
KEY3=$(cat /path/to/data/backup_api_key_3.txt)
curl -s -o /dev/null -w "HTTP:%{http_code}\n" https://tokenrhythm.studio/v1/models -H "Authorization: Bearer $KEY3" --connect-timeout 10
# 200 = 有效；402 = 余额耗尽；429 = 限流
```

### Step 2: 查看凭据池当前状态

```bash
export PATH=$PATH:/opt/hermes/bin
hermes auth list tokenrhythm
# ← 标记 = 当前生效；exhausted = 该 key 已耗尽会跳过
```

### Step 3: 剔除耗尽/坏 key

```bash
hermes auth remove tokenrhythm tr-key2    # 402 已耗尽的 key 剔除，避免轮换踩雷
```

### Step 4: 让目标 key 生效（重排池顺序）

凭据池无 `use`/`set` 命令，生效规则 = 池顺序第一个未耗尽的 key。

```bash
# 移出其他 key → ← 自动落在剩余 key
hermes auth remove tokenrhythm tr-key4
hermes auth list tokenrhythm    # 只剩 tr-key3，← 在它上 = key3 生效

# 加回备选 key（排到 #2）
KEY4=$(cat /path/to/data/backup_api_key_4.txt)
hermes auth add tokenrhythm --type api-key --label tr-key4 --api-key "$KEY4"
```

### Step 5: 往返验证（关键！）

```bash
# 切到 key4 → 验证 → 切回 key3 → 验证
python3 /path/to/data/switch_key_robust.py backup_api_key_4.txt   # .env 通道
hermes auth list tokenrhythm                                  # 池通道
python3 /path/to/data/switch_key_robust.py backup_api_key_3.txt   # 切回
hermes auth list tokenrhythm
```

## 幂等去重脚本模式（switch_key_robust.py 核心逻辑）

```python
# 1. 读 .env 全部行
# 2. 删除所有 startswith("TOKENRHYTHM_API_KEY") 的行
# 3. 追加一行 f"TOKENRHYTHM_API_KEY={new_key}\n"
# 4. 验证 len(remaining)==1 and remaining[0] == new_key
```

⚠️ 禁止裸 `sed`/`echo >>` 追加 key——会累积重复行（已实测 3 行）。

## 双重保障架构（2026-08-14 落地）

| 层 | 机制 | 状态 |
|----|------|------|
| 凭据池自动轮换 | key3(←生效) → 报错自动切 key4 | ✅ tr-key3 #1 / tr-key4 #2 / tr-key2 已剔除 |
| .env 手动通道 | `switch_key_robust.py <keyfile>` 一键切换 | ✅ 唯一 1 行 key3，往返测试通过 |

用户明确要求：**轮换不能碰到已耗尽的 key**（剔除），且**切换成功后必须验证生效**（否则下次又踩坑）。
