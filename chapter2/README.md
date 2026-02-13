# 第2章：环境准备和安装

> 🎯 学习目标：完成OpenClaw的安装配置，成功启动第一个Agent

## 🔍 **环境准备检查清单**

在开始之前，确保你的环境满足以下要求：

### **📋 硬件需求**

| 部署模式 | CPU | 内存 | 磁盘 | 网络 |
|----------|-----|------|------|------|
| 开发学习 | 2核 | 4GB | 20GB | 稳定互联网 |
| 小规模生产 | 4核 | 8GB | 50GB | 10Mbps+ |
| 企业级 | 8核+ | 16GB+ | 100GB+ | 100Mbps+ |

### **💻 操作系统支持**

✅ **Ubuntu 22.04 LTS** (推荐)  
✅ **Debian 11+**  
✅ **CentOS 8+**  
✅ **macOS 12+**  
✅ **Windows 11** (WSL2)

### **🛠️ 必需软件**

- **Node.js 18+** (必需)
- **npm 8+** (必需)  
- **Git 2.0+** (必需)
- **Docker 20+** (可选，用于容器化)
- **SSL证书** (可选，用于HTTPS)

---

## 🚀 **安装方法对比**

| 方法 | 适用场景 | 优点 | 缺点 | 时间 |
|------|----------|------|------|------|
| **一键脚本** | 新手快速体验 | 全自动化 | 定制性差 | 5-10分钟 |
| **手动安装** | 生产环境 | 完全控制 | 步骤较多 | 20-30分钟 |
| **Docker部署** | 容器化环境 | 环境隔离 | 需要Docker知识 | 10-15分钟 |
| **集群部署** | 企业级应用 | 高可用性 | 复杂度高 | 1-2小时 |

---

## 🎯 **方法1: 一键安装脚本** ⭐推荐新手

### **下载并运行脚本**
```bash
# 下载安装脚本
curl -sSL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/install.sh -o install-openclaw.sh

# 查看脚本内容（建议）
cat install-openclaw.sh

# 运行安装脚本
chmod +x install-openclaw.sh
./install-openclaw.sh
```

### **脚本功能**
✅ 自动检测操作系统  
✅ 安装Node.js和依赖  
✅ 下载OpenClaw CLI  
✅ 创建工作目录结构  
✅ 生成初始配置文件  
✅ 设置环境变量  

### **安装过程预览**
```
🔍 检测操作系统: ubuntu
✅ Node.js版本符合要求: v20.10.0  
📦 安装OpenClaw CLI...
📁 创建工作目录: ~/.openclaw/workspace-main
⚙️  生成配置文件: openclaw.json
🎉 安装完成！
```

---

## 🛠️ **方法2: 手动分步安装**

### **步骤1: 安装Node.js**

#### **Ubuntu/Debian**
```bash
# 添加NodeSource repository
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -

# 安装Node.js
sudo apt-get install -y nodejs

# 验证安装
node --version  # 应显示 v20.x.x
npm --version   # 应显示 9.x.x
```

#### **CentOS/RHEL**
```bash
# 添加NodeSource repository  
curl -fsSL https://rpm.nodesource.com/setup_20.x | sudo bash -

# 安装Node.js
sudo yum install -y nodejs

# 验证安装
node --version
npm --version
```

#### **macOS**
```bash
# 使用Homebrew
brew install node

# 或下载官方安装包
# https://nodejs.org/en/download/
```

### **步骤2: 安装OpenClaw CLI**
```bash
# 全局安装OpenClaw
sudo npm install -g openclaw

# 验证安装
openclaw --version
openclaw help
```

### **步骤3: 创建工作目录**
```bash
# 创建OpenClaw主目录
mkdir -p ~/.openclaw/workspace-main
cd ~/.openclaw/workspace-main

# 初始化git仓库
git init
git config --local user.name "Your Name"
git config --local user.email "your.email@example.com"

# 创建目录结构
mkdir -p {memory,skills,projects,logs}
```

### **步骤4: 初始化配置**
```bash
# 交互式配置向导
openclaw onboard

# 或手动设置配置文件
openclaw setup

# 或手动创建openclaw.json
cat > openclaw.json << 'EOF'
{
  "gateway": {
    "port": 18789,
    "bind": "loopback"
  },
  "agents": [
    {
      "id": "main",
      "name": "主助理", 
      "model": "anthropic/claude-sonnet-4-20250514",
      "workspace": {
        "root": "."
      }
    }
  ],
  "auth": {
    "profiles": [
      {
        "id": "anthropic",
        "provider": "anthropic"
      }
    ]
  }
}
EOF
```

---

## 🐳 **方法3: Docker容器化部署**

### **创建Dockerfile**
```dockerfile
# 基于Node.js官方镜像
FROM node:20-slim

# 设置工作目录
WORKDIR /app

# 安装系统依赖
RUN apt-get update && apt-get install -y \
    git \
    curl \
    && rm -rf /var/lib/apt/lists/*

# 安装OpenClaw
RUN npm install -g openclaw

# 创建工作目录
RUN mkdir -p /app/workspace
WORKDIR /app/workspace

# 复制配置文件
COPY openclaw.json .
COPY auth-profiles.json .

# 暴露端口
EXPOSE 18789

# 启动命令
CMD ["openclaw", "gateway", "start", "--port", "18789"]
```

