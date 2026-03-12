# OpenClaw "pairing required" 问题排查与解决

## 问题描述

登录 OpenClaw 时提示 `pairing required`，无法通过 Web UI 或 CLI 访问 Gateway。

### 错误日志

```
closed before connect conn=xxx code=1008 reason=pairing required
```

### 环境
- OpenClaw 版本：2026.3.2 → 2026.3.8
- 部署方式：Docker Compose
- 镜像：ghcr.io/openclaw/openclaw:latest

---

## 排查过程

### 1. 检查 Gateway 状态
```bash
docker compose ps
docker compose logs openclaw-gateway --tail 100
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

**常见错误：**
- `gateway token mismatch (set gateway.remote.token to match gateway.auth.token)`
- `gateway closed (1006 abnormal closure)`

### 4. 检查配置文件
```bash
# 检查配置文件中的 token
cat ~/.openclaw/openclaw.json | jq '.gateway'

# 检查环境变量中的 token
cat /home/ns/code/openclaw-in-docker/.env | grep OPENCLAW_GATEWAY_TOKEN

# 检查是否有端口冲突
docker ps -a | grep openclaw
ss -tuln | grep 18789
```

---

## 根本原因

**问题1：设备未配对**
- `paired.json` 为空
- `pending.json` 中有待配对的设备

**问题2：Token 配置不完整或不一致**
- `gateway.remote.token` 未设置（CLI 连接时使用）
- `.env` 中的 `OPENCLAW_GATEWAY_TOKEN` 与 `openclaw.json` 中的 token 不一致

**问题3：CLI WebSocket 连接失败**
- Gateway 未完全启动
- 容器网络隔离问题
- Token 配置错误导致认证失败

---

## 解决方案

### 方案 A：标准流程（CLI 命令）- 推荐首选

#### 步骤 1：添加 `remote.token` 配置

编辑 `~/.openclaw/openclaw.json`，在 `gateway` 部分添加：

```json
{
  "gateway": {
    "auth": {
      "token": "your-token-here"
    },
    "remote": {
      "token": "your-token-here"
    }
  }
}
```

**关键点：** `gateway.auth.token` 和 `gateway.remote.token` 必须一致。

#### 步骤 2：确保 Token 一致性

```bash
# 检查 token 是否一致
ENV_TOKEN=$(grep OPENCLAW_GATEWAY_TOKEN .env | cut -d= -f2)
CONFIG_TOKEN=$(cat ~/.openclaw/openclaw.json | jq -r '.gateway.auth.token')

echo "Env token: $ENV_TOKEN"
echo "Config token: $CONFIG_TOKEN"

# 如果不一致，修改 .env 文件
vim .env
# 确保 OPENCLAW_GATEWAY_TOKEN 与 openclaw.json 中的 token 一致
```

#### 步骤 3：重启 Gateway

```bash
cd /home/ns/code/openclaw-in-docker
docker compose restart openclaw-gateway

# 等待 Gateway 完全启动（健康检查通过）
sleep 5
docker compose ps
```

#### 步骤 4：批准设备配对

**方法1：交互式菜单**
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

#### 步骤 5：验证配对成功

```bash
cat ~/.openclaw/devices/paired.json
```

应该能看到已配对的设备信息。

---

### 方案 B：手动配对（CLI 失败时的备选方案）

**适用场景：** CLI 命令因网络、容器或其他原因无法使用时

#### 步骤 1：查看待配对设备

```bash
cat ~/.openclaw/devices/pending.json | jq '.'
```

记下设备的 `requestId` 和 `deviceId`。

#### 步骤 2：手动移动设备到已配对列表

```bash
# 设置变量（替换为实际的值）
REQUEST_ID="858903fc-9b95-494e-b88c-5bfee1602e61"

# 提取设备信息并创建 paired.json 条目
cat ~/.openclaw/devices/pending.json | jq \
  ".[\"$REQUEST_ID\"] | {(.deviceId): {requestId, deviceId, publicKey, platform, clientId, clientMode, role, scopes}}" \
  > ~/.openclaw/devices/paired.json.tmp

