# 🦞 OpenClaw 完整部署与进阶配置指南

[![Version](https://img.shields.io/badge/Version-2026.3.12-blue.svg)]() [![OS](https://img.shields.io/badge/OS-Windows_10%2F11_%7C_WSL2-lightgrey.svg)]() [![Node](https://img.shields.io/badge/Node.js-%3E%3D_22.0-green.svg)]()

> 欢迎来到 OpenClaw 的世界！这是一份专为 Windows 用户打造的保姆级部署指南。无论你是追求极致性能的极客，还是刚接触 AI 代理的新手，都能在这里找到适合你的安装路线。

⏱️ **预计耗时**：30-60 分钟

---

## 🔀 部署路线选择

为了获得最佳体验，我们提供了两种安装路径。请根据你的技术背景选择：

* **🐧 路线 A（推荐）：WSL2 + Ubuntu 部署。** 性能最佳，沙盒环境更安全，完美兼容各类 Linux 独占的 AI 插件。
* **🪟 路线 B：Windows 原生部署。** 适合不熟悉 Linux 命令行的新手，直接在 PowerShell 中完成所有操作。（出现很多其奇奇怪怪的问题😢）

---

## 🐧 路线 A：WSL2 极客部署（强烈推荐）

### Step 1. WSL2 基础安装与迁移
1. **一键安装 WSL2**：在 Windows PowerShell（管理员）中执行 `wsl --install`。
2. **优雅迁移至 D 盘**（强烈建议，拯救 C 盘空间）：
   ```powershell
   wsl --shutdown
   wsl --export Ubuntu D:\ubuntu-backup.tar
   wsl --unregister Ubuntu
   mkdir D:\WSL
   wsl --import Ubuntu D:\WSL D:\ubuntu-backup.tar
   wsl -d Ubuntu -u root -e bash -c "echo -e '[user]\ndefault=你的用户名' > /etc/wsl.conf"
   wsl --shutdown
   ```
3. **网络镜像服务**：在 Windows 用户目录下（`C:\Users\你的用户名`）创建或编辑 `.wslconfig` 文件：
   ```ini
   [wsl2]
   networkingMode=mirrored
   dnsTunneling=true
   autoProxy=true
   firewall=true

   [experimental]
   autoMemoryReclaim=gradual
   hostAddressLoopback=true
   ```
4. **初始化系统**：进入 Ubuntu 终端，执行 `sudo apt-get update -y && sudo apt-get upgrade -y`。

### Step 2. 安装 Node.js (v22+)
在 Ubuntu 终端中使用 `fnm` 进行版本管理：
```bash
curl -fsSL [https://fnm.vercel.app/install](https://fnm.vercel.app/install) | bash
source ~/.bashrc
fnm install 22
node --version # 确认输出 v22.x.x
```

### Step 3. 一键安装 OpenClaw 核心
```bash
curl -fsSL [https://molt.bot/install.sh](https://molt.bot/install.sh) | bash
openclaw --version # 验证安装，应显示 2026.3.12 
```
👉 *完成此步后，请直接跳至 **通用配置阶段**。*

---

## 🪟 路线 B：Windows 原生部署

### Step 1. 安装 Node.js
1. 前往 [Node.js 官网](https://nodejs.org/) 下载 **LTS 版本**。
2. ⚠️ **关键步骤**：在安装过程中，务必勾选 **"Automatically install the necessary tools"**。
3. 打开 PowerShell 验证：`node --version`。

### Step 2. 安装 OpenClaw 核心
在 PowerShell 中执行以下命令：
```powershell
iwr -useb [https://openclaw.ai/install.ps1](https://openclaw.ai/install.ps1) | iex
```
*💡 常见排错：如果提示执行策略报错，请先运行 `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`。*

---

## ⚙️ 通用配置阶段（双端适用）

### Step 4. 初始化向导与 AI 模型接入
在终端（或 PowerShell）运行配置向导：
```bash
openclaw onboard
```
* **交互建议**：选择 `Local (this machine)`，并在 Auth provider 中根据需要配置。

**🔥 热门模型配置指南：**
* **DeepSeek（国内高性价比首选👌）**：
  * Auth Provider: `Custom Provider`
  * API Base URL: `https://api.deepseek.com`
  * Endpoint compatibility: `OpenAI-compatible`
  * Model ID: `deepseek-chat` 或 `deepseek-reasoner`
* *(备选)* **MiniMax（免费15￥额度💴）**：选择 `minimax-api` 并填入 Key。
* *(备选)* **OpenRouter（完全免费，但是慢）**：选择 `OpenRouter` 并填入 Key。

### Step 5. 接入社交通道矩阵

#### 🤖 选项 A：绑定 QQ 机器人（请参考官方说明文件）
1. 在 [QQ 开放平台](https://q.qq.com/) 获取应用的 `AppID` 和 `AppSecret`。
2. 安装官方插件并配置通道：
   ```bash
   openclaw plugins install @tencent-connect/openclaw-qqbot@latest
   openclaw channels add --channel qqbot
   ```

#### 🏢 选项 B：绑定飞书企业机器人（请参考官方说明文件）
1. 在 [飞书开放平台](https://open.feishu.cn/app) 创建应用并获取密钥。
2. 开启机器人能力，配置事件订阅为**长连接**并添加 `im.message.receive_v1` 权限。
3. 安装插件并配置通道：
   ```bash
   openclaw plugins install @larksuite/openclaw-lark
   openclaw channels add --channel feishu
   ```

---

## 🚀 启动与进阶操作

### 守护进程管理
| 操作 | 命令 | 说明 |
| :--- | :--- | :--- |
| **启动测试** | `openclaw gateway start` | 前台运行，方便观察连接日志 |
| **后台安装** | `openclaw gateway install` | 注册为系统服务，开机自启 |
| **重启服务** | `openclaw gateway restart` | 修改配置文件后需执行 |
| **全景监控** | `openclaw status` | 查看核心、网络、会话等完整健康报告 |

### 🧠 动态切换模型
你可以通过多种方式无缝切换背后的 AI 大脑：
1. **聊天内直切**：对机器人发送 `/model <模型名>`。
2. **修改主配置**：编辑 `~/.openclaw/openclaw.json`，在 `agents.defaults.model` 中修改 `primary` 和 `fallbacks`（备用模型）。

---

## 🛠️ FAQ 与常见排错

* **Q: 安装 npm 插件时疯狂报错？**
  * A: 检查 Git 是否被墙，可尝试将 SSH 强制转为 HTTPS：`git config --global url."https://github.com/".insteadOf ssh://git@github.com/`
* **Q: QQ/飞书 机器人处于“已读不回”状态？**
  * A: 
    1. 运行 `openclaw logs --follow` 观察是否有网络超时报错。
    2. 确认飞书群聊中是否 **@了机器人**。
    3. 检查 JSON 配置文件中的 `allowFrom` 是否包含了你的账号或设置为 `["*"]`。
* **Q: Gateway 启动失败？**
  * A: 检查 `openclaw.json` 中的 `gateway` 节点，确保 `bind` 设置为 `loopback` 且端口 `18789` 未被占用。

> 🎉 **恭喜！你已成功部署 OpenClaw！** > 更多极客玩法与生态插件，请访问 [ClawHub 技能市场](https://clawhub.com) 或 [官方文档](https://docs.openclaw.ai/)。
