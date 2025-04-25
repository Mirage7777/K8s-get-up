# 实验1.3：多容器应用与Docker网络

## 目标
了解如何使用Docker网络连接多个容器，构建一个包含Node.js应用和Redis数据库的多容器应用程序。

## 准备工作
- 完成实验1.1和1.2
- 基本的JavaScript/Node.js知识
- 文本编辑器

## Docker网络概述
Docker提供了多种网络驱动程序，用于连接容器：
- bridge：默认网络驱动程序，适用于在同一Docker主机上运行的容器
- host：移除容器和Docker主机之间的网络隔离
- overlay：连接多个Docker守护进程，使swarm服务能够相互通信
- macvlan：为容器分配MAC地址，使其显示为网络上的物理设备
- none：禁用容器的所有联网

## 步骤

### 1. 创建项目结构

```bash
# 创建项目目录
mkdir -p node-redis-app
cd node-redis-app

# 创建必需的文件
touch package.json
touch app.js
touch Dockerfile
```

### 2. 创建Node.js应用

编辑`package.json`文件：

```json
{
  "name": "node-redis-app",
  "version": "1.0.0",
  "description": "Node.js and Redis Docker example",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "redis": "^4.6.7"
  }
}
```

编辑`app.js`文件，创建一个与Redis交互的简单Express应用：

```javascript
const express = require('express');
const redis = require('redis');

const app = express();
const port = 3000;

// 创建Redis客户端
// 'redis'是Redis容器的主机名，该名称将由Docker网络DNS解析
const client = redis.createClient({
    url: 'redis://redis:6379'
});

// 连接到Redis
async function connectRedis() {
    await client.connect();
    console.log('Connected to Redis');
}

// 应用启动时连接Redis
connectRedis().catch(console.error);

// 处理错误
client.on('error', (err) => {
    console.error('Redis error:', err);
});

// 设置路由
app.get('/', async (req, res) => {
    try {
        // 增加访问计数
        const count = await client.incr('visits');
        res.send(`Welcome to our Node.js + Redis app! You are visitor #${count}`);
    } catch (error) {
        console.error('Error handling request:', error);
        res.status(500).send('Internal Server Error');
    }
});

app.get('/stats', async (req, res) => {
    try {
        const count = await client.get('visits');
        res.json({
            visits: count || 0,
            timestamp: new Date().toISOString()
        });
    } catch (error) {
        console.error('Error getting stats:', error);
        res.status(500).send('Internal Server Error');
    }
});

// 启动服务器
app.listen(port, () => {
    console.log(`App listening at http://localhost:${port}`);
});

// 优雅关闭
process.on('SIGINT', async () => {
    console.log('Closing Redis connection...');
    await client.quit();
    process.exit();
});
```

### 3. 创建Dockerfile

```dockerfile
FROM node:16-alpine

WORKDIR /app

COPY package.json .

RUN npm install

COPY . .

EXPOSE 3000

CMD ["node", "app.js"]
```

### 4. 创建自定义网络

Docker网络允许容器通过名称相互通信，无需IP地址：

```bash
# 创建名为my-network的网络
docker network create my-network

# 查看网络
docker network ls

# 检查网络详细信息
docker network inspect my-network
```

### 5. 启动Redis容器并连接到网络

```bash
# 启动Redis容器并将其连接到网络
docker run -d --name redis --network my-network redis:alpine

# 验证Redis容器正在运行
docker ps | grep redis
```

### 6. 构建和运行Node.js应用容器

```bash
# 在项目目录中构建容器
docker build -t node-redis-app .

# 运行容器并连接到同一网络
docker run -d --name node-app --network my-network -p 3000:3000 node-redis-app

# 检查容器是否在运行
docker ps | grep node-app
```

### 7. 测试应用

在浏览器中访问：
- http://localhost:3000/
- http://localhost:3000/stats

刷新首页多次，观察访问计数器增加。查看统计页面的更新。

### 8. 探索容器网络交互

```bash
# 进入Node.js容器
docker exec -it node-app sh

# 尝试ping Redis容器
ping redis

# 使用wget检查Redis连接
apk add --no-cache wget
wget -O- redis:6379

# 退出容器
exit
```

### 9. 探索网络DNS解析

```bash
# 进入Redis容器
docker exec -it redis sh

# 安装网络工具
apk add --no-cache iputils
apk add --no-cache curl

# 尝试ping Node.js容器
ping node-app

# 退出容器
exit
```

### 10. 扩展应用：添加Redis Commander（Web UI）

```bash
# 运行Redis Commander容器
docker run -d --name redis-commander --network my-network -p 8081:8081 \
  -e REDIS_HOSTS=redis \
  rediscommander/redis-commander:latest

# 检查容器
docker ps | grep redis-commander
```

在浏览器中访问 http://localhost:8081/ 来使用Redis Commander界面查看Redis数据。

### 11. 使用Docker网络检查工具

```bash
# 使用nicolaka/netshoot工具来检查网络
docker run -it --rm --network my-network nicolaka/netshoot

# 在netshoot容器内尝试各种网络命令
nslookup redis
nslookup node-app
dig redis
traceroute redis
exit
```

## 12. 清理

```bash
# 停止并删除所有容器
docker stop node-app redis redis-commander
docker rm node-app redis redis-commander

# 删除网络
docker network rm my-network

# 删除镜像（可选）
docker rmi node-redis-app
```

## 思考问题

1. Docker如何在自定义网络中实现容器名称的DNS解析？这对微服务架构有什么意义？
2. Redis容器和Node.js容器虽然在同一网络中，但它们的文件系统和进程空间是如何隔离的？
3. 如何保障保存在Redis中的数据在容器重启后不会丢失？
4. Docker网络的桥接模式和overlay模式有什么不同，各自适用于什么场景？

## 扩展练习

1. 使用Redis持久化功能，为Redis容器添加卷(volume)挂载，使数据持久化
2. 添加一个第三个服务，例如MongoDB数据库，并修改Node.js应用同时连接Redis和MongoDB
3. 使用Docker Compose简化多容器应用的部署（参考下一个实验）
4. 尝试不同的网络模式（如host或macvlan）并比较性能差异 