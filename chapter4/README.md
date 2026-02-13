# 第4章：工具和技能使用

> 🎯 学习目标：掌握OpenClaw内置工具的使用方法，学会安装和使用Skills技能包，能够创建自定义工具

## 📚 **工具系统概览**

OpenClaw的强大之处在于其丰富的工具生态系统。Agent可以通过工具与外部世界交互，执行各种任务。

### **工具分类**
- 🔧 **内置工具**: OpenClaw核心提供的基础工具
- 🎨 **Skills技能**: 可安装的扩展功能包
- 🛠️ **自定义工具**: 用户开发的专用工具

---

## 🔧 **内置工具详解**

### **4.1 文件操作工具**

#### **read - 读取文件内容**
```bash
# 基础用法
read file_path="example.txt"

# 分页读取大文件
read file_path="large_log.txt" offset=100 limit=50

# 读取图片文件
read file_path="screenshot.png"  # 自动识别格式
```

**实际示例:**
```bash
# Agent读取配置文件
read file_path="~/.openclaw/openclaw.json"

# 检查日志文件
read file_path="/var/log/openclaw.log" limit=20
```

#### **write - 创建或覆写文件**
```bash
# 创建新文件
write file_path="output.txt" content="Hello OpenClaw"

# 覆写现有文件  
write path="config.json" content='{"debug": true}'
```

**安全注意事项:**
- write会完全覆盖现有文件
- 建议先用read检查文件是否存在
- 重要文件操作前先备份

#### **edit - 精确编辑文件**
```bash
# 替换特定内容
edit file_path="config.txt" 
     old_string="debug=false" 
     new_string="debug=true"
```

**最佳实践:**
```bash
# 1. 先读取确认内容
read file_path="config.conf"

# 2. 精确替换（匹配完整字符串包括空格）
edit file_path="config.conf"
     old_string="# production mode
server.debug = false"
     new_string="# production mode  
server.debug = true"

# 3. 验证修改结果
read file_path="config.conf"
```

### **4.2 命令执行工具**

#### **exec - 执行Shell命令**
```bash
# 基础命令执行
exec command="ls -la"

# 带工作目录
exec command="git status" workdir="/project/path"

# 后台执行
exec command="long_running_task" background=true

# 设置环境变量
exec command="npm start" env={"NODE_ENV": "production"}
```

**安全配置:**
```json
// openclaw.json中的安全设置
{
  "tools": {
    "exec": {
      "security": "allowlist",  // deny | allowlist | full
      "allowCommands": ["git", "npm", "docker", "ls", "cat"],
      "denyPaths": ["/etc", "/root", "/usr/bin/rm"]
    }
  }
}
```

#### **process - 管理后台进程**
```bash
# 列出活跃进程
process action="list"

# 查看进程输出
process action="log" sessionId="abc123"

# 向进程发送输入
process action="write" sessionId="abc123" data="exit\n"

# 终止进程
process action="kill" sessionId="abc123"
```

**实际应用场景:**
```bash
# 启动长时间运行的服务
exec command="python3 web_scraper.py" background=true

# 监控进程状态
process action="poll" sessionId="scraper-session"

# 实时查看日志
process action="log" sessionId="scraper-session" limit=50
```

### **4.3 网络工具**

#### **web_search - 网页搜索**
```bash
# 基础搜索
web_search query="OpenClaw documentation"

# 限制结果数量
web_search query="AI assistant deployment" count=5

# 区域化搜索
web_search query="人工智能" country="CN" search_lang="zh"

# 时间范围过滤
web_search query="OpenClaw news" freshness="pw"  # 过去一周
```

#### **web_fetch - 获取网页内容**
```bash
# 获取网页内容
web_fetch url="https://docs.openclaw.ai/quickstart"

# 只提取文本
web_fetch url="https://example.com" extractMode="text"

# 限制内容长度
web_fetch url="https://long-article.com" maxChars=5000
```

