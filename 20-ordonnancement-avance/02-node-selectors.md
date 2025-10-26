🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.2 Node Selectors

## Introduction

Maintenant que vous comprenez comment fonctionne le scheduler Kubernetes, il est temps de découvrir comment vous pouvez **influencer** ses décisions. Les **Node Selectors** sont le mécanisme le plus simple et le plus direct pour indiquer au scheduler sur quel type de nœud vous souhaitez que vos Pods soient exécutés.

Dans cette section, nous allons apprendre à utiliser les Node Selectors pour contrôler le placement de vos Pods de manière explicite et prévisible.

---

## Qu'est-ce qu'un Node Selector ?

Un **Node Selector** est une contrainte simple que vous pouvez ajouter à vos Pods pour spécifier qu'ils doivent être ordonnancés **uniquement** sur des nœuds possédant certains labels (étiquettes).

### Analogie Simple

Imaginez que vous organisez une fête et que vous devez assigner des invités à des tables :
- Certaines tables sont étiquetées "VIP"
- D'autres sont étiquetées "Famille"
- Certaines sont étiquetées "Amis"

Un Node Selector, c'est comme dire : "Cette personne doit absolument être assise à une table étiquetée 'VIP'". Si aucune table VIP n'est disponible, la personne ne pourra pas être placée.

---

## Pourquoi Utiliser des Node Selectors ?

Les Node Selectors sont utiles dans plusieurs situations :

### 1. **Nœuds avec du Matériel Spécifique**
Vous avez des nœuds équipés de GPU et vous voulez que seules les applications de machine learning tournent dessus.

### 2. **Séparation par Environnement**
Vous souhaitez que les applications de production tournent sur certains nœuds et les applications de développement sur d'autres.

### 3. **Optimisation des Coûts**
Dans un cloud, vous avez des instances de différents types (performance élevée vs standard) et vous voulez optimiser le placement.

### 4. **Conformité et Sécurité**
Certaines applications sensibles doivent tourner sur des nœuds dédiés et isolés.

### 5. **Différences de Stockage**
Vous avez des nœuds avec des SSD rapides et d'autres avec des disques classiques, et vous voulez placer vos bases de données sur les SSD.

---

## Labels sur les Nœuds

Pour utiliser des Node Selectors, vous devez d'abord comprendre les **labels** des nœuds.

### Qu'est-ce qu'un Label ?

Un label est une paire **clé=valeur** attachée à un objet Kubernetes (ici, un nœud). Par exemple :
- `type=gpu`
- `environment=production`
- `disktype=ssd`
- `zone=us-east-1a`

### Labels par Défaut

Kubernetes ajoute automatiquement certains labels à vos nœuds lors de leur création. Par exemple :
- `kubernetes.io/hostname` : nom d'hôte du nœud
- `kubernetes.io/os` : système d'exploitation (linux, windows)
- `kubernetes.io/arch` : architecture (amd64, arm64, etc.)

### Voir les Labels d'un Nœud

Pour afficher tous les labels d'un nœud :

```bash
kubectl get nodes --show-labels
```

Résultat exemple :
```
NAME     STATUS   ROLES    AGE   VERSION   LABELS
node1    Ready    <none>   5d    v1.28.0   kubernetes.io/arch=amd64,kubernetes.io/hostname=node1,kubernetes.io/os=linux
```

Pour voir les labels de manière plus lisible :

```bash
kubectl describe node node1
```

Dans la sortie, vous verrez une section **Labels** :
```
Labels:       kubernetes.io/arch=amd64
              kubernetes.io/hostname=node1
              kubernetes.io/os=linux
```

---

## Ajouter des Labels Personnalisés aux Nœuds

Vous pouvez ajouter vos propres labels aux nœuds pour les catégoriser selon vos besoins.

### Syntaxe de Base

```bash
kubectl label nodes <nom-du-noeud> <clé>=<valeur>
```

### Exemples Pratiques

#### Exemple 1 : Identifier un Nœud avec GPU

```bash
kubectl label nodes node1 hardware=gpu
```

#### Exemple 2 : Marquer un Environnement

```bash
kubectl label nodes node2 environment=production
kubectl label nodes node3 environment=development
```

#### Exemple 3 : Type de Stockage

```bash
kubectl label nodes node1 disktype=ssd
kubectl label nodes node2 disktype=hdd
```

#### Exemple 4 : Zone Géographique

```bash
kubectl label nodes node1 zone=europe
kubectl label nodes node2 zone=us
```

### Vérifier que le Label est Ajouté

```bash
kubectl get nodes node1 --show-labels
```

