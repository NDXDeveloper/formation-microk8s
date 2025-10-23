🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.4 Namespaces

## Introduction

Jusqu'à présent, nous avons déployé toutes nos ressources (Pods, Deployments, Services) dans un seul "espace". Mais que se passe-t-il quand votre cluster grandit ? Comment organiser efficacement des dizaines ou centaines d'applications ?

Les **Namespaces** sont la solution Kubernetes pour organiser, isoler et gérer vos ressources de manière logique.

## Le problème : organisation à grande échelle

### Scénario problématique

Imaginez un cluster Kubernetes utilisé par plusieurs équipes :

```
Cluster unique :
- Équipe Frontend : 10 applications
- Équipe Backend : 15 microservices
- Équipe Data : 5 pipelines
- Équipe ML : 8 modèles
- Environnement de dev
- Environnement de staging
- Environnement de production
```

**Problèmes rencontrés** :

1. **Conflits de noms** : Deux équipes veulent créer un Service nommé "api"
2. **Visibilité chaotique** : `kubectl get pods` retourne 200 Pods de toutes les équipes
3. **Sécurité** : Comment empêcher l'équipe Frontend de modifier les ressources Backend ?
4. **Quotas** : Comment limiter les ressources utilisées par chaque équipe ?
5. **Facile de faire des erreurs** : Risque de supprimer accidentellement les ressources d'une autre équipe

### La solution : Namespaces

Les Namespaces permettent de **diviser un cluster** en plusieurs espaces virtuels isolés, chacun avec :
- Ses propres ressources (Pods, Services, etc.)
- Ses propres permissions (RBAC)
- Ses propres quotas de ressources
- Son propre périmètre de visibilité

## Qu'est-ce qu'un Namespace ?

### Définition simple

Un **Namespace** est un espace de noms virtuel qui permet de diviser les ressources d'un cluster Kubernetes en groupes logiques isolés.

### Analogie : les appartements d'un immeuble

Imaginez un **immeuble** (le cluster Kubernetes) avec plusieurs **appartements** (les Namespaces) :

```
┌────────────────────────────────────────┐
│         IMMEUBLE (Cluster K8s)         │
│                                        │
│  ┌───────────┐  ┌───────────┐          │
│  │Appart. 1  │  │Appart. 2  │          │
│  │(dev)      │  │(prod)     │          │
│  │           │  │           │          │
│  │ Canapé    │  │ Canapé    │  ← Même nom OK!
│  │ Table     │  │ Table     │          │
│  └───────────┘  └───────────┘          │
│                                        │
│  ┌───────────┐  ┌────────────┐         │
│  │Appart. 3  │  │Appart. 4   │         │
│  │(staging)  │  │(monitoring)│         │
│  └───────────┘  └────────────┘         │
└────────────────────────────────────────┘
```

**Caractéristiques** :
- Chaque appartement est **isolé** des autres
- Vous pouvez avoir un **canapé** dans chaque appartement (même nom de ressource)
- Les habitants de l'appartement 1 ne peuvent pas accéder directement à l'appartement 2
- L'immeuble (cluster) reste un seul bâtiment physique

### Ce que sont les Namespaces

Les Namespaces sont :
- Des **séparations logiques**, pas physiques
- Un mécanisme d'**organisation** des ressources
- Un moyen d'**isoler** les environnements
- Une façon d'appliquer des **quotas** et des **permissions**

### Ce que ne sont PAS les Namespaces

Les Namespaces ne sont **PAS** :
- Une isolation de sécurité forte (les Pods peuvent communiquer entre Namespaces par défaut)
- Des clusters séparés (tout est dans le même cluster)
- Une isolation réseau par défaut (nécessite des Network Policies)
- Adaptés à séparer des clients/tenants différents (utilisez plutôt des clusters séparés)

## Namespaces par défaut de Kubernetes

Lors de l'installation, Kubernetes crée automatiquement plusieurs Namespaces système.

### Lister les Namespaces existants

```bash
microk8s kubectl get namespaces
# Ou la version courte :
microk8s kubectl get ns
```

