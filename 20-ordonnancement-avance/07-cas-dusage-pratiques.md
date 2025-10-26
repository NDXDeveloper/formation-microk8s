🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.7 Cas d'Usage Pratiques

## Introduction

Nous avons exploré individuellement les différents mécanismes d'ordonnancement de Kubernetes :
- **Scheduler** : Le cerveau qui prend les décisions
- **Node Selectors** : Direction simple vers des nœuds spécifiques
- **Node Affinity** : Règles avancées pour le placement sur les nœuds
- **Pod Affinity/Anti-Affinity** : Relations entre Pods
- **Taints/Tolerations** : Restriction d'accès aux nœuds
- **Topology Spread Constraints** : Distribution uniforme

Mais la vraie puissance vient de la **combinaison** de ces mécanismes. Dans cette section, nous allons voir des cas d'usage pratiques et réalistes qui montrent comment orchestrer intelligemment vos applications.

---

## Vue d'Ensemble des Mécanismes

### Tableau de Décision Rapide

| Besoin | Mécanisme Recommandé |
|--------|---------------------|
| "Ce Pod doit aller sur un type de nœud spécifique" | Node Selector ou Node Affinity |
| "Ce Pod doit être proche d'autres Pods" | Pod Affinity |
| "Ce Pod doit être éloigné d'autres Pods" | Pod Anti-Affinity ou Topology Spread |
| "Réserver un nœud pour un usage spécial" | Taints + Tolerations |
| "Distribuer uniformément mes Pods" | Topology Spread Constraints |
| "Haute disponibilité garantie" | Topology Spread + Node Affinity |
| "Isolation stricte" | Taints + Node Affinity |

---

## Cas d'Usage 1 : Application Web Classique (3 Tiers)

### Architecture

```
┌─────────────────────────────────────────────┐
│              Load Balancer                  │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│  Frontend (NGINX) - 6 réplicas              │
│  - Distribution uniforme par zone           │
│  - Nœuds standards                          │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│  Backend API - 9 réplicas                   │
│  - Distribution uniforme                    │
│  - Proche du cache pour performance         │
│  - Nœuds avec plus de CPU                   │
└─────────────────────────────────────────────┘
                    ↓
┌────────────────────┬────────────────────────┐
│  Redis Cache       │  PostgreSQL            │
│  - 3 réplicas      │  - 3 instances HA      │
│  - Une par zone    │  - Une par zone        │
│  - SSD requis      │  - SSD + haute mémoire │
└────────────────────┴────────────────────────┘
```

### Configuration des Nœuds

```bash
# Zone A - Nœuds standards
kubectl label nodes node1 topology.kubernetes.io/zone=zone-a
kubectl label nodes node1 node-type=standard cpu-class=medium

kubectl label nodes node2 topology.kubernetes.io/zone=zone-a
kubectl label nodes node2 node-type=standard cpu-class=medium

# Zone B - Nœuds standards
kubectl label nodes node3 topology.kubernetes.io/zone=zone-b
kubectl label nodes node3 node-type=standard cpu-class=medium

kubectl label nodes node4 topology.kubernetes.io/zone=zone-b
kubectl label nodes node4 node-type=standard cpu-class=medium

# Zone C - Nœuds pour données (SSD + haute mémoire)
kubectl label nodes node5 topology.kubernetes.io/zone=zone-c
kubectl label nodes node5 node-type=data disktype=ssd memory-class=high

kubectl label nodes node6 topology.kubernetes.io/zone=zone-c
kubectl label nodes node6 node-type=data disktype=ssd memory-class=high

# Taints sur les nœuds data (réservés)
kubectl taint nodes node5 workload=data:NoSchedule
kubectl taint nodes node6 workload=data:NoSchedule
```

### 1. Déploiement Frontend

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: webapp
  labels:
    app: webapp
    tier: frontend
