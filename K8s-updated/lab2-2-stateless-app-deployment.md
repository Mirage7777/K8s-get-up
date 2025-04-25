# 实验2.2：在Kubernetes上部署无状态应用

## 目标
掌握Kubernetes的基本对象（Pod、Deployment、Service）概念，学习如何在Kubernetes集群上部署和管理无状态应用程序。

## 准备工作
- 完成实验2.1，拥有一个运行中的Kubernetes集群（Minikube）
- 熟悉基本的kubectl命令
- 了解YAML格式

## Kubernetes核心概念

在本实验中，我们将重点关注以下Kubernetes对象：

1. **Pod**：Kubernetes中最小的可部署单元，包含一个或多个容器
2. **Deployment**：管理Pod的副本集，提供声明式更新和自愈能力
3. **Service**：为一组Pod提供统一的网络访问点
4. **Label & Selector**：用于组织和选择一组对象

## 步骤

### 1. 创建简单的Flask应用

首先，我们将创建一个简单的Flask应用，并将其容器化：

```bash
# 创建项目目录
mkdir -p flask-k8s
cd flask-k8s

# 创建应用文件
cat > app.py << 'EOF'
from flask import Flask, jsonify
import os
import socket

app = Flask(__name__)

@app.route('/')
def hello():
    return jsonify({
        'message': 'Hello from Kubernetes!',
        'hostname': socket.gethostname(),
        'pod_ip': socket.gethostbyname(socket.gethostname()),
        'environment': os.environ.get('ENVIRONMENT', 'development')
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

# 创建Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
EOF

# 创建requirements.txt
cat > requirements.txt << 'EOF'
flask==2.3.2
EOF
```

### 2. 构建并推送Docker镜像

```bash
# 构建Docker镜像
docker build -t flask-k8s-app:v1 .

# 让Minikube可以使用本地Docker镜像
# 方法1: 使用Minikube的Docker环境
eval $(minikube docker-env)
docker build -t flask-k8s-app:v1 .

# 方法2: 加载镜像到Minikube
docker build -t flask-k8s-app:v1 .
minikube image load flask-k8s-app:v1
```

### 3. 创建简单的Pod

让我们首先创建一个单独的Pod来运行应用：

```bash
# 创建Pod配置文件
cat > flask-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: flask-pod
  labels:
    app: flask
spec:
  containers:
  - name: flask
    image: flask-k8s-app:v1
    imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 5000
    env:
    - name: ENVIRONMENT
      value: "kubernetes-pod"
EOF

# 应用配置创建Pod
kubectl apply -f flask-pod.yaml

# 检查Pod状态
kubectl get pods
kubectl describe pod flask-pod
```

### 4. 端口转发以访问Pod

```bash
# 使用端口转发暂时访问Pod
kubectl port-forward flask-pod 8080:5000
```

在浏览器中访问 http://localhost:8080 查看应用响应。

### 5. 创建Deployment

单个Pod不提供高可用性。现在，让我们创建一个Deployment来管理多个Pod副本：

```bash
# 创建Deployment配置文件
cat > flask-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-deployment
  labels:
    app: flask
spec:
  replicas: 3
  selector:
    matchLabels:
      app: flask
  template:
    metadata:
      labels:
        app: flask
    spec:
      containers:
      - name: flask
        image: flask-k8s-app:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
        env:
        - name: ENVIRONMENT
          value: "kubernetes-deployment"
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        readinessProbe:
          httpGet:
            path: /
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 5
EOF

# 删除之前创建的单个Pod
kubectl delete pod flask-pod

# 应用Deployment配置
kubectl apply -f flask-deployment.yaml

# 检查Deployment状态
kubectl get deployments
kubectl get pods
```

### 6. 创建Service以访问应用

现在我们有了一个运行多个Pod的Deployment，接下来创建一个Service来暴露应用：

```bash
# 创建Service配置文件
cat > flask-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  selector:
    app: flask
  ports:
  - port: 80
    targetPort: 5000
  type: ClusterIP
EOF

# 应用Service配置
kubectl apply -f flask-service.yaml

# 检查Service状态
kubectl get services
```

