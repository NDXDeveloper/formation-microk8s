🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.5 Intégration avec les Services

## Introduction

Les Services LoadBalancer ne vivent pas en isolation. Dans un cluster Kubernetes, ils s'intègrent avec de nombreux autres composants pour former une architecture cohérente. Dans cette section, nous allons explorer comment les Services LoadBalancer interagissent avec le reste de votre infrastructure Kubernetes.

Comprendre ces intégrations vous permettra de concevoir des architectures robustes et scalables, que ce soit pour une simple application web ou pour un système de microservices complexe.

## Architecture Globale : Vue d'Ensemble

Voici comment les différents composants Kubernetes s'articulent autour des Services :

```
┌─────────────────────────────────────────────────────────┐
│                    Utilisateurs                         │
└────────────────────────┬────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────┐
│              MetalLB LoadBalancer                       │
│                  (IP Externe)                           │
└────────────────────────┬────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────┐
│                Service LoadBalancer                     │
│             (Load Balancing L4)                         │
└────────────────────────┬────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┐
          ↓              ↓              ↓
      ┌───────┐      ┌───────┐      ┌───────┐
      │ Pod 1 │      │ Pod 2 │      │ Pod 3 │
      └───────┘      └───────┘      └───────┘
          ↑              ↑              ↑
          └──────────────┴──────────────┘
                         │
                   Deployment /
                StatefulSet / DaemonSet
```

## Intégration avec les Workloads Kubernetes

Les Services LoadBalancer peuvent exposer différents types de workloads Kubernetes. Chacun a ses particularités.

### 1. Intégration avec les Deployments

Les **Deployments** sont le type de workload le plus courant. Ils gèrent des Pods sans état (stateless) qui peuvent être facilement répliqués.

**Exemple d'architecture complète** :

```yaml
# Deployment : L'application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
          name: http
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 3
---
# Service LoadBalancer : L'exposition
apiVersion: v1
kind: Service
metadata:
  name: web-app-lb
  labels:
    app: web-app
spec:
  type: LoadBalancer
  selector:
    app: web-app  # Correspondance avec les labels du Deployment
  ports:
  - name: http
    port: 80
    targetPort: http  # Référence au port nommé
    protocol: TCP
  sessionAffinity: None
```

**Points clés** :

- **Selector matching** : Le `selector` du Service doit correspondre aux `labels` des Pods
- **Port naming** : Utiliser des noms de ports facilite la maintenance
- **Health checks** : Les readinessProbes assurent que seuls les Pods sains reçoivent du trafic
- **Resources** : Définir les ressources aide le scheduler et évite la surcharge

**Comportement** :

- Si vous scalez le Deployment (`kubectl scale deployment web-app --replicas=5`), le Service distribue automatiquement le trafic vers les 5 Pods
- Si un Pod crash, le Service cesse de lui envoyer du trafic
- Les rolling updates sont transparents pour les clients

### 2. Intégration avec les StatefulSets

Les **StatefulSets** sont conçus pour les applications stateful (avec état) comme les bases de données. Chaque Pod a une identité stable.

**Particularités** :

- Les Pods ont des noms prévisibles : `nom-0`, `nom-1`, `nom-2`
- Chaque Pod peut avoir son propre volume persistant
- L'ordre de démarrage et d'arrêt est contrôlé

**Exemple : Base de données PostgreSQL** :

```yaml
# StatefulSet pour PostgreSQL
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless  # Service headless pour la communication interne
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
---
# Service LoadBalancer pour l'accès externe
apiVersion: v1
kind: Service
metadata:
  name: postgres-lb
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.1.210  # IP fixe pour la base de données
  selector:
    app: postgres
  ports:
  - name: postgres
    port: 5432
    targetPort: postgres
---
# Service Headless pour la communication interne
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None  # Headless
  selector:
    app: postgres
  ports:
  - name: postgres
    port: 5432
```

**Architecture à deux Services** :

- **LoadBalancer** : Pour l'accès externe (clients, applications externes)
- **Headless** : Pour la communication entre Pods (réplication, clustering)

### 3. Intégration avec les DaemonSets

Les **DaemonSets** déploient un Pod sur chaque nœud du cluster. Ils sont utilisés pour des agents système (monitoring, logging, réseau).

**Cas d'usage avec LoadBalancer** :

Exposer un agent de monitoring qui collecte des métriques depuis tous les nœuds :

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true  # Accès aux métriques du nœud
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
          name: metrics
