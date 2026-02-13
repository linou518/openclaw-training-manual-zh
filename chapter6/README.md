# 第5章：多Agent协作架构

> 🎯 学习目标：设计和实现多Agent协作系统，掌握Agent间通信和任务分工

## 🏗️ **为什么需要多Agent架构？**

单个Agent虽然强大，但在复杂场景下会遇到限制：

### **单Agent的局限性**
- 🧠 **认知负载过重**: 一个Agent处理所有任务类型
- 🔄 **上下文污染**: 不同任务的信息混在一起  
- ⚡ **性能瓶颈**: 单点处理能力有限
- 🎯 **专业性不足**: 无法针对特定领域深度优化
- 🔒 **安全风险**: 所有权限集中在一个Agent

### **多Agent的优势**
- 🎯 **专业分工**: 每个Agent专注特定领域
- 🚀 **并行处理**: 同时处理多个任务
- 🔒 **权限隔离**: 按需分配最小权限
- 📊 **独立监控**: 各Agent性能可独立优化
- 🛡️ **故障隔离**: 单个Agent故障不影响整体

---

## 🏛️ **多Agent架构模式**

### **模式1: 主从架构 (Master-Worker)**
```
┌─────────────────┐
│   Master Agent  │  ← 总协调，任务分发
│   (主控制器)     │
└─────┬───────────┘
      │
   ┌──┴──┬──────┬──────┐
   ▼     ▼      ▼      ▼
┌─────┐ ┌────┐ ┌────┐ ┌─────┐
│文档 │ │代码│ │数据│ │网络│
│助手 │ │助手│ │分析│ │搜索│
└─────┘ └────┘ └────┘ └─────┘
```

**适用场景**: 有明确的任务分发和控制需求
**优点**: 架构清晰，易于管理
**缺点**: Master成为瓶颈，扩展性有限

### **模式2: 平等协作 (Peer-to-Peer)**
```
┌─────┐    ┌─────┐    ┌─────┐
│Agent│◄──►│Agent│◄──►│Agent│
│  A  │    │  B  │    │  C  │
└──┬──┘    └─────┘    └──┬──┘
   │                     │
   └──────────┬──────────┘
              ▼
           ┌─────┐
           │Agent│
           │  D  │
           └─────┘
```

**适用场景**: Agent能力相对平等，需要灵活协作
**优点**: 高可用性，无单点故障
**缺点**: 协调复杂，可能出现冲突

### **模式3: 分层架构 (Layered)**
```
┌────────────────────────────┐
│     用户界面层 (UI Layer)     │
├────────────────────────────┤  
│   业务逻辑层 (Business)      │
│ ┌────────┐ ┌──────────────┐ │
│ │项目管理│ │  个人助理     │ │
│ │Agent  │ │  Agent       │ │
│ └────────┘ └──────────────┘ │
├────────────────────────────┤
│    服务层 (Service Layer)    │ 
│ ┌─────┐ ┌─────┐ ┌─────────┐ │
│ │文档 │ │邮件 │ │ 日历    │ │
│ │服务 │ │服务 │ │ 服务    │ │
│ └─────┘ └─────┘ └─────────┘ │
├────────────────────────────┤
│     数据层 (Data Layer)      │
│   ┌─────────┐ ┌──────────┐  │
│   │文件系统 │ │  数据库   │  │
│   └─────────┘ └──────────┘  │
└────────────────────────────┘
```

**适用场景**: 复杂企业应用，需要清晰的职责分离
**优点**: 结构化，易于维护和扩展
**缺点**: 架构复杂，通信开销大

---

## 🎯 **实战：构建多Agent系统**

让我们构建一个实际的多Agent协作系统，模拟一个智能办公助理平台：

### **系统需求分析**
- 📧 **邮件管理**: 自动分类、回复、提醒
- 📅 **日程安排**: 会议安排、冲突检测、提醒
- 📝 **文档处理**: 创建、编辑、总结、分享
- 💰 **项目管理**: 任务跟踪、进度报告、资源分配
- 🔍 **信息查询**: 网络搜索、数据分析、报告生成

### **Agent角色设计**

