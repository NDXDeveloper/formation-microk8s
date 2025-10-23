🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.5 Labels et Selectors

## Introduction

Vous avez maintenant créé des Pods, des Deployments, des Services et organisé vos ressources dans des Namespaces. Mais comment Kubernetes sait-il quels Pods appartiennent à quel Service ? Comment identifier rapidement toutes les ressources d'une application spécifique ?

Les **Labels** et **Selectors** sont la réponse. Ils constituent le système d'organisation et de sélection des ressources dans Kubernetes.

## Le problème : identifier et organiser les ressources

### Scénario problématique

Imaginez que vous avez déployé une application e-commerce complète dans votre cluster :

```
Cluster Kubernetes :
- 50 Pods frontend
- 30 Pods backend API
- 20 Pods worker (traitement asynchrone)
- 10 Pods base de données
- Services pour chaque composant
- Plusieurs environnements (dev, staging, prod)
- Plusieurs versions en cours (v1.0, v1.1, v2.0 canary)
```

**Questions difficiles sans système d'organisation** :
- Comment lister uniquement les Pods du frontend en production ?
- Comment trouver toutes les ressources de la version 2.0 ?
- Comment un Service sait quels Pods il doit exposer ?
- Comment faire un rollback uniquement sur les composants backend ?
- Comment appliquer une opération à tous les Pods d'une équipe spécifique ?

**Sans labels** : Impossible à gérer efficacement !

## Qu'est-ce qu'un Label ?

### Définition simple

Un **Label** est une paire **clé-valeur** attachée à une ressource Kubernetes pour l'identifier et l'organiser.

### Analogie : les étiquettes sur des boîtes

Imaginez un **entrepôt** rempli de boîtes (les ressources Kubernetes) :

```
┌─────────────────────────────────────────────────┐
│              Entrepôt (Cluster)                 │
│                                                 │
│  ┌──────────────┐  ┌──────────────┐             │
│  │ Boîte 1      │  │ Boîte 2      │             │
│  │              │  │              │             │
│  │ Étiquettes:  │  │ Étiquettes:  │             │
│  │ • type: vin  │  │ • type: eau  │             │
│  │ • région: FR │  │ • marque: X  │             │
│  │ • année:2020 │  │ • volume: 1L │             │
│  └──────────────┘  └──────────────┘             │
│                                                 │
│  ┌──────────────┐  ┌──────────────┐             │
│  │ Boîte 3      │  │ Boîte 4      │             │
│  │              │  │              │             │
│  │ Étiquettes:  │  │ Étiquettes:  │             │
│  │ • type: vin  │  │ • type: vin  │             │
│  │ • région: IT │  │ • région: FR │             │
│  │ • année:2018 │  │ • année:2020 │             │
│  └──────────────┘  └──────────────┘             │
└─────────────────────────────────────────────────┘
```

**Avec ces étiquettes**, vous pouvez facilement :
- Trouver tous les vins français : `type=vin, région=FR`
- Lister toutes les bouteilles de 2020 : `année=2020`
- Sélectionner les vins italiens de 2018 : `type=vin, région=IT, année=2018`

Les **Labels Kubernetes** fonctionnent exactement de cette manière !

### Structure d'un Label

Un Label est composé de :
- **Clé** : un nom (ex: `app`, `environment`, `version`)
- **Valeur** : une valeur correspondante (ex: `nginx`, `production`, `v1.0`)

**Format** :
```
clé: valeur
```

**Exemples** :
```yaml
app: frontend
environment: production
version: v2.1.0
tier: backend
team: platform
```

### Caractéristiques des Labels

**Règles pour les clés** :
- Maximum 63 caractères (ou 253 avec préfixe)
- Peuvent contenir : lettres, chiffres, tirets `-`, underscores `_`, points `.`
- Doivent commencer et finir par un caractère alphanumérique

**Règles pour les valeurs** :
- Maximum 63 caractères
- Peuvent contenir : lettres, chiffres, tirets `-`, underscores `_`, points `.`
- Peuvent être vides

