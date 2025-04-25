# 实验1.1：Docker基础与容器操作

## 目标
安装Docker环境并学习基本容器生命周期管理

## 准备工作
- 一台运行Linux、macOS或Windows的计算机
- 管理员/root权限
- 稳定的互联网连接

## Docker架构概述
![Docker架构](https://docs.docker.com/engine/images/architecture.svg)

Docker使用客户端-服务器架构，主要组件包括：
- Docker客户端(docker)：用户界面
- Docker守护进程(dockerd)：管理Docker对象
- Docker镜像：容器的只读模板
- Docker容器：镜像的运行实例
- Docker注册表：存储Docker镜像

## 步骤

### 1. 安装Docker

选择适合您系统的安装方法：

#### Linux (Ubuntu)
```bash
# 更新包索引
sudo apt-get update

# 安装必要的依赖
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# 添加Docker官方GPG密钥
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# 设置稳定版仓库
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# 添加当前用户到docker组（避免每次使用docker命令都需要sudo）
sudo usermod -aG docker $USER
# 注意：需要注销并重新登录才能生效
```

#### macOS
1. 下载并安装[Docker Desktop for Mac](https://www.docker.com/products/docker-desktop)
2. 按照安装向导完成安装

#### Windows
1. 下载并安装[Docker Desktop for Windows](https://www.docker.com/products/docker-desktop)
2. 按照安装向导完成安装
3. 确保WSL 2后端已启用

### 2. 验证安装

重启终端或命令提示符，然后验证Docker安装：

```bash
# 检查Docker版本
docker --version

# 查看Docker系统信息
docker info

# 验证Docker能够运行容器
docker run hello-world
```

如果看到"Hello from Docker!"消息，说明Docker安装成功。

### 3. 基本容器操作

#### 运行第一个交互式容器

```bash
# 运行Ubuntu容器并进入交互式Shell
docker run -it ubuntu bash

# 在容器内运行一些命令
ls
cat /etc/os-release
uname -a

# 退出容器
exit
```

#### 容器生命周期管理

```bash
# 列出所有运行中的容器
docker ps

# 列出所有容器（包括已停止的）
docker ps -a

# 以分离模式运行Nginx容器
docker run -d -p 8080:80 --name my-nginx nginx

# 检查容器是否运行
docker ps

# 访问Nginx网页
# 在浏览器中打开 http://localhost:8080

# 查看容器日志
docker logs my-nginx

# 停止容器
docker stop my-nginx

# 启动容器
docker start my-nginx

# 重启容器
docker restart my-nginx

# 删除容器（必须先停止）
docker stop my-nginx
docker rm my-nginx
```

#### 管理Docker镜像

```bash
# 列出本地镜像
docker images

# 拉取特定版本的镜像
docker pull nginx:1.21

# 查找Docker Hub上的镜像
docker search redis
```

### 4. 深入探索Docker

#### 检查容器资源使用情况

```bash
# 运行Redis容器
docker run -d --name my-redis redis

# 查看容器的资源统计信息
docker stats my-redis

# 查看容器的详细信息
docker inspect my-redis
```

#### 执行容器内的命令

```bash
# 在运行中的容器内执行命令
docker exec -it my-redis redis-cli

# 在Redis CLI中执行命令
SET greeting "Hello Docker"
GET greeting
exit
```

#### 容器网络测试

```bash
# 创建一个带有工具的容器来测试网络
docker run -it --rm nicolaka/netshoot

# 在容器内尝试一些网络命令
ping google.com
traceroute google.com
exit
```

### 5. 清理

```bash
# 停止所有运行中的容器
docker stop $(docker ps -q)

# 删除所有容器
docker rm $(docker ps -aq)

# 删除未使用的镜像
docker image prune -a
```

## 思考问题

1. Docker容器与传统虚拟机的主要区别是什么？
2. 容器是如何共享主机操作系统内核的？
3. Docker如何使用Linux namespace和cgroups技术实现隔离和资源限制？
4. 为什么容器比虚拟机启动更快？

## 扩展练习

1. 尝试限制容器的资源使用：
   ```bash
   docker run -d --name limited-nginx --cpus=0.5 --memory=200m nginx
   ```

2. 探索Docker Hub并找到一个有趣的镜像运行
3. 研究Docker的网络模式，尝试不同网络模式的容器 