### **Docker Compose部署**
```yaml
version: '3.8'

services:
  openclaw-gateway:
    build: .
    ports:
      - "18789:18789"
    volumes:
      - ./workspace:/app/workspace
      - ./logs:/app/logs
    environment:
      - NODE_ENV=production
    restart: unless-stopped

  openclaw-agent-1:
    build: .
    ports:
      - "18790:18789"  
    volumes:
      - ./agents/agent-1:/app/workspace
    environment:
      - AGENT_ID=agent-1
      - NODE_ENV=production
    restart: unless-stopped
```

---

## 🔧 **API密钥配置**

安装完成后，需要配置AI模型的API密钥：

### **配置Anthropic Claude**
```bash
# 交互式配置
openclaw auth add anthropic

# 或直接设置环境变量
export ANTHROPIC_API_KEY="sk-ant-api03-..."

# 验证配置
openclaw auth list
```

### **配置OpenAI GPT**
```bash
# 添加OpenAI认证
openclaw auth add openai

# 设置API密钥
export OPENAI_API_KEY="sk-..."

# 测试连接
openclaw test openai
```

### **其他模型配置**
```bash
# Google Gemini
export GOOGLE_API_KEY="..."
openclaw auth add google

# Azure OpenAI  
export AZURE_OPENAI_API_KEY="..."
export AZURE_OPENAI_ENDPOINT="https://..."
openclaw auth add azure-openai
```

---

## 🎮 **首次启动和测试**

### **启动网关**
```bash
# 进入工作目录
cd ~/.openclaw/workspace-main

# 启动网关（前台运行）
openclaw gateway start

# 或后台运行
openclaw gateway start --daemon

# 查看状态
openclaw status
```

### **测试基本功能**
```bash
# 在另一个终端中测试
curl -X POST http://localhost:18789/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "main",
    "messages": [
      {"role": "user", "content": "Hello, OpenClaw!"}
    ]
  }'
```

### **期望的响应**
```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "model": "main", 
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "你好！我是OpenClaw AI助理，很高兴为您服务！"
      }
    }
  ]
}
```

---

## ❗ **常见问题和解决方案**

### **问题1: Node.js版本过低**
```bash
# 错误信息
Error: OpenClaw requires Node.js 18 or higher

# 解决方案
# 卸载旧版本
sudo apt remove nodejs npm

# 重新安装最新版本
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
```

### **问题2: 权限错误**
```bash
# 错误信息  
EACCES: permission denied, mkdir '/usr/local/lib/node_modules/@openclaw'

# 解决方案：配置npm全局目录
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
npm install -g @openclaw/openclaw
```

### **问题3: 端口被占用**
```bash
# 错误信息
Error: listen EADDRINUSE :::18789

# 解决方案：查找占用端口的进程
sudo netstat -tlnp | grep 18789
sudo kill -9 <PID>

# 或使用其他端口
openclaw gateway start --port 18790
```

### **问题4: API密钥无效**
```bash
# 错误信息
AuthenticationError: Invalid API key

# 解决方案：重新配置密钥
openclaw auth remove anthropic
openclaw auth add anthropic
# 输入正确的API密钥

# 验证配置
openclaw auth test anthropic
```

### **问题5: 网络连接问题**
```bash
# 错误信息
fetch failed

# 解决方案：检查网络和防火墙
# 测试网络连接
curl -I https://api.anthropic.com
curl -I https://api.openai.com

# 配置代理（如需要）
export HTTPS_PROXY=http://proxy.company.com:8080
export HTTP_PROXY=http://proxy.company.com:8080
```

---

## ✅ **安装验证清单**

安装完成后，逐项检查：

- [ ] **Node.js版本**: `node --version` >= v18.0.0
- [ ] **OpenClaw CLI**: `openclaw --version` 显示版本信息
- [ ] **工作目录**: `~/.openclaw/workspace-main` 存在
- [ ] **配置文件**: `openclaw.json` 格式正确
- [ ] **API密钥**: `openclaw auth list` 显示已配置的provider
- [ ] **网关启动**: `openclaw gateway start` 无错误
- [ ] **API测试**: HTTP请求返回正常响应
- [ ] **日志记录**: `logs/gateway.log` 有正常日志

---

## 📊 **性能优化建议**

### **系统级优化**
```bash
# 增加文件描述符限制
echo "* soft nofile 65536" | sudo tee -a /etc/security/limits.conf
echo "* hard nofile 65536" | sudo tee -a /etc/security/limits.conf

# 优化Node.js内存
export NODE_OPTIONS="--max-old-space-size=4096"

# 启用高性能模式
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

### **OpenClaw配置优化**
```json
{
  "gateway": {
    "maxConcurrency": 10,
    "timeout": 30000,
    "keepAlive": true
  },
  "agents": [
    {
      "id": "main",
      "maxConcurrency": 3,
      "contextPruning": {
        "enabled": true,
        "maxTokens": 100000
      }
    }
  ]
}
```

---

## 🚀 **下一步**

恭喜！OpenClaw已经成功安装。现在你可以：

**[下一章：创建第一个Agent →](../03-first-agent/README.md)**

---

## 📝 **本章总结**

通过本章学习，你应该已经：

- [x] 完成OpenClaw的安装和基础配置
- [x] 掌握一键脚本和手动安装两种方法
- [x] 配置了AI模型的API密钥
- [x] 成功启动了OpenClaw网关
- [x] 了解了常见问题的解决方案
- [x] 知道如何进行性能优化

**准备好创建你的第一个Agent了吗？** 🎯