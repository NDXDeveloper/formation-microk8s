🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.7 StatefulSets : déployer des applications stateful

## Introduction

Jusqu'à présent, nous avons principalement travaillé avec des **Deployments** pour déployer nos applications dans Kubernetes. Les Deployments sont parfaits pour des applications **stateless** (sans état), c'est-à-dire des applications où chaque instance est interchangeable et où aucune donnée n'a besoin d'être conservée entre les redémarrages.

Mais que se passe-t-il quand vous avez besoin de déployer des applications **stateful** (avec état) comme :
- Des bases de données (PostgreSQL, MySQL, MongoDB)
- Des systèmes de cache distribués (Redis, Memcached)
- Des systèmes de messaging (Kafka, RabbitMQ)
- Des systèmes de stockage distribués (Elasticsearch, Cassandra)

Ces applications ont des besoins spécifiques :
- Chaque instance a une **identité unique et stable**
- Chaque instance a besoin de son **propre stockage persistant**
- L'**ordre** de démarrage et d'arrêt est important
- Les instances doivent pouvoir se **retrouver** entre elles de manière prévisible

C'est exactement ce que les **StatefulSets** sont conçus pour gérer.

## Qu'est-ce qu'un StatefulSet ?

### Définition

Un **StatefulSet** est un contrôleur Kubernetes spécialement conçu pour gérer des applications stateful. Il garantit :

1. **Identité stable** : Chaque pod a un nom prévisible et permanent
2. **Réseau stable** : Chaque pod a un nom DNS stable
3. **Stockage stable** : Chaque pod a son propre volume persistant dédié
4. **Ordre garanti** : Déploiement et scaling séquentiels et prévisibles

**Analogie** : Imaginez un orchestre. Dans un Deployment, tous les musiciens sont interchangeables (si l'un part, un autre peut le remplacer sans problème). Dans un StatefulSet, chaque musicien a un rôle unique (premier violon, chef d'orchestre, etc.), et on ne peut pas simplement les échanger - chacun a son identité et sa place spécifique.

### Pourquoi avons-nous besoin de StatefulSets ?

**Problèmes avec les Deployments pour les applications stateful** :

```
┌─────────────────────────────────────────────────────────┐
│              DEPLOYMENT (Stateless)                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  webapp-7f8d9c-abc12   │  webapp-7f8d9c-def34           │
│  webapp-7f8d9c-ghi56   │  webapp-7f8d9c-jkl78           │
│                                                         │
│  • Noms aléatoires                                      │
│  • Interchangeables                                     │
│  • Pas d'ordre de démarrage                             │
│  • Stockage partagé ou aucun                            │
│  • Parfait pour applications web sans état              │
│                                                         │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│              STATEFULSET (Stateful)                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  postgres-0  →  [Volume-0]                              │
│  postgres-1  →  [Volume-1]                              │
│  postgres-2  →  [Volume-2]                              │
│                                                         │
│  • Noms prévisibles (0, 1, 2...)                        │
│  • Identité unique et stable                            │
│  • Ordre de démarrage garanti (0→1→2)                   │
│  • Chaque pod a son volume dédié                        │
│  • Parfait pour bases de données, clusters              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## Deployment vs StatefulSet : Comparaison détaillée

| Caractéristique | Deployment | StatefulSet |
|----------------|------------|-------------|
| **Nom des pods** | Aléatoire (app-xxx-yyy) | Ordonné (app-0, app-1, app-2) |
| **Identité** | Interchangeable | Unique et stable |
| **Nom DNS** | Variable | Stable et prévisible |
| **Ordre de déploiement** | Parallèle | Séquentiel (0→1→2) |
| **Ordre de suppression** | Aléatoire | Inverse (2→1→0) |
| **Stockage** | Partagé ou éphémère | Dédié par pod |
| **Persistance après redémarrage** | Non garantie | Garantie |
| **Scaling** | Parallèle | Séquentiel |
| **Cas d'usage** | Applications stateless | Applications stateful |

### Exemple concret des différences

**Deployment** :
```bash
# Scaling de 1 à 3 replicas
webapp-abc123-xyz    # Démarré immédiatement
webapp-def456-uvw    # Démarré immédiatement
webapp-ghi789-rst    # Démarré immédiatement

