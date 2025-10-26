🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 25.4 Multi-cluster management

## Introduction : Un cluster ne suffit plus ?

Jusqu'à présent, nous avons travaillé avec **un seul cluster Kubernetes**. Pour beaucoup d'usages, c'est largement suffisant. Mais à mesure que vos besoins évoluent, vous pourriez vous retrouver à gérer **plusieurs clusters** simultanément.

### Pourquoi avoir plusieurs clusters ?

Imaginez ces scénarios :

**Scénario 1 : Environnements séparés**
```
Cluster DEV      → Développement et tests
Cluster STAGING  → Validation avant production
Cluster PROD     → Production
```

**Scénario 2 : Multi-région**
```
Cluster EU-WEST     → Utilisateurs européens (latence faible)
Cluster US-EAST     → Utilisateurs américains (latence faible)
Cluster ASIA-PACIFIC → Utilisateurs asiatiques (latence faible)
```

**Scénario 3 : Isolation de sécurité**
```
Cluster PUBLIC  → Applications web publiques
Cluster PRIVATE → Services internes et bases de données
Cluster PCI     → Applications manipulant des cartes de crédit
```

**Scénario 4 : Fournisseurs multiples**
```
Cluster AWS      → Infrastructure Amazon
Cluster Azure    → Infrastructure Microsoft
Cluster On-Prem  → Datacenter local
```

### Le défi du multi-cluster

Gérer plusieurs clusters pose des problèmes :

**Sans gestion centralisée** :
```
Vous devez :
- Vous connecter à chaque cluster individuellement
- Déployer la même application 3 fois (dev, staging, prod)
- Maintenir les configurations synchronisées manuellement
- Surveiller plusieurs dashboards séparés
- Gérer les secrets dans chaque cluster
- Appliquer les politiques de sécurité partout
```

**Avec gestion multi-cluster** :
```
Depuis un point central :
- Vue unifiée de tous vos clusters
- Déploiement simultané sur plusieurs clusters
- Synchronisation automatique des configurations
- Monitoring centralisé
- Gestion centralisée des secrets
- Application automatique des politiques
```

## Concepts fondamentaux

### 1. Hub and Spoke (moyeu et rayons)

Le modèle le plus courant pour gérer plusieurs clusters :

```
                  ┌─────────────────┐
                  │  Hub Cluster    │
                  │  (Management)   │
                  └────────┬────────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
    ┌────▼────┐       ┌────▼────┐       ┌────▼────┐
    │ Cluster │       │ Cluster │       │ Cluster │
    │  DEV    │       │ STAGING │       │  PROD   │
    └─────────┘       └─────────┘       └─────────┘
     (Spoke)           (Spoke)           (Spoke)
```

- **Hub** : Cluster central de management, ne fait pas tourner vos applications
- **Spokes** : Clusters qui exécutent vos workloads

### 2. Fédération vs Management centralisé

#### Kubernetes Federation (KubeFed)

La fédération permet de créer des ressources qui sont **automatiquement répliquées** sur plusieurs clusters :

```yaml
# Une seule définition
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: mon-app
spec:
  placement:
    clusters:
    - name: cluster-eu
    - name: cluster-us
  template:
    # Définition du Deployment
```

**Résultat** : Le Deployment est créé automatiquement dans cluster-eu ET cluster-us.

#### Management centralisé

Vous gérez plusieurs clusters mais déployez explicitement sur chacun :

```bash
# Déployer sur cluster 1
kubectl --context=cluster-dev apply -f app.yaml

# Déployer sur cluster 2
kubectl --context=cluster-staging apply -f app.yaml
```

### 3. Contextes kubectl

La commande `kubectl` peut gérer plusieurs clusters via les **contextes** :

```bash
# Voir tous les contextes configurés
kubectl config get-contexts

# Changer de contexte
kubectl config use-context cluster-prod

# Utiliser un contexte spécifique pour une commande
kubectl --context=cluster-dev get pods
```

Fichier kubeconfig avec plusieurs clusters :
```yaml
apiVersion: v1
kind: Config
clusters:
- name: cluster-dev
  cluster:
    server: https://dev.example.com:6443
- name: cluster-prod
  cluster:
    server: https://prod.example.com:6443
contexts:
- name: dev-context
  context:
    cluster: cluster-dev
    user: dev-user
- name: prod-context
  context:
    cluster: cluster-prod
    user: prod-user
current-context: dev-context
```

### 4. Service Mesh multi-cluster

