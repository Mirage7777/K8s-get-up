# 实验2.4：Kubernetes服务暴露与网络

## 目标
理解Kubernetes中不同类型的服务（ClusterIP、NodePort、LoadBalancer）及其用途，学习如何在集群内外公开应用，掌握Kubernetes网络模型和服务发现机制。

## 准备工作
- 完成实验2.1、2.2和2.3
- 运行中的Minikube集群
- 熟悉基本的kubectl命令

## Kubernetes服务和网络概述

Kubernetes提供了多种将应用暴露给内部或外部用户的方式：

1. **Service**：抽象，定义了Pod的逻辑集和访问策略
   - **ClusterIP**：默认类型，仅在集群内部可访问
   - **NodePort**：通过节点端口（通常是30000-32767范围）公开服务
   - **LoadBalancer**：使用云提供商的负载均衡器公开服务
   - **ExternalName**：将服务映射到DNS名称

2. **Ingress**：管理集群外部对服务的访问，通常是HTTP/HTTPS
3. **网络策略（NetworkPolicy）**：定义Pod间通信规则

## 步骤

### 1. 准备测试应用

我们将创建多个后端服务，以便测试不同的服务类型和网络功能：

```bash
# 创建项目目录
mkdir -p k8s-services-demo
cd k8s-services-demo

# 创建后端服务1
cat > backend-v1.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-v1
  labels:
    app: backend
    version: v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
      version: v1
  template:
    metadata:
      labels:
        app: backend
        version: v1
    spec:
      containers:
      - name: backend
        image: hashicorp/http-echo:latest
        args:
        - "-text=Backend Version 1"
        ports:
        - containerPort: 5678
EOF

# 创建后端服务2
cat > backend-v2.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-v2
  labels:
    app: backend
    version: v2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
      version: v2
  template:
    metadata:
      labels:
        app: backend
        version: v2
    spec:
      containers:
      - name: backend
        image: hashicorp/http-echo:latest
        args:
        - "-text=Backend Version 2"
        ports:
        - containerPort: 5678
EOF

# 创建前端服务
cat > frontend.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: default.conf
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
EOF

# 创建Nginx配置
cat > nginx-config.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  default.conf: |
    server {
        listen 80;
        server_name _;
        
        location /v1/ {
            proxy_pass http://backend-v1:5678/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location /v2/ {
            proxy_pass http://backend-v2:5678/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location / {
            return 200 'Frontend Service\n';
        }
    }
EOF

# 应用配置
kubectl apply -f nginx-config.yaml
kubectl apply -f backend-v1.yaml
kubectl apply -f backend-v2.yaml
kubectl apply -f frontend.yaml

# 检查Pod状态
kubectl get pods
```

### 2. 创建ClusterIP服务（集群内部访问）

ClusterIP是默认的服务类型，仅在集群内部可访问：

```bash
# 为后端服务创建ClusterIP服务
cat > backend-services.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: backend-v1
spec:
  selector:
    app: backend
    version: v1
  ports:
  - port: 5678
    targetPort: 5678
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: backend-v2
spec:
  selector:
    app: backend
    version: v2
  ports:
  - port: 5678
    targetPort: 5678
  type: ClusterIP
EOF

kubectl apply -f backend-services.yaml
```

### 3. 创建NodePort服务（从外部访问）

NodePort服务通过节点端口公开服务：

```bash
# 为前端服务创建NodePort服务
cat > frontend-nodeport.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: frontend-nodeport
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
EOF

kubectl apply -f frontend-nodeport.yaml

# 获取访问URL
minikube service frontend-nodeport --url
```

使用返回的URL在浏览器中访问前端服务，测试不同的路径：
- `/` - 应该显示"Frontend Service"
- `/v1/` - 应该显示"Backend Version 1"
- `/v2/` - 应该显示"Backend Version 2"

### 4. 使用环境变量和DNS进行服务发现

Kubernetes自动为每个服务提供环境变量和DNS条目。让我们探索这些功能：

```bash
# 获取一个前端Pod的名称
FRONTEND_POD=$(kubectl get pod -l app=frontend -o jsonpath='{.items[0].metadata.name}')

# 查看环境变量中的服务信息
kubectl exec -it $FRONTEND_POD -- env | grep SERVICE

# 使用DNS解析服务
kubectl exec -it $FRONTEND_POD -- nslookup backend-v1
kubectl exec -it $FRONTEND_POD -- nslookup backend-v2

# 验证服务连接
kubectl exec -it $FRONTEND_POD -- wget -qO- http://backend-v1:5678
kubectl exec -it $FRONTEND_POD -- wget -qO- http://backend-v2:5678
```

### 5. 创建LoadBalancer服务

LoadBalancer服务类型在云环境中会创建外部负载均衡器。在Minikube中，我们可以使用隧道模式模拟LoadBalancer：

