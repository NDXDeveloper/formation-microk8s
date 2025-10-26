🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 24.7 Architecture de Référence

## Introduction

Une architecture de référence est un modèle ou un blueprint qui définit la structure, les composants et les meilleures pratiques pour construire un système robuste et scalable. C'est comme un plan d'architecte pour une maison : il montre comment tous les éléments doivent s'assembler pour créer quelque chose de solide et fonctionnel.

Dans ce chapitre, nous allons explorer plusieurs architectures de référence pour différents cas d'usage avec MicroK8s. Ces architectures ont été éprouvées en production et vous serviront de guide pour vos propres projets.

## Qu'est-ce qu'une Architecture de Référence ?

### Définition

Une architecture de référence est un document ou un modèle qui décrit :

**Les composants** : Quelles technologies et services utiliser
**Les interactions** : Comment ces composants communiquent entre eux
**Les patterns** : Les modèles de conception à appliquer
**Les bonnes pratiques** : Comment configurer et sécuriser le système
**La scalabilité** : Comment le système peut grandir
**La résilience** : Comment gérer les pannes

### Pourquoi utiliser une Architecture de Référence ?

**Gain de temps** : Pas besoin de réinventer la roue, partez d'un modèle éprouvé.

**Éviter les erreurs** : Les architectures de référence intègrent les leçons apprises d'autres projets.

**Cohérence** : Toute l'équipe suit le même standard.

**Facilite les décisions** : Les choix technologiques sont déjà faits et justifiés.

**Scalabilité dès le départ** : L'architecture est pensée pour grandir.

**Documentation** : Fournit une référence pour les nouveaux membres de l'équipe.

### Types d'Architectures de Référence

Nous allons couvrir quatre architectures principales :

1. **Architecture Application Web Simple** : Site web ou application avec base de données
2. **Architecture Microservices** : Application distribuée avec plusieurs services
3. **Architecture Data Pipeline** : Traitement et analyse de données
4. **Architecture Full-Stack Production** : Solution complète prête pour la production

## Principes d'Architecture Cloud-Native

Avant de plonger dans les architectures spécifiques, comprenons les principes fondamentaux :

### Les 12 Facteurs (12-Factor App)

1. **Codebase** : Un repository par application
2. **Dépendances** : Déclaration explicite des dépendances
3. **Configuration** : Stockée dans l'environnement (ConfigMaps, Secrets)
4. **Services externes** : Traités comme des ressources attachées
5. **Build, Release, Run** : Séparation stricte des étapes
6. **Processus** : Applications stateless
7. **Port binding** : Services exposés via ports
8. **Concurrence** : Scale out via le modèle de processus
9. **Disposabilité** : Démarrage rapide et arrêt gracieux
10. **Parité dev/prod** : Environnements similaires
11. **Logs** : Traités comme des flux d'événements
12. **Processus d'administration** : Tasks ponctuelles

### Patterns Essentiels

**Stateless Applications** : Les applications ne stockent pas d'état localement.

**Externalization de la Configuration** : ConfigMaps et Secrets pour la configuration.

**Health Checks** : Probes pour la surveillance de la santé.

**Graceful Shutdown** : Arrêt propre des applications.

**Circuit Breaker** : Protection contre les cascades de pannes.

**Retry avec Backoff** : Réessayer les opérations avec délai exponentiel.

**Service Discovery** : Les services se trouvent automatiquement via DNS.

## Architecture 1 : Application Web Simple

### Vue d'Ensemble

Cette architecture convient pour :
- Sites web avec base de données
- Applications CRUD (Create, Read, Update, Delete)
- APIs simples
- MVPs (Minimum Viable Products)
- Applications monolithiques modernisées

### Diagramme d'Architecture

```
                    Internet
                       ↓
                   [Ingress]
                   (SSL/TLS)
                       ↓
              ┌────────┴────────┐
              │                 │
         [Frontend]        [Backend API]
         (3 replicas)      (3 replicas)
              │                 │
              └────────┬────────┘
                       ↓
                  [Redis Cache]
                  (1 replica)
                       ↓
                  [PostgreSQL]
                  (1 replica + PV)
                       ↓
              [Backup CronJob]
```

### Composants Détaillés

#### 1. Ingress (Point d'Entrée)

**Rôle** : Router le trafic HTTP/HTTPS vers les services appropriés

