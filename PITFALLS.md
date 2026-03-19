# OpenClaw 避坑指南

> **OpenClaw 使用误区、大坑与解决方案完整指南（42 条实战经验）**

本指南基于真实生产环境经验整理，涵盖从致命错误到轻微误区的完整知识体系。建议收藏并定期查阅。

---

## 📋 快速导航

- [🔴 致命错误（6 条）](#-致命错误)
- [🟠 常见坑（18 条）](#-常见坑)
- [🟡 轻微误区（18 条）](#-轻微误区)
- [✅ 快速检查清单](#-快速检查清单)
- [📚 推荐实践](#-推荐实践)

---

## 🔴 致命错误

> ⚠️ **警告**：以下错误可能导致系统崩溃、数据丢失或安全漏洞，请务必优先规避。

### 1. 恶意 Skills 窃取凭证

| 项目 | 描述 |
|------|------|
| **问题** | ClawHub 社区 Skill 无沙箱隔离，恶意代码可读取主机文件 |
| **症状** | 凭证泄露、异常网络请求、未知外部连接 |
| **风险等级** | 🔴 致命 |

**解决方案：**
```bash
# 1. 仅安装官方或可信来源的 Skill
# 2. 审查 Skill 代码后再安装
# 3. 定期审计已安装 Skill 的权限

# 检查 Skill 目录
ls -la ~/.openclaw/extensions/

# 审查 Skill 文件
cat ~/.openclaw/extensions/<skill-name>/SKILL.md
```

**规避方法：**
- ✅ 优先使用官方扩展市场
- ✅ 安装前审查代码（特别是网络请求和文件读写部分）
- ✅ 定期更新并审计已安装 Skill
- ❌ 不要随意安装来源不明的 Skill

---

### 2. 配置验证失败静默重置

| 项目 | 描述 |
|------|------|
| **问题** | 配置语法错误导致系统静默重置为默认值 |
| **症状** | 配置更改不生效、系统行为异常、无错误提示 |
| **风险等级** | 🔴 致命 |

**解决方案：**
```bash
# 1. 修改配置前先备份
cp ~/.openclaw/config.json ~/.openclaw/config.json.bak

# 2. 使用 JSON 验证工具检查语法
jq '.' ~/.openclaw/config.json

# 3. 重启后验证配置是否生效
openclaw gateway status
```

**规避方法：**
- ✅ 任何配置修改前先备份
- ✅ 使用 `jq` 或在线 JSON 验证器检查语法
- ✅ 小步修改，逐步验证
- ❌ 不要一次性修改多个配置项

---

### 3. 更新后权限全失

| 项目 | 描述 |
|------|------|
| **问题** | 系统更新后文件权限被重置，导致服务无法启动 |
| **症状** | `EACCES` 错误、服务启动失败、文件无法读写 |
| **风险等级** | 🔴 致命 |

**解决方案：**
```bash
# 1. 修复权限
sudo chown -R $USER:$USER ~/.openclaw/
sudo chmod -R 755 ~/.openclaw/

# 2. 检查关键目录权限
ls -la ~/.openclaw/

# 3. 重启服务
openclaw gateway restart
```

**规避方法：**
- ✅ 更新前记录当前权限设置
- ✅ 更新后检查关键文件权限
- ✅ 使用脚本自动化权限修复
- ❌ 不要用 root 运行 OpenClaw

---

### 4. npm 安装毁文件系统

| 项目 | 描述 |
|------|------|
| **问题** | 全局 npm 安装可能覆盖系统文件或安装恶意包 |
| **症状** | 系统命令异常、未知进程、磁盘空间骤减 |
| **风险等级** | 🔴 致命 |

**解决方案：**
```bash
# 1. 使用官方推荐安装方式（非 npm）
curl -fsSL https://openclaw.dev/install.sh | bash

# 2. 如已用 npm 安装，彻底清理
npm uninstall -g openclaw
rm -rf ~/.openclaw/
# 重新用官方脚本安装

# 3. 检查系统完整性
which openclaw
openclaw --version
```

**规避方法：**
- ✅ 始终使用官方安装脚本
- ✅ 不要使用 `npm install -g openclaw`
- ✅ 从官方文档获取安装指南
- ❌ 不要信任第三方安装教程

---

### 5. Podman/EACCES 权限锁死

| 项目 | 描述 |
|------|------|
| **问题** | Podman 容器权限配置错误导致文件访问被拒绝 |
| **症状** | `EACCES: permission denied`、容器无法读写卷、数据丢失 |
| **风险等级** | 🔴 致命 |

**解决方案：**
```bash
# 1. 检查容器用户映射
podman inspect <container-name> | grep -A 10 User

# 2. 修复卷权限
sudo chown -R 1000:1000 /path/to/volume

# 3. 使用正确的用户运行容器
podman run -u $(id -u):$(id -g) ...

# 4. 或改用 Docker（更成熟的用户映射）
docker run ...
```

**规避方法：**
- ✅ 明确容器内用户 ID 与主机映射关系
- ✅ 卷挂载前设置正确权限
- ✅ 考虑使用 Docker 替代 Podman
- ❌ 不要用 root 运行容器

---

### 6. Discord/飞书登录永久失败

| 项目 | 描述 |
|------|------|
| **问题** | OAuth 配置错误或 Token 过期导致永久无法登录 |
| **症状** | 登录页面循环、认证失败、无法连接频道 |
| **风险等级** | 🔴 致命 |

**解决方案：**
```bash
# 1. 检查 OAuth 配置
cat ~/.openclaw/config.json | jq '.channels'

# 2. 重新授权
openclaw auth discord  # 或 feishu
# 按提示完成浏览器授权

# 3. 如仍失败，清除缓存重新授权
rm -rf ~/.openclaw/oauth/
openclaw auth discord

# 4. 验证连接
openclaw gateway status
```

**规避方法：**
- ✅ 定期检查 Token 有效期
- ✅ 使用环境变量存储敏感凭证
- ✅ 配置自动续期机制
- ❌ 不要硬编码 Token 到配置文件

---

## 🟠 常见坑

> ⚠️ **注意**：以下问题频繁出现，可能导致服务中断或功能异常。

### 7. OOM（内存不足）

| 项目 | 描述 |
|------|------|
| **问题** | 内存限制过低导致进程被杀 |
| **症状** | 服务突然停止、`Killed` 日志、重启循环 |
| **风险等级** | 🟠 高 |

**解决方案：**
```bash
# 1. 检查内存使用
free -h
docker stats

# 2. 增加容器内存限制
docker update --memory=2g <container-name>

# 3. 优化配置减少内存占用
# 在 config.json 中调整：
{
  "model": {
    "max_tokens": 2048,  # 降低 token 限制
    "keepalive": 300     # 缩短模型保持时间
  }
}
```

---

### 8. npm 版本陷阱

| 项目 | 描述 |
|------|------|
| **问题** | npm 版本不兼容导致依赖安装失败 |
| **症状** | `npm ERR!`、依赖冲突、安装卡住 |
| **风险等级** | 🟠 高 |

**解决方案：**
```bash
# 1. 检查 npm 版本
npm --version

# 2. 升级到推荐版本（10.x）
npm install -g npm@latest

# 3. 清理缓存重试
npm cache clean --force
npm install

# 4. 使用 nvm 管理 Node.js 版本
nvm install 20
nvm use 20
```

---

### 9. API-Key 问题

| 项目 | 描述 |
|------|------|
| **问题** | API Key 格式错误、过期或额度耗尽 |
| **症状** | `401 Unauthorized`、`429 Too Many Requests`、模型调用失败 |
| **风险等级** | 🟠 高 |

**解决方案：**
```bash
# 1. 验证 API Key 格式
echo $DASHSCOPE_API_KEY

# 2. 检查额度
# 登录对应模型服务商控制台查看

# 3. 轮换 Key
# 在 config.json 中更新：
{
  "models": {
    "default": {
      "api_key": "sk-xxxxx"
    }
  }
}

# 4. 设置重试机制
{
  "retry": {
    "max_attempts": 3,
    "backoff": "exponential"
  }
}
```

---

### 10. 端口防火墙

| 项目 | 描述 |
|------|------|
| **问题** | 防火墙阻止 Gateway 端口访问 |
| **症状** | 无法连接服务、连接超时、外部无法访问 |
| **风险等级** | 🟠 高 |

**解决方案：**
```bash
# 1. 检查防火墙状态
sudo ufw status
sudo firewall-cmd --list-all

# 2. 开放必要端口（默认 8080）
sudo ufw allow 8080/tcp
sudo firewall-cmd --permanent --add-port=8080/tcp

# 3. 验证端口监听
netstat -tlnp | grep 8080
ss -tlnp | grep 8080

# 4. 云服务器检查安全组
# AWS/Aliyun/Tencent: 控制台 → 安全组 → 添加入站规则
```

---

### 11. Docker 卷挂载

| 项目 | 描述 |
|------|------|
| **问题** | 卷挂载路径错误或权限问题 |
| **症状** | 数据不持久化、文件无法读写、空目录 |
| **风险等级** | 🟠 高 |

**解决方案：**
```bash
# 1. 使用绝对路径挂载
docker run -v /absolute/path:/data ...

# 2. 检查挂载点
docker inspect <container> | grep Mounts

# 3. 验证权限
ls -la /absolute/path

# 4. 使用命名卷（推荐）
docker volume create openclaw-data
docker run -v openclaw-data:/data ...
```

---

### 12. Node.js 版本

| 项目 | 描述 |
|------|------|
| **问题** | Node.js 版本不兼容导致运行时错误 |
| **症状** | `SyntaxError`、模块加载失败、异步问题 |
| **风险等级** | 🟠 高 |

**解决方案：**
```bash
# 1. 检查当前版本
node --version  # 推荐 18.x 或 20.x

# 2. 使用 nvm 切换版本
nvm install 20
nvm use 20
nvm alias default 20

# 3. 验证兼容性
npm ls --depth=0

# 4. 查看项目要求的版本
cat package.json | grep engines
```

---

### 13. 飞书事件订阅

| 项目 | 描述 |
|------|------|
| **问题** | 事件订阅配置错误导致消息不响应 |
| **症状** | 消息无回复、事件日志为空、回调失败 |
| **风险等级** | 🟠 高 |

**解决方案：**
```bash
# 1. 验证回调 URL 可访问
curl -X POST https://your-domain.com/callback \
  -H "Content-Type: application/json" \
  -d '{"challenge": "test"}'

# 2. 检查飞书后台配置
# 开发者后台 → 事件订阅 → 确认 URL 和事件类型

# 3. 验证加密密钥
cat ~/.openclaw/config.json | jq '.channels.feishu'

# 4. 查看事件日志
tail -f ~/.openclaw/logs/events.log
```

---

### 14. Skill 安装超时

| 项目 | 描述 |
|------|------|
| **问题** | 网络问题导致 Skill 下载超时 |
| **症状** | 安装卡住、`ETIMEDOUT`、部分文件缺失 |
| **风险等级** | 🟠 高 |

**解决方案：**
```bash
# 1. 检查网络连接
ping github.com
curl -I https://github.com

# 2. 配置代理（如需要）
export HTTPS_PROXY=http://proxy:port
export HTTP_PROXY=http://proxy:port

# 3. 手动下载安装
cd ~/.openclaw/extensions/
git clone <skill-repo-url>
openclaw skills reload

# 4. 增加超时时间
# 在 config.json 中：
{
  "network": {
    "timeout": 60000
  }
}
```

---

### 15. 端口冲突

| 项目 | 描述 |
|------|------|
| **问题** | 端口被其他进程占用 |
| **症状** | `EADDRINUSE`、服务启动失败、绑定错误 |
| **风险等级** | 🟠 高 |

**解决方案：**
```bash
# 1. 查找占用端口的进程
lsof -i :8080
netstat -tlnp | grep 8080

# 2. 杀死占用进程（谨慎）
kill -9 <PID>

# 3. 或更改 OpenClaw 端口
# config.json:
{
  "gateway": {
    "port": 8081
  }
}

# 4. 重启服务
openclaw gateway restart
```

---

### 16. WSL 持久化

| 项目 | 描述 |
|------|------|
| **问题** | WSL 重启后数据丢失或服务未自启 |
| **症状** | 配置重置、数据消失、需手动启动 |
| **风险等级** | 🟠 高 |

**解决方案：**
```bash
# 1. 将数据存到 Windows 挂载点
# 使用 /mnt/c/ 路径而非 WSL  home

# 2. 配置 systemd 自启
sudo systemctl enable openclaw
sudo systemctl start openclaw

# 3. 或使用 .bashrc 自启
echo "openclaw gateway start" >> ~/.bashrc

# 4. 定期备份配置
cp -r ~/.openclaw /mnt/c/backups/openclaw-$(date +%F)
```

---

### 17. Mac 安装

| 项目 | 描述 |
|------|------|
| **问题** | macOS 权限和签名问题导致安装失败 |
| **症状** | `Permission denied`、无法验证开发者、Gatekeeper 阻止 |
| **风险等级** | 🟠 高 |

**解决方案：**
```bash
# 1. 使用 Homebrew 安装（推荐）
brew install openclaw

# 2. 或手动授予权限
xattr -cr /Applications/OpenClaw.app

# 3. 系统设置 → 隐私与安全性 → 允许

# 4. 检查 Rosetta（M 系列芯片）
softwareupdate --install-rosetta
```

---

### 18. Ubuntu/aarch64 编译

| 项目 | 描述 |
|------|------|
| **问题** | ARM 架构编译依赖缺失或失败 |
| **症状** | 编译错误、缺少头文件、链接失败 |
| **风险等级** | 🟠 高 |

**解决方案：**
```bash
# 1. 安装构建依赖
sudo apt update
sudo apt install build-essential python3-dev

# 2. 安装架构特定依赖
sudo apt install libssl-dev:arm64

# 3. 使用预编译包（避免编译）
# 下载官方提供的 aarch64 二进制

# 4. 或使用 Docker 规避编译
docker run --platform linux/arm64 ...
```

---

### 19. 模型地域

| 项目 | 描述 |
|------|------|
| **问题** | 模型服务地域限制导致访问失败 |
| **症状** | 连接超时、`403 Forbidden`、地域错误 |
| **风险等级** | 🟠 高 |

**解决方案：**
```bash
# 1. 选择正确的服务端点
# 国内模型：dashscope.aliyuncs.com
# 国际模型：api.openai.com

# 2. 配置代理
export HTTPS_PROXY=http://proxy:port

# 3. 使用国内镜像
# config.json:
{
  "models": {
    "endpoint": "https://dashscope.aliyuncs.com"
  }
}

# 4. 测试连接
curl -I https://dashscope.aliyuncs.com
```

---

### 20. Memory 焦虑

| 项目 | 描述 |
|------|------|
| **问题** | 过度担心 Memory 不足导致频繁清理 |
| **症状** | 上下文丢失、对话不连贯、过度优化 |
| **风险等级** | 🟠 中 |

**解决方案：**
```bash
# 1. 了解实际限制
# 查看模型上下文窗口
openclaw models info

# 2. 合理配置
{
  "memory": {
    "max_tokens": 4096,
    "curate_interval": 3600
  }
}

# 3. 监控使用量
openclaw memory stats

# 4. 避免过度清理
# 让系统自动管理
```

---

### 21. Token 费用

| 项目 | 描述 |
|------|------|
| **问题** | 无意识消耗大量 Token 导致费用激增 |
| **症状** | 账单异常、额度快速耗尽、成本不可控 |
| **风险等级** | 🟠 高 |

**解决方案：**
```bash
# 1. 设置使用限额
# 在模型服务商控制台设置每日/每月限额

# 2. 监控用量
openclaw usage report

# 3. 优化 Prompt
# 减少冗余上下文，精简请求

# 4. 配置告警
# 设置用量达到 80% 时通知
```

---

### 22. cron 无响应

| 项目 | 描述 |
|------|------|
| **问题** | 定时任务不执行或静默失败 |
| **症状** | 任务未触发、无日志、配置不生效 |
| **风险等级** | 🟠 高 |

**解决方案：**
```bash
# 1. 检查 cron 服务
sudo systemctl status cron

# 2. 查看 OpenClaw 定时任务
openclaw cron list

# 3. 检查日志
tail -f ~/.openclaw/logs/cron.log

# 4. 验证 cron 表达式
# 使用 crontab.guru 验证语法

# 5. 手动触发测试
openclaw cron run --id <task-id>
```

---

### 23. 浏览器插件

| 项目 | 描述 |
|------|------|
| **问题** | 浏览器扩展冲突或配置错误 |
| **症状** | 插件不工作、页面异常、认证失败 |
| **风险等级** | 🟠 中 |

**解决方案：**
```bash
# 1. 检查插件版本
# 浏览器 → 扩展管理 → 查看版本

# 2. 重新安装插件
# 移除后从官方商店重新安装

# 3. 清除插件数据
# 扩展管理 → 详情 → 清除数据

# 4. 检查浏览器控制台
F12 → Console 查看错误
```

---

### 24. SELinux

| 项目 | 描述 |
|------|------|
| **问题** | SELinux 策略阻止文件访问 |
| **症状** | `Permission denied`、审计日志拒绝、服务异常 |
| **风险等级** | 🟠 高 |

**解决方案：**
```bash
# 1. 检查 SELinux 状态
getenforce
sestatus

# 2. 查看拒绝日志
sudo ausearch -m avc -ts recent

# 3. 临时禁用（测试用）
sudo setenforce 0

# 4. 添加正确策略（推荐）
sudo semanage fcontext -a -t httpd_sys_content_t "/path/to/openclaw(/.*)?"
sudo restorecon -Rv /path/to/openclaw

# 5. 或永久禁用（不推荐）
# /etc/selinux/config: SELINUX=disabled
```

---

## 🟡 轻微误区

> ℹ️ **提示**：以下问题影响较小，但了解后可提升使用体验。

### 25. Skills 安全假设

| 项目 | 描述 |
|------|------|
| **误区** | 认为所有 Skill 都是安全的 |
| **事实** | 社区 Skill 无审核机制，可能存在风险 |
| **建议** | 安装前审查代码，优先官方扩展 |

---

### 26. 本地模型

| 项目 | 描述 |
|------|------|
| **误区** | 认为本地模型一定更快更便宜 |
| **事实** | 本地模型需要硬件资源，可能更慢 |
| **建议** | 根据实际场景选择，小模型可本地，大模型用 API |

---

### 27. Prompt 稳定

| 项目 | 描述 |
|------|------|
| **误区** | 认为 Prompt 一旦写好就永远有效 |
| **事实** | 模型更新可能导致行为变化 |
| **建议** | 定期测试关键 Prompt，准备备选方案 |

---

### 28. doctor 检查

| 项目 | 描述 |
|------|------|
| **误区** | 认为 `doctor` 能解决所有问题 |
| **事实** | doctor 仅检查基础配置，深度问题需手动排查 |
| **建议** | 将 doctor 作为第一步，而非唯一工具 |

---

### 29. Gateway 重启

| 项目 | 描述 |
|------|------|
| **误区** | 遇到问题就重启 Gateway |
| **事实** | 重启可能掩盖真正问题，导致复发 |
| **建议** | 先查看日志定位根因，再决定是否重启 |

---

### 30. 混淆 ChatGPT

| 项目 | 描述 |
|------|------|
| **误区** | 认为 OpenClaw 就是 ChatGPT 包装 |
| **事实** | OpenClaw 是独立框架，支持多模型 |
| **建议** | 阅读官方文档了解架构和特性 |

---

### 31. 升级备份

| 项目 | 描述 |
|------|------|
| **误区** | 认为升级不会丢失数据 |
| **事实** | 重大版本升级可能破坏配置格式 |
| **建议** | 升级前完整备份配置和数据 |

---

### 32. 海外模型

| 项目 | 描述 |
|------|------|
| **误区** | 认为海外模型一定更好 |
| **事实** | 国内模型在中文场景表现优秀 |
| **建议** | 根据语言和业务场景选择模型 |

---

### 33. 事件缓存

| 项目 | 描述 |
|------|------|
| **误区** | 认为事件会永久保存 |
| **事实** | 事件日志有保留期限，定期清理 |
| **建议** | 重要事件及时导出归档 |

---

### 34. Memory curate

| 项目 | 描述 |
|------|------|
| **误区** | 认为 Memory 会自动优化 |
| **事实** | 需要配置 curate 策略才能有效管理 |
| **建议** | 根据使用频率调整 curate 间隔 |

---

### 35. 成本认知

| 项目 | 描述 |
|------|------|
| **误区** | 低估 API 调用成本 |
| **事实** | 高频调用可能产生意外费用 |
| **建议** | 设置预算告警，优化调用频率 |

---

### 36. keepalive

| 项目 | 描述 |
|------|------|
| **误区** | 认为 keepalive 越长越好 |
| **事实** | 过长 keepalive 浪费资源 |
| **建议** | 根据使用频率设置合理值（300-600 秒） |

---

### 37. Podman/Docker

| 项目 | 描述 |
|------|------|
| **误区** | 认为 Podman 完全兼容 Docker |
| **事实** | 部分命令和行为有差异 |
| **建议** | 优先使用 Docker，或充分测试 Podman 兼容性 |

---

### 38. aarch64

| 项目 | 描述 |
|------|------|
| **误区** | 认为 ARM 架构支持不完善 |
| **事实** | 主流工具已良好支持 ARM64 |
| **建议** | 选择官方提供预编译包的版本 |

---

### 39. 特殊字符

| 项目 | 描述 |
|------|------|
| **误区** | 忽略配置文件中的特殊字符 |
| **事实** | JSON 对引号、斜杠等字符敏感 |
| **建议** | 使用 JSON 验证器，注意转义 |

---

### 40. 监控需求

| 项目 | 描述 |
|------|------|
| **误区** | 认为不需要监控系统状态 |
| **事实** | 及时监控可预防大部分问题 |
| **建议** | 配置基础监控（CPU、内存、日志） |

---

### 41. Skill 离线

| 项目 | 描述 |
|------|------|
| **误区** | 认为 Skill 可以完全离线运行 |
| **事实** | 部分 Skill 需要网络访问 |
| **建议** | 检查 Skill 依赖，准备离线方案 |

---

### 42. 卸载残留

| 项目 | 描述 |
|------|------|
| **误区** | 认为卸载会清理所有文件 |
| **事实** | 配置文件和数据可能残留 |
| **建议** | 手动清理残留文件，彻底重置 |

```bash
# 完全卸载清理
openclaw gateway stop
rm -rf ~/.openclaw/
rm -rf /usr/local/lib/node_modules/openclaw/
# 重新安装
```

---

## ✅ 快速检查清单

### 安装前
- [ ] 确认系统版本兼容性（Node.js、OS 架构）
- [ ] 准备足够的磁盘空间（至少 5GB）
- [ ] 确认网络连接正常
- [ ] 备份重要数据

### 安装时
- [ ] 使用官方安装脚本（非 npm）
- [ ] 验证下载文件完整性
- [ ] 记录安装日志
- [ ] 测试基础功能

### 配置时
- [ ] 备份原始配置文件
- [ ] 使用 JSON 验证器检查语法
- [ ] 逐步修改，逐项验证
- [ ] 测试关键功能

### 运行时
- [ ] 监控资源使用（CPU、内存）
- [ ] 定期检查日志
- [ ] 验证外部连接（API、数据库）
- [ ] 测试备份恢复流程

### 维护时
- [ ] 定期更新系统和依赖
- [ ] 审计已安装 Skill
- [ ] 清理过期日志和数据
- [ ] 验证监控告警有效性

---

## 📚 推荐实践

### 1. 配置管理

```bash
# 使用版本控制管理配置
cd ~/.openclaw/
git init
git add config.json
git commit -m "Initial config"

# 每次修改后提交
git add config.json
git commit -m "Update model config"
```

### 2. 日志管理

```bash
# 配置日志轮转
# /etc/logrotate.d/openclaw
~/.openclaw/logs/*.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
}

# 实时查看日志
tail -f ~/.openclaw/logs/*.log
```

### 3. 备份策略

```bash
# 每日备份脚本
#!/bin/bash
BACKUP_DIR="/backup/openclaw"
DATE=$(date +%F)
mkdir -p $BACKUP_DIR
tar -czf $BACKUP_DIR/config-$DATE.tar.gz ~/.openclaw/config.json
# 保留 30 天
find $BACKUP_DIR -name "*.tar.gz" -mtime +30 -delete
```

### 4. 监控告警

```bash
# 简单健康检查脚本
#!/bin/bash
if ! curl -s http://localhost:8080/health > /dev/null; then
    echo "OpenClaw unhealthy" | mail -s "Alert" admin@example.com
    openclaw gateway restart
fi
```

### 5. 安全加固

```bash
# 限制文件权限
chmod 600 ~/.openclaw/config.json
chmod 700 ~/.openclaw/

# 使用环境变量存储敏感信息
export OPENCLAW_API_KEY="your-key"
# config.json 中引用：
{
  "api_key": "${OPENCLAW_API_KEY}"
}
```

---

## 🆘 紧急故障排除

### 服务无法启动

```bash
# 1. 查看日志
tail -100 ~/.openclaw/logs/error.log

# 2. 检查端口
lsof -i :8080

# 3. 验证配置
jq '.' ~/.openclaw/config.json

# 4. 重置配置
cp ~/.openclaw/config.json.example ~/.openclaw/config.json
```

### 消息不响应

```bash
# 1. 检查连接状态
openclaw gateway status

# 2. 验证渠道配置
cat ~/.openclaw/config.json | jq '.channels'

# 3. 查看事件日志
tail -f ~/.openclaw/logs/events.log

# 4. 重新授权
openclaw auth feishu  # 或 discord
```

### 内存持续增长

```bash
# 1. 监控内存
watch -n 1 'ps aux | grep openclaw'

# 2. 调整配置
{
  "memory": {
    "max_tokens": 2048,
    "curate_interval": 1800
  }
}

# 3. 重启服务
openclaw gateway restart
```

---

## 📖 参考资料

- [OpenClaw 官方文档](https://openclaw.dev/docs)
- [GitHub 仓库](https://github.com/openclaw/openclaw)
- [社区论坛](https://github.com/openclaw/openclaw/discussions)
- [问题追踪](https://github.com/openclaw/openclaw/issues)

---

## 📝 更新日志

| 版本 | 日期 | 更新内容 |
|------|------|----------|
| 1.0 | 2026-03-19 | 初始版本，包含 42 条完整经验 |

---

> **最后更新：** 2026-03-19  
> **维护者：** OpenClaw 社区  
> **许可：** MIT License

**贡献欢迎：** 如发现新问题或有更好解决方案，请提交 Issue 或 PR。
