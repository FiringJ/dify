# Kubernetes部署

<cite>
**本文档中引用的文件**  
- [docker-compose.yaml](file://docker/docker-compose.yaml)
- [docker-compose.middleware.yaml](file://docker/docker-compose.middleware.yaml)
- [docker-compose-template.yaml](file://docker/docker-compose-template.yaml)
- [tidb/docker-compose.yaml](file://docker/tidb/docker-compose.yaml)
</cite>

## 目录
1. [简介](#简介)
2. [从docker-compose迁移至Kubernetes](#从docker-compose迁移至kubernetes)
3. [Helm图表使用方法](#helm图表使用方法)
4. [持久化存储配置](#持久化存储配置)
5. [服务发现与负载均衡](#服务发现与负载均衡)
6. [高可用性部署建议](#高可用性部署建议)
7. [滚动更新策略](#滚动更新策略)

## 简介

本指南旨在帮助用户将现有的Dify应用从Docker Compose环境迁移到Kubernetes集群。通过详细说明如何将docker-compose中的服务转换为Kubernetes资源对象，包括Deployment、Service、ConfigMap和Secret，并结合Helm图表进行统一管理，实现高效、可扩展的部署方案。

**Section sources**
- [docker-compose.yaml](file://docker/docker-compose.yaml)
- [docker-compose.middleware.yaml](file://docker/docker-compose.middleware.yaml)

## 从docker-compose迁移至Kubernetes

### 服务到Deployment的转换

在`docker-compose.yaml`中定义的每个服务（如api、worker、web等）应被转换为对应的Kubernetes Deployment资源。例如，`api`服务可以转换为如下Deployment：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dify-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dify-api
  template:
    metadata:
      labels:
        app: dify-api
    spec:
      containers:
      - name: api
        image: langgenius/dify-api:1.8.1
        envFrom:
        - configMapRef:
            name: dify-config
        - secretRef:
            name: dify-secrets
        volumeMounts:
        - name: storage-volume
          mountPath: /app/api/storage
      volumes:
      - name: storage-volume
        persistentVolumeClaim:
          claimName: dify-storage-claim
```

### 配置到ConfigMap和Secret的分离

所有环境变量应根据其敏感性分别存入ConfigMap或Secret。非敏感配置（如LOG_LEVEL、UPLOAD_FILE_SIZE_LIMIT）放入ConfigMap，而敏感信息（如SECRET_KEY、DB_PASSWORD）则存储在Secret中。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dify-config
data:
  LOG_LEVEL: "INFO"
  UPLOAD_FILE_SIZE_LIMIT: "15"
---
apiVersion: v1
kind: Secret
metadata:
  name: dify-secrets
type: Opaque
data:
  SECRET_KEY: c2t-OWY3M3MzbGpUWFZjTVQzQmxiM2xqVHF0c0tpR0hYVmNNVDNCcGtGSktMN1U=
  DB_PASSWORD: ZGlmeWFpMTIzNDU2
```

### 网络与服务暴露

使用Kubernetes Service资源暴露内部服务。前端服务通过NodePort或LoadBalancer类型暴露，后端服务使用ClusterIP类型仅限内部访问。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: dify-web
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
  selector:
    app: dify-web
```

**Section sources**
- [docker-compose.yaml](file://docker/docker-compose.yaml)
- [docker-compose-template.yaml](file://docker/docker-compose-template.yaml)

## Helm图表使用方法

### values.yaml配置文件参数说明

Helm图表的`values.yaml`文件用于定义可配置参数，便于不同环境下的部署定制。主要参数包括：

- **镜像配置**：指定各组件的镜像名称与标签
- **副本数量**：控制Deployment的replicas值
- **资源限制**：设置CPU和内存请求与限制
- **持久化配置**：定义PVC的大小与存储类
- **环境变量**：覆盖默认的ConfigMap与Secret内容

### 自定义选项

用户可通过`--set`参数或自定义`values.yaml`文件覆盖默认值。例如：

```bash
helm install dify ./dify-chart \
  --set api.replicaCount=3 \
  --set postgresql.persistence.size=50Gi \
  --set global.storageClass=ssd
```

此外，支持通过`extraEnv`字段添加额外环境变量，或使用`nodeSelector`将Pod调度到特定节点。

**Section sources**
- [docker-compose.yaml](file://docker/docker-compose.yaml)
- [docker-compose.middleware.yaml](file://docker/docker-compose.middleware.yaml)

## 持久化存储配置

### 向量数据库PV/PVC设置

对于向量数据库（如Weaviate、Qdrant），需创建专用的PersistentVolumeClaim以确保数据持久性。示例配置如下：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: weaviate-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: fast-ssd
```

该PVC将挂载至Weaviate容器的`/var/lib/weaviate`路径。

### 文件存储PV/PVC设置

文件存储服务（如MinIO、S3兼容存储）同样需要独立的PVC。建议使用高性能存储类并设置合理的容量：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-data-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Gi
  storageClassName: nfs-shared
```

**Section sources**
- [docker-compose.middleware.yaml](file://docker/docker-compose.middleware.yaml)
- [docker-compose-template.yaml](file://docker/docker-compose-template.yaml)

## 服务发现与负载均衡

### 服务发现实现方式

Kubernetes内置DNS服务实现服务发现。Pod可通过服务名称直接访问其他服务。例如，`api`服务可通过`http://worker.default.svc.cluster.local`访问worker服务。

### Ingress控制器配置

使用Ingress资源统一管理外部HTTP(S)流量路由。结合Nginx Ingress Controller实现路径路由与TLS终止：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dify-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
spec:
  tls:
  - hosts:
    - dify.example.com
    secretName: dify-tls-secret
  rules:
  - host: dify.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dify-web
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: dify-api
            port:
              number: 5001
```

**Section sources**
- [docker-compose.yaml](file://docker/docker-compose.yaml)
- [docker-compose.middleware.yaml](file://docker/docker-compose.middleware.yaml)

## 高可用性部署建议

为实现高可用性，建议采取以下措施：

- **多副本部署**：关键服务（api、worker、db）至少部署3个副本
- **跨节点调度**：使用podAntiAffinity确保同一服务的Pod分布在不同节点
- **有状态服务保护**：对数据库启用PodDisruptionBudget防止意外中断
- **监控与告警**：集成Prometheus与Alertmanager实现实时监控
- **备份策略**：定期备份数据库与向量存储数据至远程位置

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: postgres
```

**Section sources**
- [docker-compose.yaml](file://docker/docker-compose.yaml)
- [docker-compose.middleware.yaml](file://docker/docker-compose.middleware.yaml)

## 滚动更新策略

通过配置Deployment的更新策略实现无缝升级：

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  minReadySeconds: 30
  revisionHistoryLimit: 5
```

此配置确保在更新过程中始终有足够数量的Pod处于就绪状态，避免服务中断。结合就绪探针（readinessProbe）确保新Pod完全初始化后再接收流量。

**Section sources**
- [docker-compose.yaml](file://docker/docker-compose.yaml)
- [docker-compose-template.yaml](file://docker/docker-compose-template.yaml)