**Configuration** :
- Certificats SSL automatiques (cert-manager)
- Redirection HTTP → HTTPS
- Rate limiting pour protéger contre les abus
- Compression des réponses

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/enable-cors: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    - api.example.com
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 8080
```

#### 2. Frontend (Interface Utilisateur)

**Rôle** : Servir l'interface utilisateur (React, Vue, Angular, ou HTML statique)

**Configuration** :
- 3 replicas pour haute disponibilité
- Resources requests/limits définis
- Health checks configurés
- ConfigMap pour la configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
      tier: presentation
  template:
    metadata:
      labels:
        app: frontend
        tier: presentation
    spec:
      containers:
      - name: nginx
        image: myregistry/frontend:v1.2.3
        ports:
        - containerPort: 80
          name: http
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        env:
        - name: API_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: api.url
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: production
spec:
  selector:
    app: frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

#### 3. Backend API (Logique Métier)

**Rôle** : API REST qui gère la logique applicative

**Configuration** :
- 3 replicas pour distribuer la charge
- Connexion à la base de données
- Utilisation du cache Redis
- Secrets pour les credentials

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
      tier: application
  template:
    metadata:
      labels:
        app: backend
        tier: application
    spec:
      containers:
      - name: api
        image: myregistry/backend:v1.2.3
        ports:
        - containerPort: 8080
          name: http
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        env:
        # Configuration depuis ConfigMap
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: environment
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: log.level
        # Secrets
        - name: DB_HOST
          value: postgres
        - name: DB_PORT
          value: "5432"
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: database
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        # Redis
        - name: REDIS_HOST
          value: redis
        - name: REDIS_PORT
          value: "6379"
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
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: production
spec:
  selector:
    app: backend
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
  type: ClusterIP
```

#### 4. Redis Cache (Optimisation des Performances)

**Rôle** : Cache en mémoire pour réduire la charge sur la base de données

**Configuration** :
- 1 replica (peut être augmenté avec Redis Sentinel/Cluster)
- Persistence optionnelle
- Resource limits définis

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
      tier: cache
  template:
    metadata:
      labels:
        app: redis
        tier: cache
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        volumeMounts:
        - name: redis-data
          mountPath: /data
      volumes:
      - name: redis-data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: production
spec:
  selector:
    app: redis
  ports:
  - protocol: TCP
    port: 6379
  type: ClusterIP
```

#### 5. PostgreSQL (Base de Données)

**Rôle** : Stockage persistant des données applicatives

**Configuration** :
- StatefulSet pour la stabilité
- PersistentVolumeClaim pour la persistance
- Backup réguliers via CronJob

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: standard
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
      tier: database
  template:
    metadata:
      labels:
        app: postgres
        tier: database
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: database
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - $(POSTGRES_USER)
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - $(POSTGRES_USER)
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: production
spec:
  selector:
    app: postgres
  ports:
  - protocol: TCP
    port: 5432
  clusterIP: None  # Headless service pour StatefulSet
```

#### 6. Configuration et Secrets

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  environment: "production"
  log.level: "info"
  api.url: "https://api.example.com"
  cache.ttl: "3600"
  feature.new-ui: "true"
---
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
stringData:
  database: "appdb"
  username: "appuser"
  password: "SuperSecurePassword123!"  # À remplacer par un vrai secret
```

#### 7. Backup Automatique

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: production
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h du matin
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: postgres:15
            command:
            - sh
            - -c
            - |
              TIMESTAMP=$(date +%Y%m%d_%H%M%S)
              pg_dump -h postgres -U $DB_USER $DB_NAME > /backup/backup_${TIMESTAMP}.sql

              # Garder seulement les 30 derniers backups
              cd /backup
              ls -t | tail -n +31 | xargs rm -f

              echo "Backup completed: backup_${TIMESTAMP}.sql"
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: DB_NAME
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: database
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
```

### Points Clés de cette Architecture

**✅ Haute disponibilité** : Frontend et Backend en 3 replicas

**✅ Performance** : Cache Redis pour réduire la charge DB

**✅ Persistence** : PersistentVolume pour PostgreSQL

**✅ Sécurité** : SSL/TLS, Secrets pour les credentials

**✅ Observabilité** : Health checks sur tous les composants

**✅ Sauvegarde** : Backup automatique quotidien

**✅ Scalabilité** : Facile d'ajouter des replicas

## Architecture 2 : Microservices

### Vue d'Ensemble

