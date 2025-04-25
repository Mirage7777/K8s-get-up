# 实验3.6：Kubernetes故障排查与性能优化

## 目标
掌握Kubernetes环境中的故障排查技术和常见问题解决方法，学习如何识别和解决Pod、节点、网络和资源相关的问题，以及如何优化Kubernetes应用的性能。

## 准备工作
- 完成实验3.1至3.5
- 运行中的Minikube集群
- 熟悉基本的kubectl命令
- 基本的Linux命令行和系统排查知识

## Kubernetes故障排查概述

Kubernetes系统的故障可能出现在多个层次：
- **Pod级别**：容器无法启动、崩溃循环、健康检查失败
- **节点级别**：资源耗尽、节点不可用、kubelet问题
- **集群级别**：控制平面组件失效、DNS问题、网络连接问题
- **应用级别**：性能问题、资源配置不当、应用特定错误

本实验将涵盖各层次的常见问题和排查方法。

## 步骤

### 1. 设置故障诊断环境

首先创建一个专用的命名空间，并部署一些示例应用用于故障排查：

```bash
# 创建命名空间
kubectl create namespace troubleshooting

# 设置当前上下文使用此命名空间
kubectl config set-context --current --namespace=troubleshooting
```

### 2. Pod故障排查 - 镜像错误

```bash
# 创建一个使用错误镜像的Pod
cat > pod-wrong-image.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-wrong-image
spec:
  containers:
  - name: nginx
    image: nginx:nonexistent-tag
EOF

kubectl apply -f pod-wrong-image.yaml

# 查看Pod状态
kubectl get pod pod-wrong-image

# 查看详细信息以诊断问题
kubectl describe pod pod-wrong-image

# 查看Pod事件
kubectl get events --field-selector involvedObject.name=pod-wrong-image
```

### 3. Pod故障排查 - 资源限制问题

```bash
# 创建一个资源限制设置不当的Pod
cat > pod-resource-limit.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-resource-limit
spec:
  containers:
  - name: memory-hog
    image: polinux/stress
    resources:
      limits:
        memory: "100Mi"
      requests:
        memory: "50Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
EOF

kubectl apply -f pod-resource-limit.yaml

# 观察Pod状态和OOMKilled事件
kubectl get pod pod-resource-limit -w

# 几秒后按Ctrl+C停止观察

# 查看Pod详细信息
kubectl describe pod pod-resource-limit

# 查看Pod日志（可能看不到，因为容器会被OOMKilled）
kubectl logs pod-resource-limit || echo "Container terminated due to OOMKilled"
```

### 4. Pod故障排查 - 健康检查配置问题

```bash
# 创建一个健康检查配置有问题的Pod
cat > pod-health-check.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-health-check
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
    readinessProbe:
      httpGet:
        path: /nonexistent
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
EOF

kubectl apply -f pod-health-check.yaml

# 查看Pod状态
kubectl get pod pod-health-check -w

# 几秒后按Ctrl+C停止观察

# 查看详细信息
kubectl describe pod pod-health-check
```

### 5. Pod故障排查 - 配置和环境问题

```bash
# 创建一个依赖环境变量但配置不当的Pod
cat > pod-env-config.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-env-config
spec:
  containers:
  - name: busybox
    image: busybox:1.28
    command: ['sh', '-c', 'echo "Database URL: $DB_URL"; if [ -z "$DB_URL" ]; then echo "Error: DB_URL not set"; exit 1; fi; sleep 3600']
    # Missing environment variables
EOF

kubectl apply -f pod-env-config.yaml

# 查看Pod状态
kubectl get pod pod-env-config

# 查看Pod日志
kubectl logs pod-env-config

# 修复问题 - 更新Pod定义添加环境变量
cat > pod-env-config-fixed.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-env-config-fixed
spec:
  containers:
  - name: busybox
    image: busybox:1.28
    command: ['sh', '-c', 'echo "Database URL: $DB_URL"; if [ -z "$DB_URL" ]; then echo "Error: DB_URL not set"; exit 1; fi; sleep 3600']
    env:
    - name: DB_URL
      value: "mysql://user:password@mysql-service:3306/db"
EOF

kubectl apply -f pod-env-config-fixed.yaml

# 查看修复后的Pod日志
kubectl logs pod-env-config-fixed
```

### 6. Pod故障排查 - 应用崩溃循环

```bash
# 创建一个会崩溃循环的Pod
cat > pod-crash-loop.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-crash-loop
spec:
  containers:
  - name: crasher
    image: busybox:1.28
    command: ['sh', '-c', 'sleep 10; echo "Simulating application crash"; exit 1']
EOF

kubectl apply -f pod-crash-loop.yaml

# 观察Pod状态变为CrashLoopBackOff
kubectl get pod pod-crash-loop -w

# 几次循环后按Ctrl+C停止观察

# 查看Pod日志
kubectl logs pod-crash-loop

# 查看前一个失败容器的日志（如果有多次重启）
kubectl logs pod-crash-loop --previous
```