spec:
  replicas: 6
  selector:
    matchLabels:
      app: webapp
      tier: frontend
  template:
    metadata:
      labels:
        app: webapp
        tier: frontend
    spec:
      # Topology Spread : Distribution uniforme par zone
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: webapp
            tier: frontend
      # Node Affinity : Préfère les nœuds standards
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: node-type
                operator: In
                values:
                - standard
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
```

**Résultat attendu** :
```
Zone A: 2 Pods frontend (node1, node2)
Zone B: 2 Pods frontend (node3, node4)
Zone C: 2 Pods frontend (répartis si possible, sinon ailleurs)
```

### 2. Déploiement Backend API

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: webapp
  labels:
    app: webapp
    tier: backend
spec:
  replicas: 9
  selector:
    matchLabels:
      app: webapp
      tier: backend
  template:
    metadata:
      labels:
        app: webapp
        tier: backend
    spec:
      # Topology Spread : Distribution uniforme
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: webapp
            tier: backend
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: webapp
            tier: backend
      # Pod Affinity : Proche du cache pour performance
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: webapp
                  tier: cache
              topologyKey: kubernetes.io/hostname
        # Node Affinity : CPU moyen minimum
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: cpu-class
                operator: In
                values:
                - medium
                - high
      containers:
      - name: api
        image: webapp-api:v2.1.0
        ports:
        - containerPort: 8080
        env:
        - name: REDIS_HOST
          value: "redis-service.webapp.svc.cluster.local"
        - name: DB_HOST
          value: "postgres-service.webapp.svc.cluster.local"
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "1000m"
            memory: "2Gi"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

**Résultat attendu** :
```
3 Pods par zone, idéalement proches de Redis
Performance optimale grâce à la colocalisation avec le cache
```

### 3. Redis Cache

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: webapp
  labels:
    app: webapp
    tier: cache
spec:
  serviceName: redis
  replicas: 3
  selector:
    matchLabels:
      app: webapp
      tier: cache
  template:
    metadata:
      labels:
        app: webapp
        tier: cache
    spec:
      # Tolerations : Accès aux nœuds data
      tolerations:
      - key: "workload"
        operator: "Equal"
        value: "data"
        effect: "NoSchedule"
      # Topology Spread : Une instance par zone
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: webapp
            tier: cache
      # Pod Anti-Affinity : Instances sur nœuds différents
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: webapp
                tier: cache
            topologyKey: kubernetes.io/hostname
        # Node Affinity : SSD requis
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            cpu: "200m"
            memory: "512Mi"
          limits:
            cpu: "500m"
            memory: "1Gi"
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

### 4. PostgreSQL

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: webapp
  labels:
    app: webapp
    tier: database
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: webapp
      tier: database
  template:
    metadata:
      labels:
        app: webapp
        tier: database
    spec:
      # Tolerations : Accès aux nœuds data
      tolerations:
      - key: "workload"
        operator: "Equal"
        value: "data"
        effect: "NoSchedule"
      # Topology Spread : Distribution par zone
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: webapp
            tier: database
      # Pod Anti-Affinity : Instances séparées
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: webapp
                tier: database
            topologyKey: kubernetes.io/hostname
        # Node Affinity : SSD + Haute mémoire
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
              - key: memory-class
                operator: In
                values:
                - high
      containers:
      - name: postgres
        image: postgres:14
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: POSTGRES_DB
          value: webapp
        resources:
          requests:
            cpu: "1000m"
            memory: "4Gi"
          limits:
            cpu: "2000m"
            memory: "8Gi"
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 50Gi
```

### Services

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: webapp
spec:
  type: LoadBalancer
  selector:
    app: webapp
    tier: frontend
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: webapp
spec:
  type: ClusterIP
  selector:
    app: webapp
    tier: backend
  ports:
  - port: 8080
    targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: webapp
spec:
  type: ClusterIP
  clusterIP: None  # Headless service for StatefulSet
  selector:
    app: webapp
    tier: cache
  ports:
  - port: 6379
    targetPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: webapp
spec:
  type: ClusterIP
  clusterIP: None  # Headless service for StatefulSet
  selector:
    app: webapp
    tier: database
  ports:
  - port: 5432
    targetPort: 5432