**Exemples valides** :
```yaml
app: nginx                          ✓
environment: production             ✓
version: v1.2.3                     ✓
team.lead: john                     ✓
kubernetes.io/component: apiserver  ✓ (avec préfixe)
release: "2024"                     ✓
```

**Exemples invalides** :
```yaml
app name: nginx                     ✗ (espace dans la clé)
environment: production-env-2024-with-very-long-name-exceeding-limit  ✗ (trop long)
-version: v1                        ✗ (commence par -)
```

## Labels dans les manifestes YAML

### Ajouter des Labels à un Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod-nginx
  labels:                    # Section labels
    app: nginx               # Label 1
    environment: production  # Label 2
    version: v1.25           # Label 3
    team: frontend           # Label 4
spec:
  containers:
  - name: nginx
    image: nginx:1.25
```

### Ajouter des Labels à un Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
  labels:                        # Labels du Deployment lui-même
    app: frontend
    component: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend              # Selector pour trouver les Pods
  template:
    metadata:
      labels:                    # Labels des Pods créés
        app: frontend
        environment: production
        version: v2.0
    spec:
      containers:
      - name: web
        image: mon-frontend:v2.0
```

**Important** : Notez que le Deployment a **deux ensembles de labels** :
1. Les labels du Deployment lui-même (`metadata.labels`)
2. Les labels des Pods qu'il crée (`template.metadata.labels`)

### Ajouter des Labels à un Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  labels:                    # Labels du Service
    app: frontend
    component: loadbalancer
spec:
  selector:                  # Selector pour trouver les Pods
    app: frontend            # Sélectionne les Pods avec app=frontend
  ports:
  - port: 80
    targetPort: 8080
```

## Qu'est-ce qu'un Selector ?

### Définition simple

Un **Selector** est une requête qui sélectionne des ressources en fonction de leurs Labels.

C'est comme un **filtre de recherche** dans votre entrepôt :
- "Donne-moi toutes les boîtes avec `type=vin` ET `région=FR`"
- Le selector : `type=vin, région=FR`
- Le résultat : Toutes les boîtes correspondantes

### Utilisation des Selectors

Les Selectors sont utilisés partout dans Kubernetes :

**1. Services → Pods**
```yaml
# Service
selector:
  app: nginx
# → Expose tous les Pods avec le label app=nginx
```

**2. Deployments → Pods**
```yaml
# Deployment
selector:
  matchLabels:
    app: frontend
# → Gère tous les Pods avec le label app=frontend
```

**3. ReplicaSets → Pods**
```yaml
# ReplicaSet
selector:
  matchLabels:
    app: backend
# → Maintient le nombre de Pods avec app=backend
```

**4. Network Policies → Pods**
```yaml
# NetworkPolicy
spec:
  podSelector:
    matchLabels:
      role: database
# → S'applique aux Pods avec role=database
```

### Schéma : Labels et Selectors en action

```
┌───────────────────────────────────────────────────────┐
│                Service "frontend"                     │
│                                                       │
│  selector:                                            │
│    app: frontend        ← Recherche ce label          │
│    tier: web                                          │
└───────────────┬───────────────────────────────────────┘
                │
                │ Sélectionne tous les Pods correspondants
                │
    ┌───────────┼───────────┬───────────┐
    │           │           │           │
    v           v           v           v
┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│ Pod 1  │  │ Pod 2  │  │ Pod 3  │  │ Pod 4  │
│        │  │        │  │        │  │        │
│ app:   │  │ app:   │  │ app:   │  │ app:   │
│frontend│  │frontend│  │backend │  │frontend│
│ tier:  │  │ tier:  │  │ tier:  │  │ tier:  │
│ web    │  │ web    │  │ api    │  │ cache  │
└────────┘  └────────┘  └────────┘  └────────┘
   ✓           ✓           ✗           ✗
Sélectionné Sélectionné  Exclu     Exclu (tier≠web)
```

Dans cet exemple :
- **Pod 1** et **Pod 2** correspondent (app=frontend ET tier=web)
- **Pod 3** ne correspond pas (app=backend)
- **Pod 4** ne correspond pas (tier=cache au lieu de web)

## Types de Selectors

Kubernetes supporte deux types de selectors : **Equality-based** et **Set-based**.

### 1. Equality-based Selectors

Les selectors basés sur l'égalité utilisent les opérateurs `=`, `==`, et `!=`.

**Syntaxe** :
```yaml
selector:
  app: nginx              # app égal à nginx
  environment: production # environment égal à production
