🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.5 Taints et Tolerations

## Introduction

Nous avons exploré comment **attirer** les Pods vers certains nœuds avec Node Selectors, Node Affinity et Pod Affinity. Mais que faire si vous voulez **repousser** les Pods de certains nœuds ?

C'est exactement le rôle des **Taints** (souillures) et **Tolerations** (tolérances). Les Taints marquent un nœud comme "spécial" ou "restreint", et seuls les Pods ayant la Toleration appropriée peuvent y être ordonnancés.

### Analogie Simple

Imaginez un restaurant avec différentes sections :
- **Taints** = Panneaux "Réservé VIP", "Zone fumeurs", "Accès restreint"
- **Tolerations** = Badges ou autorisations spéciales qui permettent d'entrer dans ces zones

Si vous n'avez pas le bon badge, vous ne pouvez pas entrer dans la zone réservée.

---

## Concept de Base

### Taints (sur les Nœuds)

Un **Taint** est une propriété appliquée à un **nœud** qui repousse les Pods. Un taint est composé de trois parties :

```
<key>=<value>:<effect>
```

- **key** : L'identifiant du taint (ex: `gpu`, `maintenance`, `dedicated`)
- **value** : Une valeur optionnelle (ex: `true`, `team-a`, `high-priority`)
- **effect** : Ce qui se passe pour les Pods non tolérants

### Tolerations (sur les Pods)

Une **Toleration** est une propriété ajoutée à un **Pod** qui lui permet de "tolérer" un taint et donc d'être ordonnancé sur un nœud avec ce taint.

### Relation Taint-Toleration

```
Nœud avec Taint: "special=true:NoSchedule"
                    ↓
        Bloque tous les Pods
                    ↓
        SAUF ceux qui ont une Toleration correspondante
                    ↓
Pod avec Toleration: "special=true"
```

---

## Les Trois Effets des Taints

Kubernetes propose trois effets différents pour les taints, qui définissent le comportement vis-à-vis des Pods.

### 1. NoSchedule (Ne Pas Ordonnancer)

**Comportement** :
- Les nouveaux Pods **sans** toleration correspondante ne seront **jamais** ordonnancés sur ce nœud
- Les Pods **déjà présents** sur le nœud avant l'ajout du taint continuent de fonctionner normalement

**Usage** : Le plus courant, pour réserver des nœuds à des workloads spécifiques.

**Exemple** :
```bash
kubectl taint nodes node1 gpu=true:NoSchedule
```

### 2. PreferNoSchedule (Préférer Ne Pas Ordonnancer)

**Comportement** :
- Le scheduler **évite** de placer des Pods sans toleration sur ce nœud
- Mais si **aucun autre nœud** n'est disponible, le Pod sera quand même placé ici
- C'est une **préférence douce**, pas une interdiction stricte

**Usage** : Pour décourager l'utilisation d'un nœud sans la bloquer complètement.

**Exemple** :
```bash
kubectl taint nodes node1 low-priority=true:PreferNoSchedule
```

### 3. NoExecute (Ne Pas Exécuter)

**Comportement** :
- Les nouveaux Pods sans toleration ne seront **pas** ordonnancés
- Les Pods **déjà présents** sans toleration seront **expulsés** (évicted)
- C'est l'effet le plus strict

**Usage** : Pour évacuer un nœud ou le mettre en maintenance.

**Exemple** :
```bash
kubectl taint nodes node1 maintenance=true:NoExecute
```

**⚠️ Attention** : `NoExecute` peut interrompre des Pods en cours d'exécution !

---

## Ajouter un Taint à un Nœud

### Syntaxe Générale

```bash
kubectl taint nodes <nom-noeud> <key>=<value>:<effect>
```

### Exemples Pratiques

#### Exemple 1 : Réserver un Nœud avec GPU

```bash
kubectl taint nodes node-gpu gpu=true:NoSchedule
```