# Si un pod meurt et est recréé
webapp-jkl012-mno    # Nouveau nom aléatoire
```

**StatefulSet** :
```bash
# Scaling de 1 à 3 replicas
postgres-0    # Démarré d'abord
postgres-1    # Démarré après que postgres-0 soit Ready
postgres-2    # Démarré après que postgres-1 soit Ready

# Si postgres-1 meurt et est recréé
postgres-1    # Même nom, même volume, même DNS
```

## Concepts clés des StatefulSets

### 1. Identité de pod ordonnée

Chaque pod dans un StatefulSet reçoit un nom dans le format :
```
<statefulset-name>-<ordinal>
```

**Exemple** :
- StatefulSet nommé `mysql` avec 3 replicas
- Pods créés : `mysql-0`, `mysql-1`, `mysql-2`

**Caractéristiques** :
- L'ordinal commence à 0
- Les noms ne changent jamais, même après redémarrage
- Les pods sont créés dans l'ordre (0, puis 1, puis 2, etc.)
- Les pods sont supprimés dans l'ordre inverse (2, puis 1, puis 0)

### 2. Réseau stable avec Headless Service

Un StatefulSet nécessite un **Headless Service** pour fournir une identité réseau stable à chaque pod.

**Service normal** :
```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  selector:
    app: webapp
  ports:
  - port: 80
```
**DNS** : `webapp.default.svc.cluster.local` → load balance entre tous les pods

**Headless Service** :
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None    # Headless !
  selector:
    app: mysql
  ports:
  - port: 3306
```

**DNS individuel** :
- `mysql-0.mysql.default.svc.cluster.local` → mysql-0
- `mysql-1.mysql.default.svc.cluster.local` → mysql-1
- `mysql-2.mysql.default.svc.cluster.local` → mysql-2

**Pourquoi c'est important ?**

Les applications stateful ont souvent besoin de communiquer avec des instances spécifiques :
- Le pod 1 doit se connecter spécifiquement au pod 0 (master)
- Les pods doivent savoir qui est qui dans le cluster
- Les connexions peer-to-peer nécessitent des adresses stables

### 3. Stockage persistant avec volumeClaimTemplates

Contrairement aux Deployments où vous référencez un PVC existant, les StatefulSets utilisent des **volumeClaimTemplates** qui créent automatiquement un PVC unique pour chaque pod.

**Deployment** (tous les pods partagent le même PVC) :
```yaml
volumes:
- name: data
  persistentVolumeClaim:
    claimName: shared-pvc    # PVC existant, partagé
```

**StatefulSet** (chaque pod a son propre PVC) :
```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: [ "ReadWriteOnce" ]
    resources:
      requests:
        storage: 10Gi
# Crée automatiquement :
# - data-mysql-0 pour mysql-0
# - data-mysql-1 pour mysql-1
# - data-mysql-2 pour mysql-2
```

### 4. Garanties d'ordre

**Au déploiement** :
```
1. mysql-0 créé
2. Attendre que mysql-0 soit Ready
3. mysql-1 créé
4. Attendre que mysql-1 soit Ready
5. mysql-2 créé
```

**Au scaling down** (3 → 1) :
```
1. mysql-2 supprimé
2. Attendre que mysql-2 soit complètement terminé
3. mysql-1 supprimé
4. Attendre que mysql-1 soit complètement terminé
5. mysql-0 reste en cours d'exécution
```

**À la mise à jour** :
```
1. Mettre à jour mysql-2
2. Attendre que mysql-2 soit Ready
3. Mettre à jour mysql-1
4. Attendre que mysql-1 soit Ready
5. Mettre à jour mysql-0
```

## Anatomie d'un StatefulSet

Examinons la structure complète d'un StatefulSet :

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "web"           # Headless Service associé (REQUIS)
  replicas: 3                  # Nombre de pods
  selector:
    matchLabels:
      app: web
  template:                    # Template de pod (comme dans un Deployment)
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www            # Référence au volume template
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:        # Templates de PVC (spécifique aux StatefulSets)
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "microk8s-hostpath"
      resources:
        requests:
          storage: 1Gi
