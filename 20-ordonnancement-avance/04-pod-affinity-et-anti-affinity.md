🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.4 Pod Affinity et Anti-Affinity

## Introduction

Jusqu'à présent, nous avons vu comment contrôler le placement des Pods par rapport aux **nœuds** avec les Node Selectors et Node Affinity. Mais que faire si vous voulez contrôler le placement des Pods **les uns par rapport aux autres** ?

C'est exactement ce que permettent les **Pod Affinity** et **Pod Anti-Affinity**. Ces mécanismes vous permettent de dire des choses comme :
- "Ce Pod doit être sur le même nœud que ces autres Pods"
- "Ce Pod doit être dans la même zone que ces autres Pods"
- "Ce Pod ne doit jamais être sur le même nœud que ces autres Pods"

Dans cette section, nous allons explorer ces concepts puissants qui ouvrent de nouvelles possibilités pour orchestrer vos applications.

---

## Différence Fondamentale : Node vs Pod Affinity

### Node Affinity
- **Relation** : Pod → Nœud
- **Question** : "Sur quel **type de nœud** ce Pod doit-il tourner ?"
- **Exemple** : "Ce Pod doit être sur un nœud avec GPU"

### Pod Affinity
- **Relation** : Pod → Autres Pods
- **Question** : "Ce Pod doit-il être **proche** ou **éloigné** d'autres Pods ?"
- **Exemple** : "Ce Pod doit être sur le même nœud que les Pods du cache Redis"

---

## Pourquoi Utiliser Pod Affinity/Anti-Affinity ?

### Cas d'Usage pour Pod Affinity (Rapprocher les Pods)

#### 1. **Colocalisation pour la Performance**
Placer une application web et son cache sur le même nœud pour minimiser la latence réseau.

```
Nœud 1: [Web App] + [Redis Cache] ← Faible latence
Nœud 2: [Other App]
```

#### 2. **Partage de Données Locales**
Plusieurs Pods qui partagent un volume local doivent être sur le même nœud.

#### 3. **Batch Processing**
Placer les workers d'un job de traitement près de leur source de données.

### Cas d'Usage pour Pod Anti-Affinity (Éloigner les Pods)

#### 1. **Haute Disponibilité**
Distribuer les réplicas d'une application sur différents nœuds pour éviter qu'une panne de nœud ne coupe tout le service.

```
Nœud 1: [App Replica 1]
Nœud 2: [App Replica 2]
Nœud 3: [App Replica 3]
```

#### 2. **Éviter la Contention de Ressources**
Empêcher que plusieurs applications gourmandes en CPU tournent sur le même nœud.

#### 3. **Isolation de Sécurité**
S'assurer que certaines applications critiques ne tournent jamais ensemble.

#### 4. **Distribution Géographique**
Répartir les Pods sur différentes zones de disponibilité.

---

## Les Deux Types de Pod Affinity

Comme pour les Node Affinity, il existe deux types :

### 1. Required (Requis)

**Nom complet** : `requiredDuringSchedulingIgnoredDuringExecution`

**Comportement** :
- Le Pod **doit** être placé selon la règle
- Si impossible, le Pod reste **Pending**

### 2. Preferred (Préféré)

**Nom complet** : `preferredDuringSchedulingIgnoredDuringExecution`

**Comportement** :
- Le scheduler **préfère** placer le Pod selon la règle
- Si impossible, le Pod est placé ailleurs
- Utilise un système de **poids** (weight) comme pour Node Affinity

---

## Concept Clé : La Topologie (topologyKey)

Le **topologyKey** est le concept le plus important à comprendre pour les Pod Affinity/Anti-Affinity.

### Qu'est-ce qu'une Topologie ?

Une topologie définit un **niveau de séparation** dans votre infrastructure. Elle est représentée par un label sur les nœuds.

### Exemples de Topologies Courantes

#### 1. `kubernetes.io/hostname`
- **Signification** : Le nœud lui-même
- **Utilisation** : "Sur le même nœud" ou "Sur des nœuds différents"

```
Nœud 1 (hostname=node1)
Nœud 2 (hostname=node2)
Nœud 3 (hostname=node3)
```