Sortie exemple :
```
NAME              STATUS   AGE
default           Active   10d
kube-system       Active   10d
kube-public       Active   10d
kube-node-lease   Active   10d
```

### 1. default

**Description** : Namespace par défaut pour vos ressources utilisateur.

**Utilisation** :
- Quand vous créez une ressource sans spécifier de Namespace, elle va dans `default`
- C'est là que vos applications sont créées par défaut

**Exemple** :
```bash
microk8s kubectl create deployment nginx --image=nginx
# → Créé dans le namespace "default"
```

### 2. kube-system

**Description** : Namespace pour les composants système de Kubernetes.

**Contenu typique** :
- CoreDNS (résolution DNS)
- Kubernetes Dashboard
- Metrics Server
- Contrôleurs système

**Visibilité** :
```bash
microk8s kubectl get pods -n kube-system
```

**Important** : Ne modifiez pas les ressources dans ce Namespace sauf si vous savez exactement ce que vous faites !

### 3. kube-public

**Description** : Namespace lisible publiquement (même par les utilisateurs non authentifiés).

**Utilisation** :
- Rarement utilisé
- Peut contenir des ConfigMaps avec des informations de découverte du cluster

### 4. kube-node-lease

**Description** : Namespace contenant les objets "Lease" utilisés pour le heartbeat des nœuds.

**Utilisation** :
- Système interne pour la détection de la santé des nœuds
- Vous n'avez généralement pas à interagir avec ce Namespace

### Schéma des Namespaces par défaut

```
┌────────────────────────────────────────────────┐
│           Cluster Kubernetes                   │
│                                                │
│  ┌──────────────────────────────────────┐      │
│  │  default                             │      │
│  │  Vos applications utilisateur        │      │
│  └──────────────────────────────────────┘      │
│                                                │
│  ┌──────────────────────────────────────┐      │
│  │  kube-system                         │      │
│  │  Composants système Kubernetes       │      │
│  └──────────────────────────────────────┘      │
│                                                │
│  ┌──────────────────────────────────────┐      │
│  │  kube-public                         │      │
│  │  Ressources publiques                │      │
│  └──────────────────────────────────────┘      │
│                                                │
│  ┌──────────────────────────────────────┐      │
│  │  kube-node-lease                     │      │
│  │  Heartbeat des nœuds                 │      │
│  └──────────────────────────────────────┘      │
└────────────────────────────────────────────────┘
```

## Créer et gérer des Namespaces

### Création d'un Namespace

**Méthode 1 : Via la ligne de commande**

```bash
# Créer un namespace nommé "dev"
microk8s kubectl create namespace dev

# Créer un namespace nommé "production"
microk8s kubectl create namespace production
```

**Méthode 2 : Via un fichier YAML**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
    team: platform
```

Appliquer :
```bash
microk8s kubectl apply -f staging-namespace.yaml
```

### Voir les détails d'un Namespace

```bash
microk8s kubectl describe namespace dev
```

Sortie exemple :
```
Name:         dev
Labels:       kubernetes.io/metadata.name=dev
Annotations:  <none>
Status:       Active

Resource Quotas
  Name:     <none>
  Resource  Used  Hard
  --------  ---   ---

No LimitRange resource.
```

### Supprimer un Namespace

```bash
microk8s kubectl delete namespace staging
```

**⚠️ ATTENTION** : Supprimer un Namespace supprime **toutes les ressources** qu'il contient (Pods, Services, Deployments, etc.) !

### Vérifier qu'un Namespace est bien supprimé

```bash
microk8s kubectl get namespace staging
```

Si le Namespace n'existe plus :
```
Error from server (NotFound): namespaces "staging" not found
```

## Utiliser les Namespaces

### Créer des ressources dans un Namespace spécifique

**Méthode 1 : Flag `-n` ou `--namespace`**

```bash
# Créer un Deployment dans le namespace "dev"
microk8s kubectl create deployment nginx --image=nginx -n dev

