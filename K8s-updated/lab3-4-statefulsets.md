# 实验3.4：StatefulSet和有状态应用

## 目标
学习如何使用Kubernetes StatefulSet部署有状态应用，掌握StatefulSet的特性和使用场景，以及如何管理有状态应用的存储和网络标识。

## 准备工作
- 完成实验3.1至3.3
- 运行中的Minikube集群
- 熟悉基本的kubectl命令
- 对持久卷和持久卷声明有基本了解

## StatefulSet概述

StatefulSet是用于管理有状态应用程序的工作负载API对象，具有以下特性：
- 稳定的、唯一的网络标识符
- 稳定的、持久的存储
- 有序的、优雅的部署和扩展
- 有序的、优雅的删除和终止
- 有序的、自动的滚动更新

这些特性使StatefulSet特别适合需要以下一个或多个要求的应用程序：
- 稳定的网络标识或持久存储
- 有序部署和扩展
- 有序的自动滚动更新

## 步骤

### 1. 创建存储类(StorageClass)

首先创建一个可以动态配置持久卷的StorageClass：

```bash
# 创建StorageClass配置文件
cat > stateful-storageclass.yaml << 'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-stateful
provisioner: k8s.io/minikube-hostpath
reclaimPolicy: Delete
volumeBindingMode: Immediate
EOF

# 应用配置
kubectl apply -f stateful-storageclass.yaml

# 查看StorageClass
kubectl get storageclass
```

### 2. 创建Headless Service

StatefulSet需要一个Headless Service来控制Pod的网络域名：

```bash
# 创建Headless Service配置文件
cat > headless-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
  labels:
    app: nginx-stateful
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx-stateful
EOF

# 应用配置
kubectl apply -f headless-service.yaml

# 查看创建的Service
kubectl get service nginx-headless
```

### 3. 创建简单的StatefulSet

创建一个运行Nginx的StatefulSet示例：

```bash
# 创建StatefulSet配置文件
cat > nginx-statefulset.yaml << 'EOF'
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx-headless"
  replicas: 3
  selector:
    matchLabels:
      app: nginx-stateful
  template:
    metadata:
      labels:
        app: nginx-stateful
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "standard-stateful"
      resources:
        requests:
          storage: 1Gi
EOF

# 应用配置
kubectl apply -f nginx-statefulset.yaml

# 观察StatefulSet的创建过程
kubectl get statefulset web
kubectl get pods -w
```

注意Pod是如何按照从0到N-1的顺序被创建的，下一个Pod只有在前一个Pod变为Running和Ready状态后才会创建。

### 4. 探索StatefulSet的网络标识

```bash
# 查看创建的Pod
kubectl get pods -l app=nginx-stateful

# 查看自动创建的PVC
kubectl get pvc

# 进入到一个Pod中测试DNS解析
kubectl exec -it web-0 -- bash

# 在Pod内部执行以下命令
# 解析同一StatefulSet中的其他Pod
nslookup web-1.nginx-headless
nslookup web-2.nginx-headless

# 退出Pod
exit
```

### 5. 验证StatefulSet的存储

```bash
# 在每个Pod中创建不同的index.html
for i in 0 1 2; do
  kubectl exec "web-$i" -- bash -c "echo 'Hello from web-$i' > /usr/share/nginx/html/index.html"
done

# 创建一个临时Service以访问Pod
cat > nginx-access-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nginx-access
spec:
  ports:
  - port: 80
  selector:
    app: nginx-stateful
EOF

kubectl apply -f nginx-access-service.yaml

# 获取Service的URL
minikube service nginx-access --url
```

使用获得的URL通过Web浏览器访问服务。由于Service负载均衡，您可能会看到不同Pod的响应。

### 6. 测试有序扩展和缩减

