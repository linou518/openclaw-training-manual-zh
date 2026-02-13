# 第8章：监控和调试技术

> 🎯 学习目标：建立完善的OpenClaw监控体系，掌握性能调试方法，实现故障快速定位和自动化运维

## 📊 **监控体系概览**

一个生产级的OpenClaw部署需要全方位的监控能力：
- 🔍 **实时监控**: 系统状态、性能指标、错误率
- 📝 **日志管理**: 结构化日志、集中收集、智能分析
- ⚠️ **告警机制**: 异常检测、分级告警、自动响应
- 📈 **可视化**: 仪表板、趋势分析、容量规划

---

## 🏗️ **监控架构设计**

### **8.1 监控层次模型**

```
┌─────────────────────────────────────────┐
│              应用层监控                    │
│    Agent性能 | Session状态 | 工具调用     │
├─────────────────────────────────────────┤
│              Gateway监控                  │
│    连接数 | 响应时间 | 吞吐量 | 错误率     │
├─────────────────────────────────────────┤
│              系统层监控                    │
│    CPU | Memory | Disk | Network        │
├─────────────────────────────────────────┤
│              基础设施监控                   │
│    服务器 | 网络 | 存储 | 容器             │
└─────────────────────────────────────────┘
```

### **8.2 监控数据流**

```
数据采集 → 数据传输 → 数据存储 → 数据分析 → 可视化展示 → 告警响应
    ↓         ↓         ↓         ↓         ↓         ↓
  Agent     Message   Time      Analysis  Dashboard  Alert
 Metrics    Queue     Series     Engine    Panel     System
```

---

## 📋 **OpenClaw内置监控**

### **8.3 Gateway状态监控**

#### **基础状态查询**
```bash
# 全面系统状态
openclaw status

# 详细监控信息
openclaw status --all --deep

# JSON格式输出(便于脚本处理)
openclaw status --json
```

#### **重要监控指标**
```json
{
  "gateway": {
    "uptime": "72h 15m",
    "version": "2026.2.9", 
    "connections": 42,
    "requests_total": 15847,
    "requests_per_minute": 23.4,
    "memory_usage": "512MB",
    "cpu_usage": "15%"
  },
  "agents": {
    "total": 8,
    "active": 6, 
    "sessions": 299,
    "avg_response_time": "1.2s"
  },
  "channels": {
    "telegram": {"status": "OK", "latency": "45ms"},
    "slack": {"status": "OK", "latency": "32ms"},
    "whatsapp": {"status": "WARN", "error": "Not linked"}
  }
}
```

### **8.4 健康检查配置**

#### **自动化健康检查**
```bash
# 运行完整健康检查
openclaw doctor --non-interactive

# 检查特定组件
openclaw doctor --check-channels
openclaw doctor --check-models
openclaw doctor --check-security
```

#### **定时健康检查脚本**
```bash
#!/bin/bash
# openclaw-health-monitor.sh

LOG_FILE="/var/log/openclaw-health.log"
ALERT_WEBHOOK="https://your-alert-webhook.com"

check_component() {
    local component=$1
    local result=$(openclaw doctor --check-$component --json)
    local status=$(echo "$result" | jq -r '.status')
    
    echo "$(date): $component - $status" >> "$LOG_FILE"
    
    if [[ "$status" != "OK" ]]; then
        # 发送告警
        curl -X POST "$ALERT_WEBHOOK" \
             -H "Content-Type: application/json" \
             -d "{\"component\":\"$component\",\"status\":\"$status\",\"details\":$result}"
    fi
}

# 检查各个组件
check_component "gateway"
check_component "channels" 
check_component "models"
check_component "security"
check_component "memory"
```

---

## 📝 **日志管理系统**

### **8.5 OpenClaw日志配置**

#### **日志级别设置**
```json
{
  "logging": {
    "level": "info",
    "format": "structured",
    "outputs": [
      {
        "type": "file",
        "path": "/var/log/openclaw/gateway.log",
        "rotation": {
          "maxSize": "100MB",
          "maxFiles": 10,
          "compress": true
        }
      },
      {
        "type": "syslog", 
        "facility": "local0"
      }
    ],
    "filters": [
      {"component": "auth", "minLevel": "warn"},
      {"component": "memory", "minLevel": "debug"}
    ]
  }
}
```