# Lister les Pods dans le namespace "dev"
microk8s kubectl get pods -n dev

# Lister les Services dans le namespace "production"
microk8s kubectl get services -n production
```

**Méthode 2 : Spécifier dans le fichier YAML**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app
  namespace: dev          # ← Spécifie le Namespace
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mon-app
  template:
    metadata:
      labels:
        app: mon-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

Appliquer :
```bash
microk8s kubectl apply -f mon-app.yaml
# → Créé dans le namespace "dev"
```

### Lister les ressources de tous les Namespaces

```bash
# Voir tous les Pods de tous les Namespaces
microk8s kubectl get pods --all-namespaces
# Ou version courte :
microk8s kubectl get pods -A

# Voir tous les Services de tous les Namespaces
microk8s kubectl get services -A
```

### Définir un Namespace par défaut

Pour éviter de taper `-n <namespace>` à chaque commande, vous pouvez définir un Namespace par défaut pour votre contexte :

```bash
# Définir "dev" comme namespace par défaut
microk8s kubectl config set-context --current --namespace=dev

# Maintenant, toutes les commandes utilisent "dev" par défaut
microk8s kubectl get pods        # Liste les Pods de "dev"
microk8s kubectl get services    # Liste les Services de "dev"

# Pour revenir à "default"
microk8s kubectl config set-context --current --namespace=default
```

## Isolation et portée des ressources

### Ressources namespacées

La plupart des ressources Kubernetes sont **namespacées**, c'est-à-dire qu'elles appartiennent à un Namespace spécifique :

**Ressources namespacées** :
- Pods
- Deployments
- ReplicaSets
- Services
- ConfigMaps
- Secrets
- PersistentVolumeClaims
- Ingress

**Conséquence** : Vous pouvez avoir deux ressources avec le même nom dans des Namespaces différents.

**Exemple** :
```bash
# Service "api" dans le namespace "dev"
microk8s kubectl create service clusterip api --tcp=80:80 -n dev

# Service "api" dans le namespace "prod" (même nom, pas de conflit !)
microk8s kubectl create service clusterip api --tcp=80:80 -n prod
```

### Ressources non-namespacées (cluster-scoped)

Certaines ressources sont **cluster-scoped** (à l'échelle du cluster) et n'appartiennent à aucun Namespace :

**Ressources non-namespacées** :
- Nodes (nœuds)
- Namespaces (eux-mêmes)
- PersistentVolumes
- StorageClasses
- ClusterRoles et ClusterRoleBindings

**Conséquence** : Ces ressources sont uniques dans tout le cluster.

### Voir quelles ressources sont namespacées

```bash
# Lister les ressources namespacées
microk8s kubectl api-resources --namespaced=true

# Lister les ressources non-namespacées (cluster-scoped)
microk8s kubectl api-resources --namespaced=false
```

### Schéma de l'isolation

```
┌─────────────────────────────────────────────────────┐
│              Cluster Kubernetes                     │
│                                                     │
│  Ressources cluster-scoped (accessibles partout) :  │
│  - Nodes                                            │
│  - PersistentVolumes                                │
│  - StorageClasses                                   │
│                                                     │
│  ┌───────────────────────┐  ┌───────────────────┐   │
│  │ Namespace: dev        │  │ Namespace: prod   │   │
│  │                       │  │                   │   │
│  │ - Deployment "api"    │  │ - Deployment "api"│   │
│  │ - Service "api"       │  │ - Service "api"   │   │
│  │ - ConfigMap "config"  │  │ - ConfigMap "cfg" │   │
│  │                       │  │                   │   │
│  │ Pods peuvent accéder  │  │                   │   │
│  │ Pods de "prod" via    │◄─┼───────────────────┤   │
│  │ DNS complet           │  │  (si autorisé)    │   │
│  └───────────────────────┘  └───────────────────┘   │
└─────────────────────────────────────────────────────┘
```

## DNS et Namespaces

### Format DNS complet

Les Services dans Kubernetes sont accessibles via DNS. Le format complet inclut le Namespace :

```
<nom-service>.<namespace>.svc.cluster.local
```

**Exemples** :
```
api.dev.svc.cluster.local
database.production.svc.cluster.local
cache.staging.svc.cluster.local
```

### Communication inter-Namespaces

**Depuis le même Namespace** :
```yaml
# Pod dans le namespace "dev" accédant à un Service "api" dans "dev"
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  namespace: dev
spec:
  containers:
  - name: app
    image: mon-app:v1
    env:
    - name: API_URL
      value: "http://api:80"              # Nom court suffit
