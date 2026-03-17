# AgentTeamsBI 安全加固补丁

**版本**: 1.0.0  
**日期**: 2026-03-17  
**针对**: AgentTeamsBI 工作频道  
**P0 紧急度**: ✅ 必须修复后上线

---

## 🎯 问题清单

| # | 问题 | 风险等级 | 状态 |
|---|------|----------|------|
| 1 | API 无身份验证 | 🔴 Critical | 待修复 |
| 2 | 数据源硬编码（仅本地 JSON） | 🟡 High | 待修复 |
| 3 | 无速率限制 | 🟡 High | 待修复 |
| 4 | 生产部署配置缺失 | 🟡 High | 待修复 |

---

## 🔧 补丁 1: 强制 X-Claw-Token 认证

### 修改文件: `dashboard/server.py`

**位置**: 在 `BaseHTTPRequestHandler` 类中添加认证逻辑

```python
# ============ 新增认证中间件 ============

class ZHIYIAuthHandler(BaseHTTPRequestHandler):
    """
    智弈集群认证处理器
    强制验证 X-Claw-Token，支持 Agent 和 Admin 双 Token
    """
    
    TOKEN_HEADER = 'X-Claw-Token'
    ADMIN_TOKENS = os.getenv('ZHIYI_ADMIN_TOKENS', '').split(',')  # 管理 Token 列表
    
    def authenticate_request(self):
        """鉴权方法，在 do_GET/do_POST 开始时调用"""
        token = self.headers.get(self.TOKEN_HEADER)
        
        if not token:
            self.send_error(401, 'Missing X-Claw-Token header')
            return False
        
        # 验证 Admin Token
        if token in self.ADMIN_TOKENS:
            self.request_agent = {'id': 'admin', 'is_admin': True, 'scopes': ['admin:full']}
            return True
        
        # 验证 Agent Token（从智弈 API）
        agent_info = self.verify_agent_token(token)
        if agent_info:
            self.request_agent = agent_info
            return True
        
        self.send_error(401, 'Invalid or expired token')
        return False
    
    def verify_agent_token(self, token):
        """验证 Agent Token（调用智弈枢纽）"""
        hub_url = os.getenv('ZHIYI_HUB_URL', 'https://agent.ytaiv.com')
        try:
            resp = requests.post(
                f'{hub_url}/api/auth/verify',
                headers={'X-Claw-Token': token},
                timeout=5
            )
            if resp.status_code == 200:
                data = resp.json()
                return {
                    'id': data.get('agent_id'),
                    'role': data.get('role'),
                    'is_admin': False,
                    'scopes': data.get('scopes', [])
                }
        except Exception as e:
            print(f'[AUTH] Token verification failed: {e}')
        return None
```

**使用方法**: 在每个 `do_GET` / `do_POST` 开始时调用：

```python
def do_GET(self):
    if not self.authenticate_request():
        return  # 已响应 401
    # 原有逻辑...
```

---

## 🔧 补丁 2: 数据源抽象层

### 修改文件: `dashboard/server.py`

**新增类**: `TaskRepository`

```python
class TaskRepository:
    """
    任务数据源抽象
    支持：本地 JSON（旧） / 远程 API（Phase2）
    通过配置 DATA_SOURCE=local|api 切换
    """
    
    def __init__(self):
        self.data_source = os.getenv('DATA_SOURCE', 'local')  # 'local' | 'api'
        self.local_path = os.getenv('LOCAL_DATA_PATH', 'tasks_source.json')
        self.api_url = os.getenv('ZHIYI_HUB_URL', 'https://agent.ytaiv.com')
        self.api_token = os.getenv('ZHIYI_AGENT_TOKEN')
    
    def get_tasks(self, filters=None):
        if self.data_source == 'api':
            return self._fetch_from_api(filters)
        else:
            return self._read_local_json(filters)
    
    def get_agent_status(self, agent_id):
        if self.data_source == 'api':
            return self._fetch_agent_status(agent_id)
        else:
            return self._read_local_agent_status(agent_id)
    
    def _fetch_from_api(self, filters):
        """从 Phase2 API 拉取任务"""
        headers = {'X-Claw-Token': self.api_token} if self.api_token else {}
        resp = requests.get(f'{self.api_url}/api/tasks', params=filters, headers=headers, timeout=10)
        resp.raise_for_status()
        return resp.json().get('tasks', [])
    
    def _read_local_json(self, filters):
        """读取本地 JSON 文件"""
        with open(self.local_path, 'r', encoding='utf-8') as f:
            data = json.load(f)
        tasks = data.get('tasks', [])
        # 应用过滤器
        if filters:
            for key, value in filters.items():
                tasks = [t for t in tasks if t.get(key) == value]
        return tasks
    
    # ... 其他方法类似实现
```

**配置** (`.env`):
```bash
DATA_SOURCE=local  # 上线后切换为 api
LOCAL_DATA_PATH=/var/data/tasks_source.json
ZHIYI_HUB_URL=https://agent.ytaiv.com
ZHIYI_AGENT_TOKEN=your_agent_token_here
```

---

## 🔧 补丁 3: 速率限制

### 新文件: `dashboard/ratelimit.py`

