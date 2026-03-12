# OpenClaw "pairing required" 问题排查与解决

## 问题描述

登录 OpenClaw 时提示 `pairing required`，无法通过 Web UI 或 CLI 访问 Gateway。

### 错误日志

```
closed before connect conn=xxx code=1008 reason=pairing required
```

### 环境
- OpenClaw 版本：2026.3.2
- 部署方式：Docker Compose
- 镜像：alpine/openclaw:latest

---

## 排查过程

### 1. 检查 Gateway 状态
```bash
docker compose ps
docker logs openclaw-in-docker-openclaw-gateway-1 --tail 100
```

发现日志中有大量 `pairing required` 错误。

### 2. 检查配对状态
```bash
cat ~/.openclaw/devices/paired.json    # 空的：{}
cat ~/.openclaw/devices/pending.json   # 有待配对设备
```

### 3. 尝试 CLI 连接
```bash
docker compose run --rm openclaw-cli devices
```

**错误：** `gateway token mismatch (set gateway.remote.token to match gateway.auth.token)`

### 4. 检查配置文件
```bash
cat ~/.openclaw/openclaw.json | jq '.gateway'
cat /home/ns/code/openclaw-in-docker/.env
```

---

## 根本原因

**问题1：设备未配对**
- `paired.json` 为空
- `pending.json` 中有待配对的 Windows 设备

**问题2：Token 配置不完整**
- `.env` 和 `gateway.auth.token` 已统一
- **但 `gateway.remote.token` 未设置**（这是 CLI 连接时使用的配置项）

---

## 解决方案

### 步骤 1：添加 `remote.token` 配置

编辑 `~/.openclaw/openclaw.json`，在 `gateway` 部分添加：

```json
{
  "gateway": {
    "auth": {
      "mode": "token",
      "token": "40244d02e87e29b0de259d9bc155933b3f5654cf88624de1b36f9587687fde24"
    },
    "remote": {
      "token": "40244d02e87e29b0de259d9bc155933b3f5654cf88624de1b36f9587687fde24"
    },
    ...
  }
}
```

**关键点：** `gateway.auth.token` 和 `gateway.remote.token` 必须一致。

### 步骤 2：重启 Gateway

```bash
cd /home/ns/code/openclaw-in-docker
docker compose restart openclaw-gateway
```

### 步骤 3：批准设备配对

**方法1：交互式菜单（推荐）**
```bash
docker compose run --rm openclaw-cli devices
```

在菜单中：
1. 找到 Pending devices
2. 选择待配对的设备
3. 选择 Approve

**方法2：直接命令批准**
```bash
# 先查看待配对设备的 requestId
cat ~/.openclaw/devices/pending.json | jq 'keys[]'

# 批准指定设备（替换 <request-id>）
docker compose run --rm openclaw-cli devices approve <request-id>
```

### 步骤 4：验证配对成功

```bash
cat ~/.openclaw/devices/paired.json
```

应该能看到已配对的设备信息。

---

## 相关命令

### 查看设备配对状态
```bash
# 待配对设备
cat ~/.openclaw/devices/pending.json

# 已配对设备
cat ~/.openclaw/devices/paired.json
```

### 查看 Gateway 配置
```bash
# Gateway 认证配置
cat ~/.openclaw/openclaw.json | jq '.gateway.auth'

# 远程连接配置
cat ~/.openclaw/openclaw.json | jq '.gateway.remote'

# 绑定模式
cat ~/.openclaw/openclaw.json | jq '.gateway.bind'
```

### 查看 Gateway 日志
```bash
# 实时日志
docker compose logs -f openclaw-gateway

# 最近 50 行
docker compose logs --tail=50 openclaw-gateway

# 过滤 token/配对相关错误
docker compose logs openclaw-gateway | grep -i "token\|pairing"
```

### 检查容器状态
```bash
docker compose ps
docker port openclaw-in-docker-openclaw-gateway-1
```

---

## 配置说明

### Gateway Token 优先级

OpenClaw CLI 连接 Gateway 时，token 配置的优先级：

1. **`gateway.remote.token`** (CLI 使用这个连接远程 Gateway)
2. 环境变量 `OPENCLAW_GATEWAY_TOKEN`
3. `gateway.auth.token` (Gateway 本身使用的 token)

**重要：** 当 CLI 容器连接 Gateway 时，必须设置 `gateway.remote.token`。

### Bind 模式

- `loopback`: 仅监听 127.0.0.1（本地访问）
- `lan`: 监听 0.0.0.0（局域网访问）
- `custom`: 自定义地址

确保 `.env` 中的 `OPENCLAW_GATEWAY_BIND` 和 `openclaw.json` 中的 `gateway.bind` 一致。

---

## 故障排查清单

- [ ] 检查 `~/.openclaw/devices/paired.json` 是否为空
- [ ] 检查 `~/.openclaw/devices/pending.json` 是否有待配对设备
- [ ] 检查 `gateway.auth.token` 和 `gateway.remote.token` 是否一致
- [ ] 检查 `.env` 文件中的 `OPENCLAW_GATEWAY_TOKEN` 是否与配置文件一致
- [ ] 检查 Gateway 容器是否正常运行（`docker compose ps`）
- [ ] 检查 Gateway 日志是否有 `token mismatch` 错误

---

## 参考

- OpenClaw 官方文档：https://docs.openclaw.ai
- 配置文件位置：`~/.openclaw/openclaw.json`
- 设备配对目录：`~/.openclaw/devices/`
- Web UI 访问：`http://<服务器IP>:18789`

---

**最后更新：** 2026-03-12
**问题状态：** ✅ 已解决
