# 实验3.5：RBAC和Kubernetes安全

## 目标
理解Kubernetes中的基于角色的访问控制(RBAC)和安全机制，学习如何创建和管理用户、角色、角色绑定以及如何实施最佳安全实践。

## 准备工作
- 完成实验3.1至3.4
- 运行中的Minikube集群
- 熟悉基本的kubectl命令
- 基本的Linux命令行知识

## Kubernetes安全概述

Kubernetes安全架构围绕以下几个关键方面：
- **认证(Authentication)**: 确认用户身份
- **授权(Authorization)**: 基于用户身份确定允许的操作
- **准入控制(Admission Control)**: 拦截API请求并可能修改或拒绝它们
- **网络策略(Network Policies)**: 控制Pod之间的通信
- **安全上下文(Security Context)**: 定义Pod和容器的特权和访问控制设置

本实验将重点关注RBAC授权机制和相关安全配置。

## 步骤

### 1. 了解Kubernetes中的RBAC组件

RBAC基于四个关键对象：
- **Role/ClusterRole**: 定义可以对哪些资源执行哪些操作
- **RoleBinding/ClusterRoleBinding**: 将角色绑定到用户、组或服务账号

```bash
# 查看当前集群中的角色
kubectl get roles --all-namespaces

# 查看集群角色
kubectl get clusterroles

# 查看角色绑定
kubectl get rolebindings --all-namespaces

# 查看集群角色绑定
kubectl get clusterrolebindings
```

### 2. 创建一个命名空间

为我们的实验创建一个专用命名空间：

```bash
# 创建命名空间
kubectl create namespace security-demo

# 设置当前上下文使用此命名空间
kubectl config set-context --current --namespace=security-demo
```

### 3. 创建服务账号

服务账号是Pod中运行的应用程序的身份：

```bash
# 创建一个服务账号
kubectl create serviceaccount demo-sa

# 查看创建的服务账号
kubectl get serviceaccounts

# 查看服务账号详细信息
kubectl describe serviceaccount demo-sa
```

### 4. 创建角色

创建一个只允许读取特定资源的角色：

```bash
# 创建角色配置文件
cat > reader-role.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: security-demo
rules:
- apiGroups: [""] # 核心API组
  resources: ["pods", "pods/log"]
  verbs: ["get", "watch", "list"]
EOF

# 应用配置
kubectl apply -f reader-role.yaml

# 查看创建的角色
kubectl get roles

# 查看角色详细信息
kubectl describe role pod-reader
```

### 5. 创建角色绑定

将角色绑定到服务账号：

```bash
# 创建角色绑定配置文件
cat > reader-binding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: security-demo
subjects:
- kind: ServiceAccount
  name: demo-sa
  namespace: security-demo
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF

# 应用配置
kubectl apply -f reader-binding.yaml

# 查看创建的角色绑定
kubectl get rolebindings

# 查看角色绑定详细信息
kubectl describe rolebinding read-pods
```

### 6. 测试RBAC配置

部署一个使用demo-sa服务账号的Pod，并测试其权限：

```bash
# 创建一个测试Pod
cat > rbac-test-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: rbac-test
  namespace: security-demo
spec:
  serviceAccountName: demo-sa
  containers:
  - name: curl
    image: curlimages/curl:7.83.1
    command: ["sleep", "3600"]
EOF

# 应用配置
kubectl apply -f rbac-test-pod.yaml

# 等待Pod运行
kubectl get pods -w
```

使用此Pod测试权限：

```bash
# 在Pod中执行测试
# 列出Pods (应该成功)
kubectl exec -it rbac-test -- sh -c 'curl -s -k -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" https://kubernetes.default.svc/api/v1/namespaces/security-demo/pods'

# 尝试删除Pod (应该失败)
kubectl exec -it rbac-test -- sh -c 'curl -s -k -X DELETE -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" https://kubernetes.default.svc/api/v1/namespaces/security-demo/pods/rbac-test'

# 尝试访问其他命名空间 (应该失败)
kubectl exec -it rbac-test -- sh -c 'curl -s -k -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" https://kubernetes.default.svc/api/v1/namespaces/default/pods'
```

### 7. 创建ClusterRole和ClusterRoleBinding

