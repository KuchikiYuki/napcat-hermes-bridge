# WSL Messaging Bridge — NapCat ↔ Hermes Agent

> 在 WSL 中通过 QQ 远程操控 Hermes Agent 的完整方案。

## Quickstart

直接将本项目的git链接发给agent，让它自己根据经验总结完成部署(ps：其实这个项目全程也是靠agent自己写好的)。其他环境或者不同的agent，本项目或许也能有一定参考作用，但接入agnet的对话部分会有所区别。

## 这是什么

这是一个将 **QQ**（通过 NapCat / OneBot v11 协议）接入 **Hermes Agent** 的桥接方案。你可以在手机 QQ 上直接与 Hermes Agent 对话，Agent 保持完整上下文，支持工具调用、代码执行等全部能力——就像在终端里用 Hermes 一样。

### 核心特性

- 📱 **手机 QQ 远程操控** — 出门在外也能给 Hermes 发指令
- 🧠 **会话持久化** — 所有消息走同一个 session，上下文不丢失
- 🔄 **每次链路启动自动开新会话** — 避免和其他 Agent 进程的 session 冲突
- 🎛️ **QQ 指令管理会话** — `/new` `/switch` `/session` `/list` `/help`
- ⚡ **一键启停** — `qq-start.sh` / `qq-stop.sh` 连锁启动/停止全部服务
- 🔒 **Owner 白名单** — 只有指定 QQ 号能操控 Agent
- 🩹 **404 自动恢复** — session 丢失时自动重建并重试

## 架构

```
手机 QQ
  │
  ▼
NTQQ (headless, xvfb)       ← Linux QQ 客户端，无头模式运行
  │
  ▼
NapCat (OneBot v11)         ← 注入 NTQQ，暴露 HTTP API + 事件上报
  │ :3000 (HTTP API)          ← 发送消息
  │ :6099 (WebUI)             ← 登录管理面板
  ▼
Bridge (aiohttp)             ← 轻量 HTTP 桥接脚本
  │ :8080                      ← 接收 OneBot 事件
  ▼
Hermes API Server            ← Hermes Agent 的 Gateway 适配器
  │ :8642                      ← OpenAI 兼容端点
  ▼
Hermes Agent                 ← 实际执行任务的 AI Agent
```

**数据流：**
1. 手机 QQ 发消息 → NTQQ 收到 → NapCat 解析为 OneBot 事件
2. NapCat POST 事件到 Bridge (:8080)
3. Bridge 调用 Hermes API `POST /api/sessions/{SID}/chat` (:8642)
4. Hermes Agent 处理并返回回复
5. Bridge 通过 NapCat HTTP API (:3000) 发回 QQ

## 系统环境要求

| 组件 | 要求 |
|------|------|
| 操作系统 | WSL2 (Windows Subsystem for Linux)，推荐 Ubuntu 24.04+ |
| systemd | 需要启用（`/etc/wsl.conf` → `[boot] systemd=true`）|
| Python | 3.10+（Bridge 脚本需要） |
| Hermes Agent | 已安装并配置好（`hermes config set` 可用） |
| 网络 | Windows 侧有 HTTP 代理可从 WSL 访问（用于下载 NTQQ/NapCat） |
| sudo | 可用（安装系统依赖需要） |

### 测试环境

本方案在以下环境中开发并验证：

- **OS**: WSL2 Ubuntu 26.04 (resolute)
- **Hermes Agent**: Hermes Agent by Nous Research
- **NTQQ**: Linux QQ (NTQQ) headless + xvfb
- **NapCat**: NapCat v4.18.x (OneBot v11)
- **Python**: 3.12+ (aiohttp)
- **GPU**: 无需 GPU（`--disable-zygote --in-process-gpu` 绕过 Electron GPU 检测）

## 文件说明

| 文件 | 说明 |
|------|------|
| `wsl-messaging-bridge.md` | 完整的 Skill 文档（含部署流程、13 个踩坑记录、验证步骤） |
| `wsl-messaging-bridge-skill-backup.tar.gz` | Hermes Skill 格式的完整备份（含 templates 和 references） |

## 如何使用

### 快速开始（已有 Hermes Agent 环境）

如果你已经有一个跑着 Hermes Agent 的 WSL 环境，按以下步骤操作：

#### 1. 安装系统依赖

```bash
sudo apt-get install -y \
    xvfb unzip libnss3 libnotify4 libsecret-1-0 libgbm1 \
    libasound2t64 fonts-wqy-zenhei gnutls-bin libglib2.0-dev \
    libdbus-1-3 libgtk-3-0 libxss1 libxtst6 libatspi2.0-0 \
    libx11-xcb1 ffmpeg curl jq
```

> Ubuntu 24.04 及以下用 `libasound2` 代替 `libasound2t64`。

#### 2. 下载 NTQQ + NapCat

```bash
mkdir -p ~/napcat-astrbot && cd ~/napcat-astrbot

# 下载 Linux QQ (.deb)
# 去 https://github.com/NapNeko/NapCatQQ/releases 查看推荐的 QQ 版本
curl -L -o linuxqq.deb "https://dldir1v6.qq.com/qqfile/qq/QQNT/<hash>/linuxqq_<ver>_amd64.deb"
dpkg-deb -x linuxqq.deb ntqq/

# 下载 NapCat
curl -L -o NapCat.Shell.zip "https://github.com/NapNeko/NapCatQQ/releases/download/v<ver>/NapCat.Shell.zip"
unzip -q NapCat.Shell.zip -d napcat/
```