**Résultat** : Seuls les Pods tolérant `gpu=true` pourront être ordonnancés sur `node-gpu`.

#### Exemple 2 : Dédier un Nœud à une Équipe

```bash
kubectl taint nodes node-team-a team=team-a:NoSchedule
```

**Résultat** : Seuls les Pods de l'équipe A (avec toleration) iront sur ce nœud.

#### Exemple 3 : Décourager l'Utilisation d'un Nœud

```bash
kubectl taint nodes node-old deprecated=true:PreferNoSchedule
```

**Résultat** : Le scheduler évite ce nœud, mais l'utilise si nécessaire.

#### Exemple 4 : Mettre un Nœud en Maintenance

```bash
kubectl taint nodes node1 maintenance=scheduled:NoExecute
```

**Résultat** : Tous les Pods sans toleration seront expulsés immédiatement.

### Vérifier les Taints d'un Nœud

```bash
kubectl describe node <nom-noeud>
```

Dans la sortie, recherchez la section **Taints** :
```
Taints:    gpu=true:NoSchedule
```

Ou pour voir tous les nœuds avec leurs taints :
```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

---

## Ajouter une Toleration à un Pod

### Syntaxe de Base

Dans le manifeste YAML du Pod, ajoutez la section `tolerations` :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod
spec:
  tolerations:
  - key: "<key>"
    operator: "Equal"
    value: "<value>"
    effect: "<effect>"
  containers:
  - name: mon-conteneur
    image: nginx
```

### Opérateurs de Toleration

Il existe deux opérateurs principaux :

#### 1. Equal (Égal)

Le Pod tolère un taint si la clé, la valeur ET l'effet correspondent exactement.

```yaml
tolerations:
- key: "gpu"
  operator: "Equal"
  value: "true"
  effect: "NoSchedule"
```

**Signification** : Ce Pod tolère le taint `gpu=true:NoSchedule`.

#### 2. Exists (Existe)

