# NoneBot2 + NapCat Docker 部署

基于 Docker Compose 一键部署 [NoneBot2](https://github.com/nonebot/nonebot2) 和 [NapCat](https://github.com/NapNeko/NapCatQQ)，两个容器在同一网络内互联。

## 项目结构

```
├── Dockerfile           # 镜像：python:3.13-slim + nb-cli
├── Dockerfile.nanobot   # 镜像：python:3.13-slim + nanobot + channel-anon
├── docker-compose.yaml  # 服务：init / nonebot2 / napcat / nanobot
├── mortis-bot/          # NoneBot 项目代码（由 init 生成）
├── napcat/config/       # NapCat 配置持久化
├── napcat/QQ/           # QQ 登录态持久化
├── nanobot/config/      # nanobot 配置 + workspace 持久化
└── .env                 # 环境变量（可选）
```

## 首次使用

### 1. 配置环境变量

创建 `.env` 文件：

```bash
cat > .env << 'EOF'
NAPCAT_UID=$(id -u)
NAPCAT_GID=$(id -g)
EOF
```

> 如果不需要 NapCat 可以跳过，WARN 不影响 NoneBot 运行。

### 2. 创建 NoneBot 项目

```bash
BOT_PROJECT=mortis-bot docker compose --profile init run --rm init
```

交互式选择适配器和插件。`BOT_PROJECT` 默认为 `mortis-bot`，可省略。

### 3. 配置 NoneBot 连接 NapCat

在 `mortis-bot/.env` 中配置 WebSocket 地址：

```env
HOST=0.0.0.0
PORT=8080
NAPCAT_WS_URL=ws://napcat:3000
```

## 日常使用

```bash
# 启动
docker compose up -d nonebot2

# 如果还需要 NapCat
docker compose up -d

# 查看日志
docker compose logs -f nonebot2

# 重启 加入插件后也要重启
docker compose restart nonebot2
```

## 服务说明


| 服务       | 用途                                    | 启动命令                                      |
| ---------- | --------------------------------------- | --------------------------------------------- |
| `init`     | 交互式创建 NoneBot 项目脚手架（一次性） | `docker compose --profile init run --rm init` |
| `nonebot2` | 运行 NoneBot 机器人                     | `docker compose up -d nonebot2`               |
| `napcat`   | QQ 协议适配器                           | `docker compose up -d napcat`                 |
| `nanobot`  | AI agent，经 channel-anon 接入 NapCat   | `docker compose up -d nanobot`                |

## 端口


| 端口 | 服务     | 用途                   |
| ---- | -------- | ---------------------- |
| 8080 | nonebot2 | NoneBot HTTP/WebSocket |
| 5680 | nonebot2 | NoneBot 额外端口       |
| 6099 | napcat   | Web 管理面板           |
| 3001 | napcat   | OneBot v11 WS 服务端（供 nanobot 连入） |
| 8765 | nanobot  | WebUI（仅绑定 127.0.0.1） |

容器间通过 `bot-net` 网络互通，NoneBot 用 `ws://napcat:3000` 连接 NapCat。

## nanobot 部署

[nanobot](https://github.com/HKUDS/nanobot) 是轻量 AI agent runtime，通过
[nanobot-channel-anon](https://github.com/ByteColtX/nanobot-channel-anon) 插件接入同一个 NapCat，
实现 QQ 群内的 AI 对话与群管理。

> 连接方向与 nonebot2 相反：NapCat 以 WS *客户端*反向连 nonebot2，而 nanobot 作为 WS *客户端*
> 主动连 NapCat 的 WS *服务端*（`ws://napcat:3001`）。两者互不冲突。

### 1. 准备配置

```bash
cp nanobot/config/config.example.json nanobot/config/config.json
```

编辑 `nanobot/config/config.json`：

- `providers`：填入 LLM provider 的 `apiKey`（示例用 openrouter，可换成任意兼容 provider）
- `agents.defaults`：默认 provider 与 model
- `channels.websocket.token`：WebUI 访问 token（0.0.0.0 绑定时必填）
- `channels.anon.accessToken`：需与 NapCat WS 服务端的 token 一致（默认 `ciallo0721anon`）
- `channels.anon.allowFrom`：允许的 QQ / 群号；`[]` 表示**拒绝所有**，`["*"]` 表示全部放行

NapCat 侧已在 `onebot11_*.json` 的 `websocketServers` 开启 `3001` 端口，token 为 `ciallo0721anon`。
修改 token 时两边需保持一致，并重启 napcat 生效。

### 2. 构建与运行

```bash
# 构建镜像
docker compose build nanobot

# 确认插件已装好（应看到 Anon / channel: anon）
docker compose run --rm nanobot nanobot plugins list

# 启动
docker compose up -d nanobot

# 查看日志（确认 gateway 启动、anon channel 连上 NapCat）
docker compose logs -f nanobot
```

WebUI 在 `http://127.0.0.1:8765`，用 config 里的 token 进入。

## 参考

- [NapCat-Docker](https://github.com/NapNeko/NapCat-Docker)
- [NoneBot 配置文档](https://nonebot.dev/docs/appendices/config)
- [nonebot2 插件](https://github.com/ByteColtX/nonebot2)