Ou :

```bash
kubectl describe node node1 | grep Labels -A 5
```

---

## Utiliser un Node Selector dans un Pod

Une fois que vos nœuds sont labellisés, vous pouvez utiliser un Node Selector dans vos Pods pour contrôler leur placement.

### Syntaxe de Base

Dans le fichier YAML de votre Pod, ajoutez la section `nodeSelector` :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod
spec:
  nodeSelector:
    clé: valeur
  containers:
  - name: mon-conteneur
    image: nginx
```

### Exemple Complet : Pod sur un Nœud GPU

Supposons que vous ayez labellisé un nœud avec `hardware=gpu`. Voici comment créer un Pod qui s'exécutera uniquement sur ce nœud :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-training-pod
  labels:
    app: machine-learning
spec:
  nodeSelector:
    hardware: gpu
  containers:
  - name: tensorflow
    image: tensorflow/tensorflow:latest-gpu
    resources:
      limits:
        nvidia.com/gpu: 1
```

**Ce que fait ce manifeste :**
- Le scheduler cherchera **uniquement** les nœuds ayant le label `hardware=gpu`
- Si aucun nœud avec ce label n'existe ou n'a de ressources disponibles, le Pod restera en état **Pending**

### Exemple Complet : Pod en Production

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-production
spec:
  nodeSelector:
    environment: production
  containers:
  - name: api
    image: mon-api:v1.2.3
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
```

---

## Node Selectors avec les Deployments

Les Node Selectors fonctionnent également avec les Deployments, ce qui est plus courant en pratique.

### Exemple : Deployment avec Node Selector

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-frontend
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
      nodeSelector:
        disktype: ssd
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

**Résultat :**
- Les 3 réplicas du Pod seront ordonnancés uniquement sur les nœuds ayant le label `disktype=ssd`
- Si vous avez plusieurs nœuds avec ce label, les Pods seront répartis entre eux par le scheduler

---

## Plusieurs Critères dans un Node Selector

Vous pouvez spécifier plusieurs paires clé-valeur dans un Node Selector. Le Pod sera ordonnancé **uniquement** sur les nœuds qui ont **tous** les labels spécifiés.

### Exemple : Plusieurs Labels

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-exigeant
spec:
  nodeSelector:
    environment: production
    disktype: ssd
    zone: europe
  containers:
  - name: app
    image: mon-app:latest
```

**Comportement :**
- Le Pod nécessite un nœud ayant les **trois** labels :
  - `environment=production`
  - `disktype=ssd`
  - `zone=europe`
- Si aucun nœud ne possède ces trois labels simultanément, le Pod reste **Pending**

---

## Que se Passe-t-il si Aucun Nœud ne Correspond ?

Si vous définissez un Node Selector mais qu'aucun nœud n'a les labels correspondants, le Pod restera indéfiniment en état **Pending**.

### Diagnostic

```bash
kubectl get pods
```

Résultat :
```
NAME            READY   STATUS    RESTARTS   AGE
mon-pod         0/1     Pending   0          5m
```

Pour comprendre le problème :

```bash
kubectl describe pod mon-pod
```

Dans les événements, vous verrez :
```
Events:
  Type     Reason            Message
  ----     ------            -------
  Warning  FailedScheduling  0/3 nodes are available:
                             3 node(s) didn't match Pod's node affinity/selector.
```

### Solutions

1. **Vérifier les labels des nœuds**
   ```bash
   kubectl get nodes --show-labels
   ```

2. **Ajouter le label manquant**
   ```bash
   kubectl label nodes node1 <clé>=<valeur>
   ```

3. **Modifier ou retirer le nodeSelector du Pod**

---

## Modifier ou Supprimer un Label

### Modifier un Label Existant

Pour changer la valeur d'un label :

```bash
kubectl label nodes node1 environment=staging --overwrite
```

L'option `--overwrite` est nécessaire si le label existe déjà.

### Supprimer un Label

Pour retirer un label d'un nœud, ajoutez un `-` à la fin du nom du label :

```bash
kubectl label nodes node1 disktype-
```

---

## Cas d'Usage Pratiques avec MicroK8s

### Cas 1 : Cluster Multi-Node avec Rôles Différents

Imaginons que vous ayez un cluster MicroK8s avec 3 nœuds :
- `node1` : nœud puissant pour les applications critiques
- `node2` : nœud standard pour les applications normales
- `node3` : nœud de test pour le développement

**Configuration :**

```bash
# Labelliser les nœuds
kubectl label nodes node1 tier=critical
kubectl label nodes node2 tier=standard
kubectl label nodes node3 tier=development

# Vérification
kubectl get nodes --show-labels
```