Les Service Mesh comme Istio peuvent étendre le réseau sur plusieurs clusters :

```
Cluster A                          Cluster B
┌─────────────────┐                ┌─────────────────┐
│ Service Frontend│───────────────▶│ Service Backend │
│ (Pod Paris)     │   mTLS + LB    │ (Pod Londres)   │
└─────────────────┘                └─────────────────┘
```

Le frontend dans le cluster A peut appeler le backend dans le cluster B comme s'ils étaient dans le même cluster !

## Solutions de Multi-cluster Management

### 1. Rancher : La plateforme tout-en-un

**Rancher** est une plateforme complète de management Kubernetes qui excelle dans la gestion multi-cluster.

#### Fonctionnalités principales

- **Interface web intuitive** : Gérez tous vos clusters depuis un navigateur
- **Import de clusters** : Connectez n'importe quel cluster Kubernetes existant
- **Provisioning** : Créez de nouveaux clusters sur différents cloud providers
- **RBAC centralisé** : Gérez les accès à tous vos clusters depuis un point central
- **Catalogue d'applications** : Installez des apps sur plusieurs clusters en un clic
- **Monitoring intégré** : Prometheus et Grafana pour tous vos clusters

#### Architecture Rancher

```
┌──────────────────────────────────────────┐
│         Rancher Server (Hub)             │
│  Interface Web + API + Authentification  │
└──────────────────┬───────────────────────┘
                   │
     ┌─────────────┼─────────────┐
     │             │             │
┌────▼────┐   ┌────▼────┐   ┌────▼────┐
│ Cluster │   │ Cluster │   │ Cluster │
│   1     │   │   2     │   │   3     │
│ (Agent) │   │ (Agent) │   │ (Agent) │
└─────────┘   └─────────┘   └─────────┘
```

Chaque cluster a un **agent Rancher** qui communique avec le serveur central.

#### Installation de Rancher

```bash
# Sur votre cluster de management (Hub)
# Prérequis : Helm installé

# Ajouter le repo Helm Rancher
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update

# Créer le namespace
kubectl create namespace cattle-system

# Installer Cert-Manager (requis pour Rancher)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Installer Rancher
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.example.com \
  --set bootstrapPassword=admin \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=admin@example.com

# Attendre que Rancher soit prêt
kubectl -n cattle-system rollout status deploy/rancher
```

Accédez ensuite à `https://rancher.example.com` et connectez-vous.

#### Importer un cluster dans Rancher

Depuis l'interface Rancher :
1. Cliquez sur "Add Cluster"
2. Choisissez "Import an existing cluster"
3. Donnez un nom au cluster
4. Copiez la commande kubectl générée
5. Exécutez cette commande sur le cluster à importer

```bash
# Exemple de commande générée par Rancher
kubectl apply -f https://rancher.example.com/v3/import/xxxxx.yaml
```

Le cluster apparaît maintenant dans Rancher !

#### Cas d'usage Rancher

**Déploiement multi-cluster** :
```
1. Créer une application dans le catalogue
2. Sélectionner les clusters cibles (dev, staging, prod)
3. Cliquer sur "Deploy"
→ L'application est déployée simultanément sur tous les clusters
```

**Gestion centralisée des utilisateurs** :
```
1. Créer un utilisateur dans Rancher
2. Lui donner accès au cluster-dev (role: developer)
3. Lui donner accès au cluster-prod (role: view-only)
→ L'utilisateur peut switcher entre clusters dans l'interface
```

### 2. KubeFed : Kubernetes Federation

**KubeFed** est le projet officiel de fédération Kubernetes. Il permet de gérer des ressources qui existent dans plusieurs clusters simultanément.

#### Concepts KubeFed

**Federated Types** : Des versions "fédérées" des ressources Kubernetes standard.

```yaml
# Au lieu de créer un Deployment normal
apiVersion: apps/v1
kind: Deployment

# Vous créez un FederatedDeployment
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: mon-app
  namespace: default
spec:
  # Où déployer
  placement:
    clusters:
    - name: cluster-paris
    - name: cluster-londres

  # Template du Deployment
  template:
    metadata:
      labels:
        app: mon-app
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: mon-app
      template:
        spec:
          containers:
          - name: nginx
            image: nginx:latest

  # Overrides spécifiques par cluster
  overrides:
  - clusterName: cluster-paris
    clusterOverrides:
    - path: "/spec/replicas"
      value: 5  # 5 replicas à Paris
  - clusterName: cluster-londres
    clusterOverrides:
    - path: "/spec/replicas"
      value: 2  # 2 replicas à Londres
```

