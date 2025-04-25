# 实验2.1：设置Kubernetes集群

## 目标
建立本地Kubernetes环境，了解Kubernetes基本架构和组件，掌握`kubectl`命令行工具的使用。

## 准备工作
- 完成Docker基础实验（1.1-1.5）
- 至少8GB内存的计算机
- 稳定的互联网连接

## Kubernetes架构概述

Kubernetes是一个容器编排平台，用于自动化应用部署、扩展和管理。其主要组件包括：

**控制平面组件**：
- **kube-apiserver**：暴露Kubernetes API，是整个集群的前端
- **etcd**：一致且高可用的键值存储，存储所有集群数据
- **kube-scheduler**：监视新创建的Pod，并决定在哪个节点上运行
- **kube-controller-manager**：运行控制器进程，如节点控制器、副本控制器等
- **cloud-controller-manager**：（可选）与云提供商交互

**节点组件**：
- **kubelet**：确保容器在Pod中运行
- **kube-proxy**：维护节点上的网络规则，实现服务抽象
- **容器运行时**：运行容器的软件，如Docker、containerd等

## 步骤

### 1. 安装kubectl

kubectl是与Kubernetes集群交互的命令行工具。安装方法因操作系统而异：

#### Linux
```bash
# 下载最新版本
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# 设置执行权限
chmod +x kubectl

# 移动到PATH中的目录
sudo mv kubectl /usr/local/bin/
```

#### macOS
```bash
# 使用Homebrew
brew install kubectl

# 或使用curl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

#### Windows
```powershell
# 使用PowerShell
curl.exe -LO "https://dl.k8s.io/release/v1.27.3/bin/windows/amd64/kubectl.exe"

# 将kubectl.exe添加到PATH
# 将文件移动到C:\Windows\System32\或其他在PATH中的目录
```

验证安装：
```bash
kubectl version --client
```

### 2. 安装Minikube

Minikube是一个工具，可在本地计算机上运行单节点Kubernetes集群。

#### Linux
```bash
# 下载并安装
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

#### macOS
```bash
# 使用Homebrew
brew install minikube

# 或使用curl
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
sudo install minikube-darwin-amd64 /usr/local/bin/minikube
```

#### Windows
```powershell
# 使用Chocolatey
choco install minikube

# 或下载安装程序
# 从https://github.com/kubernetes/minikube/releases下载.exe文件
# 添加到PATH
```

### 3. 启动Minikube集群

```bash
# 默认启动
minikube start

# 或指定驱动程序和资源
minikube start --driver=docker --cpus=2 --memory=4g
```

### 4. 验证集群状态

```bash
# 检查集群信息
kubectl cluster-info

# 检查节点状态
kubectl get nodes

# 查看minikube状态
minikube status
```

### 5. 了解Kubernetes命名空间

命名空间用于将集群资源划分为虚拟集群。

```bash
# 列出所有命名空间
kubectl get namespaces

# 查看特定命名空间中的资源
kubectl get pods -n kube-system
```

### 6. 探索Kubernetes仪表板

```bash
# 启动Kubernetes仪表板
minikube dashboard
```

Kubernetes仪表板是一个基于Web的UI，用于管理集群资源。在仪表板中浏览不同资源类型：
- 工作负载（Pods、Deployments、ReplicaSets）
- 服务
- 配置（ConfigMaps、Secrets）
- 存储（PersistentVolumes）

### 7. 部署第一个应用

我们将部署一个简单的Nginx应用：

```bash
# 使用kubectl创建部署
kubectl create deployment nginx --image=nginx

# 查看部署状态
kubectl get deployments

# 查看创建的Pod
kubectl get pods
```

### 8. 公开服务

将Nginx部署暴露为服务，使其可访问：

```bash
# 创建一个服务，将Deployment暴露到集群外
kubectl expose deployment nginx --port=80 --type=NodePort

# 查看创建的服务
kubectl get services

# 获取访问URL
minikube service nginx --url
```

使用浏览器访问提供的URL，应该能看到Nginx欢迎页面。

### 9. 深入了解Pod

Pod是Kubernetes中最小的可部署单元。

```bash
# 获取Pod的详细信息
kubectl describe pod $(kubectl get pods -o jsonpath='{.items[0].metadata.name}')

# 查看Pod日志
kubectl logs $(kubectl get pods -o jsonpath='{.items[0].metadata.name}')

# 进入Pod内部执行命令
kubectl exec -it $(kubectl get pods -o jsonpath='{.items[0].metadata.name}') -- /bin/bash
```

在Pod内部执行一些命令，例如：
```bash
# 在Pod内查看Nginx配置
cat /etc/nginx/nginx.conf

# 检查Nginx进程
ps aux

# 退出Pod
exit
```

### 10. 使用清单文件

到目前为止，我们使用了命令式命令创建资源。实际应用中，通常使用声明式YAML文件。

创建一个名为`nginx-deployment.yaml`的文件：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-declarative
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

应用此清单：

```bash
kubectl apply -f nginx-deployment.yaml

# 验证部署
kubectl get deployments
kubectl get pods
```

### 11. 探索Kubernetes控制平面

Minikube在单节点上运行Kubernetes控制平面组件。

```bash
# 查看kube-system命名空间中的Pod
kubectl get pods -n kube-system

# 查看某个控制平面组件的详细信息
kubectl describe pod -n kube-system kube-apiserver-minikube
```

### 12. 清理

完成实验后，可以清理资源或停止Minikube：

```bash
# 删除所有创建的资源
kubectl delete deployment nginx nginx-declarative
kubectl delete service nginx

# 停止Minikube
minikube stop

# 如果不再需要，可以删除Minikube集群
# minikube delete
```

## 思考问题

1. Kubernetes控制平面组件之间如何协作以维护集群的期望状态？
2. kube-scheduler是如何决定将Pod分配到哪个节点上的？
3. 生产环境中的Kubernetes集群与Minikube有哪些主要区别？
4. 命令式管理与声明式管理在Kubernetes中各有什么优缺点？

## 扩展练习

1. 设置Minikube，并启用多个附加组件，例如Ingress、Registry和Metrics Server
2. 探索其他Kubernetes本地开发工具，如Kind或k3d，了解它们与Minikube的区别
3. 使用`kubectl explain`命令深入研究不同Kubernetes资源的字段
4. 尝试在Minikube上部署一个有状态应用，如MySQL，并理解其中的挑战 