# 如果 paired.json 已有设备，需要合并
if [ -s ~/.openclaw/devices/paired.json ] && [ "$(cat ~/.openclaw/devices/paired.json)" != "{}" ]; then
  jq -s '.[0] * .[1]' ~/.openclaw/devices/paired.json ~/.openclaw/devices/paired.json.tmp > ~/.openclaw/devices/paired.json.new
  mv ~/.openclaw/devices/paired.json.new ~/.openclaw/devices/paired.json
  rm ~/.openclaw/devices/paired.json.tmp
else
  mv ~/.openclaw/devices/paired.json.tmp ~/.openclaw/devices/paired.json
fi
```

#### 步骤 3：从 pending.json 删除已配对设备

```bash
REQUEST_ID="858903fc-9b95-494e-b88c-5bfee1602e61"

cat ~/.openclaw/devices/pending.json | jq "del(.[\"$REQUEST_ID\"])" > ~/.openclaw/devices/pending.json.tmp
mv ~/.openclaw/devices/pending.json.tmp ~/.openclaw/devices/pending.json
```

#### 步骤 4：重启 Gateway

```bash
docker compose restart openclaw-gateway
sleep 5
```

#### 步骤 5：验证配对成功

```bash
# 检查 paired.json
cat ~/.openclaw/devices/paired.json | jq '.'

# 检查 pending.json 应该为空或已删除该设备
cat ~/.openclaw/devices/pending.json | jq '.'

# 检查 Gateway 日志，确认没有 pairing required 错误
docker compose logs openclaw-gateway --tail=20 | grep -i "pairing"
```

---

## 常见错误及解决方案

### 错误 1：gateway closed (1006 abnormal closure)

**错误信息：**
```
CLI failed: Error: gateway closed (1006 abnormal closure): no close reason
```

**原因：** CLI 无法连接到 Gateway

**解决方案：**
```bash
# 1. 确认 Gateway 完全启动且健康
docker compose ps
docker compose logs openclaw-gateway --tail=20

# 2. 等待健康检查通过
sleep 10

# 3. 检查 token 一致性
grep OPENCLAW_GATEWAY_TOKEN .env
cat ~/.openclaw/openclaw.json | jq '.gateway.remote.token'

# 4. 如果仍然失败，使用手动配对方法（方案 B）
```

### 错误 2：token mismatch

**错误信息：**
```
gateway token mismatch (set gateway.remote.token to match gateway.auth.token)
```

**原因：** `.env` 和 `openclaw.json` 的 token 不一致

**解决方案：**
```bash
# 统一 token 配置
ENV_TOKEN=$(grep OPENCLAW_GATEWAY_TOKEN .env | cut -d= -f2)
CONFIG_TOKEN=$(cat ~/.openclaw/openclaw.json | jq -r '.gateway.auth.token')

# 修改 .env 使其与 openclaw.json 一致
sed -i "s/OPENCLAW_GATEWAY_TOKEN=.*/OPENCLAW_GATEWAY_TOKEN=$CONFIG_TOKEN/" .env

# 确保 gateway.remote.token 也存在
cat ~/.openclaw/openclaw.json | jq '.gateway.remote.token'

# 如果不存在，添加它
jq ".gateway.remote.token = \"$CONFIG_TOKEN\"" ~/.openclaw/openclaw.json > ~/.openclaw/openclaw.json.tmp
mv ~/.openclaw/openclaw.json.tmp ~/.openclaw/openclaw.json

# 重启 Gateway
docker compose restart openclaw-gateway
```

### 错误 3：No API key found for provider "anthropic"

**错误信息：**
```
Agent failed before reply: No API key found for provider "anthropic"
```

**原因：** OpenClaw Agent 缺少 Anthropic API key

**解决方案：**
```bash
# 方式1：通过环境变量配置（推荐）
echo "ANTHROPIC_API_KEY=sk-ant-your-key-here" >> .env
docker compose restart openclaw-gateway

# 方式2：创建 auth-profiles.json
mkdir -p ~/.openclaw/agents/main/agent
cat > ~/.openclaw/agents/main/agent/auth-profiles.json << EOF
{
  "profiles": {
    "anthropic": {
      "apiKey": "sk-ant-your-key-here"
    }
  }
}
EOF

docker compose restart openclaw-gateway
```

---

## 相关命令

### 查看设备配对状态
```bash
# 待配对设备
cat ~/.openclaw/devices/pending.json