```

Décortiquons chaque section :

### serviceName : Le service headless

```yaml
spec:
  serviceName: "web"    # REQUIS - Nom du Headless Service
```

**Points importants** :
- C'est un champ **obligatoire**
- Doit correspondre au nom d'un Headless Service existant
- Fournit l'identité réseau stable

### selector : Sélection des pods

```yaml
spec:
  selector:
    matchLabels:
      app: web
```

**Fonction** :
- Identifie les pods gérés par ce StatefulSet
- Doit correspondre aux labels dans `template.metadata.labels`
- Immuable après création

### template : Template de pod

```yaml
spec:
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        # ... configuration du conteneur
```

**Points clés** :
- Identique à la section template d'un Deployment
- Définit la configuration de chaque pod
- Les volumeMounts référencent les volumeClaimTemplates

### volumeClaimTemplates : Stockage automatique

```yaml
volumeClaimTemplates:
- metadata:
    name: www           # Nom du volume
  spec:
    accessModes: [ "ReadWriteOnce" ]
    storageClassName: "microk8s-hostpath"
    resources:
      requests:
        storage: 1Gi
```

**Comportement** :
- Crée automatiquement un PVC pour chaque pod
- Format du PVC : `<nom-volume>-<nom-statefulset>-<ordinal>`
- Exemple : `www-web-0`, `www-web-1`, `www-web-2`
- Les PVC persistent même si le StatefulSet est supprimé (protection des données)

### Champs optionnels importants

#### podManagementPolicy

```yaml
spec:
  podManagementPolicy: OrderedReady    # ou Parallel
```

**OrderedReady** (défaut) :
- Déploiement séquentiel strict
- Attendre que chaque pod soit Ready avant le suivant

**Parallel** :
- Déploiement parallèle
- Ne garantit pas l'ordre
- Plus rapide mais moins sûr

#### updateStrategy

```yaml
spec:
  updateStrategy:
    type: RollingUpdate    # ou OnDelete
    rollingUpdate:
      partition: 0         # Pods >= partition sont mis à jour
```

**RollingUpdate** (défaut) :
- Mise à jour automatique des pods
- Dans l'ordre inverse (n-1 → 0)

**OnDelete** :
- Les pods ne sont mis à jour que quand ils sont supprimés manuellement
- Contrôle total sur le moment de la mise à jour

## Exemple complet : Déploiement d'un cluster Redis

Voyons un exemple concret de StatefulSet avec Redis :

```yaml
# Headless Service pour Redis
apiVersion: v1
kind: Service
metadata:
  name: redis
  labels:
    app: redis
spec:
  ports:
  - port: 6379
    name: redis
  clusterIP: None    # Headless !
  selector:
    app: redis
---
# StatefulSet Redis
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
          name: redis
        command:
        - redis-server
        args:
        - --appendonly
        - "yes"
        - --dir
        - /data
        volumeMounts:
        - name: data
          mountPath: /data
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          tcpSocket:
            port: 6379
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 5
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: microk8s-hostpath
      resources:
        requests:
          storage: 5Gi
```

**Déploiement** :
```bash
microk8s kubectl apply -f redis-statefulset.yaml
```

**Ce qui est créé** :

1. **Headless Service** : `redis`
2. **Pods** :
   - `redis-0` avec DNS `redis-0.redis.default.svc.cluster.local`
   - `redis-1` avec DNS `redis-1.redis.default.svc.cluster.local`
   - `redis-2` avec DNS `redis-2.redis.default.svc.cluster.local`
3. **PVCs** :
   - `data-redis-0` (5Gi)
   - `data-redis-1` (5Gi)
   - `data-redis-2` (5Gi)

**Observation du déploiement** :
```bash
# Regarder les pods se créer séquentiellement
watch microk8s kubectl get pods

# Résultat :
# redis-0   1/1   Running   (démarre d'abord)
# redis-1   0/1   Pending   (attend que redis-0 soit Ready)
# ...
# redis-1   1/1   Running   (maintenant démarre)
# redis-2   0/1   Pending   (attend que redis-1 soit Ready)
# ...
# redis-2   1/1   Running
```

## Exemple complet : Base de données PostgreSQL

Un exemple plus élaboré avec PostgreSQL :

```yaml
# Service Headless
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  ports:
  - port: 5432
    name: postgres
  clusterIP: None
  selector:
    app: postgres