#### **1. 主协调Agent (Coordinator)**
```json
{
  "id": "coordinator",
  "name": "主协调员",
  "role": "任务分发和协调",
  "model": "anthropic/claude-sonnet-4-20250514",
  "systemPrompt": "你是一个智能协调员，负责理解用户需求，将复杂任务分解并分配给相应的专业Agent。你需要：\n\n1. 分析用户请求的类型和复杂度\n2. 确定需要哪些Agent参与\n3. 分解任务并分配给对应Agent\n4. 协调多个Agent的工作\n5. 整合结果并向用户汇报\n\n始终保持高效和条理性。",
  "tools": {
    "allowlist": [
      "sessions_send", "sessions_list", "memory_search", 
      "memory_get", "read", "write", "message"
    ]
  }
}
```

#### **2. 邮件管理Agent (Email Manager)**
```json
{
  "id": "email-manager", 
  "name": "邮件管理员",
  "role": "邮件处理和管理",
  "model": "anthropic/claude-sonnet-4-20250514", 
  "systemPrompt": "你是专业的邮件管理助理，精通邮件分类、回复和管理。你的职责包括：\n\n1. 邮件分类和优先级排序\n2. 起草邮件回复\n3. 设置邮件提醒和跟进\n4. 邮件归档和搜索\n5. 垃圾邮件识别\n\n保持专业、礼貌和高效的邮件处理风格。",
  "tools": {
    "allowlist": ["read", "write", "web_search", "message", "cron"]
  }
}
```

#### **3. 日程管理Agent (Calendar Manager)**
```json
{
  "id": "calendar-manager",
  "name": "日程管理员", 
  "role": "日程安排和时间管理",
  "model": "anthropic/claude-sonnet-4-20250514",
  "systemPrompt": "你是高效的日程管理专家，负责时间安排和会议协调。你的核心能力：\n\n1. 智能日程安排和冲突检测\n2. 会议室预订和资源安排\n3. 提醒和通知设置\n4. 时间分析和优化建议\n5. 多时区协调\n\n始终考虑效率和用户偏好。",
  "tools": {
    "allowlist": ["read", "write", "cron", "web_search", "message"]
  }
}
```

#### **4. 文档处理Agent (Document Processor)**
```json
{
  "id": "doc-processor",
  "name": "文档处理器",
  "role": "文档创建、编辑和管理", 
  "model": "anthropic/claude-sonnet-4-20250514",
  "systemPrompt": "你是文档处理专家，擅长各种文档的创建、编辑和管理。你的专长包括：\n\n1. 文档模板创建和应用\n2. 内容总结和提取\n3. 格式化和排版\n4. 多格式转换\n5. 版本控制和协作\n\n确保文档质量和用户体验。",
  "tools": {
    "allowlist": ["read", "write", "exec", "web_search", "memory_search"]
  }
}
```

#### **5. 数据分析Agent (Data Analyst)**
```json
{
  "id": "data-analyst",
  "name": "数据分析师",
  "role": "数据分析和报告生成",
  "model": "anthropic/claude-sonnet-4-20250514",
  "systemPrompt": "你是数据分析专家，能够处理各种数据分析任务。你的核心技能：\n\n1. 数据收集和清洗\n2. 统计分析和趋势识别\n3. 图表制作和可视化\n4. 报告生成和展示\n5. 预测和建议\n\n提供准确、清晰的数据洞察。",
  "tools": {
    "allowlist": ["read", "write", "exec", "web_search", "canvas"]
  }
}
```

---

## 🔄 **Agent间通信机制**

### **1. 消息总线模式**

创建消息总线服务：