#### **日志查看命令**
```bash
# 实时日志流
openclaw logs --follow

# 错误日志筛选
openclaw logs --level error --since "1h"

# 特定Agent日志
openclaw logs --agent main --limit 100

# 按渠道筛选
openclaw logs --channel telegram --since "2026-02-13"
```

### **8.6 日志分析和聚合**

#### **ELK Stack集成**
```yaml
# docker-compose-logging.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
      - /var/log/openclaw:/logs:ro
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

volumes:
  es_data:
```

#### **Logstash配置示例**
```ruby
# logstash.conf
input {
  file {
    path => "/logs/gateway.log"
    start_position => "beginning"
    codec => json
  }
}

filter {
  if [level] == "ERROR" {
    mutate {
      add_tag => ["error"]
    }
  }
  
  if [component] == "agent" {
    grok {
      match => { "message" => "Agent %{WORD:agent_id} %{GREEDYDATA:action}" }
    }
  }
  
  date {
    match => [ "timestamp", "ISO8601" ]
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "openclaw-logs-%{+YYYY.MM.dd}"
  }
}
```

### **8.7 结构化日志最佳实践**

#### **日志格式标准化**
```json
{
  "timestamp": "2026-02-13T11:02:45.123Z",
  "level": "INFO",
  "component": "gateway",
  "agent_id": "main", 
  "session_id": "abc123",
  "channel": "telegram",
  "user_id": "7996447774",
  "action": "message_received",
  "message": "User message processed successfully",
  "duration_ms": 1250,
  "metadata": {
    "tool_calls": 3,
    "tokens_used": 1847,
    "model": "claude-sonnet-4"
  }
}
```

#### **关键事件标识**
```bash
# 高价值日志事件
ERROR    - 错误和异常
SECURITY - 安全相关事件  
PERF     - 性能问题
AUTH     - 认证授权
AGENT    - Agent状态变化
CHANNEL  - 渠道连接状态
```

---

## 📈 **性能监控和调试**

### **8.8 性能指标收集**

#### **核心性能指标**
```json
{
  "metrics": {
    "gateway": {
      "requests_per_second": 23.4,
      "avg_response_time": 1200,
      "p95_response_time": 3500,
      "p99_response_time": 8000,
      "error_rate": 0.02,
      "concurrent_sessions": 45
    },
    "agents": {
      "avg_thinking_time": 800,
      "tool_call_success_rate": 0.987,
      "memory_search_latency": 150,
      "token_usage_rate": 15000
    },
    "system": {
      "cpu_usage": 0.25,
      "memory_usage": 0.68,
      "disk_io_rate": 120,
      "network_throughput": "15MB/s"
    }
  }
}
```

#### **Prometheus集成**
```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'openclaw-gateway'
    static_configs:
      - targets: ['localhost:18789']
    metrics_path: '/metrics'
    scrape_interval: 10s
    
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['localhost:9100']
```

#### **自定义指标导出**
```bash
# OpenClaw Prometheus指标端点
curl http://localhost:18789/metrics

# 示例指标输出
openclaw_gateway_requests_total{channel="telegram"} 15847
openclaw_gateway_response_time_seconds{quantile="0.5"} 1.2
openclaw_gateway_response_time_seconds{quantile="0.95"} 3.5
openclaw_agent_sessions_active{agent="main"} 12
openclaw_memory_search_duration_seconds{operation="search"} 0.15
```

### **8.9 性能调试工具**

#### **内置性能分析**
```bash
# Agent性能分析
openclaw agent --profile --message "test query"

# Memory搜索性能测试
openclaw memory benchmark --queries 1000

# Gateway负载测试
openclaw gateway --benchmark --duration 60s
```

#### **分布式追踪**
```json
{
  "tracing": {
    "enabled": true,
    "provider": "jaeger",
    "endpoint": "http://localhost:14268/api/traces",
    "samplingRate": 0.1,
    "tags": {
      "service": "openclaw-gateway",
      "version": "2026.2.9"
    }
  }
}
```

---

## 🚨 **告警和通知系统**

### **8.10 告警规则配置**