```

**Depuis un autre Namespace** :
```yaml
# Pod dans le namespace "frontend" accédant à un Service "api" dans "backend"
apiVersion: v1
kind: Pod
metadata:
  name: web
  namespace: frontend
spec:
  containers:
  - name: app
    image: mon-app:v1
    env:
    - name: API_URL
      value: "http://api.backend:80"      # Inclure le namespace
      # Ou version complète :
      # value: "http://api.backend.svc.cluster.local:80"
```

### Tableau des formats DNS

| Localisation du client | Format DNS pour accéder au Service |
|------------------------|-----------------------------------|
| Même Namespace | `service-name` |
| Même Namespace | `service-name.namespace` |
| Autre Namespace | `service-name.namespace` (obligatoire) |
| Autre Namespace | `service-name.namespace.svc.cluster.local` |

### Schéma de communication DNS

```
┌────────────────────────────────────────────────────┐
│              Cluster Kubernetes                    │
│                                                    │
│  ┌─────────────────────┐                           │
│  │ Namespace: frontend │                           │
│  │                     │                           │
│  │  ┌──────────────┐   │                           │
│  │  │ Pod web      │   │                           │
│  │  │              │   │                           │
│  │  │ Appelle:     │   │                           │
│  │  │ api.backend  │───┼──────────┐                │
│  │  └──────────────┘   │          │                │
│  └─────────────────────┘          │                │
│                                   v                │
│  ┌─────────────────────────────────────────┐       │
│  │ Namespace: backend                      │       │
│  │                                         │       │
│  │  ┌────────────────────────────────┐     │       │
│  │  │ Service: api                   │     │       │
│  │  │ (api.backend.svc.cluster.local)│     │       │
│  │  └─────────────┬──────────────────┘     │       │
│  │                │                        │       │
│  │           ┌────┼────┐                   │       │
│  │           v    v    v                   │       │
│  │        ┌───┐┌───┐┌───┐                  │       │
│  │        │Pod││Pod││Pod│                  │       │
│  │        └───┘└───┘└───┘                  │       │
│  └─────────────────────────────────────────┘       │
└────────────────────────────────────────────────────┘
```

## Quotas de ressources (Resource Quotas)

Les Resource Quotas permettent de limiter la consommation de ressources dans un Namespace.

### Pourquoi des quotas ?

**Problèmes sans quotas** :
- Une équipe peut consommer toutes les ressources du cluster
- Impossible de garantir des ressources pour les environnements critiques
- Pas de contrôle des coûts par équipe/projet

### Créer un Resource Quota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    # Limites sur les ressources compute
    requests.cpu: "10"              # 10 CPUs au total
    requests.memory: 20Gi           # 20 GB de RAM au total
    limits.cpu: "20"                # Maximum 20 CPUs
    limits.memory: 40Gi             # Maximum 40 GB de RAM

    # Limites sur le nombre d'objets
    pods: "50"                      # Maximum 50 Pods
    services: "20"                  # Maximum 20 Services
    persistentvolumeclaims: "10"    # Maximum 10 PVCs
    secrets: "30"                   # Maximum 30 Secrets
    configmaps: "30"                # Maximum 30 ConfigMaps
```

Appliquer :
```bash
microk8s kubectl apply -f dev-quota.yaml
```

### Voir les quotas d'un Namespace

```bash
microk8s kubectl get resourcequota -n dev
```