```

**Opérateurs** :
- `=` ou `==` : égal à
- `!=` : différent de

**Exemples** :

```yaml
# Sélectionne les Pods où app=nginx
selector:
  app: nginx

# Sélectionne les Pods où app=nginx ET environment≠dev
selector:
  app: nginx
  environment: production
```

**Utilisation en ligne de commande** :
```bash
# Lister les Pods où app=nginx
microk8s kubectl get pods -l app=nginx

# Lister les Pods où app=nginx ET environment=production
microk8s kubectl get pods -l app=nginx,environment=production

# Lister les Pods où environment≠dev
microk8s kubectl get pods -l environment!=dev
```

### 2. Set-based Selectors

Les selectors basés sur les ensembles utilisent les opérateurs `in`, `notin`, et `exists`.

**Syntaxe** :
```yaml
selector:
  matchExpressions:
  - key: environment
    operator: In
    values:
    - production
    - staging
  - key: tier
    operator: NotIn
    values:
    - cache
```

**Opérateurs** :
- `In` : la valeur est dans la liste
- `NotIn` : la valeur n'est pas dans la liste
- `Exists` : la clé existe (quelle que soit la valeur)
- `DoesNotExist` : la clé n'existe pas

**Exemples** :

**Exemple 1 : In**
```yaml
selector:
  matchExpressions:
  - key: environment
    operator: In
    values:
    - production
    - staging
# → Sélectionne les Pods où environment est "production" OU "staging"
```

**Exemple 2 : NotIn**
```yaml
selector:
  matchExpressions:
  - key: tier
    operator: NotIn
    values:
    - cache
    - legacy
# → Sélectionne les Pods où tier n'est PAS "cache" ou "legacy"
```

**Exemple 3 : Exists**
```yaml
selector:
  matchExpressions:
  - key: beta-feature
    operator: Exists
# → Sélectionne les Pods qui ont le label "beta-feature" (quelle que soit sa valeur)
```

**Exemple 4 : DoesNotExist**
```yaml
selector:
  matchExpressions:
  - key: deprecated
    operator: DoesNotExist
# → Sélectionne les Pods qui n'ont PAS le label "deprecated"
```

### Combiner plusieurs conditions

Vous pouvez combiner `matchLabels` (equality-based) et `matchExpressions` (set-based) :

```yaml
selector:
  matchLabels:                   # ET logique
    app: frontend
  matchExpressions:              # ET logique
  - key: environment
    operator: In
    values:
    - production
    - staging
  - key: version
    operator: NotIn
    values:
    - legacy
    - deprecated
```

**Signification** :
- Le Pod doit avoir `app=frontend` (matchLabels)
- **ET** `environment` doit être "production" OU "staging"
- **ET** `version` ne doit PAS être "legacy" ou "deprecated"

## Labels recommandés (Best Practices)

Kubernetes recommande un ensemble de labels standards pour améliorer l'organisation.

### Labels standards recommandés

```yaml
metadata:
  labels:
    # Identification de l'application
    app.kubernetes.io/name: mysql                    # Nom de l'application
    app.kubernetes.io/instance: mysql-instance-1     # Instance unique
    app.kubernetes.io/version: "5.7.21"              # Version de l'application
    app.kubernetes.io/component: database            # Composant (database, frontend, etc.)
    app.kubernetes.io/part-of: ecommerce             # Application parente

    # Gestion
    app.kubernetes.io/managed-by: helm               # Outil de gestion (helm, kubectl, etc.)

    # Personnalisés (exemples)
    environment: production
    team: backend
    cost-center: "12345"