```python
# message-bus.py - Agent间消息总线
import asyncio
import json
import logging
from typing import Dict, List, Callable
from datetime import datetime

class MessageBus:
    def __init__(self):
        self.subscribers: Dict[str, List[Callable]] = {}
        self.message_history: List[Dict] = []
        
    def subscribe(self, topic: str, callback: Callable):
        """订阅消息主题"""
        if topic not in self.subscribers:
            self.subscribers[topic] = []
        self.subscribers[topic].append(callback)
        
    def publish(self, topic: str, message: Dict, sender: str = None):
        """发布消息"""
        msg = {
            "id": f"msg_{len(self.message_history)}",
            "topic": topic,
            "message": message,
            "sender": sender,
            "timestamp": datetime.now().isoformat()
        }
        
        self.message_history.append(msg)
        
        # 通知订阅者
        if topic in self.subscribers:
            for callback in self.subscribers[topic]:
                try:
                    callback(msg)
                except Exception as e:
                    logging.error(f"Message delivery failed: {e}")
                    
    def get_history(self, topic: str = None, limit: int = 100):
        """获取消息历史"""
        messages = self.message_history
        if topic:
            messages = [m for m in messages if m["topic"] == topic]
        return messages[-limit:]

# 使用示例
bus = MessageBus()

# Coordinator 订阅任务结果
def handle_task_result(message):
    print(f"Task completed: {message['message']}")

bus.subscribe("task.completed", handle_task_result)

# Email Agent 发布任务完成消息
bus.publish("task.completed", {
    "task_id": "email_001",
    "status": "success",
    "result": "3 emails processed"
}, sender="email-manager")
```

### **2. 直接通信模式**

使用OpenClaw的内置通信机制：

```python
# 在Coordinator Agent中
async def delegate_task(task_type: str, task_data: dict):
    """将任务委托给专业Agent"""
    
    agent_mapping = {
        "email": "email-manager",
        "calendar": "calendar-manager", 
        "document": "doc-processor",
        "analysis": "data-analyst"
    }
    
    target_agent = agent_mapping.get(task_type)
    if not target_agent:
        return {"error": "Unknown task type"}
    
    # 使用sessions_send发送任务
    message = f"请处理以下任务: {json.dumps(task_data, ensure_ascii=False)}"
    
    result = await sessions_send(
        sessionKey=f"agent:{target_agent}:main",
        message=message,
        timeoutSeconds=60
    )
    
    return result
```

### **3. 工作流编排**

创建复杂的多Agent工作流：

```json
{
  "workflow": {
    "name": "智能邮件处理流程",
    "steps": [
      {
        "id": "email_classification",
        "agent": "email-manager", 
        "action": "classify_emails",
        "input": "${user_input}",
        "output": "email_categories"
      },
      {
        "id": "urgent_handling",
        "agent": "coordinator",
        "condition": "${email_categories.urgent_count} > 0",
        "action": "handle_urgent_emails", 
        "input": "${email_categories.urgent_emails}"
      },
      {
        "id": "schedule_check",
        "agent": "calendar-manager",
        "parallel": true,
        "action": "check_calendar_conflicts",
        "input": "${email_categories.meeting_requests}"
      },
      {
        "id": "generate_report",
        "agent": "doc-processor",
        "action": "create_email_summary",
        "input": {
          "classified": "${email_categories}",
          "calendar": "${schedule_check.result}"
        }
      }
    ]
  }
}
```

---

## 📝 **配置多Agent系统**

### **完整的openclaw.json配置**

```json
{
  "gateway": {
    "port": 18789,
    "bind": "loopback",
    "cors": true,
    "maxConcurrency": 20
  },
  "agents": [
    {
      "id": "coordinator",
      "name": "主协调员",
      "model": "anthropic/claude-sonnet-4-20250514",
      "systemPrompt": "你是智能办公协调员...",
      "workspace": {"root": "./agents/coordinator"},
      "maxConcurrency": 5
    },
    {
      "id": "email-manager", 
      "name": "邮件管理员",
      "model": "anthropic/claude-sonnet-4-20250514",
      "systemPrompt": "你是专业邮件管理助理...",
      "workspace": {"root": "./agents/email-manager"},
      "maxConcurrency": 3
    },
    {
      "id": "calendar-manager",
      "name": "日程管理员",
      "model": "anthropic/claude-sonnet-4-20250514", 
      "systemPrompt": "你是高效日程管理专家...",
      "workspace": {"root": "./agents/calendar-manager"},
      "maxConcurrency": 2
    },
    {
      "id": "doc-processor",
      "name": "文档处理器",
      "model": "anthropic/claude-sonnet-4-20250514",
      "systemPrompt": "你是文档处理专家...",
      "workspace": {"root": "./agents/doc-processor"},
      "maxConcurrency": 3
    },
    {
      "id": "data-analyst",
      "name": "数据分析师", 
      "model": "anthropic/claude-sonnet-4-20250514",
      "systemPrompt": "你是数据分析专家...",
      "workspace": {"root": "./agents/data-analyst"},
      "maxConcurrency": 2
    }
  ],
  "channels": {
    "telegram": {
      "accounts": [
        {
          "name": "office-coordinator",
          "botToken": "YOUR_BOT_TOKEN",
          "binding": "coordinator"
        }
      ]
    }
  },
  "tools": {
    "allowlists": {
      "coordinator": [
        "sessions_send", "sessions_list", "memory_search", 
        "memory_get", "read", "write", "message"
      ],
      "email-manager": [
        "read", "write", "web_search", "message", "cron"
      ],
      "calendar-manager": [
        "read", "write", "cron", "web_search", "message"
      ],
      "doc-processor": [
        "read", "write", "exec", "web_search", "memory_search"
      ],
      "data-analyst": [
        "read", "write", "exec", "web_search", "canvas"
      ]
    }
  },
  "auth": {
    "profiles": [
      {"id": "anthropic", "provider": "anthropic"}
    ]
  }
}
```