```

### Résumé des Choix d'Ordonnancement

| Composant | Mécanismes Utilisés | Justification |
|-----------|---------------------|---------------|
| **Frontend** | Topology Spread, Node Affinity (preferred) | Distribution uniforme pour HA, préfère nœuds standards |
| **Backend** | Topology Spread, Pod Affinity, Node Affinity | Distribution + colocalisation avec cache + CPU suffisant |
| **Redis** | Taints/Tolerations, Topology Spread, Anti-Affinity, Node Affinity | Réservé sur nœuds data SSD, une par zone |
| **PostgreSQL** | Taints/Tolerations, Topology Spread, Anti-Affinity, Node Affinity | Réservé sur nœuds data SSD haute mémoire, HA stricte |

---

## Cas d'Usage 2 : Plateforme ML/AI

### Architecture

```
┌──────────────────────────────────────────────┐
│  Jupyter Notebooks - 2 réplicas              │
│  - Nœuds GPU requis                          │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│  Training Jobs - Variables                   │
│  - GPU requis, tolère interruptions          │
│  - Peut utiliser nœuds spot                  │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│  Model Serving API - 3 réplicas              │
│  - CPU uniquement                            │
│  - Haute disponibilité                       │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│  Data Storage (MinIO) - 3 instances          │
│  - SSD requis                                │
│  - Distribution par zone                     │
└──────────────────────────────────────────────┘
```

### Configuration des Nœuds

```bash
# Nœuds GPU (onéreux, réservés)
kubectl label nodes gpu-node1 hardware=gpu gpu-type=v100
kubectl taint nodes gpu-node1 hardware=gpu:NoSchedule
kubectl label nodes gpu-node2 hardware=gpu gpu-type=v100
kubectl taint nodes gpu-node2 hardware=gpu:NoSchedule

# Nœuds Spot (moins chers, peuvent être interrompus)
kubectl label nodes spot-node1 instance-type=spot
kubectl label nodes spot-node1 hardware=gpu gpu-type=t4
kubectl taint nodes spot-node1 spot=true:PreferNoSchedule

# Nœuds CPU standards
kubectl label nodes cpu-node1 hardware=cpu
kubectl label nodes cpu-node2 hardware=cpu
kubectl label nodes cpu-node3 hardware=cpu

# Nœuds storage
kubectl label nodes storage-node1 disktype=ssd topology.kubernetes.io/zone=zone-a
kubectl label nodes storage-node2 disktype=ssd topology.kubernetes.io/zone=zone-b
kubectl label nodes storage-node3 disktype=ssd topology.kubernetes.io/zone=zone-c
```

### 1. Jupyter Notebooks (Développement)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jupyter
  namespace: ml-platform
spec:
  replicas: 2
  selector:
    matchLabels:
      app: jupyter
  template:
    metadata:
      labels:
        app: jupyter
    spec:
      # Toleration : Accès aux GPU
      tolerations:
      - key: "hardware"
        operator: "Equal"
        value: "gpu"
        effect: "NoSchedule"
      # Node Affinity : GPU requis
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: hardware
                operator: In
                values:
                - gpu
              - key: gpu-type
                operator: In
                values:
                - v100  # GPU premium uniquement
      containers:
      - name: jupyter
        image: jupyter/tensorflow-notebook:latest
        ports:
        - containerPort: 8888
        resources:
          requests:
            cpu: "2000m"
            memory: "8Gi"
            nvidia.com/gpu: 1
          limits:
            cpu: "4000m"
            memory: "16Gi"
            nvidia.com/gpu: 1
```

### 2. Training Jobs (Batch)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: model-training
  namespace: ml-platform