#### 3. 注入 NapCat 到 NTQQ

```bash
QQ_APP="ntqq/opt/QQ/resources/app"
cat > "$QQ_APP/loadNapCat.js" << EOF
(async () => {await import('file://$HOME/napcat-astrbot/napcat/napcat.mjs');})();
EOF

python3 -c "
import json; p='$QQ_APP/package.json'; f=open(p); d=json.load(f); f.close()
d['main']='./loadNapCat.js'; open(p,'w').write(json.dumps(d,indent=2))
"
```

> ⚠️ `loadNapCat.js` 中必须使用**绝对路径**，相对路径会报 `ERR_MODULE_NOT_FOUND`。

#### 4. 配置 NapCat OneBot v11

创建 `napcat/config/onebot11.json`：

```json
{
  "network": {
    "httpServers": [{
      "enable": true, "name": "api",
      "host": "127.0.0.1", "port": 3000,
      "messagePostFormat": "array", "token": "",
      "debug": false, "enableForcePushEvent": true
    }],
    "httpClients": [{
      "enable": true, "name": "bridge",
      "url": "http://127.0.0.1:8080/onebot",
      "messagePostFormat": "array", "reportSelfMessage": false, "token": ""
    }],
    "websocketServers": [], "websocketClients": [], "plugins": []
  },
  "musicSignUrl": "", "enableLocalFile2Url": false, "parseMultMsg": false
}
```

#### 5. 启用 Hermes API Server

```bash
hermes config set platforms.api_server.enabled true
hermes config set platforms.api_server.extra.key "your-api-key"
hermes config set platforms.api_server.extra.host "127.0.0.1"
hermes config set platforms.api_server.extra.port 8642
```

#### 6. 部署 Bridge 脚本

从 `wsl-messaging-bridge.md` 的 "Template: bridge.py" 部分提取 `bridge.py`，或解压 `wsl-messaging-bridge-skill-backup.tar.gz` 获取 `templates/bridge.py`。

```bash
# 解压获取模板文件
tar xzf wsl-messaging-bridge-skill-backup.tar.gz
cp wsl-messaging-bridge/templates/bridge.py ~/napcat-astrbot/bridge.py
cp wsl-messaging-bridge/templates/qq-start.sh ~/napcat-astrbot/qq-start.sh
cp wsl-messaging-bridge/templates/qq-stop.sh ~/napcat-astrbot/qq-stop.sh
chmod +x ~/napcat-astrbot/qq-start.sh ~/napcat-astrbot/qq-stop.sh
```

#### 7. 配置并启动

编辑 `qq-start.sh`，修改以下变量为你的环境：

```bash
QQ_OWNER=你的大号QQ     # 接收确认消息的 QQ
QQ_BOT=你的小号QQ       # NapCat 登录用的 QQ
PROXY_HOST="172.x.x.1"  # 你的 WSL 网关 IP
PROXY_PORT="<PROXY_PORT>"  # 代理端口
WIN_USER="你的Windows用户名"
```

启动：

```bash
bash ~/napcat-astrbot/qq-start.sh
```

首次启动需要用手机 QQ 扫码登录。之后会自动快速登录（约 15 秒）。启动成功后你会收到 QQ 确认消息，包含当前 Session ID。

#### 8. 开始使用

在 QQ 中给 bot 发消息即可与 Hermes Agent 对话。Bridge 指令：

| 指令 | 功能 |
|------|------|
| `/new` | 开启新会话（清除上下文） |
| `/switch <id>` | 切换到指定会话 |
| `/session` | 显示当前会话 ID |
| `/list` | 列出所有会话 |
| `/help` | 显示帮助 |

### 作为 Hermes Skill 使用

如果你是 Hermes Agent 用户，可以直接安装为 Skill：

```bash
mkdir -p ~/.hermes/skills/software-development/
tar xzf wsl-messaging-bridge-skill-backup.tar.gz -C ~/.hermes/skills/software-development/
```

安装后 Hermes Agent 在处理相关任务时会自动加载这个 Skill。

## 踩过的坑（全部记录在 md 文档中）

这个方案在生产环境中踩过 13 个坑，全部记录在 `wsl-messaging-bridge.md` 的 Pitfalls 章节：

1. NapCat 账号级配置文件覆盖通用配置
2. localhost 请求被代理拦截（NapCat 的 Node.js HTTP Client 问题）
3. NapCat 文件路径必须用绝对路径
4. 国内 PyPI 镜像 wheel 下载 403
5. apt 残留代理配置
6. xvfb 授权问题
7. nohup 被 Hermes terminal 工具拦截
8. NapCat 配置目录位置
9. **Session ID 硬编码导致上下文丢失**（v2 → v3 重构的根因）
10. **NapCat GPU 进程 crash**（`--disable-zygote --in-process-gpu` 修复）
11. **Hermes API 响应格式不匹配**（`/list` 返回空）
12. 确认消息应包含 Session ID


## 技术栈

- **Hermes Agent** (by Nous Research) — AI Agent 框架
- **NapCat** (by NapNeko) — OneBot v11 协议实现
- **NTQQ** — Linux QQ 客户端 (Electron)
- **aiohttp** — Bridge 的异步 HTTP 服务器/客户端
- **xvfb** — 无头 X 服务器（运行 Electron 应用）

## License

MIT