---
# ConfigMap pour la configuration PostgreSQL
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
data:
  postgresql.conf: |
    max_connections = 100
    shared_buffers = 128MB
    effective_cache_size = 256MB
    maintenance_work_mem = 64MB
    checkpoint_completion_target = 0.9
    wal_buffers = 16MB
    default_statistics_target = 100
    random_page_cost = 1.1
    effective_io_concurrency = 200
---
# Secret pour le mot de passe
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
stringData:
  password: "MySecurePassword123!"
---
# StatefulSet PostgreSQL
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
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_DB
          value: mydb
        - name: POSTGRES_USER
          value: admin
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        - name: config
          mountPath: /etc/postgresql
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - pg_isready -U admin -d mydb
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - pg_isready -U admin -d mydb
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
      volumes:
      - name: config
        configMap:
          name: postgres-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: microk8s-hostpath
      resources:
        requests:
          storage: 20Gi
```

**Déploiement** :
```bash
microk8s kubectl apply -f postgres-statefulset.yaml
```

**Connexion à PostgreSQL** :
```bash
# Depuis l'intérieur du cluster
microk8s kubectl run psql-client --rm -it --image=postgres:15-alpine -- \
  psql -h postgres-0.postgres.default.svc.cluster.local -U admin -d mydb

# Port-forward pour accès local
microk8s kubectl port-forward postgres-0 5432:5432
psql -h localhost -U admin -d mydb
```

## Gestion des StatefulSets

### Déployer un StatefulSet

```bash
# Créer depuis un fichier
microk8s kubectl apply -f statefulset.yaml

# Vérifier le statut
microk8s kubectl get statefulset

# Voir les pods créés
microk8s kubectl get pods -l app=<nom-app>

# Voir les PVCs créés
microk8s kubectl get pvc
```

### Scaling d'un StatefulSet

#### Scale up (augmenter le nombre de replicas)

```bash
# Via kubectl scale
microk8s kubectl scale statefulset redis --replicas=5