---

## 🚀 **部署和测试**

### **创建Agent工作空间**

```bash
# 创建各Agent的工作目录
mkdir -p agents/{coordinator,email-manager,calendar-manager,doc-processor,data-analyst}

# 为每个Agent创建基础文件
for agent in coordinator email-manager calendar-manager doc-processor data-analyst; do
  mkdir -p agents/$agent/{memory,logs,projects}
  touch agents/$agent/{MEMORY.md,SOUL.md,USER.md}
done
```

### **启动系统**
```bash
# 启动OpenClaw网关
openclaw gateway start

# 验证所有Agent加载成功
openclaw status

# 检查Agent通信
curl -X POST http://localhost:18789/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "coordinator",
    "messages": [
      {"role": "user", "content": "请介绍一下团队成员"}
    ]
  }'
```

### **多Agent协作测试**

```bash
# 测试任务分发
{
  "model": "coordinator", 
  "messages": [
    {
      "role": "user",
      "content": "我需要处理今天的邮件，安排明天的会议，并准备一份周报。请帮我协调完成这些任务。"
    }
  ]
}

# 期望的工作流程:
# 1. Coordinator 分析任务
# 2. 分别联系 email-manager, calendar-manager, doc-processor
# 3. 各Agent执行专业任务
# 4. Coordinator 整合结果并汇报
```

---

## 📊 **监控和管理**

### **Agent状态监控**
```python
# agent-monitor.py - Agent状态监控脚本
import asyncio
import aiohttp
import json
from datetime import datetime

async def check_agent_status(agent_id: str):
    """检查单个Agent状态"""
    try:
        async with aiohttp.ClientSession() as session:
            async with session.post(
                "http://localhost:18789/v1/chat/completions",
                json={
                    "model": agent_id,
                    "messages": [{"role": "user", "content": "ping"}],
                    "max_tokens": 10
                }
            ) as response:
                if response.status == 200:
                    data = await response.json()
                    return {
                        "agent": agent_id,
                        "status": "healthy",
                        "response_time": response.headers.get("X-Response-Time"),
                        "timestamp": datetime.now().isoformat()
                    }
                else:
                    return {
                        "agent": agent_id,
                        "status": "unhealthy", 
                        "error": f"HTTP {response.status}",
                        "timestamp": datetime.now().isoformat()
                    }
    except Exception as e:
        return {
            "agent": agent_id,
            "status": "error",
            "error": str(e),
            "timestamp": datetime.now().isoformat()
        }

async def monitor_all_agents():
    """监控所有Agent"""
    agents = ["coordinator", "email-manager", "calendar-manager", 
              "doc-processor", "data-analyst"]
    
    tasks = [check_agent_status(agent) for agent in agents]
    results = await asyncio.gather(*tasks)
    
    print("=== Agent Status Report ===")
    for result in results:
        status_icon = "✅" if result["status"] == "healthy" else "❌"
        print(f"{status_icon} {result['agent']}: {result['status']}")
        if "error" in result:
            print(f"   Error: {result['error']}")

# 运行监控
if __name__ == "__main__":
    asyncio.run(monitor_all_agents())
```