Sortie exemple :
```
NAME        AGE   REQUEST                                                                                      LIMIT
dev-quota   5m    pods: 15/50, requests.cpu: 3/10, requests.memory: 6Gi/20Gi, services: 5/20, secrets: 10/30  limits.cpu: 6/20, limits.memory: 12Gi/40Gi
```

**Lecture** : `15/50` signifie "15 Pods utilisés sur 50 autorisés"

### Détails d'un Resource Quota

```bash
microk8s kubectl describe resourcequota dev-quota -n dev
```

### Comportement avec les quotas

**Quand un quota est atteint** :
- Kubernetes refuse de créer de nouvelles ressources
- Vous recevez une erreur explicite

**Exemple d'erreur** :
```
Error from server (Forbidden): pods "nginx-xyz" is forbidden:
exceeded quota: dev-quota, requested: pods=1, used: pods=50, limited: pods=50
```

### Quotas et requests/limits

Quand un Resource Quota définit `requests.cpu` ou `requests.memory`, **tous les conteneurs** créés dans ce Namespace **doivent** spécifier des `requests` et `limits`.

**Sans requests/limits** :
```yaml
# ❌ Sera rejeté si un ResourceQuota avec requests.cpu existe
spec:
  containers:
  - name: nginx
    image: nginx:1.25
```

**Avec requests/limits** :
```yaml
# ✅ Accepté
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
```

## LimitRanges : limites par défaut

Les LimitRanges définissent des valeurs par défaut et des limites pour les ressources individuelles (Pods, conteneurs).

### Pourquoi des LimitRanges ?

**Problèmes sans LimitRanges** :
- Un Pod peut demander 100 CPUs et bloquer tout le cluster
- Les développeurs oublient de spécifier requests/limits
- Incohérence des valeurs entre les Pods

### Créer un LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: dev
spec:
  limits:
  # Limites pour les conteneurs
  - type: Container
    default:                    # Limits par défaut
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:             # Requests par défaut
      cpu: "100m"
      memory: "128Mi"
    max:                        # Maximum autorisé
      cpu: "2"
      memory: "2Gi"
    min:                        # Minimum requis
      cpu: "50m"
      memory: "64Mi"

  # Limites pour les Pods
  - type: Pod
    max:
      cpu: "4"
      memory: "4Gi"
    min:
      cpu: "100m"
      memory: "128Mi"

  # Limites pour les PersistentVolumeClaims
  - type: PersistentVolumeClaim
    max:
      storage: "10Gi"
    min:
      storage: "1Gi"
```

Appliquer :
```bash
microk8s kubectl apply -f dev-limits.yaml
```

### Voir les LimitRanges

```bash
microk8s kubectl get limitrange -n dev
microk8s kubectl describe limitrange dev-limits -n dev
```

### Comportement avec LimitRange

**Pod sans requests/limits** :
```yaml
# Les valeurs par défaut sont appliquées automatiquement
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    # → requests.cpu: 100m, requests.memory: 128Mi (defaultRequest)
    # → limits.cpu: 500m, limits.memory: 512Mi (default)
```

**Pod qui dépasse les limites** :
```yaml
# ❌ Sera rejeté (dépasse le max)
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    resources:
      limits:
        cpu: "5"              # Dépasse le max de 2