#### **Alert Manager配置**
```yaml
# alertmanager.yml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@yourcompany.com'

route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: 'web.hook'

receivers:
  - name: 'web.hook'
    webhook_configs:
      - url: 'http://localhost:8091/api/alerts'
        send_resolved: true

  - name: 'telegram'
    telegram_configs:
      - bot_token: 'YOUR_BOT_TOKEN'
        chat_id: 'YOUR_CHAT_ID'
        message: |
          🚨 OpenClaw Alert 
          Alert: {{ .GroupLabels.alertname }}
          Instance: {{ .GroupLabels.instance }}
          Description: {{ .CommonAnnotations.description }}
```

#### **Prometheus告警规则**
```yaml
# openclaw-alerts.yml
groups:
  - name: openclaw.rules
    rules:
      - alert: HighErrorRate
        expr: rate(openclaw_gateway_errors_total[5m]) > 0.1
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is above 10% for 2 minutes"

      - alert: HighResponseTime  
        expr: openclaw_gateway_response_time_seconds{quantile="0.95"} > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High response time"
          description: "95th percentile response time > 5 seconds"

      - alert: AgentDown
        expr: openclaw_agent_status == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Agent is down"
          description: "Agent {{ $labels.agent_id }} has been down for 1 minute"
```

### **8.11 智能告警策略**

#### **告警分级和升级**
```json
{
  "alerting": {
    "levels": [
      {
        "name": "info",
        "threshold": 0,
        "channels": ["log"],
        "escalation": false
      },
      {
        "name": "warning", 
        "threshold": 1,
        "channels": ["slack", "email"],
        "escalation": {
          "after": "15m",
          "to": "critical"
        }
      },
      {
        "name": "critical",
        "threshold": 3, 
        "channels": ["telegram", "phone", "pager"],
        "escalation": {
          "after": "5m",
          "to": "emergency"
        }
      }
    ]
  }
}
```

#### **告警静默和抑制**
```yaml
# 维护窗口静默
silences:
  - matchers:
      - name: instance
        value: "openclaw-prod"
    starts_at: "2026-02-13T02:00:00Z"
    ends_at: "2026-02-13T04:00:00Z"
    comment: "Planned maintenance"

# 告警抑制规则  
inhibit_rules:
  - source_match:
      alertname: 'GatewayDown'
    target_match:
      alertname: 'AgentUnreachable' 
    equal: ['instance']
```

---

## 📊 **可视化监控面板**

### **8.12 Grafana Dashboard**

#### **OpenClaw Overview Dashboard**
```json
{
  "dashboard": {
    "title": "OpenClaw System Overview",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(openclaw_gateway_requests_total[5m])",
            "legendFormat": "{{channel}}"
          }
        ]
      },
      {
        "title": "Response Time Percentiles",
        "type": "graph", 
        "targets": [
          {
            "expr": "openclaw_gateway_response_time_seconds{quantile=\"0.5\"}",
            "legendFormat": "p50"
          },
          {
            "expr": "openclaw_gateway_response_time_seconds{quantile=\"0.95\"}", 
            "legendFormat": "p95"
          }
        ]
      },
      {
        "title": "Active Sessions",
        "type": "singlestat",
        "targets": [
          {
            "expr": "sum(openclaw_agent_sessions_active)"
          }
        ]
      }
    ]
  }
}
```

#### **Agent Performance Dashboard**
```json
{
  "dashboard": {
    "title": "Agent Performance Metrics",
    "panels": [
      {
        "title": "Agent Response Times",
        "type": "heatmap",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, openclaw_agent_response_time_bucket)",
            "legendFormat": "{{agent_id}}"
          }
        ]
      },
      {
        "title": "Tool Call Success Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(openclaw_tool_calls_success_total[5m]) / rate(openclaw_tool_calls_total[5m])"
          }
        ]
      }
    ]
  }
}
```

### **8.13 自定义监控面板**

