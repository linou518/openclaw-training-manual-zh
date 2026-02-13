# 第9章：多服务器集群部署

> 🎯 学习目标：掌握OpenClaw的多服务器部署架构，实现高可用和水平扩展

## 🏗️ **为什么需要多服务器部署？**

### **单服务器的限制**
- 🔥 **性能瓶颈**: 单机资源有限，无法支撑大量并发
- 💥 **单点故障**: 服务器故障导致整个系统不可用  
- 📈 **扩展困难**: 垂直扩展成本高，水平扩展不可行
- 🌍 **地理限制**: 无法就近服务全球用户
- 🔒 **安全风险**: 所有服务集中在一台机器上

### **多服务器的价值**
- ⚡ **高性能**: 分布式计算，并行处理大量请求
- 🛡️ **高可用**: 服务器故障不影响整体可用性
- 📊 **水平扩展**: 根据负载动态添加服务器
- 🌐 **地理分布**: 全球部署，就近服务用户
- 🔐 **安全隔离**: 不同服务运行在隔离的环境中

---

## 🏛️ **多服务器架构设计**

### **架构模式1: 功能分离型**
```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Gateway       │  │   Agent Farm    │  │   Service       │
│   Server        │  │   Server        │  │   Server        │
├─────────────────┤  ├─────────────────┤  ├─────────────────┤
│ • Load Balancer │  │ • Agent-1       │  │ • Database      │
│ • API Gateway   │  │ • Agent-2       │  │ • File Storage  │
│ • Auth Service  │  │ • Agent-3       │  │ • Message Queue │
│ • Rate Limiting │  │ • Agent-4       │  │ • Cache         │
└─────────────────┘  └─────────────────┘  └─────────────────┘
         │                      │                      │
         └──────────────────────┼──────────────────────┘
                                │
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Monitoring    │  │   Tools &       │  │   Backup &      │ 
│   Server        │  │   Analytics     │  │   Archive       │
├─────────────────┤  ├─────────────────┤  ├─────────────────┤
│ • Metrics       │  │ • Search Engine │  │ • Backup Jobs   │
│ • Logging       │  │ • Analytics     │  │ • Archive       │
│ • Alerting      │  │ • Reporting     │  │ • Disaster      │
│ • Health Check  │  │ • ML Pipeline   │  │   Recovery      │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

### **架构模式2: 地理分布型**
```
                    ┌─────────────────┐
                    │   Global CDN    │
                    │  & Load Balancer│
                    └─────────┬───────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
┌───────▼──────┐     ┌────────▼────────┐     ┌──────▼───────┐
│  US-East     │     │    EU-Central   │     │  Asia-Pacific│
│  Region      │     │    Region       │     │   Region     │
├──────────────┤     ├─────────────────┤     ├──────────────┤
│ Gateway      │     │ Gateway         │     │ Gateway      │
│ Agent Farm   │     │ Agent Farm      │     │ Agent Farm   │
│ Services     │     │ Services        │     │ Services     │
└──────────────┘     └─────────────────┘     └──────────────┘
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              │
                    ┌─────────▼───────┐
                    │  Global State   │
                    │  Sync Service   │
                    └─────────────────┘
```

### **架构模式3: 微服务化**
```
┌─────────────────────────────────────────────────────────┐
│                    Service Mesh                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │
│  │Gateway  │  │ Agent   │  │ Session │  │  Tool   │    │
│  │Service  │  │Manager  │  │Manager  │  │Service  │    │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘    │
│                                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │
│  │Channel  │  │Memory   │  │ Config  │  │Monitor  │    │
│  │Service  │  │Service  │  │Service  │  │Service  │    │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘    │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                Infrastructure                           │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │
│  │Database │  │Message  │  │Storage  │  │  Cache  │    │
│  │Cluster  │  │ Queue   │  │Cluster  │  │Cluster  │    │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘    │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ **实战：3服务器集群部署**

让我们设计一个实际的3服务器OpenClaw集群：

### **服务器规划**