**Déploiement d'une application critique :**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-critique
spec:
  replicas: 2
  selector:
    matchLabels:
      app: critique
  template:
    metadata:
      labels:
        app: critique
    spec:
      nodeSelector:
        tier: critical
      containers:
      - name: app
        image: mon-app-critique:latest
```

### Cas 2 : Isolation des Bases de Données

Vous voulez que vos bases de données tournent sur un nœud dédié avec des disques SSD.

**Configuration :**

```bash
# Labelliser le nœud database
kubectl label nodes node1 workload=database
kubectl label nodes node1 disktype=ssd
```

**Déploiement PostgreSQL :**

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
      nodeSelector:
        workload: database
        disktype: ssd
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_PASSWORD
          value: "motdepasse"
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

### Cas 3 : Environnement de Test Isolé

Vous voulez isoler votre environnement de test sur un nœud spécifique.

**Configuration :**

```bash
kubectl label nodes node3 env=test
```

**Déploiement :**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-test
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      nodeSelector:
        env: test
      containers:
      - name: app
        image: mon-app:dev
```

---

## Limitations des Node Selectors

Bien que les Node Selectors soient simples et efficaces, ils ont certaines limitations :

### 1. **Contrainte Rigide (Hard Constraint)**
Un Node Selector est une contrainte **obligatoire**. Si aucun nœud ne correspond, le Pod ne sera jamais ordonnancé. Il n'y a pas de notion de "préférence".

### 2. **Logique Simple (ET uniquement)**
- Vous ne pouvez spécifier que des conditions "ET" (AND)
- Impossible d'exprimer des conditions "OU" (OR) comme "environment=prod OU environment=staging"
- Impossible d'exprimer des négations comme "PAS environment=test"

### 3. **Pas de Relations Entre Pods**
Les Node Selectors ne permettent pas d'exprimer des règles comme :
- "Ce Pod doit être sur le même nœud que cet autre Pod"
- "Ce Pod doit être sur un nœud différent de cet autre Pod"

### Solution pour des Besoins Plus Complexes

Pour des besoins plus avancés, Kubernetes offre les **Node Affinity** et **Pod Affinity/Anti-Affinity** que nous verrons dans les prochaines sections. Ces mécanismes sont plus puissants mais aussi plus complexes.

---

## Bonnes Pratiques

### 1. **Nommage Cohérent des Labels**

Utilisez des conventions de nommage claires et cohérentes :

✅ **Bon :**
```
environment=production
disktype=ssd
hardware=gpu
zone=europe-west1
```

❌ **Mauvais :**
```
env=prod
Disk_Type=SSD
HW=gpu
loc=ew1
```

### 2. **Documentation des Labels**

Maintenez une documentation des labels utilisés dans votre cluster :

```
# Labels Standard du Cluster
- environment: production, staging, development
- disktype: ssd, hdd, nvme
- hardware: standard, gpu, high-memory
- zone: europe, us, asia
- tier: critical, standard, low-priority
```

### 3. **Éviter la Sur-Contrainte**

N'ajoutez des Node Selectors que lorsque c'est vraiment nécessaire. Trop de contraintes peuvent :
- Compliquer l'ordonnancement
- Réduire la flexibilité
- Causer des problèmes si un nœud tombe en panne

### 4. **Tester les Node Selectors**

Avant de déployer en production, testez vos Node Selectors :

```bash
# Créer un Pod de test
kubectl run test-pod --image=nginx --dry-run=client -o yaml > test-pod.yaml

# Ajouter le nodeSelector dans test-pod.yaml
# Puis appliquer
kubectl apply -f test-pod.yaml

# Vérifier le placement
kubectl get pod test-pod -o wide
```

### 5. **Combiner avec des Resource Requests**

Toujours définir des `requests` même avec des Node Selectors :

```yaml
spec:
  nodeSelector:
    disktype: ssd
  containers:
  - name: app
    image: mon-app
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
```

### 6. **Prévoir les Scénarios de Panne**

Que se passe-t-il si votre unique nœud `environment=production` tombe en panne ? Ayez toujours plusieurs nœuds avec les mêmes labels critiques.

---

## Node Selectors vs Node Affinity

Vous entendrez souvent parler de **Node Affinity** comme alternative aux Node Selectors. Voici une comparaison rapide :

