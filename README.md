# Hermes Agent + OpenClaw 飞书部署踩坑指南

> 在同一台 Windows 机器上同时部署 OpenClaw（小助理）和 Hermes Agent（海马士），各自使用独立的飞书应用，互不冲突。

## 背景

- **OpenClaw**：AI 助手，飞书 + QQ 机器人，端口 18789
- **Hermes Agent**：AI 智能体框架，支持飞书/Telegram/Discord 等多平台，端口 18790
- 两个服务需要共存，各自有独立的飞书应用

## 踩过的坑（血泪教训）

### 坑1：config.yaml 覆盖 .env

**现象**：明明 .env 配的是正确的飞书应用，但 Hermes 就是连到错误的应用。

**原因**：Hermes 的 `feishu.py` 读取配置时，**config.yaml 优先级高于 .env**：

```python
app_id=str(extra.get("app_id") or os.getenv("FEISHU_APP_ID", "")).strip(),
```

`extra.get("app_id")` 来自 config.yaml，`os.getenv("FEISHU_APP_ID")` 来自 .env。如果 config.yaml 里写了 `app_id`，.env 的配置会被忽略。

**解决**：config.yaml 的 `feishu` 段不要写 `app_id` 和 `app_secret`，只放 .env 里：

```yaml
# config.yaml - 正确写法
feishu:
  enabled: true
  connection_mode: websocket
  # app_id 和 app_secret 放 .env
```

```env
# .env
FEISHU_APP_ID=cli_xxxxx
FEISHU_APP_SECRET=xxxxx
```

### 坑2：飞书用户ID格式不匹配

**现象**：消息收到了，但回复被拦截：`Unauthorized user: g8c98291 (用户967150) on feishu`

**原因**：飞书有三种用户ID：
- `user_id`（租户级，如 `g8c98291`）— **Hermes 优先用这个**
- `open_id`（应用级，如 `ou_xxxxxxx`）
- `union_id`（开发者级）

Hermes 的 `_resolve_sender_identity` 方法优先使用 `user_id`，但 .env 里配的是 `open_id`。

**解决**：直接允许所有用户，省得折腾 ID：

```env
FEISHU_ALLOW_ALL_USERS=true
```

### 坑3：飞书事件订阅"加了不生效"

**现象**：飞书后台已经添加了 `im.message.receive_v1` 事件，也发布了版本，但就是收不到消息。WebSocket 连接成功，ping/pong 正常，零事件。

**原因**：飞书的已知 Bug —— 新建应用的事件订阅有时需要"重置"才能生效。

**解决**：先删后加，每步都发布版本：

1. 删除 `im.message.receive_v1` 事件
2. **发布新版本**（带着空事件发布）
3. 重新添加 `im.message.receive_v1` 事件
4. **再发布新版本**

这个方法亲测有效，多个 AI 工具（Claude Code、Aider、Cursor、DeepSeek）都没排查出来的问题，靠这个方法解决了。

### 坑4：多个进程抢同一个飞书应用

**现象**：Hermes 连上了但收不到消息，日志里没有任何事件。

**原因**：飞书的 WebSocket 长连接**一个应用只能有一个连接**。如果有测试脚本、旧进程还在用同一个应用的凭据连接，新连接会挤掉旧连接，或者旧连接抢占了事件推送。

**解决**：启动前确保没有其他进程在用同一个飞书应用：

```powershell
# 查看所有 Python 进程
tasklist | findstr python

# 杀掉测试脚本
taskkill /PID <pid> /F
```

## 最小配置清单

### 飞书应用权限（13个）

| 权限 | 用途 |
|------|------|
| `im:message` | 收发消息 |
| `im:message:send` | 发送消息 |
| `im:resource` | 下载图片/文件 |
| `im:image` | 上传图片 |
| `im:file` | 上传文件 |
| `im:chat` | 读取群信息 |
| `im:message.reaction` | 表情回应 |
| `contact:user.base:readonly` | 获取用户名 |
| `application:bot.basic_info:read` | 机器人信息 |
| `application:application:readonly` | 应用信息 |
| `docs:doc:readonly` | 读文档 |
| `drive:drive:readonly` | 读云盘文件 |
| `wiki:node:readonly` | 读知识库 |