Le Pod tolère un taint si la clé (et optionnellement l'effet) correspondent, **quelle que soit la valeur**.

```yaml
tolerations:
- key: "gpu"
  operator: "Exists"
  effect: "NoSchedule"
```

**Signification** : Ce Pod tolère **n'importe quel** taint avec la clé `gpu` et l'effet `NoSchedule`, peu importe la valeur.

### Toleration Universelle

Pour tolérer **tous** les taints d'un nœud :

```yaml
tolerations:
- operator: "Exists"
```

**⚠️ Attention** : À utiliser avec précaution, cela ignore tous les taints !

---

## Exemples Pratiques Complets

### Exemple 1 : Nœud GPU Réservé

**Étape 1** : Ajouter un taint au nœud GPU

```bash
kubectl taint nodes node-gpu hardware=gpu:NoSchedule
```

**Étape 2** : Déployer une application de machine learning

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tensorflow-training
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ml-training
  template:
    metadata:
      labels:
        app: ml-training
    spec:
      tolerations:
      - key: "hardware"
        operator: "Equal"
        value: "gpu"
        effect: "NoSchedule"
      containers:
      - name: tensorflow
        image: tensorflow/tensorflow:latest-gpu
        resources:
          limits:
            nvidia.com/gpu: 1
```

**Résultat** :
- Les Pods de TensorFlow peuvent être ordonnancés sur `node-gpu`
- Tous les autres Pods (sans cette toleration) seront bloqués

### Exemple 2 : Nœuds Dédiés par Environnement

**Configuration** :

```bash
# Nœuds de production
kubectl taint nodes node-prod-1 environment=production:NoSchedule
kubectl taint nodes node-prod-2 environment=production:NoSchedule

# Nœuds de staging
kubectl taint nodes node-staging-1 environment=staging:NoSchedule
```

**Déploiement Production** :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-production
  namespace: production
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
      tolerations:
      - key: "environment"
        operator: "Equal"
        value: "production"
        effect: "NoSchedule"
      containers:
      - name: app
        image: mon-app:v1.2.3
        resources:
          requests:
            cpu: "1000m"
            memory: "2Gi"
```

**Déploiement Staging** :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-staging
  namespace: staging
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mon-app
      env: staging
  template:
    metadata:
      labels:
        app: mon-app
        env: staging
    spec:
      tolerations:
      - key: "environment"
        operator: "Equal"
        value: "staging"
        effect: "NoSchedule"
      containers:
      - name: app
        image: mon-app:latest
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
```

**Résultat** :
- Les apps de production ne vont que sur les nœuds de production
- Les apps de staging ne vont que sur les nœuds de staging
- Isolation complète des environnements

### Exemple 3 : Maintenance d'un Nœud

Vous devez effectuer une maintenance sur un nœud.

**Étape 1** : Marquer le nœud en maintenance (expulse les Pods existants)

```bash
kubectl taint nodes node1 maintenance=true:NoExecute
```

**Étape 2** : Les Pods sans toleration sont expulsés automatiquement et re-créés ailleurs

**Étape 3** : Effectuer la maintenance

**Étape 4** : Retirer le taint quand c'est terminé

```bash
kubectl taint nodes node1 maintenance:NoExecute-
```

### Exemple 4 : Pods Système avec Toleration Universelle

Certains Pods système doivent pouvoir tourner partout, même sur des nœuds avec des taints.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-agent
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: monitoring
  template:
    metadata:
      labels:
        app: monitoring
    spec:
      tolerations:
      # Tolère tous les taints
      - operator: "Exists"
      containers:
      - name: agent
        image: monitoring-agent:latest
```

**Résultat** : L'agent de monitoring tourne sur **tous** les nœuds, quels que soient leurs taints.

---

## Taints par Défaut de Kubernetes

Kubernetes ajoute automatiquement certains taints dans des situations spécifiques.

### 1. node.kubernetes.io/not-ready

**Quand** : Nœud pas encore prêt (au démarrage)

**Effet** : `NoExecute`

**Signification** : Les Pods ne peuvent pas être ordonnancés tant que le nœud n'est pas prêt.

### 2. node.kubernetes.io/unreachable

**Quand** : Nœud injoignable par le control plane

**Effet** : `NoExecute`

**Signification** : Les Pods sont évacués si le nœud devient injoignable.

### 3. node.kubernetes.io/memory-pressure

**Quand** : Nœud sous pression mémoire

**Effet** : `NoSchedule`

**Signification** : Nouveaux Pods non critiques ne seront pas ordonnancés.

### 4. node.kubernetes.io/disk-pressure

**Quand** : Nœud sous pression disque

**Effet** : `NoSchedule`

**Signification** : Nouveaux Pods ne seront pas ordonnancés.

### 5. node.kubernetes.io/unschedulable

**Quand** : Nœud marqué comme non-ordonnançable (avec `kubectl cordon`)

**Effet** : `NoSchedule`

**Signification** : Aucun nouveau Pod ne sera ordonnancé.

### Voir les Taints Automatiques

```bash
kubectl describe node <nom-noeud> | grep Taints
```

---

## Toleration avec Durée (tolerationSeconds)

Pour l'effet `NoExecute`, vous pouvez spécifier combien de temps un Pod peut rester sur un nœud après qu'un taint `NoExecute` a été ajouté.

### Syntaxe

```yaml
tolerations:
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300
```

**Signification** :
- Si le nœud devient "not-ready", le Pod peut rester pendant 300 secondes (5 minutes)
- Après ce délai, le Pod sera expulsé

### Cas d'Usage

Utile pour les Pods qui peuvent tolérer des pannes temporaires sans être immédiatement expulsés.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-resiliente
spec:
  replicas: 3
  selector:
    matchLabels:
      app: resilient
  template:
    metadata:
      labels:
        app: resilient
    spec:
      tolerations:
      # Tolère un nœud qui ne répond plus pendant 10 minutes
      - key: "node.kubernetes.io/unreachable"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 600
      # Tolère un nœud not-ready pendant 5 minutes
      - key: "node.kubernetes.io/not-ready"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 300
      containers:
      - name: app
        image: mon-app:latest
```

**Résultat** :
- Si un nœud devient injoignable, les Pods attendent 10 minutes avant d'être relocalisés
- Évite des migrations inutiles pour des pannes courtes

---

## Retirer un Taint

Pour retirer un taint d'un nœud, ajoutez un `-` à la fin de la commande.

### Syntaxe

```bash
kubectl taint nodes <nom-noeud> <key>:<effect>-
```

### Exemples

```bash
# Retirer un taint spécifique
kubectl taint nodes node1 gpu=true:NoSchedule-

# Si vous ne connaissez pas la valeur, utilisez juste la clé
kubectl taint nodes node1 gpu:NoSchedule-

# Retirer un taint NoExecute
kubectl taint nodes node1 maintenance:NoExecute-
```

### Retirer Tous les Taints d'un Nœud

```bash
# Voir les taints actuels
kubectl describe node node1 | grep Taints

# Retirer chaque taint individuellement
kubectl taint nodes node1 <key1>:<effect1>-
kubectl taint nodes node1 <key2>:<effect2>-
```

---

## Combinaison avec d'Autres Mécanismes

Les taints et tolerations se combinent avec les autres mécanismes d'ordonnancement.

### Taint + Node Selector

```yaml
spec:
  nodeSelector:
    disktype: ssd
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "database"
    effect: "NoSchedule"
  containers:
  - name: postgres
    image: postgres:14
```

**Comportement** :
- Le Pod doit être sur un nœud avec `disktype=ssd` (Node Selector)
- Le Pod peut tolérer le taint `dedicated=database:NoSchedule`
- Si un nœud a un SSD ET le taint, le Pod peut y aller

### Taint + Node Affinity

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: performance
            operator: In
            values:
            - high
  tolerations:
  - key: "expensive"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  containers:
  - name: app
    image: app-intensive:latest
```

### Taint + Pod Anti-Affinity

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: mon-app
        topologyKey: kubernetes.io/hostname
  tolerations:
  - key: "special"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  containers:
  - name: app
    image: mon-app:latest
```

---

## Cas d'Usage Pratiques avec MicroK8s

### Cas 1 : Cluster Hybride (Compute vs Storage)

Vous avez des nœuds pour le calcul et d'autres pour le stockage.

**Configuration** :

```bash
# Nœuds de calcul
kubectl label nodes node1 role=compute
kubectl label nodes node2 role=compute

# Nœuds de stockage (avec taint pour les réserver)
kubectl label nodes node3 role=storage
kubectl taint nodes node3 workload=storage:NoSchedule
kubectl label nodes node4 role=storage
kubectl taint nodes node4 workload=storage:NoSchedule
```

**Application Standard (va sur les nœuds de calcul)** :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 4
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      # Pas de toleration = ne va pas sur les nœuds storage
      containers:
      - name: nginx
        image: nginx:latest
```

**Base de Données (va sur les nœuds de stockage)** :

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: db
  replicas: 2
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      tolerations:
      - key: "workload"
        operator: "Equal"
        value: "storage"
        effect: "NoSchedule"
      nodeSelector:
        role: storage
      containers:
      - name: postgres
        image: postgres:14
```

**Résultat** :
- Applications normales sur nœuds compute (node1, node2)
- Bases de données sur nœuds storage (node3, node4)
- Isolation claire des workloads

### Cas 2 : Nœuds à Coût Élevé (Spot Instances)

Vous avez des nœuds spot (moins chers mais peuvent être interrompus).

**Configuration** :

```bash
kubectl label nodes node-spot-1 instance-type=spot
kubectl taint nodes node-spot-1 spot=true:PreferNoSchedule
kubectl label nodes node-spot-2 instance-type=spot
kubectl taint nodes node-spot-2 spot=true:PreferNoSchedule
```

**Application Batch (tolère les spots)** :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processing
spec:
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      tolerations:
      - key: "spot"
        operator: "Equal"
        value: "true"
        effect: "PreferNoSchedule"
      restartPolicy: OnFailure
      containers:
      - name: processor
        image: data-processor:latest
```

**Application Critique (évite les spots)** :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      # Pas de toleration = évite les nœuds spot (grâce à PreferNoSchedule)
      containers:
      - name: api
        image: api:v1
```

**Résultat** :
- Jobs batch utilisent de préférence les instances spot (moins chers)
- Applications critiques évitent les spots mais peuvent les utiliser en dernier recours

### Cas 3 : Rotation et Mise à Jour des Nœuds

Vous voulez vider un nœud pour le mettre à jour.

**Étape 1** : Cordon (empêche les nouveaux Pods)

```bash
kubectl cordon node1
```

Cela ajoute automatiquement le taint : `node.kubernetes.io/unschedulable:NoSchedule`

**Étape 2** : Drain (évacue les Pods existants)

```bash
kubectl drain node1 --ignore-daemonsets --delete-emptydir-data
```

Cela ajoute un taint `NoExecute` qui expulse tous les Pods.

**Étape 3** : Effectuer la mise à jour du nœud

**Étape 4** : Uncordon (réactive le nœud)

```bash
kubectl uncordon node1
```

Retire tous les taints ajoutés.

---

## Débogage et Diagnostics

### Pod en Pending à Cause d'un Taint

```bash
kubectl describe pod <pod-name>
```

Événements typiques :
```
Events:
  Type     Reason            Message
  ----     ------            -------
  Warning  FailedScheduling  0/3 nodes are available:
                             1 node(s) had taint {gpu: true}, that the pod didn't tolerate.
                             2 node(s) had taint {environment: production}, that the pod didn't tolerate.
```

**Solution** : Ajouter la toleration appropriée au Pod.

### Lister les Nœuds avec leurs Taints

```bash
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, taints: .spec.taints}'
```

Ou plus simple :

```bash
kubectl describe nodes | grep -A 5 "Taints:"
```

### Voir Quels Pods Tournent sur un Nœud avec Taints

```bash
kubectl get pods --all-namespaces -o wide --field-selector spec.nodeName=node1
```

---

## Bonnes Pratiques

### 1. Utilisez des Clés Descriptives

```yaml
# ✅ Bon : Clé claire
tolerations:
- key: "dedicated-gpu"
  operator: "Equal"
  value: "ml-training"
  effect: "NoSchedule"
