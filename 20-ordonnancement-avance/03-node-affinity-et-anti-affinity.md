🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.3 Node Affinity et Anti-Affinity

## Introduction

Dans la section précédente, vous avez découvert les **Node Selectors**, un moyen simple de diriger vos Pods vers des nœuds spécifiques. Les **Node Affinity** (affinité de nœud) sont la version évoluée et beaucoup plus puissante des Node Selectors.

Si les Node Selectors sont comme dire "Je veux absolument cette table", les Node Affinity permettent de dire "Je préférerais cette table, mais si elle n'est pas disponible, cette autre table fera l'affaire" ou encore "Je veux n'importe quelle table sauf celle-ci".

Dans cette section, nous allons explorer ce mécanisme sophistiqué qui vous donne un contrôle fin sur l'ordonnancement de vos Pods.

---

## Pourquoi Node Affinity au Lieu de Node Selectors ?

Les Node Selectors ont des limitations importantes :

### Limitations des Node Selectors

1. **Contrainte rigide uniquement** : Soit le Pod est placé, soit il reste Pending
2. **Logique simple** : Uniquement des conditions "ET" (AND)
3. **Pas de préférences** : Impossible d'exprimer "je préfère X mais j'accepte Y"
4. **Pas de négation** : Impossible de dire "PAS ce nœud"
5. **Pas d'opérateurs** : Impossible de dire "version > 1.20"

### Avantages des Node Affinity

Les Node Affinity résolvent toutes ces limitations :

✅ **Contraintes souples et rigides** : Préférences (soft) vs obligations (hard)
✅ **Logique complexe** : Opérateurs OR, NOT, comparaisons
✅ **Préférences graduées** : Poids pour prioriser certains nœuds
✅ **Expressions riches** : In, NotIn, Exists, DoesNotExist, Gt, Lt
✅ **Flexibilité** : Comportement dégradé si les nœuds préférés sont indisponibles

---

## Les Deux Types d'Affinity

Kubernetes propose deux types principaux de Node Affinity :

### 1. Required During Scheduling (Requis)

**Nom complet** : `requiredDuringSchedulingIgnoredDuringExecution`

**Comportement** :
- Le Pod **doit** être placé sur un nœud qui satisfait la règle
- Si aucun nœud ne correspond, le Pod reste **Pending**
- C'est l'équivalent d'un Node Selector, mais plus puissant

**Analogie** : "Je dois absolument avoir un siège près de la fenêtre, sinon je ne monte pas dans l'avion"

### 2. Preferred During Scheduling (Préféré)

**Nom complet** : `preferredDuringSchedulingIgnoredDuringExecution`

**Comportement** :
- Le scheduler **préfère** placer le Pod sur un nœud qui satisfait la règle
- Si aucun nœud ne correspond, le Pod sera quand même placé ailleurs
- Vous pouvez définir un **poids** (weight) pour prioriser les préférences

**Analogie** : "J'aimerais bien un siège près de la fenêtre, mais n'importe quel siège fera l'affaire"

### Note sur "IgnoredDuringExecution"

La partie `IgnoredDuringExecution` signifie que si les labels d'un nœud changent après que le Pod y soit déjà en cours d'exécution, le Pod ne sera **pas** expulsé. Les règles d'affinity ne sont évaluées qu'au moment de l'ordonnancement.

---

## Syntaxe de Base

La syntaxe des Node Affinity est plus verbeuse que celle des Node Selectors, mais elle offre beaucoup plus de possibilités.

### Structure Générale

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: <label-key>
            operator: <operator>
            values:
            - <value>
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: <1-100>
        preference:
          matchExpressions:
          - key: <label-key>
            operator: <operator>
            values:
            - <value>
  containers:
  - name: mon-conteneur
    image: nginx
```

---

## Les Opérateurs Disponibles

Les Node Affinity supportent plusieurs opérateurs pour créer des règles sophistiquées :

### 1. **In** - Dans la liste

Le label doit avoir l'une des valeurs spécifiées.

```yaml
- key: environment
  operator: In
  values:
  - production
  - staging
```

**Signification** : Le nœud doit avoir le label `environment` avec la valeur `production` **ou** `staging`.

### 2. **NotIn** - Pas dans la liste

Le label ne doit **pas** avoir l'une des valeurs spécifiées.

```yaml
- key: environment
  operator: NotIn
  values:
  - development
  - test
```

**Signification** : Le nœud ne doit **pas** avoir le label `environment` avec la valeur `development` ou `test`.

### 3. **Exists** - Le label existe

Le nœud doit avoir ce label, peu importe sa valeur.

```yaml
- key: ssd
  operator: Exists