---
apiVersion: v1
kind: Service
metadata:
  name: node-exporter-lb
spec:
  type: LoadBalancer
  selector:
    app: node-exporter
  ports:
  - name: metrics
    port: 9100
    targetPort: metrics
```

Le Service distribue les requêtes vers les Pods node-exporter sur différents nœuds.

## Intégration avec Différents Types de Services

Dans une architecture complète, vous aurez plusieurs types de Services qui coexistent et interagissent.

### Architecture Multi-Services

**Exemple : Application E-Commerce** :

```yaml
# Frontend Web (LoadBalancer - exposition publique)
apiVersion: v1
kind: Service
metadata:
  name: frontend-lb
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 3000
---
# API Backend (ClusterIP - interne uniquement)
apiVersion: v1
kind: Service
metadata:
  name: api-backend
spec:
  type: ClusterIP  # Pas d'accès externe
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080
---
# Base de données (ClusterIP - interne uniquement)
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
  - port: 5432
---
# Cache Redis (ClusterIP - interne uniquement)
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  type: ClusterIP
  selector:
    app: redis
  ports:
  - port: 6379
```

**Flux de communication** :

```
Internet
   ↓
LoadBalancer (192.168.1.200:80)
   ↓
Frontend Pods
   ↓
api-backend.default.svc.cluster.local:8080 (ClusterIP)
   ↓
Backend Pods
   ↓ ↓
   database.default.svc.cluster.local:5432
   redis.default.svc.cluster.local:6379
```

**Principe de sécurité** : Seul le frontend est exposé publiquement. Le backend, la base de données et le cache ne sont accessibles qu'en interne.

## DNS et Découverte de Services

Kubernetes fournit automatiquement un DNS interne qui permet aux Services de se découvrir mutuellement.

### Noms DNS des Services

Chaque Service reçoit un nom DNS selon le format :

```
<nom-service>.<namespace>.svc.cluster.local
```

**Exemples** :

- Service `web-app` dans namespace `default` : `web-app.default.svc.cluster.local`
- Service `api` dans namespace `production` : `api.production.svc.cluster.local`

### Raccourcis DNS

Dans le même namespace, vous pouvez utiliser des noms courts :

```yaml
# Depuis un Pod dans le namespace default
env:
- name: DATABASE_HOST
  value: postgres  # Équivalent à postgres.default.svc.cluster.local

- name: API_URL
  value: http://api-backend:8080
```

Depuis un autre namespace :

```yaml
env:
- name: DATABASE_HOST
  value: postgres.default  # Depuis un autre namespace
```

### Communication Inter-Services

**Exemple : Frontend appelant le Backend** :

```yaml
# Deployment Frontend
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: mon-frontend:v1
        env:
        - name: BACKEND_API_URL
          value: "http://api-backend:8080"  # Nom DNS du Service backend
        - name: DATABASE_HOST
          value: "postgres"
        - name: DATABASE_PORT
          value: "5432"
        ports:
        - containerPort: 3000
```

Le frontend utilise les noms DNS des Services pour communiquer avec le backend et la base de données, sans avoir besoin de connaître les IP des Pods.

## Intégration avec ConfigMaps et Secrets

Les Services sont souvent configurés via ConfigMaps (configuration) et Secrets (données sensibles).

### Exemple : Configuration Centralisée

```yaml
# ConfigMap pour la configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database.host: "postgres"
  database.port: "5432"
  redis.host: "redis"
  redis.port: "6379"
  api.url: "http://api-backend:8080"
---
# Secret pour les données sensibles
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  database.password: "motdepasse_secret"
  api.key: "cle_api_secrete"
---
# Deployment utilisant ConfigMap et Secret
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: mon-app:v1
        envFrom:
        - configMapRef:
            name: app-config  # Toutes les clés du ConfigMap deviennent des variables d'env
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database.password
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: api.key
```

**Avantages** :

- Configuration centralisée et versionnable
- Séparation des données sensibles
- Facilite le déploiement dans différents environnements

## Patterns d'Architecture

Explorons quelques patterns d'architecture courants utilisant les Services LoadBalancer.

### Pattern 1 : Architecture à 3 Tiers

L'architecture classique web : présentation, logique métier, données.

```yaml
# Tier 1 : Présentation (LoadBalancer)
apiVersion: v1
kind: Service
metadata:
  name: web-frontend-lb