# 已配对设备
cat ~/.openclaw/devices/paired.json

# 格式化显示
cat ~/.openclaw/devices/paired.json | jq '.'
```

### 查看 Gateway 配置
```bash
# Gateway 认证配置
cat ~/.openclaw/openclaw.json | jq '.gateway.auth'

# 远程连接配置
cat ~/.openclaw/openclaw.json | jq '.gateway.remote'

# 绑定模式
cat ~/.openclaw/openclaw.json | jq '.gateway.bind'

# 完整 gateway 配置
cat ~/.openclaw/openclaw.json | jq '.gateway'
```

### Token 一致性检查
```bash
# 快速检查 token 是否一致
diff <(grep OPENCLAW_GATEWAY_TOKEN .env | cut -d= -f2) \
     <(cat ~/.openclaw/openclaw.json | jq -r '.gateway.auth.token')

# 或使用 echo 对比
echo "Env: $(grep OPENCLAW_GATEWAY_TOKEN .env | cut -d= -f2)"
echo "Auth: $(cat ~/.openclaw/openclaw.json | jq -r '.gateway.auth.token')"
echo "Remote: $(cat ~/.openclaw/openclaw.json | jq -r '.gateway.remote.token')"
```

### 查看 Gateway 日志
```bash
# 实时日志
docker compose logs -f openclaw-gateway

# 最近 50 行
docker compose logs --tail=50 openclaw-gateway

# 过滤 token/配对相关错误
docker compose logs openclaw-gateway | grep -i "token\|pairing"

# 过滤 API key 相关错误
docker compose logs openclaw-gateway | grep -i "api.*key\|anthropic"
```

### 检查容器状态
```bash
# 查看所有容器
docker compose ps

# 查看 Gateway 健康状态
docker inspect openclaw-in-docker-openclaw-gateway-1 | jq '.[0].State.Health'

# 检查端口绑定
docker port openclaw-in-docker-openclaw-gateway-1
```

### 检查端口占用
```bash
# 检查 18789 端口占用
ss -tuln | grep 18789

# 或使用 lsof（如果可用）
sudo lsof -i :18789

# 查看所有 OpenClaw 相关容器
docker ps -a | grep openclaw
```

---

## 配置说明

### Gateway Token 优先级

OpenClaw CLI 连接 Gateway 时，token 配置的优先级：

1. **`gateway.remote.token`** (CLI 使用这个连接远程 Gateway)
2. 环境变量 `OPENCLAW_GATEWAY_TOKEN`
3. `gateway.auth.token` (Gateway 本身使用的 token)

**重要：**
- 当 CLI 容器连接 Gateway 时，优先使用 `gateway.remote.token`
- Gateway 容器启动时使用 `.env` 中的 `OPENCLAW_GATEWAY_TOKEN` 或 `gateway.auth.token`
- **三者必须保持一致**，否则会导致认证失败

### Bind 模式

- `loopback`: 仅监听 127.0.0.1（本地访问）
- `lan`: 监听 0.0.0.0（局域网访问）
- `custom`: 自定义地址

确保 `.env` 中的 `OPENCLAW_GATEWAY_BIND` 和 `openclaw.json` 中的 `gateway.bind` 一致。

### 多项目共存配置

如果有多个 OpenClaw 项目，建议：

1. **使用不同端口**：
   ```bash
   # 项目 A
   OPENCLAW_GATEWAY_PORT=18789

   # 项目 B
   OPENCLAW_GATEWAY_PORT=18790
   ```

2. **使用不同 token**：
   每个项目使用独立的 token，增强安全性

3. **统一管理**：
   考虑使用一个中心 Gateway，其他项目作为客户端连接

---

## 扩展故障排查清单

### 基础检查
- [ ] 检查 `~/.openclaw/devices/paired.json` 是否为空
- [ ] 检查 `~/.openclaw/devices/pending.json` 是否有待配对设备
- [ ] 检查 Gateway 容器是否正常运行（`docker compose ps`）

### Token 配置检查
- [ ] 检查 `gateway.auth.token` 是否存在
- [ ] 检查 `gateway.remote.token` 是否存在
- [ ] 检查 `gateway.auth.token` 和 `gateway.remote.token` 是否一致
- [ ] 检查 `.env` 文件中的 `OPENCLAW_GATEWAY_TOKEN` 是否与配置文件一致
- [ ] 运行 token 一致性检查命令

### 网络和端口检查
- [ ] 检查是否有其他 OpenClaw 项目占用端口（`docker ps -a | grep openclaw`）
- [ ] 检查端口 18789 是否被占用（`ss -tuln | grep 18789`）
- [ ] 检查 Gateway 健康状态（`docker inspect` 查看健康检查）
- [ ] 确认 Gateway bind 模式（loopback/lan/custom）

### 日志检查
- [ ] 检查 Gateway 日志是否有 `token mismatch` 错误
- [ ] 检查 Gateway 日志是否有 `pairing required` 错误
- [ ] 检查 Gateway 日志是否有 `API key` 相关错误
- [ ] 检查 Gateway 是否成功启动（`listening on ws://...`）

