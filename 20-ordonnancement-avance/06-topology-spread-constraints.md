🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.6 Topology Spread Constraints

## Introduction

Vous avez découvert la **Pod Anti-Affinity** pour distribuer vos Pods sur différents nœuds ou zones. Mais cette approche peut être complexe et verbeuse. Kubernetes propose un mécanisme plus moderne et plus simple : les **Topology Spread Constraints** (contraintes de distribution topologique).

Les Topology Spread Constraints permettent de contrôler comment vos Pods sont répartis à travers votre infrastructure (nœuds, zones, régions) de manière **uniforme** et **équilibrée**, avec une syntaxe beaucoup plus simple et intuitive.

### Analogie Simple

Imaginez que vous distribuez des cartes à des joueurs :
- **Pod Anti-Affinity** : "Cette carte ne peut pas être à côté d'une autre carte identique"
- **Topology Spread Constraints** : "Distribuez les cartes de manière équitable entre tous les joueurs, avec maximum 1 carte d'écart"

---

## Pourquoi Topology Spread Constraints ?

### Limitations de Pod Anti-Affinity

Pod Anti-Affinity est puissant mais présente des défis :

```yaml
# ❌ Complexe et verbeux
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchLabels:
        app: mon-app
    topologyKey: kubernetes.io/hostname
```

**Problèmes** :
- Syntaxe verbeuse
- Distribution "tout ou rien" (required) ou "au mieux" (preferred)
- Difficile de contrôler finement l'équilibre
- Pas de notion de "max d'écart" entre les nœuds

### Avantages de Topology Spread Constraints

```yaml
# ✅ Plus simple et plus puissant
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: mon-app
```

**Avantages** :
- Syntaxe plus claire et concise
- Contrôle fin de l'équilibre avec `maxSkew`
- Distribution vraiment uniforme
- Plus flexible et prévisible

---

## Concepts Fondamentaux

### 1. Topologie (Topology)

Une **topologie** définit un niveau de séparation dans votre infrastructure, représenté par un label sur les nœuds.

**Exemples** :
- `kubernetes.io/hostname` → Nœud individuel
- `topology.kubernetes.io/zone` → Zone de disponibilité
- `topology.kubernetes.io/region` → Région géographique

### 2. Domaine de Topologie (Topology Domain)

Un **domaine** est un ensemble de nœuds partageant la même valeur pour une clé de topologie.

**Exemple** :
```
topologyKey: topology.kubernetes.io/zone

Zone A: [node1, node2]     ← Un domaine
Zone B: [node3, node4]     ← Un autre domaine
Zone C: [node5]            ← Encore un autre domaine
```

### 3. Skew (Écart)

Le **skew** est la différence entre le nombre de Pods dans différents domaines.

**Exemple** :
```
Zone A: 3 Pods
Zone B: 2 Pods
Zone C: 1 Pod

Skew maximum = 3 - 1 = 2
```

### 4. MaxSkew (Écart Maximum)

Le **maxSkew** définit l'écart maximum toléré entre le domaine le plus chargé et le moins chargé.

---

## Structure de Base

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod
spec:
  topologySpreadConstraints:
  - maxSkew: <nombre>
    topologyKey: <clé-de-topologie>
    whenUnsatisfiable: <DoNotSchedule|ScheduleAnyway>
    labelSelector:
      matchLabels:
        <label>: <valeur>
  containers:
  - name: mon-conteneur
    image: nginx
```

### Paramètres Essentiels

#### 1. maxSkew (Obligatoire)

Définit l'écart maximum autorisé entre les domaines.

- `maxSkew: 1` → Distribution très uniforme (différence de 1 Pod max)
- `maxSkew: 2` → Distribution moins stricte (différence de 2 Pods max)
- `maxSkew: 5` → Distribution lâche

**Règle** : Plus le nombre est petit, plus la distribution est uniforme.

#### 2. topologyKey (Obligatoire)

La clé de label qui définit les domaines.

**Valeurs courantes** :
- `kubernetes.io/hostname` → Distribution par nœud
- `topology.kubernetes.io/zone` → Distribution par zone
- `topology.kubernetes.io/region` → Distribution par région

#### 3. whenUnsatisfiable (Obligatoire)

Définit que faire si la contrainte ne peut pas être satisfaite.

**DoNotSchedule** (Strict) :
- Le Pod ne sera **pas** ordonnancé si cela viole la contrainte
- Comportement "hard" (obligatoire)

**ScheduleAnyway** (Flexible) :
- Le Pod sera ordonnancé même si cela viole la contrainte
- Le scheduler favorise quand même une distribution équilibrée
- Comportement "soft" (préférence)

#### 4. labelSelector (Obligatoire)

Identifie les Pods à considérer pour le calcul de la distribution.

```yaml
labelSelector:
  matchLabels:
    app: mon-app
    version: v1
