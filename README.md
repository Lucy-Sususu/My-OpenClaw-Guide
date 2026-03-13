# 🦞 我的 OpenClaw 完整部署与进阶配置踩坑指南

[![Version](https://img.shields.io/badge/Version-2026.3.12-blue.svg)]() [![OS](https://img.shields.io/badge/OS-Windows_10%2F11_%7C_WSL2-lightgrey.svg)]() [![Node](https://img.shields.io/badge/Node.js-%3E%3D_22.0-green.svg)]()

> 欢迎来到我的 OpenClaw 折腾记录！作为一名在 Windows 环境下折腾 AI 的玩家，我踩过不少坑，也总结出了这份保姆级的部署指南。无论你是追求极致性能的极客，还是刚接触 AI Agent 的新手，希望能少走弯路，一次点亮！

⏱️ **我的实测耗时**：30-60 分钟

---

## 🔀 部署路线选择

在实际折腾中，我总结了两种安装路径。大家可以根据自己的技术背景来选择：

* **🐧 路线 A（我强烈推荐）：WSL2 + Ubuntu 部署。** 性能最佳，沙盒环境不伤原系统，最重要的是能完美兼容各类 Linux 独占的 AI 插件。
* **🪟 路线 B：Windows 原生部署。** 适合完全不想碰 Linux 命令行的新手朋友，直接在 PowerShell 搞定。

---

## 🐧 路线 A：WSL2 极客部署（我目前在用的方案）

### Step 1. WSL2 基础安装与迁移
1. **一键安装 WSL2**：打开我的 Windows PowerShell（管理员模式），直接执行 `wsl --install`。
2. **优雅迁移至 D 盘**（我强烈建议这一步，拯救 C 盘空间）：
   ```powershell
   wsl --shutdown
   wsl --export Ubuntu D:\ubuntu-backup.tar
   wsl --unregister Ubuntu
   mkdir D:\WSL
   wsl --import Ubuntu D:\WSL D:\ubuntu-backup.tar
   # 这一步是设置默认登录用户，别忘了把“你的用户名”换掉
   wsl -d Ubuntu -u root -e bash -c "echo -e '[user]\ndefault=你的用户名' > /etc/wsl.conf"
   wsl --shutdown
   ```
3. **网络满血优化**：为了防止在 WSL 里拉取模型和插件时网络超时，我在 Windows 用户目录下（`C:\Users\你的用户名`）创建了 `.wslconfig` 文件：
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
4. **初始化系统**：进入 Ubuntu 终端，先给系统洗个澡 `sudo apt-get update -y && sudo apt-get upgrade -y`。

### Step 2. 安装 Node.js (v22+)
我不喜欢用系统自带的包管理器装 Node，我个人更习惯用 `fnm` 进行版本管理：
```bash
curl -fsSL [https://fnm.vercel.app/install](https://fnm.vercel.app/install) | bash
source ~/.bashrc
fnm install 22
node --version # 确认输出是不是 v22.x.x
```

### Step 3. 一键安装 OpenClaw 核心
一行代码搞定核心引擎：
```bash
curl -fsSL [https://molt.bot/install.sh](https://molt.bot/install.sh) | bash
openclaw --version # 验证安装，我目前用的是 2026.3.12 
```
👉 *搞定这步后，咱们直接跳到下面的 **通用配置阶段**。*

---

## 🪟 路线 B：Windows 原生部署（不折腾党的福音）

### Step 1. 安装 Node.js
1. 去 [Node.js 官网](https://nodejs.org/) 下载 **LTS 版本**。
2. ⚠️ **我踩过的坑**：安装时，千万记得勾选 **"Automatically install the necessary tools"**，不然编译依赖会报错！
3. 打开 PowerShell 验证一下：`node --version`。

### Step 2. 安装 OpenClaw 核心
在 PowerShell 中执行：
```powershell
iwr -useb [https://openclaw.ai/install.ps1](https://openclaw.ai/install.ps1) | iex
```
*💡 小贴士：如果系统提示“执行策略报错”，先敲一行这个：`Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` 解除封印。*

---

## ⚙️ 通用配置阶段（核心大脑注入）

### Step 4. 初始化向导与 AI 模型接入
在终端运行配置向导：
```bash
openclaw onboard
```
* **我的交互选择**：运行位置选 `Local (this machine)`，然后重点是选模型。

**🔥 我个人的常用模型配置：**
* **DeepSeek（便宜但一般聪明）**：
  * Auth Provider: 选 `Custom Provider`
  * API Base URL: 填 `https://api.deepseek.com`
  * Endpoint compatibility: 选 `OpenAI-compatible`
  * Model ID: 填 `deepseek-chat` 或 `deepseek-reasoner`
* *(备选)* **MiniMax（不便宜但聪明）**：国内网络极其稳定，选 `minimax` ，填 Key 就行。
* *(备选)* **OpenRouter（白嫖各路大厂模型，直接用这个聚合渠道）**：选 `OpenRouter`，填 Key 就行。

### Step 5. 把 AI 接到我的社交软件上

#### 🤖 玩法 A：变身我的专属 QQ 机器人
1. 先去 [QQ 开放平台](https://q.qq.com/) 申请个应用，拿到 `AppID` 和 `AppSecret`。
2. 在终端安装官方插件并配置通道：
   ```bash
   openclaw plugins install @tencent-connect/openclaw-qqbot@latest
   openclaw channels add --channel qqbot
   ```

#### 🏢 玩法 B：接入飞书企业机器人（超强生产力）
1. 在 [飞书开放平台](https://open.feishu.cn/app) 创个应用拿密钥。
2. 开启机器人能力，把事件订阅改成**长连接**，顺手加上 `im.message.receive_v1` 权限。
3. 一键装插件配通道：
   ```bash
   openclaw plugins install @larksuite/openclaw-lark
   openclaw channels add --channel feishu
   ```

---

## 🚀 启动与进阶操作

### 我的常用管理命令
平时折腾的时候，这几个命令我敲得最多：

| 操作 | 命令 | 我的使用场景 |
| :--- | :--- | :--- |
| **测试启动** | `openclaw gateway start` | 刚改完配置，前台盯着日志看有没有报错 |
| **后台驻留** | `openclaw gateway install` | 调教完美后，直接写进系统服务开机自启 |
| **重启网关** | `openclaw gateway restart` | 每次改完 `openclaw.json` 必备操作 |
| **全景体检** | `openclaw status` | 检查缓存命中率和网络延迟 |

### 🧠 动态切模型的小技巧
我不喜欢每次都去改代码换模型：
1. **聊天界面直接切**：在 QQ 或飞书里，我直接给它发 `/model <模型名>`，瞬间换脑。
2. **改底座文件**：如果想彻底改默认模型，就去编辑 `~/.openclaw/openclaw.json`，在 `agents.defaults.model` 里把 `primary` 换掉。

---

## 🛠️ 我的填坑记录 (FAQ)

* **Q: npm 装插件时卡死或报网络错误？**
  * A: 这是国内网络通病。我一般强制把 Git 的 SSH 换成 HTTPS：`git config --global url."https://github.com/".insteadOf ssh://git@github.com/`。
* **Q: 飞书机器人“已读不回”，日志里一直显示 queued？**
  * A: 我在这个坑里躺了很久！后来发现：
    1. 大概率是大模型 API 超时了（尤其是用国外源的时候）。
    2. 如果秒回 Timeout，去查查 `openclaw.json` 里的 `allowFrom`，是不是把自己防蹭网拦截了。如果是单人使用，记得设为 `["*"]`（如果是飞书，记得配好群聊 @）。
* **Q: 日志里老是跳出什么 `tools.profile (coding) ... unknown entries`？**
  * A: 这是系统的强迫症报警。我嫌烦，直接在 JSON 里把 `"profile": "coding"` 改成了 `"profile": "default"`，世界清静了。

> 🎉 **写在最后** > 这就是我把本地电脑变成超级 AI 助理的全过程。如果你觉得有帮助，欢迎给我点个 Star！更多好玩的极客生态，大家可以去 [ClawHub 技能市场](https://clawhub.com) 淘一淘。