```

## Organisation typique des Namespaces

### Par environnement

Structure classique pour séparer les environnements :

```
┌────────────────────────────────────────────────────┐
│              Cluster Kubernetes                    │
│                                                    │
│  ┌──────────────────┐                              │
│  │ dev              │  ← Développement             │
│  │ - Quotas légers  │                              │
│  │ - Accès facile   │                              │
│  └──────────────────┘                              │
│                                                    │
│  ┌──────────────────┐                              │
│  │ staging          │  ← Tests pré-production      │
│  │ - Quotas moyens  │                              │
│  │ - Similaire prod │                              │
│  └──────────────────┘                              │
│                                                    │
│  ┌──────────────────┐                              │
│  │ production       │  ← Production                │
│  │ - Quotas élevés  │                              │
│  │ - Haute sécu     │                              │
│  └──────────────────┘                              │
└────────────────────────────────────────────────────┘
```

**Exemple de création** :
```bash
microk8s kubectl create namespace dev
microk8s kubectl create namespace staging
microk8s kubectl create namespace production
```

### Par équipe

Structure pour les grandes organisations :

```
┌────────────────────────────────────────────────────┐
│              Cluster Kubernetes                    │
│                                                    │
│  ┌──────────────────┐  ┌──────────────────┐        │
│  │ team-frontend    │  │ team-backend     │        │
│  │                  │  │                  │        │
│  │ - App web 1      │  │ - API REST       │        │
│  │ - App web 2      │  │ - GraphQL        │        │
│  │ - Landing pages  │  │ - Microservices  │        │
│  └──────────────────┘  └──────────────────┘        │
│                                                    │
│  ┌──────────────────┐  ┌──────────────────┐        │
│  │ team-data        │  │ team-ml          │        │
│  │                  │  │                  │        │
│  │ - ETL pipelines  │  │ - ML models      │        │
│  │ - Databases      │  │ - Training jobs  │        │
│  └──────────────────┘  └──────────────────┘        │
└────────────────────────────────────────────────────┘
```

### Par application/projet

Structure pour isoler des applications complexes :

```
┌────────────────────────────────────────────────────┐
│              Cluster Kubernetes                    │
│                                                    │
│  ┌──────────────────┐                              │
│  │ ecommerce        │  ← Application e-commerce    │
│  │ - frontend       │                              │
│  │ - cart-service   │                              │
│  │ - payment-api    │                              │
│  └──────────────────┘                              │
│                                                    │
│  ┌──────────────────┐                              │
│  │ blog             │  ← Application blog          │
│  │ - wordpress      │                              │
│  │ - mysql          │                              │
│  └──────────────────┘                              │
│                                                    │
│  ┌──────────────────┐                              │
│  │ monitoring       │  ← Infrastructure monitoring │
│  │ - prometheus     │                              │
│  │ - grafana        │                              │
│  └──────────────────┘                              │
└────────────────────────────────────────────────────┘
```

### Approche hybride

Combinaison de plusieurs stratégies :

```
Namespaces créés :
- ecommerce-dev
- ecommerce-staging
- ecommerce-prod
- blog-dev
- blog-prod
- monitoring
- team-platform
```

## Exemple complet : architecture multi-environnements

### Création des Namespaces

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    environment: development
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
---
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
```

### Quotas par environnement

**Développement (dev) - quotas légers** :
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "5"
    requests.memory: 10Gi
    limits.cpu: "10"
    limits.memory: 20Gi
    pods: "30"
    services: "15"
```

**Production - quotas élevés** :
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prod-quota
  namespace: production
spec:
  hard:
    requests.cpu: "50"
    requests.memory: 100Gi
    limits.cpu: "100"
    limits.memory: 200Gi
    pods: "200"
    services: "50"
```

### Déploiement de la même application dans plusieurs environnements

**Application en dev** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: dev
spec:
  replicas: 1                    # 1 seul Pod en dev
  selector:
    matchLabels:
      app: webapp
      environment: dev
  template:
    metadata:
      labels:
        app: webapp
        environment: dev
    spec:
      containers:
      - name: app
        image: mon-app:dev-latest
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
```

**Application en production** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: production
spec:
  replicas: 5                    # 5 Pods en production
  selector:
    matchLabels:
      app: webapp
      environment: production
  template:
    metadata:
      labels:
        app: webapp
        environment: production
    spec:
      containers:
      - name: app
        image: mon-app:v1.2.3      # Version stable
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

## Bonnes pratiques

### 1. Nommage cohérent

**Conventions recommandées** :
```bash
# Par environnement
dev, staging, production

# Par équipe
team-frontend, team-backend, team-data

# Par application
myapp-dev, myapp-staging, myapp-prod

# Par projet
project-alpha, project-beta
```

### 2. Utiliser des labels

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    team: platform
    cost-center: "12345"
    contact: "ops@example.com"
```