### CLI 连接检查
- [ ] 尝试 CLI 连接是否成功
- [ ] 如果 CLI 失败（1006 错误），尝试手动配对方法
- [ ] 检查容器网络是否正常

### API Key 检查（如果使用 Agent）
- [ ] 检查 `.env` 是否包含 `ANTHROPIC_API_KEY`
- [ ] 检查 `~/.openclaw/agents/main/agent/auth-profiles.json` 是否存在
- [ ] 检查 Gateway 日志是否有 API key 相关错误

---

## 预防措施

### 1. 初始化配置检查清单

新项目启动时，确保：

```bash
# 1. 检查 token 配置
cat ~/.openclaw/openclaw.json | jq '.gateway.auth, .gateway.remote'
grep OPENCLAW_GATEWAY_TOKEN .env

# 2. 检查端口占用
ss -tuln | grep 18789

# 3. 检查设备配对状态
cat ~/.openclaw/devices/paired.json
cat ~/.openclaw/devices/pending.json

# 4. 检查 API Key 配置（如果需要）
grep ANTHROPIC_API_KEY .env
```

### 2. 配置文件同步

创建脚本自动同步 token：

```bash
#!/bin/bash
# sync-token.sh

CONFIG_TOKEN=$(cat ~/.openclaw/openclaw.json | jq -r '.gateway.auth.token')
ENV_FILE=".env"

# 更新 .env 文件
if grep -q "OPENCLAW_GATEWAY_TOKEN" "$ENV_FILE"; then
  sed -i "s/OPENCLAW_GATEWAY_TOKEN=.*/OPENCLAW_GATEWAY_TOKEN=$CONFIG_TOKEN/" "$ENV_FILE"
else
  echo "OPENCLAW_GATEWAY_TOKEN=$CONFIG_TOKEN" >> "$ENV_FILE"
fi

# 确保 gateway.remote.token 存在
jq ".gateway.remote.token = \"$CONFIG_TOKEN\"" ~/.openclaw/openclaw.json > /tmp/openclaw.json.tmp
mv /tmp/openclaw.json.tmp ~/.openclaw/openclaw.json

echo "Token synchronized successfully"
```

### 3. 多项目端口管理

建议在项目根目录创建 `.env.local` 文件：

```bash
# .env.local
OPENCLAW_GATEWAY_PORT=18789
OPENCLAW_GATEWAY_BIND=loopback  # 仅本地开发
```

---

## 参考

- OpenClaw 官方文档：https://docs.openclaw.ai
- 配置文件位置：`~/.openclaw/openclaw.json`
- 设备配对目录：`~/.openclaw/devices/`
- Web UI 访问：`http://<服务器IP>:18789`
- Anthropic API Key 申请：https://console.anthropic.com

---

## 变更历史

| 日期 | 版本 | 变更内容 |
|------|------|----------|
| 2026-03-12 | 1.0 | 初始版本，基础 pairing required 问题解决 |
| 2026-03-13 | 2.0 | 添加 token 一致性检查、手动配对方法、常见错误解决方案、多项目共存配置 |

---

**最后更新：** 2026-03-13
**文档版本：** 2.0
**问题状态：** ✅ 已解决（包含多种解决方案）
**维护者：** OpenClaw Docker 项目组