Cette architecture convient pour :
- Applications complexes nécessitant une séparation des responsabilités
- Équipes multiples travaillant indépendamment
- Besoin de scaler différentes parties différemment
- Déploiements indépendants de chaque service

### Diagramme d'Architecture

```
                        Internet
                           ↓
                      [Ingress]
                           ↓
                    [API Gateway]
                           ↓
            ┌──────────────┼──────────────┐
            │              │              │
      [User Service]  [Order Service] [Payment Service]
      (3 replicas)     (5 replicas)   (3 replicas)
            │              │              │
            ↓              ↓              ↓
      [User DB]       [Order DB]    [Payment DB]
      (PostgreSQL)    (MongoDB)     (PostgreSQL)
            │              │              │
            └──────────────┼──────────────┘
                           ↓
                    [Message Queue]
                      (RabbitMQ)
                           ↓
                  [Worker Services]
                   (Event Processing)
```

### Composants Détaillés

#### 1. API Gateway (Kong/Ambassador)

**Rôle** : Point d'entrée unique, routing, authentification, rate limiting

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: microservices
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: kong
        image: kong:3.4
        ports:
        - containerPort: 8000
          name: proxy
        - containerPort: 8001
          name: admin
        env:
        - name: KONG_DATABASE
          value: "off"
        - name: KONG_PROXY_ACCESS_LOG
          value: "/dev/stdout"
        - name: KONG_ADMIN_ACCESS_LOG
          value: "/dev/stdout"
        - name: KONG_PROXY_ERROR_LOG
          value: "/dev/stderr"
        - name: KONG_ADMIN_ERROR_LOG
          value: "/dev/stderr"
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

#### 2. Service Utilisateur

**Rôle** : Gestion des utilisateurs, authentification, profils

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: microservices
  labels:
    app: user-service
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
        version: v1
    spec:
      containers:
      - name: user-service
        image: myregistry/user-service:v1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: SERVICE_NAME
          value: "user-service"
        - name: DB_HOST
          value: "user-db"
        - name: DB_PORT
          value: "5432"
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: jwt-secret
              key: secret
        - name: RABBITMQ_URL
          valueFrom:
            secretKeyRef:
              name: rabbitmq-secret
              key: url
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: microservices
spec:
  selector:
    app: user-service
  ports:
  - name: http
    port: 80
    targetPort: 8080
  type: ClusterIP
```

#### 3. Service Commandes

**Rôle** : Gestion des commandes, workflows de commande

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: microservices
spec:
  replicas: 5  # Plus de replicas car c'est le service le plus sollicité
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
        version: v1
    spec:
      containers:
      - name: order-service
        image: myregistry/order-service:v1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: SERVICE_NAME
          value: "order-service"
        - name: MONGODB_URI
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: uri
        - name: USER_SERVICE_URL
          value: "http://user-service"
        - name: PAYMENT_SERVICE_URL
          value: "http://payment-service"
        - name: RABBITMQ_URL
          valueFrom:
            secretKeyRef:
              name: rabbitmq-secret
              key: url
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
```

#### 4. Message Queue (Communication Asynchrone)

**Rôle** : Communication asynchrone entre services

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
  namespace: microservices
spec:
  serviceName: rabbitmq
  replicas: 1
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      containers:
      - name: rabbitmq
        image: rabbitmq:3-management
        ports:
        - containerPort: 5672
          name: amqp
        - containerPort: 15672
          name: management
        env:
        - name: RABBITMQ_DEFAULT_USER
          valueFrom:
            secretKeyRef:
              name: rabbitmq-secret
              key: username
        - name: RABBITMQ_DEFAULT_PASS
          valueFrom:
            secretKeyRef:
              name: rabbitmq-secret
              key: password
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        volumeMounts:
        - name: rabbitmq-data
          mountPath: /var/lib/rabbitmq
  volumeClaimTemplates:
  - metadata:
      name: rabbitmq-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 5Gi
```

#### 5. Service Mesh (Optionnel mais Recommandé)

Pour les architectures microservices complexes, un service mesh comme Istio ou Linkerd apporte :

- **Traffic Management** : Routing avancé, load balancing
- **Security** : mTLS automatique entre services
- **Observability** : Métriques et tracing distribué
- **Resilience** : Circuit breakers, retries, timeouts

### Communication entre Services

#### Communication Synchrone (REST)

```yaml
# Dans order-service, appel au user-service
http://user-service/api/users/123