Les labels permettent de :
- Filtrer et sélectionner les Namespaces
- Organiser la facturation
- Automatiser les opérations

### 3. Définir des quotas

**Toujours** définir des Resource Quotas pour éviter qu'un Namespace consomme toutes les ressources :

```yaml
# Même pour "dev", définissez un quota raisonnable
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "5"
    requests.memory: 10Gi
    pods: "30"
```

### 4. Utiliser des LimitRanges

Définissez des valeurs par défaut pour éviter les erreurs :

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: dev
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
```

### 5. Ne pas utiliser "default" en production

```bash
# ❌ Mauvais
microk8s kubectl apply -f production-app.yaml
# → Va dans "default" par erreur

# ✅ Bon
microk8s kubectl apply -f production-app.yaml -n production
# Ou spécifier dans le YAML
```

### 6. Documentation des Namespaces

Utilisez des annotations pour documenter :

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  annotations:
    description: "Production environment for customer-facing applications"
    owner: "platform-team@example.com"
    incident-contact: "oncall@example.com"
    runbook: "https://wiki.example.com/runbooks/production"
```

### 7. Séparer les environnements sensibles

Ne mélangez **jamais** production et développement dans le même Namespace :

```bash
# ❌ Très mauvais
Namespace: default
- app-prod (production)
- app-dev (développement)
- test-stuff (expérimentation)

# ✅ Excellent
Namespace: production
- app (version stable)

Namespace: dev
- app (version de développement)
```

### 8. Utiliser des Network Policies

Les Namespaces seuls n'isolent pas le réseau. Utilisez des Network Policies :

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: production
spec:
  podSelector: {}                    # S'applique à tous les Pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}                # Autorise seulement le trafic depuis le même Namespace
```

## Commandes utiles

### Gestion des Namespaces

```bash
# Créer un Namespace
microk8s kubectl create namespace dev

# Lister tous les Namespaces
microk8s kubectl get namespaces
microk8s kubectl get ns

# Voir les détails d'un Namespace
microk8s kubectl describe namespace dev

# Supprimer un Namespace (⚠️ supprime tout son contenu)
microk8s kubectl delete namespace staging
```

### Opérations dans un Namespace

```bash
# Lister les Pods dans un Namespace
microk8s kubectl get pods -n dev

# Lister tous les Pods de tous les Namespaces
microk8s kubectl get pods --all-namespaces
microk8s kubectl get pods -A

# Décrire un Pod dans un Namespace
microk8s kubectl describe pod mon-pod -n dev

# Voir les logs d'un Pod
microk8s kubectl logs mon-pod -n production

# Exécuter une commande dans un Pod
microk8s kubectl exec -it mon-pod -n dev -- bash
```

### Gestion des quotas et limites

```bash
# Voir les Resource Quotas
microk8s kubectl get resourcequota -n dev
microk8s kubectl describe resourcequota dev-quota -n dev

# Voir les LimitRanges
microk8s kubectl get limitrange -n dev
microk8s kubectl describe limitrange dev-limits -n dev
```

### Changer le Namespace par défaut

```bash
# Définir "dev" comme namespace par défaut
microk8s kubectl config set-context --current --namespace=dev

# Vérifier le namespace actuel
microk8s kubectl config view --minify | grep namespace:

# Revenir à "default"
microk8s kubectl config set-context --current --namespace=default
```

### Copier des ressources entre Namespaces

```bash
# Exporter une ressource depuis un namespace
microk8s kubectl get configmap ma-config -n dev -o yaml > config.yaml

# Modifier le namespace dans le fichier
# metadata.namespace: staging

# Appliquer dans le nouveau namespace
microk8s kubectl apply -f config.yaml -n staging
```

## Dépannage

### Problème 1 : Ressource non trouvée

**Symptôme** :
```bash
microk8s kubectl get pod mon-pod
# Error from server (NotFound): pods "mon-pod" not found
```

**Cause** : La ressource n'est pas dans le Namespace actuel.

**Solution** :
```bash
# Chercher dans tous les Namespaces
microk8s kubectl get pod mon-pod --all-namespaces

