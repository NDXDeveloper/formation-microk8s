🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 25.3 Operators et Custom Resources

## Introduction : Étendre Kubernetes au-delà de ses limites

Kubernetes propose des ressources natives comme les Pods, Deployments, Services, etc. Ces ressources couvrent la plupart des besoins, mais que faire quand vous voulez gérer des applications complexes avec leur propre logique spécifique ?

C'est là qu'interviennent les **Custom Resources** (ressources personnalisées) et les **Operators** (opérateurs). Ils permettent d'**étendre Kubernetes** pour qu'il puisse gérer n'importe quel type d'application, aussi complexe soit-elle.

### Une analogie simple

Imaginez Kubernetes comme un système d'exploitation :

- **Ressources natives** : Les applications préinstallées (calculatrice, bloc-notes, navigateur)
- **Custom Resources** : De nouveaux types de fichiers que vous définissez (`.monfichier`)
- **Operators** : Des applications que vous installez pour gérer ces nouveaux types de fichiers

Tout comme vous pouvez installer Adobe Photoshop pour gérer des fichiers `.psd`, vous pouvez installer un Operator pour gérer, par exemple, un cluster PostgreSQL dans Kubernetes.

## Custom Resource Definitions (CRD)

### Qu'est-ce qu'une CRD ?

Une **CRD (Custom Resource Definition)** est un moyen de créer vos propres types de ressources dans Kubernetes, en plus des types standards (Pod, Deployment, Service, etc.).

#### Ressources Kubernetes natives

Kubernetes comprend nativement ces ressources :

```yaml
apiVersion: v1
kind: Pod              # Type natif
---
apiVersion: apps/v1
kind: Deployment       # Type natif
---
apiVersion: v1
kind: Service          # Type natif
```

#### Votre propre ressource personnalisée

Avec une CRD, vous pouvez créer vos propres types :

```yaml
apiVersion: databases.example.com/v1
kind: PostgreSQLCluster    # Votre type personnalisé !
metadata:
  name: mon-cluster-postgres
spec:
  replicas: 3
  version: "15.2"
  storage: "100Gi"
```

### Pourquoi créer des Custom Resources ?

Les Custom Resources permettent de :

1. **Abstraire la complexité** : Cacher les détails d'implémentation derrière une interface simple
2. **Utiliser kubectl normalement** : `kubectl get postgresqlcluster`
3. **Déclarer l'état désiré** : Comme avec toutes les ressources Kubernetes
4. **Stocker dans etcd** : Vos ressources sont persistées comme les ressources natives
5. **Utiliser RBAC** : Contrôler les accès à vos ressources

### Anatomie d'une CRD

Voici un exemple de définition de CRD simplifié :

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresqlclusters.databases.example.com
spec:
  group: databases.example.com      # Groupe API
  names:
    kind: PostgreSQLCluster          # Nom du type
    plural: postgresqlclusters       # Nom pluriel
    singular: postgresqlcluster      # Nom singulier
    shortNames:
    - pgc                             # Alias court
  scope: Namespaced                   # Scope (Namespaced ou Cluster)
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              replicas:
                type: integer
                minimum: 1
                maximum: 10
              version:
                type: string
              storage:
                type: string
```

Une fois cette CRD créée, vous pouvez utiliser :

```bash
# Créer une instance de votre ressource
kubectl apply -f mon-cluster-postgres.yaml

# Lister vos ressources
kubectl get postgresqlclusters
kubectl get pgc  # Avec le shortName

# Décrire une ressource
kubectl describe postgresqlcluster mon-cluster-postgres