ClusterRole和ClusterRoleBinding在集群范围内生效：

```bash
# 创建一个ClusterRole
cat > deployment-viewer.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployment-viewer
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
EOF

kubectl apply -f deployment-viewer.yaml

# 创建ClusterRoleBinding
cat > deployment-viewer-binding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: deployment-view
subjects:
- kind: ServiceAccount
  name: demo-sa
  namespace: security-demo
roleRef:
  kind: ClusterRole
  name: deployment-viewer
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f deployment-viewer-binding.yaml

# 验证权限
kubectl exec -it rbac-test -- sh -c 'curl -s -k -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" https://kubernetes.default.svc/apis/apps/v1/namespaces/security-demo/deployments'
```

### 8. 创建限制性Network Policy

网络策略限制Pod之间的通信：

```bash
# 创建测试应用
cat > web-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: security-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
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
  name: web-service
  namespace: security-demo
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
EOF

kubectl apply -f web-app.yaml

# 创建一个访问者Pod
cat > client-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: client
  namespace: security-demo
  labels:
    app: client
spec:
  containers:
  - name: busybox
    image: busybox:1.28
    command: ['sh', '-c', 'while true; do sleep 10; done']
EOF

kubectl apply -f client-pod.yaml

# 测试不受限的网络访问
kubectl exec -it client -- wget -O- --timeout=2 web-service

# 创建限制性网络策略
cat > restrict-network.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-deny-all
  namespace: security-demo
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
EOF

kubectl apply -f restrict-network.yaml

# 再次测试网络访问（现在应该超时）
kubectl exec -it client -- wget -O- --timeout=2 web-service || echo "Connection blocked as expected"

# 创建允许特定标签访问的网络策略
cat > allow-client.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-allow-client
  namespace: security-demo
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: client
EOF

kubectl apply -f allow-client.yaml

# 再次测试（现在应该成功）
kubectl exec -it client -- wget -O- --timeout=2 web-service
```

### 9. 配置Security Context

SecurityContext定义Pod或容器的特权和访问控制设置：

```bash
# 创建一个有安全上下文配置的Pod
cat > security-context-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: security-context-pod
  namespace: security-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: sec-ctx-demo
    image: busybox:1.28
    command: ["sh", "-c", "sleep 3600"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: data-volume
      mountPath: /data/demo
  volumes:
  - name: data-volume
    emptyDir: {}
EOF

kubectl apply -f security-context-pod.yaml

# 验证安全上下文设置
kubectl exec -it security-context-pod -- sh -c "id"

# 验证文件权限
kubectl exec -it security-context-pod -- sh -c "ls -la /data"

# 尝试提权（应该失败）
kubectl exec -it security-context-pod -- sh -c "ping -c 1 8.8.8.8" || echo "Capability dropped as expected"
```

### 10. 实施Pod安全策略

Pod安全策略在Kubernetes 1.21+已被Pod安全准入控制器替代，但了解其概念仍很重要：

```bash
# 在Minikube中，我们可以使用Pod Security Standards通过命名空间标签实现类似功能

# 给命名空间添加安全标签
kubectl label namespace security-demo pod-security.kubernetes.io/enforce=baseline

# 创建一个特权Pod (应该失败)
cat > privileged-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: security-demo
spec:
  containers:
  - name: privileged
    image: nginx:1.21
    securityContext:
      privileged: true
EOF

kubectl apply -f privileged-pod.yaml
```

### 11. 使用Secrets管理敏感数据

```bash
# 创建一个包含敏感数据的Secret
kubectl create secret generic app-secrets \
  --from-literal=db-password=MySecr3tP@ssw0rd \
  --from-literal=api-key=Ak92jDwsuaT6hGqp

# 查看创建的Secret
kubectl get secrets

# 查看Secret详情（注意值是base64编码的）
kubectl describe secret app-secrets

# 创建使用Secret的Pod
cat > secret-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
  namespace: security-demo
spec:
  containers:
  - name: secret-container
    image: busybox:1.28
    command: ["sh", "-c", "sleep 3600"]
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: db-password
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: app-secrets
EOF

kubectl apply -f secret-pod.yaml

# 验证Secret作为环境变量
kubectl exec -it secret-pod -- sh -c "echo \$DB_PASSWORD"

# 验证Secret作为文件
kubectl exec -it secret-pod -- sh -c "cat /etc/secrets/api-key"
```