#### **browser - 浏览器自动化**
```bash
# 打开网页
browser action="open" targetUrl="https://github.com"

# 截图
browser action="screenshot"

# 点击元素
browser action="act" request={"kind": "click", "ref": "login-button"}

# 填写表单
browser action="act" request={
  "kind": "fill",
  "ref": "username-field", 
  "text": "myusername"
}
```

### **4.4 消息工具**

#### **message - 发送消息**
```bash
# 发送简单消息
message action="send" target="user123" message="Hello!"

# 指定渠道
message action="send" channel="telegram" target="@mychat" message="Status update"

# 发送文件
message action="send" target="user123" media="/path/to/file.pdf"
```

#### **tts - 文字转语音**
```bash
# 生成语音文件
tts text="欢迎使用OpenClaw助理" channel="telegram"
```

---

## 🎨 **Skills技能系统**

### **4.5 什么是Skills**

Skills是OpenClaw的扩展功能包，为Agent提供专业领域的能力。

**核心概念:**
- **Skill包**: 包含SKILL.md配置和相关脚本/资源
- **技能描述**: AI模型理解的功能说明
- **工具集成**: Skills可以调用工具完成复杂任务

### **4.6 安装和使用Skills**

#### **从ClawHub安装**
```bash
# 列出可用技能
openclaw skills list

# 安装天气技能
openclaw skills install weather

# 更新已安装技能
openclaw skills update weather

# 卸载技能
openclaw skills uninstall weather
```

#### **本地技能开发**
```bash
# 创建技能目录
mkdir -p ~/.openclaw/workspace-main/skills/my-custom-skill

# 创建技能配置
cat > ~/.openclaw/workspace-main/skills/my-custom-skill/SKILL.md << 'EOF'
---
name: my-custom-skill
description: Custom automation for specific tasks
version: 1.0.0
---

# My Custom Skill

This skill provides automation for...

## Usage
When user asks for X, do Y using the following tools:
1. tool1 with parameters...
2. tool2 with results from step 1...
EOF
```

### **4.7 常用Skills推荐**

| 技能名 | 功能 | 适用场景 |
|--------|------|----------|
| weather | 天气查询 | 日常助理、出行规划 |
| healthcheck | 系统监控 | 服务器运维、安全检查 |
| slack | Slack集成 | 企业协作、通知管理 |
| continuous-learning-v2 | 学习记录 | 知识管理、经验积累 |

#### **实际使用示例**

**天气技能使用:**
```bash
# Agent会自动调用weather技能
User: "今天东京天气如何？"
Agent: [调用weather技能] -> "东京今天多云，15-22°C，降雨概率20%"
```

**健康检查技能:**
```bash
# 系统自动健康检查
User: "检查服务器状态"
Agent: [调用healthcheck技能] -> 生成安全报告和优化建议
```

---

## 🛠️ **工具安全和配置**

### **4.8 安全配置最佳实践**

#### **工具权限控制**
```json
{
  "tools": {
    "exec": {
      "security": "allowlist",
      "allowCommands": [
        "git", "npm", "yarn", "docker", "kubectl",
        "ls", "cat", "grep", "find", "ps"
      ],
      "denyCommands": ["rm", "sudo", "chmod", "chown"],
      "denyPaths": ["/etc", "/root", "/bin", "/sbin"]
    },
    "read": {
      "denyPaths": ["/etc/passwd", "/etc/shadow", "~/.ssh"]
    },
    "write": {
      "denyPaths": ["/etc", "/root", "/sys", "/proc"]
    }
  }
}
```

#### **网络访问控制**
```json
{
  "tools": {
    "web_fetch": {
      "allowDomains": ["docs.openclaw.ai", "api.github.com"],
      "denyDomains": ["internal.company.com"],
      "maxContentSize": "10MB"
    },
    "browser": {
      "headless": true,
      "allowedDomains": ["trusted-site.com"],
      "timeout": 30000
    }
  }
}
```

### **4.9 工具使用最佳实践**

#### **错误处理**
```bash
# 好的做法: 检查文件是否存在
read file_path="config.json"
if [[ $? -eq 0 ]]; then
  edit file_path="config.json" old_string="..." new_string="..."
else
  write file_path="config.json" content='{"default": true}'
fi
```