```

### Exemple complet avec labels recommandés

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
  labels:
    app.kubernetes.io/name: frontend
    app.kubernetes.io/instance: frontend-prod
    app.kubernetes.io/version: "2.1.0"
    app.kubernetes.io/component: web
    app.kubernetes.io/part-of: ecommerce
    app.kubernetes.io/managed-by: kubectl
    environment: production
    team: frontend
spec:
  replicas: 5
  selector:
    matchLabels:
      app.kubernetes.io/name: frontend
      app.kubernetes.io/instance: frontend-prod
  template:
    metadata:
      labels:
        app.kubernetes.io/name: frontend
        app.kubernetes.io/instance: frontend-prod
        app.kubernetes.io/version: "2.1.0"
        app.kubernetes.io/component: web
        app.kubernetes.io/part-of: ecommerce
        environment: production
        team: frontend
    spec:
      containers:
      - name: web
        image: mon-frontend:2.1.0
```

### Labels simples vs labels recommandés

**Labels simples (OK pour débuter)** :
```yaml
labels:
  app: frontend
  environment: production
  version: v2.1
```

**Labels recommandés (meilleur pour la production)** :
```yaml
labels:
  app.kubernetes.io/name: frontend
  app.kubernetes.io/instance: frontend-prod
  app.kubernetes.io/version: "2.1.0"
  app.kubernetes.io/component: web
  environment: production
```

Les deux approches fonctionnent. Les labels recommandés offrent plus de standardisation et d'interopérabilité.

## Opérations avec Labels et Selectors

### Lister les ressources avec des labels

```bash
# Lister tous les Pods avec leurs labels
microk8s kubectl get pods --show-labels

# Lister les Pods avec un label spécifique
microk8s kubectl get pods -l app=nginx

# Lister les Pods avec plusieurs labels (ET logique)
microk8s kubectl get pods -l app=nginx,environment=production

# Lister les Pods où environment n'est pas "dev"
microk8s kubectl get pods -l environment!=dev

# Lister avec un selector set-based (In)
microk8s kubectl get pods -l 'environment in (production,staging)'

# Lister avec un selector set-based (NotIn)
microk8s kubectl get pods -l 'tier notin (cache,legacy)'

# Lister les Pods qui ont un label "beta-feature" (quelle que soit la valeur)
microk8s kubectl get pods -l beta-feature

# Lister les Pods qui n'ont PAS de label "deprecated"
microk8s kubectl get pods -l '!deprecated'
```

### Ajouter des labels à une ressource existante

```bash
# Ajouter un label à un Pod
microk8s kubectl label pod mon-pod environment=production

# Ajouter plusieurs labels
microk8s kubectl label pod mon-pod team=frontend tier=web

# Ajouter un label à tous les Pods d'un Deployment
microk8s kubectl label pods -l app=nginx version=v1.25
```

### Modifier un label existant

```bash
# Modifier un label (nécessite --overwrite)
microk8s kubectl label pod mon-pod environment=staging --overwrite

# Changer la version de tous les Pods nginx
microk8s kubectl label pods -l app=nginx version=v1.26 --overwrite
```

### Supprimer un label

```bash
# Supprimer un label (ajouter un - après le nom)
microk8s kubectl label pod mon-pod environment-

# Supprimer plusieurs labels
microk8s kubectl label pod mon-pod team- tier-
```

### Afficher des colonnes personnalisées basées sur les labels

```bash
# Afficher une colonne avec la valeur d'un label
microk8s kubectl get pods -L app,environment,version
```

Sortie exemple :
```
NAME              READY   STATUS    APP        ENVIRONMENT   VERSION
pod-frontend-1    1/1     Running   frontend   production    v2.1
pod-frontend-2    1/1     Running   frontend   production    v2.1
pod-backend-1     1/1     Running   backend    staging       v1.5
```

## Cas d'usage pratiques

### Cas 1 : Service exposant plusieurs Deployments

Parfois, vous voulez qu'un Service expose des Pods de plusieurs Deployments.

**Architecture** :
```
Service "web"
  └─ Sélectionne tous les Pods avec app=web
      ├─ Deployment "frontend" (3 Pods)
      └─ Deployment "admin-panel" (2 Pods)
```

**Deployment Frontend** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web              # Label commun
      component: frontend
  template:
    metadata:
      labels:
        app: web            # Label commun
        component: frontend
    spec:
      containers:
      - name: web
        image: frontend:v1