# Supprimer une ressource
kubectl delete postgresqlcluster mon-cluster-postgres
```

### Custom Resource vs CRD

Il est important de distinguer ces deux concepts :

- **CRD (Custom Resource Definition)** : La définition, le "schéma" de votre nouveau type de ressource
- **CR (Custom Resource)** : Une instance, un objet créé à partir de cette définition

```
CRD = Le moule à gâteau (la définition)
CR = Le gâteau (l'instance créée avec le moule)
```

Exemple :
```yaml
# Ceci est une CRD (le moule)
kind: CustomResourceDefinition
metadata:
  name: postgresqlclusters.databases.example.com

---

# Ceci est une CR (le gâteau)
kind: PostgreSQLCluster
metadata:
  name: mon-cluster-prod
```

## Operators : Donner vie aux Custom Resources

### Le problème sans Operator

Vous venez de créer une Custom Resource `PostgreSQLCluster`. Vous créez une instance :

```yaml
apiVersion: databases.example.com/v1
kind: PostgreSQLCluster
metadata:
  name: mon-cluster
spec:
  replicas: 3
  version: "15.2"
```

Vous faites `kubectl apply -f ...` et... rien ne se passe !

**Pourquoi ?** Parce que Kubernetes ne sait pas quoi faire avec cette ressource. Il l'a stockée dans etcd, c'est tout.

Il manque quelqu'un pour **surveiller** cette ressource et **agir** en conséquence. Ce quelqu'un, c'est l'**Operator**.

### Qu'est-ce qu'un Operator ?

Un **Operator** est une application qui tourne dans Kubernetes et qui :

1. **Surveille** vos Custom Resources (ou des ressources natives)
2. **Comprend** ce que vous voulez (l'état désiré)
3. **Effectue les actions** nécessaires pour atteindre cet état
4. **Surveille en continu** pour maintenir l'état désiré

```
Vous créez : PostgreSQLCluster (replicas: 3)
       ↓
Operator détecte : "Ah ! L'utilisateur veut un cluster PostgreSQL avec 3 réplicas"
       ↓
Operator agit :
  - Crée 3 Pods PostgreSQL
  - Crée un Service
  - Configure la réplication
  - Crée les volumes persistants
  - Configure les backups
       ↓
Operator surveille : "Tout va bien ? Un Pod est down ? Je le recrée !"
```

### L'Operator Pattern : Automatiser l'expertise humaine

Le concept d'Operator a été popularisé par CoreOS (maintenant Red Hat). L'idée est d'**encoder l'expertise humaine** dans du code.

**Exemple avec PostgreSQL** :

Un administrateur de base de données expérimenté sait :
- Comment installer PostgreSQL
- Comment configurer la réplication
- Comment faire un backup
- Comment restaurer depuis un backup
- Comment gérer une mise à jour de version
- Comment diagnostiquer et corriger des problèmes

Un **PostgreSQL Operator** encode toute cette expertise. Il sait faire tout ça automatiquement !

```
Sans Operator :
Vous : "Je veux PostgreSQL avec réplication"
Vous devez :
  1. Créer un StatefulSet
  2. Configurer les volumes
  3. Créer un Service headless
  4. Créer un Service read-only
  5. Écrire des scripts de configuration
  6. Configurer la réplication dans postgresql.conf
  7. Initialiser le cluster
  8. Créer un CronJob pour les backups
  → 50+ lignes de YAML complexe + scripts

Avec Operator :
Vous : "Je veux PostgreSQL avec réplication"
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: mon-cluster
spec:
  postgresVersion: 15
  instances:
    - replicas: 3
  backups:
    pgbackrest:
      repos:
      - name: repo1
        volume:
          volumeClaimSpec:
            accessModes:
            - ReadWriteOnce
            resources:
              requests:
                storage: 100Gi
→ L'Operator gère tout automatiquement !
```

## Les trois niveaux d'Operators

Selon la **Operator Maturity Model** de Red Hat, il existe plusieurs niveaux de sophistication :

### Niveau 1 : Installation Basique (Basic Install)

L'Operator peut :
- ✅ Installer l'application
- ✅ Configurer les ressources de base
- ❌ Ne gère pas les mises à jour
- ❌ Ne fait pas de backup

**Exemple** : Vous créez une CR, l'Operator crée les Pods et Services nécessaires.

### Niveau 2 : Mise à jour sans interruption (Seamless Upgrades)

L'Operator peut en plus :
- ✅ Mettre à jour l'application sans downtime
- ✅ Faire un rollback en cas d'échec
- ❌ Ne gère pas encore les backups

**Exemple** : Vous changez `version: "15.1"` en `version: "15.2"`, l'Operator met à jour progressivement.

### Niveau 3 : Cycle de vie complet (Full Lifecycle)

L'Operator gère :
- ✅ Les backups automatiques
- ✅ Les restaurations
- ✅ Le monitoring
- ✅ Les alertes

**Exemple** : L'Operator fait des backups réguliers et peut restaurer depuis un backup automatiquement.

### Niveau 4 : Observabilité profonde (Deep Insights)

L'Operator offre :
- ✅ Métriques détaillées de l'application
- ✅ Dashboards Grafana intégrés
- ✅ Alertes intelligentes
- ✅ Logs structurés

### Niveau 5 : Auto-pilotage (Auto-Pilot)

L'Operator peut :
- ✅ Détecter les anomalies
- ✅ S'auto-réparer
- ✅ Optimiser automatiquement (autoscaling, tuning)
- ✅ Prédire les problèmes

**Exemple** : L'Operator détecte que les performances se dégradent et ajuste automatiquement la configuration.

## Comment fonctionne un Operator ?

### Architecture d'un Operator

```
┌─────────────────────────────────────────┐
│        Kubernetes API Server            │
│  (stocke les Custom Resources dans etcd)│
└──────────────┬──────────────────────────┘
               │
               │ Watch (surveille)
               ↓
┌─────────────────────────────────────────┐
│           Operator (Pod)                │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │  Controller / Reconciliation     │   │
│  │  Loop (boucle de réconciliation) │   │
│  └──────────────────────────────────┘   │
│                                         │
│  Logique :                              │
│  1. Lire l'état actuel                  │
│  2. Comparer avec l'état désiré         │
│  3. Calculer les actions nécessaires    │
│  4. Exécuter ces actions                │
│  5. Répéter continuellement             │
└──────────────┬──────────────────────────┘
               │
               │ Gère les ressources
               ↓
┌─────────────────────────────────────────┐
│    Ressources Kubernetes créées         │
│  - StatefulSets                         │
│  - Services                             │
│  - ConfigMaps                           │
│  - Secrets                              │
│  - PersistentVolumeClaims               │
└─────────────────────────────────────────┘
```

### La boucle de réconciliation (Reconciliation Loop)

C'est le cœur d'un Operator. Elle fonctionne en continu :

```python
# Pseudo-code simplifié
def reconciliation_loop():
    while True:
        # 1. Récupérer toutes les Custom Resources
        custom_resources = api.get_all_custom_resources()

        for resource in custom_resources:
            # 2. Lire l'état désiré (spec)
            desired_state = resource.spec

            # 3. Lire l'état actuel
            actual_state = get_current_state(resource)

            # 4. Comparer
            if actual_state != desired_state:
                # 5. Calculer les actions nécessaires
                actions = calculate_diff(desired_state, actual_state)

                # 6. Exécuter les actions
                execute_actions(actions)

                # 7. Mettre à jour le status
                resource.status = get_new_status()
                api.update_status(resource)

        # 8. Attendre un peu avant la prochaine itération
        sleep(30)  # Ou attendre un événement
```

### Événements déclencheurs

Les Operators réagissent à différents types d'événements :

1. **Création** : Une nouvelle CR est créée
   ```bash
   kubectl apply -f mon-cluster.yaml
   → L'Operator détecte et crée les ressources
   ```

2. **Modification** : Une CR existante est modifiée
   ```bash
   kubectl edit postgresqlcluster mon-cluster
   → L'Operator détecte et applique les changements
   ```

3. **Suppression** : Une CR est supprimée
   ```bash
   kubectl delete postgresqlcluster mon-cluster
   → L'Operator nettoie toutes les ressources associées
   ```

4. **Événements externes** : Un Pod crash, un volume est plein, etc.
   ```
   Pod PostgreSQL crash
   → L'Operator détecte et recrée le Pod
   ```

## Operators populaires et cas d'usage

### 1. Prometheus Operator

**Objectif** : Gérer Prometheus et ses composants

**Custom Resources** :
- `Prometheus` : Déploie un serveur Prometheus
- `ServiceMonitor` : Définit ce qu'il faut monitorer
- `PrometheusRule` : Définit les alertes
- `Alertmanager` : Gère les notifications

**Exemple** :
```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: mon-prometheus
spec:
  replicas: 2
  serviceMonitorSelector:
    matchLabels:
      team: backend
  resources:
    requests:
      memory: 400Mi
```

L'Operator crée automatiquement :
- StatefulSet Prometheus avec 2 réplicas
- Services associés
- Configuration Prometheus
- Gestion de la haute disponibilité

### 2. PostgreSQL Operators

Plusieurs Operators PostgreSQL existent :

#### Zalando PostgreSQL Operator
```yaml
apiVersion: "acid.zalan.do/v1"
kind: postgresql
metadata:
  name: acid-minimal-cluster
spec:
  teamId: "acid"
  volume:
    size: 10Gi
  numberOfInstances: 3
  users:
    zalando:
    - superuser
    - createdb
  databases:
    foo: zalando
  postgresql:
    version: "15"
```

L'Operator gère :
- ✅ Réplication streaming
- ✅ Backups avec WAL archiving
- ✅ Point-in-time recovery
- ✅ Connection pooling avec PgBouncer
- ✅ Monitoring avec exporters

#### CloudNativePG (recommandé)
```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: mon-cluster
spec:
  instances: 3
  primaryUpdateStrategy: unsupervised
  storage:
    size: 20Gi
  backup:
    barmanObjectStore:
      destinationPath: s3://mes-backups/
      s3Credentials:
        accessKeyId:
          name: aws-creds
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: aws-creds
          key: SECRET_ACCESS_KEY
```

### 3. MySQL Operator (Oracle)

```yaml
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: mon-cluster-mysql
spec:
  secretName: mysql-secret
  instances: 3
  router:
    instances: 2
  datadirVolumeClaimTemplate:
    accessModes: [ "ReadWriteOnce" ]
    resources:
      requests:
        storage: 50Gi
```

### 4. Elasticsearch Operator (Elastic)

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: mon-elasticsearch
spec:
  version: 8.11.0
  nodeSets:
  - name: default
    count: 3
    config:
      node.store.allow_mmap: false
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 100Gi
```

L'Operator gère :
- Déploiement du cluster
- Mise à l'échelle
- Mises à jour rolling
- Gestion des certificats TLS
- Snapshots et restauration

### 5. Cert-Manager

Bien que techniquement pas un Operator de database, Cert-Manager est un excellent exemple d'Operator :

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: mon-certificat
spec:
  secretName: mon-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - example.com
  - www.example.com
```

L'Operator :
- Demande le certificat à Let's Encrypt
- Stocke le certificat dans un Secret
- Renouvelle automatiquement avant expiration

### 6. ArgoCD

ArgoCD peut être considéré comme un Operator GitOps :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mon-app
spec:
  project: default
  source:
    repoURL: https://github.com/mon-org/mon-repo
    targetRevision: HEAD
    path: kubernetes/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

L'Operator :
- Surveille le repository Git
- Détecte les changements
- Synchronise automatiquement le cluster
- Maintient la cohérence Git ↔ Cluster

## Créer un Operator simple

### Framework pour créer des Operators

Plusieurs frameworks existent pour faciliter la création d'Operators :

#### 1. Operator SDK (le plus populaire)

Supporte plusieurs langages :
- **Go** : Performance maximale, le plus utilisé
- **Ansible** : Pour ceux qui connaissent Ansible
- **Helm** : Wrapper autour de Helm charts

```bash
# Installer Operator SDK
curl -LO https://github.com/operator-framework/operator-sdk/releases/download/v1.32.0/operator-sdk_linux_amd64
chmod +x operator-sdk_linux_amd64
sudo mv operator-sdk_linux_amd64 /usr/local/bin/operator-sdk

# Créer un nouveau projet Operator
operator-sdk init --domain example.com --repo github.com/example/mon-operator

# Créer une nouvelle API (CRD)
operator-sdk create api --group cache --version v1alpha1 --kind Memcached --resource --controller
```

#### 2. Kubebuilder

Framework de référence pour Go, maintenu par Kubernetes :

```bash
# Installer Kubebuilder
curl -L -o kubebuilder https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)
chmod +x kubebuilder
sudo mv kubebuilder /usr/local/bin/