#### 2. `topology.kubernetes.io/zone`
- **Signification** : La zone de disponibilité (datacenter, rack, etc.)
- **Utilisation** : "Dans la même zone" ou "Dans des zones différentes"

```
Zone A: [Nœud 1, Nœud 2]
Zone B: [Nœud 3, Nœud 4]
Zone C: [Nœud 5, Nœud 6]
```

#### 3. `topology.kubernetes.io/region`
- **Signification** : La région géographique
- **Utilisation** : "Dans la même région" ou "Dans des régions différentes"

```
Région EU: [Zone EU-West, Zone EU-East]
Région US: [Zone US-West, Zone US-East]
```

### Comment Fonctionne le topologyKey ?

Le topologyKey définit **à quel niveau** les Pods doivent être ensemble ou séparés.

**Exemple avec Pod Affinity** :
- `topologyKey: kubernetes.io/hostname` → "Sur le **même nœud**"
- `topologyKey: topology.kubernetes.io/zone` → "Dans la **même zone**"

**Exemple avec Pod Anti-Affinity** :
- `topologyKey: kubernetes.io/hostname` → "Sur des **nœuds différents**"
- `topologyKey: topology.kubernetes.io/zone` → "Dans des **zones différentes**"

---

## Syntaxe de Base

### Structure Générale - Pod Affinity

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - cache
        topologyKey: kubernetes.io/hostname
  containers:
  - name: mon-conteneur
    image: nginx
```

### Structure Générale - Pod Anti-Affinity

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - mon-app
        topologyKey: kubernetes.io/hostname
  containers:
  - name: mon-conteneur
    image: nginx
```

---

## Pod Affinity - Exemples Pratiques

### Exemple 1 : Application Web et Cache sur le Même Nœud

Vous voulez que votre application web soit toujours sur le même nœud que Redis pour minimiser la latence.

**Déploiement du Cache Redis** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
      role: cache
  template:
    metadata:
      labels:
        app: redis
        role: cache
    spec:
      containers:
      - name: redis
        image: redis:7
        ports:
        - containerPort: 6379
```

**Déploiement de l'Application Web** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - redis
              - key: role
                operator: In
                values:
                - cache
            topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx:latest
```

**Ce que fait ce manifeste** :
- Les Pods `web-app` **doivent** être sur le même nœud (`topologyKey: kubernetes.io/hostname`) que les Pods ayant les labels `app=redis` **et** `role=cache`
- Si Redis est sur `node1`, toutes les réplicas de web-app iront sur `node1`

### Exemple 2 : Dans la Même Zone (Plus Flexible)

Au lieu d'exiger le même nœud, vous voulez juste être dans la même zone.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-flexible
spec:
  replicas: 5
  selector:
    matchLabels:
      app: web-flexible
  template:
    metadata:
      labels:
        app: web-flexible
    spec:
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - redis
              topologyKey: topology.kubernetes.io/zone
      containers:
      - name: nginx
        image: nginx:latest
```

**Ce que fait ce manifeste** :
- Le scheduler **préfère** placer web-app dans la même **zone** que Redis
- Si impossible (zone pleine), les Pods iront dans une autre zone
- Les Pods peuvent être sur des nœuds différents, tant qu'ils sont dans la même zone

### Exemple 3 : Colocalisation de Plusieurs Services

Un frontend, un backend et un cache qui doivent tous être ensemble.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  labels:
    tier: frontend
    app: shop
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      # Doit être avec le backend
      - labelSelector:
          matchLabels:
            tier: backend
            app: shop
        topologyKey: kubernetes.io/hostname
      # Doit être avec le cache
      - labelSelector:
          matchLabels:
            tier: cache
            app: shop
        topologyKey: kubernetes.io/hostname
  containers:
  - name: nginx
    image: nginx
```

**Ce que fait ce manifeste** :
- Le frontend doit être sur le même nœud que le backend **ET** le cache
- Tous les composants de l'application `shop` seront colocalisés

---

## Pod Anti-Affinity - Exemples Pratiques