| Aspect | Node Selector | Node Affinity |
|--------|---------------|---------------|
| **Simplicité** | Très simple | Plus complexe |
| **Flexibilité** | Limitée (AND uniquement) | Grande (OR, NOT, préférences) |
| **Contrainte** | Hard (obligatoire) | Hard ou Soft (préférence) |
| **Syntaxe** | Concise | Plus verbeuse |
| **Cas d'usage** | Besoins simples | Besoins avancés |

**Recommandation :** Commencez par les Node Selectors. Si vous avez besoin de plus de flexibilité, explorez les Node Affinity (section 20.3).

---

## Vérifier l'Impact des Node Selectors

### Voir sur Quel Nœud un Pod Tourne

```bash
kubectl get pods -o wide
```

La colonne `NODE` indique sur quel nœud chaque Pod s'exécute.

### Filtrer les Pods par Nœud

```bash
kubectl get pods --field-selector spec.nodeName=node1
```

### Voir Tous les Pods d'un Nœud

```bash
kubectl get pods --all-namespaces -o wide | grep node1
```

---

## Commandes Utiles - Récapitulatif

```bash
# Afficher les labels des nœuds
kubectl get nodes --show-labels

# Ajouter un label à un nœud
kubectl label nodes <node> <key>=<value>

# Modifier un label existant
kubectl label nodes <node> <key>=<value> --overwrite

# Supprimer un label
kubectl label nodes <node> <key>-

# Afficher les détails d'un nœud (incluant labels)
kubectl describe node <node>

# Voir les Pods avec leur nœud
kubectl get pods -o wide

# Décrire un Pod (voir les events de scheduling)
kubectl describe pod <pod-name>

# Filtrer les nœuds par label
kubectl get nodes -l <key>=<value>
```

---

## Exemple Complet : Mise en Place d'un Environnement Multi-Tiers

Imaginons un scénario complet avec 3 nœuds dans MicroK8s.

### 1. Configuration des Nœuds

```bash
# Nœud 1 : Frontend (SSD, performances élevées)
kubectl label nodes node1 tier=frontend
kubectl label nodes node1 disktype=ssd

# Nœud 2 : Backend (standard)
kubectl label nodes node2 tier=backend
kubectl label nodes node2 disktype=ssd

# Nœud 3 : Database (SSD, haute mémoire)
kubectl label nodes node3 tier=database
kubectl label nodes node3 disktype=ssd
kubectl label nodes node3 memory=high
```

### 2. Déploiement Frontend

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      nodeSelector:
        tier: frontend
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
```

### 3. Déploiement Backend

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
      nodeSelector:
        tier: backend
      containers:
      - name: api
        image: mon-api:v1
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
```

### 4. Déploiement Database

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
      nodeSelector:
        tier: database
        memory: high
      containers:
      - name: postgres
        image: postgres:14
        resources:
          requests:
            cpu: "1000m"
            memory: "2Gi"
```

---

## Points Clés à Retenir

1. **Les Node Selectors sont simples** : Une manière directe et facile de contrôler le placement des Pods

2. **Les labels sont la clé** : Organisez vos nœuds avec des labels significatifs et cohérents

3. **Contrainte obligatoire** : Un Node Selector est une règle stricte - le Pod ne s'exécutera que sur les nœuds correspondants

4. **Plusieurs labels = ET logique** : Tous les labels spécifiés doivent être présents sur le nœud

5. **Diagnostic** : Utilisez `kubectl describe pod` pour comprendre les problèmes d'ordonnancement

6. **Commencez simple** : Pour des besoins simples, les Node Selectors suffisent amplement

---

## Conclusion

Les Node Selectors sont votre premier outil pour influencer les décisions du scheduler Kubernetes. Ils offrent un équilibre parfait entre simplicité et utilité pour la plupart des cas d'usage courants.

En maîtrisant les Node Selectors, vous pouvez :
- Optimiser l'utilisation de votre matériel
- Isoler les environnements
- Garantir que les applications critiques tournent sur les bons nœuds
- Améliorer les performances en plaçant les Pods stratégiquement

Dans la prochaine section, nous découvrirons les **Node Affinity**, qui offrent encore plus de flexibilité et de puissance pour des scénarios d'ordonnancement plus complexes.

---

## Pour Aller Plus Loin

- **Documentation officielle** : [Assign Pods to Nodes](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/)
- **Labels et Selectors** : [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- **Node Affinity** : Une alternative plus puissante (prochaine section)

---

**Prochain chapitre** : 20.3 Node Affinity et Anti-Affinity - Pour des règles d'ordonnancement plus sophistiquées

⏭️ [Node Affinity et Anti-Affinity](/20-ordonnancement-avance/03-node-affinity-et-anti-affinity.md)