#### **业务指标面板**
```python
# custom-dashboard-generator.py
import json
import requests

def create_business_dashboard():
    """创建业务监控面板"""
    dashboard = {
        "title": "OpenClaw Business Metrics",
        "tags": ["openclaw", "business"],
        "panels": [
            {
                "title": "Daily Active Users",
                "type": "graph",
                "targets": [{
                    "expr": "count(count by (user_id) (openclaw_sessions_created[24h]))"
                }]
            },
            {
                "title": "Token Usage by Model", 
                "type": "piechart",
                "targets": [{
                    "expr": "sum by (model) (openclaw_tokens_used_total[24h])"
                }]
            },
            {
                "title": "Channel Usage Distribution",
                "type": "bargraph", 
                "targets": [{
                    "expr": "sum by (channel) (openclaw_messages_total[24h])"
                }]
            }
        ]
    }
    
    # 推送到Grafana
    grafana_api = "http://admin:admin@localhost:3000/api/dashboards/db"
    response = requests.post(grafana_api, json={"dashboard": dashboard})
    
    if response.status_code == 200:
        print("✅ Business dashboard created successfully")
    else:
        print(f"❌ Failed to create dashboard: {response.text}")

if __name__ == "__main__":
    create_business_dashboard()
```

---

## 🔧 **故障诊断和调试**

### **8.14 常见故障诊断流程**

#### **故障分类决策树**
```
故障报告 → 
├── 用户无法访问?
│   ├── 检查Gateway状态 → openclaw status
│   ├── 检查渠道连接 → openclaw status --channels  
│   └── 检查网络连接 → ping/traceroute
│
├── 响应缓慢?
│   ├── 检查系统负载 → top/htop
│   ├── 检查Agent性能 → openclaw logs --level performance
│   └── 检查内存使用 → openclaw memory stats
│
└── 功能异常?
    ├── 检查错误日志 → openclaw logs --level error
    ├── 检查配置问题 → openclaw doctor
    └── 检查模型状态 → openclaw status --models
```

#### **自动化故障诊断脚本**
```bash
#!/bin/bash
# openclaw-troubleshoot.sh

echo "🔍 OpenClaw自动故障诊断"
echo "========================="

# 1. 基础连通性检查
echo "📡 检查Gateway连通性..."
if ! curl -s http://localhost:18789/health > /dev/null; then
    echo "❌ Gateway无响应，检查服务状态"
    systemctl --user status openclaw-gateway
    exit 1
fi
echo "✅ Gateway运行正常"

# 2. 渠道状态检查
echo "📱 检查渠道状态..."
CHANNELS=$(openclaw status --json | jq -r '.channels | to_entries[] | select(.value.status != "OK") | .key')
if [[ -n "$CHANNELS" ]]; then
    echo "⚠️ 发现渠道问题: $CHANNELS"
    openclaw doctor --check-channels
else
    echo "✅ 所有渠道正常"
fi

# 3. 性能指标检查
echo "⚡ 检查性能指标..."
RESPONSE_TIME=$(openclaw status --json | jq -r '.gateway.avg_response_time // "0"' | sed 's/s//')
if (( $(echo "$RESPONSE_TIME > 3.0" | bc -l) )); then
    echo "⚠️ 响应时间过慢: ${RESPONSE_TIME}s"
    echo "建议检查:"
    echo "  - 系统负载: $(uptime)"
    echo "  - 内存使用: $(free -h | grep Mem | awk '{print $3"/"$2}')"
    echo "  - 磁盘IO: $(iostat -x 1 1 | tail -n +4)"
fi

# 4. 错误日志分析
echo "📋 检查最近错误..."
ERROR_COUNT=$(openclaw logs --level error --since "1h" --json | jq '. | length')
if [[ "$ERROR_COUNT" -gt 10 ]]; then
    echo "⚠️ 发现 $ERROR_COUNT 个错误，最近的错误:"
    openclaw logs --level error --limit 5
fi

# 5. 建议修复方案
echo ""
echo "🛠️ 自动修复建议:"
echo "  1. 重启Gateway: openclaw gateway restart"
echo "  2. 清理缓存: openclaw memory cache clear"
echo "  3. 检查配置: openclaw doctor --non-interactive"
echo "  4. 查看详细日志: openclaw logs --follow"
```

### **8.15 性能问题调试**