```

**Deployment Admin Panel** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin-panel
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web              # Label commun
      component: admin
  template:
    metadata:
      labels:
        app: web            # Label commun
        component: admin
    spec:
      containers:
      - name: admin
        image: admin:v1
```

**Service unique** :
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web                # Sélectionne TOUS les Pods avec app=web
  ports:
  - port: 80
    targetPort: 8080
```

Le Service expose les 5 Pods (3 frontend + 2 admin) !

### Cas 2 : Déploiement Canary

Un déploiement canary expose une nouvelle version à un petit pourcentage d'utilisateurs.

**Architecture** :
```
Service "api"
  └─ Sélectionne tous les Pods avec app=api
      ├─ Deployment "api-stable" (9 Pods) version=v1.0
      └─ Deployment "api-canary" (1 Pod) version=v2.0
```

**90% du trafic → v1.0, 10% → v2.0**

**Deployment Stable** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: api
      version: v1.0
  template:
    metadata:
      labels:
        app: api
        version: v1.0
    spec:
      containers:
      - name: api
        image: api:v1.0
```

**Deployment Canary** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
      version: v2.0
  template:
    metadata:
      labels:
        app: api
        version: v2.0
    spec:
      containers:
      - name: api
        image: api:v2.0
```

**Service** :
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api              # Sélectionne les deux versions
  ports:
  - port: 80
    targetPort: 8080
```

### Cas 3 : Sélection par environnement

Lister et gérer les ressources par environnement.

```bash
# Lister tous les Pods de production
microk8s kubectl get pods -l environment=production

# Lister tous les Services de staging
microk8s kubectl get services -l environment=staging

# Supprimer tous les Pods de dev
microk8s kubectl delete pods -l environment=dev

# Redémarrer tous les Deployments de production
microk8s kubectl rollout restart deployment -l environment=production
```

### Cas 4 : Sélection par équipe

Gérer les ressources par équipe.

```bash
# Lister toutes les ressources de l'équipe frontend
microk8s kubectl get all -l team=frontend

# Voir les ressources consommées par l'équipe backend
microk8s kubectl top pods -l team=backend

# Appliquer une configuration à tous les Pods de l'équipe data
microk8s kubectl annotate pods -l team=data monitoring=enabled
```

### Cas 5 : Blue-Green Deployment

Deux environnements identiques, on bascule le trafic de l'un à l'autre.

**Deployment Blue (version actuelle)** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  replicas: 5
  selector:
    matchLabels:
      app: myapp
      color: blue
  template:
    metadata:
      labels:
        app: myapp
        color: blue
        version: v1.0
    spec:
      containers:
      - name: app
        image: myapp:v1.0
```

**Deployment Green (nouvelle version)** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  replicas: 5
  selector:
    matchLabels:
      app: myapp
      color: green
  template:
    metadata:
      labels:
        app: myapp
        color: green
        version: v2.0
    spec:
      containers:
      - name: app
        image: myapp:v2.0
```

**Service pointant vers Blue** :
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    color: blue        # Trafic vers blue
  ports:
  - port: 80
    targetPort: 8080
```

**Pour basculer vers Green** :
```bash
# Modifier le Service pour pointer vers green
microk8s kubectl patch service myapp-service -p '{"spec":{"selector":{"color":"green"}}}'
```

Instantanément, tout le trafic bascule vers la nouvelle version !

## Labels vs Annotations

### Différence clé

| Caractéristique | Labels | Annotations |
|----------------|--------|-------------|
| **But** | Identification et sélection | Métadonnées non-identifiantes |
| **Utilisé par** | Kubernetes (selectors) | Outils externes, documentation |
| **Taille** | Limité (63 caractères) | Plus grande (256 Ko) |
| **Requêtes** | Peut être requis/filtré | Ne peut pas être requis |
| **Contenu** | Valeurs simples | Peut contenir JSON, URLs, etc. |

### Quand utiliser des Labels

Utilisez des **Labels** quand :
- Vous voulez sélectionner/filtrer des ressources
- L'information est utilisée par Kubernetes (Services, Deployments, etc.)
- Vous avez besoin de requêtes et de filtres
- L'information identifie la ressource

**Exemples** :
```yaml
labels:
  app: nginx
  environment: production
  version: v1.25
  tier: frontend
