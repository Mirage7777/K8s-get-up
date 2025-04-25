# 实验2.5：Kubernetes配置与密钥管理

## 目标
学习如何在Kubernetes中使用ConfigMap和Secret管理应用配置和敏感数据，理解不同的注入方式（环境变量、卷挂载）及其适用场景。

## 准备工作
- 完成实验2.1到2.4
- 运行中的Minikube集群
- 熟悉基本的kubectl命令

## 配置管理概述

在Kubernetes中，应用程序配置和敏感数据管理主要通过以下资源实现：

1. **ConfigMap**：用于存储非敏感的配置数据，如配置文件、命令行参数等
2. **Secret**：专门用于存储敏感数据，如密码、OAuth令牌、SSH密钥等
3. **环境变量**：将配置作为环境变量注入到容器中
4. **卷挂载**：将配置作为文件挂载到容器的文件系统中

## 步骤

### 1. 准备测试应用

我们将创建一个能够演示配置和密钥使用的应用：

```bash
# 创建项目目录
mkdir -p k8s-config-demo
cd k8s-config-demo

# 创建应用文件
cat > app.py << 'EOF'
from flask import Flask, jsonify
import os
import json

app = Flask(__name__)

@app.route('/')
def index():
    return jsonify({
        'message': 'Kubernetes Config and Secret Demo',
        'env_config': get_env_config(),
        'file_config': get_file_config(),
        'env_secrets': get_env_secrets(),
        'file_secrets': get_file_secrets()
    })

def get_env_config():
    """获取环境变量形式的配置"""
    return {
        'DB_HOST': os.environ.get('DB_HOST', 'not set'),
        'DB_PORT': os.environ.get('DB_PORT', 'not set'),
        'APP_MODE': os.environ.get('APP_MODE', 'not set'),
        'LOG_LEVEL': os.environ.get('LOG_LEVEL', 'not set'),
    }

def get_file_config():
    """获取文件形式的配置"""
    config = {}
    config_path = '/config'
    
    if os.path.exists(config_path):
        for filename in os.listdir(config_path):
            file_path = os.path.join(config_path, filename)
            if os.path.isfile(file_path):
                with open(file_path, 'r') as f:
                    config[filename] = f.read().strip()
                    
                    # 尝试解析JSON文件
                    if filename.endswith('.json'):
                        try:
                            config[filename] = json.loads(config[filename])
                        except:
                            pass
    return config

def get_env_secrets():
    """获取环境变量形式的密钥"""
    return {
        'DB_USER': os.environ.get('DB_USER', 'not set'),
        'DB_PASSWORD': os.environ.get('DB_PASSWORD', 'not set'),
        # 脱敏处理，仅显示长度
        'API_KEY': f"{'*' * len(os.environ.get('API_KEY', ''))} (length: {len(os.environ.get('API_KEY', ''))})"
    }

def get_file_secrets():
    """获取文件形式的密钥"""
    secrets = {}
    secrets_path = '/secrets'
    
    if os.path.exists(secrets_path):
        for filename in os.listdir(secrets_path):
            file_path = os.path.join(secrets_path, filename)
            if os.path.isfile(file_path):
                with open(file_path, 'r') as f:
                    content = f.read().strip()
                    # 脱敏处理，仅显示长度
                    secrets[filename] = f"{'*' * len(content)} (length: {len(content)})"
    return secrets

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
docker build -t config-demo:v1 .
minikube image load config-demo:v1
```

### 2. 创建基本ConfigMap

从各种来源创建ConfigMap：

```bash
# 从文字值创建ConfigMap
kubectl create configmap app-env-config \
  --from-literal=APP_MODE=development \
  --from-literal=LOG_LEVEL=DEBUG \
  --from-literal=DB_HOST=mysql-service \
  --from-literal=DB_PORT=3306

# 查看创建的ConfigMap
kubectl get configmap app-env-config -o yaml
```

创建包含多行内容和JSON配置的ConfigMap：

```bash
# 创建一个包含多行内容的ConfigMap
cat > config-files.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-file-config
data:
  app.properties: |
    # 应用属性配置
    app.name=Config Demo
    app.version=1.0
    app.description=Kubernetes Config and Secret Demo
    
  app-config.json: |
    {
      "database": {
        "maxConnections": 100,
        "timeout": 30,
        "retryCount": 3
      },
      "cache": {
        "enabled": true,
        "ttl": 300
      },
      "features": {
        "featureA": true,
        "featureB": false,
        "featureC": true
      }
    }
    
  nginx.conf: |
    server {
        listen 80;
        server_name example.com;
        
        location / {
            proxy_pass http://localhost:5000;
        }
    }
EOF

kubectl apply -f config-files.yaml
```