#### **内存泄漏检测**
```bash
# 内存使用趋势监控
watch -n 5 'ps -p $(pgrep -f openclaw) -o pid,ppid,cmd,%mem,%cpu --sort=-%mem'

# 堆内存分析
node --inspect=0.0.0.0:9229 $(which openclaw) gateway

# 生成内存快照
kill -USR2 $(pgrep -f openclaw)
```

#### **CPU性能分析**
```bash
# CPU使用率监控
perf top -p $(pgrep -f openclaw)

# 函数调用分析
perf record -p $(pgrep -f openclaw) -g -- sleep 60
perf report
```

#### **网络问题诊断**
```bash
# 连接状态检查
ss -tulpn | grep :18789

# 网络延迟测试
for host in api.anthropic.com api.openai.com; do
    echo "$host: $(ping -c 3 $host | tail -1 | awk '{print $4}' | cut -d/ -f2)ms"
done

# 带宽测试
iperf3 -c speedtest.net -p 5201
```

---

## 🤖 **自动化运维**

### **8.16 自愈系统设计**

#### **服务监控和重启**
```bash
#!/bin/bash
# auto-heal.sh - OpenClaw自愈脚本

HEALTH_CHECK_URL="http://localhost:18789/health"
MAX_RETRIES=3
RESTART_COOLDOWN=300  # 5分钟冷却期

check_health() {
    local response=$(curl -s -w "%{http_code}" -o /dev/null "$HEALTH_CHECK_URL")
    [[ "$response" == "200" ]]
}

restart_gateway() {
    echo "$(date): 检测到Gateway异常，准备重启..."
    
    # 优雅关闭
    openclaw gateway stop --graceful --timeout 30s
    
    # 等待进程结束
    sleep 5
    
    # 启动服务
    openclaw gateway start --background
    
    # 等待启动完成
    sleep 10
    
    if check_health; then
        echo "$(date): Gateway重启成功"
        return 0
    else
        echo "$(date): Gateway重启失败"
        return 1
    fi
}

# 主循环
while true; do
    if ! check_health; then
        echo "$(date): Gateway健康检查失败"
        
        # 重试机制
        for i in $(seq 1 $MAX_RETRIES); do
            echo "$(date): 尝试重启 ($i/$MAX_RETRIES)"
            
            if restart_gateway; then
                break
            fi
            
            if [[ $i -eq $MAX_RETRIES ]]; then
                echo "$(date): 重启失败，发送告警"
                curl -X POST "$ALERT_WEBHOOK" \
                     -d '{"level":"critical","message":"OpenClaw Gateway重启失败"}'
            fi
            
            sleep $RESTART_COOLDOWN
        done
    fi
    
    sleep 60  # 每分钟检查一次
done
```

#### **动态扩缩容**
```python
# auto-scaling.py
import psutil
import requests
import time
import subprocess

class AutoScaler:
    def __init__(self):
        self.min_instances = 1
        self.max_instances = 5
        self.current_instances = 1
        self.scale_up_threshold = 0.8    # CPU使用率80%
        self.scale_down_threshold = 0.3  # CPU使用率30%
        
    def get_system_metrics(self):
        """获取系统指标"""
        return {
            'cpu_percent': psutil.cpu_percent(interval=1),
            'memory_percent': psutil.virtual_memory().percent,
            'load_average': psutil.getloadavg()[0]
        }
    
    def get_openclaw_metrics(self):
        """获取OpenClaw指标"""
        try:
            response = requests.get('http://localhost:18789/metrics')
            # 解析Prometheus格式指标
            metrics = self.parse_prometheus_metrics(response.text)
            return metrics
        except:
            return None
    
    def should_scale_up(self, metrics):
        """判断是否需要扩容"""
        cpu_high = metrics['cpu_percent'] > self.scale_up_threshold * 100
        load_high = metrics['load_average'] > psutil.cpu_count() * 0.8
        
        return (cpu_high or load_high) and self.current_instances < self.max_instances
    
    def should_scale_down(self, metrics):
        """判断是否需要缩容"""
        cpu_low = metrics['cpu_percent'] < self.scale_down_threshold * 100
        load_low = metrics['load_average'] < psutil.cpu_count() * 0.3
        
        return (cpu_low and load_low) and self.current_instances > self.min_instances
    
    def scale_up(self):
        """扩容操作"""
        print(f"扩容: {self.current_instances} -> {self.current_instances + 1}")
        
        # 启动新的Gateway实例
        port = 18789 + self.current_instances
        cmd = f"openclaw gateway --port {port} --background"
        subprocess.run(cmd.split())
        
        self.current_instances += 1
        
    def scale_down(self):
        """缩容操作"""
        print(f"缩容: {self.current_instances} -> {self.current_instances - 1}")
        
        # 停止最后一个实例
        port = 18789 + self.current_instances - 1
        cmd = f"pkill -f 'openclaw.*--port {port}'"
        subprocess.run(cmd, shell=True)
        
        self.current_instances -= 1
    
    def run(self):
        """主循环"""
        while True:
            metrics = self.get_system_metrics()
            print(f"CPU: {metrics['cpu_percent']:.1f}%, Memory: {metrics['memory_percent']:.1f}%, Load: {metrics['load_average']:.2f}")
            
            if self.should_scale_up(metrics):
                self.scale_up()
            elif self.should_scale_down(metrics):
                self.scale_down()
                
            time.sleep(60)  # 每分钟检查一次

if __name__ == "__main__":
    scaler = AutoScaler()
    scaler.run()
```

