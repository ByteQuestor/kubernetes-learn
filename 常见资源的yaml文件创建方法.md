# 常见资源的yaml文件创建方法

## Deployment

```shell
kubectl create deployment gitlab \
--image=gitlab/gitlab-ce:latest \
--port=80 \
# 试运行，不实际创建资源
--dry-run -oyaml > gitlab.yaml
```

## Service

```shell
kubectl create service nodeport gitlab \
--tcp=80:80 \
--dry-run \
-oyaml >> gitlab.yaml
```

## ConfigMap

```shell
kubectl create configmap gitlab-config \
  --from-file=path/to/your/config/file \
  --dry-run -o yaml >> gitlab.yaml
```

将 `ConfigMap` 与 `GitLab` 部署中的容器挂载

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gitlab-config
data:
  config_file: |
    # contents of your config file

```

## Secret

```shell
kubectl create secret generic gitlab-secret \
  --from-literal=gitlab-password=your_password \
  --dry-run -o yaml >> gitlab.yaml
```

## PersistentVolume（PV）

```shell
kubectl create pv gitlab-pv \
  --capacity=10Gi \
  --access-modes=ReadWriteOnce \
  --persistent-volume-reclaim-policy=Retain \
  --dry-run -o yaml >> gitlab.yaml
```

## PersistentVolumeClaim（PVC）

```shell
kubectl create pvc gitlab-pvc \
  --access-modes=ReadWriteOnce \
  --resources=requests.storage=10Gi \
  --dry-run -o yaml >> gitlab.yaml
```

## 挂载

```yaml
volumeMounts:
  - name: gitlab-data
    mountPath: /var/opt/gitlab
volumes:
  - name: gitlab-data
    persistentVolumeClaim:
      claimName: gitlab-pvc
```

## Ingress

```shell
kubectl create ingress gitlab-ingress \
  --rule="gitlab.example.com/*=gitlab:80" \
  --dry-run -o yaml >> gitlab.yaml
```

# 完整示例

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitlab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitlab
  template:
    metadata:
      labels:
        app: gitlab
    spec:
      containers:
        - name: gitlab
          image: gitlab/gitlab-ce:latest
          ports:
            - containerPort: 80
          volumeMounts:
            - name: gitlab-data
              mountPath: /var/opt/gitlab
          envFrom:
            - secretRef:
                name: gitlab-secret
      volumes:
        - name: gitlab-data
          persistentVolumeClaim:
            claimName: gitlab-pvc

---
apiVersion: v1
kind: Service
metadata:
  name: gitlab
spec:
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: gitlab
  type: NodePort

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gitlab-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/gitlab

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: gitlab-config
data:
  gitlab.rb: |
    # GitLab configuration
    external_url 'http://gitlab.example.com'

---
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-secret
type: Opaque
data:
  gitlab-password: <base64-encoded-password>

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitlab-ingress
spec:
  rules:
    - host: gitlab.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: gitlab
                port:
                  number: 80
```