# Créer un projet
kubebuilder init --domain example.com --repo github.com/example/mon-operator
kubebuilder create api --group apps --version v1 --kind MyApp
```

#### 3. KOPF (Kubernetes Operator Pythonic Framework)

Pour ceux qui préfèrent Python :

```python
import kopf
import kubernetes

@kopf.on.create('example.com', 'v1', 'myapps')
def create_fn(spec, name, namespace, **kwargs):
    # Logique de création
    replicas = spec.get('replicas', 1)
    image = spec.get('image', 'nginx:latest')

    # Créer un Deployment
    api = kubernetes.client.AppsV1Api()
    deployment = kubernetes.client.V1Deployment(
        metadata=kubernetes.client.V1ObjectMeta(name=name),
        spec=kubernetes.client.V1DeploymentSpec(
            replicas=replicas,
            selector=kubernetes.client.V1LabelSelector(
                match_labels={"app": name}
            ),
            template=kubernetes.client.V1PodTemplateSpec(
                metadata=kubernetes.client.V1ObjectMeta(
                    labels={"app": name}
                ),
                spec=kubernetes.client.V1PodSpec(
                    containers=[
                        kubernetes.client.V1Container(
                            name=name,
                            image=image
                        )
                    ]
                )
            )
        )
    )
    api.create_namespaced_deployment(namespace, deployment)
    return {'message': 'Deployment created'}