spec:
  parallelism: 3
  completions: 3
  template:
    metadata:
      labels:
        app: training
        job-type: batch
    spec:
      restartPolicy: OnFailure
      # Tolerations : Peut utiliser GPU standard ET spot
      tolerations:
      - key: "hardware"
        operator: "Equal"
        value: "gpu"
        effect: "NoSchedule"
      - key: "spot"
        operator: "Equal"
        value: "true"
        effect: "PreferNoSchedule"
      # Node Affinity : GPU requis, préfère spot pour économiser
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            # Option 1 : GPU standard
            - matchExpressions:
              - key: hardware
                operator: In
                values:
                - gpu
          preferredDuringSchedulingIgnoredDuringExecution:
          # Préfère les instances spot (moins chères)
          - weight: 100
            preference:
              matchExpressions:
              - key: instance-type
                operator: In
                values:
                - spot
      containers:
      - name: trainer
        image: ml-training:latest
        command: ["python", "train.py"]
        resources:
          requests:
            cpu: "4000m"
            memory: "16Gi"
            nvidia.com/gpu: 1
          limits:
            nvidia.com/gpu: 1
```

### 3. Model Serving API

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: model-serving
  namespace: ml-platform
spec:
  replicas: 3
  selector:
    matchLabels:
      app: model-serving
  template:
    metadata:
      labels:
        app: model-serving
    spec:
      # Topology Spread : Distribution pour HA
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: model-serving
      # Node Affinity : CPU uniquement (pas de GPU gaspillé)
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: hardware
                operator: In
                values:
                - cpu
      containers:
      - name: api
        image: model-serving-api:v1
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "1000m"
            memory: "2Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
```

### 4. Data Storage (MinIO)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: minio
  namespace: ml-platform
spec:
  serviceName: minio
  replicas: 3
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      # Topology Spread : Une instance par zone
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: minio
      # Pod Anti-Affinity : Instances séparées
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: minio
            topologyKey: kubernetes.io/hostname
        # Node Affinity : SSD requis
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
      containers:
      - name: minio
        image: minio/minio:latest
        args:
        - server
        - /data
        ports:
        - containerPort: 9000
        resources:
          requests:
            cpu: "500m"
            memory: "2Gi"
          limits:
            cpu: "1000m"
            memory: "4Gi"
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 100Gi
```

### Résumé des Choix

| Composant | Stratégie | Économie vs Performance |
|-----------|-----------|------------------------|
| **Jupyter** | GPU premium uniquement | Performance (développement) |
| **Training** | GPU spot préféré | Économie (jobs batch) |
| **Serving** | CPU, distribution HA | Efficacité (pas de GPU gaspillé) |
| **Storage** | SSD, distribution géographique | Fiabilité + Performance |

---

## Cas d'Usage 3 : Multi-Tenant SaaS Platform

### Architecture

Plusieurs clients (tenants) sur la même infrastructure, avec isolation et garanties.

```
Tenant A (Premium)    Tenant B (Standard)    Tenant C (Free)
     ↓                      ↓                      ↓
Nœuds dédiés          Nœuds partagés         Nœuds best-effort
SSD + haute mémoire   Standards              Spot instances
```

### Configuration des Nœuds

```bash
# Nœuds Premium (dédiés, haute performance)
kubectl label nodes premium-node1 tier=premium disktype=nvme memory-class=high
kubectl taint nodes premium-node1 tier=premium:NoSchedule

kubectl label nodes premium-node2 tier=premium disktype=nvme memory-class=high
kubectl taint nodes premium-node2 tier=premium:NoSchedule

# Nœuds Standard (partagés)
kubectl label nodes standard-node1 tier=standard disktype=ssd
kubectl label nodes standard-node2 tier=standard disktype=ssd
kubectl label nodes standard-node3 tier=standard disktype=ssd