```bash
# 扩展StatefulSet
kubectl scale statefulset web --replicas=5

# 观察新Pod的创建顺序
kubectl get pods -l app=nginx-stateful -w

# 在新Pod中也创建index.html
for i in 3 4; do
  kubectl exec "web-$i" -- bash -c "echo 'Hello from web-$i' > /usr/share/nginx/html/index.html"
done

# 缩减StatefulSet
kubectl scale statefulset web --replicas=2

# 观察Pod的删除顺序（从高索引到低索引）
kubectl get pods -l app=nginx-stateful -w
```

### 7. 测试有序更新

```bash
# 修改StatefulSet的镜像版本
kubectl set image statefulset/web nginx=nginx:1.21

# 观察Pod的更新顺序
kubectl get pods -l app=nginx-stateful -w

# 检查更新状态
kubectl rollout status statefulset/web
```

### 8. 测试Pod失败和自动恢复

```bash
# 删除其中一个Pod，观察自动重建
kubectl delete pod web-1

# 查看StatefulSet确保Pod被重建
kubectl get pods -l app=nginx-stateful

# 验证重建的Pod保留了原来的PVC和数据
kubectl exec web-1 -- cat /usr/share/nginx/html/index.html
```

### 9. 使用配置StatefulSet的更新策略

```bash
# 编辑StatefulSet添加更新策略
kubectl edit statefulset web
```

在spec部分添加以下配置：

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    partition: 2
```

这将设置只有索引大于或等于2的Pod才会更新。

```bash
# 再次更新镜像版本
kubectl set image statefulset/web nginx=nginx:1.22

# 观察只有web-2及更高索引的Pod更新
kubectl get pods -l app=nginx-stateful -o wide
```

### 10. 部署一个真实的有状态应用：MySQL主从复制

现在让我们部署一个更复杂的有状态应用：MySQL主从复制集群。

```bash
# 创建ConfigMap存储MySQL配置
cat > mysql-configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  master.cnf: |
    [mysqld]
    log-bin
  slave.cnf: |
    [mysqld]
    super-read-only
EOF

kubectl apply -f mysql-configmap.yaml

# 创建Secret存储MySQL密码
kubectl create secret generic mysql-secret \
  --from-literal=mysql-root-password=rootpass \
  --from-literal=mysql-replication-password=replpass

# 创建MySQL Service
cat > mysql-services.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
  labels:
    app: mysql
spec:
  ports:
  - port: 3306
  clusterIP: None
  selector:
    app: mysql
EOF

kubectl apply -f mysql-services.yaml

# 创建MySQL StatefulSet
cat > mysql-statefulset.yaml << 'EOF'
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql-headless
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:5.7
        command:
        - bash
        - "-c"
        - |
          set -ex
          # 根据Pod序号决定是主节点还是从节点
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          # 添加唯一的server-id
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # 如果是0号Pod为主节点，否则为从节点
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/master.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/slave.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      - name: clone-mysql
        image: gcr.io/google-samples/xtrabackup:1.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # 仅在从节点上执行
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          if [[ $ordinal -eq 0 ]]; then
            # 主节点不需要克隆数据
            exit 0
          fi
          # 克隆主节点数据
          ncat --recv-only mysql-0.mysql-headless 3307 | xbstream -x -C /var/lib/mysql
          # 准备备份
          xtrabackup --prepare --target-dir=/var/lib/mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-root-password
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
      - name: xtrabackup
        image: gcr.io/google-samples/xtrabackup:1.0
        ports:
        - name: xtrabackup
          containerPort: 3307
        command:
        - bash
        - "-c"
        - |
          set -ex
          cd /var/lib/mysql
          
          # 确定自己是主节点还是从节点
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          
          # 从节点如果没有数据则等待数据传输并启动复制
          if [[ $ordinal -gt 0 ]]; then
            # 等待MySQL启动
            until mysql -h 127.0.0.1 -e "SELECT 1"; do sleep 1; done
            
            # 如果未配置复制，则配置复制
            echo "Checking if replication is setup..."
            if [[ $(mysql -h 127.0.0.1 -e "SHOW SLAVE STATUS\G" | grep Slave_IO_Running | wc -l) -eq 0 ]]; then
              echo "Setting up replication from mysql-0..."
              mysql -h 127.0.0.1 \
                    -e "CHANGE MASTER TO MASTER_HOST='mysql-0.mysql-headless', \
                        MASTER_USER='root', \
                        MASTER_PASSWORD='${MYSQL_ROOT_PASSWORD}', \
                        MASTER_LOG_FILE='mysql-bin.000001', \
                        MASTER_LOG_POS=4; \
                        START SLAVE;"
            fi
          fi
          
          # 不断地进行备份，供其他节点克隆数据使用
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root --password=${MYSQL_ROOT_PASSWORD}"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-root-password
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard-stateful
      resources:
        requests:
          storage: 2Gi