```

### Exemple conceptuel : Operator pour gérer un blog

Imaginons que vous voulez créer un Operator qui gère un blog WordPress :

#### 1. Définir la CRD

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: blogs.myblog.example.com
spec:
  group: myblog.example.com
  names:
    kind: Blog
    plural: blogs
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              title:
                type: string
              domain:
                type: string
              theme:
                type: string
              plugins:
                type: array
                items:
                  type: string
```

#### 2. Créer une instance

```yaml
apiVersion: myblog.example.com/v1
kind: Blog
metadata:
  name: mon-super-blog
spec:
  title: "Mon Super Blog Tech"
  domain: "monsuperblog.com"
  theme: "twentytwentythree"
  plugins:
  - "yoast-seo"
  - "wp-super-cache"
```

#### 3. L'Operator réagit

Quand vous créez cette ressource, l'Operator :

1. Crée un Deployment WordPress
2. Crée un Deployment MySQL
3. Crée les Services associés
4. Crée les PersistentVolumeClaims
5. Configure WordPress avec le titre, thème, plugins
6. Crée un Ingress pour le domaine
7. Configure les certificats SSL avec Cert-Manager

Tout ça automatiquement !

## Installation d'Operators sur MicroK8s

### Méthode 1 : Operator Lifecycle Manager (OLM)