# Nœuds Free (best-effort, spot)
kubectl label nodes free-node1 tier=free instance-type=spot
kubectl taint nodes free-node1 tier=free:PreferNoSchedule
```

### 1. Application Tenant Premium

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tenant-a-app
  namespace: tenant-a
  labels:
    tenant: tenant-a
    tier: premium
spec:
  replicas: 3
  selector:
    matchLabels:
      tenant: tenant-a
      app: webapp
  template:
    metadata:
      labels:
        tenant: tenant-a
        app: webapp
        tier: premium
    spec:
      # Toleration : Accès aux nœuds premium
      tolerations:
      - key: "tier"
        operator: "Equal"
        value: "premium"
        effect: "NoSchedule"
      # Topology Spread : Distribution uniforme
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            tenant: tenant-a
      # Pod Anti-Affinity : Isolation des autres tenants
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          # Ne jamais être avec d'autres tenants
          - labelSelector:
              matchExpressions:
              - key: tenant
                operator: NotIn
                values:
                - tenant-a
            topologyKey: kubernetes.io/hostname
        # Node Affinity : Nœuds premium uniquement
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: tier
                operator: In
                values:
                - premium
              - key: disktype
                operator: In
                values:
                - nvme
      # Priority Class : Haute priorité
      priorityClassName: high-priority
      containers:
      - name: app
        image: webapp:premium
        resources:
          requests:
            cpu: "2000m"
            memory: "4Gi"
          limits:
            cpu: "4000m"
            memory: "8Gi"
```

### 2. Application Tenant Standard

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tenant-b-app
  namespace: tenant-b
  labels:
    tenant: tenant-b
    tier: standard
spec:
  replicas: 2
  selector:
    matchLabels:
      tenant: tenant-b
      app: webapp
  template:
    metadata:
      labels:
        tenant: tenant-b
        app: webapp
        tier: standard
    spec:
      # Topology Spread : Distribution équilibrée
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            tenant: tenant-b
      # Node Affinity : Nœuds standard
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: tier
                operator: In
                values:
                - standard
      # Priority Class : Priorité moyenne
      priorityClassName: medium-priority
      containers:
      - name: app
        image: webapp:standard
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "1000m"
            memory: "2Gi"
```

### 3. Application Tenant Free

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tenant-c-app
  namespace: tenant-c
  labels:
    tenant: tenant-c
    tier: free
spec:
  replicas: 1
  selector:
    matchLabels:
      tenant: tenant-c
      app: webapp
  template:
    metadata:
      labels:
        tenant: tenant-c
        app: webapp
        tier: free
    spec:
      # Toleration : Accepte les nœuds free
      tolerations:
      - key: "tier"
        operator: "Equal"
        value: "free"
        effect: "PreferNoSchedule"
      # Node Affinity : Préfère les nœuds free
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: tier
                operator: In
                values:
                - free
      # Priority Class : Basse priorité
      priorityClassName: low-priority
      containers:
      - name: app
        image: webapp:free
        resources:
          requests:
            cpu: "100m"
            memory: "256Mi"
          limits:
            cpu: "200m"
            memory: "512Mi"
```

### PriorityClasses

```yaml
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false
description: "High priority for premium tenants"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: medium-priority
value: 500
globalDefault: false
description: "Medium priority for standard tenants"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
description: "Low priority for free tenants"
```

### Résumé de l'Isolation

| Tenant | Nœuds | Isolation | Garanties |
|--------|-------|-----------|-----------|
| **Premium** | Dédiés + Taints | Stricte (Anti-Affinity) | Ressources garanties, haute performance |
| **Standard** | Partagés | Modérée | Ressources raisonnables |
| **Free** | Best-effort/Spot | Aucune | Aucune garantie, peut être évincé |

---

## Cas d'Usage 4 : Environnements Dev/Staging/Production

### Architecture par Environnement

```
Production        Staging          Development
    ↓                ↓                  ↓
Nœuds dédiés    Nœuds partagés    Nœuds partagés
Haute perf      Performance       Best-effort
Taints strict   Taints modéré     Pas de taints
```

### Configuration des Nœuds

```bash
# Production - Nœuds dédiés, haute qualité
kubectl label nodes prod-node1 environment=production disktype=ssd
kubectl taint nodes prod-node1 environment=production:NoSchedule

kubectl label nodes prod-node2 environment=production disktype=ssd
kubectl taint nodes prod-node2 environment=production:NoSchedule

# Staging - Nœuds dédiés, qualité moyenne
kubectl label nodes staging-node1 environment=staging
kubectl taint nodes staging-node1 environment=staging:PreferNoSchedule

# Dev - Nœuds partagés, best-effort
kubectl label nodes dev-node1 environment=development
kubectl label nodes dev-node2 environment=development
```