#### **日志记录**
```bash
# 重要操作记录日志
echo "$(date): Starting backup process" >> backup.log
exec command="rsync -av /data/ /backup/" workdir="/scripts"
echo "$(date): Backup completed" >> backup.log
```

#### **资源清理**
```bash
# 清理临时文件
exec command="find /tmp -name 'openclaw-*' -mtime +1 -delete"

# 清理后台进程
process action="list" | grep "completed" | process action="kill"
```

---

## 🎯 **实战演练**

### **4.10 综合案例: 自动化报告生成**

**需求**: 每日自动生成系统状态报告

```bash
#!/bin/bash
# 自动化报告生成流程

# 1. 收集系统信息
exec command="uptime" > system_uptime
exec command="df -h" > disk_usage  
exec command="free -h" > memory_usage

# 2. 检查服务状态
exec command="systemctl status openclaw-gateway" > gateway_status
exec command="docker ps" > container_status

# 3. 获取最新日志
read file_path="/var/log/openclaw.log" offset=1000 limit=100 > recent_logs

# 4. 编译报告
write file_path="daily_report_$(date +%Y%m%d).md" content="
# 系统状态报告 - $(date)

## 系统概况
$(cat system_uptime)

## 磁盘使用
\`\`\`
$(cat disk_usage)
\`\`\`

## 内存使用  
\`\`\`
$(cat memory_usage)
\`\`\`

## 服务状态
$(cat gateway_status)

## 最近日志
\`\`\`
$(cat recent_logs)
\`\`\`
"

# 5. 发送通知
message action="send" target="admin@company.com" 
        message="日报已生成" 
        media="daily_report_$(date +%Y%m%d).md"

# 6. 清理临时文件
exec command="rm -f system_* disk_* memory_* gateway_* container_* recent_logs"
```

**定时执行设置:**
```bash
# 使用OpenClaw的cron功能
openclaw cron add --name "daily-report" 
                  --schedule "0 8 * * *" 
                  --command "bash report_generator.sh"
```

### **4.11 故障排除指南**

#### **常见错误及解决方案**

**错误1: exec工具被拒绝**
```
Error: Command 'sudo apt update' denied by security policy
```
**解决:**
```json
// 修改openclaw.json
{
  "tools": {
    "exec": {
      "security": "allowlist",
      "allowCommands": ["sudo"]  // 添加sudo到允许列表
    }
  }
}
```

**错误2: 文件权限不足**
```
Error: EACCES: permission denied, open '/etc/hosts'
```
**解决:**
```bash
# 方法1: 修改文件权限
sudo chmod 644 /etc/hosts

# 方法2: 使用sudo执行
exec command="sudo cat /etc/hosts"
```

**错误3: 网络请求超时**
```
Error: Request timeout after 30000ms
```
**解决:**
```json
{
  "tools": {
    "web_fetch": {
      "timeout": 60000,  // 增加超时时间
      "retry": 3         // 重试次数
    }
  }
}
```

---

## 📋 **本章总结**

### **关键要点**
1. **工具分类**: 内置工具 + Skills技能 + 自定义开发
2. **安全第一**: 严格配置工具权限，使用allowlist模式
3. **错误处理**: 检查返回值，记录日志，优雅降级
4. **最佳实践**: 测试验证、资源清理、安全配置

### **下一步学习**
- 第5章: 学习如何连接不同的消息渠道
- 第6章: 掌握多Agent协作架构
- 第7章: 深入理解记忆管理机制

### **练习建议**
1. 尝试每个内置工具的基础用法
2. 安装并使用2-3个常用Skills
3. 创建一个自定义的自动化脚本
4. 配置适合你环境的安全策略

---

**📚 延伸阅读:**
- [OpenClaw官方工具文档](https://docs.openclaw.ai/tools)
- [ClawHub技能市场](https://clawhub.com)
- [安全配置最佳实践](https://docs.openclaw.ai/security)