OLM est le "package manager" des Operators. Il simplifie l'installation et la gestion.

```bash
# Installer OLM
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.26.0/install.sh | bash -s v0.26.0

# Vérifier l'installation
kubectl get pods -n olm

# Installer un Operator depuis OperatorHub
kubectl create -f https://operatorhub.io/install/prometheus.yaml

# Vérifier que l'Operator est installé
kubectl get csv -n operators
```

### Méthode 2 : Helm Charts

Beaucoup d'Operators sont disponibles via Helm :

```bash
# Installer CloudNativePG Operator
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm upgrade --install cnpg \
  --namespace cnpg-system \
  --create-namespace \
  cnpg/cloudnative-pg

# Vérifier
kubectl get pods -n cnpg-system
```

### Méthode 3 : Manifestes YAML directs

Certains Operators fournissent des fichiers YAML simples :

```bash
# Installer Cert-Manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Vérifier
kubectl get pods -n cert-manager
```

## Exemple pratique : Utiliser CloudNativePG sur MicroK8s

### Installation

```bash
# 1. Installer l'Operator
kubectl apply -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.21/releases/cnpg-1.21.0.yaml

# 2. Vérifier que l'Operator tourne
kubectl get pods -n cnpg-system

# Vous devriez voir :
# NAME                                  READY   STATUS    RESTARTS   AGE
# cnpg-controller-manager-xxx-xxx       1/1     Running   0          30s
```

