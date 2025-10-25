🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.6 Helm : gestionnaire de packages

## Introduction

Vous avez appris à gérer vos applications Kubernetes avec des manifestes YAML versionnés dans Git et synchronisés via ArgoCD. Mais imaginez que vous vouliez déployer une application complexe comme PostgreSQL, Redis, ou Prometheus : vous devez gérer des dizaines de fichiers YAML, avec des configurations différentes selon les environnements, et des valeurs à personnaliser.

C'est là qu'intervient **Helm**, le gestionnaire de packages de Kubernetes. Helm vous permet de **packager**, **versionner** et **distribuer** vos applications Kubernetes de manière simple et réutilisable. C'est l'équivalent de `apt` pour Ubuntu, `npm` pour Node.js, ou `pip` pour Python, mais pour Kubernetes.

Dans ce chapitre, nous allons découvrir Helm, comprendre ses concepts fondamentaux, et apprendre à utiliser et créer des packages Helm pour simplifier drastiquement vos déploiements.

---

## Qu'est-ce que Helm ?

### Définition

**Helm** est le **gestionnaire de packages** pour Kubernetes. Il permet de définir, installer et mettre à jour des applications Kubernetes complexes de manière standardisée et reproductible.

Créé par Deis (acquis par Microsoft) en 2015, Helm est devenu un projet gradué de la **CNCF** et est aujourd'hui l'outil de packaging le plus utilisé dans l'écosystème Kubernetes.

### La métaphore nautique

Le nom "Helm" (gouvernail en français) s'inscrit dans la métaphore nautique de Kubernetes :
- **Kubernetes** = le timonier (kubernetes en grec)
- **Helm** = le gouvernail qui permet de diriger le navire

### Le problème que Helm résout

**Sans Helm**, déployer une application complexe ressemble à ceci :

```bash
# Déployer PostgreSQL manuellement
kubectl apply -f postgres-configmap.yaml
kubectl apply -f postgres-secret.yaml
kubectl apply -f postgres-pvc.yaml
kubectl apply -f postgres-statefulset.yaml
kubectl apply -f postgres-service.yaml
kubectl apply -f postgres-networkpolicy.yaml

# Puis pour Redis...
kubectl apply -f redis-configmap.yaml
kubectl apply -f redis-deployment.yaml
# ... et ainsi de suite
```

**Avec Helm**, c'est aussi simple que :

```bash
helm install my-postgres bitnami/postgresql
helm install my-redis bitnami/redis
```

Une seule commande déploie tous les composants nécessaires, correctement configurés et interconnectés !

### Les avantages de Helm

**Packaging** :
- ✅ Regrouper tous les manifestes d'une application dans un seul package
- ✅ Versionner l'ensemble de l'application
- ✅ Distribuer facilement via des repositories

**Réutilisabilité** :
- ✅ Déployer la même application dans différents environnements
- ✅ Customiser facilement via des valeurs
- ✅ Partager des packages avec la communauté

**Gestion du cycle de vie** :
- ✅ Installer, mettre à jour, supprimer des applications
- ✅ Rollback vers une version précédente
- ✅ Historique des déploiements

**Templating** :
- ✅ Générer dynamiquement des manifestes
- ✅ Réutiliser des patterns communs
- ✅ Éviter la duplication de code

---

## Concepts fondamentaux de Helm

### Chart

Un **Chart** est un package Helm. C'est un ensemble de fichiers qui décrit une application Kubernetes complète.

**Structure d'un Chart** :

```
mon-app/
├── Chart.yaml              # Métadonnées du chart
├── values.yaml             # Valeurs par défaut
├── charts/                 # Charts dépendants (optionnel)
├── templates/              # Templates des manifestes K8s
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl        # Fonctions helper
│   └── NOTES.txt          # Instructions post-installation
├── .helmignore            # Fichiers à ignorer
└── README.md              # Documentation
```

**Analogies** :
- Chart Helm = Package Debian (.deb)
- Chart Helm = Package NPM
- Chart Helm = Image Docker (mais pour des applications K8s complètes)

### Release

Une **Release** est une instance d'un Chart déployée dans un cluster Kubernetes.