```

### Quand utiliser des Annotations

Utilisez des **Annotations** quand :
- Vous stockez des métadonnées pour des outils externes
- Vous documentez la ressource
- Vous avez besoin de stocker des données structurées (JSON)
- L'information ne sert pas à sélectionner/filtrer

**Exemples** :
```yaml
annotations:
  description: "Frontend web server for the main application"
  contact: "team-frontend@example.com"
  documentation: "https://wiki.example.com/frontend"
  prometheus.io/scrape: "true"
  prometheus.io/port: "9090"
  kubectl.kubernetes.io/last-applied-configuration: '{"apiVersion":"v1"...}'
```

### Exemple combiné

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
  labels:                              # Pour sélection
    app: frontend
    environment: production
    version: v2.1.0
    tier: web
  annotations:                         # Pour documentation
    description: "Main frontend web server"
    owner: "team-frontend@example.com"
    runbook: "https://wiki.example.com/runbooks/frontend"
    created-by: "John Doe"
    jira-ticket: "PROJ-1234"
    prometheus.io/scrape: "true"
spec:
  containers:
  - name: web
    image: frontend:v2.1.0
```

## Bonnes pratiques

### 1. Utiliser des labels cohérents et standards

**✅ Bon** :
```yaml
labels:
  app: nginx
  environment: production
  version: v1.25
  team: platform
```

**❌ Mauvais** :
```yaml
labels:
  application: nginx    # Incohérent (app vs application)
  env: prod             # Abréviation (environment vs env)
  ver: 1.25             # Abréviation (version vs ver)
  owner: platform       # Incohérent (team vs owner)
```

### 2. Définir une convention de nommage

Établissez une convention et respectez-la dans tout le cluster :

**Exemple de convention** :
```yaml
# Obligatoires
app: <nom-application>
environment: <dev|staging|production>
version: <version-semantique>

# Optionnels
component: <frontend|backend|database|cache>
team: <nom-equipe>
tier: <web|api|data>
```

### 3. Labels courts et simples

**✅ Bon** :
```yaml
labels:
  app: nginx
  env: prod
```

**❌ Mauvais** :
```yaml
labels:
  application-name-for-identification: nginx-web-server
  deployment-environment-type: production-environment
```

### 4. Toujours utiliser des labels pour les Deployments

**✅ Bon** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend        # Labels du Deployment
spec:
  selector:
    matchLabels:
      app: frontend      # Doit correspondre aux Pods
  template:
    metadata:
      labels:
        app: frontend    # Labels des Pods
```

**❌ Mauvais (labels incohérents)** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  selector:
    matchLabels:
      app: web-app       # ❌ Ne correspond pas !
  template:
    metadata:
      labels:
        app: web-app
```

### 5. Ne pas abuser des labels

**Trop de labels** → Complexité inutile

**✅ Bon (labels essentiels)** :
```yaml
labels:
  app: nginx
  environment: production
  version: v1.25
  team: platform
```

**❌ Trop (surcharge)** :
```yaml
labels:
  app: nginx
  environment: production
  version: v1.25
  team: platform
  subteam: infrastructure
  cost-center: 12345
  project: project-alpha
  billing-code: ABC-123
  created-by: john
  approved-by: jane
  ticket-number: JIRA-1234
  region: europe
  datacenter: dc1
```

Si vous avez trop d'informations, utilisez des **Annotations** plutôt que des Labels.

### 6. Labels des Pods doivent correspondre au Selector

**Règle d'or** : Les labels dans `template.metadata.labels` DOIVENT inclure tous les labels du `selector`.

**✅ Bon** :
```yaml
spec:
  selector:
    matchLabels:
      app: nginx        # ← Ces labels
      tier: frontend
  template:
    metadata:
      labels:
        app: nginx      # ← Doivent être présents ici
        tier: frontend
        version: v1.25  # ← Labels supplémentaires OK
```

**❌ Mauvais** :
```yaml
spec:
  selector:
    matchLabels:
      app: nginx
      tier: frontend    # ← Ce label
  template:
    metadata:
      labels:
        app: nginx      # ← Manque tier: frontend !
```

