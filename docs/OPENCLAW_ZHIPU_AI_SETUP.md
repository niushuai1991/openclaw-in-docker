# OpenClaw 配置智谱 AI (Z.AI) GLM-4.7 指南

## 问题描述

**目标**: 配置智谱 AI (Z.AI) GLM-4.7 Coding Plan 模型

**环境**: 
- Docker Compose 部署 OpenClaw
- 镜像: ghcr.io/openclaw/openclaw:latest

**初始状态**: Gateway 可正常启动，但缺少模型配置

---

## 快速配置步骤

### 步骤 1：获取 API Key

访问智谱 AI 控制台创建 API Key：
- 地址: https://bigmodel.cn/usercenter/proj-mgmt/apikeys
- 选择: Coding Plan（建议使用 Coding Plan Lite）

### 步骤 2：添加环境变量

编辑 `.env` 文件：

```bash
cd /home/ns/code/openclaw-in-docker
echo "ZAI_API_KEY=your-api-key-here" >> .env
```

替换 `your-api-key-here` 为你的实际 API Key。

### 步骤 3：修改 docker-compose.yml

在 `docker-compose.yml` 的两个服务（`openclaw-gateway` 和 `openclaw-cli`）的 `environment` 部分添加：

```yaml
environment:
  HOME: /home/node
  TERM: xterm-256color
  OPENCLAW_GATEWAY_TOKEN: ${OPENCLAW_GATEWAY_TOKEN}
  ZAI_API_KEY: ${ZAI_API_KEY}  # 添加这一行
  CLAUDE_AI_SESSION_KEY: ${CLAUDE_AI_SESSION_KEY}
  CLAUDE_WEB_SESSION_KEY: ${CLAUDE_WEB_SESSION_KEY}
  CLAUDE_WEB_COOKIE: ${CLAUDE_WEB_COOKIE}
```

### 步骤 4：运行 onboarding 配置认证

```bash
docker compose run --rm openclaw-cli onboard --non-interactive --accept-risk \
  --auth-choice zai-coding-cn \
  --zai-api-key "your-api-key-here"
```

**注意**: 
- `--accept-risk` 是必需的（安全要求）
- `zai-coding-cn` 是中国版 endpoint

### 步骤 5：修正模型配置（重要）

onboarding 可能会将模型设置为 `glm-5`，但 **Coding Plan Lite 不支持 GLM-5**，会导致速率限制错误。

修改为 GLM-4.7：

```bash
# 修改主模型
docker compose exec openclaw-gateway openclaw config set agents.defaults.model.primary "zai/glm-4.7"

# 添加模型别名
docker compose exec openclaw-gateway openclaw config set agents.defaults.models["zai/glm-4.7"].alias "GLM-4.7"
```

或者直接编辑配置文件：

```bash
nano /home/ns/.openclaw/openclaw.json
```

修改 `agents.defaults.model.primary` 为 `"zai/glm-4.7"`

### 步骤 6：重启 Gateway

```bash
docker compose restart openclaw-gateway
```

---

## 关键问题和解决方案

### 问题 1：缺少 API key 认证

**错误信息**:
```
No API key found for provider "zai". 
Auth store: /home/node/.openclaw/agents/main/agent/auth-profiles.json
```

**原因**: 
- `auth-profiles.json` 文件不存在
- 或 API key 未正确配置

**解决方案**: 
运行 onboarding 命令（步骤 4），会自动创建 `auth-profiles.json` 并配置认证。

---

### 问题 2：onboarding 需要 --accept-risk

**错误信息**:
```
Non-interactive onboarding requires explicit risk acknowledgement.
Re-run with: openclaw onboard --non-interactive --accept-risk ...
```

**原因**: 
安全要求，非交互式模式必须明确接受风险。

**解决方案**: 
在 onboarding 命令中添加 `--accept-risk` 参数。

---

### 问题 3：API 速率限制（关键）

**错误信息**:
```
⚠️ API rate limit reached. Please try again later.
```

**原因**: 
- **Coding Plan Lite 不支持 GLM-5**
- onboarding 默认可能将模型设置为 `glm-5`
- 导致 API 调用被速率限制

**解决方案**: 
手动修改模型配置为 `glm-4.7`（步骤 5）

**检查当前模型**:
```bash
docker compose logs openclaw-gateway --tail 20 | grep "agent model"
```

如果显示 `agent model: zai/glm-5`，需要修改为 `zai/glm-4.7`