### Exemple 1 : Haute Disponibilité - Réplicas sur Différents Nœuds

C'est l'usage le plus courant de Pod Anti-Affinity.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-ha
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mon-app-ha
  template:
    metadata:
      labels:
        app: mon-app-ha
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - mon-app-ha
            topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: mon-app:latest
```

**Ce que fait ce manifeste** :
- Chaque réplica de `mon-app-ha` **doit** être sur un nœud différent
- Si vous avez 3 nœuds : un Pod par nœud
- Si vous avez 2 nœuds : seulement 2 Pods pourront être ordonnancés (le 3e restera Pending)

**Diagramme** :
```
Nœud 1: [mon-app-ha Pod 1]
Nœud 2: [mon-app-ha Pod 2]
Nœud 3: [mon-app-ha Pod 3]
```

### Exemple 2 : Haute Disponibilité - Préférence Douce

Version plus flexible qui préfère distribuer mais ne bloque pas le déploiement.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-ha-flexible
spec:
  replicas: 5
  selector:
    matchLabels:
      app: mon-app-flexible
  template:
    metadata:
      labels:
        app: mon-app-flexible
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - mon-app-flexible
              topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: mon-app:latest
```

**Ce que fait ce manifeste** :
- Le scheduler **préfère** distribuer les Pods sur différents nœuds
- Si vous avez 3 nœuds, il mettra idéalement 2 Pods sur certains nœuds
- Tous les 5 Pods seront déployés même si vous n'avez qu'un seul nœud

### Exemple 3 : Distribution par Zone

Pour une vraie haute disponibilité, distribuez sur différentes zones.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-multi-zone
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app-critique
  template:
    metadata:
      labels:
        app: app-critique
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - app-critique
            topologyKey: topology.kubernetes.io/zone
      containers:
      - name: app
        image: app-critique:v1
```

**Ce que fait ce manifeste** :
- Chaque réplica doit être dans une **zone différente**
- Protection contre la panne d'une zone entière

**Diagramme** :
```
Zone A: [app-critique Pod 1]
Zone B: [app-critique Pod 2]
Zone C: [app-critique Pod 3]
```

### Exemple 4 : Isoler les Environnements

Empêcher que les Pods de production et de test tournent ensemble.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mon-app
      env: production
  template:
    metadata:
      labels:
        app: mon-app
        env: production
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: env
                operator: In
                values:
                - development
                - test
            topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: mon-app:prod
```

**Ce que fait ce manifeste** :
- Les Pods de production ne seront **jamais** sur le même nœud que les Pods de dev ou test
- Garantit l'isolation des environnements

---

## Combinaison Pod Affinity + Anti-Affinity

Vous pouvez combiner affinity et anti-affinity pour créer des règles sophistiquées.

### Exemple : Application Web Optimisée

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-optimized
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-optimized
  template:
    metadata:
      labels:
        app: web-optimized
        tier: frontend
    spec:
      affinity:
        # Affinity : Être proche du cache
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - redis
              topologyKey: kubernetes.io/hostname
        # Anti-Affinity : Réplicas sur différents nœuds
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 90
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web-optimized
              topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx:latest
```

**Ce que fait ce manifeste** :
- **Préférence 1 (poids 100)** : Être sur le même nœud que Redis
- **Préférence 2 (poids 90)** : Les réplicas sur des nœuds différents
- Le scheduler équilibrera ces deux objectifs

---

## Combinaison avec Node Affinity

Vous pouvez également combiner Pod Affinity/Anti-Affinity avec Node Affinity.

### Exemple : Architecture Multi-Contraintes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-complexe
spec:
  replicas: 3
  selector:
    matchLabels:
      app: complexe
  template:
    metadata:
      labels:
        app: complexe
    spec:
      affinity:
        # Node Affinity : Seulement sur des nœuds production avec SSD
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
        # Pod Affinity : Proche du cache
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - cache
              topologyKey: kubernetes.io/hostname
        # Pod Anti-Affinity : Réplicas distribués
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 70
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - complexe
              topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: mon-app:latest
```