### Créer un cluster PostgreSQL

```yaml
# Fichier : postgres-cluster.yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: cluster-example
spec:
  instances: 3

  postgresql:
    parameters:
      max_connections: "100"
      shared_buffers: "256MB"

  bootstrap:
    initdb:
      database: myapp
      owner: myapp
      secret:
        name: app-secret

  storage:
    size: 10Gi
    storageClass: microk8s-hostpath

  monitoring:
    enablePodMonitor: true
```

```bash
# Créer le Secret pour l'utilisateur
kubectl create secret generic app-secret \
  --from-literal=username=myapp \
  --from-literal=password=SuperSecretPassword123

# Déployer le cluster
kubectl apply -f postgres-cluster.yaml

# Observer la magie opérer !
kubectl get pods -w

# Vous verrez :
# cluster-example-1      0/1     Pending     0          0s
# cluster-example-1      0/1     Init:0/1    0          2s
# cluster-example-1      1/1     Running     0          10s
# cluster-example-2      0/1     Pending     0          0s
# cluster-example-2      1/1     Running     0          15s
# cluster-example-3      0/1     Pending     0          0s
# cluster-example-3      1/1     Running     0          20s
```

### Interagir avec le cluster

```bash
# Voir le status du cluster
kubectl get cluster

# Voir les détails
kubectl describe cluster cluster-example

# Se connecter au primary
kubectl exec -it cluster-example-1 -- psql -U myapp myapp

# Faire un backup
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: backup-$(date +%Y%m%d-%H%M%S)
spec:
  cluster:
    name: cluster-example
EOF

# Lister les backups
kubectl get backups
```

### Scaler le cluster

```bash
# Passer de 3 à 5 instances
kubectl patch cluster cluster-example --type='merge' -p '{"spec":{"instances":5}}'

# L'Operator va automatiquement :
# - Créer 2 nouveaux Pods
# - Les configurer en replica
# - Mettre à jour la réplication
```

## Opérateurs vs Helm Charts

On confond souvent Operators et Helm, mais ce sont des outils complémentaires :

| Critère | Helm | Operator |
|---------|------|----------|
| **But principal** | Templating et packaging | Gestion du cycle de vie |
| **Intelligence** | Aucune après installation | Intelligence continue |
| **Day 1** | Excellent | Bon |
| **Day 2+** | Limité | Excellent |
| **Mises à jour** | Manuelles | Automatiques possibles |
| **Auto-réparation** | Non | Oui |
| **Backups/Restore** | Non natif | Souvent intégré |
| **Complexité** | Simple | Plus complexe |
| **Cas d'usage** | Déploiement initial | Applications stateful complexes |

**Day 1** = Installation initiale
**Day 2+** = Opérations quotidiennes (backups, scaling, upgrades, troubleshooting)

### Quand utiliser quoi ?

**Utilisez Helm** pour :
- Applications stateless simples
- Déploiement initial rapide
- Packaging et distribution
- Applications sans état complexe

**Utilisez un Operator** pour :
- Bases de données
- Systèmes distribués (Kafka, Elasticsearch)
- Applications avec logique métier complexe
- Besoin de gestion automatisée du cycle de vie