# DNS Kubernetes résout automatiquement
# user-service → user-service.microservices.svc.cluster.local
```

#### Communication Asynchrone (Events)

```yaml
# Order-service publie un événement
Event: OrderCreated
{
  "orderId": "12345",
  "userId": "user-123",
  "amount": 99.99
}

# Payment-service écoute et traite
Event Handler: ProcessPayment
```

### Patterns Microservices Importants

#### 1. Circuit Breaker

Évite les cascades de pannes en "coupant le circuit" vers un service défaillant.

```yaml
# Configuration avec Istio
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service-circuit-breaker
spec:
  host: payment-service
  trafficPolicy:
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

#### 2. Saga Pattern

Gère les transactions distribuées à travers plusieurs services.

```
1. Order Service: Créer commande
2. Payment Service: Réserver paiement
3. Inventory Service: Réserver stock
4. Shipping Service: Créer expédition

Si une étape échoue → Compensation des étapes précédentes
```

#### 3. API Composition

L'API Gateway compose les réponses de plusieurs services.

```yaml
# Client fait une requête
GET /api/orders/123/full

# API Gateway appelle
GET user-service/users/456
GET order-service/orders/123
GET payment-service/payments/789

# Et retourne une réponse combinée
```

## Architecture 3 : Data Pipeline

### Vue d'Ensemble

Cette architecture convient pour :
- Traitement et analyse de données
- ETL (Extract, Transform, Load)
- Machine Learning pipelines
- Analytics en temps réel

### Diagramme d'Architecture

```
     [Data Sources]
     (API, Files, DB)
            ↓
      [Ingestion]
      (Kafka/NATS)
            ↓
    ┌───────┴───────┐
    │               │
[Stream Processing] [Batch Processing]
(Kafka Streams)     (Jobs/CronJobs)
    │               │
    └───────┬───────┘
            ↓
      [Data Lake]
      (MinIO/S3)
            ↓
    ┌───────┴───────┐
    │               │
[Analytics DB]  [ML Models]
(ClickHouse)    (TensorFlow)
    │               │
    └───────┬───────┘
            ↓
    [Visualization]
    (Grafana/Superset)
```

### Composants Détaillés

#### 1. Ingestion de Données

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-ingestion
  namespace: data-pipeline
spec:
  replicas: 3
  selector:
    matchLabels:
      app: data-ingestion
  template:
    metadata:
      labels:
        app: data-ingestion
    spec:
      containers:
      - name: ingestion
        image: myregistry/data-ingestion:v1.0.0
        env:
        - name: KAFKA_BROKERS
          value: "kafka:9092"
        - name: TOPIC_NAME
          value: "raw-events"
        - name: BATCH_SIZE
          value: "1000"
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
```

#### 2. Stream Processing

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stream-processor
  namespace: data-pipeline
spec:
  replicas: 3
  selector:
    matchLabels:
      app: stream-processor
  template:
    metadata:
      labels:
        app: stream-processor
    spec:
      containers:
      - name: processor
        image: myregistry/stream-processor:v1.0.0
        env:
        - name: INPUT_TOPIC
          value: "raw-events"
        - name: OUTPUT_TOPIC
          value: "processed-events"
        - name: STATE_STORE
          value: "/data/state"
        resources:
          requests:
            memory: "1Gi"
            cpu: "1000m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
        volumeMounts:
        - name: state-store
          mountPath: /data/state
      volumes:
      - name: state-store
        persistentVolumeClaim:
          claimName: stream-processor-state
```

#### 3. Batch Processing (Jobs)

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-aggregation
  namespace: data-pipeline
spec:
  schedule: "0 1 * * *"  # Tous les jours à 1h du matin
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: aggregator
            image: myregistry/batch-aggregator:v1.0.0
            env:
            - name: INPUT_PATH
              value: "s3://data-lake/events/"
            - name: OUTPUT_PATH
              value: "s3://data-lake/aggregated/"
            - name: DATE
              value: "{{ .Values.date }}"
            resources:
              requests:
                memory: "2Gi"
                cpu: "2000m"
              limits:
                memory: "4Gi"
                cpu: "4000m"