### 12. 创建自定义用户证书

在生产环境中，通常需要为人类用户创建证书：

```bash
# 生成私钥
openssl genrsa -out dev-user.key 2048

# 创建证书签名请求 (CSR)
openssl req -new -key dev-user.key -out dev-user.csr -subj "/CN=dev-user/O=development"

# 获取Minikube CA证书和密钥的路径
MINIKUBE_CA_CERT=$(minikube ssh "sudo cat /var/lib/minikube/certs/ca.crt")
MINIKUBE_CA_KEY=$(minikube ssh "sudo cat /var/lib/minikube/certs/ca.key")

# 将CA证书和密钥保存到本地文件
echo "$MINIKUBE_CA_CERT" > ca.crt
echo "$MINIKUBE_CA_KEY" > ca.key

# 使用CA签名证书
openssl x509 -req -in dev-user.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out dev-user.crt -days 365

# 在kubectl中配置用户
kubectl config set-credentials dev-user --client-certificate=dev-user.crt --client-key=dev-user.key

# 创建新的上下文
kubectl config set-context dev-user-context --cluster=minikube --user=dev-user --namespace=security-demo

# 测试新用户（应该没有权限）
kubectl --context=dev-user-context get pods
```

### 13. 为新用户创建有限权限

```bash
# 创建一个只读ClusterRole
cat > readonly-role.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-readonly
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
EOF

kubectl apply -f readonly-role.yaml

# 为dev-user创建RoleBinding
cat > dev-user-binding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-user-view
  namespace: security-demo
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: namespace-readonly
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f dev-user-binding.yaml

# 再次测试（现在应该可以读取但不能修改）
kubectl --context=dev-user-context get pods
kubectl --context=dev-user-context get deployments
kubectl --context=dev-user-context create deployment test-nginx --image=nginx
```

### 14. 自动挂载服务账号令牌

在Kubernetes 1.21+中，令牌的自动挂载有了变化，可以显式控制：

```bash
# 创建不自动挂载令牌的服务账号
cat > no-automount-sa.yaml << 'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-automount-sa
  namespace: security-demo
automountServiceAccountToken: false
EOF

kubectl apply -f no-automount-sa.yaml

# 创建一个使用此服务账号的Pod
cat > no-token-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: no-token-pod
  namespace: security-demo
spec:
  serviceAccountName: no-automount-sa
  containers:
  - name: busybox
    image: busybox:1.28
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply -f no-token-pod.yaml

# 验证令牌不存在
kubectl exec -it no-token-pod -- sh -c "ls -la /var/run/secrets/kubernetes.io/serviceaccount || echo 'Token not mounted'"
```

### 15. 清理资源

```bash
# 删除命名空间将删除其中的所有资源
kubectl delete namespace security-demo

# 删除ClusterRole和ClusterRoleBinding
kubectl delete clusterrole deployment-viewer namespace-readonly
kubectl delete clusterrolebinding deployment-view

# 删除用户上下文和证书
kubectl config delete-context dev-user-context
kubectl config unset users.dev-user
rm -f dev-user.key dev-user.crt dev-user.csr ca.crt ca.key ca.srl
```

## 思考问题

1. RBAC中的Role和ClusterRole有什么区别？何时应使用其中一个而不是另一个？
2. 服务账号与用户账号在Kubernetes中有什么不同？各自在什么场景下使用？
3. NetworkPolicy如何提高集群的安全性？在微服务架构中如何使用它们？
4. Secret相比ConfigMap有哪些安全优势和限制？有哪些最佳实践来保护Secret中的敏感数据？
5. 最小权限原则如何应用于Kubernetes环境？如何平衡安全性和可用性？

## 扩展练习

1. 实现基于组的RBAC，创建开发、测试和运维三个不同权限级别的组
2. 配置Pod使用非root用户运行，并验证其安全性
3. 研究并实现Pod Security Standards的不同安全级别（Privileged、Baseline、Restricted）
4. 使用外部身份提供者（如LDAP或OIDC）配置Kubernetes身份验证
5. 探索Kubernetes中的审计日志功能，并实现一个基本的安全审计系统 