**Ce manifeste combine** :
1. **Node Affinity** : Doit être sur nœud production + SSD
2. **Pod Affinity** : Préfère être avec le cache
3. **Pod Anti-Affinity** : Préfère être distribué

---

## Cas d'Usage Pratiques avec MicroK8s

### Cas 1 : Cluster de Base de Données avec Haute Disponibilité

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-cluster
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
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - postgres
            topologyKey: kubernetes.io/hostname
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_PASSWORD
          value: "password"
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
          storage: 10Gi
```

**Résultat** :
- Chaque instance PostgreSQL sur un nœud différent
- Si un nœud tombe, les 2 autres instances continuent
- Protection contre la défaillance d'un nœud

### Cas 2 : Application avec Workers et Coordinateur

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coordinator
spec:
  replicas: 1
  selector:
    matchLabels:
      app: job-system
      role: coordinator
  template:
    metadata:
      labels:
        app: job-system
        role: coordinator
    spec:
      containers:
      - name: coordinator
        image: coordinator:latest
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workers
spec:
  replicas: 5
  selector:
    matchLabels:
      app: job-system
      role: worker
  template:
    metadata:
      labels:
        app: job-system
        role: worker
    spec:
      affinity:
        # Les workers préfèrent être avec le coordinateur
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: role
                  operator: In
                  values:
                  - coordinator
              topologyKey: kubernetes.io/hostname
        # Mais les workers préfèrent être distribués entre eux
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 50
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: role
                  operator: In
                  values:
                  - worker
              topologyKey: kubernetes.io/hostname
      containers:
      - name: worker
        image: worker:latest
```

**Résultat** :
- Workers préfèrent être sur le même nœud que le coordinateur (faible latence)
- Mais ils préfèrent aussi être distribués entre eux (résilience)
- Le scheduler équilibre ces deux objectifs

### Cas 3 : Stack Monitoring (Prometheus + Grafana)

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
      component: monitoring
  template:
    metadata:
      labels:
        app: prometheus
        component: monitoring
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:latest
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
      component: monitoring
  template:
    metadata:
      labels:
        app: grafana
        component: monitoring
    spec:
      affinity:
        # Grafana préfère être avec Prometheus
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - prometheus
              topologyKey: kubernetes.io/hostname
      containers:
      - name: grafana
        image: grafana/grafana:latest
```

**Résultat** :
- Grafana préfère être sur le même nœud que Prometheus
- Réduit la latence des requêtes de données
- Si impossible, Grafana sera quand même déployé ailleurs

---

## Namespace et Pod Affinity

Par défaut, les Pod Affinity/Anti-Affinity s'appliquent aux Pods du **même namespace**.

### Spécifier des Namespaces Différents

```yaml
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - cache
      topologyKey: kubernetes.io/hostname
      namespaces:
      - production
      - staging
```

**Ce que fait ce manifeste** :
- Cherche les Pods avec `app=cache` dans les namespaces `production` **et** `staging`

### Tous les Namespaces

Pour chercher dans tous les namespaces, utilisez un selector vide :

```yaml
namespaceSelector: {}
```

---

## Limitations et Contraintes

### 1. Performance du Scheduler

Les règles Pod Affinity/Anti-Affinity sont coûteuses en calcul :
- Le scheduler doit examiner tous les Pods existants
- Plus vous avez de Pods, plus c'est lent
- Limité à environ 5000 Pods dans un cluster

### 2. Required Anti-Affinity peut Bloquer des Déploiements

```yaml
# ⚠️ Attention avec cette configuration
spec:
  replicas: 10  # Vous voulez 10 réplicas
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: mon-app
        topologyKey: kubernetes.io/hostname
```

**Problème** :
- Si vous n'avez que 5 nœuds, seulement 5 Pods seront déployés
- Les 5 autres resteront Pending indéfiniment

**Solution** : Utilisez `preferred` au lieu de `required`.

### 3. TopologyKey Obligatoire

Le `topologyKey` est toujours obligatoire pour Pod Affinity/Anti-Affinity.

```yaml
# ❌ Ne fonctionne pas
podAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchLabels:
        app: cache
  # topologyKey manquant !
