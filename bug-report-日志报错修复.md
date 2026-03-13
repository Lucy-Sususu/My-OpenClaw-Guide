# Bug Report: OpenClaw 部署常见日志报错及修复方案

**日期**：2026-03-14  
**来源**：实际部署过程中遇到的日志报错整理  
**环境**：WSL2 (Ubuntu) + Windows 10/11  
**影响**：网关启动异常、插件冲突、网络绑定失败

---

## 问题 1：apply_patch / image 工具警告

### 症状

网关启动或运行时日志中出现 `apply_patch` 或 `image` 相关的警告信息。

### 原因

`openclaw.json` 配置文件中的 `tools.profile` 设置为 `"coding"`，但该 profile 的 allowlist 中包含了 `apply_patch` 和 `image` 工具。某些环境下这两个工具可能未正确注册或不兼容，导致警告。

### 修复方法

编辑 `~/.openclaw/openclaw.json`，找到 `tools` 配置段：

```json
{
  "tools": {
    "profile": "coding"
  }
}
```

如果存在 `allow` 数组且包含 `apply_patch` 或 `image`，将其移除：

```json
{
  "tools": {
    "profile": "coding",
    "allow": [
      "read",
      "write",
      "edit",
      "exec",
      "web_search",
      "web_fetch"     ← 删除 "apply_patch" 和 "image"
    ]
  }
}
```

### 验证

```bash
openclaw gateway restart
openclaw status  # 确认无相关警告
```

---

## 问题 2：feishu_chat 插件冲突

### 症状

日志中出现 `feishu_chat` 或 `feishu` 相关的冲突/重复注册错误。

### 原因

配置文件的 `plugins.allow` 列表中同时存在 `openclaw-lark` 和 `feishu` 两个插件。`openclaw-lark` 已包含完整的飞书（Lark/Feishu）功能，与旧的 `feishu` 插件产生冲突。

### 修复方法

编辑 `~/.openclaw/openclaw.json`，找到 `plugins` 配置段：

```json
{
  "plugins": {
    "allow": [
      "openclaw-qqbot",
      "openclaw-lark",
      "feishu"         ← 删除这一行，openclaw-lark 已包含飞书功能
    ]
  }
}
```

修改后：

```json
{
  "plugins": {
    "allow": [
      "openclaw-qqbot",
      "openclaw-lark"
    ]
  }
}
```

### 验证

```bash
openclaw gateway restart
openclaw status  # 确认插件正常加载，无冲突
```

---

## 问题 3：no IPv4 address on loopback0

### 症状

网关启动失败或日志报错：

```
Error: no IPv4 address on loopback0
```

或类似网络绑定失败的错误。

### 原因

WSL2 环境下，loopback 接口（lo）可能未配置 `127.0.0.1` 回环地址。OpenClaw 网关需要绑定到 `127.0.0.1`，缺少该地址会导致启动失败。

### 修复方法

在 WSL2 终端中执行：

```bash
# 为 loopback 接口添加 127.0.0.1 地址
sudo ip addr add 127.0.0.1/8 dev lo
```

然后重启网关：

```bash
openclaw gateway restart
```

### 永久修复（推荐）

WSL2 重启后上述配置会丢失。将其添加到启动脚本中：

```bash
# 方式 1：添加到 ~/.bashrc 或 ~/.profile
echo 'sudo ip addr add 127.0.0.1/8 dev lo 2>/dev/null' >> ~/.bashrc

# 方式 2：创建 systemd 服务（如果 WSL 支持 systemd）
# 创建 /etc/systemd/system/loopback-fix.service
```

### 验证

```bash
ip addr show lo | grep 127.0.0.1
# 应输出：inet 127.0.0.1/8 scope host lo
```

---

## ⚠️ 重要提醒：修改配置前备份

任何配置修改前，请先备份：

```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak
```

如需恢复：

```bash
cp ~/.openclaw/openclaw.json.bak ~/.openclaw/openclaw.json
openclaw gateway restart
```

---

## 快速修复清单

| 问题 | 关键命令/操作 |
|------|-------------|
| apply_patch/image 警告 | 从 `tools.allow` 中移除对应工具 |
| feishu_chat 冲突 | 从 `plugins.allow` 中移除 `"feishu"` |
| 无 loopback IPv4 | `sudo ip addr add 127.0.0.1/8 dev lo` |
| 通用 | 改配置前 `cp openclaw.json openclaw.json.bak` |