**Résultat** :
- Un Deployment avec 5 replicas est créé dans cluster-paris
- Un Deployment avec 2 replicas est créé dans cluster-londres

#### Installation de KubeFed

```bash
# Sur votre cluster Hub
helm repo add kubefed-charts https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts
helm repo update

helm install kubefed kubefed-charts/kubefed \
  --namespace kube-federation-system \
  --create-namespace

# Installer le CLI kubefedctl
curl -LO https://github.com/kubernetes-sigs/kubefed/releases/download/v0.10.0/kubefedctl-0.10.0-linux-amd64.tgz
tar -xzf kubefedctl-0.10.0-linux-amd64.tgz
sudo mv kubefedctl /usr/local/bin/
```

#### Joindre des clusters à la fédération

```bash
# Joindre cluster-paris
kubefedctl join cluster-paris \
  --cluster-context cluster-paris-context \
  --host-cluster-context hub-context

# Joindre cluster-londres
kubefedctl join cluster-londres \
  --cluster-context cluster-londres-context \
  --host-cluster-context hub-context

# Vérifier les clusters fédérés
kubectl get kubefedclusters -n kube-federation-system
```

#### Activer la fédération pour des types de ressources

```bash
# Fédérer les Deployments
kubefedctl enable deployments

# Fédérer les Services
kubefedctl enable services

# Fédérer les Secrets
kubefedctl enable secrets

# Voir tous les types fédérés activés
kubectl get federatedtypeconfigs -n kube-federation-system
```

#### Avantages et inconvénients de KubeFed

**✅ Avantages** :
- Projet officiel Kubernetes
- Synchronisation automatique
- Overrides par cluster puissants
- Placement intelligent avec labels

**❌ Inconvénients** :
- Complexité élevée
- Overhead de gestion
- Debugging difficile
- Moins mature que d'autres solutions

### 3. GitOps multi-cluster avec ArgoCD

**ArgoCD** peut gérer plusieurs clusters depuis un point central, suivant le paradigme GitOps.

#### Architecture ArgoCD multi-cluster

```
┌────────────────────────────────┐
│     Git Repository             │
│  ├── cluster-dev/              │
│  │   └── app.yaml              │
│  ├── cluster-staging/          │
│  │   └── app.yaml              │
│  └── cluster-prod/             │
│      └── app.yaml              │
└─────────────┬──────────────────┘
              │
              │ Synchronise
              ↓
┌─────────────────────────────────┐
│   ArgoCD (dans le Hub Cluster)  │
└─────────────┬───────────────────┘
              │
     ┌────────┼──────────┐
     │        │          │
┌────▼───┐ ┌──▼─────┐ ┌──▼──────┐
│ DEV    │ │STAGING │ │ PROD    │
└────────┘ └────────┘ └─────────┘
```

#### Ajouter des clusters dans ArgoCD

```bash
# Lister les contextes kubectl disponibles
kubectl config get-contexts

# Ajouter un cluster dans ArgoCD
argocd cluster add cluster-dev-context
argocd cluster add cluster-staging-context
argocd cluster add cluster-prod-context

# Lister les clusters dans ArgoCD
argocd cluster list
```

#### Créer des Applications pour différents clusters

```yaml
# Application pour le cluster DEV
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mon-app-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/mon-org/mon-repo
    targetRevision: HEAD
    path: cluster-dev
  destination:
    server: https://cluster-dev.example.com:6443
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
---
# Application pour le cluster PROD
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mon-app-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/mon-org/mon-repo
    targetRevision: HEAD
    path: cluster-prod
  destination:
    server: https://cluster-prod.example.com:6443
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

#### ApplicationSet : Gérer plusieurs Applications

Pour éviter de créer manuellement une Application par cluster, utilisez **ApplicationSet** :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: mon-app-multi-cluster
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - cluster: dev
        url: https://cluster-dev.example.com:6443
        replicas: "2"
      - cluster: staging
        url: https://cluster-staging.example.com:6443
        replicas: "3"
      - cluster: prod
        url: https://cluster-prod.example.com:6443
        replicas: "5"

  template:
    metadata:
      name: 'mon-app-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/mon-org/mon-repo
        targetRevision: HEAD
        path: 'clusters/{{cluster}}'
      destination:
        server: '{{url}}'
        namespace: default
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

Un seul ApplicationSet crée automatiquement 3 Applications (une par cluster) !

### 4. Flux CD : Alternative GitOps

**Flux** est une autre solution GitOps qui supporte naturellement le multi-cluster.

#### Architecture Flux multi-cluster

Flux peut être installé soit :
- **Dans chaque cluster** (approche distribuée)
- **Dans un cluster hub** qui gère les autres (approche centralisée)

```bash
# Installer Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Bootstrap Flux dans le cluster Hub
flux bootstrap github \
  --owner=mon-org \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/hub