---

## 验证配置

### 检查模型状态

```bash
docker compose exec openclaw-gateway openclaw models status
```

**预期输出**:
```
Config        : ~/.openclaw/openclaw.json
Agent dir     : ~/.openclaw/agents/main/agent
Default       : zai/glm-4.7
Fallbacks (0) : -
Aliases (1)   : GLM-4.7 -> zai/glm-4.7

Auth overview
Auth store    : ~/.openclaw/agents/main/agent/auth-profiles.json
Providers w/ OAuth/tokens (0): -
- zai effective=profiles:~/.openclaw/agents/main/agent/auth-profiles.json | profiles=1 (oauth=0, token=0, api_key=1) | zai:default=29e51688...RvdW7jTz
```

关键点：
- ✅ Default: `zai/glm-4.7`
- ✅ Auth: `zai (configured)`

### 查看 Gateway 日志

```bash
docker compose logs openclaw-gateway --tail 20 | grep "agent model"
```

**预期输出**:
```
[gateway] agent model: zai/glm-4.7
```

### 检查认证文件

```bash
docker compose exec openclaw-gateway cat /home/node/.openclaw/agents/main/agent/auth-profiles.json
```

应该包含 `zai:default` 的 API key 配置。

---

## 配置文件说明

| 文件 | 作用 | 关键配置 |
|------|------|----------|
| `/home/ns/code/openclaw-in-docker/.env` | 环境变量 | `ZAI_API_KEY=sk-...` |
| `/home/ns/code/openclaw-in-docker/docker-compose.yml` | 容器环境变量传递 | `ZAI_API_KEY: ${ZAI_API_KEY}` |
| `~/.openclaw/openclaw.json` | 主配置 | `agents.defaults.model.primary: "zai/glm-4.7"` |
| `~/.openclaw/agents/main/agent/auth-profiles.json` | 认证配置 | `profiles.zai:default.apiKey` |
| `~/.openclaw/agents/main/agent/models.json` | Provider 配置 | `providers.zai.baseUrl`, `models` |

### 重要配置文件内容

**openclaw.json (agents.defaults)**:
```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "zai/glm-4.7"
      },
      "models": {
        "zai/glm-4.7": {
          "alias": "GLM-4.7"
        }
      }
    }
  }
}
```

**auth-profiles.json**:
```json
{
  "version": 1,
  "profiles": {
    "zai:default": {
      "type": "api_key",
      "provider": "zai",
      "key": "your-api-key-here"
    }
  }
}
```

**models.json (Provider 配置)**:
```json
{
  "providers": {
    "zai": {
      "baseUrl": "https://open.bigmodel.cn/api/coding/paas/v4",
      "api": "openai-completions",
      "apiKey": "your-api-key-here",
      "models": [
        {
          "id": "glm-4.7",
          "name": "GLM-4.7"
        }
      ]
    }
  }
}
```

---

## 常见问题 (FAQ)

### Q: 为什么不能用 GLM-5？

**A**: Coding Plan Lite **不支持 GLM-5**。如果使用 GLM-5，会遇到：
```
⚠️ API rate limit reached. Please try again later.
```

**解决方案**: 使用 GLM-4.7 或其他支持的模型：
- `zai/glm-4.7`
- `zai/glm-4.7-flash`
- `zai/glm-4.7-flashx`

---

### Q: 如何查看 API 配额和使用情况？

**A**: 登录智谱 AI 控制台查看：
- 地址: https://open.bigmodel.cn/
- 路径: 控制台 → API 密钥 → 使用情况

---

### Q: 配置后仍报 "No API key found for provider 'zai'"？

**A**: 检查以下几点：

1. **环境变量是否存在**:
   ```bash
   docker compose exec openclaw-gateway printenv | grep ZAI_API_KEY
   ```

2. **docker-compose.yml 是否传递环境变量**:
   ```bash
   grep -A 5 "environment:" docker-compose.yml | grep ZAI_API_KEY
   ```

3. **auth-profiles.json 是否创建成功**:
   ```bash
   ls -la ~/.openclaw/agents/main/agent/auth-profiles.json
   cat ~/.openclaw/agents/main/agent/auth-profiles.json
   ```

4. **重启容器**:
   ```bash
   docker compose restart openclaw-gateway
   ```

---

### Q: 如何切换到其他 GLM 模型？

**A**: 修改 `openclaw.json` 中的 `agents.defaults.model.primary`：

