# 实验1.5：使用Docker Compose编排多服务应用

## 目标
学习如何使用Docker Compose以声明式方式定义、配置和管理多服务应用程序，简化多容器应用的开发和部署。

## 准备工作
- 完成实验1.1到1.4
- 安装Docker Compose
- 了解YAML格式

## Docker Compose概述
Docker Compose是一个工具，用于定义和运行多容器Docker应用程序。使用YAML文件配置应用程序的服务，然后使用单个命令创建和启动所有服务。

主要优势：
- 在单个文件中声明式定义应用程序环境
- 一键创建和启动所有服务
- 管理服务依赖关系
- 共享环境变量
- 统一日志处理

## 步骤

### 1. 安装Docker Compose（如果尚未安装）

大多数Docker Desktop版本已包含Docker Compose。如需手动安装：

**Linux**:
```bash
# 下载Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.3/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# 应用可执行权限
sudo chmod +x /usr/local/bin/docker-compose

# 验证安装
docker-compose --version
```

**macOS/Windows**:
如果已安装Docker Desktop，则已包含Docker Compose。

### 2. 创建项目结构

我们将创建一个包含Web应用、Redis和PostgreSQL的完整应用程序：

```bash
# 创建项目目录
mkdir -p compose-demo
cd compose-demo

# 创建Web应用目录
mkdir -p web

# 创建必要的文件
touch docker-compose.yml
touch web/app.py
touch web/requirements.txt
touch web/Dockerfile
```

### 3. 创建Web应用

编辑`web/app.py`文件，创建一个连接Redis和PostgreSQL的Flask应用：

```python
from flask import Flask, jsonify, request
import os
import redis
import psycopg2
import socket
import time
from datetime import datetime

app = Flask(__name__)

# 环境变量配置
REDIS_URL = os.getenv('REDIS_URL', 'redis://redis:6379')
POSTGRES_USER = os.getenv('POSTGRES_USER', 'postgres')
POSTGRES_PASSWORD = os.getenv('POSTGRES_PASSWORD', 'example')
POSTGRES_DB = os.getenv('POSTGRES_DB', 'postgres')
POSTGRES_HOST = os.getenv('POSTGRES_HOST', 'postgres')

# Redis连接
def get_redis_client():
    try:
        client = redis.from_url(REDIS_URL)
        return client
    except Exception as e:
        app.logger.error(f"Redis连接错误: {e}")
        return None

# PostgreSQL连接
def get_postgres_conn():
    try:
        conn = psycopg2.connect(
            user=POSTGRES_USER,
            password=POSTGRES_PASSWORD,
            host=POSTGRES_HOST,
            database=POSTGRES_DB
        )
        return conn
    except Exception as e:
        app.logger.error(f"PostgreSQL连接错误: {e}")
        return None

# 确保数据库表存在
def init_db():
    conn = get_postgres_conn()
    if conn:
        try:
            cursor = conn.cursor()
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS visits (
                    id SERIAL PRIMARY KEY,
                    path VARCHAR(255) NOT NULL,
                    visited_at TIMESTAMP NOT NULL,
                    visitor_ip VARCHAR(50)
                )
            ''')
            conn.commit()
            cursor.close()
            conn.close()
            app.logger.info("数据库初始化成功")
        except Exception as e:
            app.logger.error(f"数据库初始化错误: {e}")

# 初始化数据库（在应用启动时）
@app.before_first_request
def setup():
    # 等待PostgreSQL准备就绪
    max_retries = 5
    retry_count = 0
    
    while retry_count < max_retries:
        if get_postgres_conn():
            init_db()
            break
        else:
            retry_count += 1
            time.sleep(2)  # 等待2秒后重试

# 记录访问到数据库
def record_visit(path, ip):
    conn = get_postgres_conn()
    if conn:
        try:
            cursor = conn.cursor()
            cursor.execute(
                "INSERT INTO visits (path, visited_at, visitor_ip) VALUES (%s, %s, %s)",
                (path, datetime.now(), ip)
            )
            conn.commit()
            cursor.close()
            conn.close()
        except Exception as e:
            app.logger.error(f"记录访问错误: {e}")

# 路由
@app.route('/')
def index():
    # 增加Redis计数器
    redis_client = get_redis_client()
    if redis_client:
        visits = redis_client.incr('visits')
    else:
        visits = "无法连接Redis"
    
    # 记录访问到PostgreSQL
    record_visit('/', request.remote_addr)
    
    return f'''
    <h1>欢迎来到Docker Compose演示!</h1>
    <p>这个页面已被访问 <strong>{visits}</strong> 次</p>
    <p>容器主机名: {socket.gethostname()}</p>
    <p><a href="/stats">查看详细统计</a></p>
    '''

@app.route('/stats')
def stats():
    redis_client = get_redis_client()
    
    # 获取Redis统计信息
    if redis_client:
        total_visits = redis_client.get('visits')
        if total_visits is None:
            total_visits = 0
        else:
            total_visits = int(total_visits)
    else:
        total_visits = "无法连接Redis"
    
    # 获取PostgreSQL统计信息
    conn = get_postgres_conn()
    recent_visits = []
    if conn:
        try:
            cursor = conn.cursor()
            cursor.execute("""
                SELECT path, visited_at, visitor_ip 
                FROM visits 
                ORDER BY visited_at DESC 
                LIMIT 10
            """)
            recent_visits = cursor.fetchall()
            cursor.close()
            conn.close()
        except Exception as e:
            app.logger.error(f"获取统计信息错误: {e}")
    
    # 构建HTML响应
    stats_html = f'''
    <h1>应用统计信息</h1>
    <p>总访问次数: {total_visits}</p>
    <h2>最近10次访问:</h2>
    <table border="1">
        <tr>
            <th>路径</th>
            <th>时间</th>
            <th>访问者IP</th>
        </tr>
    '''
    
    for visit in recent_visits:
        stats_html += f'''
        <tr>
            <td>{visit[0]}</td>
            <td>{visit[1]}</td>
            <td>{visit[2]}</td>
        </tr>
        '''
    
    stats_html += '''
    </table>
    <p><a href="/">返回首页</a></p>
    '''
    
    return stats_html

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

编辑`web/requirements.txt`文件：

```
flask==2.3.2
redis==4.6.0
psycopg2-binary==2.9.6
```

编辑`web/Dockerfile`文件：

```dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