### 3. 创建Secret

```bash
# 从文字值创建Secret
kubectl create secret generic app-env-secrets \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=super-secret-password \
  --from-literal=API_KEY=a1b2c3d4e5f6g7h8i9j0

# 查看创建的Secret
kubectl get secret app-env-secrets -o yaml

# 创建包含文件的Secret
# 先创建一些示例密钥文件
echo "-----BEGIN PRIVATE KEY-----
MIIEvwIBADANBgkqhkiG9w0BAQEFAASCBKkwggSlAgEAAoIBAQDEJ0Eee4A5xNQl
2ivxUmdQaH7MJ5lnSgdSuaLZhT1JBlbxSFYEJ4p6UcXQCi0DnNZYfa8CDO7ToVCp
..." > dummy-private-key.pem

echo "{\"auths\":{\"https://index.docker.io/v1/\":{\"auth\":\"c2VjcmV0Y3JlZGVudGlhbHM=\"}}}" > docker-config.json

# 从文件创建Secret
kubectl create secret generic app-file-secrets \
  --from-file=ssh-private-key=dummy-private-key.pem \
  --from-file=docker-config=docker-config.json

# 查看创建的Secret
kubectl get secret app-file-secrets -o yaml
```

### 4. 使用环境变量方式注入配置和密钥

```bash
# 创建使用环境变量的部署
cat > env-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-demo-env
  labels:
    app: config-demo
    method: env
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-demo
      method: env
  template:
    metadata:
      labels:
        app: config-demo
        method: env
    spec:
      containers:
      - name: config-demo
        image: config-demo:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
        env:
        # 直接设置的环境变量
        - name: POD_DEPLOY_METHOD
          value: "environment variables"
        
        # 从ConfigMap注入的环境变量
        - name: APP_MODE
          valueFrom:
            configMapKeyRef:
              name: app-env-config
              key: APP_MODE
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-env-config
              key: LOG_LEVEL
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-env-config
              key: DB_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: app-env-config
              key: DB_PORT
              
        # 从Secret注入的环境变量
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: app-env-secrets
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-env-secrets
              key: DB_PASSWORD
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-env-secrets
              key: API_KEY
EOF

kubectl apply -f env-deployment.yaml

# 创建服务
cat > demo-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: config-demo
spec:
  selector:
    app: config-demo
  ports:
  - port: 80
    targetPort: 5000
  type: NodePort
EOF

kubectl apply -f demo-service.yaml
```

### 5. 使用卷挂载方式注入配置和密钥

```bash
# 创建使用卷挂载的部署
cat > volume-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-demo-volume
  labels:
    app: config-demo
    method: volume
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-demo
      method: volume
  template:
    metadata:
      labels:
        app: config-demo
        method: volume
    spec:
      containers:
      - name: config-demo
        image: config-demo:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
        env:
        - name: POD_DEPLOY_METHOD
          value: "volume mounts"
        volumeMounts:
        # 挂载ConfigMap作为文件
        - name: config-volume
          mountPath: /config
        # 挂载Secret作为文件
        - name: secrets-volume
          mountPath: /secrets
          readOnly: true
      volumes:
      # ConfigMap卷
      - name: config-volume
        configMap:
          name: app-file-config
      # Secret卷
      - name: secrets-volume
        secret:
          secretName: app-file-secrets
EOF

kubectl apply -f volume-deployment.yaml
```

### 6. 测试两种部署方式

```bash
# 获取服务URL
minikube service config-demo --url

# 获取两个不同部署的Pod名称
ENV_POD=$(kubectl get pod -l app=config-demo,method=env -o jsonpath='{.items[0].metadata.name}')
VOLUME_POD=$(kubectl get pod -l app=config-demo,method=volume -o jsonpath='{.items[0].metadata.name}')

# 端口转发
kubectl port-forward $ENV_POD 5001:5000 &
kubectl port-forward $VOLUME_POD 5002:5000 &

# 测试环境变量方式
curl http://localhost:5001/

# 测试卷挂载方式
curl http://localhost:5002/
```

### 7. 更新ConfigMap和Secret并观察变化

```bash
# 更新ConfigMap
kubectl patch configmap app-env-config -p '{"data":{"LOG_LEVEL":"INFO"}}'

# 检查环境变量方式的Pod是否反映了变化（需要重启Pod）
kubectl delete pod $ENV_POD
# 等待新Pod启动
sleep 10
ENV_POD=$(kubectl get pod -l app=config-demo,method=env -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward $ENV_POD 5001:5000 &
curl http://localhost:5001/

# 更新文件ConfigMap
kubectl patch configmap app-file-config --patch '{"data":{"app-config.json":"{\"database\":{\"maxConnections\":200,\"timeout\":30,\"retryCount\":3},\"cache\":{\"enabled\":true,\"ttl\":600},\"features\":{\"featureA\":true,\"featureB\":true,\"featureC\":true}}"}}'

# 观察卷挂载方式的变化（会自动更新，但可能有延迟）
sleep 30
curl http://localhost:5002/
```