### 7. 网络排查 - 服务连接问题

```bash
# 创建后端服务
cat > backend-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
EOF

kubectl apply -f backend-app.yaml

# 创建依赖后端服务的前端应用
cat > frontend-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: busybox
        image: busybox:1.28
        command: ['sh', '-c', 'while true; do wget -T 2 -q -O- backend-service.wrong-namespace || echo "Failed to connect to backend"; sleep 5; done']
EOF

kubectl apply -f frontend-app.yaml

# 查看前端Pod日志，观察连接问题
sleep 5
kubectl logs -l app=frontend

# 修复连接问题 - 更新前端应用使用正确的服务名称
cat > frontend-app-fixed.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-fixed
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend-fixed
  template:
    metadata:
      labels:
        app: frontend-fixed
    spec:
      containers:
      - name: busybox
        image: busybox:1.28
        command: ['sh', '-c', 'while true; do wget -T 2 -q -O- backend-service || echo "Connected to backend"; sleep 5; done']
EOF

kubectl apply -f frontend-app-fixed.yaml

# 查看修复后的日志
sleep 5
kubectl logs -l app=frontend-fixed
```

### 8. 网络排查 - DNS问题

```bash
# 创建DNS排查工具Pod
cat > dns-debug.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: dns-debug
spec:
  containers:
  - name: dnsutils
    image: registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3
    command:
      - sleep
      - "3600"
EOF

kubectl apply -f dns-debug.yaml

# 等待Pod运行
kubectl wait --for=condition=Ready pod/dns-debug

# 检查DNS解析
kubectl exec -it dns-debug -- nslookup kubernetes.default

# 检查CoreDNS服务本身
kubectl exec -it dns-debug -- nslookup kube-dns.kube-system.svc.cluster.local

# 检查不存在的服务
kubectl exec -it dns-debug -- nslookup nonexistent-service || echo "DNS resolution for nonexistent service failed as expected"

# 查看resolv.conf配置
kubectl exec -it dns-debug -- cat /etc/resolv.conf
```

### 9. 存储排查 - 持久卷问题

```bash
# 创建StorageClass
cat > local-storage.yaml << 'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: k8s.io/minikube-hostpath
EOF

kubectl apply -f local-storage.yaml

# 创建PVC
cat > storage-pvc.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: storage-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-storage
  resources:
    requests:
      storage: 1Gi
EOF

kubectl apply -f storage-pvc.yaml

# 创建使用PVC的Pod
cat > storage-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: storage-pod
spec:
  containers:
  - name: busybox
    image: busybox:1.28
    command: ['sh', '-c', 'echo "Testing storage" > /data/test.txt; sleep 3600']
    volumeMounts:
    - name: data-volume
      mountPath: /data
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: storage-pvc
EOF

kubectl apply -f storage-pod.yaml

# 检查PVC状态
kubectl get pvc storage-pvc

# 检查Pod状态
kubectl get pod storage-pod

# 验证数据写入
kubectl exec -it storage-pod -- cat /data/test.txt

# 模拟持久卷问题 - 创建引用不存在PVC的Pod
cat > storage-problem.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: storage-problem
spec:
  containers:
  - name: busybox
    image: busybox:1.28
    command: ['sh', '-c', 'sleep 3600']
    volumeMounts:
    - name: data-volume
      mountPath: /data
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: nonexistent-pvc
EOF

kubectl apply -f storage-problem.yaml

# 检查问题Pod状态
kubectl get pod storage-problem

# 查看详细信息
kubectl describe pod storage-problem
```

### 10. 资源和性能排查 - 资源使用情况

```bash
# 部署一个资源使用监控Pod
cat > resource-monitor.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: resource-monitor
spec:
  containers:
  - name: stress
    image: polinux/stress
    resources:
      requests:
        memory: "50Mi"
        cpu: "100m"
      limits:
        memory: "100Mi"
        cpu: "200m"
    command: ["stress"]
    args: ["--cpu", "1", "--vm", "1", "--vm-bytes", "50M"]
EOF

kubectl apply -f resource-monitor.yaml

# 等待Pod运行
kubectl wait --for=condition=Ready pod/resource-monitor

# 查看Pod的资源使用情况
kubectl top pod resource-monitor

# 如果 kubectl top 命令不可用，需要启用metrics-server
# minikube addons enable metrics-server

# 查看节点资源使用情况
kubectl top node
```

### 11. 控制平面排查

在Minikube环境中，控制平面组件运行在单个节点上：