EOF

kubectl apply -f mysql-statefulset.yaml

# 观察StatefulSet创建过程
kubectl get pods -l app=mysql -w
```

### 11. 验证MySQL主从复制设置

等待所有MySQL Pod运行后，验证主从复制是否正常工作：

```bash
# 在主库（mysql-0）创建测试数据库和表
kubectl exec -it mysql-0 -- mysql -uroot -prootpass -e "CREATE DATABASE test; USE test; CREATE TABLE messages (id INT AUTO_INCREMENT PRIMARY KEY, message TEXT); INSERT INTO messages (message) VALUES ('Hello from StatefulSet master');"

# 验证数据已复制到从库
kubectl exec -it mysql-1 -- mysql -uroot -prootpass -e "SELECT * FROM test.messages;"
kubectl exec -it mysql-2 -- mysql -uroot -prootpass -e "SELECT * FROM test.messages;"

# 检查复制状态
kubectl exec -it mysql-1 -- mysql -uroot -prootpass -e "SHOW SLAVE STATUS\G"
```

### 12. 测试MySQL集群高可用性

```bash
# 模拟从节点故障
kubectl delete pod mysql-2

# 观察自动重建
kubectl get pods -l app=mysql -w

# 验证重建后的从节点能否正确同步数据
kubectl exec -it mysql-2 -- mysql -uroot -prootpass -e "SELECT * FROM test.messages;"

# 在主节点添加新数据
kubectl exec -it mysql-0 -- mysql -uroot -prootpass -e "USE test; INSERT INTO messages (message) VALUES ('Another message after pod recovery');"

# 验证数据同步到所有从节点
kubectl exec -it mysql-1 -- mysql -uroot -prootpass -e "SELECT * FROM test.messages;"
kubectl exec -it mysql-2 -- mysql -uroot -prootpass -e "SELECT * FROM test.messages;"
```

### 13. 清理资源

```bash
# 删除MySQL资源
kubectl delete statefulset mysql
kubectl delete service mysql mysql-headless
kubectl delete configmap mysql-config
kubectl delete secret mysql-secret
kubectl delete pvc -l app=mysql

# 删除Nginx StatefulSet资源
kubectl delete statefulset web
kubectl delete service nginx-headless nginx-access
kubectl delete pvc -l app=nginx-stateful

# 删除StorageClass
kubectl delete storageclass standard-stateful
```

## 思考问题

1. 相比Deployment，StatefulSet有哪些独特的特性？什么类型的应用适合使用StatefulSet部署？
2. StatefulSet中的Pod是如何保持稳定网络标识的？这一特性为何对分布式系统很重要？
3. volumeClaimTemplates在StatefulSet中的作用是什么？它如何简化有状态应用的部署？
4. 在StatefulSet中，Pod的创建、删除和更新顺序为什么很重要？
5. 分布式数据库系统（如MySQL集群、Cassandra、MongoDB等）使用StatefulSet部署时需要考虑哪些特殊因素？

## 扩展练习

1. 部署一个三节点的Redis Sentinel集群，使用StatefulSet确保高可用性
2. 探索StatefulSet的不同更新策略，并比较它们的行为和适用场景
3. 实现一个带有持久存储和自动扩缩容的Elasticsearch集群
4. 尝试部署Kafka集群，处理有状态应用中的复杂依赖关系
5. 研究StatefulSet与高级网络功能（如Headless Service和PodDisruptionBudget）的结合使用 