### Déploiement Multi-Environnement

```yaml
---
# Production
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: production
spec:
  replicas: 5
  selector:
    matchLabels:
      app: api
      env: production
  template:
    metadata:
      labels:
        app: api
        env: production
    spec:
      tolerations:
      - key: "environment"
        operator: "Equal"
        value: "production"
        effect: "NoSchedule"
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: api
            env: production
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: env
                operator: In
                values:
                - staging
                - development
            topologyKey: kubernetes.io/hostname
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: environment
                operator: In
                values:
                - production
              - key: disktype
                operator: In
                values:
                - ssd
      priorityClassName: production-priority
      containers:
      - name: api
        image: api:v2.1.0
        resources:
          requests:
            cpu: "1000m"
            memory: "2Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
---
# Staging
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: staging
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
      env: staging
  template:
    metadata:
      labels:
        app: api
        env: staging
    spec:
      tolerations:
      - key: "environment"
        operator: "Equal"
        value: "staging"
        effect: "PreferNoSchedule"
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  env: production
              topologyKey: kubernetes.io/hostname
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: environment
                operator: In
                values:
                - staging
      priorityClassName: staging-priority
      containers:
      - name: api
        image: api:v2.1.0-staging
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "1000m"
            memory: "2Gi"
---
# Development
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: development
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
      env: development
  template:
    metadata:
      labels:
        app: api
        env: development
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: environment
                operator: In
                values:
                - development
      priorityClassName: dev-priority
      containers:
      - name: api
        image: api:latest
        resources:
          requests:
            cpu: "100m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "1Gi"
```

---

## Bonnes Pratiques Générales

### 1. Commencez Simple, Complexifiez Progressivement

```yaml
# ✅ Phase 1 : Simple
nodeSelector:
  disktype: ssd

# ✅ Phase 2 : Plus sophistiqué si nécessaire
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values:
          - ssd
          - nvme

# ✅ Phase 3 : Complet si vraiment nécessaire
affinity:
  nodeAffinity: ...
  podAntiAffinity: ...
topologySpreadConstraints: ...
tolerations: ...
```

### 2. Documentez vos Choix

```yaml
metadata:
  name: critical-app
  annotations:
    scheduling-strategy: |
      - Node Affinity: SSD required for database performance
      - Topology Spread: maxSkew=1 for high availability across zones
      - Pod Anti-Affinity: Ensure replicas on different nodes
      - Tolerations: Access to dedicated database nodes
    last-reviewed: "2024-01-15"
    owner: "platform-team"
```

### 3. Utilisez des Valeurs Cohérentes

```yaml
# ✅ Bon : Labels et valeurs standardisés
labels:
  environment: production    # production, staging, development
  tier: backend             # frontend, backend, cache, database
  sensitivity: high         # high, medium, low
```

### 4. Préférez ScheduleAnyway sauf Nécessité

```yaml
# ✅ Flexible - Recommandé
topologySpreadConstraints:
- maxSkew: 1
  whenUnsatisfiable: ScheduleAnyway

# ⚠️ Strict - Uniquement si vraiment nécessaire
topologySpreadConstraints:
- maxSkew: 1
  whenUnsatisfiable: DoNotSchedule
```

### 5. Testez dans un Environnement Non-Critique

Avant de déployer en production :

```bash
# 1. Créer un namespace de test
kubectl create namespace scheduling-test

# 2. Déployer avec vos règles
kubectl apply -f deployment.yaml -n scheduling-test

# 3. Vérifier la distribution
kubectl get pods -n scheduling-test -o wide

# 4. Simuler une panne
kubectl drain node1 --ignore-daemonsets

# 5. Vérifier le rebalancement
kubectl get pods -n scheduling-test -o wide
```

### 6. Surveillez et Ajustez