**Exemple** :
```bash
# Installer un chart PostgreSQL → Créer une release
helm install my-database bitnami/postgresql

# "my-database" = nom de la release
# "bitnami/postgresql" = le chart utilisé
```

Vous pouvez installer le même Chart plusieurs fois avec des noms différents, créant ainsi plusieurs Releases :

```bash
helm install prod-database bitnami/postgresql
helm install staging-database bitnami/postgresql
helm install dev-database bitnami/postgresql
```

Chaque release est indépendante et peut avoir des configurations différentes.

### Repository

Un **Repository** (dépôt) est un serveur HTTP qui héberge des Charts Helm packagés.

**Repositories populaires** :
- **Artifact Hub** (https://artifacthub.io) : Le "Docker Hub" de Helm, index de milliers de charts
- **Bitnami** : Charts de haute qualité pour applications populaires
- **Stable** : Charts maintenus par la communauté (historique)
- **Votre propre repo** : Vous pouvez héberger vos charts privés

**Analogies** :
- Repository Helm = Repository APT
- Repository Helm = Registry NPM
- Repository Helm = Docker Registry (mais pour Charts)

### Values

Les **Values** sont les valeurs de configuration qui personnalisent un Chart.

**Fichier values.yaml par défaut** :

```yaml
# values.yaml du chart
replicaCount: 1

image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false

resources:
  limits:
    cpu: 100m
    memory: 128Mi
```

**Personnalisation lors de l'installation** :

```bash
# Surcharger des valeurs
helm install my-app ./mon-chart \
  --set replicaCount=3 \
  --set image.tag=1.26

# Ou via un fichier
helm install my-app ./mon-chart -f custom-values.yaml
```

### Templates

Les **Templates** sont des fichiers YAML avec des directives Go Template qui génèrent les manifestes Kubernetes.

**Exemple de template** :

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
  labels:
    app: {{ .Chart.Name }}
    version: {{ .Chart.Version }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.port }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

Les parties entre `{{ }}` sont remplacées dynamiquement par les valeurs lors de l'installation.

---

## Installation de Helm

### Linux

```bash
# Méthode 1 : Script d'installation officiel (recommandé)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Méthode 2 : Via le gestionnaire de packages
# Ubuntu/Debian
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

# Méthode 3 : Téléchargement manuel
wget https://get.helm.sh/helm-v3.13.0-linux-amd64.tar.gz
tar -zxvf helm-v3.13.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
```

### macOS

```bash
# Avec Homebrew
brew install helm
```

### Windows

```powershell
# Avec Chocolatey
choco install kubernetes-helm

# Avec Scoop
scoop install helm
```

### Vérification

```bash
helm version

# Devrait afficher quelque chose comme :
# version.BuildInfo{Version:"v3.13.0", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.21.0"}
```

### Configuration initiale

Helm 3 n'a pas besoin de Tiller (composant serveur de Helm 2), il fonctionne directement avec votre kubeconfig.

```bash
# Vérifier la configuration
helm env

# Lister les releases (vide pour l'instant)
helm list
```

---

## Utiliser des Charts Helm

### Ajouter des repositories

Avant d'installer des charts, ajoutez des repositories :

```bash
# Ajouter le repository Bitnami (très populaire)
helm repo add bitnami https://charts.bitnami.com/bitnami

# Ajouter d'autres repositories
helm repo add stable https://charts.helm.sh/stable
helm repo add nginx-stable https://helm.nginx.com/stable

# Lister les repositories configurés
helm repo list

# Mettre à jour les repositories (comme apt-get update)
helm repo update
```

### Rechercher des Charts

```bash
# Rechercher un chart dans les repositories
helm search repo postgres

# Résultat :
# NAME                    CHART VERSION   APP VERSION     DESCRIPTION
# bitnami/postgresql      13.2.0          16.1.0          PostgreSQL is a powerful, open source...
# bitnami/postgresql-ha   12.0.0          16.1.0          PostgreSQL with HA architecture...

# Rechercher sur Artifact Hub (tous les repositories publics)
helm search hub postgres

# Afficher les informations d'un chart
helm show chart bitnami/postgresql
helm show readme bitnami/postgresql
helm show values bitnami/postgresql
```

### Installer un Chart

#### Installation basique

```bash
# Syntaxe : helm install [NOM-RELEASE] [CHART]
helm install my-postgres bitnami/postgresql

# Helm crée automatiquement toutes les ressources nécessaires :
# - StatefulSet PostgreSQL
# - Service
# - PersistentVolumeClaim
# - Secrets pour les mots de passe
# - ConfigMap
```

Après l'installation, Helm affiche des **NOTES** avec des instructions utiles :

```
NAME: my-postgres
LAST DEPLOYED: Sat Oct 25 15:30:00 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
** Please be patient while the chart is being deployed **

PostgreSQL can be accessed via port 5432 on the following DNS names from within your cluster:

    my-postgres-postgresql.default.svc.cluster.local - Read/Write connection

To get the password for "postgres" run:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace default my-postgres-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

To connect to your database run the following command:

    kubectl run my-postgres-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:16.1.0-debian-11-r0 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host my-postgres-postgresql -U postgres -d postgres -p 5432
```

#### Installation avec personnalisation

```bash
# Surcharger des valeurs en ligne de commande
helm install my-postgres bitnami/postgresql \
  --set auth.postgresPassword=SuperSecret123 \
  --set primary.persistence.size=20Gi

# Avec un fichier de valeurs personnalisées
helm install my-postgres bitnami/postgresql -f my-values.yaml

# Dans un namespace spécifique
helm install my-postgres bitnami/postgresql \
  --namespace database \
  --create-namespace

# Générer un nom aléatoire
helm install bitnami/postgresql --generate-name
```

#### Dry-run : Tester sans installer

```bash
# Voir ce qui serait installé sans l'installer réellement
helm install my-postgres bitnami/postgresql --dry-run --debug

# Cela affiche tous les manifestes YAML qui seraient appliqués
```

### Lister les Releases

```bash
# Lister toutes les releases installées
helm list

# Dans tous les namespaces
helm list --all-namespaces

# Afficher aussi les releases désinstallées
helm list --all
```

**Résultat** :
```
NAME            NAMESPACE   REVISION    UPDATED         STATUS      CHART               APP VERSION
my-postgres     default     1           2025-10-25      deployed    postgresql-13.2.0   16.1.0
my-redis        default     1           2025-10-25      deployed    redis-18.2.0        7.2.3
```

### Obtenir des informations sur une Release

```bash
# Statut général
helm status my-postgres

# Historique des révisions
helm history my-postgres

# Valeurs utilisées lors de l'installation
helm get values my-postgres

# Valeurs complètes (par défaut + personnalisées)
helm get values my-postgres --all

# Manifestes générés
helm get manifest my-postgres

# Tout (status + values + manifest + hooks)
helm get all my-postgres
```

### Mettre à jour une Release

```bash
# Mettre à jour avec de nouvelles valeurs
helm upgrade my-postgres bitnami/postgresql \
  --set auth.postgresPassword=NewPassword456

# Mettre à jour vers une nouvelle version du chart
helm upgrade my-postgres bitnami/postgresql --version 13.3.0

# Réutiliser les valeurs de la release existante
helm upgrade my-postgres bitnami/postgresql --reuse-values

# Ou avec un nouveau fichier de values
helm upgrade my-postgres bitnami/postgresql -f updated-values.yaml
```

**Note importante** : `helm upgrade` peut aussi servir à **installer** si la release n'existe pas :

```bash
# Install or upgrade (idempotent)
helm upgrade --install my-postgres bitnami/postgresql
```

### Rollback d'une Release

Si une mise à jour pose problème, revenez facilement en arrière :

```bash
# Voir l'historique
helm history my-postgres

# REVISION  UPDATED                  STATUS      CHART               DESCRIPTION
# 1         Sat Oct 25 15:30:00 2025 superseded  postgresql-13.2.0   Install complete
# 2         Sat Oct 25 16:00:00 2025 superseded  postgresql-13.2.1   Upgrade complete
# 3         Sat Oct 25 16:30:00 2025 deployed    postgresql-13.3.0   Upgrade complete

# Rollback vers la révision 2
helm rollback my-postgres 2

# Rollback vers la révision précédente
helm rollback my-postgres
```

### Désinstaller une Release

```bash
# Désinstaller (supprime toutes les ressources)
helm uninstall my-postgres

# Garder l'historique (pour pouvoir rollback même après suppression)
helm uninstall my-postgres --keep-history
```

---

## Personnalisation avec Values

### Fichier values.yaml

Le fichier `values.yaml` contient toutes les valeurs configurables d'un Chart.

**Exemple pour une application web** :

```yaml
# values.yaml

# Nombre de replicas
replicaCount: 2

# Configuration de l'image
image:
  repository: localhost:32000/mon-app
  tag: "v1.2.3"
  pullPolicy: IfNotPresent

# Service
service:
  type: ClusterIP
  port: 80
  targetPort: 8080

# Ingress
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: app-tls
      hosts:
        - app.example.com

# Ressources
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

# Autoscaling
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

# Variables d'environnement
env:
  - name: DATABASE_URL
    value: "postgresql://postgres:5432/mydb"
  - name: LOG_LEVEL
    value: "info"

# Configuration custom
config:
  debug: false
  apiKey: ""
  features:
    newUI: true
    betaFeatures: false
```

### Hiérarchie des values

Les values peuvent être structurées hiérarchiquement :

```yaml
database:
  host: postgres.default.svc.cluster.local
  port: 5432
  name: myapp
  user: admin
  pool:
    min: 2
    max: 10

redis:
  host: redis.default.svc.cluster.local
  port: 6379
  db: 0
```

**Accès dans les templates** :

```yaml
env:
  - name: DB_HOST
    value: {{ .Values.database.host }}
  - name: DB_PORT
    value: {{ .Values.database.port | quote }}
  - name: REDIS_HOST
    value: {{ .Values.redis.host }}
```

### Plusieurs fichiers de values

Vous pouvez avoir différents fichiers pour différents environnements :

```
charts/mon-app/
├── values.yaml              # Valeurs par défaut
├── values-dev.yaml          # Surcharges pour dev
├── values-staging.yaml      # Surcharges pour staging
└── values-production.yaml   # Surcharges pour prod
```

**values-production.yaml** :

```yaml
# Surcharger seulement ce qui change
replicaCount: 5

resources:
  limits:
    cpu: 2000m
    memory: 2Gi
  requests:
    cpu: 1000m
    memory: 1Gi

autoscaling:
  enabled: true
  maxReplicas: 20

ingress:
  hosts:
    - host: app.example.com
```

**Installation** :

```bash
# Development
helm install mon-app ./charts/mon-app -f charts/mon-app/values-dev.yaml

# Production
helm install mon-app ./charts/mon-app -f charts/mon-app/values-production.yaml
```

### Surcharge via ligne de commande

```bash
# --set pour des valeurs simples
helm install mon-app ./charts/mon-app \
  --set replicaCount=3 \
  --set image.tag=v2.0.0

# --set pour des valeurs imbriquées
helm install mon-app ./charts/mon-app \
  --set database.host=postgres.prod.svc.cluster.local \
  --set database.port=5432

# --set pour des tableaux
helm install mon-app ./charts/mon-app \
  --set "ingress.hosts[0].host=app1.example.com" \
  --set "ingress.hosts[1].host=app2.example.com"

# --set-string pour forcer une valeur en string
helm install mon-app ./charts/mon-app \
  --set-string resources.limits.cpu="1000m"

# --set-file pour lire depuis un fichier
helm install mon-app ./charts/mon-app \
  --set-file config.json=./config.json
```

### Ordre de priorité

Les values sont fusionnées avec cette priorité (de la plus faible à la plus forte) :

1. **values.yaml** du chart (par défaut)
2. **values.yaml** des charts parents (si sous-chart)
3. **Fichiers -f** (dans l'ordre spécifié)
4. **Paramètres --set** (dernier mot)

```bash
# Exemple avec priorités
helm install mon-app ./chart \
  -f values-base.yaml \      # Priorité 3
  -f values-prod.yaml \      # Priorité 4 (écrase values-base)
  --set replicaCount=5       # Priorité 5 (écrase tout)
```

---

## Templates Helm avancés

### Objets prédéfinis

Dans les templates, plusieurs objets sont disponibles :

**`.Release`** : Informations sur la release

```yaml
metadata:
  name: {{ .Release.Name }}-deployment
  namespace: {{ .Release.Namespace }}
  labels:
    release: {{ .Release.Name }}
    service: {{ .Release.Service }}  # "Helm"
```

**`.Chart`** : Informations du fichier Chart.yaml

```yaml
labels:
  app: {{ .Chart.Name }}
  version: {{ .Chart.Version }}
  appVersion: {{ .Chart.AppVersion }}
```

**`.Values`** : Valeurs du fichier values.yaml

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
```

**`.Capabilities`** : Informations sur le cluster

```yaml
{{- if .Capabilities.APIVersions.Has "networking.k8s.io/v1/Ingress" }}
# Utiliser la nouvelle API Ingress
{{- end }}
```

### Fonctions et pipelines

Helm utilise le moteur de template de Go avec des fonctions supplémentaires.

**Fonctions de base** :

```yaml
# Mettre en majuscules
{{ .Values.appName | upper }}

# Mettre en minuscules
{{ .Values.appName | lower }}

# Quote (ajouter des guillemets)
{{ .Values.database.port | quote }}

# Valeur par défaut
{{ .Values.description | default "My application" }}

# Indentation
{{ .Values.config | toYaml | indent 2 }}
{{ .Values.config | toYaml | nindent 4 }}

# Encodage base64
{{ .Values.secret | b64enc }}
{{ .Values.secret | b64dec }}
```

**Fonctions de chaînes** :

```yaml
# Trim (supprimer espaces)
{{ .Values.text | trim }}

# Truncate (tronquer)
{{ .Values.longText | trunc 50 }}

# Replace
{{ .Values.url | replace "http://" "https://" }}

# Split
{{ .Values.tags | split "," }}
```

**Fonctions logiques** :

```yaml
# If/else
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
# ...
{{- end }}

# If avec condition
{{- if eq .Values.environment "production" }}
replicas: 5
{{- else }}
replicas: 1
{{- end }}

# And, or, not
{{- if and .Values.ingress.enabled .Values.ingress.tls }}
tls: true
{{- end }}
```

### Boucles

```yaml
# Range sur une liste
{{- range .Values.hosts }}
- {{ . }}
{{- end }}

# Range sur un dictionnaire
{{- range $key, $value := .Values.env }}
- name: {{ $key }}
  value: {{ $value }}
{{- end }}

# Exemple complet
env:
{{- range $key, $value := .Values.env }}
- name: {{ $key | upper }}
  value: {{ $value | quote }}
{{- end }}
```

### Variables

```yaml
{{- $appName := .Chart.Name -}}
{{- $namespace := .Release.Namespace -}}

metadata:
  name: {{ $appName }}-service
  namespace: {{ $namespace }}
```

### Helpers et fonctions réutilisables

Créez des fonctions dans `_helpers.tpl` :

```yaml
{{/* _helpers.tpl */}}

{{/*
Nom complet de l'application
*/}}
{{- define "mon-app.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{/*
Labels communs
*/}}
{{- define "mon-app.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}
```

**Utilisation dans les templates** :

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mon-app.fullname" . }}
  labels:
    {{- include "mon-app.labels" . | nindent 4 }}
spec:
  # ...
```

### Conditionnels avancés

```yaml
{{/* Activer l'ingress seulement en production */}}
{{- if and .Values.ingress.enabled (eq .Values.environment "production") }}
apiVersion: networking.k8s.io/v1
kind: Ingress
# ...
{{- end }}

{{/* Choisir le type de service selon l'environnement */}}
service:
  type: {{ if eq .Values.environment "production" }}LoadBalancer{{ else }}ClusterIP{{ end }}
```

---

## Charts personnalisés avec dépendances

### Dépendances de Charts

Un Chart peut dépendre d'autres Charts (sub-charts).

**Chart.yaml avec dépendances** :

```yaml
# Chart.yaml
apiVersion: v2
name: mon-app-complete
version: 1.0.0
dependencies:
  - name: postgresql
    version: "13.2.0"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
  - name: redis
    version: "18.2.0"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

**Télécharger les dépendances** :

```bash
# Télécharger les charts dépendants
helm dependency update ./mon-app-complete

# Les charts sont téléchargés dans charts/
ls mon-app-complete/charts/
# postgresql-13.2.0.tgz
# redis-18.2.0.tgz
```

**Configuration des sous-charts** :

```yaml
# values.yaml
postgresql:
  enabled: true
  auth:
    postgresPassword: MySecretPassword
    database: myappdb
  primary:
    persistence:
      size: 10Gi

redis:
  enabled: true
  auth:
    enabled: false
  master:
    persistence:
      size: 5Gi
```

### Umbrella Charts

Un **Umbrella Chart** est un chart qui ne contient que des dépendances (pas de templates propres).

**Structure** :

```
mon-stack/
├── Chart.yaml
├── values.yaml
└── charts/  (vide, rempli par helm dependency update)
```

**Chart.yaml** :

```yaml
apiVersion: v2
name: mon-stack
version: 1.0.0
description: Stack complète de mon application

dependencies:
  - name: postgresql
    version: "13.2.0"
    repository: https://charts.bitnami.com/bitnami
  - name: redis
    version: "18.2.0"
    repository: https://charts.bitnami.com/bitnami
  - name: nginx-ingress-controller
    version: "4.8.0"
    repository: https://charts.bitnami.com/bitnami
  - name: prometheus
    version: "25.0.0"
    repository: https://prometheus-community.github.io/helm-charts
```

Une seule commande installe toute votre stack :

```bash
helm dependency update ./mon-stack
helm install my-stack ./mon-stack
```

---

## Intégration Helm avec ArgoCD

ArgoCD supporte nativement les Charts Helm, combinant le meilleur des deux mondes.

### Application ArgoCD avec Helm

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mon-app
  namespace: argocd
spec:
  project: default

  source:
    # Option 1 : Chart depuis un repo Git
    repoURL: https://github.com/votre-org/helm-charts.git
    targetRevision: main
    path: charts/mon-app
    helm:
      valueFiles:
        - values-production.yaml
      parameters:
        - name: replicaCount
          value: "3"
        - name: image.tag
          value: "v1.2.3"

  # Option 2 : Chart depuis un Helm repository
  source:
    repoURL: https://charts.bitnami.com/bitnami
    chart: postgresql
    targetRevision: 13.2.0
    helm:
      values: |
        auth:
          postgresPassword: MySecretPassword
        primary:
          persistence:
            size: 20Gi

  destination:
    server: https://kubernetes.default.svc
    namespace: database

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Avantages de Helm + ArgoCD

✅ **Packaging** : Helm structure et package vos applications

✅ **GitOps** : ArgoCD synchronise depuis Git automatiquement

✅ **Templating** : Helm génère les manifestes selon l'environnement

✅ **Visualisation** : ArgoCD montre l'état des ressources

✅ **Historique** : Git + Helm = traçabilité complète

✅ **Rollback** : Via Git ou via ArgoCD

---

## Bonnes pratiques Helm

### 1. Versioning sémantique

Utilisez le versioning sémantique pour vos charts :

```yaml
# Chart.yaml
version: 1.2.3  # MAJOR.MINOR.PATCH
appVersion: "v2.4.1"  # Version de l'application elle-même
```

- **MAJOR** : Changements incompatibles
- **MINOR** : Nouvelles fonctionnalités compatibles
- **PATCH** : Corrections de bugs

### 2. Documentation complète

```yaml
# Chart.yaml
description: |
  Application web de gestion complète avec base de données
  PostgreSQL et cache Redis intégrés.

  Fonctionnalités :
  - Autoscaling automatique
  - Monitoring Prometheus
  - Backup automatique

keywords:
  - web
  - api
  - database
maintainers:
  - name: Jean Dupont
    email: jean@example.com
home: https://github.com/org/mon-app
sources:
  - https://github.com/org/mon-app
```

**README.md détaillé** avec exemples d'installation et configuration.

### 3. Values bien structurés

```yaml
# values.yaml avec commentaires

# -- Nombre de replicas (Integer)
replicaCount: 1

image:
  # -- Repository de l'image
  repository: nginx
  # -- Pull policy: Always, IfNotPresent, Never
  pullPolicy: IfNotPresent
  # -- Tag de l'image (laissez vide pour Chart.AppVersion)
  tag: ""

# Ressources recommandées
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

### 4. Templates réutilisables

Créez des helpers pour éviter la duplication :

```yaml
{{/* _helpers.tpl */}}

{{/*
Labels standard
*/}}
{{- define "mon-app.labels" -}}
helm.sh/chart: {{ include "mon-app.chart" . }}
{{ include "mon-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "mon-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mon-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### 5. Tests Helm

Incluez des tests pour valider les déploiements :

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "mon-app.fullname" . }}-test-connection"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
  - name: wget
    image: busybox
    command: ['wget']
    args: ['{{ include "mon-app.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

Exécuter les tests :

```bash
helm test my-release
```

### 6. Hooks pour le cycle de vie

```yaml
# templates/job-pre-install.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-pre-install"
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
      - name: pre-install-job
        image: busybox
        command: ['sh', '-c', 'echo "Préparation de l'installation..."']
      restartPolicy: Never
```

**Types de hooks** :
- `pre-install` : Avant l'installation
- `post-install` : Après l'installation
- `pre-upgrade` : Avant la mise à jour
- `post-upgrade` : Après la mise à jour
- `pre-delete` : Avant la suppression
- `post-delete` : Après la suppression
- `pre-rollback` : Avant le rollback
- `post-rollback` : Après le rollback

### 7. Validation avec schema

Créez un `values.schema.json` pour valider les values :

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["replicaCount", "image"],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "maximum": 10
    },
    "image": {
      "type": "object",
      "required": ["repository", "tag"],
      "properties": {
        "repository": {
          "type": "string"
        },
        "tag": {
          "type": "string"
        }
      }
    }
  }
}
```

Helm validera automatiquement les values contre ce schéma.

### 8. .helmignore pour exclure des fichiers

```
# .helmignore
.git/
.gitignore
*.md
docs/
tests/
.vscode/
*.tmp
```

### 9. Séparation des concerns

```
charts/mon-app/
├── templates/
│   ├── application/     # Ressources applicatives
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── hpa.yaml
│   ├── networking/      # Ressources réseau
│   │   └── ingress.yaml
│   ├── storage/         # Ressources stockage
│   │   └── pvc.yaml
│   └── monitoring/      # Ressources monitoring
│       └── servicemonitor.yaml
```

### 10. Ne pas hardcoder les valeurs

❌ **Mauvais** :

```yaml
spec:
  replicas: 3  # Hardcodé
  image: myapp:v1.0.0  # Hardcodé
```

✅ **Bon** :

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
  image: {{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}
```

---

## Sécurité des Charts Helm

### 1. Scan des vulnérabilités

```bash
# Utiliser Checkov
pip install checkov
checkov -d ./charts/mon-app/

# Utiliser Kubesec
docker run -i kubesec/kubesec scan /dev/stdin < deployment.yaml
```

### 2. Pas de secrets en clair

❌ **Ne jamais faire** :

```yaml
# values.yaml
database:
  password: SuperSecretPassword123  # ❌ JAMAIS !
```

✅ **Utiliser External Secrets ou Sealed Secrets** :

```yaml
# values.yaml
database:
  existingSecret: postgres-credentials
  secretKey: password
```

### 3. RBAC minimal

```yaml
# templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "mon-app.fullname" . }}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: {{ include "mon-app.fullname" . }}
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]  # Permissions minimales
```

### 4. Security Context

```yaml
securityContext:
  runAsNonRoot: {{ .Values.securityContext.runAsNonRoot }}
  runAsUser: {{ .Values.securityContext.runAsUser }}
  fsGroup: {{ .Values.securityContext.fsGroup }}
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: true
```

---

## Helm Plugins utiles

Helm peut être étendu avec des plugins :

### Helm Diff

Voir les différences avant d'appliquer :

```bash
# Installer
helm plugin install https://github.com/databus23/helm-diff

# Utiliser
helm diff upgrade my-release ./chart -f values-new.yaml
```

### Helm Secrets

Gérer des secrets chiffrés :

```bash
# Installer
helm plugin install https://github.com/jkroepke/helm-secrets

# Utiliser
helm secrets install my-release ./chart -f secrets.yaml
```

### Helm Dashboard

Interface web pour Helm :

```bash
# Installer
helm plugin install https://github.com/komodorio/helm-dashboard

# Lancer
helm dashboard
```

---

## Récapitulatif

### Points clés

**Helm, c'est** :
- Le gestionnaire de packages pour Kubernetes
- Packaging, templating et versioning des applications
- Réutilisabilité et partage de configurations
- Gestion du cycle de vie (install, upgrade, rollback)

**Concepts fondamentaux** :
- **Chart** : Package d'application
- **Release** : Instance déployée d'un chart
- **Repository** : Serveur hébergeant des charts
- **Values** : Configuration personnalisable
- **Templates** : Manifestes dynamiques

**Commandes essentielles** :
```bash
helm repo add    # Ajouter un repository
helm search      # Chercher un chart
helm install     # Installer un chart
helm upgrade     # Mettre à jour
helm rollback    # Revenir en arrière
helm list        # Lister les releases
helm uninstall   # Désinstaller
```

**Bonnes pratiques** :
- Versioning sémantique
- Documentation complète
- Values bien structurés avec commentaires
- Helpers réutilisables
- Tests inclus
- Validation avec schema
- Sécurité (pas de secrets en clair, RBAC minimal)

### Bénéfices immédiats

Avec Helm :
- ✅ Déploiement d'applications complexes en une commande
- ✅ Réutilisation de configurations entre environnements
- ✅ Mises à jour et rollbacks simplifiés
- ✅ Partage de packages avec la communauté
- ✅ Templating puissant pour générer des manifestes
- ✅ Gestion centralisée des versions

---

## Conclusion : Helm dans l'écosystème DevOps

Helm s'intègre parfaitement dans votre workflow DevOps :

```
┌──────────┐    ┌─────────┐    ┌──────────┐    ┌─────────┐    ┌─────────┐
│   Git    │ -> │  CI/CD  │ -> │ Registry │ -> │  Helm   │ -> │ ArgoCD  │
│ (code)   │    │ (build) │    │ (images) │    │ (pack)  │    │ (deploy)│
└──────────┘    └─────────┘    └──────────┘    └─────────┘    └─────────┘
                                                      ↓
                                                ┌──────────┐
                                                │   K8s    │
                                                │ Cluster  │
                                                └──────────┘
```

**Le workflow complet** :
1. Code et manifestes dans Git
2. CI/CD build l'application
3. Image stockée dans le registry
4. Helm package l'application
5. ArgoCD déploie le Helm chart
6. Kubernetes exécute l'application

Helm est la **couche d'abstraction** qui simplifie vos déploiements Kubernetes, rendant vos applications portables, réutilisables et maintenables.

---

## Prochaines étapes

Maintenant que vous maîtrisez Helm, vous êtes prêt pour :

1. **Section 17.7** : Créer vos propres Helm Charts de A à Z
2. **Section 17.8** : Tests automatisés de vos Charts
3. **Section 17.9** : Stratégies de déploiement avancées (Blue-Green, Canary)

Helm est un outil puissant qui va considérablement simplifier votre travail quotidien avec Kubernetes. Combiné avec ArgoCD, vous avez maintenant une infrastructure GitOps complète et professionnelle !

**Prêt à créer votre premier Helm Chart ?** Passons à la section 17.7 : Création de Helm Charts !

⏭️ [Création de Helm Charts](/17-devops-et-cicd/07-creation-de-helm-charts.md)