```bash
# 创建LoadBalancer服务
cat > frontend-loadbalancer.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: frontend-lb
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
EOF

kubectl apply -f frontend-loadbalancer.yaml

# 在另一个终端启动隧道
# minikube tunnel

# 查看服务状态
kubectl get service frontend-lb
```

### 6. 设置Ingress控制器

Ingress提供了比服务更强大的路由和负载均衡功能：

```bash
# 启用Ingress插件
minikube addons enable ingress

# 检查Ingress控制器是否运行
kubectl get pods -n ingress-nginx

# 创建Ingress资源
cat > frontend-ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
  - host: services-demo.info
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-nodeport
            port:
              number: 80
EOF

kubectl apply -f frontend-ingress.yaml

# 获取Ingress地址
kubectl get ingress
```

更新本地hosts文件以使用Ingress：

```bash
# 获取Minikube IP地址
MINIKUBE_IP=$(minikube ip)
echo "$MINIKUBE_IP services-demo.info" | sudo tee -a /etc/hosts
```

现在可以通过http://services-demo.info访问应用。

### 7. 配置不同后端服务的精确路由

我们可以创建一个更复杂的Ingress，直接路由到不同的后端服务：

```bash
# 创建端口转发的后端服务
cat > backend-services-nodeport.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: backend-v1-np
spec:
  selector:
    app: backend
    version: v1
  ports:
  - port: 5678
    targetPort: 5678
    nodePort: 30001
  type: NodePort
---
apiVersion: v1
kind: Service
metadata:
  name: backend-v2-np
spec:
  selector:
    app: backend
    version: v2
  ports:
  - port: 5678
    targetPort: 5678
    nodePort: 30002
  type: NodePort
EOF

kubectl apply -f backend-services-nodeport.yaml

# 创建直接访问后端的Ingress
cat > backend-ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend-ingress
spec:
  rules:
  - host: services-demo.info
    http:
      paths:
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: backend-v1
            port:
              number: 5678
      - path: /api/v2
        pathType: Prefix
        backend:
          service:
            name: backend-v2
            port:
              number: 5678
EOF

kubectl apply -f backend-ingress.yaml
```

### 8. 探索服务内部工作原理

```bash
# 查看服务的端点（Endpoints）
kubectl get endpoints

# 查看服务详情
kubectl describe service backend-v1
kubectl describe service frontend-nodeport

# 检查kube-proxy创建的iptables规则（如果使用iptables模式）
kubectl get pods -n kube-system | grep kube-proxy
```

### 9. 创建和测试无头服务（Headless Service）

无头服务用于需要直接访问每个Pod的场景，如有状态应用：

```bash
# 创建无头服务
cat > headless-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: backend-headless
spec:
  selector:
    app: backend
  ports:
  - port: 5678
    targetPort: 5678
  clusterIP: None
EOF

kubectl apply -f headless-service.yaml

# 测试无头服务的DNS解析
kubectl run -it --rm debug --image=alpine --restart=Never -- sh -c "nslookup backend-headless"
```

### 10. 实施基本的网络策略

网络策略用于控制Pod之间的通信：

```bash
# 创建默认拒绝的网络策略
cat > default-deny.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF

# 允许前端访问后端的网络策略
cat > frontend-to-backend.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
EOF

# 如果Minikube启用了CNI插件，可以应用这些策略
# kubectl apply -f default-deny.yaml
# kubectl apply -f frontend-to-backend.yaml
```

注意：网络策略需要网络插件支持，不是所有Minikube配置都支持。

### 11. 清理资源

```bash
# 删除创建的所有资源
kubectl delete service frontend-nodeport frontend-lb backend-v1 backend-v2 backend-v1-np backend-v2-np backend-headless
kubectl delete deployment frontend backend-v1 backend-v2
kubectl delete configmap nginx-config
kubectl delete ingress frontend-ingress backend-ingress
# kubectl delete networkpolicy default-deny frontend-to-backend
```

## 思考问题

1. 不同类型的服务（ClusterIP、NodePort、LoadBalancer、ExternalName）各有什么优缺点？在什么场景下使用每种类型更合适？
2. Kubernetes如何使用服务和DNS实现服务发现？这对微服务架构有什么意义？
3. Ingress和Service的关系是什么？为什么需要Ingress？
4. 无头服务（Headless Service）的使用场景是什么？它与标准ClusterIP服务有什么区别？
5. 网络策略如何帮助保护微服务应用的安全？

## 扩展练习

1. 配置服务的会话亲和性（Session Affinity）
2. 实现基于路径的Ingress路由，同时支持TLS终止
3. 使用外部DNS服务器和ExternalName服务连接集群外部资源
4. 在多节点集群中，比较使用kube-proxy的不同模式（userspace、iptables、IPVS）的性能差异
5. 配置网络策略，仅允许特定命名空间的Pod访问应用，阻止其他所有访问 