### 8. 使用环境变量集注入所有ConfigMap键值对

```bash
# 创建使用envFrom的部署
cat > envfrom-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-demo-envfrom
  labels:
    app: config-demo
    method: envfrom
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-demo
      method: envfrom
  template:
    metadata:
      labels:
        app: config-demo
        method: envfrom
    spec:
      containers:
      - name: config-demo
        image: config-demo:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
        env:
        - name: POD_DEPLOY_METHOD
          value: "envFrom"
        envFrom:
        - configMapRef:
            name: app-env-config
        - secretRef:
            name: app-env-secrets
EOF

kubectl apply -f envfrom-deployment.yaml
```

### 9. 挂载特定的配置文件并设置权限

```bash
# 创建具有特定文件挂载和权限的部署
cat > specific-mount-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-demo-specific
  labels:
    app: config-demo
    method: specific
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-demo
      method: specific
  template:
    metadata:
      labels:
        app: config-demo
        method: specific
    spec:
      containers:
      - name: config-demo
        image: config-demo:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
        env:
        - name: POD_DEPLOY_METHOD
          value: "specific volume mounts"
        volumeMounts:
        # 只挂载特定文件
        - name: config-volume
          mountPath: /config/app-config.json
          subPath: app-config.json
        - name: config-volume
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        - name: secrets-volume
          mountPath: /etc/ssh/id_rsa
          subPath: ssh-private-key
      volumes:
      - name: config-volume
        configMap:
          name: app-file-config
          defaultMode: 0644  # 设置文件权限
      - name: secrets-volume
        secret:
          secretName: app-file-secrets
          defaultMode: 0600  # 设置Secret文件更严格的权限
EOF

kubectl apply -f specific-mount-deployment.yaml
```

### 10. 使用不可变ConfigMap和Secret

```bash
# 创建不可变ConfigMap
cat > immutable-config.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: immutable-config
data:
  APP_VERSION: "1.0.0"
  FEATURE_FLAGS: "auth=true,metrics=true,notifications=false"
immutable: true
EOF

kubectl apply -f immutable-config.yaml

# 创建不可变Secret
cat > immutable-secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: immutable-secret
type: Opaque
data:
  production-api-key: cHJvZHVjdGlvbi1hcGkta2V5LXZhbHVl  # base64编码
  admin-token: YWRtaW4tdG9rZW4tdmFsdWU=  # base64编码
immutable: true
EOF

kubectl apply -f immutable-secret.yaml

# 尝试修改不可变ConfigMap（将失败）
kubectl patch configmap immutable-config -p '{"data":{"APP_VERSION":"1.0.1"}}'
```

### 11. 清理资源

```bash
# 停止端口转发
pkill -f "kubectl port-forward"

# 删除创建的所有资源
kubectl delete deployment config-demo-env config-demo-volume config-demo-envfrom config-demo-specific
kubectl delete service config-demo
kubectl delete configmap app-env-config app-file-config immutable-config
kubectl delete secret app-env-secrets app-file-secrets immutable-secret

# 删除临时文件
rm -f dummy-private-key.pem docker-config.json
```

## 思考问题

1. 什么情况下应该使用ConfigMap，什么情况下应该使用Secret？它们在安全性上有什么区别？
2. 环境变量注入和卷挂载注入各有什么优缺点？不同场景下应该如何选择？
3. ConfigMap和Secret更新后，两种注入方式的更新行为有什么不同？这对应用有什么影响？
4. 不可变ConfigMap和Secret有什么优势？它们适用于哪些场景？
5. 在多环境部署（开发、测试、生产）中，如何最佳地管理配置？

## 扩展练习

1. 实现使用Kubernetes Downward API将Pod信息（如名称、IP、标签）注入到容器中
2. 探索使用HashiCorp Vault与Kubernetes集成，实现更安全的密钥管理
3. 设计一个具有多个容器的Pod，其中一个容器作为init容器，从ConfigMap生成其他容器需要的配置
4. 使用Sealed Secrets或Bitnami Sealed Secrets加密Kubernetes Secret
5. 实现配置热重载 - 当ConfigMap更新时，应用程序自动重新加载配置，而不需要重启Pod 