```

```yaml
# ❌ Mauvais : Clé vague
tolerations:
- key: "special"
  operator: "Equal"
  value: "yes"
  effect: "NoSchedule"
```

### 2. Documentez vos Taints

Maintenez une documentation des taints utilisés dans votre cluster :

```markdown
# Taints Standard du Cluster

## Nœuds GPU
- Key: `hardware`
- Value: `gpu`
- Effect: `NoSchedule`
- Usage: Réservé pour workloads ML/AI

## Nœuds Production
- Key: `environment`
- Value: `production`
- Effect: `NoSchedule`
- Usage: Applications de production uniquement

## Maintenance
- Key: `maintenance`
- Value: `true`
- Effect: `NoExecute`
- Usage: Nœud en maintenance, évacuation des Pods
```

### 3. NoExecute avec Précaution

L'effet `NoExecute` peut interrompre des services. Utilisez-le avec précaution :

```bash
# ✅ Bon : Drain pour évacuer progressivement
kubectl drain node1 --ignore-daemonsets

# ⚠️ Attention : NoExecute évacue brutalement
kubectl taint nodes node1 maintenance=true:NoExecute
```

### 4. Combinez avec Labels

Les taints vont bien avec les labels pour une organisation claire :

```bash
# Label pour identifier le nœud
kubectl label nodes node1 hardware=gpu

