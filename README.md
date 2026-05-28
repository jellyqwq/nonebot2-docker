# NoneBot2 + NapCat Docker 部署

基于 Docker Compose 一键部署 [NoneBot2](https://github.com/nonebot/nonebot2) 和 [NapCat](https://github.com/NapNeko/NapCatQQ)，两个容器在同一网络内互联。

## 项目结构

```
├── Dockerfile           # 镜像：python:3.13-slim + nb-cli
├── docker-compose.yaml  # 三个服务：init / nonebot2 / napcat
├── mortis-bot/          # NoneBot 项目代码（由 init 生成）
├── napcat/config/       # NapCat 配置持久化
├── napcat/QQ/           # QQ 登录态持久化
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

# 重启
docker compose restart nonebot2
```

## 服务说明

| 服务 | 用途 | 启动命令 |
|------|------|----------|
| `init` | 交互式创建 NoneBot 项目脚手架（一次性） | `docker compose --profile init run --rm init` |
| `nonebot2` | 运行 NoneBot 机器人 | `docker compose up -d nonebot2` |
| `napcat` | QQ 协议适配器 | `docker compose up -d napcat` |

## 端口

| 端口 | 服务 | 用途 |
|------|------|------|
| 8080 | nonebot2 | NoneBot HTTP/WebSocket |
| 5680 | nonebot2 | NoneBot 额外端口 |
| 6099 | napcat | Web 管理面板 |

容器间通过 `bot-net` 网络互通，NoneBot 用 `ws://napcat:3000` 连接 NapCat。

## 参考

- [NapCat-Docker](https://github.com/NapNeko/NapCat-Docker)
- [NoneBot 配置文档](https://nonebot.dev/docs/appendices/config)
- [nonebot2 插件](https://github.com/ByteColtX/nonebot2)