# Ajouter un cluster distant (depuis le Hub)
flux create secret git cluster-dev-deploy-key \
  --url=ssh://git@github.com/mon-org/cluster-dev-config

flux create kustomization cluster-dev \
  --source=GitRepository/cluster-dev-config \
  --path="./clusters/dev" \
  --prune=true \
  --interval=5m \
  --kubeconfig=/path/to/dev-kubeconfig
```

### 5. Lens : IDE Kubernetes multi-cluster

**Lens** (aussi appelé OpenLens pour la version open-source) est un IDE de bureau pour Kubernetes qui simplifie la gestion multi-cluster.

#### Fonctionnalités Lens

- **Vue unifiée** : Tous vos clusters dans une seule interface
- **Switch rapide** : Changez de cluster en un clic
- **Terminal intégré** : kubectl directement dans l'interface
- **Observabilité** : Métriques, logs, événements en temps réel
- **Extensions** : Ajoutez des fonctionnalités (Helm, ArgoCD, etc.)

#### Installation

```bash
# Sur Ubuntu/Debian
wget https://api.k8slens.dev/binaries/Lens-latest.amd64.deb
sudo dpkg -i Lens-latest.amd64.deb

# Ou télécharger depuis https://k8slens.dev/
```

Une fois installé :
1. Lens détecte automatiquement tous les contextes dans votre `~/.kube/config`
2. Vous pouvez ajouter des clusters manuellement
3. Chaque cluster apparaît dans le panneau de gauche

### 6. K9s : TUI multi-cluster

**K9s** est un terminal UI (TUI) qui supporte plusieurs clusters.

```bash
# Installer K9s
curl -sS https://webinstall.dev/k9s | bash

# Lancer K9s
k9s

# Dans K9s, changer de cluster :
# Appuyez sur ':ctx' puis sélectionnez le contexte
```

## Stratégies de déploiement multi-cluster

### 1. Promotion progressive

Déployer d'abord en DEV, puis STAGING, puis PROD :

```
Git commit → CI/CD Pipeline
    ↓
Deploy to DEV → Tests automatisés
    ↓ ✅ Tests OK
Deploy to STAGING → Tests d'intégration
    ↓ ✅ Validation manuelle
Deploy to PROD → Monitoring
```

Avec ArgoCD :
```yaml
# Branche dev → cluster-dev
source:
  targetRevision: dev

# Branche main → cluster-prod
source:
  targetRevision: main
```

### 2. Blue-Green multi-cluster

Avoir deux clusters de production identiques :

```
Cluster BLUE (actif)  → Reçoit 100% du trafic
Cluster GREEN (idle)  → Nouvelle version déployée

Après validation :
Cluster BLUE (idle)   → Ancienne version
Cluster GREEN (actif) → Reçoit 100% du trafic
```

**Avantage** : Rollback instantané en basculant sur BLUE.

### 3. Canary multi-cluster

Déployer progressivement sur plusieurs clusters :

```
Semaine 1 : 10% des clusters (cluster-test)
Semaine 2 : 30% des clusters (clusters EU)
Semaine 3 : 100% des clusters (tous)
```

### 4. Déploiement géographique

Chaque cluster dessert une région géographique :

```yaml
# Cluster EU
apiVersion: v1
kind: Service
metadata:
  name: api-frontend
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  selector:
    app: frontend
    region: eu
```

Utilisez un **Global Load Balancer** (Cloudflare, AWS Route 53, etc.) pour diriger les utilisateurs vers le cluster le plus proche.

## Gestion des données multi-cluster

### Le défi de la cohérence des données

**Problème** : Vous avez une base de données dans chaque cluster. Comment garder les données synchronisées ?

#### Option 1 : Base de données centralisée

Tous les clusters accèdent à une base de données centrale :

```
Cluster DEV    ──┐
Cluster STAGING──┼──→ [Database Centrale]
Cluster PROD   ──┘
```

**✅ Avantages** :
- Cohérence garantie
- Une seule source de vérité

**❌ Inconvénients** :
- Point de défaillance unique
- Latence pour les clusters distants
- Dépendance réseau

#### Option 2 : Réplication de bases de données

Chaque cluster a sa base de données, avec réplication :

```
[DB Cluster 1] ←→ [DB Cluster 2] ←→ [DB Cluster 3]
     ↑                                      ↑