```

**Signification** : Le nœud doit avoir un label `ssd` (avec n'importe quelle valeur, même vide).

**Note** : Pas besoin de spécifier `values` avec cet opérateur.

### 4. **DoesNotExist** - Le label n'existe pas

Le nœud ne doit **pas** avoir ce label.

```yaml
- key: gpu
  operator: DoesNotExist
```

**Signification** : Le nœud ne doit **pas** avoir de label `gpu`.

### 5. **Gt** - Plus grand que (Greater Than)

La valeur du label doit être numériquement supérieure.

```yaml
- key: cpu-cores
  operator: Gt
  values:
  - "8"
```

**Signification** : Le nœud doit avoir un label `cpu-cores` avec une valeur > 8.

### 6. **Lt** - Moins que (Less Than)

La valeur du label doit être numériquement inférieure.

```yaml
- key: load-average
  operator: Lt
  values:
  - "2.0"
```

**Signification** : Le nœud doit avoir un label `load-average` avec une valeur < 2.0.

---

## Required Affinity - Exemples Pratiques

### Exemple 1 : Équivalent à un Node Selector Simple

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-production
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: environment
            operator: In
            values:
            - production
  containers:
  - name: nginx
    image: nginx
```

**Ce que fait ce manifeste** :
- Le Pod **doit** être placé sur un nœud ayant le label `environment=production`
- C'est strictement équivalent à : `nodeSelector: { environment: production }`

### Exemple 2 : Logique OR (OU)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-prod-ou-staging
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: environment
            operator: In
            values:
            - production
            - staging
  containers:
  - name: nginx
    image: nginx
```

**Ce que fait ce manifeste** :
- Le Pod peut être placé sur un nœud avec `environment=production` **OU** `environment=staging`
- Impossible à faire avec un simple Node Selector !

### Exemple 3 : Exclusion de Nœuds

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-pas-en-dev
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: environment
            operator: NotIn
            values:
            - development
  containers:
  - name: nginx
    image: nginx
```

**Ce que fait ce manifeste** :
- Le Pod peut être placé sur n'importe quel nœud **sauf** ceux ayant `environment=development`
- Permet d'éviter explicitement certains nœuds

### Exemple 4 : Vérifier l'Existence d'un Label

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-gpu
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: hardware.gpu
            operator: Exists
  containers:
  - name: tensorflow
    image: tensorflow/tensorflow:latest-gpu
```

**Ce que fait ce manifeste** :
- Le Pod doit être placé sur un nœud qui a un label `hardware.gpu`, quelle que soit sa valeur
- Utile quand la présence du label suffit, peu importe sa valeur exacte

### Exemple 5 : Conditions Multiples (AND)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-conditions-multiples
spec:
  affinity:
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
          - key: zone
            operator: In
            values:
            - europe
  containers:
  - name: nginx
    image: nginx
```

**Ce que fait ce manifeste** :
- Le nœud doit avoir `environment=production` **ET**
- Le nœud doit avoir `disktype=ssd` **ET**
- Le nœud doit avoir `zone=europe`
- Toutes les conditions doivent être satisfaites simultanément

### Exemple 6 : Plusieurs Groupes de Conditions (OR de AND)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-flexible
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: environment
            operator: In
            values:
            - production
          - key: zone
            operator: In
            values:
            - europe
        - matchExpressions:
          - key: environment
            operator: In
            values:
            - staging
          - key: zone
            operator: In
            values:
            - us
  containers:
  - name: nginx
    image: nginx
```

**Ce que fait ce manifeste** :
- **Option 1** : (`environment=production` ET `zone=europe`) **OU**
- **Option 2** : (`environment=staging` ET `zone=us`)
- Le Pod peut être placé si l'une des deux options est satisfaite

---

## Preferred Affinity - Exemples Pratiques

Les Preferred Affinity permettent d'exprimer des **préférences** plutôt que des obligations.

### Exemple 1 : Préférence Simple

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-preference-ssd
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: nginx
    image: nginx
```

**Ce que fait ce manifeste** :
- Le scheduler **préfère** placer le Pod sur un nœud avec `disktype=ssd`
- Si aucun nœud SSD n'est disponible, le Pod sera placé sur n'importe quel autre nœud
- Le `weight` (poids) de 100 indique la force de cette préférence

### Exemple 2 : Comprendre le Weight (Poids)

Le poids (weight) va de 1 à 100 et permet de prioriser les préférences.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-preferences-multiples
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
      - weight: 20
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - europe
  containers:
  - name: nginx
    image: nginx
