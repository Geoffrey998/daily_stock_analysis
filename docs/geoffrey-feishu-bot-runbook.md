# Geoffrey 飞书问答部署清单

这份清单面向不写代码的部署流程，只记录本项目当前推荐路线。

## 先记住三个结论

1. 不需要注册 Docker Hub。云平台可以直接从 GitHub 仓库构建，或者在云服务器本机构建镜像。
2. 飞书问答使用 Stream 长连接，不需要公网域名，也不需要配置 Webhook 回调地址。
3. MacBook Air 不参与生产运行。电脑休眠后，云端日报和飞书问答仍应独立运行。

## 今晚先拆成两条线

### 每日股票分析

如果要求准时，不再把 GitHub Actions 当主链路。

- GitHub Actions 适合备用、手动触发和临时补跑。
- 准点日报应放到云端常驻容器的 `cloud-worker` 服务里跑。
- 云端容器不依赖 GitHub runner 排队，也不依赖 MacBook Air 是否休眠。

### 飞书问答

优先用同一个云端常驻容器 `cloud-worker` 运行飞书问答；如果只想单独跑问答，也可以运行 `feishu-bot` 服务。

这个服务会启动项目后端，并自动打开：

```env
BOT_ENABLED=true
FEISHU_STREAM_ENABLED=true
AGENT_MODE=true
AGENT_NL_ROUTING=true
SCHEDULE_ENABLED=true
SCHEDULE_TIMES=12:35,20:00
```

支持的使用方式：

```text
/help
/ask 600519
/chat 今天大盘怎么看？
@机器人 帮我看看茅台
@机器人 用缠论分析 AAPL
```

## 需要准备的账号

### 必须

- GitHub：已经有。
- 飞书开放平台：创建一个企业自建应用，用于 Stream Bot。
- 一个 LLM API：例如 Anspire、AIHubMix、OpenAI 兼容服务、DeepSeek 等，至少一个。
- 一个云端常驻运行环境。

### 不必须

- Docker Hub：不需要。
- 域名：不需要。
- 备案：不需要，除非未来要公开访问 WebUI。

## 云端选择建议

优先选择能从 GitHub 直接部署 Dockerfile 的平台，少碰服务器命令。

推荐顺序：

1. Railway / Render / Fly.io 这类容器平台：适合先跑通，少运维。
2. 阿里云 / 腾讯云轻量服务器：更传统、更稳定，但需要 SSH、Docker、日志、重启等服务器操作。

如果今晚目标是“先跑通飞书问答”，优先选第 1 类。

## GitHub 需要配置的变量

Actions 每日分析至少需要：

```env
STOCK_LIST=600519,hk00700,AAPL
ANSPIRE_API_KEYS=你的 key
# 或者 AIHUBMIX_KEY / OPENAI_API_KEY / DEEPSEEK_API_KEY 等至少一个
FEISHU_WEBHOOK_URL=你的飞书群机器人 webhook
```

云端主服务至少需要：

```env
FEISHU_APP_ID=cli_xxx
FEISHU_APP_SECRET=xxx
BOT_ENABLED=true
FEISHU_STREAM_ENABLED=true
AGENT_MODE=true
AGENT_NL_ROUTING=true
SCHEDULE_ENABLED=true
SCHEDULE_TIMES=12:35,20:00
ANSPIRE_API_KEYS=你的 key
# 或者 AIHUBMIX_KEY / OPENAI_API_KEY / DEEPSEEK_API_KEY 等至少一个
```

## 飞书开放平台要做的事

1. 创建企业自建应用。
2. 打开机器人能力。
3. 给应用开通消息读取和发送相关权限。
4. 启用事件订阅的长连接 / Stream 模式。
5. 发布或安装应用到你的飞书组织。
6. 把应用加到目标群，或先用私聊测试。

## Docker Compose 本地/云端启动入口

云端主服务（准点日报 + 飞书问答）：

```bash
docker-compose -f ./docker/docker-compose.yml up -d cloud-worker
```

定时报表：

```bash
docker-compose -f ./docker/docker-compose.yml up -d analyzer
```

Web/API：

```bash
docker-compose -f ./docker/docker-compose.yml up -d server
```

飞书问答：

```bash
docker-compose -f ./docker/docker-compose.yml up -d feishu-bot
```

查看飞书问答日志：

```bash
docker-compose -f ./docker/docker-compose.yml logs -f cloud-worker
```

## 验收标准

1. 云端 `cloud-worker` 能按预期时间运行，并收到日报。
2. GitHub Actions 能手动 Run workflow，作为备用补跑入口。
3. 飞书私聊机器人发送 `/help`，机器人能回复。
4. 飞书私聊机器人发送 `/chat 今天大盘怎么看？`，机器人能回复。
5. 飞书群里 @机器人并问股票，机器人只在被 @ 时回复。
6. MacBook Air 合上盖子后，日报和飞书问答仍能运行。

## 不建议今晚做的事

- 暴露 WebUI 到公网。
- 上来就买复杂云服务器套餐。
- 把 API Key 写进仓库文件。
- 同时改太多通知渠道。