```

---

## Exemples Pratiques

### Exemple 1 : Distribution Uniforme sur les Nœuds

Vous voulez distribuer vos Pods équitablement sur tous les nœuds.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 6
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: web
      containers:
      - name: nginx
        image: nginx:latest
```

**Comportement avec 3 nœuds** :
```
Nœud 1: 2 Pods
Nœud 2: 2 Pods
Nœud 3: 2 Pods
```

**Avec maxSkew: 1** :
- Différence maximale entre nœuds = 1 Pod
- Distribution très équilibrée

**Si vous aviez 7 réplicas** :
```
Nœud 1: 3 Pods  ← Un de plus
Nœud 2: 2 Pods
Nœud 3: 2 Pods
Skew = 3 - 2 = 1 ✅ Respecte maxSkew: 1
```

### Exemple 2 : Distribution par Zone

Distribuer les Pods équitablement entre les zones de disponibilité.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-multi-zone
spec:
  replicas: 9
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: api
      containers:
      - name: api
        image: api:v1
```

**Comportement avec 3 zones** :
```
Zone A: 3 Pods
Zone B: 3 Pods
Zone C: 3 Pods
```

**Haute disponibilité garantie** : Chaque zone a le même nombre de Pods (±1).

### Exemple 3 : Distribution Flexible

Si vous voulez une distribution équilibrée mais sans bloquer le déploiement.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-flexible
spec:
  replicas: 10
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      topologySpreadConstraints:
      - maxSkew: 2
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: worker
      containers:
      - name: worker
        image: worker:latest
```

**Comportement** :
- Le scheduler **essaie** de respecter maxSkew: 2
- Si impossible (ex: pas assez de nœuds), les Pods seront quand même déployés
- `ScheduleAnyway` = préférence, pas obligation

**Avec 3 nœuds et 10 réplicas** :
```
Idéal avec maxSkew: 2
Nœud 1: 4 Pods
Nœud 2: 3 Pods
Nœud 3: 3 Pods
Skew = 4 - 3 = 1 ✅
```

### Exemple 4 : Plusieurs Contraintes

Vous pouvez définir plusieurs contraintes simultanément.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-multi-contraintes
spec:
  replicas: 12
  selector:
    matchLabels:
      app: critical-app
  template:
    metadata:
      labels:
        app: critical-app
    spec:
      topologySpreadConstraints:
      # Contrainte 1 : Distribution par zone
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: critical-app
      # Contrainte 2 : Distribution par nœud
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: critical-app
      containers:
      - name: app
        image: critical-app:v1
```

**Comportement** :
- **Obligatoire** : Distribution équilibrée par zone (DoNotSchedule)
- **Préféré** : Distribution équilibrée par nœud (ScheduleAnyway)

**Exemple avec 3 zones de 2 nœuds chacune** :
```
Zone A:
  Nœud 1: 2 Pods
  Nœud 2: 2 Pods
Zone B:
  Nœud 3: 2 Pods
  Nœud 4: 2 Pods
Zone C:
  Nœud 5: 2 Pods
  Nœud 6: 2 Pods
```

---

## Comparaison : maxSkew et whenUnsatisfiable

### Impact de maxSkew

| maxSkew | Distribution | Flexibilité |
|---------|--------------|-------------|
| 1 | Très uniforme | Très strict |
| 2 | Équilibrée | Modéré |
| 3-5 | Acceptable | Flexible |
| >5 | Lâche | Très flexible |

**Exemple avec 10 Pods et 4 nœuds** :

```
maxSkew: 1
Nœud 1: 3 Pods
Nœud 2: 3 Pods
Nœud 3: 2 Pods
Nœud 4: 2 Pods
Écart = 3 - 2 = 1 ✅