### 4. 创建Docker Compose配置

现在，编辑项目根目录下的`docker-compose.yml`文件：

```yaml
version: '3.8'

services:
  web:
    build: ./web
    ports:
      - "5000:5000"
    environment:
      - REDIS_URL=redis://redis:6379
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=example
      - POSTGRES_DB=postgres
      - POSTGRES_HOST=postgres
    depends_on:
      - redis
      - postgres
    volumes:
      - ./web:/app
    restart: always

  redis:
    image: redis:alpine
    volumes:
      - redis-data:/data
    restart: always

  postgres:
    image: postgres:13
    environment:
      POSTGRES_PASSWORD: example
      POSTGRES_USER: postgres
      POSTGRES_DB: postgres
    volumes:
      - postgres-data:/var/lib/postgresql/data
    restart: always

  adminer:
    image: adminer
    ports:
      - "8080:8080"
    depends_on:
      - postgres
    restart: always

volumes:
  redis-data:
  postgres-data:
```

### 5. 启动服务

```bash
# 启动所有服务（在后台运行）
docker-compose up -d

# 查看正在运行的服务
docker-compose ps

# 查看服务日志
docker-compose logs

# 查看特定服务的日志
docker-compose logs web

# 查看实时日志
docker-compose logs -f
```

### 6. 测试应用

在浏览器中访问以下地址：
- Web应用：http://localhost:5000/
- Adminer（数据库管理）：http://localhost:8080/
  - 系统：PostgreSQL
  - 服务器：postgres
  - 用户名：postgres
  - 密码：example
  - 数据库：postgres

### 7. 探索Docker Compose命令

```bash
# 检查web服务的日志
docker-compose logs web

# 在web容器中执行命令
docker-compose exec web python -c "import socket; print(socket.gethostname())"

# 查看容器中的进程
docker-compose top

# 停止服务但不删除容器
docker-compose stop

# 启动已停止的服务
docker-compose start

# 重启服务
docker-compose restart web

# 暂停服务
docker-compose pause web

# 恢复暂停的服务
docker-compose unpause web
```

### 8. 修改应用并查看变更

由于我们使用了卷挂载，可以实时编辑代码并查看变更：

1. 编辑`web/app.py`，在index路由中添加以下内容：
   ```python
   # 在return语句之前添加
   current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
   ```

   同时更新返回的HTML：
   ```python
   return f'''
   <h1>欢迎来到Docker Compose演示!</h1>
   <p>这个页面已被访问 <strong>{visits}</strong> 次</p>
   <p>容器主机名: {socket.gethostname()}</p>
   <p>当前时间: {current_time}</p>
   <p><a href="/stats">查看详细统计</a></p>
   '''
   ```

2. 刷新浏览器查看变更（由于开启了Flask的调试模式，代码变更应自动重新加载）

### 9. 扩展服务

Docker Compose可以轻松扩展服务：

```bash
# 扩展web服务到3个实例
docker-compose up -d --scale web=3
```

注意：这可能会引发端口冲突，因为多个web实例会尝试绑定到同一端口。解决方法是修改`docker-compose.yml`中的端口映射：

```yaml
# 将web服务的端口映射修改为：
ports:
  - "5000-5002:5000"
```

然后重启并扩展：

```bash
docker-compose down
docker-compose up -d --scale web=3
```

### 10. 管理应用生命周期

```bash
# 停止并移除所有容器、网络，但保留卷
docker-compose down

# 停止并移除所有容器、网络和卷
docker-compose down -v

# 重新构建服务并启动
docker-compose up -d --build
```

## 清理

```bash
# 停止并移除所有资源（保留卷）
docker-compose down

# 如果要移除卷
docker-compose down -v
```

## 思考问题

1. Docker Compose如何简化多容器应用的开发和部署工作流程？
2. Docker Compose文件中的`depends_on`指令确保了服务启动顺序，但它是否确保服务已准备好接受连接？请讨论应用程序中处理服务依赖关系的最佳方法。
3. 卷(volumes)在Docker Compose中有什么作用？为什么对于PostgreSQL和Redis这类有状态服务尤其重要？
4. 如何使用Docker Compose在开发、测试和生产环境中管理不同的配置？

## 扩展练习

1. 将应用扩展为包含更多服务，如Nginx反向代理和Elasticsearch搜索服务
2. 创建多个Compose文件（如docker-compose.override.yml）来管理不同环境的配置
3. 实现服务健康检查，确保只有当依赖服务完全准备好时才启动应用
4. 配置Docker Compose以使用外部网络或已有的卷 