Réplication bidirectionnelle
```

Solutions :
- **PostgreSQL** : Streaming replication, logical replication
- **MySQL** : Group replication, Galera Cluster
- **MongoDB** : Replica sets, sharding

**✅ Avantages** :
- Haute disponibilité
- Faible latence locale
- Résilience

**❌ Inconvénients** :
- Complexité accrue
- Risque de conflits
- Overhead de réplication

#### Option 3 : Base de données distribuée

Utiliser une base de données conçue pour être distribuée :

- **CockroachDB** : PostgreSQL compatible, multi-région native
- **Cassandra** : NoSQL distribuée
- **TiDB** : MySQL compatible, distribué
- **YugabyteDB** : PostgreSQL compatible, distribué

```yaml
# Exemple CockroachDB multi-cluster
apiVersion: crdb.cockroachlabs.com/v1alpha1
kind: CrdbCluster
metadata:
  name: cockroachdb
spec:
  nodes: 9
  dataStore:
    pvc:
      spec:
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 100Gi
  topology:
    zones:
    - name: eu-west-1a
      nodes: 3
    - name: us-east-1a
      nodes: 3
    - name: asia-east-1a
      nodes: 3
```

### Gestion des Secrets multi-cluster

#### Option 1 : Secret management externe

Utiliser un système centralisé :

```
┌────────────────────┐
│  HashiCorp Vault   │ ← Source unique de vérité
│  (External)        │
└─────────┬──────────┘
          │
    ┌─────┼─────┐
    │     │     │
┌───▼─┐ ┌─▼──┐ ┌▼───┐
│ DEV │ │STG │ │PRD │
└─────┘ └────┘ └────┘
```

Chaque cluster récupère les secrets depuis Vault :

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-password
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "myapp"
    vault.hashicorp.com/agent-inject-secret-password: "secret/data/database/password"
```

#### Option 2 : Sealed Secrets

Chiffrer les secrets dans Git, déchiffrer dans chaque cluster :

```bash
# Installer Sealed Secrets dans chaque cluster
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Créer un SealedSecret
kubectl create secret generic mysecret --dry-run=client --from-literal=password=secret123 -o yaml | \
  kubeseal -o yaml > mysealedsecret.yaml

# Commit dans Git
git add mysealedsecret.yaml
git commit -m "Add sealed secret"

# Déployer sur tous les clusters (via ArgoCD par exemple)
# Chaque cluster déchiffre automatiquement avec sa clé privée
```

#### Option 3 : External Secrets Operator

Synchroniser automatiquement depuis des secret stores externes :

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "my-app"
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: database-credentials
  data:
  - secretKey: password
    remoteRef:
      key: database
      property: password
```

Déployez les mêmes manifestes sur tous les clusters → ils récupèrent tous leurs secrets depuis Vault.

## Monitoring multi-cluster

### Prometheus Fédération

Agréger les métriques de plusieurs clusters dans un Prometheus central :

```
┌──────────────────────────────────┐
│   Prometheus Global (Hub)        │
│   Agrège toutes les métriques    │
└────────┬─────────────────────────┘
         │ Federate
    ┌────┼────┐
    │    │    │
┌───▼┐ ┌─▼─┐ ┌▼──┐
│P-1 │ │P-2│ │P-3│  Prometheus par cluster
└────┘ └───┘ └───┘
  ↑     ↑     ↑
[C1]  [C2]  [C3]    Clusters
```

Configuration du Prometheus global :

```yaml
scrape_configs:
- job_name: 'federate-cluster-1'
  scrape_interval: 30s
  honor_labels: true
  metrics_path: '/federate'
  params:
    'match[]':
    - '{job="kubernetes-nodes"}'
    - '{job="kubernetes-pods"}'
  static_configs:
  - targets:
    - 'prometheus-cluster-1.example.com:9090'
    labels:
      cluster: 'cluster-1'

- job_name: 'federate-cluster-2'
  scrape_interval: 30s
  honor_labels: true
  metrics_path: '/federate'
  params:
    'match[]':
    - '{job="kubernetes-nodes"}'
    - '{job="kubernetes-pods"}'
  static_configs:
  - targets:
    - 'prometheus-cluster-2.example.com:9090'
    labels:
      cluster: 'cluster-2'