**可用模型**:
- `zai/glm-5` (需要完整 Coding Plan)
- `zai/glm-4.7` (推荐，Coding Plan Lite 支持)
- `zai/glm-4.7-flash` (快速版本)
- `zai/glm-4.7-flashx` (超快速版本)

**修改方法**:
```bash
# 方法 1: 使用 config 命令
docker compose exec openclaw-gateway openclaw config set agents.defaults.model.primary "zai/glm-4.7-flash"

# 方法 2: 直接编辑文件
nano ~/.openclaw/openclaw.json
# 修改 agents.defaults.model.primary 的值

# 重启 gateway
docker compose restart openclaw-gateway
```

---

### Q: onboarding 创建的模型配置不对怎么办？

**A**: onboarding 可能会将模型设置为 `glm-5`，需要手动修改：

```bash
# 检查当前模型
docker compose exec openclaw-gateway openclaw config get agents.defaults.model.primary

# 如果显示 glm-5，修改为 glm-4.7
docker compose exec openclaw-gateway openclaw config set agents.defaults.model.primary "zai/glm-4.7"

# 添加别名
docker compose exec openclaw-gateway openclaw config set agents.defaults.models["zai/glm-4.7"].alias "GLM-4.7"

# 删除旧的 glm-5 配置（可选）
docker compose exec openclaw-gateway openclaw config unset agents.defaults.models["zai/glm-5"]

# 重启 gateway
docker compose restart openclaw-gateway
```

---

### Q: 如何测试模型是否工作？

**A**: 有几种方法：

**方法 1: 通过 WebChat**
```bash
# 访问 Web UI
echo "打开浏览器访问: http://localhost:18789"
```

**方法 2: 查看日志**
```bash
# 实时查看日志
docker compose logs -f openclaw-gateway

# 发送消息后，检查是否有速率限制错误
docker compose logs openclaw-gateway | grep "rate limit"
```

**方法 3: 使用 CLI** (如果已配置 channel)
```bash
docker compose run --rm openclaw-cli message "测试消息"
```

---

## 相关命令参考

### 查看模型状态

```bash
# 完整状态
docker compose exec openclaw-gateway openclaw models status

# 只看主模型
docker compose exec openclaw-gateway openclaw config get agents.defaults.model.primary

# 查看所有可用模型
docker compose exec openclaw-gateway openclaw models list
```

### 修改模型配置

```bash
# 设置主模型
docker compose exec openclaw-gateway openclaw config set agents.defaults.model.primary "zai/glm-4.7"

# 添加模型别名
docker compose exec openclaw-gateway openclaw config set agents.defaults.models["zai/glm-4.7"].alias "GLM-4.7"

# 删除模型配置
docker compose exec openclaw-gateway openclaw config unset agents.defaults.models["zai/glm-5"]
```

### 查看认证信息

```bash
# 查看认证状态
docker compose exec openclaw-gateway openclaw models status | grep -A 10 "Auth overview"

# 查看认证文件
docker compose exec openclaw-gateway cat /home/node/.openclaw/agents/main/agent/auth-profiles.json

# 查看环境变量
docker compose exec openclaw-gateway printenv | grep ZAI
```

### 查看日志

```bash
# 实时日志
docker compose logs -f openclaw-gateway

# 最近 50 行
docker compose logs openclaw-gateway --tail 50

# 过滤模型相关
docker compose logs openclaw-gateway | grep "agent model"

# 过滤错误
docker compose logs openclaw-gateway | grep -i "error\|rate limit"
```

### 服务管理

```bash
# 重启 gateway
docker compose restart openclaw-gateway

# 停止服务
docker compose down

# 启动服务
docker compose up -d

# 查看状态
docker compose ps
```

---

## 变更历史

| 日期 | 版本 | 变更内容 |
|------|------|----------|
| 2026-03-13 | 1.0 | 初始版本，智谱 AI GLM-4.7 配置指南 |

---

## 参考

- 智谱 AI 官网: https://open.bigmodel.cn/
- API Key 管理: https://bigmodel.cn/usercenter/proj-mgmt/apikeys
- OpenClaw 官方文档: https://docs.openclaw.ai
- OpenClaw Z.AI Provider 文档: https://docs.openclaw.ai/providers/zai

---

**最后更新**: 2026-03-13  
**文档版本**: 1.0  
**维护者**: OpenClaw Docker 项目组