### 7. Utiliser des labels pour l'organisation multi-environnements

```yaml
# dev-namespace.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    environment: dev
---
# production-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
```

Ensuite :
```bash
# Lister tous les Namespaces de production
microk8s kubectl get namespaces -l environment=production
```

### 8. Documenter vos conventions de labels

Créez un document partagé avec votre équipe :

```markdown
# Convention de Labels

## Labels obligatoires
- `app`: Nom de l'application (ex: nginx, backend-api)
- `environment`: Environnement (dev, staging, production)
- `version`: Version sémantique (ex: v1.2.3)

## Labels optionnels
- `component`: Composant (frontend, backend, database)
- `team`: Équipe responsable (platform, frontend, backend)
- `tier`: Niveau architectural (web, api, data)

## Exemples
```yaml
labels:
  app: ecommerce-api
  environment: production
  version: v2.1.0
  component: backend
  team: backend
  tier: api
```
```

## Commandes utiles

### Gestion des labels

```bash
# Ajouter un label
microk8s kubectl label pod mon-pod app=nginx

# Ajouter plusieurs labels
microk8s kubectl label pod mon-pod app=nginx tier=frontend version=v1.25

# Modifier un label
microk8s kubectl label pod mon-pod version=v1.26 --overwrite

# Supprimer un label
microk8s kubectl label pod mon-pod version-

# Voir tous les labels d'une ressource
microk8s kubectl get pod mon-pod --show-labels

# Voir des labels spécifiques en colonnes
microk8s kubectl get pods -L app,version,environment
```

### Filtrage avec selectors

```bash
# Sélection simple
microk8s kubectl get pods -l app=nginx

# Sélection multiple (ET logique)
microk8s kubectl get pods -l app=nginx,environment=production

# Différent de
microk8s kubectl get pods -l environment!=dev

# In (OU logique)
microk8s kubectl get pods -l 'environment in (production,staging)'

# NotIn
microk8s kubectl get pods -l 'tier notin (cache,legacy)'

# Existe
microk8s kubectl get pods -l beta-feature

# N'existe pas
microk8s kubectl get pods -l '!deprecated'

# Combinaisons complexes
microk8s kubectl get pods -l 'app=nginx,environment in (prod,staging),tier!=cache'
```

### Opérations sur plusieurs ressources

```bash
# Supprimer tous les Pods de dev
microk8s kubectl delete pods -l environment=dev

# Redémarrer tous les Deployments de production
microk8s kubectl rollout restart deployment -l environment=production

# Scaler tous les Deployments frontend
microk8s kubectl scale deployment -l app=frontend --replicas=5

# Voir les logs de tous les Pods nginx
microk8s kubectl logs -l app=nginx --all-containers=true
```

## Résumé des points clés

- Les **Labels** sont des paires clé-valeur pour identifier et organiser les ressources
- Les **Selectors** permettent de sélectionner des ressources en fonction de leurs labels
- Deux types de selectors : **Equality-based** (=, !=) et **Set-based** (in, notin, exists)
- Les Labels sont utilisés partout : Services → Pods, Deployments → Pods, etc.
- Les **labels recommandés** par Kubernetes améliorent la standardisation
- Les **Labels** sont pour l'identification et la sélection
- Les **Annotations** sont pour les métadonnées et la documentation
- Utilisez des **conventions cohérentes** dans tout le cluster
- Les labels du `selector` doivent correspondre aux labels des Pods

## Prochaines étapes

Maintenant que vous maîtrisez les Labels et Selectors, vous êtes prêt à découvrir :
- **Les ConfigMaps** : comment gérer la configuration de vos applications de manière flexible
- **Les Secrets** : comment stocker et utiliser des données sensibles de manière sécurisée
- **Les Volumes** : comment persister les données au-delà du cycle de vie des Pods

Les Labels et Selectors sont le système nerveux de Kubernetes. Ils permettent de créer des architectures flexibles, évolutives et faciles à gérer !

⏭️ [ConfigMaps](/03-concepts-kubernetes-essentiels/06-configmaps.md)
