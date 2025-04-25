# 实验2.3：Kubernetes扩展和自愈机制

## 目标
深入了解Kubernetes的自动扩展和自愈机制，学习如何配置和验证这些功能，理解Kubernetes如何保障应用的高可用性。

## 准备工作
- 完成实验2.1和2.2
- 运行中的Minikube集群
- 基本的kubectl命令知识

## Kubernetes自愈和扩展机制概述

Kubernetes提供了多种机制来保障应用程序的健康运行和自动扩展：

1. **自愈机制**：通过控制器确保当Pod故障时会自动重新创建
2. **存活探针（Liveness Probe）**：检测容器是否仍在运行
3. **就绪探针（Readiness Probe）**：检测容器是否准备好接收流量
4. **启动探针（Startup Probe）**：检测容器中的应用是否已经启动
5. **水平自动扩缩容（HPA）**：根据CPU等指标自动扩展Pod数量
6. **垂直自动扩缩容（VPA）**：调整Pod的CPU和内存请求/限制

## 步骤

### 1. 准备测试应用

我们将创建一个包含探针和可控制资源使用的应用：

```bash
# 创建项目目录
mkdir -p scaling-demo
cd scaling-demo

# 创建应用文件
cat > app.py << 'EOF'
from flask import Flask, jsonify, request
import os
import socket
import time
import threading
import logging

logging.basicConfig(level=logging.INFO)
app = Flask(__name__)

# 全局状态跟踪
health_status = True
ready_status = True
cpu_load = False
load_thread = None

@app.route('/')
def hello():
    return jsonify({
        'message': 'Kubernetes Scaling & Self-healing Demo',
        'hostname': socket.gethostname(),
        'healthy': health_status,
        'ready': ready_status,
        'cpu_load': cpu_load
    })

@app.route('/health')
def health():
    if health_status:
        return jsonify({'status': 'healthy'})
    else:
        return jsonify({'status': 'unhealthy'}), 500

@app.route('/ready')
def ready():
    if ready_status:
        return jsonify({'status': 'ready'})
    else:
        return jsonify({'status': 'not ready'}), 503

@app.route('/fail')
def fail():
    global health_status
    health_status = False
    return jsonify({'status': 'health check will now fail'})

@app.route('/recover')
def recover():
    global health_status
    health_status = True
    return jsonify({'status': 'health check recovered'})

@app.route('/not-ready')
def not_ready():
    global ready_status
    ready_status = False
    return jsonify({'status': 'readiness check will now fail'})

@app.route('/ready-again')
def ready_again():
    global ready_status
    ready_status = True
    return jsonify({'status': 'readiness check recovered'})

def cpu_intensive_task():
    """消耗CPU资源的函数"""
    logging.info("Starting CPU intensive task")
    while cpu_load:
        # 执行计算密集型操作
        _ = [i**2 for i in range(10000000)]
        time.sleep(0.01)  # 短暂休息，避免完全阻塞
    logging.info("CPU intensive task stopped")

@app.route('/start-load')
def start_load():
    global cpu_load, load_thread
    if not cpu_load:
        cpu_load = True
        load_thread = threading.Thread(target=cpu_intensive_task)
        load_thread.daemon = True
        load_thread.start()
        return jsonify({'status': 'CPU load started'})
    return jsonify({'status': 'CPU load already running'})

@app.route('/stop-load')
def stop_load():
    global cpu_load
    if cpu_load:
        cpu_load = False
        return jsonify({'status': 'CPU load stopped'})
    return jsonify({'status': 'No CPU load was running'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

# 创建requirements.txt
cat > requirements.txt << 'EOF'
flask==2.3.2
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

# 构建Docker镜像
docker build -t scaling-demo:v1 .
minikube image load scaling-demo:v1
```

### 2. 部署带有探针的应用

创建一个包含存活探针和就绪探针的Deployment：

```bash
# 创建Deployment配置
cat > probe-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: scaling-demo
  labels:
    app: scaling-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: scaling-demo
  template:
    metadata:
      labels:
        app: scaling-demo
    spec:
      containers:
      - name: scaling-demo
        image: scaling-demo:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 2
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 3
EOF

# 应用配置
kubectl apply -f probe-deployment.yaml

# 创建service以访问应用
cat > demo-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: scaling-demo
spec:
  selector:
    app: scaling-demo
  ports:
  - port: 80
    targetPort: 5000
  type: NodePort
EOF

kubectl apply -f demo-service.yaml

# 检查deployment状态
kubectl get deployments
kubectl get pods
```

### 3. 测试存活探针触发自愈

现在我们来测试当存活探针失败时，Kubernetes的自愈机制：

```bash
# 获取服务URL
minikube service scaling-demo --url

# 获取POD名称
POD_NAME=$(kubectl get pods -l app=scaling-demo -o jsonpath='{.items[0].metadata.name}')
echo $POD_NAME
```

使用浏览器或curl访问服务URL，确认应用正常运行。然后触发健康检查失败：