maxSkew: 2
Nœud 1: 4 Pods
Nœud 2: 3 Pods
Nœud 3: 2 Pods
Nœud 4: 1 Pod
Écart = 4 - 1 = 3 ❌ Viole la contrainte avec DoNotSchedule
```

### Impact de whenUnsatisfiable

| whenUnsatisfiable | Comportement | Usage |
|-------------------|--------------|-------|
| DoNotSchedule | Pod reste Pending si violation | Applications critiques |
| ScheduleAnyway | Pod déployé même si violation | Applications flexibles |

---

## Cas d'Usage Pratiques avec MicroK8s

### Cas 1 : Application Web Haute Disponibilité

```bash
# Labelliser les nœuds par zone
kubectl label nodes node1 topology.kubernetes.io/zone=zone-a
kubectl label nodes node2 topology.kubernetes.io/zone=zone-b
kubectl label nodes node3 topology.kubernetes.io/zone=zone-c
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-ha
spec:
  replicas: 6
  selector:
    matchLabels:
      app: frontend
      tier: web
  template:
    metadata:
      labels:
        app: frontend
        tier: web
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: frontend
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
```

**Résultat** :
- 2 Pods par zone garantis
- Protection contre la panne d'une zone entière
- Service continue même si une zone est down

### Cas 2 : Workers Distribués

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workers
spec:
  replicas: 8
  selector:
    matchLabels:
      app: worker
      type: batch
  template:
    metadata:
      labels:
        app: worker
        type: batch
    spec:
      topologySpreadConstraints:
      # Distribution équilibrée sur les nœuds
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: worker
            type: batch
      containers:
      - name: worker
        image: worker:v2
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
```

**Avec 4 nœuds** :
```
Nœud 1: 2 Workers
Nœud 2: 2 Workers
Nœud 3: 2 Workers
Nœud 4: 2 Workers
```

**Avantages** :
- Charge CPU équilibrée
- Pas de nœud surchargé
- Performance optimale

### Cas 3 : Base de Données Répliquée

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-replica
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
      topologySpreadConstraints:
      # Une instance par nœud obligatoire
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: postgres
      # Distribution par zone préférée
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: postgres
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
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 20Gi
```

**Résultat** :
- Chaque instance PostgreSQL sur un nœud différent
- Idéalement dans des zones différentes
- Résilience maximale

### Cas 4 : Micro-services avec Dépendances

```yaml
---
# Service Cache
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  replicas: 3
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
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: redis
      containers:
      - name: redis
        image: redis:7-alpine
---
# Service API
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend
spec:
  replicas: 6
  selector:
    matchLabels:
      app: api
      tier: backend
  template:
    metadata:
      labels:
        app: api
        tier: backend
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: api
      affinity:
        # Préfère être avec Redis
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: redis
              topologyKey: kubernetes.io/hostname
      containers:
      - name: api
        image: api:v2
```

**Résultat** :
- Redis distribué uniformément
- API distribuée uniformément ET proche de Redis
- Équilibre entre performance et résilience

---

## Comparaison : Topology Spread vs Pod Anti-Affinity

### Distribution de 6 Pods sur 3 Nœuds

#### Avec Pod Anti-Affinity

```yaml
# Complexe et limité
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchLabels:
        app: mon-app
    topologyKey: kubernetes.io/hostname
```

**Résultat** :
```
Nœud 1: 1 Pod
Nœud 2: 1 Pod
Nœud 3: 1 Pod
3 autres Pods restent Pending ❌
```

**Problème** : Required anti-affinity = maximum 1 Pod par nœud.

#### Avec Topology Spread Constraints

```yaml
# Simple et intelligent
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: mon-app
```

**Résultat** :
```
Nœud 1: 2 Pods
Nœud 2: 2 Pods
Nœud 3: 2 Pods
Tous les Pods déployés ✅
```

**Avantage** : Distribution uniforme ET tous les Pods déployés.

### Tableau Comparatif

| Aspect | Pod Anti-Affinity | Topology Spread Constraints |
|--------|-------------------|----------------------------|
| **Syntaxe** | Verbeuse | Concise |
| **Distribution** | Tout ou rien | Uniforme avec maxSkew |
| **Flexibilité** | Limitée | Excellente |
| **Contrôle fin** | Non | Oui (maxSkew) |
| **Cas d'usage** | Séparation stricte | Distribution équilibrée |
| **Complexité** | Élevée | Moyenne |

**Recommandation** : Préférez Topology Spread Constraints pour distribuer uniformément vos Pods.

---

## MinDomains (Kubernetes 1.25+)

Une fonctionnalité plus récente permet de spécifier un nombre minimum de domaines.

```yaml
topologySpreadConstraints:
- maxSkew: 1
  minDomains: 3
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: mon-app
```

**Comportement** :
- Exige que les Pods soient répartis sur **au moins 3 zones**
- Si seulement 2 zones sont disponibles, les Pods restent Pending

**Usage** : Pour garantir une distribution géographique minimale.

---

## Combinaison avec d'Autres Mécanismes

### Topology Spread + Node Affinity

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: mon-app
  containers:
  - name: app
    image: mon-app:latest
```

**Comportement** :
- Pods **uniquement** sur nœuds SSD (Node Affinity)
- Distribution **uniforme** par zone (Topology Spread)