```

```yaml
# ✅ Correct
podAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchLabels:
        app: cache
    topologyKey: kubernetes.io/hostname
```

---

## Débogage et Diagnostics

### Pod en Pending à Cause de l'Affinity

```bash
kubectl describe pod <pod-name>
```

Événements typiques :
```
Events:
  Type     Reason            Message
  ----     ------            -------
  Warning  FailedScheduling  0/3 nodes are available:
                             3 node(s) didn't match pod affinity rules.
```

Ou pour anti-affinity :
```
  Warning  FailedScheduling  0/3 nodes are available:
                             3 node(s) didn't match pod anti-affinity rules.
```

### Vérifier les Labels des Pods Existants

```bash
# Voir tous les Pods avec leurs labels
kubectl get pods --show-labels

# Filtrer par label
kubectl get pods -l app=redis

# Dans un namespace spécifique
kubectl get pods -n production -l app=cache
```

### Voir la Distribution des Pods

```bash
# Voir sur quels nœuds les Pods sont déployés
kubectl get pods -o wide

# Compter les Pods par nœud
kubectl get pods -o wide --no-headers | awk '{print $7}' | sort | uniq -c
```

---

## Bonnes Pratiques

### 1. Préférez Preferred à Required

```yaml
# ✅ Recommandé : Flexible
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    podAffinityTerm:
      labelSelector:
        matchLabels:
          app: mon-app
      topologyKey: kubernetes.io/hostname
```

```yaml
# ⚠️ À éviter : Peut bloquer des déploiements
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchLabels:
        app: mon-app
    topologyKey: kubernetes.io/hostname
```

### 2. Utilisez des Labels Précis

```yaml
# ✅ Bon : Labels spécifiques
labelSelector:
  matchLabels:
    app: mon-app
    version: v2
    tier: frontend
```

```yaml
# ❌ Mauvais : Labels trop génériques
labelSelector:
  matchLabels:
    app: app  # Trop vague
```

### 3. Planifiez pour la Croissance

Si vous utilisez `required` anti-affinity, assurez-vous d'avoir assez de nœuds :

- 3 réplicas avec anti-affinity → minimum 3 nœuds
- 5 réplicas avec anti-affinity → minimum 5 nœuds

### 4. Surveillez les Pods Pending

```bash
# Script de surveillance
watch 'kubectl get pods --all-namespaces | grep Pending'
```

Si vous voyez beaucoup de Pods Pending à cause d'affinity, revoyez vos règles.

### 5. Documentez vos Règles

```yaml
metadata:
  name: app-web
  annotations:
    description: "Pod Affinity with Redis for low latency"
    affinity-rules: "Prefers same node as Redis, distributed replicas"
spec:
  affinity:
    # ...
```

### 6. Testez en Environnement de Dev

Avant de déployer en production, testez vos règles d'affinity/anti-affinity en dev :

```bash
# Créer des nœuds fictifs avec labels
kubectl label nodes node1 topology.kubernetes.io/zone=zone-a
kubectl label nodes node2 topology.kubernetes.io/zone=zone-b
kubectl label nodes node3 topology.kubernetes.io/zone=zone-c

# Déployer et vérifier
kubectl apply -f mon-deployment.yaml
kubectl get pods -o wide
```

### 7. Attention aux Combinaisons Contradictoires

```yaml
# ⚠️ Règles potentiellement contradictoires
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: cache
        topologyKey: kubernetes.io/hostname  # Doit être avec cache
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: cache
        topologyKey: kubernetes.io/hostname  # Ne doit pas être avec cache !?
```

Cela créera une situation impossible à satisfaire.

---

## Tableau Récapitulatif

| Type | Usage | TopologyKey Commun | Cas d'Usage |
|------|-------|-------------------|-------------|
| **Pod Affinity** | Rapprocher les Pods | `kubernetes.io/hostname` | Performance, partage de données |
| **Pod Anti-Affinity** | Éloigner les Pods | `kubernetes.io/hostname` | Haute disponibilité |
| **Pod Anti-Affinity** | Distribuer par zone | `topology.kubernetes.io/zone` | Résilience géographique |
| **Required** | Règle obligatoire | Tout | Contraintes strictes |
| **Preferred** | Règle souple | Tout | Préférences flexibles |

---

## Exemple Complet : Application E-Commerce

Voici une architecture complète d'une application e-commerce avec toutes les bonnes pratiques.

### Architecture

```
Frontend (3 réplicas) ─┐
                       ├─> Redis Cache (1 instance)