# Taint pour restreindre l'accès
kubectl taint nodes node1 hardware=gpu:NoSchedule
```

### 5. Préférez PreferNoSchedule pour Commencer

Si vous n'êtes pas sûr, utilisez `PreferNoSchedule` plutôt que `NoSchedule` :

```bash
# ✅ Plus flexible
kubectl taint nodes node1 deprecated=true:PreferNoSchedule

# ⚠️ Plus strict
kubectl taint nodes node1 deprecated=true:NoSchedule
```

### 6. Taints Temporaires

Pour des taints temporaires, utilisez des noms explicites :

```bash
kubectl taint nodes node1 temporary-maintenance=$(date +%Y%m%d):NoExecute
```

Facilite l'identification des taints anciens à nettoyer.

### 7. DaemonSets et Tolerations

Les DaemonSets ont souvent besoin de tolerations pour tourner partout :

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
spec:
  selector:
    matchLabels:
      app: logs
  template:
    metadata:
      labels:
        app: logs
    spec:
      tolerations:
      # Tolère tous les taints pour tourner sur tous les nœuds
      - operator: "Exists"
      containers:
      - name: fluentd
        image: fluentd:latest
```

---

## Comparaison : Taints vs Node Affinity

| Aspect | Taints + Tolerations | Node Affinity |
|--------|---------------------|---------------|
| **Direction** | Nœud → Pod (repousse) | Pod → Nœud (attire) |
| **Philosophie** | "Qui ne peut PAS venir ici ?" | "Où ce Pod DOIT-il aller ?" |
| **Par défaut** | Bloque tout le monde | Autorise tout le monde |
| **Flexibilité** | 3 effets (NoSchedule, Prefer, NoExecute) | Required vs Preferred |
| **Éviction** | Oui (NoExecute) | Non |
| **Cas d'usage** | Réserver, isoler, maintenir | Optimiser, préférer, exiger |

