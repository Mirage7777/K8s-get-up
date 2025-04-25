# 实验3.3：Helm包管理器

## 目标

学习使用Helm进行Kubernetes应用的包管理，掌握创建、安装、升级和管理Helm图表的方法，理解Helm如何简化Kubernetes应用的部署和管理。

## 准备工作

- 完成实验2.1至3.2
- 运行中的Minikube集群
- 熟悉基本的kubectl命令
- 对Kubernetes资源有基本了解

## Helm概述

Helm是Kubernetes的包管理器，它允许您定义、安装和升级复杂的Kubernetes应用程序。主要概念包括：

- **Chart**：Helm包，包含所有预定义的Kubernetes资源
- **Release**：Chart的运行实例，每次安装Chart都会创建一个新的release
- **Repository**：存储和共享Chart的地方
- **Values**：用于自定义Chart的配置值

## 步骤

### 1. 安装Helm客户端

```bash
# 下载并安装Helm
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash

# 验证安装
helm version
```

### 2. 添加和管理Helm仓库

```bash
# 添加官方稳定版仓库
helm repo add stable https://charts.helm.sh/stable

# 添加Bitnami仓库，提供许多常用应用
helm repo add bitnami https://charts.bitnami.com/bitnami

# 更新仓库
helm repo update

# 列出添加的仓库
helm repo list

# 搜索可用的Chart
helm search repo nginx
helm search repo database
```

### 3. 部署预构建的Helm Chart

```bash
# 搜索MySQL Chart
helm search repo mysql

# 查看Chart的详细信息和可配置参数
helm show chart bitnami/mysql
helm show values bitnami/mysql

# 安装MySQL Chart
helm install my-mysql bitnami/mysql --set auth.rootPassword=password

# 查看安装的release
helm list

# 查看部署状态
kubectl get pods
kubectl get services
```

### 4. 自定义Chart安装配置

```bash
# 创建自定义values文件
cat > mysql-values.yaml << 'EOF'
auth:
  rootPassword: "mysecretpassword"
  database: "mydb"
  username: "myuser"
  password: "myuserpassword"
primary:
  persistence:
    size: 2Gi
  resources:
    limits:
      memory: 512Mi
      cpu: 500m
    requests:
      memory: 256Mi
      cpu: 250m
EOF

# 使用自定义values文件安装Chart
helm install mysql-custom bitnami/mysql -f mysql-values.yaml

# 验证自定义安装
kubectl get pods
kubectl get pvc
```

### 5. 管理Helm Release

```bash
# 查看运行的release
helm list

# 查看特定release的状态
helm status my-mysql

# 查看release的所有信息，包括生成的manifest
helm get all my-mysql

# 查看release的values
helm get values my-mysql
```

### 6. 升级和回滚Release

```bash
# 更新自定义values文件
cat > mysql-values-updated.yaml << 'EOF'
auth:
  rootPassword: "mysecretpassword"
  database: "mydb"
  username: "myuser"
  password: "myuserpassword"
primary:
  persistence:
    size: 4Gi
  resources:
    limits:
      memory: 1Gi
      cpu: 1000m
    requests:
      memory: 512Mi
      cpu: 500m
EOF

# 升级release
helm upgrade mysql-custom bitnami/mysql -f mysql-values-updated.yaml

# 查看升级后的资源
kubectl get pvc
kubectl get pods

# 查看release历史
helm history mysql-custom

# 回滚到之前的版本
helm rollback mysql-custom 1

# 验证回滚
helm status mysql-custom
kubectl get pvc
```

### 7. 卸载Release

```bash
# 列出所有release
helm list

# 卸载release
helm uninstall my-mysql

# 确认已卸载
helm list
kubectl get pods
```

### 8. 创建自定义Helm Chart

```bash
# 创建新Chart
helm create my-webapp

# 探索Chart结构
ls -la my-webapp
```