**Les deux ensemble** :
Beaucoup d'Operators sont distribués via Helm ! Vous utilisez Helm pour installer l'Operator, puis l'Operator gère vos ressources.

## Bonnes pratiques

### 1. Choisir le bon Operator

Avant d'installer un Operator, vérifiez :

- ✅ **Maturité** : Niveau dans l'Operator Maturity Model
- ✅ **Maintenance** : Dernière mise à jour récente ?
- ✅ **Communauté** : Nombre de stars GitHub, issues actives
- ✅ **Documentation** : Est-elle complète et claire ?
- ✅ **Production-ready** : Utilisé par d'autres en production ?

Sites utiles :
- OperatorHub.io : Catalogue d'Operators
- Artifact Hub : Recherche d'Operators et Charts
- Awesome Operators : Liste curatée sur GitHub

### 2. Tester avant la production

```bash
# Créer un namespace de test
kubectl create namespace operator-test

# Installer l'Operator dans ce namespace
# Tester exhaustivement
# Supprimer proprement
kubectl delete namespace operator-test
```

### 3. Surveiller l'Operator lui-même

Les Operators sont des applications comme les autres :

```bash
# Voir les logs de l'Operator
kubectl logs -n cnpg-system deployment/cnpg-controller-manager -f

# Métriques
kubectl top pod -n cnpg-system

# Surveiller avec Prometheus
kubectl get servicemonitor -n cnpg-system
```

### 4. Comprendre les finalizers

Les Operators utilisent souvent des **finalizers** pour nettoyer les ressources :

```yaml
metadata:
  finalizers:
  - postgresql.cnpg.io/finalizer
```

Si vous supprimez une CR et qu'elle reste en "Terminating" :
```bash
# Vérifier les finalizers
kubectl get postgresqlcluster mon-cluster -o yaml

# Si l'Operator est cassé, vous pouvez forcer la suppression (ATTENTION !)
kubectl patch postgresqlcluster mon-cluster -p '{"metadata":{"finalizers":null}}' --type=merge
```

### 5. Versioning et upgrades

```yaml
apiVersion: postgresql.cnpg.io/v1  # Toujours vérifier la version de l'API
kind: Cluster
metadata:
  name: mon-cluster
spec:
  imageName: ghcr.io/cloudnative-pg/postgresql:15.2  # Version spécifique
```

- Utilisez des versions spécifiques, pas `latest`
- Testez les upgrades dans un environnement de staging
- Lisez les release notes avant d'upgrader

### 6. Backup de vos Custom Resources

Les CRs sont stockées dans etcd. Pensez à les sauvegarder :

```bash
# Backup de toutes les CRs d'un type
kubectl get postgresqlclusters -A -o yaml > backup-postgres-clusters.yaml

# Backup avec Velero (inclut les CRDs et CRs)
velero backup create --include-cluster-resources=true
```

## Limitations et considérations

### ⚠️ Complexité accrue