### **性能分析**
```python
# performance-tracker.py - Agent性能追踪
import time
import json
from collections import defaultdict, deque

class AgentPerformanceTracker:
    def __init__(self, window_size: int = 100):
        self.response_times = defaultdict(lambda: deque(maxlen=window_size))
        self.error_counts = defaultdict(int)
        self.request_counts = defaultdict(int)
        
    def record_request(self, agent_id: str, response_time: float, success: bool):
        """记录请求性能数据"""
        self.request_counts[agent_id] += 1
        self.response_times[agent_id].append(response_time)
        
        if not success:
            self.error_counts[agent_id] += 1
            
    def get_statistics(self, agent_id: str):
        """获取Agent性能统计"""
        times = list(self.response_times[agent_id])
        if not times:
            return None
            
        return {
            "agent_id": agent_id,
            "total_requests": self.request_counts[agent_id],
            "error_count": self.error_counts[agent_id],
            "error_rate": self.error_counts[agent_id] / self.request_counts[agent_id],
            "avg_response_time": sum(times) / len(times),
            "min_response_time": min(times),
            "max_response_time": max(times)
        }
        
    def get_all_statistics(self):
        """获取所有Agent的性能统计"""
        return {
            agent_id: self.get_statistics(agent_id)
            for agent_id in self.request_counts.keys()
        }

# 使用示例
tracker = AgentPerformanceTracker()

# 模拟记录性能数据
tracker.record_request("coordinator", 1.2, True)
tracker.record_request("email-manager", 0.8, True)
tracker.record_request("coordinator", 15.0, False)  # 超时失败

# 获取性能报告
stats = tracker.get_all_statistics()
print(json.dumps(stats, indent=2))
```

---

## 🛠️ **优化和最佳实践**

### **1. 负载均衡**
```json
{
  "agents": [
    {
      "id": "email-manager-1",
      "name": "邮件管理员-1",
      "model": "anthropic/claude-sonnet-4-20250514",
      "loadBalancing": {
        "enabled": true,
        "weight": 1
      }
    },
    {
      "id": "email-manager-2", 
      "name": "邮件管理员-2",
      "model": "anthropic/claude-sonnet-4-20250514",
      "loadBalancing": {
        "enabled": true,
        "weight": 1
      }
    }
  ],
  "loadBalancer": {
    "strategy": "round-robin",  // round-robin, least-connections, weighted
    "healthCheck": {
      "enabled": true,
      "interval": 30000,
      "timeout": 5000
    }
  }
}
```

### **2. 故障恢复**
```json
{
  "agents": [
    {
      "id": "coordinator",
      "retry": {
        "enabled": true,
        "maxAttempts": 3,
        "backoff": "exponential"
      },
      "circuit": {
        "enabled": true,
        "threshold": 5,
        "timeout": 60000
      },
      "fallback": {
        "agent": "backup-coordinator",
        "message": "主协调员暂时不可用，由备用协调员为您服务"
      }
    }
  ]
}
```

### **3. 缓存优化**
```json
{
  "agents": [
    {
      "id": "data-analyst",
      "caching": {
        "enabled": true,
        "ttl": 3600,  // 1小时
        "maxSize": "100MB",
        "strategy": "LRU"
      }
    }
  ]
}
```

---

## ✅ **本章总结**

通过本章学习，你应该已经掌握：

- [x] 多Agent架构的设计模式和适用场景
- [x] Agent间通信机制的实现方法
- [x] 完整多Agent系统的配置和部署
- [x] 任务分发和协作流程的设计
- [x] 系统监控和性能优化方法
- [x] 故障处理和负载均衡策略

---

## 🚀 **下一步**

现在你已经掌握了多Agent架构，接下来可以：

**[下一章：消息渠道集成 →](../06-channels/README.md)**

了解如何将多Agent系统与各种消息平台集成，实现更丰富的用户交互体验。

---

## 📝 **实践项目**

### **项目1: 智能客服系统**
- 设计3-5个专业Agent (售前、售后、技术支持等)
- 实现自动问题分类和转接
- 建立知识库和FAQ系统

### **项目2: 内容创作工厂** 
- 创建多个内容创作Agent (写作、设计、视频等)
- 实现创作流程的自动化编排
- 建立质量控制和审核机制

### **项目3: 企业自动化平台**
- 设计部门专用Agent (HR、财务、IT等)
- 实现跨部门协作工作流
- 建立权限管理和审批流程

**准备好挑战更复杂的系统集成了吗？** 🎯