| 服务器 | 角色 | 规格 | 组件 |
|--------|------|------|------|
| **Server-1** | Gateway + LB | 4核8GB | 网关、负载均衡、认证 |
| **Server-2** | Agent Farm | 8核16GB | 主要Agent实例 |
| **Server-3** | Services | 4核8GB | 数据库、缓存、存储 |

### **网络架构**
```
Internet
    │
    ▼
┌─────────────┐
│ Load        │
│ Balancer    │  ← Nginx/HAProxy
│ (Server-1)  │
└──────┬──────┘
       │
   ┌───┴───┐
   ▼       ▼
┌─────────────┐    ┌─────────────┐
│ OpenClaw    │    │ OpenClaw    │
│ Gateway     │    │ Agents      │
│ (Server-1)  │◄──►│ (Server-2)  │
└─────────────┘    └──────┬──────┘
       │                  │
       └──────────────────┼────────┐
                          │        │
                    ┌─────▼────────▼─┐
                    │   Services     │
                    │  • PostgreSQL  │
                    │  • Redis       │
                    │  • File Store  │
                    │  (Server-3)    │
                    └────────────────┘
```

---

## 🚀 **部署实施指南**

### **步骤1: 服务器准备**

#### **基础环境配置 (所有服务器)**
```bash
#!/bin/bash
# setup-base.sh - 基础环境配置脚本

# 更新系统
sudo apt update && sudo apt upgrade -y

# 安装基础软件
sudo apt install -y \
    curl wget git vim htop \
    docker.io docker-compose \
    nginx certbot \
    postgresql-client redis-tools

# 配置Docker
sudo usermod -aG docker $USER
sudo systemctl enable docker
sudo systemctl start docker

# 安装Node.js
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# 安装OpenClaw
sudo npm install -g @openclaw/openclaw

# 创建OpenClaw用户
sudo useradd -m -s /bin/bash openclaw
sudo mkdir -p /home/openclaw/.openclaw
sudo chown -R openclaw:openclaw /home/openclaw/.openclaw

# 配置防火墙
sudo ufw allow ssh
sudo ufw allow 80
sudo ufw allow 443
sudo ufw allow 18789  # OpenClaw Gateway
sudo ufw --force enable

echo "基础环境配置完成"
```

### **步骤2: Server-1 (Gateway) 配置**

