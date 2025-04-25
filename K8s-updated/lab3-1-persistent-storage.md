# 实验3.1：Kubernetes持久化存储 (PV/PVC)

## 目标
学习如何在Kubernetes中使用持久卷（PersistentVolume，PV）和持久卷声明（PersistentVolumeClaim，PVC）管理有状态应用程序的数据，掌握不同的存储类型和访问模式。

## 准备工作
- 完成实验2.1到2.5
- 运行中的Minikube集群
- 熟悉基本的kubectl命令

## Kubernetes存储抽象概述

Kubernetes提供了多层存储抽象，使应用程序能够独立于底层存储基础设施：

1. **持久卷（PersistentVolume，PV）**：集群中的存储资源，由管理员预先配置或动态配置
2. **持久卷声明（PersistentVolumeClaim，PVC）**：用户对存储的请求
3. **存储类（StorageClass）**：定义PV的配置参数和供应方式
4. **卷（Volume）**：Pod中容器可访问的目录，可以挂载到容器的文件系统上

## 步骤

### 1. 了解Minikube中的存储选项

```bash
# 查看Minikube中可用的StorageClass
kubectl get storageclass

# 查看默认StorageClass的详细信息
kubectl describe storageclass standard
```

### 2. 创建静态持久卷（Static PV）

```bash
# 创建持久卷
cat > static-pv.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: static-data-pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
EOF

kubectl apply -f static-pv.yaml

# 查看创建的PV
kubectl get pv
kubectl describe pv static-data-pv
```

### 3. 创建持久卷声明（PVC）

```bash
# 创建PVC以请求存储
cat > static-pvc.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: static-data-pvc
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

kubectl apply -f static-pvc.yaml

# 检查PVC状态
kubectl get pvc
kubectl describe pvc static-data-pvc
```

### 4. 在Pod中使用PVC

```bash
# 创建使用PVC的Pod
cat > pv-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pv-pod
spec:
  volumes:
    - name: data-volume
      persistentVolumeClaim:
        claimName: static-data-pvc
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: data-volume
EOF

kubectl apply -f pv-pod.yaml

# 验证Pod是否成功运行
kubectl get pod pv-pod
```

### 5. 将数据写入持久卷

```bash
# 在卷中创建一些数据
kubectl exec pv-pod -- sh -c "echo 'Hello from Kubernetes PV' > /usr/share/nginx/html/index.html"

# 验证数据已被写入
kubectl exec pv-pod -- cat /usr/share/nginx/html/index.html

# 创建服务以访问Pod
kubectl expose pod pv-pod --name=pv-pod-service --port=80 --type=NodePort

# 获取服务URL
minikube service pv-pod-service --url
```

在浏览器中访问提供的URL，应该能看到"Hello from Kubernetes PV"。

### 6. 验证数据持久性

```bash
# 删除并重新创建Pod
kubectl delete pod pv-pod
kubectl apply -f pv-pod.yaml

# 等待Pod重新启动
sleep 5

# 验证数据是否仍然存在
kubectl exec pv-pod -- cat /usr/share/nginx/html/index.html
```

### 7. 使用动态存储配置

```bash
# 创建使用默认StorageClass的PVC
cat > dynamic-pvc.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

kubectl apply -f dynamic-pvc.yaml

# 检查动态创建的PVC
kubectl get pvc dynamic-data-pvc
```

### 8. 部署有状态应用：MySQL数据库

```bash
# 创建MySQL密码的Secret
kubectl create secret generic mysql-pass --from-literal=password=mysqlpassword

# 创建MySQL Deployment和Service
cat > mysql-deployment.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.7
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: dynamic-data-pvc
EOF

kubectl apply -f mysql-deployment.yaml

# 检查MySQL是否运行
kubectl get pods -l app=mysql
```

### 9. 向MySQL数据库添加数据

```bash
# 等待MySQL Pod运行
sleep 30
MYSQL_POD=$(kubectl get pod -l app=mysql -o jsonpath='{.items[0].metadata.name}')

# 创建测试数据库和表
kubectl exec -it $MYSQL_POD -- mysql -u root -pmysqlpassword -e "CREATE DATABASE test; USE test; CREATE TABLE messages (id INT AUTO_INCREMENT PRIMARY KEY, message TEXT); INSERT INTO messages (message) VALUES ('This data is stored in a persistent volume');"

# 查询数据验证
kubectl exec -it $MYSQL_POD -- mysql -u root -pmysqlpassword -e "SELECT * FROM test.messages;"
```

