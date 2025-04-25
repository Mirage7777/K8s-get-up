# 实验1.2：容器化Python Web应用

## 目标
学习Dockerfile基础知识，构建镜像，以及运行和管理容器化的Web应用程序。

## 准备工作
- 完成实验1.1，已安装Docker
- 基本Python知识
- 文本编辑器（例如VSCode、Sublime Text或Vim）

## 步骤

### 1. 创建项目结构

首先，我们需要创建一个项目目录并设置文件：

```bash
# 创建项目目录
mkdir -p flask-docker-app
cd flask-docker-app

# 使用您喜欢的编辑器创建文件
# 示例：使用touch命令创建空文件
touch app.py
touch requirements.txt
touch Dockerfile
```

### 2. 编写Flask应用

现在，我们将创建一个简单的Flask应用。使用您的编辑器打开`app.py`，并添加以下代码：

```python
from flask import Flask, jsonify
import socket

app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello from Docker!"

@app.route('/info')
def info():
    hostname = socket.gethostname()
    ip = socket.gethostbyname(hostname)
    return jsonify({
        'hostname': hostname,
        'ip_address': ip,
        'message': 'This information is coming from inside a Docker container!'
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

### 3. 创建项目依赖文件

编辑`requirements.txt`文件，指定需要的Python包：

```
flask==2.3.2
```

### 4. 创建Dockerfile

Dockerfile是Docker构建镜像的指令集。编辑`Dockerfile`文件，添加以下内容：

```dockerfile
# 使用Python官方镜像作为基础镜像
FROM python:3.10-slim

# 设置工作目录
WORKDIR /app

# 将依赖文件复制到容器中
COPY requirements.txt .

# 安装依赖
RUN pip install --no-cache-dir -r requirements.txt

# 将应用程序代码复制到容器中
COPY . .

# 暴露应用程序使用的端口
EXPOSE 5000

# 定义启动应用程序的命令
CMD ["python", "app.py"]
```

### 5. 构建Docker镜像

现在，让我们构建Docker镜像：

```bash
# 确保您位于项目目录中
docker build -t flask-app .
```

这个命令会使用当前目录（`.`）中的Dockerfile构建一个名为`flask-app`的镜像。

### 6. 运行容器

构建完成后，我们可以从这个镜像运行一个容器：

```bash
# 运行容器，并将容器内的5000端口映射到主机的5000端口
docker run -p 5000:5000 --name my-flask-app flask-app
```

### 7. 测试应用程序

在浏览器中打开以下URL测试应用程序：
- 基本页面：http://localhost:5000/
- 容器信息页面：http://localhost:5000/info

### 8. 探索Docker层和镜像优化

让我们探索刚刚构建的镜像的层结构：

```bash
# 检查镜像的层
docker history flask-app
```

查看镜像的大小：

```bash
docker images flask-app
```

### 9. 以分离模式运行容器

按下`Ctrl+C`停止当前正在运行的容器，然后以分离模式启动它：

```bash
# 移除之前创建的容器
docker rm my-flask-app

# 以分离模式运行
docker run -d -p 5000:5000 --name my-flask-app flask-app

# 确认容器正在运行
docker ps
```

### 10. 查看容器日志和状态

```bash
# 查看容器日志
docker logs my-flask-app

# 持续查看日志
docker logs -f my-flask-app

# 查看容器详细信息
docker inspect my-flask-app
```

### 11. 优化Dockerfile

现在，让我们创建一个优化的Dockerfile，名为`Dockerfile.optimized`：

```dockerfile
# 使用Python官方镜像作为基础镜像
FROM python:3.10-slim

# 设置工作目录
WORKDIR /app

# 设置环境变量
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

# 将依赖文件复制到容器中
COPY requirements.txt .

# 安装依赖
RUN pip install --no-cache-dir -r requirements.txt

# 创建非root用户
RUN adduser --disabled-password --gecos "" appuser

# 将应用程序代码复制到容器中
COPY . .

# 修改所有权
RUN chown -R appuser:appuser /app

# 切换到非root用户
USER appuser

# 暴露应用程序使用的端口
EXPOSE 5000

# 使用健康检查
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:5000/ || exit 1

# 定义启动应用程序的命令
CMD ["python", "app.py"]
```

构建优化的镜像：

```bash
docker build -t flask-app-optimized -f Dockerfile.optimized .
```

运行优化的容器：

```bash
docker run -d -p 5001:5000 --name optimized-flask-app flask-app-optimized
```

### 12. 比较两个镜像

```bash
# 比较镜像大小和层数
docker images | grep flask-app
docker history flask-app
docker history flask-app-optimized
```

## 清理

完成实验后，清理资源：

```bash
# 停止并删除容器
docker stop my-flask-app optimized-flask-app
docker rm my-flask-app optimized-flask-app

# 删除镜像（可选）
docker rmi flask-app flask-app-optimized
```

## 思考问题

1. Docker镜像层是如何工作的？为什么合并命令（如使用`&&`连接多个RUN命令）可以减小镜像大小？
2. 为什么在Dockerfile中使用非root用户运行应用程序更安全？
3. 在开发和生产环境中，您可能需要使用不同的Docker配置。如何管理这些差异？
4. Docker镜像如何实现应用程序的可移植性？

## 扩展练习

1. 修改Flask应用，添加一个新的路由，例如`/time`，返回当前服务器时间
2. 为应用程序创建一个Docker Compose配置，包括Flask应用和Redis缓存
3. 实现多阶段构建的Dockerfile，以进一步减小最终镜像大小 