spec:
  type: LoadBalancer
  selector:
    tier: frontend
  ports:
  - port: 80
---
# Tier 2 : Logique Métier (ClusterIP)
apiVersion: v1
kind: Service
metadata:
  name: app-logic
spec:
  type: ClusterIP
  selector:
    tier: backend
  ports:
  - port: 8080
---
# Tier 3 : Données (ClusterIP)
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  type: ClusterIP
  selector:
    tier: database
  ports:
  - port: 5432
```

**Communication** :

```
Internet → LoadBalancer → Frontend Pods
                              ↓
                      app-logic:8080 (ClusterIP)
                              ↓
                      Backend Pods
                              ↓
                      database:5432 (ClusterIP)
                              ↓
                      Database Pods
```

### Pattern 2 : Microservices avec Gateway API

Une architecture microservices avec un API Gateway comme point d'entrée unique.

```yaml
# API Gateway (LoadBalancer - point d'entrée unique)
apiVersion: v1
kind: Service
metadata:
  name: api-gateway-lb
spec:
  type: LoadBalancer
  selector:
    app: api-gateway
  ports:
  - name: http
    port: 80
  - name: https
    port: 443
---
# Microservice: User Service (ClusterIP)
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  type: ClusterIP
  selector:
    app: user-service
  ports:
  - port: 8081
---
# Microservice: Order Service (ClusterIP)
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  type: ClusterIP
  selector:
    app: order-service
  ports:
  - port: 8082
---
# Microservice: Payment Service (ClusterIP)
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  type: ClusterIP
  selector:
    app: payment-service
  ports:
  - port: 8083
```

**Architecture** :

```
Internet
   ↓
API Gateway (LoadBalancer)
   ↓
   ├── /users → user-service:8081
   ├── /orders → order-service:8082
   └── /payments → payment-service:8083
```

L'API Gateway route les requêtes vers les microservices appropriés basé sur l'URL.

### Pattern 3 : Services Publics et Privés

Certains services doivent être accessibles publiquement, d'autres non.

```yaml
# Service Public : Site Web (LoadBalancer)
apiVersion: v1
kind: Service
metadata:
  name: public-website-lb
  annotations:
    metallb.universe.tf/address-pool: public-pool
spec:
  type: LoadBalancer
  selector:
    app: website
  ports:
  - port: 80
---
# Service Public : API REST (LoadBalancer)
apiVersion: v1
kind: Service
metadata:
  name: public-api-lb
  annotations:
    metallb.universe.tf/address-pool: public-pool
spec:
  type: LoadBalancer
  selector:
    app: public-api
  ports:
  - port: 443
---
# Service Privé : Admin Panel (ClusterIP ou LoadBalancer avec IP interne)
apiVersion: v1
kind: Service
metadata:
  name: admin-panel
spec:
  type: ClusterIP  # Ou LoadBalancer avec un pool privé
  selector:
    app: admin-panel
  ports:
  - port: 8080
```

**Sécurité** :

- Services publics : Accessibles depuis Internet
- Services privés : Accessibles uniquement via VPN ou réseau interne

## Intégration avec Ingress (Aperçu)

Les Services LoadBalancer et Ingress travaillent souvent ensemble. Voici un aperçu de cette intégration (détails au chapitre 10).

### Architecture LoadBalancer + Ingress

```yaml
# L'Ingress Controller lui-même est exposé via LoadBalancer
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-lb
  namespace: ingress-nginx
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: ingress-nginx
  ports:
  - name: http
    port: 80
  - name: https
    port: 443
---
# Les applications utilisent ClusterIP + Ingress
apiVersion: v1
kind: Service
metadata:
  name: app1
spec:
  type: ClusterIP  # Pas LoadBalancer !
  selector:
    app: app1
  ports:
  - port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: app2
spec:
  type: ClusterIP
  selector:
    app: app2
  ports:
  - port: 80
---
# Ingress pour le routage
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: apps-ingress
spec:
  rules:
  - host: app1.mondomaine.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1
            port:
              number: 80
  - host: app2.mondomaine.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2
            port:
              number: 80
```

**Flux** :

```
Internet
   ↓
LoadBalancer (192.168.1.200)
   ↓
Ingress Controller
   ↓
   ├── app1.mondomaine.com → Service app1 → Pods app1
   └── app2.mondomaine.com → Service app2 → Pods app2