```

### Thanos : Prometheus à grande échelle

**Thanos** étend Prometheus pour le multi-cluster et le stockage long terme :

```
[Prometheus Cluster-1] ──→ [Thanos Sidecar] ──┐
[Prometheus Cluster-2] ──→ [Thanos Sidecar] ──┼──→ [Object Storage S3]
[Prometheus Cluster-3] ──→ [Thanos Sidecar] ──┘
                                                       ↑
                                                       │
                                            [Thanos Query] ← Interface unifiée
                                                       │
                                                [Grafana]
```

**Avantages** :
- Vue unifiée de toutes les métriques
- Stockage illimité (S3, GCS, etc.)
- Déduplication automatique
- Downsampling pour l'historique

### Grafana avec plusieurs datasources

Configurer Grafana pour interroger plusieurs Prometheus :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
    - name: Prometheus-Cluster-1
      type: prometheus
      url: http://prometheus-cluster-1:9090
      jsonData:
        timeInterval: 30s
    - name: Prometheus-Cluster-2
      type: prometheus
      url: http://prometheus-cluster-2:9090
      jsonData:
        timeInterval: 30s
    - name: Prometheus-Cluster-3
      type: prometheus
      url: http://prometheus-cluster-3:9090
      jsonData:
        timeInterval: 30s
```

Dans vos dashboards, utilisez des variables pour switcher entre clusters :

```
Variable : $cluster
Query : label_values(cluster)

Panel query : up{cluster="$cluster"}
```

## Réseau multi-cluster

### Service Mesh multi-cluster

Les Service Mesh comme **Istio** et **Linkerd** supportent le multi-cluster.

#### Istio Multi-Primary

Chaque cluster a son propre control plane, mais ils partagent la même mesh :

```
Cluster 1 (Primary)          Cluster 2 (Primary)
┌────────────────┐          ┌────────────────┐
│ Istio Control  │←────────→│ Istio Control  │
│ Plane          │  Trust   │ Plane          │
├────────────────┤          ├────────────────┤
│ Service A      │          │ Service B      │
│ (Pod)          │←────────→│ (Pod)          │
└────────────────┘  mTLS    └────────────────┘
```

Service A dans Cluster 1 peut appeler Service B dans Cluster 2 directement !

#### Configuration Istio multi-cluster

```bash
# Sur chaque cluster, installer Istio avec multi-cluster enabled
istioctl install --set values.global.meshID=mesh1 \
  --set values.global.multiCluster.clusterName=cluster-1 \
  --set values.global.network=network1

# Permettre aux clusters de se découvrir
istioctl x create-remote-secret \
  --context=cluster-1 \
  --name=cluster-1 | \
  kubectl apply -f - --context=cluster-2
```

### Cilium Cluster Mesh

**Cilium** permet de connecter plusieurs clusters au niveau réseau :

```
┌────────────────────┐         ┌────────────────────┐
│   Cluster Paris    │         │  Cluster Londres   │
│                    │         │                    │
│ Pod-A              │         │ Pod-B              │
│ 10.1.1.5           │←───────→│ 10.2.1.8           │
│                    │ Cluster │                    │
│                    │  Mesh   │                    │
└────────────────────┘         └────────────────────┘
```

Les Pods peuvent communiquer directement entre clusters avec leur IP.

### VPN / WireGuard

Pour des besoins simples, vous pouvez utiliser un VPN :

```
Cluster 1 (10.1.0.0/16) ←─── WireGuard VPN ───→ Cluster 2 (10.2.0.0/16)
```

Tous les Pods peuvent se joindre directement.

## Multi-cluster management avec MicroK8s

### Scénario : Lab multi-environnement

Vous avez 3 machines (ou VMs) :
- Machine 1 : Cluster DEV
- Machine 2 : Cluster STAGING
- Machine 3 : Cluster PROD

#### Configuration des contextes

```bash
# Sur votre poste de travail, récupérer les kubeconfig

# Cluster DEV
microk8s config > dev-kubeconfig

# Cluster STAGING
microk8s config > staging-kubeconfig

# Cluster PROD
microk8s config > prod-kubeconfig

# Fusionner dans un seul fichier ~/.kube/config
KUBECONFIG=dev-kubeconfig:staging-kubeconfig:prod-kubeconfig \
  kubectl config view --flatten > ~/.kube/config

# Renommer les contextes pour clarté
kubectl config rename-context microk8s dev
kubectl config rename-context microk8s staging
kubectl config rename-context microk8s prod

# Lister les contextes
kubectl config get-contexts

# Utiliser un contexte
kubectl config use-context dev
```