```

#### 4. Storage (MinIO comme S3-compatible)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: minio
  namespace: data-pipeline
spec:
  serviceName: minio
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: minio/minio:latest
        command:
        - /bin/bash
        - -c
        args:
        - minio server /data --console-address :9001
        ports:
        - containerPort: 9000
          name: api
        - containerPort: 9001
          name: console
        env:
        - name: MINIO_ROOT_USER
          valueFrom:
            secretKeyRef:
              name: minio-secret
              key: root-user
        - name: MINIO_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: minio-secret
              key: root-password
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
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

## Architecture 4 : Full-Stack Production

### Vue d'Ensemble

Cette architecture complète intègre tous les éléments nécessaires pour un environnement de production robuste :

- Application multi-tiers
- Monitoring et observabilité complets
- Sécurité renforcée
- CI/CD intégré
- Backup et disaster recovery
- Haute disponibilité

### Diagramme d'Architecture Complet

```
                         Internet
                            ↓
                    [Load Balancer]
                       (MetalLB)
                            ↓
                      [Ingress]
                    (nginx + SSL)
                            ↓
              ┌─────────────┼─────────────┐
              │             │             │
         [Frontend]     [Backend]    [Admin Panel]
              │             │             │
              └─────────────┼─────────────┘
                            ↓
              ┌─────────────┼─────────────┐
              │             │             │
          [Redis]      [PostgreSQL]   [ElasticSearch]
              │             │             │
              └─────────────┼─────────────┘
                            ↓
                   [Monitoring Stack]
                            │
              ┌─────────────┼─────────────┐
              │             │             │
        [Prometheus]    [Grafana]       [Loki]
              │             │             │
              └─────────────┼─────────────┘
                            ↓
                      [Alerting]
                   (Alertmanager)
                            ↓
                  [Notification]
                 (Slack/Email)
```

### Namespace Organization

```yaml
# Namespaces pour séparer les responsabilités
production          # Application principale
monitoring          # Prometheus, Grafana, Loki
logging             # ElasticSearch, Kibana
security            # RBAC, Network Policies, OPA
cicd                # GitLab Runner, ArgoCD
backup              # Backup jobs et storage
```

### Monitoring et Observabilité

#### 1. Prometheus (Métriques)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s

    scrape_configs:
    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
      - role: node

    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true

    - job_name: 'application-metrics'
      static_configs:
      - targets: ['backend.production:8080']
```

#### 2. Grafana (Visualisation)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - containerPort: 3000
        env:
        - name: GF_SECURITY_ADMIN_USER
          valueFrom:
            secretKeyRef:
              name: grafana-secret
              key: admin-user
        - name: GF_SECURITY_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: grafana-secret
              key: admin-password
        - name: GF_SERVER_ROOT_URL
          value: "https://grafana.example.com"
        volumeMounts:
        - name: grafana-storage
          mountPath: /var/lib/grafana
        - name: grafana-datasources
          mountPath: /etc/grafana/provisioning/datasources
        - name: grafana-dashboards
          mountPath: /etc/grafana/provisioning/dashboards
      volumes:
      - name: grafana-storage
        persistentVolumeClaim:
          claimName: grafana-pvc
      - name: grafana-datasources
        configMap:
          name: grafana-datasources
      - name: grafana-dashboards
        configMap:
          name: grafana-dashboards
```

#### 3. Alerting

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 5m
      slack_api_url: 'YOUR_SLACK_WEBHOOK_URL'

    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      receiver: 'slack-notifications'
      routes:
      - match:
          severity: critical
        receiver: 'pagerduty-critical'
      - match:
          severity: warning
        receiver: 'slack-notifications'

    receivers:
    - name: 'slack-notifications'
      slack_configs:
      - channel: '#alerts'
        title: 'Alert: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

    - name: 'pagerduty-critical'
      pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_KEY'
```

### Sécurité

#### 1. Network Policies

```yaml
# Politique par défaut : tout bloquer
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Autoriser seulement le trafic nécessaire
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Autoriser depuis frontend
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  # Autoriser depuis ingress
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
  egress:
  # Autoriser vers PostgreSQL
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  # Autoriser vers Redis
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
  # Autoriser DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

#### 2. RBAC

```yaml
# ServiceAccount pour l'application
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend-sa
  namespace: production
---
# Role avec permissions minimales
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: backend-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-rolebinding
  namespace: production
subjects:
- kind: ServiceAccount
  name: backend-sa
  namespace: production
roleRef:
  kind: Role
  name: backend-role
  apiGroup: rbac.authorization.k8s.io