### Quand Utiliser Quoi ?

**Utilisez Taints** quand :
- Vous voulez **réserver** un nœud pour un usage spécial
- Vous voulez **protéger** un nœud contre les Pods normaux
- Vous voulez **évacuer** un nœud

**Utilisez Node Affinity** quand :
- Vous voulez **diriger** des Pods vers certains nœuds
- Vous voulez **exprimer des préférences** de placement
- Vous avez des **critères complexes** de sélection

**Souvent, utilisez les deux ensemble** pour un contrôle complet !

---

## Exemple Complet : Architecture Enterprise

Voici une architecture complète utilisant taints, tolerations et autres mécanismes.

### Architecture

```
Cluster de 6 nœuds :
- 2 nœuds GPU (taints: gpu=true)
- 2 nœuds Production (taints: env=prod)
- 2 nœuds Développement (pas de taints)
```

### Configuration des Nœuds

```bash
# Nœuds GPU
kubectl label nodes node-gpu-1 hardware=gpu zone=a
kubectl taint nodes node-gpu-1 hardware=gpu:NoSchedule
kubectl label nodes node-gpu-2 hardware=gpu zone=b
kubectl taint nodes node-gpu-2 hardware=gpu:NoSchedule

# Nœuds Production
kubectl label nodes node-prod-1 environment=production zone=a disktype=ssd
kubectl taint nodes node-prod-1 environment=production:NoSchedule
kubectl label nodes node-prod-2 environment=production zone=b disktype=ssd
kubectl taint nodes node-prod-2 environment=production:NoSchedule

# Nœuds Dev (pas de taints)
kubectl label nodes node-dev-1 environment=development zone=a
kubectl label nodes node-dev-2 environment=development zone=b
```