```bash
# 使用命令行触发健康检查失败
SERVICE_URL=$(minikube service scaling-demo --url)
curl $SERVICE_URL/fail

# 观察Pod状态变化
kubectl get pods -w
```

你应该看到Kubernetes检测到容器不健康，并自动重启容器。

### 4. 测试就绪探针

现在测试就绪探针的行为：

```bash
# 触发就绪检查失败
curl $SERVICE_URL/not-ready

# 观察Pod状态变化
kubectl get pods
```

此时Pod应该仍在运行，但处于未就绪状态，不会接收服务的流量。通过以下命令检查：

```bash
# 查看Pod详情
kubectl describe pod $POD_NAME
```

在输出中，你应该能看到容器的就绪状态变为`False`。

恢复就绪状态：

```bash
curl $SERVICE_URL/ready-again

# 再次检查Pod状态
kubectl get pods
```

### 5. 配置Horizontal Pod Autoscaler (HPA)

首先，确保Minikube启用了指标服务器：

```bash
# 启用metrics-server插件
minikube addons enable metrics-server

# 等待metrics-server部署完成
kubectl get deployment metrics-server -n kube-system
```

创建HPA配置：

```bash
# 创建HPA配置文件
cat > demo-hpa.yaml << 'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: scaling-demo-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: scaling-demo
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
EOF

# 应用HPA配置
kubectl apply -f demo-hpa.yaml

# 检查HPA状态
kubectl get hpa
```

### 6. 测试自动扩缩容

生成CPU负载以触发自动扩缩容：

```bash
# 获取服务URL
SERVICE_URL=$(minikube service scaling-demo --url)

# 触发CPU负载
curl $SERVICE_URL/start-load

# 在另一个终端监控HPA状态
kubectl get hpa -w

# 同时监控Pod数量
kubectl get pods -w
```

应该能观察到CPU使用率上升，并且HPA开始增加Pod副本数。等待一段时间后，停止负载：

```bash
curl $SERVICE_URL/stop-load
```

继续观察HPA状态，副本数应该会在稳定期后慢慢减少回最小值。

### 7. 配置重启策略和探测失败处理

创建一个具有不同重启策略的Pod：

```bash
# 创建Pod配置
cat > restart-policy-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: restart-demo
  labels:
    app: restart-demo
spec:
  restartPolicy: OnFailure
  containers:
  - name: scaling-demo
    image: scaling-demo:v1
    imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 5000
    livenessProbe:
      httpGet:
        path: /health
        port: 5000
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 1
EOF

# 应用配置
kubectl apply -f restart-policy-pod.yaml

# 查看Pod状态
kubectl get pods
```

### 8. 研究和对比不同的就绪/存活探针类型

修改我们的Deployment，使用不同类型的探针：

```bash
# 创建使用TCP探针的配置
cat > tcp-probe-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tcp-probe-demo
  labels:
    app: tcp-probe-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tcp-probe-demo
  template:
    metadata:
      labels:
        app: tcp-probe-demo
    spec:
      containers:
      - name: scaling-demo
        image: scaling-demo:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
        livenessProbe:
          tcpSocket:
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 5
EOF

# 应用TCP探针配置
kubectl apply -f tcp-probe-deployment.yaml

# 创建使用Exec探针的配置
cat > exec-probe-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: exec-probe-demo
  labels:
    app: exec-probe-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: exec-probe-demo
  template:
    metadata:
      labels:
        app: exec-probe-demo
    spec:
      containers:
      - name: scaling-demo
        image: scaling-demo:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
        livenessProbe:
          exec:
            command:
            - curl
            - -f
            - http://localhost:5000/health
          initialDelaySeconds: 10
          periodSeconds: 5
EOF

# 应用Exec探针配置
kubectl apply -f exec-probe-deployment.yaml

# 查看所有Pod状态
kubectl get pods
```

### 9. 清理资源

完成实验后，清理创建的资源：

```bash
# 删除所有资源
kubectl delete deployment scaling-demo tcp-probe-demo exec-probe-demo
kubectl delete service scaling-demo
kubectl delete pod restart-demo
kubectl delete hpa scaling-demo-hpa
```

## 思考问题

1. Kubernetes的自愈机制是如何工作的？控制器在其中扮演什么角色？
2. 存活探针和就绪探针之间有什么区别？各自解决什么问题？
3. 不同类型的探针（HTTP、TCP、Exec）各有什么优缺点和适用场景？
4. HPA如何根据CPU/内存使用率自动扩展Pod数量？这与手动扩展相比有什么优势？
5. 在实际生产环境中，如何权衡探针的参数设置（如检查间隔、超时时间、失败阈值等）？

## 扩展练习

1. 配置基于自定义指标的HPA（例如，基于请求数量或队列长度）
2. 实现启动探针（Startup Probe）来处理启动缓慢的应用
3. 尝试使用Vertical Pod Autoscaler（VPA）调整容器资源配置
4. 模拟节点故障，观察Kubernetes如何将Pod重新调度到健康节点上
5. 探索Pod中断预算（PodDisruptionBudget）如何在集群维护期间保证服务可用性 