### 10. 验证MySQL数据持久性

```bash
# 删除并重新创建MySQL Deployment
kubectl delete deployment mysql
kubectl apply -f mysql-deployment.yaml

# 等待新的MySQL Pod运行
sleep 30
MYSQL_POD=$(kubectl get pod -l app=mysql -o jsonpath='{.items[0].metadata.name}')

# 验证数据是否仍然存在
kubectl exec -it $MYSQL_POD -- mysql -u root -pmysqlpassword -e "SELECT * FROM test.messages;"
```

### 11. 探索不同的访问模式

```bash
# 创建具有不同访问模式的PV
cat > different-access-modes-pv.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-rwo
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data-rwo"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-rom
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadOnlyMany
  hostPath:
    path: "/mnt/data-rom"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-rwm
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/mnt/data-rwm"
EOF

kubectl apply -f different-access-modes-pv.yaml

# 检查不同访问模式的PV
kubectl get pv | grep manual
```

### 12. 创建自定义存储类

```bash
# 创建自定义StorageClass
cat > custom-storageclass.yaml << 'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: k8s.io/minikube-hostpath
parameters:
  type: pd-ssd
reclaimPolicy: Delete
volumeBindingMode: Immediate
EOF

kubectl apply -f custom-storageclass.yaml

# 使用自定义StorageClass创建PVC
cat > fast-pvc.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast
  resources:
    requests:
      storage: 1Gi
EOF

kubectl apply -f fast-pvc.yaml

# 检查是否创建成功
kubectl get pvc fast-pvc
```

### 13. 了解回收策略

```bash
# 检查当前PV的回收策略
kubectl get pv -o custom-columns=NAME:.metadata.name,RECLAIM:.spec.persistentVolumeReclaimPolicy

# 创建具有不同回收策略的PV
cat > reclaim-policy-pv.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-retain
spec:
  storageClassName: manual-retain
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/mnt/data-retain"
EOF

kubectl apply -f reclaim-policy-pv.yaml

# 检查回收策略
kubectl get pv pv-retain -o custom-columns=NAME:.metadata.name,RECLAIM:.spec.persistentVolumeReclaimPolicy
```

### 14. 实验回收策略效果

```bash
# 创建对应的PVC
cat > retain-pvc.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: retain-pvc
spec:
  storageClassName: manual-retain
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

kubectl apply -f retain-pvc.yaml

# 检查绑定状态
kubectl get pvc retain-pvc
kubectl get pv pv-retain

# 删除PVC观察PV状态
kubectl delete pvc retain-pvc

# 检查PV状态（应该是Released而不是Available）
kubectl get pv pv-retain
```

### 15. 清理资源

```bash
# 删除所有创建的资源
kubectl delete service pv-pod-service mysql
kubectl delete pod pv-pod
kubectl delete deployment mysql
kubectl delete pvc static-data-pvc dynamic-data-pvc fast-pvc
kubectl delete pv static-data-pv pv-rwo pv-rom pv-rwm pv-retain
kubectl delete storageclass fast
kubectl delete secret mysql-pass
```

## 思考问题

1. PV和PVC的绑定机制是什么？什么决定了特定的PVC会绑定到哪个PV？
2. 不同的存储访问模式（ReadWriteOnce、ReadOnlyMany、ReadWriteMany）各有什么特点和适用场景？
3. 动态存储配置与静态存储配置有什么区别？各自在什么场景下更适合？
4. PV的不同回收策略（Retain、Delete、Recycle）对PV生命周期有什么影响？
5. 在实际生产环境中，应如何规划和管理Kubernetes集群的存储资源？

## 扩展练习

1. 在Minikube中配置并使用NFS作为存储后端
2. 实现一个StatefulSet应用（如Redis集群），每个Pod使用独立的持久存储
3. 研究并实现存储快照和备份策略
4. 探索CSI（容器存储接口）驱动程序及其在Kubernetes中的使用
5. 实现一个场景，在不同命名空间中的多个Pod之间共享同一个只读存储卷 