```

**Comment le scheduler calcule** :
- Chaque nœud reçoit un score basé sur les préférences satisfaites
- Nœud avec `disktype=ssd` : +80 points
- Nœud avec `zone=europe` : +20 points
- Nœud avec les deux : +100 points
- Le scheduler choisit le nœud avec le score le plus élevé

**Signification pratique** :
- La présence d'un SSD est **4 fois plus importante** que la zone géographique (80 vs 20)
- Si aucun nœud ne satisfait ces préférences, le Pod sera quand même placé

### Exemple 3 : Combinaison Required + Preferred

Vous pouvez combiner des contraintes obligatoires et des préférences.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-smart
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: environment
            operator: In
            values:
            - production
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
      - weight: 50
        preference:
          matchExpressions:
          - key: cpu-cores
            operator: Gt
            values:
            - "8"
  containers:
  - name: nginx
    image: nginx
```

**Ce que fait ce manifeste** :
- **Obligatoire** : Le Pod **doit** être sur un nœud `environment=production`
- **Préféré (poids 100)** : Idéalement avec `disktype=ssd`
- **Préféré (poids 50)** : Idéalement avec plus de 8 cœurs CPU
- Si aucun nœud de production n'a de SSD ni beaucoup de CPU, le Pod ira quand même sur un nœud de production

---

## Cas d'Usage Pratiques avec MicroK8s

### Cas 1 : Optimisation des Performances

Vous avez un cluster avec des nœuds de différentes puissances.

**Configuration des nœuds** :
```bash
kubectl label nodes node1 performance=high cpu-cores=16
kubectl label nodes node2 performance=medium cpu-cores=8
kubectl label nodes node3 performance=low cpu-cores=4
```

**Déploiement d'une application exigeante** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-exigeante
spec:
  replicas: 3
  selector:
    matchLabels:
      app: exigeante
  template:
    metadata:
      labels:
        app: exigeante
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: performance
                operator: In
                values:
                - high
          - weight: 50
            preference:
              matchExpressions:
              - key: cpu-cores
                operator: Gt
                values:
                - "8"
      containers:
      - name: app
        image: mon-app-intensive
        resources:
          requests:
            cpu: "2000m"
            memory: "4Gi"
```

**Résultat** :
- Le scheduler privilégie fortement les nœuds haute performance
- Si tous les nœuds haute performance sont pleins, il utilise les nœuds medium
- Les Pods seront quand même déployés même si aucun nœud préféré n'est disponible

### Cas 2 : Distribution Géographique avec Préférence

```bash
# Labellisation
kubectl label nodes node1 zone=europe region=west
kubectl label nodes node2 zone=europe region=east
kubectl label nodes node3 zone=us region=east
```

**Déploiement** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-globale
spec:
  replicas: 5
  selector:
    matchLabels:
      app: globale
  template:
    metadata:
      labels:
        app: globale
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: zone
                operator: In
                values:
                - europe
                - us
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            preference:
              matchExpressions:
              - key: zone
                operator: In
                values:
                - europe
          - weight: 40
            preference:
              matchExpressions:
              - key: region
                operator: In
                values:
                - west
      containers:
      - name: app
        image: mon-app:latest
```

**Résultat** :
- Tous les Pods doivent être en Europe ou aux US (obligatoire)
- Préférence forte pour l'Europe (weight 80)
- Préférence secondaire pour la région West (weight 40)
- Ordre de préférence : europe-west > europe-east > us-east

### Cas 3 : Éviter les Nœuds Surchargés

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-legere
spec:
  replicas: 10
  selector:
    matchLabels:
      app: legere
  template:
    metadata:
      labels:
        app: legere
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: workload
                operator: NotIn
                values:
                - heavy
                - database
      containers:
      - name: app
        image: app-legere:latest
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
```

**Résultat** :
- L'application légère **ne sera jamais** placée sur des nœuds marqués `workload=heavy` ou `workload=database`
- Permet de réserver certains nœuds pour des charges spécifiques

### Cas 4 : Base de Données avec Exigences Strictes

```yaml
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
              - key: memory
                operator: In
                values:
                - high
                - very-high
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: disktype
                operator: In
                values:
                - nvme
          - weight: 80
            preference:
              matchExpressions:
              - key: workload
                operator: In
                values:
                - database
      containers:
      - name: postgres
        image: postgres:14
        resources:
          requests:
            cpu: "2000m"
            memory: "8Gi"