### 7. 创建NodePort Service以从外部访问

ClusterIP Service仅在集群内部可访问。创建一个NodePort Service使应用从外部可访问：

```bash
# 创建NodePort Service配置
cat > flask-nodeport.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: flask-nodeport
spec:
  selector:
    app: flask
  ports:
  - port: 80
    targetPort: 5000
    nodePort: 30080
  type: NodePort
EOF

# 应用配置
kubectl apply -f flask-nodeport.yaml

# 获取访问URL
minikube service flask-nodeport --url
```

使用提供的URL在浏览器中访问应用。刷新页面，观察hostname如何变化，这表明请求被分发到不同的Pod实例。

### 8. 使用kubectl查看和调试

```bash
# 查看所有资源
kubectl get all

# 查看Pod日志
kubectl logs -l app=flask

# 查看特定Pod的日志
POD_NAME=$(kubectl get pods -l app=flask -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD_NAME

# 在Pod中执行命令
kubectl exec -it $POD_NAME -- /bin/bash
```

### 9. 更新应用程序 - 滚动更新

修改我们的Flask应用，然后执行滚动更新：

```bash
# 修改app.py添加一个新路由
cat > app.py << 'EOF'
from flask import Flask, jsonify
import os
import socket
import datetime

app = Flask(__name__)

@app.route('/')
def hello():
    return jsonify({
        'message': 'Hello from Kubernetes!',
        'hostname': socket.gethostname(),
        'pod_ip': socket.gethostbyname(socket.gethostname()),
        'environment': os.environ.get('ENVIRONMENT', 'development'),
        'version': 'v2'
    })

@app.route('/time')
def get_time():
    return jsonify({
        'current_time': datetime.datetime.now().isoformat(),
        'hostname': socket.gethostname()
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

# 构建新版本镜像
docker build -t flask-k8s-app:v2 .
minikube image load flask-k8s-app:v2

# 更新Deployment使用新镜像
kubectl set image deployment/flask-deployment flask=flask-k8s-app:v2

# 观察滚动更新过程
kubectl rollout status deployment/flask-deployment
kubectl get pods
```

访问应用，验证新版本是否已部署（应该显示version: v2和新的/time路由）。

### 10. 使用kubectl scale命令扩展应用

```bash
# 将副本数扩展到5个
kubectl scale deployment flask-deployment --replicas=5

# 查看变化
kubectl get pods
```

### 11. 回滚部署

如果新版本出现问题，可以执行回滚：

```bash
# 查看部署历史
kubectl rollout history deployment/flask-deployment

# 回滚到上一版本
kubectl rollout undo deployment/flask-deployment

# 查看Pod，确认版本已回滚
kubectl get pods
```

### 12. 探索标签和选择器

标签和选择器是Kubernetes中组织和选择资源的关键：

```bash
# 查看带有特定标签的Pod
kubectl get pods -l app=flask

# 添加自定义标签
kubectl label pods -l app=flask environment=demo

# 使用多个标签查询
kubectl get pods -l app=flask,environment=demo
```

### 13. 清理资源

完成实验后，清理创建的资源：

```bash
# 删除所有资源
kubectl delete service flask-service flask-nodeport
kubectl delete deployment flask-deployment
```

## 思考问题

1. Deployment如何确保应用程序的高可用性？滚动更新的机制是什么？
2. Service如何将流量负载均衡到多个Pod？不同类型的Service（ClusterIP、NodePort、LoadBalancer）各有什么用途？
3. 标签和选择器在Kubernetes中扮演什么角色？它们如何实现松耦合架构？
4. Pod内为什么可以包含多个容器？这种设计模式适用于哪些场景？

## 扩展练习

1. 为应用配置水平自动扩缩容（HorizontalPodAutoscaler）
2. 尝试在Minikube上设置Ingress控制器，并通过域名访问应用
3. 实现蓝绿部署或金丝雀部署策略
4. 创建一个包含多个容器的Pod（边车模式），例如主应用容器和日志收集容器 