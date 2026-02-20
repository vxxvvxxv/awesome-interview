# Kubernetes (K8s)

Полезные ссылки:
- [Официальная документация](https://kubernetes.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

---

## Что такое Kubernetes?

**Kubernetes** (K8s) — система оркестрации контейнеров. Автоматизирует деплой, масштабирование и управление контейнеризованными приложениями.

**Решает проблемы:**
- Где запустить контейнеры (планирование на ноды).
- Что делать если контейнер упал (self-healing).
- Как масштабировать под нагрузку.
- Как обновлять без даунтайма.

## Архитектура кластера

```
┌──────────────────────────────────────────┐
│              Control Plane               │
│  API Server │ etcd │ Scheduler │ CM      │
└──────────────────────────────────────────┘
           │               │
    ┌──────┴───────────────┴──────┐
┌───▼────────────┐   ┌────────────▼───┐
│    Node 1      │   │    Node 2      │
│ kubelet / kube-proxy               │
│  [Pod1] [Pod2] │   │  [Pod3] [Pod4] │
└────────────────┘   └────────────────┘
```

### Компоненты Control Plane

| Компонент | Роль |
|-----------|------|
| **API Server** | Единая точка входа (REST API). Всё общение через него. |
| **etcd** | Распределённый key-value store — хранит всё состояние кластера. |
| **Scheduler** | Выбирает ноду для нового Pod на основе ресурсов. |
| **Controller Manager** | Reconciliation loop — следит чтобы состояние = желаемому. |

### Компоненты Node

| Компонент | Роль |
|-----------|------|
| **kubelet** | Агент на каждой ноде. Запускает и следит за Pod. |
| **kube-proxy** | Сетевые правила (iptables/ipvs) для Service. |
| **Container Runtime** | containerd / CRI-O — запуск контейнеров. |

## Основные объекты

### Pod

Минимальная единица деплоя. Содержит один или несколько контейнеров:
- Делят сетевой namespace (один IP).
- Делят volumes.
- Запускаются и останавливаются вместе.

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
    ports:
    - containerPort: 8080
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"       # 0.25 CPU ядра
      limits:
        memory: "128Mi"
        cpu: "500m"
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 30
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
```

**Liveness vs Readiness Probe:**
- **Liveness**: живой ли? → если нет, перезапустить Pod.
- **Readiness**: готов ли принимать трафик? → если нет, убрать из балансировщика.

### Deployment

Управляет ReplicaSet → управляет Pod. Обеспечивает rolling update, rollback.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # максимум недоступных Pod при обновлении
      maxSurge: 1          # максимум лишних Pod при обновлении
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
```

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/myapp
kubectl rollout undo deployment/myapp      # откат
kubectl rollout history deployment/myapp
kubectl scale deployment myapp --replicas=5
```

### Service

Стабильный DNS-адрес и балансировка нагрузки для набора Pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp       # находит Pod по метке
  ports:
  - port: 80         # порт Service
    targetPort: 8080 # порт контейнера
  type: ClusterIP
```

**Типы Service:**

| Тип | Описание |
|-----|---------|
| `ClusterIP` | Только внутри кластера (default) |
| `NodePort` | Доступен через порт ноды (30000–32767) |
| `LoadBalancer` | Внешний load balancer (облако) |
| `ExternalName` | DNS-alias на внешний ресурс |

### Ingress

HTTP/HTTPS маршрутизация снаружи в Service:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
  tls:
  - hosts:
    - api.example.com
    secretName: tls-secret
```

### ConfigMap и Secret

```yaml
# ConfigMap — несекретные конфиги
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"

# Secret — секретные данные (base64)
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
type: Opaque
data:
  database-url: cG9zdGdyZXM6Ly8...
```

```yaml
# Использование в Pod
spec:
  containers:
  - name: myapp
    envFrom:
    - configMapRef:
        name: myapp-config
    - secretRef:
        name: myapp-secrets
```

## HPA — Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

## StatefulSet vs Deployment

| | Deployment | StatefulSet |
|-|------------|-------------|
| Pod имена | Случайные (myapp-abc) | Стабильные (myapp-0, myapp-1) |
| Storage | Общий | Уникальный PVC на каждый Pod |
| Порядок запуска | Параллельный | Последовательный |
| Применение | Stateless сервисы | БД, Kafka, ZooKeeper |

## PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

**Access Modes:**
- `ReadWriteOnce` (RWO) — один Pod.
- `ReadOnlyMany` (ROX) — много Pod на чтение.
- `ReadWriteMany` (RWX) — много Pod на чтение и запись (NFS, CephFS).

## kubectl — основные команды

```bash
# Просмотр
kubectl get pods -o wide
kubectl get all -n production
kubectl describe pod myapp-xxx
kubectl logs -f myapp-xxx

# Управление
kubectl apply -f manifest.yaml
kubectl delete pod myapp-xxx
kubectl exec -it myapp-xxx -- sh
kubectl port-forward pod/myapp-xxx 8080:8080

# Rollout
kubectl rollout status deployment/myapp
kubectl rollout undo deployment/myapp
kubectl set image deployment/myapp myapp=myapp:2.0.0

# Метрики
kubectl top pods
kubectl top nodes
```

## Что спрашивают на собеседованиях

1. **Pod vs Deployment vs ReplicaSet?** → Pod — один экземпляр; RS — поддерживает N реплик; Deployment — управляет RS + rolling update.
2. **Liveness vs Readiness?** → liveness: перезапустить; readiness: убрать из балансировщика.
3. **Как работает rolling update?** → постепенно заменяет Pod (maxUnavailable + maxSurge).
4. **Как Service находит Pod?** → label selector.
5. **Что такое etcd?** → key-value store всего состояния кластера.
6. **Зачем requests и limits?** → requests — гарантия ресурсов для Scheduler; limits — жёсткое ограничение.
7. **Что такое Namespace?** → логическая изоляция внутри кластера (dev, staging, prod).
8. **Разница ClusterIP / NodePort / LoadBalancer?** → область доступности.