```

**Résultat** :
- **Obligatoire** : Disque SSD ou NVMe + Haute mémoire
- **Préféré** : NVMe plutôt que SSD (weight 100)
- **Préféré** : Nœud dédié aux bases de données (weight 80)

---

## Node Anti-Affinity ?

Vous pourriez vous demander s'il existe une "Node Anti-Affinity" explicite. La réponse est **non**, mais vous pouvez obtenir le même résultat avec les opérateurs existants.

### Simuler une Anti-Affinity

Pour dire "N'importe quel nœud SAUF ceux-ci" :

```yaml
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
    - matchExpressions:
      - key: environment
        operator: NotIn
        values:
        - test
        - development
```

Pour dire "Nœuds qui n'ont PAS ce label" :

```yaml
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
    - matchExpressions:
      - key: restricted
        operator: DoesNotExist
```

---

## Débogage et Diagnostics

### Voir Pourquoi un Pod est Pending

```bash
kubectl describe pod <pod-name>
```

Recherchez dans les Events :
```
Events:
  Type     Reason            Message
  ----     ------            -------
  Warning  FailedScheduling  0/3 nodes are available:
                             3 node(s) didn't match Pod's node affinity/selector.
```

### Vérifier les Labels des Nœuds

```bash
kubectl get nodes --show-labels
```

### Tester une Affinity Complexe

Créez d'abord un Pod de test :

```bash
kubectl run test-affinity --image=nginx --dry-run=client -o yaml > test-affinity.yaml
```

Ajoutez vos règles d'affinity dans le fichier, puis :

```bash
kubectl apply -f test-affinity.yaml
kubectl get pod test-affinity -o wide
```

### Voir sur Quel Nœud un Pod a été Placé

```bash
kubectl get pods -o wide
```

---

## Comparaison : Node Selector vs Node Affinity

| Aspect | Node Selector | Node Affinity |
|--------|---------------|---------------|
| **Simplicité** | ⭐⭐⭐⭐⭐ Très simple | ⭐⭐ Plus complexe |
| **Flexibilité** | ⭐⭐ Limitée | ⭐⭐⭐⭐⭐ Très flexible |
| **Logique OR** | ❌ Non | ✅ Oui |
| **Négation** | ❌ Non | ✅ Oui (NotIn, DoesNotExist) |
| **Préférences** | ❌ Non | ✅ Oui (preferred) |
| **Poids** | ❌ Non | ✅ Oui (weight) |
| **Opérateurs** | ❌ Égalité uniquement | ✅ In, NotIn, Exists, Gt, Lt, etc. |
| **Contrainte douce** | ❌ Non | ✅ Oui (preferred) |

**Recommandation** :
- Utilisez **Node Selector** pour des besoins simples (équivalent à un seul label)
- Utilisez **Node Affinity** pour des besoins complexes ou des préférences

---

## Bonnes Pratiques

### 1. Commencez Simple

Ne créez pas des règles d'affinity complexes dès le départ. Commencez simple et ajoutez de la complexité uniquement si nécessaire.

### 2. Préférez les Preferred Quand Possible

Les `preferredDuringScheduling` sont plus flexibles et résilientes. Le Pod sera toujours placé quelque part.

```yaml
# ✅ Bon : Préférence avec fallback
preferredDuringSchedulingIgnoredDuringExecution:
- weight: 100
  preference:
    matchExpressions:
    - key: disktype
      operator: In
      values:
      - ssd
```

```yaml
# ⚠️ À éviter sauf si vraiment nécessaire : Contrainte rigide
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
  - matchExpressions:
    - key: disktype
      operator: In
      values:
      - ssd
```

### 3. Utilisez des Poids Significatifs

Espacez les poids pour exprimer clairement les priorités :

```yaml
# ✅ Bon : Différences claires
- weight: 100  # Très important
  preference: ...
- weight: 50   # Moyennement important
  preference: ...
- weight: 10   # Peu important
  preference: ...
```

```yaml
# ❌ Mauvais : Pas de distinction claire
- weight: 90
  preference: ...
- weight: 88
  preference: ...
- weight: 85
  preference: ...
```

### 4. Documentez vos Règles

Ajoutez des commentaires pour expliquer les règles complexes :

```yaml
affinity:
  nodeAffinity:
    # Obligation : Seulement en production avec SSD
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
    # Préférence : Idéalement en Europe avec > 16 cores
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      preference:
        matchExpressions:
        - key: zone
          operator: In
          values:
          - europe
    - weight: 50
      preference:
        matchExpressions:
        - key: cpu-cores
          operator: Gt
          values:
          - "16"