# Ou spécifier le bon Namespace
microk8s kubectl get pod mon-pod -n production
```

### Problème 2 : Quota dépassé

**Symptôme** :
```
Error from server (Forbidden): pods "nginx" is forbidden:
exceeded quota: dev-quota, requested: pods=1, used: pods=30, limited: pods=30
```

**Cause** : Le quota de Pods est atteint dans ce Namespace.

**Solution** :
```bash
# Voir l'utilisation actuelle
microk8s kubectl describe resourcequota dev-quota -n dev

# Augmenter le quota (si nécessaire)
# Éditer le ResourceQuota
microk8s kubectl edit resourcequota dev-quota -n dev

# Ou supprimer des Pods inutilisés
microk8s kubectl delete pod old-pod -n dev
```

### Problème 3 : Impossible de supprimer un Namespace

**Symptôme** :
```bash
microk8s kubectl delete namespace dev
# Le namespace reste en état "Terminating" indéfiniment
```

**Cause** : Des finalizers empêchent la suppression ou des ressources ne peuvent être supprimées.

**Diagnostic** :
```bash
microk8s kubectl get namespace dev -o yaml
```

**Solution** :
```bash
# Forcer la suppression (à utiliser avec précaution)
microk8s kubectl get namespace dev -o json \
  | jq '.spec.finalizers = []' \
  | microk8s kubectl replace --raw "/api/v1/namespaces/dev/finalize" -f -
```

### Problème 4 : Services non accessibles entre Namespaces

**Symptôme** : Un Pod dans le Namespace "frontend" ne peut pas accéder à un Service dans "backend".

**Cause** : Format DNS incorrect ou Network Policy bloquante.

**Solution** :
```bash
# Utiliser le format DNS complet
# service-name.namespace.svc.cluster.local

# Exemple :
curl http://api.backend.svc.cluster.local:80
```

**Vérifier les Network Policies** :
```bash
microk8s kubectl get networkpolicies -n backend
```

### Problème 5 : Requests/Limits requis

**Symptôme** :
```
Error from server (Forbidden): error when creating "pod.yaml":
pods "test" is forbidden: failed quota: dev-quota:
must specify requests.cpu,requests.memory
```

**Cause** : Un ResourceQuota existe et nécessite des requests/limits.

**Solution** : Ajouter les requests/limits dans votre Pod/Deployment :
```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

## Résumé des points clés

- Les **Namespaces** permettent d'organiser et d'isoler logiquement les ressources dans un cluster
- Kubernetes crée 4 Namespaces par défaut : `default`, `kube-system`, `kube-public`, `kube-node-lease`
- La plupart des ressources sont **namespacées** (Pods, Services, Deployments)
- Certaines ressources sont **cluster-scoped** (Nodes, PersistentVolumes)
- Le **DNS** entre Namespaces utilise le format : `service.namespace.svc.cluster.local`
- Les **Resource Quotas** limitent la consommation de ressources par Namespace
- Les **LimitRanges** définissent des valeurs par défaut et des limites pour les ressources individuelles
- Organisez vos Namespaces **par environnement**, **par équipe**, ou **par application**
- Les Namespaces ne fournissent **pas d'isolation réseau** par défaut (utilisez Network Policies)
- Utilisez `-n <namespace>` pour spécifier le Namespace dans les commandes kubectl

## Prochaines étapes

Maintenant que vous maîtrisez les Namespaces, vous êtes prêt à découvrir :
- **Les Labels et Selectors** : comment organiser et sélectionner précisément vos ressources
- **Les ConfigMaps** : comment gérer la configuration de vos applications
- **Les Secrets** : comment stocker et utiliser des données sensibles de manière sécurisée

Les Namespaces sont essentiels pour organiser un cluster Kubernetes de manière professionnelle, surtout quand il grandit en complexité !

⏭️ [Labels et Selectors](/03-concepts-kubernetes-essentiels/05-labels-et-selectors.md)