#### **负载均衡配置**
```nginx
# /etc/nginx/sites-available/openclaw
upstream openclaw_gateway {
    least_conn;
    server 127.0.0.1:18789 max_fails=3 fail_timeout=30s;
    server SERVER-2-IP:18789 max_fails=3 fail_timeout=30s backup;
}

server {
    listen 80;
    server_name your-domain.com;
    
    # 重定向到HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;
    
    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;
    
    # SSL配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    
    # 安全头
    add_header Strict-Transport-Security "max-age=63072000" always;
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    
    # 限流
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req zone=api burst=20 nodelay;
    
    location / {
        proxy_pass http://openclaw_gateway;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        
        # 超时配置
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 300s;
        
        # 缓冲配置
        proxy_buffering on;
        proxy_buffer_size 128k;
        proxy_buffers 4 256k;
        proxy_busy_buffers_size 256k;
    }
    
    # 健康检查端点
    location /health {
        access_log off;
        proxy_pass http://openclaw_gateway/health;
    }
    
    # 静态文件
    location /static/ {
        alias /var/www/openclaw/static/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

#### **Gateway配置文件**
```json
{
  "gateway": {
    "port": 18789,
    "bind": "all",
    "cors": {
      "enabled": true,
      "origins": ["https://your-domain.com"]
    },
    "rateLimit": {
      "enabled": true,
      "windowMs": 60000,
      "max": 100
    },
    "cluster": {
      "enabled": true,
      "mode": "gateway",
      "peers": [
        "http://SERVER-2-IP:18789",
        "http://SERVER-3-IP:18789"
      ]
    }
  },
  "agents": [
    {
      "id": "coordinator",
      "name": "主协调员",
      "model": "anthropic/claude-sonnet-4-20250514",
      "workspace": {"root": "./agents/coordinator"},
      "cluster": {
        "enabled": true,
        "sticky": false,
        "replicas": 2
      }
    }
  ],
  "database": {
    "type": "postgresql",
    "host": "SERVER-3-IP",
    "port": 5432,
    "database": "openclaw",
    "username": "openclaw",
    "password": "SECURE_PASSWORD",
    "pool": {
      "min": 2,
      "max": 10
    }
  },
  "cache": {
    "type": "redis",
    "host": "SERVER-3-IP", 
    "port": 6379,
    "password": "REDIS_PASSWORD"
  },
  "auth": {
    "profiles": [
      {"id": "anthropic", "provider": "anthropic"}
    ]
  },
  "monitoring": {
    "enabled": true,
    "metrics": {
      "enabled": true,
      "port": 9090
    },
    "logging": {
      "level": "info",
      "destination": "file",
      "path": "/var/log/openclaw/"
    }
  }
}
```

### **步骤3: Server-2 (Agent Farm) 配置**

#### **Agent专用配置**
```json
{
  "gateway": {
    "port": 18789,
    "bind": "all",
    "cluster": {
      "enabled": true,
      "mode": "agent",
      "primary": "SERVER-1-IP:18789"
    }
  },
  "agents": [
    {
      "id": "email-manager",
      "name": "邮件管理员",
      "model": "anthropic/claude-sonnet-4-20250514",
      "workspace": {"root": "./agents/email-manager"},
      "resources": {
        "memory": "2GB",
        "cpu": "2"
      }
    },
    {
      "id": "calendar-manager", 
      "name": "日程管理员",
      "model": "anthropic/claude-sonnet-4-20250514",
      "workspace": {"root": "./agents/calendar-manager"},
      "resources": {
        "memory": "1.5GB",
        "cpu": "1.5"
      }
    },
    {
      "id": "doc-processor",
      "name": "文档处理器",
      "model": "anthropic/claude-sonnet-4-20250514", 
      "workspace": {"root": "./agents/doc-processor"},
      "resources": {
        "memory": "3GB",
        "cpu": "2.5"
      }
    },
    {
      "id": "data-analyst",
      "name": "数据分析师",
      "model": "anthropic/claude-sonnet-4-20250514",
      "workspace": {"root": "./agents/data-analyst"}, 
      "resources": {
        "memory": "4GB",
        "cpu": "3"
      }
    }
  ],
  "database": {
    "type": "postgresql",
    "host": "SERVER-3-IP",
    "port": 5432,
    "database": "openclaw",
    "username": "openclaw",
    "password": "SECURE_PASSWORD"
  },
  "cache": {
    "type": "redis",
    "host": "SERVER-3-IP",
    "port": 6379
  }
}
```

#### **资源监控脚本**
```python
# resource-monitor.py - Agent服务器资源监控
import psutil
import json
import time
import logging
from datetime import datetime