# Ou éditer directement
microk8s kubectl edit statefulset redis
```

**Processus** :
```
redis-0 (existe déjà)
redis-1 (existe déjà)
redis-2 (existe déjà)
redis-3 (créé, attend d'être Ready)
redis-4 (sera créé après redis-3)
```

**Observation** :
```bash
# Observer le scaling
watch microk8s kubectl get pods -l app=redis

# Voir les nouveaux PVCs
microk8s kubectl get pvc
```

#### Scale down (réduire le nombre de replicas)

```bash
# Réduire à 2 replicas
microk8s kubectl scale statefulset redis --replicas=2
```

**Processus** :
```
1. redis-4 supprimé
2. Attendre terminaison complète
3. redis-3 supprimé
4. Attendre terminaison complète
5. redis-2 supprimé
6. redis-0 et redis-1 restent
```

**⚠️ Important** : Les PVCs ne sont **pas supprimés** automatiquement !

```bash
# Les PVCs persistent
microk8s kubectl get pvc
# data-redis-0  (utilisé par redis-0)
# data-redis-1  (utilisé par redis-1)
# data-redis-2  (orphelin, non utilisé)
# data-redis-3  (orphelin, non utilisé)
# data-redis-4  (orphelin, non utilisé)
```

**Nettoyage des PVCs orphelins** :
```bash
# Supprimer manuellement
microk8s kubectl delete pvc data-redis-2 data-redis-3 data-redis-4
```

### Mettre à jour un StatefulSet

#### Rolling Update

Par défaut, les mises à jour se font pod par pod, dans l'ordre inverse :

```bash
# Mettre à jour l'image
microk8s kubectl set image statefulset/redis redis=redis:7.2-alpine

# Ou éditer
microk8s kubectl edit statefulset redis
```

**Processus (pour 3 replicas)** :
```
1. redis-2 mis à jour (dernière ordinal)
2. Attendre que redis-2 soit Ready
3. redis-1 mis à jour
4. Attendre que redis-1 soit Ready
5. redis-0 mis à jour
6. Attendre que redis-0 soit Ready
```

**Observer la mise à jour** :
```bash
# Voir le statut de rollout
microk8s kubectl rollout status statefulset/redis

# Voir l'historique
microk8s kubectl rollout history statefulset/redis
```

#### Mise à jour progressive avec partition

Permet de mettre à jour seulement une partie des pods :

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2    # Seuls les pods >= 2 seront mis à jour
```

```bash
# Appliquer
microk8s kubectl patch statefulset redis -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}'
```

**Résultat** :
```
redis-0   (ancienne version, non mis à jour)
redis-1   (ancienne version, non mis à jour)
redis-2   (nouvelle version, mis à jour)
redis-3   (nouvelle version, mis à jour)
```

**Usage** :
- Tester la nouvelle version sur quelques pods
- Rollout progressif contrôlé
- Canary deployments

#### Annuler une mise à jour

```bash
# Annuler le dernier rollout
microk8s kubectl rollout undo statefulset/redis

# Annuler vers une révision spécifique
microk8s kubectl rollout undo statefulset/redis --to-revision=2
```

### Supprimer un StatefulSet

#### Option 1 : Suppression cascade (défaut)

Supprime le StatefulSet ET les pods :

```bash
microk8s kubectl delete statefulset redis
```

**Résultat** :
- StatefulSet supprimé
- Pods supprimés (dans l'ordre inverse : 2→1→0)
- PVCs **conservés** (protection des données)

#### Option 2 : Suppression non-cascade

Supprime le StatefulSet mais garde les pods :

```bash
microk8s kubectl delete statefulset redis --cascade=orphan
```

**Résultat** :
- StatefulSet supprimé
- Pods continuent de fonctionner
- PVCs conservés

**Usage** : Recréer le StatefulSet sans perturber les pods en cours.

#### Suppression complète (StatefulSet + Pods + PVCs)

```bash
# 1. Supprimer le StatefulSet
microk8s kubectl delete statefulset redis

# 2. Attendre que les pods soient terminés
microk8s kubectl wait --for=delete pod -l app=redis --timeout=60s

# 3. Supprimer les PVCs
microk8s kubectl delete pvc -l app=redis

# Ou manuellement
microk8s kubectl delete pvc data-redis-0 data-redis-1 data-redis-2
```

## Patterns et cas d'usage

### Pattern 1 : Master-Slave avec PostgreSQL

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      initContainers:
      - name: init-postgres
        image: postgres:15-alpine
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Si c'est le pod-0 (master)
          if [[ $HOSTNAME =~ -0$ ]]; then
            echo "I am the master"
            # Configuration master
          else
            echo "I am a replica"
            # Configuration réplica - se connecter au master
            # postgres-0.postgres pour réplication
          fi
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
      containers:
      - name: postgres
        image: postgres:15-alpine
        # ... (configuration du conteneur)
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 20Gi
```

**Points clés** :
- `postgres-0` = Master (écriture)
- `postgres-1`, `postgres-2` = Replicas (lecture)
- Les replicas se connectent au master via `postgres-0.postgres`

### Pattern 2 : Cluster Elasticsearch

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      initContainers:
      - name: increase-vm-max-map
        image: busybox:1.35
        command:
        - sysctl
        - -w
        - vm.max_map_count=262144
        securityContext:
          privileged: true
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
        env:
        - name: cluster.name
          value: k8s-logs
        - name: cluster.initial_master_nodes
          value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
        - name: discovery.seed_hosts
          value: "elasticsearch-0.elasticsearch,elasticsearch-1.elasticsearch,elasticsearch-2.elasticsearch"
        - name: ES_JAVA_OPTS
          value: "-Xms512m -Xmx512m"
        - name: xpack.security.enabled
          value: "false"
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
        resources:
          limits:
            memory: "1Gi"
            cpu: "1000m"
          requests:
            memory: "512Mi"
            cpu: "500m"
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

**Points clés** :
- Cluster de 3 nœuds Elasticsearch
- Chaque nœud découvre les autres via DNS stable
- Formation automatique du cluster

### Pattern 3 : Cache Redis avec réplication

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
data:
  master.conf: |
    bind 0.0.0.0
    protected-mode no
    port 6379
  slave.conf: |
    bind 0.0.0.0
    protected-mode no
    port 6379
    slaveof redis-0.redis.default.svc.cluster.local 6379
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      initContainers:
      - name: config
        image: redis:7-alpine
        command:
        - sh
        - -c
        - |
          if [[ $HOSTNAME =~ -0$ ]]; then
            cp /mnt/master.conf /mnt/redis.conf
          else
            cp /mnt/slave.conf /mnt/redis.conf
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt
        - name: config
          mountPath: /etc/redis
      containers:
      - name: redis
        image: redis:7-alpine
        command:
        - redis-server
        - /etc/redis/redis.conf
        ports:
        - containerPort: 6379
          name: redis
        volumeMounts:
        - name: data
          mountPath: /data
        - name: config
          mountPath: /etc/redis
      volumes:
      - name: conf
        configMap:
          name: redis-config
      - name: config
        emptyDir: {}
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 5Gi
```

**Points clés** :
- `redis-0` = Master
- `redis-1`, `redis-2` = Slaves qui répliquent depuis `redis-0`
- Configuration différente selon l'ordinal

## Bonnes pratiques

### 1. Toujours créer un Headless Service

```yaml
# Service requis AVANT le StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: mon-app
spec:
  clusterIP: None    # Headless !
  selector:
    app: mon-app
  ports:
  - port: 8080
```

### 2. Définir des readiness et liveness probes

```yaml
containers:
- name: postgres
  # ...
  livenessProbe:
    exec:
      command:
      - pg_isready
    initialDelaySeconds: 30
    periodSeconds: 10
  readinessProbe:
    exec:
      command:
      - pg_isready
    initialDelaySeconds: 5
    periodSeconds: 5
```

**Importance** : Les pods suivants ne démarrent que quand le précédent est **Ready**.

### 3. Utiliser des init containers pour la configuration

```yaml
initContainers:
- name: init-config
  image: busybox:1.35
  command:
  - sh
  - -c
  - |
    # Configuration basée sur le nom du pod
    if [[ $HOSTNAME =~ -0$ ]]; then
      echo "Configuration master"
    else
      echo "Configuration replica"
    fi
```

### 4. Définir des ressources appropriées

```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

### 5. Utiliser PodDisruptionBudgets

Protéger contre les interruptions involontaires :

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

**Effet** : Garantit qu'au moins 1 pod reste disponible lors de maintenances.

### 6. Nommer clairement les volumeClaimTemplates

```yaml
volumeClaimTemplates:
- metadata:
    name: data    # Sera data-postgres-0, data-postgres-1, etc.
```

### 7. Sauvegarder régulièrement

Les StatefulSets ne protègent pas contre :
- Suppression accidentelle des PVCs
- Corruption de données
- Erreurs d'application

**Solution** : Mettez en place des sauvegardes automatiques (voir section 6.6).

### 8. Documenter avec des annotations

```yaml
metadata:
  annotations:
    description: "Cluster PostgreSQL - Master sur pod-0"
    replication: "Streaming replication"
    backup-schedule: "daily at 2AM"
    contact: "dba-team@example.com"
```

## Dépannage des StatefulSets

### Problème 1 : Les pods ne démarrent pas séquentiellement

**Symptômes** :
```bash
$ microk8s kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
postgres-0  0/1     Pending   0          5m
postgres-1  0/1     Pending   0          5m
```

**Cause** : Le Headless Service n'existe pas ou est mal configuré.

**Solution** :
```bash
# Vérifier le service
microk8s kubectl get service postgres

# Vérifier qu'il est headless
microk8s kubectl get service postgres -o jsonpath='{.spec.clusterIP}'
# Doit retourner: None
```

### Problème 2 : Un pod est bloqué en "Pending"

**Diagnostic** :
```bash
microk8s kubectl describe pod postgres-0
```

**Causes courantes** :
1. PVC ne peut pas être créé (pas de StorageClass, quota dépassé)
2. Ressources insuffisantes sur le nœud
3. Le pod précédent n'est pas Ready

**Solutions** :
```bash
# Vérifier les PVCs
microk8s kubectl get pvc

# Vérifier les événements
microk8s kubectl get events --sort-by='.lastTimestamp'

# Vérifier le pod précédent
microk8s kubectl get pod postgres-0 -o jsonpath='{.status.conditions}'
```

### Problème 3 : Un pod se redémarre en boucle

**Diagnostic** :
```bash
microk8s kubectl logs postgres-1 --previous
microk8s kubectl describe pod postgres-1
```

**Causes courantes** :
1. Problème de configuration
2. Ne peut pas se connecter au master
3. Problème de permissions sur le volume

**Solution** :
```bash
# Vérifier les logs
microk8s kubectl logs postgres-1

# Tester la connectivité réseau
microk8s kubectl exec postgres-1 -- ping postgres-0.postgres
microk8s kubectl exec postgres-1 -- nslookup postgres-0.postgres
```

### Problème 4 : Les PVCs ne sont pas créés

**Symptômes** :
```bash
$ microk8s kubectl get pvc
No resources found
```

**Causes** :
1. StorageClass inexistante ou incorrecte
2. Pas de provisioner configuré

**Solution** :
```bash
# Vérifier les StorageClasses
microk8s kubectl get storageclass

# Pour MicroK8s, activer l'addon
microk8s enable hostpath-storage

# Vérifier le StatefulSet
microk8s kubectl describe statefulset postgres | grep -A5 "Volume Claims"
```

### Problème 5 : Mise à jour bloquée

**Symptômes** : Le rollout ne progresse pas.

**Diagnostic** :
```bash
microk8s kubectl rollout status statefulset/postgres
```

**Causes** :
- Un pod ne devient jamais Ready
- Erreur dans la nouvelle version

**Solution** :
```bash
# Voir les pods
microk8s kubectl get pods -l app=postgres

# Vérifier le pod problématique
microk8s kubectl describe pod postgres-2
microk8s kubectl logs postgres-2

# Annuler le rollout si nécessaire
microk8s kubectl rollout undo statefulset/postgres
```

## Commandes essentielles pour les StatefulSets

```bash
# Créer
microk8s kubectl apply -f statefulset.yaml

# Lister
microk8s kubectl get statefulset
microk8s kubectl get sts    # Abréviation

# Détails
microk8s kubectl describe statefulset <nom>

# Voir les pods
microk8s kubectl get pods -l app=<nom>

# Scaling
microk8s kubectl scale statefulset <nom> --replicas=5

# Mise à jour
microk8s kubectl set image statefulset/<nom> <container>=<image>

# Rollout status
microk8s kubectl rollout status statefulset/<nom>

# Rollback
microk8s kubectl rollout undo statefulset/<nom>

# Supprimer (garde les PVCs)
microk8s kubectl delete statefulset <nom>

# Supprimer (et les PVCs)
microk8s kubectl delete statefulset <nom>
microk8s kubectl delete pvc -l app=<nom>

# Voir les PVCs
microk8s kubectl get pvc -l app=<nom>

# Exec dans un pod spécifique
microk8s kubectl exec -it <nom>-0 -- /bin/bash

# Logs
microk8s kubectl logs -f <nom>-0

# Port-forward
microk8s kubectl port-forward <nom>-0 8080:8080
```

## StatefulSets vs Deployments : Quand utiliser quoi ?

### Utilisez un Deployment quand :
- ✅ Application stateless
- ✅ Toutes les instances sont identiques
- ✅ Pas besoin de stockage persistant par instance
- ✅ L'ordre de démarrage n'importe pas
- ✅ Exemples : API REST, serveurs web, workers

### Utilisez un StatefulSet quand :
- ✅ Application stateful
- ✅ Chaque instance a une identité unique
- ✅ Besoin de stockage persistant dédié par instance
- ✅ L'ordre de démarrage/arrêt est important
- ✅ Instances doivent se découvrir mutuellement
- ✅ Exemples : bases de données, systèmes de cache distribués, systèmes de messaging

## Limitations des StatefulSets

### Limitations à connaître

1. **Pas de scaling automatique par défaut**
   - HPA (Horizontal Pod Autoscaler) fonctionne mais avec précautions
   - Le scaling automatique d'applications stateful est complexe

2. **Suppression des PVCs manuelle**
   - Les PVCs ne sont jamais supprimés automatiquement
   - Protection contre la perte de données, mais nécessite nettoyage manuel

3. **Complexité de la migration**
   - Déplacer un StatefulSet est plus complexe qu'un Deployment
   - Nécessite planification pour conserver l'identité et les données

4. **Responsabilité de la configuration**
   - Le StatefulSet ne configure pas automatiquement les applications
   - Vous devez gérer la configuration master/replica vous-même

5. **Pas de Load Balancing automatique intelligent**
   - Le Headless Service ne fait pas de load balancing
   - Vous devez implémenter la logique de routage (master vs replicas)

## Résumé visuel : Architecture complète

```
┌────────────────────────────────────────────────────────────┐
│                     STATEFULSET                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Headless Service: postgres                          │  │
│  │  clusterIP: None                                     │  │
│  └──────────────────────────────────────────────────────┘  │
│                          │                                 │
│          ┌───────────────┼───────────────┐                 │
│          │               │               │                 │
│     ┌────▼─────┐    ┌────▼─────┐    ┌───▼──────┐           │
│     │postgres-0│    │postgres-1│    │postgres-2│           │
│     │  Master  │    │ Replica  │    │ Replica  │           │
│     └────┬─────┘    └────┬─────┘    └────┬─────┘           │
│          │               │               │                 │
│     ┌────▼────┐     ┌────▼────┐     ┌────▼────┐            │
│     │PVC-0    │     │PVC-1    │     │PVC-2    │            │
│     │20Gi     │     │20Gi     │     │20Gi     │            │
│     └────┬────┘     └────┬────┘     └────┬────┘            │
│          │               │               │                 │
│     ┌────▼────┐     ┌────▼────┐     ┌────▼────┐            │
│     │PV-0     │     │PV-1     │     │PV-2     │            │
│     │Stockage │     │Stockage │     │Stockage │            │
│     └─────────┘     └─────────┘     └─────────┘            │
│                                                            │
│  Propriétés:                                               │
│  • Noms stables: postgres-0, postgres-1, postgres-2        │
│  • DNS stables: postgres-X.postgres.default.svc...         │
│  • Stockage dédié: chaque pod a son PVC                    │
│  • Ordre garanti: 0 → 1 → 2 au déploiement                 │
│  • Ordre inverse: 2 → 1 → 0 à la suppression               │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

## Conclusion

Les StatefulSets sont un outil puissant pour déployer des applications stateful dans Kubernetes. Ils fournissent les garanties nécessaires pour exécuter des bases de données, des systèmes de cache distribués et d'autres applications qui nécessitent :
- Une identité stable
- Un stockage dédié
- Un ordre de déploiement prévisible

**Points clés à retenir** :

- Les StatefulSets créent des pods avec des identités ordonnées et stables
- Chaque pod a son propre stockage persistant via volumeClaimTemplates
- Un Headless Service est requis pour fournir l'identité réseau
- Le déploiement et le scaling sont séquentiels et ordonnés
- Les PVCs persistent même après suppression du StatefulSet
- Parfait pour les bases de données et applications distribuées

Avec MicroK8s, déployer des StatefulSets est simple grâce au provisionnement dynamique intégré. Vous pouvez maintenant exécuter des applications stateful sophistiquées dans votre environnement Kubernetes !

---

**Points clés à retenir** :

✅ **StatefulSet** = Pour applications stateful nécessitant identité stable
✅ **Identité stable** = Noms prévisibles (app-0, app-1, app-2)
✅ **Headless Service** = Requis pour DNS stable
✅ **volumeClaimTemplates** = Crée automatiquement un PVC par pod
✅ **Ordre garanti** = Déploiement séquentiel (0→1→2)
✅ **Persistence** = PVCs conservés même après suppression
✅ **Cas d'usage** = Bases de données, caches distribués, systèmes stateful
✅ **vs Deployment** = Utilisez StatefulSet quand l'identité et l'ordre comptent

⏭️ [Exemples pratiques (bases de données, etc.)](/06-stockage-persistant/08-exemples-pratiques.md)