```bash
# 检查控制平面组件状态
kubectl get pods -n kube-system

# 检查API服务器日志
minikube ssh "sudo journalctl -u kubelet | grep apiserver | tail -30"

# 检查etcd状态
minikube ssh "sudo crictl ps | grep etcd"

# 检查kubelet状态
minikube ssh "sudo systemctl status kubelet"
```

### 12. 使用kubectl排查命令

Kubectl提供了多种用于排查问题的命令：

```bash
# 查看资源详情
kubectl describe pod resource-monitor

# 查看日志
kubectl logs resource-monitor

# 实时查看日志
kubectl logs -f resource-monitor

# 执行命令
kubectl exec -it resource-monitor -- sh -c "ps aux"

# 检查集群信息
kubectl cluster-info

# 检查集群状态
kubectl get componentstatuses

# 转发端口
kubectl port-forward pod/resource-monitor 8080:80
```

### 13. 常见问题排查场景 - 应用无响应

```bash
# 部署一个可能无响应的Web应用
cat > unresponsive-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: unresponsive-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: unresponsive
  template:
    metadata:
      labels:
        app: unresponsive
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: unresponsive-service
spec:
  selector:
    app: unresponsive
  ports:
  - port: 80
    targetPort: 80
EOF

kubectl apply -f unresponsive-app.yaml

# 排查步骤
# 1. 检查Pod状态
kubectl get pods -l app=unresponsive

# 2. 检查服务状态
kubectl get service unresponsive-service

# 3. 检查Pod和服务的端点
kubectl describe service unresponsive-service

# 4. 从集群内部测试服务
kubectl run test-client --image=busybox:1.28 --rm -it --restart=Never -- wget -T 2 -O- unresponsive-service

# 5. 检查容器日志
kubectl logs -l app=unresponsive

# 6. 检查容器内部进程
kubectl exec -it $(kubectl get pod -l app=unresponsive -o jsonpath='{.items[0].metadata.name}') -- ps aux
```

### 14. 常见问题排查场景 - Deployment未创建预期的Pod数量

```bash
# 创建一个资源不足的节点可能无法满足的Deployment
cat > large-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: large-deployment
spec:
  replicas: 10
  selector:
    matchLabels:
      app: large-app
  template:
    metadata:
      labels:
        app: large-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        resources:
          requests:
            memory: "256Mi"
            cpu: "500m"
          limits:
            memory: "512Mi"
            cpu: "1000m"
EOF

kubectl apply -f large-deployment.yaml

# 检查Deployment状态
kubectl get deployment large-deployment

# 检查ReplicaSet
kubectl get rs -l app=large-app

# 检查Pod状态
kubectl get pods -l app=large-app

# 检查Pod调度情况
kubectl describe pod -l app=large-app | grep -A 10 "Events:"

# 检查节点资源情况
kubectl describe nodes
```

### 15. 命令行调试技巧

```bash
# 使用标签选择器过滤资源
kubectl get pods -l app=large-app

# 使用JSONPath提取特定信息
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# 使用custom-columns格式化输出
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName

# 使用--field-selector过滤资源
kubectl get pods --field-selector=status.phase=Running

# 使用explain了解资源字段
kubectl explain pod.spec.containers

# 使用--watch实时监控资源变化
kubectl get pods --watch
```

### 16. 性能优化策略

```bash
# 示例：为应用配置资源请求和限制
cat > optimized-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: optimized-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: optimized
  template:
    metadata:
      labels:
        app: optimized
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
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
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
EOF

kubectl apply -f optimized-app.yaml

# 验证部署
kubectl rollout status deployment/optimized-app
```

### 17. 清理资源

```bash
# 删除命名空间及其中的所有资源
kubectl delete namespace troubleshooting

# 恢复默认命名空间
kubectl config set-context --current --namespace=default
```

## 思考问题

1. 如何系统地排查Pod无法启动的问题？哪些是最常见的Pod启动失败原因？
2. 在Kubernetes集群中，网络问题通常表现为哪些症状？如何逐步排查这些问题？
3. 如何确定应用程序的适当资源请求和限制值？设置不当会导致什么问题？
4. 在排查Kubernetes问题时，如何有效地利用日志和事件信息？
5. 为什么了解Kubernetes控制平面组件如何工作对故障排查很重要？

## 扩展练习

1. 设计并实现一个综合故障场景，包含多种类型的错误（存储、网络、配置等）
2. 创建一个小型监控系统来跟踪集群中的资源使用情况和性能指标
3. 实施自动化故障检测和恢复策略（例如，使用健康检查、重启策略等）
4. 研究和实施Kubernetes中的调度高级特性（节点亲和性、污点和容忍等）
5. 探索Kubernetes外部工具如Helm、Kustomize等如何有助于减少配置错误 