class ResourceMonitor:
    def __init__(self, threshold_cpu=80, threshold_memory=85):
        self.threshold_cpu = threshold_cpu
        self.threshold_memory = threshold_memory
        
    def get_system_stats(self):
        """获取系统资源统计"""
        return {
            "timestamp": datetime.now().isoformat(),
            "cpu": {
                "percent": psutil.cpu_percent(interval=1),
                "count": psutil.cpu_count(),
                "load_avg": psutil.getloadavg()
            },
            "memory": {
                "total": psutil.virtual_memory().total,
                "available": psutil.virtual_memory().available,
                "percent": psutil.virtual_memory().percent,
                "used": psutil.virtual_memory().used
            },
            "disk": {
                "total": psutil.disk_usage('/').total,
                "used": psutil.disk_usage('/').used,
                "free": psutil.disk_usage('/').free,
                "percent": psutil.disk_usage('/').percent
            },
            "network": {
                "bytes_sent": psutil.net_io_counters().bytes_sent,
                "bytes_recv": psutil.net_io_counters().bytes_recv
            }
        }
        
    def check_agent_processes(self):
        """检查OpenClaw Agent进程"""
        agents = []
        for proc in psutil.process_iter(['pid', 'name', 'cmdline', 'cpu_percent', 'memory_info']):
            try:
                if 'openclaw' in ' '.join(proc.info['cmdline'] or []):
                    agents.append({
                        "pid": proc.info['pid'],
                        "name": proc.info['name'],
                        "cpu_percent": proc.info['cpu_percent'],
                        "memory_mb": proc.info['memory_info'].rss / 1024 / 1024,
                        "cmdline": ' '.join(proc.info['cmdline'] or [])
                    })
            except (psutil.NoSuchProcess, psutil.AccessDenied):
                pass
                
        return agents
        
    def check_alerts(self, stats):
        """检查是否需要告警"""
        alerts = []
        
        if stats["cpu"]["percent"] > self.threshold_cpu:
            alerts.append({
                "type": "cpu_high",
                "message": f"CPU使用率过高: {stats['cpu']['percent']:.1f}%",
                "severity": "warning"
            })
            
        if stats["memory"]["percent"] > self.threshold_memory:
            alerts.append({
                "type": "memory_high", 
                "message": f"内存使用率过高: {stats['memory']['percent']:.1f}%",
                "severity": "warning"
            })
            
        if stats["disk"]["percent"] > 90:
            alerts.append({
                "type": "disk_full",
                "message": f"磁盘空间不足: {stats['disk']['percent']:.1f}%",
                "severity": "critical"
            })
            
        return alerts
        
    def run_monitoring_loop(self, interval=60):
        """运行监控循环"""
        while True:
            try:
                stats = self.get_system_stats()
                agents = self.check_agent_processes()
                alerts = self.check_alerts(stats)
                
                report = {
                    "system": stats,
                    "agents": agents,
                    "alerts": alerts
                }
                
                # 记录到日志
                logging.info(f"Resource Report: {json.dumps(report)}")
                
                # 如果有告警，发送通知
                if alerts:
                    self.send_alerts(alerts)
                    
                time.sleep(interval)
                
            except Exception as e:
                logging.error(f"Monitoring error: {e}")
                time.sleep(10)
                
    def send_alerts(self, alerts):
        """发送告警通知"""
        # 这里可以集成各种通知方式：邮件、Slack、钉钉等
        for alert in alerts:
            print(f"🚨 ALERT: {alert['message']}")
            
if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    monitor = ResourceMonitor()
    monitor.run_monitoring_loop()
```

### **步骤4: Server-3 (Services) 配置**

#### **数据库设置**
```sql
-- setup-database.sql
-- 创建OpenClaw数据库和用户

-- 创建数据库
CREATE DATABASE openclaw;

-- 创建用户
CREATE USER openclaw WITH PASSWORD 'SECURE_PASSWORD';

-- 授权
GRANT ALL PRIVILEGES ON DATABASE openclaw TO openclaw;

-- 连接到openclaw数据库
\c openclaw

-- 创建扩展
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements";

-- 创建表结构
CREATE TABLE IF NOT EXISTS agents (
    id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    config JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS sessions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    agent_id VARCHAR(50) NOT NULL,
    user_id VARCHAR(100),
    channel VARCHAR(50),
    context JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (agent_id) REFERENCES agents(id)
);

CREATE TABLE IF NOT EXISTS messages (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    session_id UUID NOT NULL,
    role VARCHAR(20) NOT NULL,
    content TEXT NOT NULL,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (session_id) REFERENCES sessions(id)
);

-- 创建索引
CREATE INDEX idx_sessions_agent_id ON sessions(agent_id);
CREATE INDEX idx_sessions_user_id ON sessions(user_id);  
CREATE INDEX idx_messages_session_id ON messages(session_id);
CREATE INDEX idx_messages_created_at ON messages(created_at);

-- 创建视图
CREATE VIEW agent_stats AS
SELECT 
    a.id,
    a.name,
    COUNT(DISTINCT s.id) as session_count,
    COUNT(m.id) as message_count,
    MAX(s.updated_at) as last_activity
FROM agents a
LEFT JOIN sessions s ON a.id = s.agent_id
LEFT JOIN messages m ON s.id = m.session_id
GROUP BY a.id, a.name;

GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO openclaw;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO openclaw;
```

#### **Redis配置**
```redis
# /etc/redis/redis.conf

# 基础配置
port 6379
bind 0.0.0.0
protected-mode yes
requirepass REDIS_PASSWORD

# 持久化
save 900 1
save 300 10
save 60 10000
rdbcompression yes
dbfilename dump.rdb
dir /var/lib/redis

# 日志
loglevel notice
logfile /var/log/redis/redis-server.log

# 内存管理
maxmemory 2gb
maxmemory-policy allkeys-lru

# 网络
tcp-keepalive 300
timeout 0

# 性能
tcp-backlog 511
databases 16

# 安全
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command SHUTDOWN SHUTDOWN_REDIS
```

#### **文件存储配置**
```yaml
# docker-compose.yml - 文件存储服务
version: '3.8'

services:
  minio:
    image: minio/minio:latest
    container_name: openclaw-storage
    environment:
      MINIO_ROOT_USER: openclaw
      MINIO_ROOT_PASSWORD: SECURE_STORAGE_PASSWORD
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - /data/minio:/data
    command: server /data --console-address ":9001"
    restart: unless-stopped

  nginx-storage:
    image: nginx:alpine
    container_name: openclaw-storage-proxy
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - /data/files:/usr/share/nginx/html/files:ro
    depends_on:
      - minio
    restart: unless-stopped
```

---

## 🔄 **服务编排和自动化**

### **Docker Compose全栈部署**
```yaml
# docker-compose.production.yml
version: '3.8'

networks:
  openclaw-net:
    driver: bridge

services:
  # 负载均衡器
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx:/etc/nginx
      - ./ssl:/etc/ssl
    depends_on:
      - gateway-1
      - gateway-2
    networks:
      - openclaw-net
    restart: unless-stopped

  # Gateway实例
  gateway-1:
    image: openclaw/gateway:latest
    environment:
      - NODE_ENV=production
      - OPENCLAW_CONFIG=/app/config/gateway.json
      - DATABASE_URL=postgresql://openclaw:password@postgres:5432/openclaw
      - REDIS_URL=redis://redis:6379
    volumes:
      - ./config/gateway-1.json:/app/config/gateway.json
      - ./data/gateway-1:/app/data
    networks:
      - openclaw-net
    restart: unless-stopped

  gateway-2:
    image: openclaw/gateway:latest
    environment:
      - NODE_ENV=production  
      - OPENCLAW_CONFIG=/app/config/gateway.json
      - DATABASE_URL=postgresql://openclaw:password@postgres:5432/openclaw
      - REDIS_URL=redis://redis:6379
    volumes:
      - ./config/gateway-2.json:/app/config/gateway.json
      - ./data/gateway-2:/app/data
    networks:
      - openclaw-net
    restart: unless-stopped

  # Agent实例
  agent-farm:
    image: openclaw/agents:latest
    deploy:
      replicas: 3
    environment:
      - NODE_ENV=production
      - OPENCLAW_CONFIG=/app/config/agents.json
      - DATABASE_URL=postgresql://openclaw:password@postgres:5432/openclaw
      - REDIS_URL=redis://redis:6379
    volumes:
      - ./config/agents.json:/app/config/agents.json
      - ./data/agents:/app/data
    networks:
      - openclaw-net
    restart: unless-stopped

  # 数据库
  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=openclaw
      - POSTGRES_USER=openclaw
      - POSTGRES_PASSWORD=SECURE_PASSWORD
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - openclaw-net
    restart: unless-stopped

  # 缓存
  redis:
    image: redis:7-alpine
    command: redis-server --requirepass REDIS_PASSWORD
    volumes:
      - redis_data:/data
    networks:
      - openclaw-net
    restart: unless-stopped

  # 监控
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    networks:
      - openclaw-net
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=GRAFANA_PASSWORD
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana:/etc/grafana/provisioning
    networks:
      - openclaw-net
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  prometheus_data:
  grafana_data:
```

### **自动化部署脚本**
```bash
#!/bin/bash
# deploy.sh - 自动化部署脚本

set -e