```

**Avantages** :

- Une seule IP LoadBalancer pour plusieurs applications
- Routage intelligent par nom de domaine
- Gestion centralisée des certificats SSL

## Monitoring et Observabilité

L'intégration des Services avec le monitoring est essentielle pour la production.

### Annotations Prometheus

Si vous utilisez Prometheus (chapitre 12), annotez vos Services pour activer le scraping automatique :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-app-lb
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
spec:
  type: LoadBalancer
  selector:
    app: mon-app
  ports:
  - name: http
    port: 80
  - name: metrics
    port: 9090  # Port de métriques
```

### Service Monitors (Prometheus Operator)

Avec le Prometheus Operator, utilisez des ServiceMonitor :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mon-app-monitor
spec:
  selector:
    matchLabels:
      app: mon-app
  endpoints:
  - port: metrics
    interval: 30s
```

## Labels et Sélecteurs Avancés

Les labels permettent une gestion fine des Services et de leurs cibles.

### Labels Multi-Critères

```yaml
# Service pour l'environnement de production uniquement
apiVersion: v1
kind: Service
metadata:
  name: prod-app-lb
spec:
  type: LoadBalancer
  selector:
    app: mon-app
    environment: production
    version: v2
  ports:
  - port: 80
```

Ce Service ne cible que les Pods avec **tous** ces labels.

### Plusieurs Services pour le Même Deployment

Vous pouvez avoir plusieurs Services pointant vers le même ensemble de Pods :

```yaml
# Service public (LoadBalancer)
apiVersion: v1
kind: Service
metadata:
  name: app-public-lb
spec:
  type: LoadBalancer
  selector:
    app: mon-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
---
# Service admin (LoadBalancer avec IP différente)
apiVersion: v1
kind: Service
metadata:
  name: app-admin-lb
  annotations:
    metallb.universe.tf/address-pool: admin-pool
spec:
  type: LoadBalancer
  selector:
    app: mon-app
  ports:
  - name: admin
    port: 8443
    targetPort: 9090  # Port admin de l'app
```

Les deux Services exposent la même application mais sur des ports différents avec des IP différentes.

## Exemple d'Architecture Complète

Voici un exemple complet d'une application moderne avec tous les composants intégrés.

### Application de Blog avec Microservices

```yaml
# 1. Frontend (React) - LoadBalancer
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
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
      - name: react-app
        image: mon-blog-frontend:v1.0
        ports:
        - containerPort: 3000
        env:
        - name: API_URL
          value: "http://api-backend:8080"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-lb
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 3000
---
# 2. API Backend (Node.js) - ClusterIP
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-backend
      tier: application
  template:
    metadata:
      labels:
        app: api-backend
        tier: application
    spec:
      containers:
      - name: nodejs-api
        image: mon-blog-api:v1.0
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_HOST
          value: "postgres"
        - name: DATABASE_PORT
          value: "5432"
        - name: REDIS_HOST
          value: "redis"
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: api-backend
spec:
  type: ClusterIP
  selector:
    app: api-backend
  ports:
  - port: 8080
---
# 3. PostgreSQL Database - StatefulSet + ClusterIP
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
      tier: data
  template:
    metadata:
      labels:
        app: postgres
        tier: data
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
  - port: 5432
---
# 4. Redis Cache - Deployment + ClusterIP
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
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
---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  type: ClusterIP
  selector:
    app: redis
  ports:
  - port: 6379
---
# 5. Secret pour les credentials
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  password: "mot_de_passe_super_secret"
```

**Architecture complète** :

```
Internet
   ↓
frontend-lb (LoadBalancer: 192.168.1.200:80)
   ↓
Frontend Pods (React)
   ↓
api-backend (ClusterIP: 10.152.183.x:8080)
   ↓
Backend Pods (Node.js)
   ↓ ↓
   postgres (ClusterIP: 10.152.183.y:5432)
   redis (ClusterIP: 10.152.183.z:6379)
```

## Bonnes Pratiques d'Intégration

### 1. Nommage Cohérent

Utilisez une convention de nommage claire :

```yaml
# Pattern : <app>-<component>-<type>
frontend-web-lb          # LoadBalancer frontend
frontend-web             # Deployment frontend
api-backend              # Service ClusterIP backend
api-backend-deployment   # Deployment backend
postgres-db              # Service ClusterIP database
postgres-db-statefulset  # StatefulSet database
```

### 2. Utilisation des Namespaces

Organisez vos ressources par namespace :

```bash
# Namespace pour chaque environnement
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace production