```

#### 3. Pod Security Standards

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### CI/CD avec GitOps

#### 1. ArgoCD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp.git
    targetRevision: HEAD
    path: k8s/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

#### 2. GitLab CI Pipeline

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

build:
  stage: build
  script:
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG

test:
  stage: test
  script:
    - docker run $IMAGE_TAG npm test

deploy:
  stage: deploy
  script:
    - kubectl set image deployment/backend backend=$IMAGE_TAG -n production
    - kubectl rollout status deployment/backend -n production
  only:
    - main
```

### Disaster Recovery

#### 1. Backup Strategy

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: full-backup
  namespace: backup
spec:
  schedule: "0 3 * * 0"  # Dimanche à 3h du matin
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: myregistry/backup-tool:v1.0.0
            command:
            - /bin/bash
            - -c
            - |
              # Backup PostgreSQL
              pg_dumpall -h postgres.production -U postgres > /backup/postgres_full_$(date +%Y%m%d).sql

              # Backup PersistentVolumes
              kubectl get pvc -n production -o json > /backup/pvc_snapshot_$(date +%Y%m%d).json

              # Backup Configurations
              kubectl get configmap -n production -o yaml > /backup/configmaps_$(date +%Y%m%d).yaml
              kubectl get secret -n production -o yaml > /backup/secrets_$(date +%Y%m%d).yaml

              # Upload to S3
              aws s3 sync /backup s3://my-backup-bucket/full-backup/
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
```

#### 2. Restore Procedure

```bash
# Script de restauration
#!/bin/bash

BACKUP_DATE=$1

# Restaurer PostgreSQL
kubectl exec -n production postgres-0 -- psql -U postgres < postgres_full_${BACKUP_DATE}.sql

# Restaurer les configurations
kubectl apply -f configmaps_${BACKUP_DATE}.yaml
kubectl apply -f secrets_${BACKUP_DATE}.yaml

# Recréer les PVCs si nécessaire
kubectl apply -f pvc_snapshot_${BACKUP_DATE}.json

# Redémarrer les applications
kubectl rollout restart deployment -n production
```

## Bonnes Pratiques d'Architecture

### 1. Principes de Conception

**Separation of Concerns** : Chaque composant a une responsabilité unique et bien définie.

**Loose Coupling** : Les composants sont faiblement couplés, peuvent évoluer indépendamment.

**High Cohesion** : Les éléments liés sont regroupés ensemble.

**Fail Fast** : Détectez et signalez les erreurs rapidement.

**Defense in Depth** : Plusieurs couches de sécurité.

### 2. Configuration Management

**Externalisation** : Configuration hors du code, dans ConfigMaps et Secrets.

**Hiérarchie** : Configuration par défaut → environnement → override.

**Immutabilité** : Une fois déployée, la configuration ne change pas (redéploiement).

**Validation** : Validation de la configuration avant déploiement.

### 3. Scalabilité

**Horizontal Scaling** : Ajouter des replicas plutôt qu'augmenter les ressources.

**Stateless Design** : Applications sans état local, faciles à scaler.

**Caching Strategy** : Utiliser des caches (Redis, CDN) pour réduire la charge.

**Database Optimization** : Indexes, requêtes optimisées, read replicas.

### 4. Résilience

**Health Checks** : Liveness et Readiness probes sur tous les pods.

**Graceful Degradation** : L'application fonctionne en mode dégradé si un service est down.

**Circuit Breaker** : Protection contre les cascades de pannes.

**Retry Logic** : Réessayer avec backoff exponentiel.

**Timeouts** : Tous les appels réseau ont des timeouts définis.

### 5. Observabilité

**Structured Logging** : Logs au format JSON avec contexte.

**Distributed Tracing** : Tracer les requêtes à travers les services.

**Metrics Collection** : Exposer des métriques Prometheus.

**Dashboards** : Dashboards Grafana pour chaque service.

**Alerting** : Alertes sur les KPIs critiques.

### 6. Sécurité

**Principle of Least Privilege** : Permissions minimales nécessaires.

**Network Segmentation** : Network Policies pour isoler les services.

**Secrets Management** : Ne jamais mettre de secrets en clair dans le code.

**Regular Updates** : Mettre à jour régulièrement les images et dépendances.

**Security Scanning** : Scanner les images pour les vulnérabilités.

## Adaptation des Architectures

### Comment Adapter une Architecture de Référence

**1. Évaluer vos besoins** :
- Volume de trafic attendu
- Criticité de l'application
- Budget disponible
- Compétences de l'équipe

**2. Simplifier si nécessaire** :
- Commencer simple (Architecture 1)
- Ajouter de la complexité seulement si nécessaire
- Ne pas sur-engineer

**3. Ajuster les ressources** :
```yaml
# Petit traffic
resources:
  requests:
    memory: "64Mi"
    cpu: "50m"
  limits:
    memory: "128Mi"
    cpu: "100m"