### Topology Spread + Taints

```yaml
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "production"
    effect: "NoSchedule"
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: mon-app
  containers:
  - name: app
    image: mon-app:latest
```

**Comportement** :
- Pods peuvent aller sur nœuds production (Toleration)
- Distribution uniforme sur ces nœuds (Topology Spread)

---

## Débogage et Diagnostics

### Pod en Pending

```bash
kubectl describe pod <pod-name>
```

**Événements typiques** :
```
Events:
  Type     Reason            Message
  ----     ------            -------
  Warning  FailedScheduling  0/3 nodes are available:
                             3 node(s) didn't match pod topology spread constraints.
```

**Causes possibles** :
1. maxSkew trop strict (essayez d'augmenter)
2. Pas assez de nœuds/zones disponibles
3. whenUnsatisfiable: DoNotSchedule trop restrictif

### Vérifier la Distribution Actuelle

```bash
# Voir sur quels nœuds les Pods tournent
kubectl get pods -l app=mon-app -o wide

# Compter les Pods par nœud
kubectl get pods -l app=mon-app -o wide --no-headers | \
  awk '{print $7}' | sort | uniq -c
```

**Exemple de sortie** :
```
3 node1
3 node2
2 node3
```

Skew = 3 - 2 = 1

### Vérifier les Labels de Topologie

```bash
# Voir les zones des nœuds
kubectl get nodes -L topology.kubernetes.io/zone

# Voir tous les labels
kubectl get nodes --show-labels
```

---

## Bonnes Pratiques

### 1. Commencez avec maxSkew: 1

Pour une distribution vraiment équilibrée :

```yaml
# ✅ Recommandé
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels:
      app: mon-app
```

### 2. Utilisez ScheduleAnyway pour la Flexibilité

À moins que la distribution parfaite soit critique :

```yaml
# ✅ Flexible
whenUnsatisfiable: ScheduleAnyway

# ⚠️ Strict - peut bloquer des déploiements
whenUnsatisfiable: DoNotSchedule
```

### 3. Labellisez Correctement vos Nœuds

```bash
# ✅ Labels de topologie clairs
kubectl label nodes node1 topology.kubernetes.io/zone=zone-a
kubectl label nodes node1 topology.kubernetes.io/rack=rack-1
kubectl label nodes node1 topology.kubernetes.io/datacenter=dc-west
```

### 4. Testez vos Contraintes

Avant de déployer en production :

```bash
# Déployer avec 1 réplica
kubectl scale deployment mon-app --replicas=1

# Augmenter progressivement
kubectl scale deployment mon-app --replicas=3
kubectl scale deployment mon-app --replicas=6

# Vérifier la distribution à chaque étape
kubectl get pods -l app=mon-app -o wide
```

### 5. Documentez vos Topologies

```yaml
metadata:
  name: app-production
  annotations:
    topology-strategy: "Uniform distribution across zones (maxSkew: 1)"
    high-availability: "true"
```

### 6. Combinez Plusieurs Niveaux de Topologie

```yaml
topologySpreadConstraints:
# Niveau 1 : Zone (strict)
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: mon-app
# Niveau 2 : Nœud (flexible)
- maxSkew: 1
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels:
      app: mon-app
```

### 7. Adaptez maxSkew au Nombre de Réplicas

| Réplicas | Domaines | maxSkew Recommandé |
|----------|----------|-------------------|
| 3 | 3 zones | 1 (distribution parfaite) |
| 5 | 3 zones | 1 (2-2-1) |
| 10 | 4 nœuds | 1 (3-3-2-2) |
| 20 | 5 zones | 2 (acceptable) |

---

## Exemple Complet : Application E-Commerce

Architecture complète utilisant Topology Spread Constraints.

### Infrastructure

```bash
# Zone A
kubectl label nodes node1 topology.kubernetes.io/zone=zone-a
kubectl label nodes node2 topology.kubernetes.io/zone=zone-a

# Zone B
kubectl label nodes node3 topology.kubernetes.io/zone=zone-b
kubectl label nodes node4 topology.kubernetes.io/zone=zone-b

# Zone C
kubectl label nodes node5 topology.kubernetes.io/zone=zone-c
kubectl label nodes node6 topology.kubernetes.io/zone=zone-c
```

### 1. Frontend (9 réplicas)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: ecommerce
    tier: frontend
spec:
  replicas: 9
  selector:
    matchLabels:
      app: ecommerce
      tier: frontend
  template:
    metadata:
      labels:
        app: ecommerce
        tier: frontend
    spec:
      topologySpreadConstraints:
      # Distribution par zone
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: ecommerce
            tier: frontend
      # Distribution par nœud
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: ecommerce
            tier: frontend
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
```

**Distribution attendue** :
```
Zone A:
  node1: 2 Pods
  node2: 1 Pod
Zone B:
  node3: 2 Pods
  node4: 1 Pod
Zone C:
  node5: 2 Pods
  node6: 1 Pod
```

### 2. Backend API (12 réplicas)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  labels:
    app: ecommerce
    tier: backend
spec:
  replicas: 12
  selector:
    matchLabels:
      app: ecommerce
      tier: backend
  template:
    metadata:
      labels:
        app: ecommerce
        tier: backend
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: ecommerce
            tier: backend
      containers:
      - name: api
        image: ecommerce-api:v2
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
```

**Distribution attendue** :
```
Zone A: 4 Pods
Zone B: 4 Pods
Zone C: 4 Pods
```

### 3. Cache Redis (3 réplicas)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  labels:
    app: ecommerce
    tier: cache
spec:
  serviceName: redis
  replicas: 3
  selector:
    matchLabels:
      app: ecommerce
      tier: cache
  template:
    metadata:
      labels:
        app: ecommerce
        tier: cache
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: ecommerce
            tier: cache
      containers:
      - name: redis
        image: redis:7-alpine
        resources:
          requests:
            cpu: "200m"
            memory: "512Mi"
```

**Distribution attendue** :
```
Zone A: 1 Pod (redis-0)
Zone B: 1 Pod (redis-1)
Zone C: 1 Pod (redis-2)
```

---

## Scénarios Avancés

### Scénario 1 : Rolling Update et Topology Spread

Lors d'un rolling update, Kubernetes respecte les Topology Spread Constraints.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: mon-app
  template:
    metadata:
      labels:
        app: mon-app
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: mon-app
      containers:
      - name: app
        image: mon-app:v2
```

**Comportement** :
- Les nouveaux Pods respectent la distribution
- Les anciens Pods sont supprimés progressivement
- La distribution reste équilibrée pendant la mise à jour

### Scénario 2 : Scaling et Topology Spread

```bash
# Scale up
kubectl scale deployment mon-app --replicas=12

# Kubernetes distribue automatiquement les nouveaux Pods
```

**Avec 4 nœuds et maxSkew: 1** :
```
Avant (6 Pods):
node1: 2, node2: 2, node3: 1, node4: 1

Après (12 Pods):
node1: 3, node2: 3, node3: 3, node4: 3
```

---

## Points Clés à Retenir

1. **Plus simple que Pod Anti-Affinity** pour la distribution uniforme
2. **maxSkew** contrôle l'écart maximum entre domaines
3. **topologyKey** définit le niveau de distribution (nœud, zone, région)
4. **whenUnsatisfiable** définit le comportement (strict ou flexible)
5. **Distribution intelligente** : Kubernetes calcule automatiquement
6. **Combinable** avec Node Affinity, Taints, etc.
7. **Préférez ScheduleAnyway** pour éviter les Pods Pending

---

## Conclusion

Les **Topology Spread Constraints** représentent une évolution majeure dans l'ordonnancement Kubernetes. Ils offrent une manière simple, élégante et puissante de distribuer uniformément vos Pods à travers votre infrastructure.

**Avantages principaux** :
- Syntaxe claire et concise
- Distribution vraiment uniforme
- Contrôle fin avec maxSkew
- Plus flexible que Pod Anti-Affinity
- Meilleure utilisation des ressources

**Quand les utiliser** :
- Haute disponibilité (distribution par zone)
- Équilibrage de charge (distribution par nœud)
- Optimisation des ressources
- Résilience géographique

Les Topology Spread Constraints sont le mécanisme moderne et recommandé pour distribuer vos Pods. Ils remplacent avantageusement Pod Anti-Affinity dans la majorité des cas d'usage.

Dans la prochaine section, nous conclurons ce chapitre sur l'ordonnancement avec des **cas d'usage pratiques** qui combinent tous les mécanismes que nous avons appris.

---

## Pour Aller Plus Loin

- **Documentation officielle** : [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- **Blog Kubernetes** : [Introducing PodTopologySpread](https://kubernetes.io/blog/2020/05/introducing-podtopologyspread/)
- **Évolutions** : Suivre les nouvelles fonctionnalités (minDomains, nodeAffinityPolicy, etc.)

---

**Prochain chapitre** : 20.7 Cas d'Usage Pratiques - Combinaison de tous les mécanismes d'ordonnancement

⏭️ [Cas d'usage pratiques](/20-ordonnancement-avance/07-cas-dusage-pratiques.md)