Les Operators ajoutent une couche de complexité :
- Un nouveau composant à maintenir (l'Operator lui-même)
- Debugging plus difficile (il faut comprendre la logique de l'Operator)
- Dépendance à un projet tiers

### ⚠️ Consommation de ressources

Un Operator tourne 24/7 dans votre cluster :
- CPU : généralement faible (~50-100m)
- RAM : ~100-200 Mo par Operator
- Pour un lab MicroK8s, n'installez que les Operators nécessaires

### ⚠️ Versions et compatibilité

- Les Operators peuvent avoir des breaking changes
- Les CRDs peuvent évoluer (v1alpha1 → v1beta1 → v1)
- Toujours vérifier la compatibilité avec votre version de Kubernetes

### ⚠️ Vendor lock-in potentiel

Si vous utilisez massivement des CRs spécifiques à un Operator, migrer devient difficile :

```yaml
# Spécifique à CloudNativePG
apiVersion: postgresql.cnpg.io/v1
kind: Cluster

# Si vous voulez changer pour Zalando Operator
apiVersion: acid.zalan.do/v1
kind: postgresql
# → Il faut réécrire toutes vos définitions !
```

## Operators sur MicroK8s : Recommandations

### Pour un lab personnel

**Operators recommandés** :
1. **Cert-Manager** : Gestion des certificats SSL
2. **Prometheus Operator** : Si vous faites du monitoring
3. **CloudNativePG** OU **MySQL Operator** : Si vous avez besoin d'une database
4. **Sealed Secrets** : Gestion sécurisée des secrets

**À éviter pour un lab** :
- Operators très lourds (Istio Operator, Elasticsearch sur 3 nodes)
- Multiplier les Operators (choisissez-en 2-3 maximum)

### Configuration pour MicroK8s

```yaml
# Limiter les ressources des Operators
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-operator
spec:
  template:
    spec:
      containers:
      - name: operator
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
```

## Créer votre propre Operator : Faut-il le faire ?

### ✅ OUI si :

- Vous gérez une application très spécifique à votre entreprise
- La logique métier est complexe et répétitive
- Vous voulez standardiser les déploiements dans toute l'organisation
- Vous avez le temps et les compétences pour le maintenir

### ❌ NON si :

- Un Operator existant fait déjà le job
- Votre application est simple (un Deployment suffit)
- Vous n'avez pas les ressources pour maintenir du code
- Vous débutez avec Kubernetes (apprenez d'abord les bases)

### Alternative : Utiliser Helm avec des hooks

Avant de créer un Operator complet, considérez Helm avec des hooks pour de la logique simple :

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "0"
spec:
  containers:
  - name: post-install-job
    image: alpine
    command: ["/bin/sh", "-c", "echo Installation terminée"]
  restartPolicy: Never
```

## Conclusion

Les **Custom Resources** et **Operators** sont des concepts puissants qui permettent d'étendre Kubernetes pour gérer n'importe quel type d'application, aussi complexe soit-elle.

### Points clés à retenir

1. **CRD** = Définition d'un nouveau type de ressource
2. **CR** = Instance de cette ressource
3. **Operator** = Application qui donne vie aux CRs
4. Les Operators **encodent l'expertise humaine** dans du code
5. Ils utilisent une **boucle de réconciliation** pour maintenir l'état désiré
6. Il existe des **niveaux de maturité** (de 1 à 5)
7. De nombreux **Operators production-ready** existent déjà
8. Créer son propre Operator n'est **pas toujours nécessaire**

### Recommandations pour MicroK8s

**Pour apprendre** :
- Installez Cert-Manager pour comprendre les concepts
- Testez un Operator de database (CloudNativePG)
- Explorez les CRDs : `kubectl get crd`
- Regardez comment un Operator réagit aux changements

**Pour un usage pratique** :
- Choisissez 2-3 Operators maximum
- Privilégiez les Operators matures et maintenus
- Lisez la documentation avant d'installer
- Testez dans un namespace dédié

**Pour aller plus loin** :
- Étudiez le code d'Operators open-source
- Suivez des tutoriels Operator SDK
- Contribuez à des projets d'Operators existants

### Prochaines étapes

Après avoir exploré les Operators, vous pourriez vous intéresser à :
- **17.5 ArgoCD pour GitOps** : Operator GitOps pour déploiement déclaratif
- **25.1 Service Mesh (Istio)** : Qui utilise des Operators pour se gérer
- **22.2 Sauvegarde avec Velero** : Un Operator de backup

---

**Ressources complémentaires** :

- OperatorHub.io : Catalogue d'Operators https://operatorhub.io/
- Operator SDK : https://sdk.operatorframework.io/
- Kubebuilder Book : https://book.kubebuilder.io/
- CNCF Operators White Paper : https://www.cncf.io/wp-content/uploads/2021/07/CNCF_Operator_WhitePaper.pdf
- Awesome Operators : https://github.com/operator-framework/awesome-operators
- KOPF (Python) : https://kopf.readthedocs.io/

⏭️ [Multi-cluster management](/25-aller-plus-loin/04-multi-cluster-management.md)