# 配置变量
SERVERS=("SERVER-1-IP" "SERVER-2-IP" "SERVER-3-IP")
SSH_USER="openclaw"
SSH_KEY="~/.ssh/openclaw-deploy"

# 颜色输出
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

log_info() { echo -e "${GREEN}[INFO]${NC} $1"; }
log_warn() { echo -e "${YELLOW}[WARN]${NC} $1"; }
log_error() { echo -e "${RED}[ERROR]${NC} $1"; }

# 检查前置条件
check_prerequisites() {
    log_info "检查部署前置条件..."
    
    # 检查SSH连接
    for server in "${SERVERS[@]}"; do
        if ! ssh -i $SSH_KEY -o ConnectTimeout=5 $SSH_USER@$server "echo 'SSH OK'" &>/dev/null; then
            log_error "无法SSH连接到 $server"
            exit 1
        fi
    done
    
    # 检查Docker
    for server in "${SERVERS[@]}"; do
        if ! ssh -i $SSH_KEY $SSH_USER@$server "docker --version" &>/dev/null; then
            log_error "$server 上未安装Docker"
            exit 1
        fi
    done
    
    log_info "前置条件检查通过"
}

# 上传配置文件
upload_configs() {
    log_info "上传配置文件..."
    
    # 上传到Gateway服务器
    scp -i $SSH_KEY -r config/gateway/ $SSH_USER@${SERVERS[0]}:~/openclaw/
    scp -i $SSH_KEY -r config/nginx/ $SSH_USER@${SERVERS[0]}:~/openclaw/
    
    # 上传到Agent服务器  
    scp -i $SSH_KEY -r config/agents/ $SSH_USER@${SERVERS[1]}:~/openclaw/
    
    # 上传到Services服务器
    scp -i $SSH_KEY -r config/database/ $SSH_USER@${SERVERS[2]}:~/openclaw/
    scp -i $SSH_KEY docker-compose.services.yml $SSH_USER@${SERVERS[2]}:~/openclaw/
    
    log_info "配置文件上传完成"
}

# 部署Services服务器
deploy_services() {
    log_info "部署Services服务器..."
    
    ssh -i $SSH_KEY $SSH_USER@${SERVERS[2]} << 'EOF'
        cd ~/openclaw
        docker-compose -f docker-compose.services.yml down
        docker-compose -f docker-compose.services.yml pull
        docker-compose -f docker-compose.services.yml up -d
        
        # 等待数据库启动
        sleep 30
        
        # 初始化数据库
        docker-compose -f docker-compose.services.yml exec -T postgres psql -U openclaw -d openclaw -f /docker-entrypoint-initdb.d/init.sql
EOF
    
    log_info "Services服务器部署完成"
}

# 部署Agent服务器
deploy_agents() {
    log_info "部署Agent服务器..."
    
    ssh -i $SSH_KEY $SSH_USER@${SERVERS[1]} << 'EOF'
        cd ~/openclaw
        
        # 停止现有服务
        openclaw gateway stop 2>/dev/null || true
        
        # 更新OpenClaw
        sudo npm update -g @openclaw/openclaw
        
        # 启动Agent服务
        openclaw gateway start --config agents/config.json --daemon
        
        # 验证启动
        sleep 10
        openclaw status
EOF
    
    log_info "Agent服务器部署完成"  
}

# 部署Gateway服务器
deploy_gateway() {
    log_info "部署Gateway服务器..."
    
    ssh -i $SSH_KEY $SSH_USER@${SERVERS[0]} << 'EOF'
        cd ~/openclaw
        
        # 更新Nginx配置
        sudo cp nginx/openclaw.conf /etc/nginx/sites-available/
        sudo ln -sf /etc/nginx/sites-available/openclaw.conf /etc/nginx/sites-enabled/
        sudo nginx -t
        sudo systemctl reload nginx
        
        # 停止现有服务
        openclaw gateway stop 2>/dev/null || true
        
        # 更新OpenClaw
        sudo npm update -g @openclaw/openclaw
        
        # 启动Gateway服务
        openclaw gateway start --config gateway/config.json --daemon
        
        # 验证启动
        sleep 10
        openclaw status
        curl -f http://localhost:18789/health || exit 1
EOF
    
    log_info "Gateway服务器部署完成"
}