```bash
# Pods en Pending (signe de contraintes trop strictes)
kubectl get pods --all-namespaces | grep Pending

# Distribution actuelle
kubectl get pods -l app=mon-app -o wide | \
  awk '{print $7}' | sort | uniq -c

# Événements de scheduling
kubectl get events --sort-by='.lastTimestamp' | \
  grep FailedScheduling
```

---

## Checklist de Décision

### Pour Chaque Application, Posez-vous :

**1. Haute Disponibilité ?**
- ✅ Oui → Topology Spread Constraints + Pod Anti-Affinity
- ❌ Non → Pas nécessaire

**2. Performance Critique ?**
- ✅ Oui → Node Affinity (SSD, GPU, haute mémoire) + Pod Affinity (colocalisation)
- ❌ Non → Node Selector simple suffit

**3. Isolation Nécessaire ?**
- ✅ Oui → Taints + Tolerations + Pod Anti-Affinity
- ❌ Non → Partage OK

**4. Coût Important ?**
- ✅ Oui → Utiliser Spot instances avec Tolerations appropriées
- ❌ Non → Nœuds standards

**5. Multi-Zone ?**
- ✅ Oui → Topology Spread avec topologyKey: zone
- ❌ Non → Topology Spread par hostname suffit

---

## Tableau Récapitulatif : Quand Utiliser Quoi

| Scénario | Solution Recommandée |
|----------|---------------------|
| **HA simple** | Topology Spread Constraints (maxSkew: 1) |
| **HA stricte** | Topology Spread + Pod Anti-Affinity |
| **Performance** | Node Affinity (SSD, GPU) + Pod Affinity (colocalisation) |
| **Isolation** | Taints + Tolerations |
| **Multi-tenant** | Namespaces + Taints + Node Affinity + Priority |
| **Économie** | Spot instances + Tolerations PreferNoSchedule |
| **Dev vs Prod** | Taints différents + Node Affinity |
| **Données critiques** | Node Affinity (SSD) + Topology Spread (zones) |

---

## Conclusion

L'ordonnancement avancé dans Kubernetes est comme un puzzle : chaque pièce (Node Selector, Affinity, Taints, Topology Spread) a son rôle, et c'est en les combinant intelligemment que vous créez une architecture robuste, performante et économique.

### Points Clés à Retenir

1. **Commencez simple** : Node Selector pour débuter
2. **Complexifiez si nécessaire** : Ajoutez Affinity, Taints, etc. selon vos besoins
3. **Documentez** : Expliquez vos choix pour l'équipe
4. **Testez** : Validez dans un environnement non-critique
5. **Surveillez** : Vérifiez que vos contraintes ne bloquent pas les déploiements
6. **Ajustez** : Évoluez vos stratégies avec l'expérience

### Hiérarchie de Complexité

```
Node Selector (simple)
    ↓
Node Affinity (flexible)
    ↓
+ Topology Spread (distribution)
    ↓
+ Pod Affinity/Anti-Affinity (relations)
    ↓
+ Taints/Tolerations (restrictions)
    ↓
Architecture Complète (tous ensemble)
```

En maîtrisant ces mécanismes et en les combinant judicieusement, vous pouvez orchestrer vos applications Kubernetes de manière optimale, en équilibrant haute disponibilité, performance, coût et isolation selon vos besoins spécifiques.

---

## Pour Aller Plus Loin

- **Expérimentez** : Créez votre propre lab MicroK8s multi-node
- **Mesurez** : Utilisez des outils de monitoring pour valider vos choix
- **Partagez** : Documentez vos patterns pour votre équipe
- **Évoluez** : Kubernetes évolue, restez à jour sur les nouvelles fonctionnalités

---

**Fin du chapitre 20 : Ordonnancement Avancé**

Vous avez maintenant une compréhension complète de l'ordonnancement dans Kubernetes, de la théorie du scheduler aux cas d'usage pratiques complexes. Ces connaissances vous permettront de concevoir et déployer des architectures Kubernetes sophistiquées et adaptées à vos besoins.

⏭️ [Multi-Node et Haute Disponibilité](/21-multi-node-et-haute-disponibilite/README.md)