### 1. Application de Machine Learning

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-training
  namespace: ml
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ml-training
  template:
    metadata:
      labels:
        app: ml-training
    spec:
      # Toleration pour accéder aux nœuds GPU
      tolerations:
      - key: "hardware"
        operator: "Equal"
        value: "gpu"
        effect: "NoSchedule"
      # Node Affinity pour exiger un GPU
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: hardware
                operator: In
                values:
                - gpu
        # Pod Anti-Affinity pour distribuer
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: ml-training
              topologyKey: kubernetes.io/hostname
      containers:
      - name: tensorflow
        image: tensorflow/tensorflow:latest-gpu
        resources:
          limits:
            nvidia.com/gpu: 1
```

### 2. API de Production

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-production
  namespace: production
spec:
  replicas: 4
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
      # Toleration pour accéder aux nœuds production
      tolerations:
      - key: "environment"
        operator: "Equal"
        value: "production"
        effect: "NoSchedule"
      # Node Affinity pour exiger production + SSD
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
        # Pod Anti-Affinity pour haute disponibilité
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: api
            topologyKey: zone
      containers:
      - name: api
        image: api:v2.1.0
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
```

### 3. Application de Développement

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-dev
  namespace: development
spec:
  replicas: 2
  selector:
    matchLabels:
      app: test-app
      env: dev
  template:
    metadata:
      labels:
        app: test-app
        env: dev
    spec:
      # Pas de tolerations = va uniquement sur les nœuds dev (sans taints)
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: environment
                operator: In
                values:
                - development
      containers:
      - name: app
        image: test-app:latest
        resources:
          requests:
            cpu: "100m"
            memory: "256Mi"
```

### 4. Monitoring (DaemonSet sur tous les nœuds)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      # Tolère tous les taints pour tourner partout
      tolerations:
      - operator: "Exists"
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
```

**Résultat de cette architecture** :
- ML training : Uniquement sur les nœuds GPU
- API Production : Uniquement sur les nœuds production avec SSD, distribuée par zone
- App Dev : Uniquement sur les nœuds dev
- Monitoring : Sur tous les nœuds
- Isolation complète et optimisation des ressources

---

## Points Clés à Retenir

1. **Taints** = Marquent les nœuds comme "spéciaux" et repoussent les Pods
2. **Tolerations** = Permettent aux Pods de "tolérer" les taints
3. **Trois effets** :
   - `NoSchedule` : Bloque les nouveaux Pods
   - `PreferNoSchedule` : Décourage mais n'empêche pas
   - `NoExecute` : Évacue les Pods existants
4. **Direction** : Nœud → Pod (contrairement à l'affinity qui va Pod → Nœud)
5. **Complémentaire** : Combinez avec Node/Pod Affinity pour un contrôle total
6. **Maintenance** : Très utile pour cordon/drain des nœuds
7. **DaemonSets** : Nécessitent souvent `tolerations: operator: Exists`

---

## Conclusion

Les Taints et Tolerations sont un mécanisme puissant et complémentaire aux autres outils d'ordonnancement. Ils fonctionnent dans la direction inverse des affinities : au lieu de dire où les Pods **doivent** aller, vous dites où ils **ne peuvent pas** aller (sauf exception).

**Cas d'usage principaux** :
- **Réserver** des nœuds pour des workloads spéciaux (GPU, production, etc.)
- **Isoler** les environnements
- **Maintenir** les nœuds en les vidant de leurs Pods
- **Protéger** les nœuds critiques

En combinant intelligemment Taints, Tolerations, Node Affinity et Pod Affinity, vous disposez d'un arsenal complet pour orchestrer précisément le placement de vos Pods dans votre cluster Kubernetes.

Dans la prochaine section, nous découvrirons les **Topology Spread Constraints**, un mécanisme plus récent et souvent plus simple pour distribuer uniformément vos Pods à travers votre infrastructure.

---

## Pour Aller Plus Loin

- **Documentation officielle** : [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- **Node Maintenance** : [Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
- **Eviction** : [Pod Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-eviction/)

---

**Prochain chapitre** : 20.6 Topology Spread Constraints - Distribuer uniformément vos Pods avec simplicité

⏭️ [Topology Spread Constraints](/20-ordonnancement-avance/06-topology-spread-constraints.md)