#### Option 1 : ArgoCD pour GitOps

1. Installer ArgoCD sur un des clusters (ou une 4ème machine hub)
2. Ajouter les 3 clusters dans ArgoCD
3. Créer une structure Git :

```
mon-repo/
├── apps/
│   ├── dev/
│   │   └── my-app.yaml
│   ├── staging/
│   │   └── my-app.yaml
│   └── prod/
│       └── my-app.yaml
└── argocd/
    └── applications.yaml
```

4. Créer les Applications ArgoCD pointant vers chaque environnement

#### Option 2 : Scripts de déploiement

Pour un lab simple, des scripts peuvent suffire :

```bash
#!/bin/bash
# deploy-all.sh

ENVIRONMENTS=("dev" "staging" "prod")

for ENV in "${ENVIRONMENTS[@]}"; do
  echo "Déploiement sur $ENV..."
  kubectl --context=$ENV apply -f apps/$ENV/

  # Attendre que le déploiement soit prêt
  kubectl --context=$ENV rollout status deployment/my-app

  echo "✅ $ENV déployé avec succès"
done
```

#### Option 3 : Rancher pour une interface graphique

Si vous préférez une GUI :

1. Installer Rancher sur une machine dédiée (ou le cluster dev)
2. Importer les 3 clusters MicroK8s dans Rancher
3. Gérer tout depuis l'interface web Rancher

### Considérations de ressources

Le multi-cluster consomme plus de ressources :

```
1 Cluster :
- 2 GB RAM
- 2 CPU cores

3 Clusters + Hub (Rancher ou ArgoCD) :
- 8-10 GB RAM total
- 6-8 CPU cores total
```

Pour un lab MicroK8s, soyez conscient des limites de votre matériel.

## Défis et bonnes pratiques

### Défis du multi-cluster

#### 1. Complexité opérationnelle

Plus de clusters = plus de complexité :
- Plus de monitoring
- Plus de mises à jour
- Plus de dépannage
- Plus de configuration réseau

**Mitigation** :
- Automatisez au maximum
- Utilisez GitOps
- Documentez vos procédures

#### 2. Cohérence des configurations

Comment s'assurer que tous les clusters sont configurés identiquement ?

**Mitigation** :
- Infrastructure as Code (Terraform, Pulumi)
- Configuration centralisée (ArgoCD, Flux)
- Templates et politiques (Kyverno, OPA)

#### 3. Gestion des versions

Tous les clusters doivent-ils être sur la même version de Kubernetes ?

**Recommandation** :
- DEV : N (dernière version)
- STAGING : N-1
- PROD : N-1 (stabilité)

#### 4. Coûts

Plus de clusters = plus de coûts (surtout dans le cloud).

**Optimisation** :
- Cluster autoscaling
- Scale-to-zero pour les environnements non-prod
- Cluster partagés avec isolation par namespace (alternative)

### Bonnes pratiques

#### 1. Nomenclature cohérente

Nommez vos clusters de manière logique :

```
Mauvais :
- cluster-1
- k8s-test
- production

Bon :
- app-prod-eu-west-1
- app-staging-us-east-1
- app-dev-local
```

#### 2. Tagging et labels

Taggez tout :

```yaml
metadata:
  labels:
    environment: production
    region: eu-west-1
    team: backend
    cost-center: engineering
```

#### 3. Politiques uniformes

Appliquez les mêmes politiques de sécurité partout :

```yaml
# NetworkPolicy identique sur tous les clusters
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

#### 4. Disaster Recovery

Ayez un plan :
- Backups réguliers (Velero)
- Procédures de restauration documentées
- Tests de DR réguliers

#### 5. Observabilité centralisée

Ne fragmentez pas votre monitoring :
- Logs centralisés (ELK, Loki)
- Métriques agrégées (Thanos, Cortex)
- Traces distribuées (Jaeger, Tempo)
- Dashboards unifiés (Grafana)

#### 6. Automatisation des tests

Testez la configuration multi-cluster :

```yaml
# test-connectivity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-connectivity
spec:
  containers:
  - name: curl
    image: curlimages/curl
    command:
    - sh
    - -c
    - |
      # Tester la connectivité vers les autres clusters
      curl -I https://api.cluster-2.example.com
      curl -I https://api.cluster-3.example.com
