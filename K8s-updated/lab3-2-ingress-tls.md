# 实验3.2：Ingress控制器与TLS终止

## 目标
学习如何在Kubernetes中设置Ingress控制器，配置基于路径和主机的路由规则，实现TLS终止以保护Web应用程序流量，并了解Ingress的高级特性。

## 准备工作
- 完成实验2.1至2.5和3.1
- 运行中的Minikube集群
- 熟悉基本的kubectl命令
- 了解HTTPS/TLS基础知识

## Ingress概述

Kubernetes Ingress是一种API对象，管理集群外部访问集群内服务的规则，通常是HTTP/HTTPS流量。
它提供了以下功能：
- 负载均衡
- SSL/TLS终止
- 基于名称的虚拟主机
- 路径路由

## 步骤

### 1. 启用Minikube Ingress插件

```bash
# 启用Ingress控制器插件
minikube addons enable ingress

# 验证Ingress控制器Pod是否运行
kubectl get pods -n ingress-nginx

# 查看Ingress控制器服务
kubectl get svc -n ingress-nginx
```

### 2. 部署多个后端服务

首先，我们需要创建多个服务作为我们的Ingress规则目标：

```bash
# 创建第一个示例应用：蓝色应用
cat > blue-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blue-deployment
  labels:
    app: blue
spec:
  replicas: 2
  selector:
    matchLabels:
      app: blue
  template:
    metadata:
      labels:
        app: blue
    spec:
      containers:
      - name: blue-app
        image: hashicorp/http-echo:latest
        args:
        - "-text=blue"
        - "-listen=:8080"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: blue-service
spec:
  selector:
    app: blue
  ports:
  - port: 80
    targetPort: 8080
EOF

kubectl apply -f blue-deployment.yaml

# 创建第二个示例应用：绿色应用
cat > green-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: green-deployment
  labels:
    app: green
spec:
  replicas: 2
  selector:
    matchLabels:
      app: green
  template:
    metadata:
      labels:
        app: green
    spec:
      containers:
      - name: green-app
        image: hashicorp/http-echo:latest
        args:
        - "-text=green"
        - "-listen=:8080"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: green-service
spec:
  selector:
    app: green
  ports:
  - port: 80
    targetPort: 8080
EOF

kubectl apply -f green-deployment.yaml

# 创建默认后端应用
cat > default-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: default-deployment
  labels:
    app: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: default
  template:
    metadata:
      labels:
        app: default
    spec:
      containers:
      - name: default-app
        image: hashicorp/http-echo:latest
        args:
        - "-text=default backend"
        - "-listen=:8080"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: default-service
spec:
  selector:
    app: default
  ports:
  - port: 80
    targetPort: 8080
EOF

kubectl apply -f default-deployment.yaml

# 验证所有服务是否运行
kubectl get deployments
kubectl get services
```

### 3. 创建基本的Ingress资源

```bash
# 创建基于路径的Ingress
cat > path-ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /blue
        pathType: Prefix
        backend:
          service:
            name: blue-service
            port:
              number: 80
      - path: /green
        pathType: Prefix
        backend:
          service:
            name: green-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: default-service
            port:
              number: 80
EOF

kubectl apply -f path-ingress.yaml

# 获取Ingress地址
kubectl get ingress path-ingress
```

获取Minikube IP地址，以访问Ingress：

```bash
minikube ip
```

### 4. 测试路径路由

使用curl测试不同路径的路由：

```bash
# 设置MINIKUBE_IP环境变量
export MINIKUBE_IP=$(minikube ip)

# 测试默认后端
curl $MINIKUBE_IP

# 测试蓝色后端
curl $MINIKUBE_IP/blue

# 测试绿色后端
curl $MINIKUBE_IP/green
```

### 5. 创建基于主机的虚拟主机

```bash
# 创建基于主机的Ingress
cat > host-ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
spec:
  rules:
  - host: blue.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: blue-service
            port:
              number: 80
  - host: green.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: green-service
            port:
              number: 80
EOF

kubectl apply -f host-ingress.yaml
```

### 6. 测试基于主机的路由

修改本地hosts文件以使用自定义域名（需要管理员/root权限）：

```bash
# Linux/Mac: 在/etc/hosts文件中添加
echo "$MINIKUBE_IP blue.example.com green.example.com" | sudo tee -a /etc/hosts

# 测试基于主机的路由
curl http://blue.example.com
curl http://green.example.com
```

### 7. 生成自签名TLS证书

为了配置HTTPS，我们需要创建TLS证书：