# 健康检查
health_check() {
    log_info "执行健康检查..."
    
    # 检查各服务状态
    for server in "${SERVERS[@]}"; do
        log_info "检查 $server 状态..."
        ssh -i $SSH_KEY $SSH_USER@$server "openclaw status" || log_warn "$server 状态异常"
    done
    
    # 检查负载均衡
    if curl -f https://your-domain.com/health &>/dev/null; then
        log_info "负载均衡健康检查通过"
    else
        log_warn "负载均衡健康检查失败"
    fi
    
    log_info "健康检查完成"
}

# 主部署流程
main() {
    log_info "开始OpenClaw集群部署..."
    
    check_prerequisites
    upload_configs
    deploy_services
    sleep 60  # 等待服务启动
    deploy_agents
    sleep 30
    deploy_gateway
    sleep 30
    health_check
    
    log_info "🎉 OpenClaw集群部署完成！"
    echo
    echo "访问地址: https://your-domain.com"
    echo "监控面板: https://your-domain.com:3000"
    echo "API文档: https://your-domain.com/docs"
}

# 运行部署
main "$@"
```

---

## 📊 **监控和运维**

### **Prometheus监控配置**
```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "rules/*.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  - job_name: 'openclaw-gateway'
    static_configs:
      - targets: ['SERVER-1-IP:9090', 'SERVER-2-IP:9090']
    metrics_path: /metrics
    scrape_interval: 30s

  - job_name: 'openclaw-agents'
    static_configs:
      - targets: ['SERVER-2-IP:9091']
    metrics_path: /agents/metrics
    scrape_interval: 30s

  - job_name: 'system'
    static_configs:
      - targets: ['SERVER-1-IP:9100', 'SERVER-2-IP:9100', 'SERVER-3-IP:9100']

  - job_name: 'postgres'
    static_configs:
      - targets: ['SERVER-3-IP:9187']

  - job_name: 'redis'
    static_configs:
      - targets: ['SERVER-3-IP:9121']
```

### **告警规则**
```yaml
# rules/openclaw.yml
groups:
  - name: openclaw
    rules:
      - alert: GatewayDown
        expr: up{job="openclaw-gateway"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "OpenClaw Gateway is down"
          description: "Gateway on {{ $labels.instance }} has been down for more than 1 minute"

      - alert: HighResponseTime
        expr: openclaw_request_duration_seconds{quantile="0.95"} > 5
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High response time detected"
          description: "95th percentile response time is {{ $value }}s on {{ $labels.instance }}"

      - alert: HighErrorRate
        expr: rate(openclaw_requests_total{status=~"5.."}[5m]) > 0.1
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} on {{ $labels.instance }}"

      - alert: AgentMemoryHigh
        expr: openclaw_agent_memory_usage_bytes / 1024 / 1024 / 1024 > 4
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Agent memory usage high"
          description: "Agent {{ $labels.agent_id }} is using {{ $value }}GB memory"
```

---

## ✅ **本章总结**

通过本章学习，你已经掌握：

- [x] 多服务器架构设计的核心概念和模式
- [x] 3服务器集群的完整部署方案
- [x] 负载均衡和高可用配置
- [x] 数据库集群和缓存配置
- [x] 自动化部署和配置管理
- [x] 监控告警体系搭建
- [x] 运维和故障处理流程

---

## 🚀 **下一步**

掌握了多服务器部署后，你可以继续学习：

**[下一章：容器化和编排 →](../10-containerization/README.md)**

深入了解Kubernetes集群部署和容器编排技术。

---

## 📝 **扩展练习**

1. **高可用实践**: 模拟服务器故障，测试自动切换
2. **性能调优**: 使用压测工具测试集群性能极限  
3. **灾备演练**: 实施完整的备份和恢复流程
4. **多地域部署**: 扩展到跨地域的全球化部署
5. **监控优化**: 建立完整的SRE监控体系

**准备好挑战更大规模的企业级部署了吗？** 🎯