```

### 5. Testez Vos Règles

Avant de déployer en production :

1. Créez un Pod de test avec vos règles d'affinity
2. Vérifiez qu'il est placé sur le bon type de nœud
3. Testez des scénarios de défaillance (nœud down, etc.)

### 6. Surveillez les Pods Pending

```bash
# Script de surveillance simple
kubectl get pods --all-namespaces | grep Pending
```

Si vous avez beaucoup de Pods Pending, revoyez vos règles d'affinity - elles sont peut-être trop restrictives.

### 7. Combinez avec Resource Requests

Les affinity ne remplacent pas les resource requests :

```yaml
spec:
  affinity:
    nodeAffinity:
      # ... vos règles ...
  containers:
  - name: app
    image: mon-app
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
      limits:
        cpu: "1000m"
        memory: "2Gi"
```

---

## Exemple Complet : Application Multi-Tiers Avancée

Voici un exemple complet combinant plusieurs concepts pour une architecture à trois niveaux.

### Labellisation des Nœuds

```bash
# Frontend nodes
kubectl label nodes node1 tier=frontend disktype=ssd zone=europe
kubectl label nodes node2 tier=frontend disktype=ssd zone=us

# Backend nodes
kubectl label nodes node3 tier=backend disktype=ssd zone=europe cpu-cores=16
kubectl label nodes node4 tier=backend disktype=ssd zone=us cpu-cores=8

# Database node
kubectl label nodes node5 tier=database disktype=nvme zone=europe memory=very-high cpu-cores=32
```

### Frontend Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 4
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: tier
                operator: In
                values:
                - frontend
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            preference:
              matchExpressions:
              - key: zone
                operator: In
                values:
                - europe
          - weight: 20
            preference:
              matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
```

### Backend Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: tier
                operator: In
                values:
                - backend
              - key: disktype
                operator: In
                values:
                - ssd
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: cpu-cores
                operator: Gt
                values:
                - "12"
          - weight: 60
            preference:
              matchExpressions:
              - key: zone
                operator: In
                values:
                - europe
      containers:
      - name: api
        image: mon-api:v2
        resources:
          requests:
            cpu: "1000m"
            memory: "2Gi"
```

### Database StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: db
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: tier
                operator: In
                values:
                - database
              - key: disktype
                operator: In
                values:
                - nvme
                - ssd
              - key: memory
                operator: In
                values:
                - very-high
                - high
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: disktype
                operator: In
                values:
                - nvme
          - weight: 80
            preference:
              matchExpressions:
              - key: cpu-cores
                operator: Gt
                values:
                - "24"
      containers:
      - name: postgres
        image: postgres:14
        resources:
          requests:
            cpu: "4000m"
            memory: "16Gi"
```

---

## Points Clés à Retenir

1. **Node Affinity = Node Selector++** : Plus puissant mais plus complexe

2. **Deux types** :
   - `required` : Obligation (comme Node Selector mais plus flexible)
   - `preferred` : Préférence (avec fallback)

3. **Opérateurs riches** : In, NotIn, Exists, DoesNotExist, Gt, Lt

4. **Poids (weight)** : Permet de prioriser les préférences (1-100)

5. **Logique complexe** : Supporte OR, AND, NOT

6. **Préférez preferred** : Plus flexible et résilient

7. **Testez toujours** : Vérifiez que vos règles fonctionnent comme prévu

---

## Conclusion

Les Node Affinity représentent un outil puissant pour contrôler finement le placement de vos Pods dans un cluster Kubernetes. Bien qu'elles soient plus complexes que les Node Selectors, elles offrent une flexibilité inégalée pour gérer des scénarios d'ordonnancement sophistiqués.

En maîtrisant les Node Affinity, vous pouvez :
- Optimiser l'utilisation des ressources matérielles
- Améliorer les performances en plaçant les Pods stratégiquement
- Créer des architectures résilientes avec des préférences dégradées
- Isoler les workloads selon vos besoins

Dans la prochaine section, nous découvrirons les **Pod Affinity et Anti-Affinity**, qui permettent de contrôler le placement des Pods **les uns par rapport aux autres**, ouvrant encore plus de possibilités pour l'orchestration de vos applications.

---

## Pour Aller Plus Loin

- **Documentation officielle** : [Node Affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)
- **Scheduling Framework** : Comprendre comment le scheduler calcule les scores
- **Pod Affinity** : Contrôler le placement relatif des Pods (prochaine section)

---

**Prochain chapitre** : 20.4 Pod Affinity et Anti-Affinity - Orchestrer le placement relatif de vos Pods

⏭️ [Pod Affinity et Anti-Affinity](/20-ordonnancement-avance/04-pod-affinity-et-anti-affinity.md)