### **8.17 备份和恢复自动化**

#### **自动化备份脚本**
```bash
#!/bin/bash
# backup-automation.sh

BACKUP_DIR="/backup/openclaw/$(date +%Y%m%d)"
RETENTION_DAYS=30
S3_BUCKET="s3://openclaw-backups"

# 创建备份目录
mkdir -p "$BACKUP_DIR"

# 备份配置文件
echo "备份配置文件..."
tar -czf "$BACKUP_DIR/config-$(date +%H%M).tar.gz" \
    ~/.openclaw/openclaw.json \
    ~/.openclaw/auth-profiles.json

# 备份记忆文件  
echo "备份记忆文件..."
tar -czf "$BACKUP_DIR/memory-$(date +%H%M).tar.gz" \
    ~/.openclaw/workspace-main/memory/

# 备份数据库
echo "备份向量数据库..."
cp -r ~/.openclaw/workspace-main/memory/vectors/ "$BACKUP_DIR/"

# 上传到云存储
if command -v aws &> /dev/null; then
    echo "上传到S3..."
    aws s3 sync "$BACKUP_DIR" "$S3_BUCKET/$(date +%Y%m%d)/"
fi

# 清理旧备份
find /backup/openclaw -type d -mtime +$RETENTION_DAYS -exec rm -rf {} +

echo "✅ 备份完成: $BACKUP_DIR"
```

---

## 📋 **本章总结**

### **核心要点**
1. **分层监控**: 应用层、Gateway层、系统层、基础设施层
2. **全面可观测性**: 指标、日志、追踪三大支柱
3. **智能告警**: 分级告警、升级策略、静默机制
4. **自动化运维**: 自愈系统、动态扩缩容、备份恢复

### **监控清单**
- ✅ **基础指标**: CPU、内存、磁盘、网络
- ✅ **应用指标**: 请求率、响应时间、错误率、并发数
- ✅ **业务指标**: 用户活跃度、Token使用量、渠道分布
- ✅ **安全指标**: 认证失败、异常访问、权限变更

### **告警策略**
- ✅ **分级响应**: Info → Warning → Critical → Emergency  
- ✅ **多渠道通知**: 邮件、Slack、Telegram、短信
- ✅ **智能抑制**: 避免告警风暴、维护窗口静默
- ✅ **自动化响应**: 自愈脚本、扩缩容、故障切换

### **下一步学习**
- 第9章: 多服务器部署架构
- 附录A: 命令速查表
- 附录B: 故障排除指南

### **实践建议**
1. 从基础监控开始，逐步完善指标体系
2. 建立标准化的告警响应流程
3. 定期进行故障演练和恢复测试
4. 持续优化监控策略和告警阈值

---

**🔗 相关资源:**
- [OpenClaw监控文档](https://docs.openclaw.ai/monitoring)
- [Prometheus OpenClaw Exporter](https://github.com/openclaw/prometheus-exporter)
- [Grafana Dashboard Templates](https://grafana.com/dashboards/openclaw)