# 实验1.4：使用多阶段构建优化Docker镜像

## 目标
学习如何使用Docker多阶段构建来创建更小、更安全的容器镜像，减少攻击面并提高部署效率。

## 准备工作
- 完成实验1.1到1.3
- 基本的Go语言知识（不需要深入了解）
- 文本编辑器

## 多阶段构建概述

多阶段构建是一种Docker构建技术，允许您：
- 在同一Dockerfile中使用多个FROM语句
- 从一个阶段复制文件到另一个阶段
- 仅保留最终阶段中需要的文件
- 显著减小最终镜像大小
- 从构建环境中分离运行环境

这种方法特别适合编译型语言（如Go、Java、C++等），其中构建工具和依赖项在运行时不需要。

## 步骤

### 1. 创建项目结构

首先，创建一个项目目录并设置文件：

```bash
# 创建项目目录
mkdir -p go-web-app
cd go-web-app

# 创建必需的文件
touch main.go
touch Dockerfile
touch Dockerfile.single
```

### 2. 编写简单的Go Web应用

`main.go` 将包含一个简单的HTTP服务器：

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"os"
	"runtime"
)

func main() {
	http.HandleFunc("/", handleRoot)
	http.HandleFunc("/info", handleInfo)

	port := getEnv("PORT", "8080")
	log.Printf("Server starting on port %s...\n", port)
	log.Fatal(http.ListenAndServe(":"+port, nil))
}

func handleRoot(w http.ResponseWriter, r *http.Request) {
	log.Println("Handling request to /")
	fmt.Fprintf(w, "Hello from Go in Docker! Try /info for more details.")
}

func handleInfo(w http.ResponseWriter, r *http.Request) {
	log.Println("Handling request to /info")
	info := fmt.Sprintf(`
Server Information:
-------------------
Go Version: %s
OS/Arch: %s/%s
Hostname: %s
`,
		runtime.Version(),
		runtime.GOOS,
		runtime.GOARCH,
		getHostname(),
	)
	fmt.Fprint(w, info)
}

func getEnv(key, fallback string) string {
	if value, exists := os.LookupEnv(key); exists {
		return value
	}
	return fallback
}

func getHostname() string {
	hostname, err := os.Hostname()
	if err != nil {
		return "unknown"
	}
	return hostname
}
```

### 3. 创建单阶段Dockerfile

`Dockerfile.single` 将是我们的单阶段构建示例：

```dockerfile
FROM golang:1.18

WORKDIR /app

COPY main.go .

RUN go build -o server main.go

EXPOSE 8080

CMD ["/app/server"]
```

### 4. 创建多阶段Dockerfile

现在，创建使用多阶段构建的优化Dockerfile：

```dockerfile
# 构建阶段
FROM golang:1.18 AS builder

WORKDIR /app

# 复制源代码
COPY main.go .

# 构建应用程序
# -ldflags="-s -w" 减小二进制文件大小
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o server main.go

# 运行阶段
FROM alpine:latest

# 安装CA证书 - 对于HTTPS请求是必要的
RUN apk --no-cache add ca-certificates

WORKDIR /root/

# 从构建阶段复制二进制文件
COPY --from=builder /app/server .

# 暴露端口
EXPOSE 8080

# 运行应用
CMD ["./server"]
```

### 5. 构建单阶段镜像

```bash
# 构建单阶段版本
docker build -t go-app-single -f Dockerfile.single .

# 检查镜像大小
docker images go-app-single
```

### 6. 构建多阶段镜像

```bash
# 构建多阶段版本
docker build -t go-app-multi .

# 检查镜像大小
docker images go-app-multi
```

### 7. 比较镜像大小和层

```bash
# 比较两个镜像的大小
docker images | grep go-app

# 检查单阶段镜像的层
docker history go-app-single

# 检查多阶段镜像的层
docker history go-app-multi
```

### 8. 运行并测试容器

```bash
# 运行单阶段容器
docker run -d -p 8081:8080 --name single-stage go-app-single

# 运行多阶段容器
docker run -d -p 8082:8080 --name multi-stage go-app-multi

# 测试两个容器
curl http://localhost:8081/
curl http://localhost:8081/info

curl http://localhost:8082/
curl http://localhost:8082/info
```

### 9. 更进一步优化：使用scratch基础镜像

创建一个新的Dockerfile，名为`Dockerfile.scratch`：

```dockerfile
# 构建阶段
FROM golang:1.18 AS builder

WORKDIR /app

# 复制源代码
COPY main.go .

# 构建静态链接的二进制文件
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -a -o server main.go

# 最小化运行阶段 - 使用空白镜像
FROM scratch

# 从构建阶段复制二进制文件
COPY --from=builder /app/server /server

# 暴露端口
EXPOSE 8080

# 运行应用
CMD ["/server"]
```

构建和测试scratch镜像：

```bash
# 构建scratch版本
docker build -t go-app-scratch -f Dockerfile.scratch .

# 检查镜像大小
docker images go-app-scratch

# 运行容器
docker run -d -p 8083:8080 --name scratch-stage go-app-scratch

# 测试容器
curl http://localhost:8083/
curl http://localhost:8083/info
```

### 10. 安全扫描比较

使用Docker's Snyk扫描或Trivy等工具比较不同镜像的安全性：

```bash
# 如果您已安装Trivy
trivy image go-app-single
trivy image go-app-multi
trivy image go-app-scratch
```

### 11. 清理

```bash
# 停止并删除所有容器
docker stop single-stage multi-stage scratch-stage
docker rm single-stage multi-stage scratch-stage

# 删除镜像
docker rmi go-app-single go-app-multi go-app-scratch
```

## 最终尺寸对比

创建一个表格记录三个镜像的大小差异：

| 镜像名称       | 构建类型 | 大小   | 基础镜像      |
| -------------- | -------- | ------ | ------------- |
| go-app-single  | 单阶段   | ~800MB | golang:1.18   |
| go-app-multi   | 多阶段   | ~10MB  | alpine:latest |
| go-app-scratch | 多阶段   | ~2MB   | scratch       |

## 思考问题

1. 为什么多阶段构建在生产环境部署中很重要？
2. 镜像大小如何影响容器的启动时间、网络传输效率和安全性？
3. 使用较小基础镜像（如alpine或scratch）的优势和潜在挑战是什么？
4. 多阶段构建如何改善CI/CD流程和部署效率？

## 扩展练习

1. 为Go应用添加更多功能（例如，连接数据库、调用外部API），并仍使用多阶段构建保持较小的镜像大小
2. 尝试使用不同的基础镜像（如distroless），并比较结果
3. 为Java、Python或Node.js应用实现多阶段构建，处理这些语言特有的挑战
4. 研究并实现Docker层缓存优化策略，加快构建速度 