```

## Alternatives au multi-cluster

Avant de vous lancer dans le multi-cluster, considérez ces alternatives :

### 1. Multi-tenant sur un seul cluster

Utilisez des **namespaces** pour isoler les environnements :

```
Cluster unique
├── Namespace : dev
├── Namespace : staging
└── Namespace : prod
```

**Avantages** :
- ✅ Moins de complexité
- ✅ Moins de ressources
- ✅ Partage des ressources

**Inconvénients** :
- ❌ Isolation moins forte
- ❌ Risque de noisy neighbor
- ❌ Un point de défaillance

### 2. Environnements virtuels

Utilisez des outils comme **vCluster** pour créer des clusters virtuels :

```
Cluster physique
├── vCluster : dev    (cluster virtuel)
├── vCluster : staging (cluster virtuel)
└── vCluster : prod   (cluster virtuel)
```

**Avantages** :
- ✅ Isolation forte (chaque vCluster a son API server)
- ✅ Moins de ressources qu'un cluster complet
- ✅ Rapidité de provisioning

### 3. Hybrid : Quelques gros clusters

Au lieu de 10 petits clusters :

```
Avant :
- cluster-app-1
- cluster-app-2
- cluster-app-3
- cluster-service-1
- cluster-service-2
[...]

Après :
- cluster-production (multi-tenant)
- cluster-non-production (dev + staging)
```

## Conclusion

Le **multi-cluster management** est une étape naturelle quand vos besoins en Kubernetes évoluent. Que ce soit pour séparer les environnements, distribuer géographiquement, ou améliorer la résilience, de nombreux outils existent pour simplifier cette gestion.

### Points clés à retenir

1. **Plusieurs clusters** = isolation forte, mais plus de complexité
2. **Hub and Spoke** : Architecture de référence pour le multi-cluster
3. **GitOps** (ArgoCD/Flux) : Approche recommandée pour la cohérence
4. **Rancher** : Solution tout-en-un avec interface graphique
5. **KubeFed** : Fédération officielle mais complexe
6. **Service Mesh** : Pour la communication cross-cluster
7. **Monitoring centralisé** : Essentiel pour l'observabilité
8. **Secrets management** : Vault ou External Secrets Operator

### Recommandations pour MicroK8s

**Pour apprendre** :
- Commencez avec 2-3 clusters MicroK8s (dev, staging, prod)
- Utilisez des contextes kubectl pour switcher
- Expérimentez avec ArgoCD pour le GitOps
- Si vous avez les ressources, testez Rancher

**Pour un usage pratique** :
- Évaluez si vous avez vraiment besoin de multi-cluster
- Considérez les alternatives (namespaces, vCluster)
- Automatisez dès le début (scripts, GitOps)
- Documentez votre architecture

**Progression recommandée** :
1. Maîtriser un seul cluster d'abord
2. Ajouter un 2ème cluster et gérer avec des scripts
3. Implémenter GitOps avec ArgoCD
4. Si besoin, ajouter Rancher ou Lens pour la gestion

### Quand choisir le multi-cluster ?

**✅ OUI si** :
- Isolation stricte requise (sécurité, compliance)
- Multi-région pour latence faible
- Haute disponibilité critique
- Équipes multiples avec besoins différents
- Différents fournisseurs cloud

**❌ NON si** :
- Vous débutez avec Kubernetes
- Ressources limitées (lab personnel)
- Applications simples
- Un seul environnement suffit

### Prochaines étapes

Après avoir exploré le multi-cluster management, vous pourriez vous intéresser à :
- **25.1 Service Mesh** : Communication sécurisée inter-cluster
- **17.5 ArgoCD** : GitOps pour déploiement multi-cluster
- **21. Multi-Node et Haute Disponibilité** : Élargir un cluster

---

**Ressources complémentaires** :

- Kubernetes Multi-Cluster SIG : https://github.com/kubernetes/community/tree/master/sig-multicluster
- Rancher Documentation : https://rancher.com/docs/
- KubeFed User Guide : https://github.com/kubernetes-sigs/kubefed
- ArgoCD ApplicationSet : https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/
- Istio Multi-Cluster : https://istio.io/latest/docs/setup/install/multicluster/
- Cilium Cluster Mesh : https://docs.cilium.io/en/stable/gettingstarted/clustermesh/

⏭️ [Edge computing avec K8s](/25-aller-plus-loin/05-edge-computing-avec-k8s.md)