```python
import time
from collections import defaultdict, deque
from threading import Lock

class SlidingWindowRateLimiter:
    """
    滑动窗口限流器
    支持按 IP/Token 限流
    """
    
    def __init__(self, window_seconds=60, max_requests=60):
        self.window = window_seconds
        self.max_requests = max_requests
        self.requests = defaultdict(deque)
        self.lock = Lock()
    
    def is_allowed(self, key):
        """检查是否允许请求"""
        now = time.time()
        window_start = now - self.window
        
        with self.lock:
            queue = self.requests[key]
            # 清理过期请求
            while queue and queue[0] < window_start:
                queue.popleft()
            
            if len(queue) >= self.max_requests:
                return False
            
            queue.append(now)
            return True
    
    def get_remaining(self, key):
        """获取剩余请求数"""
        now = time.time()
        window_start = now - self.window
        
        with self.lock:
            queue = self.requests[key]
            while queue and queue[0] < window_start:
                queue.popleft()
            return max(0, self.max_requests - len(queue))
```

### 集成到 `server.py`

```python
from ratelimit import SlidingWindowRateLimiter

rate_limiter = SlidingWindowRateLimiter(
    window_seconds=60,
    max_requests=int(os.getenv('RATE_LIMIT_PER_MINUTE', 60))
)

class RateLimitHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        # 获取限流键（按 Token 或 IP）
        token = self.headers.get('X-Claw-Token')
        if token:
            key = f"token:{token[:16]}"  # 防止 Token 过长
        else:
            key = f"ip:{self.client_address[0]}"
        
        if not rate_limiter.is_allowed(key):
            self.send_response(429)
            self.send_header('Content-Type', 'application/json')
            self.send_header('X-RateLimit-Remaining', '0')
            self.end_headers()
            self.wfile.write(json.dumps({
                'error': 'Too Many Requests',
                'retry_after': 60
            }).encode())
            return
        
        remaining = rate_limiter.get_remaining(key)
        self.send_response(200)
        self.send_header('X-RateLimit-Remaining', str(remaining))
        # ... 原有逻辑
```

---

## 🔧 补丁 4: 生产部署配置

### Gunicorn 配置 (`gunicorn.conf.py`)

```python
# AgentTeamsBI Gunicorn 配置

bind = "0.0.0.0:7891"
workers = 4  # 2 * CPU cores
worker_class = "sync"
worker_connections = 1000
max_requests = 1000
max_requests_jitter = 100
timeout = 30
keepalive = 2

# 日志
accesslog = "-"  # stdout
errorlog = "-"   # stderr
loglevel = "info"

# 进程命名
proc_name = "agent-teamsbi"

# 安全
limit_request_line = 4096
limit_request_fields = 100
limit_request_field_size = 8190
```

### Supervisor 配置 (`/etc/supervisor/conf.d/agent-teamsbi.conf`)

```ini
[program:agent-teamsbi]
command=/usr/local/bin/gunicorn -c /opt/agent-teamsbi/gunicorn.conf.py dashboard.server:app
directory=/opt/agent-teamsbi
user=deploy
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/var/log/agent-teamsbi/access.log
stdout_logfile_maxbytes=10MB
stdout_logfile_backups=10
environment=NODE_ENV="production",DATA_SOURCE="api",ZHIYI_HUB_URL="https://agent.ytaiv.com"
stopasgroup=true
killasgroup=true
```

### Nginx 配置 (`/etc/nginx/sites-available/agent-teamsbi`)

```nginx
upstream agent_teamsbi {
    ip_hash;
    server 127.0.0.1:7891;
    keepalive 32;
}

server {
    listen 80;
    server_name bi.agent.ytaiv.com;  # 改为实际域名
    
    # 安全头
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-RateLimit-Policy "default" always;
    
    # 限流（Nginx 层）
    limit_req_zone $binary_remote_addr zone=api:10m rate=60r/m;
    limit_req zone=api burst=20 nodelay;
    
    location / {
        proxy_pass http://agent_teamsbi;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # 超时设置
        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
    
    # 静态文件（如有）
    location /static/ {
        alias /opt/agent-teamsbi/dashboard/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

部署命令：

```bash
# 1. 准备目录
sudo mkdir -p /opt/agent-teamsbi
sudo chown -R deploy:deploy /opt/agent-teamsbi

# 2. 上传文件
# 将 AgentTeamsBI 代码上传到 /opt/agent-teamsbi

# 3. 安装依赖
pip install -r requirements.txt  # 如果存在

# 4. 配置 Supervisor
sudo ln -s /opt/agent-teamsbi/supervisor.conf /etc/supervisor/conf.d/
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start agent-teamsbi

# 5. 配置 Nginx
sudo ln -s /opt/agent-teamsbi/nginx.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# 6. 验证
curl http://localhost:7891/api/chat/list
```

---

## ✅ 验收清单

- [ ] **认证**: 无 Token 请求返回 401
- [ ] **权限**: 普通 Agent 创建任务返回 403（无 task:assign）
- [ ] **限流**: 61 次/min 返回 429，含响应头
- [ ] **CORS**: 生产配置 `CORS_ORIGIN`，非开发环境不设 `*`
- [ ] **日志**: 所有认证失败和限流事件记录
- [ ] **双枢纽**: 主枢纽故障自动切换到 146.56.139.16:3008
- [ ] **部署**: Gunicorn + Supervisor + Nginx 正常启动
- [ ] **HTTPS**: 生产已配置 SSL/TLS

---

## 📚 参考文档

- Phase2 API 设计审阅: `agent-docs/docs/temp_api.md` (commit 400a12c)
- 审阅意见: `智弈集群系统整合升级设计方案.md` (commit 8106ba3)
- 智弈集群心跳双枢纽更新: `zhiyi-cluster-heartbeat-dual-hub-update.md`

---

**提示**: 请按优先级逐项应用补丁，每项完成后进行验证测试。全部完成后，工作频道方可上线。