# Traffic moyen
resources:
  requests:
    memory: "256Mi"
    cpu: "200m"
  limits:
    memory: "512Mi"
    cpu: "500m"

# Fort traffic
resources:
  requests:
    memory: "1Gi"
    cpu: "1000m"
  limits:
    memory: "2Gi"
    cpu: "2000m"
```

**4. Ajuster les replicas** :
```yaml
# Développement : 1 replica
# Staging : 2 replicas
# Production petit : 3 replicas
# Production moyen : 5 replicas
# Production grand : 10+ replicas avec HPA
```

**5. Choisir les bons composants** :
- PostgreSQL vs MySQL vs MongoDB : selon vos données
- Redis vs Memcached : selon vos besoins de cache
- RabbitMQ vs Kafka vs NATS : selon votre volume de messages

## Checklist d'Architecture

Avant de déployer en production, vérifiez :

### Disponibilité
- [ ] Au moins 3 replicas pour les composants critiques
- [ ] Health checks configurés (liveness et readiness)
- [ ] PodDisruptionBudget définis
- [ ] Anti-affinity pour répartir les pods sur différents nœuds

### Performance
- [ ] Resources requests et limits définis
- [ ] Cache configuré (Redis, CDN)
- [ ] Database indexes optimisés
- [ ] HPA (Horizontal Pod Autoscaler) configuré si nécessaire

### Sécurité
- [ ] Network Policies en place
- [ ] RBAC configuré
- [ ] Secrets utilisés pour les credentials
- [ ] SSL/TLS activé
- [ ] Images scannées pour vulnérabilités
- [ ] Pod Security Standards appliqués

### Observabilité
- [ ] Logging centralisé configuré
- [ ] Métriques Prometheus exposées
- [ ] Dashboards Grafana créés
- [ ] Alertes configurées
- [ ] Distributed tracing (si microservices)

### Data
- [ ] PersistentVolumes pour les données importantes
- [ ] Backup automatique configuré
- [ ] Procédure de restore testée
- [ ] Retention policy définie

### CI/CD
- [ ] Pipeline automatisé
- [ ] Tests automatisés
- [ ] Déploiement GitOps (ArgoCD)
- [ ] Rollback facile

### Documentation
- [ ] Architecture documentée
- [ ] Runbooks pour les incidents
- [ ] Procédures de déploiement
- [ ] Contacts et escalation

## Conclusion

Les architectures de référence vous fournissent des modèles éprouvés pour démarrer rapidement et correctement. Avec MicroK8s, vous pouvez implémenter ces architectures localement, les tester, et les adapter à vos besoins spécifiques avant de les déployer en production.

**Points clés à retenir** :

✅ **Commencez simple** : Ne pas sur-engineer, partir de l'architecture la plus simple qui répond à vos besoins

✅ **Évoluez progressivement** : Ajouter de la complexité seulement quand nécessaire

✅ **Suivez les bonnes pratiques** : Les architectures de référence intègrent des années d'expérience

✅ **Adaptez à votre contexte** : Chaque projet est unique, ajustez selon vos contraintes

✅ **Documentez vos choix** : Expliquez pourquoi vous avez choisi telle ou telle approche

✅ **Testez sur MicroK8s** : Validez votre architecture localement avant la production

✅ **Préparez la production** : Monitoring, backup, sécurité dès le départ

**Prochaines étapes** :

1. Choisissez l'architecture qui correspond à votre cas d'usage
2. Déployez-la sur MicroK8s
3. Adaptez-la à vos besoins spécifiques
4. Testez et validez
5. Documentez votre architecture finale
6. Déployez en production avec confiance

**Vous avez maintenant tous les éléments pour concevoir et déployer des architectures robustes et scalables avec MicroK8s ! 🏗️**

⏭️ [Aller Plus Loin](/25-aller-plus-loin/README.md)