Backend API (5 reps)  ─┘
                       └─> PostgreSQL (3 instances HA)
```

### 1. Redis Cache

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
      tier: cache
      system: ecommerce
  template:
    metadata:
      labels:
        app: redis
        tier: cache
        system: ecommerce
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
```

### 2. Frontend

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
      tier: web
      system: ecommerce
  template:
    metadata:
      labels:
        app: frontend
        tier: web
        system: ecommerce
    spec:
      affinity:
        # Affinity : Proche du cache Redis
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: redis
                  tier: cache
              topologyKey: kubernetes.io/hostname
        # Anti-Affinity : Réplicas distribués
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: frontend
              topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
```

### 3. Backend API

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
spec:
  replicas: 5
  selector:
    matchLabels:
      app: backend
      tier: api
      system: ecommerce
  template:
    metadata:
      labels:
        app: backend
        tier: api
        system: ecommerce
    spec:
      affinity:
        # Affinity : Proche du cache Redis
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 90
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: redis
              topologyKey: kubernetes.io/hostname
        # Anti-Affinity : Réplicas distribués
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: backend
              topologyKey: kubernetes.io/hostname
      containers:
      - name: api
        image: mon-api:v1
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
```

### 4. PostgreSQL (Haute Disponibilité)

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
      tier: database
      system: ecommerce
  template:
    metadata:
      labels:
        app: postgres
        tier: database
        system: ecommerce
    spec:
      affinity:
        # Anti-Affinity : Instances sur différents nœuds
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: postgres
            topologyKey: kubernetes.io/hostname
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            cpu: "1000m"
            memory: "2Gi"
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 20Gi
```

---

## Points Clés à Retenir

1. **Pod Affinity** = Rapprocher les Pods (même nœud, même zone)
2. **Pod Anti-Affinity** = Éloigner les Pods (nœuds différents, zones différentes)
3. **TopologyKey** = Définit le niveau de séparation
4. **Required vs Preferred** = Obligatoire vs Souple
5. **Haute Disponibilité** = Anti-affinity + `topologyKey: kubernetes.io/hostname`
6. **Performance** = Affinity pour colocaliser les services liés
7. **Préférez Preferred** = Plus flexible et résilient

---

## Conclusion

Les Pod Affinity et Anti-Affinity sont des outils puissants qui vous permettent de contrôler finement comment vos Pods sont distribués dans votre cluster. Ils complètent parfaitement les Node Affinity en ajoutant une dimension relationnelle entre les Pods.

**Cas d'usage principaux** :
- **Pod Affinity** : Optimiser les performances en colocalisant les services
- **Pod Anti-Affinity** : Garantir la haute disponibilité en distribuant les réplicas

En combinant intelligemment ces mécanismes avec Node Affinity et les resource requests, vous pouvez créer des architectures Kubernetes sophistiquées, performantes et résilientes.

Dans la prochaine section, nous découvrirons les **Taints et Tolerations**, un autre mécanisme d'ordonnancement qui fonctionne dans le sens inverse : au lieu de dire où les Pods **doivent** aller, les Taints disent où ils **ne peuvent pas** aller (sauf tolérance spéciale).

---

## Pour Aller Plus Loin

- **Documentation officielle** : [Pod Affinity and Anti-Affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity)
- **Topology Spread Constraints** : Un mécanisme plus récent et plus simple pour distribuer les Pods
- **Descheduler** : Pour rééquilibrer les Pods qui tournent déjà

---

**Prochain chapitre** : 20.5 Taints et Tolerations - Marquer les nœuds comme spéciaux et contrôler qui peut y accéder

⏭️ [Taints et Tolerations](/20-ordonnancement-avance/05-taints-et-tolerations.md)