Chart的基本结构：
- **Chart.yaml**: Chart的元数据
- **values.yaml**: 默认配置值
- **templates/**: 模板文件目录
- **charts/**: 依赖的子Chart
- **templates/NOTES.txt**: 安装后显示的说明

### 9. 修改自定义Chart

```bash
# 编辑Chart.yaml
cat > my-webapp/Chart.yaml << 'EOF'
apiVersion: v2
name: my-webapp
description: A simple web application Helm chart
type: application
version: 0.1.0
appVersion: "1.0.0"
EOF

# 编辑values.yaml
cat > my-webapp/values.yaml << 'EOF'
replicaCount: 2

image:
  repository: nginx
  tag: "1.21.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi

ingress:
  enabled: true
  className: "nginx"
  hosts:
    - host: myapp.local
      paths:
        - path: /
          pathType: Prefix
EOF

# 查看并修改部署模板
cat > my-webapp/templates/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-webapp.fullname" . }}
  labels:
    {{- include "my-webapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-webapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-webapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
EOF

# 添加ConfigMap示例
cat > my-webapp/templates/configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "my-webapp.fullname" . }}-config
  labels:
    {{- include "my-webapp.labels" . | nindent 4 }}
data:
  nginx.conf: |
    server {
      listen 80;
      server_name localhost;
      
      location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        try_files $uri $uri/ /index.html;
      }
    }
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
      <title>Welcome to My Helm Chart!</title>
      <style>
        body {
          font-family: Arial, sans-serif;
          margin: 40px;
          text-align: center;
        }
        h1 {
          color: #3476db;
        }
      </style>
    </head>
    <body>
      <h1>Hello from My Custom Helm Chart!</h1>
      <p>Deployed with Helm version: {{ .Capabilities.HelmVersion.Version }}</p>
      <p>Kubernetes version: {{ .Capabilities.KubeVersion.Version }}</p>
      <p>App version: {{ .Chart.AppVersion }}</p>
    </body>
    </html>
EOF

# 更新部署模板以使用ConfigMap
cat > my-webapp/templates/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-webapp.fullname" . }}
  labels:
    {{- include "my-webapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-webapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-webapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          volumeMounts:
            - name: config-volume
              mountPath: /usr/share/nginx/html/index.html
              subPath: index.html
            - name: config-volume
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: nginx.conf
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      volumes:
        - name: config-volume
          configMap:
            name: {{ include "my-webapp.fullname" . }}-config
EOF
```

### 10. 验证和安装自定义Chart

```bash
# 检查Chart语法
helm lint my-webapp

# 查看生成的模板（不安装）
helm template my-webapp ./my-webapp

# 进行试运行（不安装）
helm install --dry-run --debug my-release ./my-webapp

# 安装Chart
helm install my-release ./my-webapp

# 查看安装状态
helm list
kubectl get all -l app.kubernetes.io/instance=my-release
```

### 11. 打包和分发Chart

```bash
# 打包Chart
helm package ./my-webapp

# 创建本地Chart仓库
mkdir -p ~/helm-repo
mv my-webapp-0.1.0.tgz ~/helm-repo/

# 创建仓库索引
helm repo index ~/helm-repo/

# 查看生成的索引文件
cat ~/helm-repo/index.yaml

# 添加本地仓库
helm repo add local file://$HOME/helm-repo

# 查询本地仓库中的Chart
helm search repo local
```

### 12. 使用依赖项创建复合应用

```bash
# 创建带有依赖的应用
helm create my-platform

# 编辑Chart.yaml，添加依赖项
cat > my-platform/Chart.yaml << 'EOF'
apiVersion: v2
name: my-platform
description: A platform with multiple components
type: application
version: 0.1.0
appVersion: "1.0.0"
dependencies:
  - name: mysql
    version: "~9.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: mysql.enabled
  - name: redis
    version: "~17.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
EOF

# 创建自定义values.yaml
cat > my-platform/values.yaml << 'EOF'
# 全局配置
global:
  storageClass: "standard"

# MySQL配置
mysql:
  enabled: true
  auth:
    rootPassword: "platformpassword"
    database: "platform_db"
  primary:
    persistence:
      size: 1Gi

# Redis配置
redis:
  enabled: true
  auth:
    password: "redis-password"
  master:
    persistence:
      size: 1Gi
EOF

# 更新依赖
helm dependency update ./my-platform

# 安装平台Chart
helm install my-platform ./my-platform

# 查看部署状态
helm list
kubectl get all
```

### 13. Helm Hooks的使用

创建包含Hooks的Chart示例：

```bash
# 回到我们的web应用Chart
cd my-webapp

# 添加pre-install hook示例
cat > my-webapp/templates/pre-install-job.yaml << 'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "my-webapp.fullname" . }}-pre-install
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
  labels:
    {{- include "my-webapp.labels" . | nindent 4 }}
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: pre-install-job
          image: busybox
          command: ['sh', '-c', 'echo "Preparing for installation..."; sleep 5; echo "Environment ready for installation"']
EOF

# 添加post-install hook示例
cat > my-webapp/templates/post-install-job.yaml << 'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "my-webapp.fullname" . }}-post-install
  annotations:
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "5"
    "helm.sh/hook-delete-policy": hook-succeeded
  labels:
    {{- include "my-webapp.labels" . | nindent 4 }}
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: post-install-job
          image: busybox
          command: ['sh', '-c', 'echo "Application installed successfully"; echo "Performing post-installation checks..."; sleep 5; echo "All checks passed!"']
EOF

# 升级应用以应用新hooks
cd ..
helm upgrade my-release ./my-webapp

# 查看Jobs
kubectl get jobs
```

### 14. 使用Helm进行生产部署策略

```bash
# 创建用于生产环境的values文件
cat > my-webapp/production-values.yaml << 'EOF'
replicaCount: 3

image:
  repository: nginx
  tag: "stable"
  pullPolicy: Always

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

# 启用水平自动缩放
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

# 添加PodDisruptionBudget
pdb:
  enabled: true
  minAvailable: 2

# 添加存活探针和就绪探针
probes:
  liveness:
    enabled: true
    initialDelaySeconds: 30
    periodSeconds: 10
  readiness:
    enabled: true
    initialDelaySeconds: 10
    periodSeconds: 5
EOF

# 添加HPA和PDB模板
cat > my-webapp/templates/hpa.yaml << 'EOF'
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "my-webapp.fullname" . }}
  labels:
    {{- include "my-webapp.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "my-webapp.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
    {{- end }}
{{- end }}
EOF

cat > my-webapp/templates/pdb.yaml << 'EOF'
{{- if .Values.pdb.enabled }}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "my-webapp.fullname" . }}
  labels:
    {{- include "my-webapp.labels" . | nindent 4 }}
spec:
  minAvailable: {{ .Values.pdb.minAvailable }}
  selector:
    matchLabels:
      {{- include "my-webapp.selectorLabels" . | nindent 6 }}
{{- end }}
EOF

# 更新部署模板以添加探针
cat >> my-webapp/templates/deployment.yaml << 'EOF'
          {{- if .Values.probes.liveness.enabled }}
          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
          {{- end }}
          {{- if .Values.probes.readiness.enabled }}
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
          {{- end }}
EOF

# 创建生产版本的部署
helm upgrade --install production-webapp ./my-webapp -f my-webapp/production-values.yaml

# 查看部署
kubectl get all -l app.kubernetes.io/instance=production-webapp
```

### 15. 清理资源

```bash
# 卸载所有已安装的Chart
helm uninstall my-mysql mysql-custom my-release my-platform production-webapp

# 验证资源已清理
kubectl get all
helm list
```

## 思考问题

1. Helm与直接使用kubectl应用YAML清单文件相比有什么优势？在什么情况下您会选择使用Helm而不是kubectl？
2. Helm的模板系统提供了哪些功能，这些功能如何帮助解决复杂应用部署中的常见问题？
3. 在使用Helm部署应用时，如何处理环境特定的配置（如开发、测试和生产环境）？
4. Helm Chart版本控制和发布策略应遵循哪些最佳实践？
5. 在多集群或多租户环境中使用Helm时需要考虑哪些特殊因素？

## 扩展练习

1. 将现有的Kubernetes应用转换为Helm Chart
2. 创建一个包含多个微服务的复杂应用Chart，并实现不同组件之间的通信
3. 实现使用外部密钥管理系统（如Vault）的Helm Chart
4. 为一个数据库应用创建Helm Chart，包括备份和恢复功能
5. 构建带有自定义Webhook的Chart，实现更细粒度的部署控制 