# Ou par application
kubectl create namespace blog
kubectl create namespace ecommerce
kubectl create namespace monitoring
```

### 3. Labels Standardisés

Utilisez des labels communs pour faciliter la gestion :

```yaml
metadata:
  labels:
    app: mon-application      # Nom de l'application
    component: backend        # Composant (frontend, backend, db)
    tier: application         # Tier (presentation, application, data)
    environment: production   # Environnement
    version: v1.0            # Version
    managed-by: kubectl      # Outil de gestion
```

### 4. Documentation dans les Annotations

Utilisez les annotations pour documenter :

```yaml
metadata:
  annotations:
    description: "Service LoadBalancer pour le frontend de l'application blog"
    owner: "equipe-frontend@example.com"
    documentation: "https://wiki.example.com/blog-frontend"
    cost-center: "IT-2024"
```

### 5. Gestion des Dépendances

Documentez les dépendances entre services :

```yaml
# Dans le Deployment
spec:
  template:
    metadata:
      annotations:
        dependencies: "postgres,redis,api-backend"
```

### 6. Health Checks Obligatoires

Toujours définir des health checks :

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3

livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
```

## Résolution de Problèmes d'Intégration

### Problème : Service ne Trouve Pas les Pods

**Symptôme** : Le Service n'a pas d'endpoints.

```bash
microk8s kubectl get endpoints mon-service
# Retourne : No resources found
```

**Vérifications** :

1. **Labels correspondent** :
   ```bash
   # Voir les labels du Service
   microk8s kubectl get svc mon-service -o yaml | grep -A 5 selector

   # Voir les labels des Pods
   microk8s kubectl get pods --show-labels
   ```

2. **Pods sont en Running** :
   ```bash
   microk8s kubectl get pods -l app=mon-app
   ```

3. **Readiness probe passe** :
   ```bash
   microk8s kubectl describe pod <nom-pod> | grep -A 10 Readiness
   ```

### Problème : Communication Inter-Services Échoue

**Symptôme** : Un Pod ne peut pas contacter un autre Service.

**Tests** :

```bash
# Depuis un Pod, tester la résolution DNS
microk8s kubectl exec -it <pod-frontend> -- nslookup api-backend

# Tester la connectivité
microk8s kubectl exec -it <pod-frontend> -- curl http://api-backend:8080/health

# Vérifier les NetworkPolicies (si configurées)
microk8s kubectl get networkpolicies
```

### Problème : Trafic Déséquilibré

**Symptôme** : Un Pod reçoit beaucoup plus de trafic que les autres.

**Causes possibles** :

1. **SessionAffinity activée** : Vérifiez `sessionAffinity: ClientIP`
2. **Health checks échouent** : Certains Pods sont marqués "not ready"
3. **Ressources insuffisantes** : Certains Pods sont ralentis

**Diagnostic** :

```bash
# Voir la distribution des connexions
microk8s kubectl top pods -l app=mon-app

# Vérifier les endpoints
microk8s kubectl get endpoints mon-service -o yaml
```

## Résumé

L'intégration des Services LoadBalancer avec l'écosystème Kubernetes est essentielle pour construire des architectures robustes. Vous avez appris :

✅ **Intégration avec les workloads** : Deployments, StatefulSets, DaemonSets
✅ **Types de Services multiples** : LoadBalancer, ClusterIP, Headless
✅ **DNS et découverte** : Communication inter-services transparente
✅ **ConfigMaps et Secrets** : Configuration centralisée
✅ **Patterns d'architecture** : 3-tiers, microservices, public/privé
✅ **Intégration avec Ingress** : Point d'entrée unique pour plusieurs services
✅ **Monitoring** : Annotations Prometheus et observabilité
✅ **Bonnes pratiques** : Nommage, labels, documentation
✅ **Troubleshooting** : Diagnostic des problèmes d'intégration

Avec ces connaissances, vous pouvez maintenant concevoir et déployer des architectures complètes dans MicroK8s, où les Services LoadBalancer jouent un rôle clé dans l'exposition et la communication de vos applications.

---

**Prochaine étape** : Tests et Validation (Section 9.6) ou Ingress et Routage (Chapitre 10)

⏭️ [Tests et validation](/09-load-balancing-avec-metallb/06-tests-et-validation.md)