### 飞书事件订阅（10个）

| 事件 | 用途 |
|------|------|
| `im.message.receive_v1` | **必须** 收到消息 |
| `im.message.message_read_v1` | 已读回执 |
| `im.message.reaction.created_v1` | 表情回应添加 |
| `im.message.reaction.deleted_v1` | 表情回应删除 |
| `im.chat.member.bot.added_v1` | 机器人被拉群 |
| `im.chat.member.bot.deleted_v1` | 机器人被移出群 |
| `im.chat_access_event.bot_p2p_chat_entered_v1` | 用户进入私聊 |
| `im.message.recalled_v1` | 消息撤回 |
| `card.action.trigger` | 卡片按钮点击 |
| `drive.notice.comment_add_v1` | 文档评论@机器人 |

### .env 最小配置

```env
# 模型配置
OPENAI_API_KEY=your-api-key
OPENAI_BASE_URL=https://your-api-endpoint/v1

# 飞书配置
FEISHU_APP_ID=cli_xxxxx
FEISHU_APP_SECRET=xxxxx
FEISHU_DOMAIN=feishu
FEISHU_CONNECTION_MODE=websocket
FEISHU_ALLOW_ALL_USERS=true
FEISHU_GROUP_POLICY=open

# API 服务器（与 OpenClaw 的 18789 分离）
API_SERVER_ENABLED=true
API_SERVER_PORT=18790
```

### config.yaml 最小配置

```yaml
model:
  provider: custom
  model: your-model-name
  base_url: https://your-api-endpoint/v1
  api_key: your-api-key

feishu:
  enabled: true
  connection_mode: websocket
  # 不要在这里写 app_id 和 app_secret！
```

## 两个服务共存的关键

| 项目 | OpenClaw（小助理） | Hermes（海马士） |
|------|-------------------|-----------------|
| 飞书应用 | cli_a9264fb4c3f8dcce | cli_aa89b8fef5bddcb6 |
| 端口 | 18789 | 18790 |
| 运行方式 | Node.js | Python |

**核心原则**：不同的飞书应用 + 不同的端口 = 互不冲突。

## 测试连通性

写一个最小测试脚本验证飞书应用能否收到消息：

```python
import sys
sys.path.insert(0, r'path\to\hermes\venv\Lib\site-packages')

from lark_oapi.ws import Client as FeishuWSClient
from lark_oapi.event.dispatcher_handler import EventDispatcherHandler

APP_ID = "cli_xxxxx"
APP_SECRET = "xxxxx"

def on_message(data):
    print(f"[MSG] {data}", flush=True)

handler = (
    EventDispatcherHandler.builder("", "")
    .register_p2_im_message_receive_v1(on_message)
    .build()
)

ws_client = FeishuWSClient(APP_ID, APP_SECRET, event_handler=handler)
print("等待消息...", flush=True)
ws_client.start()
```

如果看到 `[MSG]` 输出，说明事件订阅生效了。

## 排查顺序

遇到飞书收不到消息时，按这个顺序排查：

1. **检查进程**：有没有其他进程在用同一个飞书应用？
2. **检查配置**：config.yaml 有没有覆盖 .env？
3. **检查事件**：事件订阅是否已发布新版本？
4. **先删后加**：删掉事件 → 发布 → 重新添加 → 发布
5. **检查权限**：`im:message` 是否已开通？
6. **检查可用范围**：应用是否发布了？

## 致谢

踩坑过程中得到了以下 AI 工具的帮助（虽然都没能完全解决问题）：
- Claude Code（最终解决了 config.yaml 优先级问题）
- Aider
- Cursor
- Cline
- DeepSeek
- 小助理（OpenClaw）

最终靠"先删后加事件"这个飞书玄学操作搞定了。

## 许可

MIT License