```bash
# 创建私钥和自签名证书
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout tls.key -out tls.crt \
    -subj "/CN=*.example.com" \
    -addext "subjectAltName = DNS:*.example.com,DNS:example.com"

# 检查生成的证书
openssl x509 -in tls.crt -text -noout | grep -E "Subject:|DNS:"

# 创建Kubernetes Secret来存储TLS证书
kubectl create secret tls example-tls --key tls.key --cert tls.crt
```

### 8. 配置TLS终止的Ingress

```bash
# 创建启用TLS的Ingress
cat > tls-ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - blue.example.com
    - green.example.com
    secretName: example-tls
  rules:
  - host: blue.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: blue-service
            port:
              number: 80
  - host: green.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: green-service
            port:
              number: 80
EOF

kubectl apply -f tls-ingress.yaml
```

### 9. 测试HTTPS访问

```bash
# 使用curl测试HTTPS连接（忽略证书验证，因为是自签名的）
curl -k https://blue.example.com
curl -k https://green.example.com
```

### 10. 配置Ingress高级功能：会话亲和性和速率限制

```bash
# 创建带有会话亲和性和速率限制的Ingress
cat > advanced-ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: advanced-ingress
  annotations:
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "INGRESSCOOKIE"
    nginx.ingress.kubernetes.io/session-cookie-expires: "172800"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "172800"
    
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-connections: "5"
    
    # 为了演示，也启用HTTPS重定向
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - advanced.example.com
    secretName: example-tls
  rules:
  - host: advanced.example.com
    http:
      paths:
      - path: /blue
        pathType: Prefix
        backend:
          service:
            name: blue-service
            port:
              number: 80
      - path: /green
        pathType: Prefix
        backend:
          service:
            name: green-service
            port:
              number: 80
EOF

kubectl apply -f advanced-ingress.yaml

# 更新hosts文件
echo "$MINIKUBE_IP advanced.example.com" | sudo tee -a /etc/hosts
```

### 11. 测试会话亲和性

```bash
# 多次请求应该返回相同的后端（查看cookie）
curl -k -v https://advanced.example.com/blue
```

观察响应头中的`Set-Cookie`，它应该包含`INGRESSCOOKIE`。

### 12. 使用反向代理实现更复杂的流量控制

```bash
# 部署另一个应用，用于演示更复杂的代理规则
cat > api-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
  labels:
    app: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo:latest
        args:
        - "-text=api service"
        - "-listen=:8080"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080
EOF

kubectl apply -f api-deployment.yaml

# 创建具有自定义代理配置的Ingress
cat > proxy-ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: proxy-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_set_header Host $http_host;
      
      # 添加自定义响应头
      add_header X-Ingress-Controller "NGINX";
      add_header X-App-Version "1.0.0";
    
    # 压缩响应
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, PUT, POST, DELETE, PATCH, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
EOF

kubectl apply -f proxy-ingress.yaml

# 更新hosts文件
echo "$MINIKUBE_IP api.example.com" | sudo tee -a /etc/hosts
```

### 13. 测试复杂代理规则

```bash
# 测试API访问
curl -v http://api.example.com/api/

# 查看自定义响应头
curl -I http://api.example.com/api/
```

### 14. 监控Ingress控制器

```bash
# 查看Ingress控制器的日志
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# 查看Ingress控制器的性能指标
kubectl get pods -n ingress-nginx
POD_NAME=$(kubectl get pods -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n ingress-nginx $POD_NAME 9090:9090

# 在另一个终端中访问指标
curl http://localhost:9090/metrics
```

### 15. 清理资源

```bash
# 删除所有创建的资源
kubectl delete ingress path-ingress host-ingress tls-ingress advanced-ingress proxy-ingress
kubectl delete service blue-service green-service default-service api-service
kubectl delete deployment blue-deployment green-deployment default-deployment api-deployment
kubectl delete secret example-tls

# 清理生成的证书文件
rm tls.key tls.crt
```

## 思考问题

1. Ingress与常规的Kubernetes Service (NodePort/LoadBalancer)相比有什么优势？它在什么场景下更为适用？
2. TLS终止在Ingress控制器上与在应用容器内部进行各有什么优缺点？
3. 在多租户环境中，如何使用Ingress实现不同团队/服务的隔离？
4. Ingress控制器的高可用性如何保障？在生产环境中应如何部署？
5. 不同的Ingress控制器实现（NGINX、Traefik、HAProxy等）各有什么特点和适用场景？

## 扩展练习

1. 配置Ingress使用Let's Encrypt自动获取和更新TLS证书
2. 使用Ingress实现蓝绿部署或金丝雀发布策略
3. 配置OAuth或基本认证来保护Ingress后端服务
4. 实现API网关模式，使用Ingress将请求路由到不同的微服务
5. 部署多个Ingress控制器，并将不同的服务路由到不同的控制器 