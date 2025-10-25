🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.7 Création de Helm Charts

## Introduction

Dans la section précédente, vous avez appris à utiliser des Helm Charts existants. Maintenant, il est temps de passer à l'étape suivante : **créer vos propres Charts** pour packager vos applications personnalisées.

Créer un Helm Chart peut sembler intimidant au premier abord, mais c'est en réalité un processus logique et structuré. Dans ce chapitre, nous allons construire ensemble un Chart complet, étape par étape, en partant d'une application simple pour progressivement ajouter toutes les fonctionnalités nécessaires à un Chart professionnel.

À la fin de ce chapitre, vous serez capable de créer des Charts réutilisables, maintenables et distribuables pour toutes vos applications Kubernetes.

---

## Créer votre premier Chart

### Initialiser un nouveau Chart

Helm fournit une commande pour créer la structure de base d'un Chart :

```bash
# Créer un nouveau chart
helm create mon-app

# Examiner la structure créée
tree mon-app/
```

**Structure générée** :

```
mon-app/
├── .helmignore           # Fichiers à ignorer lors du packaging
├── Chart.yaml            # Métadonnées du chart
├── values.yaml           # Valeurs par défaut
├── charts/               # Charts dépendants (vide initialement)
└── templates/            # Templates Kubernetes
    ├── NOTES.txt         # Message affiché après installation
    ├── _helpers.tpl      # Fonctions helper réutilisables
    ├── deployment.yaml   # Template de Deployment
    ├── hpa.yaml          # Horizontal Pod Autoscaler
    ├── ingress.yaml      # Template d'Ingress
    ├── service.yaml      # Template de Service
    ├── serviceaccount.yaml  # Service Account
    └── tests/
        └── test-connection.yaml  # Test de connexion
```

### Comprendre les fichiers générés

#### Chart.yaml

Le fichier `Chart.yaml` contient les métadonnées du Chart :

```yaml
apiVersion: v2              # Version de l'API Helm (v2 pour Helm 3)
name: mon-app               # Nom du chart
description: A Helm chart for Kubernetes
type: application           # Type: application ou library

# Version du chart (incrémentée à chaque modification)
version: 0.1.0

# Version de l'application packagée (version de votre app)
appVersion: "1.0.0"
```

#### values.yaml

Le fichier `values.yaml` contient les valeurs configurables par défaut :

```yaml
# Nombre de replicas
replicaCount: 1

# Configuration de l'image
image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: ""  # Par défaut, utilise appVersion du Chart.yaml

# Service
service:
  type: ClusterIP
  port: 80

# Ingress
ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

# Ressources
resources: {}

# Autoscaling
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
```

### Première installation

Testons le Chart généré tel quel :

```bash
# Installer depuis le dossier local
helm install my-test ./mon-app

# Vérifier l'installation
helm list
kubectl get all
```

**Résultat** : Un déploiement nginx basique est créé !

---

## Personnaliser le Chart pour votre application

### Scénario : Application web Python Flask

Imaginons que nous voulons créer un Chart pour une application Flask simple.

#### Étape 1 : Modifier Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2
name: flask-app
description: Chart Helm pour application Flask
type: application

version: 1.0.0
appVersion: "2.0.1"

keywords:
  - flask
  - python
  - web

maintainers:
  - name: Jean Dupont
    email: jean@example.com

home: https://github.com/mon-org/flask-app
sources:
  - https://github.com/mon-org/flask-app
```

#### Étape 2 : Adapter values.yaml

```yaml
# values.yaml

# Nombre de replicas
replicaCount: 2

# Configuration de l'image
image:
  repository: localhost:32000/flask-app
  pullPolicy: IfNotPresent
  tag: "v2.0.1"

# Pull secrets pour registry privé (optionnel)
imagePullSecrets: []

# Nom du service account
serviceAccount:
  create: true
  name: ""

# Service
service:
  type: ClusterIP
  port: 80
  targetPort: 5000  # Port de l'application Flask

# Ingress
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: flask-app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: flask-app-tls
      hosts:
        - flask-app.example.com

# Ressources recommandées
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
  targetCPUUtilizationPercentage: 75

# Variables d'environnement
env:
  - name: FLASK_ENV
    value: production
  - name: LOG_LEVEL
    value: info

# ConfigMap pour configurations
config:
  debug: false
  maxConnections: 100

# Probes de santé
livenessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 5000
  initialDelaySeconds: 5
  periodSeconds: 5
```

#### Étape 3 : Modifier le template de Deployment

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "flask-app.fullname" . }}
  labels:
    {{- include "flask-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "flask-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "flask-app.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "flask-app.serviceAccountName" . }}
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort }}
          protocol: TCP
        env:
        {{- range .Values.env }}
        - name: {{ .name }}
          value: {{ .value | quote }}
        {{- end }}
        - name: CONFIG_PATH
          value: /config/app-config.yaml
        {{- if .Values.livenessProbe }}
        livenessProbe:
          {{- toYaml .Values.livenessProbe | nindent 10 }}
        {{- end }}
        {{- if .Values.readinessProbe }}
        readinessProbe:
          {{- toYaml .Values.readinessProbe | nindent 10 }}
        {{- end }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        volumeMounts:
        - name: config
          mountPath: /config
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: {{ include "flask-app.fullname" . }}-config
```

#### Étape 4 : Créer un ConfigMap

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "flask-app.fullname" . }}-config
  labels:
    {{- include "flask-app.labels" . | nindent 4 }}
data:
  app-config.yaml: |
    debug: {{ .Values.config.debug }}
    max_connections: {{ .Values.config.maxConnections }}
    {{- if .Values.config.features }}
    features:
      {{- toYaml .Values.config.features | nindent 6 }}
    {{- end }}
```

#### Étape 5 : Ajuster le Service

```yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "flask-app.fullname" . }}
  labels:
    {{- include "flask-app.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
  - port: {{ .Values.service.port }}
    targetPort: http
    protocol: TCP
    name: http
  selector:
    {{- include "flask-app.selectorLabels" . | nindent 4 }}
```

---

## Créer des Helpers réutilisables

Le fichier `_helpers.tpl` contient des fonctions helper pour éviter la duplication.

### Helpers de base

```yaml
{{/* templates/_helpers.tpl */}}

{{/*
Expand the name of the chart.
*/}}
{{- define "flask-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "flask-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "flask-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "flask-app.labels" -}}
helm.sh/chart: {{ include "flask-app.chart" . }}
{{ include "flask-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "flask-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "flask-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "flask-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "flask-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

### Helpers avancés

```yaml
{{/* templates/_helpers.tpl (suite) */}}

{{/*
Generate image string
*/}}
{{- define "flask-app.image" -}}
{{- $tag := .Values.image.tag | default .Chart.AppVersion -}}
{{- printf "%s:%s" .Values.image.repository $tag -}}
{{- end }}

{{/*
Generate environment variables from a map
*/}}
{{- define "flask-app.envVars" -}}
{{- range $key, $value := . }}
- name: {{ $key }}
  value: {{ $value | quote }}
{{- end }}
{{- end }}

{{/*
Return the appropriate apiVersion for HPA
*/}}
{{- define "flask-app.hpa.apiVersion" -}}
{{- if .Capabilities.APIVersions.Has "autoscaling/v2" }}
{{- print "autoscaling/v2" }}
{{- else }}
{{- print "autoscaling/v2beta2" }}
{{- end }}
{{- end }}

{{/*
Validate that required values are set
*/}}
{{- define "flask-app.validateValues" -}}
{{- if not .Values.image.repository }}
  {{- fail "image.repository is required" }}
{{- end }}
{{- if and .Values.ingress.enabled (not .Values.ingress.hosts) }}
  {{- fail "ingress.hosts is required when ingress is enabled" }}
{{- end }}
{{- end }}
```

---

## Ajouter des ressources supplémentaires

### Secret pour données sensibles

```yaml
# templates/secret.yaml
{{- if .Values.secrets }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "flask-app.fullname" . }}-secrets
  labels:
    {{- include "flask-app.labels" . | nindent 4 }}
type: Opaque
data:
  {{- range $key, $value := .Values.secrets }}
  {{ $key }}: {{ $value | b64enc | quote }}
  {{- end }}
{{- end }}
```

**Dans values.yaml** :

```yaml
# Secrets (NE PAS commiter de vraies valeurs)
secrets: {}
  # database-password: ""
  # api-key: ""
```

### PersistentVolumeClaim

```yaml
# templates/pvc.yaml
{{- if .Values.persistence.enabled }}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "flask-app.fullname" . }}-data
  labels:
    {{- include "flask-app.labels" . | nindent 4 }}
spec:
  accessModes:
    - {{ .Values.persistence.accessMode | default "ReadWriteOnce" }}
  resources:
    requests:
      storage: {{ .Values.persistence.size | default "8Gi" }}
  {{- if .Values.persistence.storageClass }}
  storageClassName: {{ .Values.persistence.storageClass }}
  {{- end }}
{{- end }}
```

**Dans values.yaml** :

```yaml
# Persistence
persistence:
  enabled: false
  accessMode: ReadWriteOnce
  size: 10Gi
  storageClass: ""  # Utilise la StorageClass par défaut
```

### NetworkPolicy

```yaml
# templates/networkpolicy.yaml
{{- if .Values.networkPolicy.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "flask-app.fullname" . }}
  labels:
    {{- include "flask-app.labels" . | nindent 4 }}
spec:
  podSelector:
    matchLabels:
      {{- include "flask-app.selectorLabels" . | nindent 6 }}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    {{- if .Values.networkPolicy.ingress.allowExternal }}
    - namespaceSelector: {}
    {{- else }}
    - podSelector:
        matchLabels:
          {{ include "flask-app.fullname" . }}-client: "true"
    {{- end }}
    ports:
    - protocol: TCP
      port: {{ .Values.service.targetPort }}
  egress:
  {{- range .Values.networkPolicy.egress }}
  - to:
    {{- if .namespaceSelector }}
    - namespaceSelector:
        matchLabels:
          {{- toYaml .namespaceSelector | nindent 10 }}
    {{- end }}
    {{- if .podSelector }}
    - podSelector:
        matchLabels:
          {{- toYaml .podSelector | nindent 10 }}
    {{- end }}
    ports:
    {{- range .ports }}
    - protocol: {{ .protocol }}
      port: {{ .port }}
    {{- end }}
  {{- end }}
{{- end }}
```

**Dans values.yaml** :

```yaml
# Network Policy
networkPolicy:
  enabled: false
  ingress:
    allowExternal: true
  egress:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      ports:
        - protocol: TCP
          port: 53  # DNS
        - protocol: UDP
          port: 53
```

### ServiceMonitor pour Prometheus

```yaml
# templates/servicemonitor.yaml
{{- if and .Values.metrics.enabled .Values.metrics.serviceMonitor.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ include "flask-app.fullname" . }}
  labels:
    {{- include "flask-app.labels" . | nindent 4 }}
    {{- with .Values.metrics.serviceMonitor.labels }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
spec:
  selector:
    matchLabels:
      {{- include "flask-app.selectorLabels" . | nindent 6 }}
  endpoints:
  - port: http
    path: {{ .Values.metrics.path }}
    interval: {{ .Values.metrics.serviceMonitor.interval }}
    {{- if .Values.metrics.serviceMonitor.scrapeTimeout }}
    scrapeTimeout: {{ .Values.metrics.serviceMonitor.scrapeTimeout }}
    {{- end }}
{{- end }}
```

**Dans values.yaml** :

```yaml
# Métriques Prometheus
metrics:
  enabled: false
  path: /metrics
  serviceMonitor:
    enabled: false
    interval: 30s
    scrapeTimeout: 10s
    labels: {}
```

---

## Gestion des environnements multiples

### Approche avec fichiers values séparés

Créez des fichiers values pour chaque environnement :

```
flask-app/
├── values.yaml              # Valeurs par défaut
├── values-dev.yaml          # Development
├── values-staging.yaml      # Staging
└── values-production.yaml   # Production
```

**values-dev.yaml** :

```yaml
# Environnement de développement
replicaCount: 1

image:
  tag: "latest"
  pullPolicy: Always

ingress:
  enabled: true
  hosts:
    - host: flask-app-dev.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

env:
  - name: FLASK_ENV
    value: development
  - name: LOG_LEVEL
    value: debug

config:
  debug: true
  maxConnections: 10

autoscaling:
  enabled: false
```

**values-production.yaml** :

```yaml
# Environnement de production
replicaCount: 5

image:
  tag: "v2.0.1"
  pullPolicy: IfNotPresent

ingress:
  enabled: true
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
  hosts:
    - host: flask-app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: flask-app-tls
      hosts:
        - flask-app.example.com

resources:
  limits:
    cpu: 2000m
    memory: 2Gi
  requests:
    cpu: 1000m
    memory: 1Gi

env:
  - name: FLASK_ENV
    value: production
  - name: LOG_LEVEL
    value: warning

config:
  debug: false
  maxConnections: 100

autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

networkPolicy:
  enabled: true

metrics:
  enabled: true
  serviceMonitor:
    enabled: true
```

**Installation selon l'environnement** :

```bash
# Development
helm install flask-app-dev ./flask-app -f ./flask-app/values-dev.yaml -n dev

# Production
helm install flask-app-prod ./flask-app -f ./flask-app/values-production.yaml -n production
```

---

## Conditionnels et logique avancée

### Activer/désactiver des ressources

```yaml
# templates/ingress.yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
# ... reste du template
{{- end }}
```

### Conditions multiples

```yaml
{{- if and .Values.ingress.enabled .Values.ingress.tls }}
tls:
{{- range .Values.ingress.tls }}
- hosts:
  {{- range .hosts }}
  - {{ . | quote }}
  {{- end }}
  secretName: {{ .secretName }}
{{- end }}
{{- end }}
```

### Valeurs par défaut conditionnelles

```yaml
image: {{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}

{{- $serviceType := .Values.service.type | default "ClusterIP" }}
type: {{ $serviceType }}
```

### Logique selon l'environnement

```yaml
# Dans values.yaml
environment: production

# Dans le template
{{- if eq .Values.environment "production" }}
replicas: 5
resources:
  limits:
    cpu: 2000m
{{- else if eq .Values.environment "staging" }}
replicas: 3
resources:
  limits:
    cpu: 1000m
{{- else }}
replicas: 1
resources:
  limits:
    cpu: 500m
{{- end }}
```

---

## NOTES.txt : Instructions post-installation

Le fichier `templates/NOTES.txt` affiche des informations utiles après l'installation.

```
# templates/NOTES.txt
🎉 {{ .Chart.Name }} has been installed successfully!

Chart Version: {{ .Chart.Version }}
App Version: {{ .Chart.AppVersion }}
Release Name: {{ .Release.Name }}
Namespace: {{ .Release.Namespace }}

📊 To check the status:
  kubectl get pods -n {{ .Release.Namespace }} -l app.kubernetes.io/instance={{ .Release.Name }}

{{- if .Values.ingress.enabled }}

🌐 Your application is accessible at:
{{- range .Values.ingress.hosts }}
  - https://{{ .host }}
{{- end }}

{{- else }}

🔗 To access your application:
  export POD_NAME=$(kubectl get pods -n {{ .Release.Namespace }} -l "app.kubernetes.io/name={{ include "flask-app.name" . }},app.kubernetes.io/instance={{ .Release.Name }}" -o jsonpath="{.items[0].metadata.name}")
  kubectl port-forward -n {{ .Release.Namespace }} $POD_NAME 8080:{{ .Values.service.targetPort }}

  Then visit: http://127.0.0.1:8080

{{- end }}

{{- if .Values.metrics.enabled }}

📈 Metrics are available at:
  http://{{ include "flask-app.fullname" . }}:{{ .Values.service.port }}{{ .Values.metrics.path }}

{{- end }}

{{- if .Values.persistence.enabled }}

💾 Persistent storage is enabled:
  Storage Class: {{ .Values.persistence.storageClass | default "default" }}
  Size: {{ .Values.persistence.size }}

{{- end }}

📚 For more information, visit: {{ .Chart.Home }}

{{- if .Values.secrets }}

⚠️  IMPORTANT: Secrets have been created. Make sure to store them securely!
{{- end }}
```

---

## Validation et debugging

### Valider la syntaxe YAML

```bash
# Vérifier la syntaxe sans installer
helm lint ./flask-app

# Résultat :
# ==> Linting flask-app
# [INFO] Chart.yaml: icon is recommended
#
# 1 chart(s) linted, 0 chart(s) failed
```

### Dry-run complet

```bash
# Voir tous les manifestes qui seraient générés
helm install flask-app-test ./flask-app --dry-run --debug

# Avec des values personnalisées
helm install flask-app-test ./flask-app \
  --dry-run --debug \
  -f values-production.yaml \
  --set image.tag=v2.0.2
```

### Générer les templates

```bash
# Générer les manifestes dans un fichier
helm template flask-app ./flask-app > generated-manifests.yaml

# Avec des values spécifiques
helm template flask-app ./flask-app -f values-production.yaml > prod-manifests.yaml

# Puis inspecter
cat generated-manifests.yaml
```

### Débugger un template spécifique

```bash
# Afficher seulement le deployment
helm template flask-app ./flask-app --show-only templates/deployment.yaml

# Avec des values
helm template flask-app ./flask-app \
  --show-only templates/deployment.yaml \
  --set replicaCount=3
```

### Valider avec kubectl

```bash
# Générer et valider avec kubectl
helm template flask-app ./flask-app | kubectl apply --dry-run=client -f -

# Validation serveur (vérifie aussi les quotas, RBAC, etc.)
helm template flask-app ./flask-app | kubectl apply --dry-run=server -f -
```

---

## Packaging et distribution

### Créer un package Chart

```bash
# Packager le chart
helm package ./flask-app

# Résultat :
# Successfully packaged chart and saved it to: flask-app-1.0.0.tgz

# Avec un numéro de version spécifique
helm package ./flask-app --version 1.0.1
```

### Structure du package

Le fichier `.tgz` contient tous les fichiers du chart compressés :

```bash
# Voir le contenu
tar -tzf flask-app-1.0.0.tgz

# Extraire
tar -xzf flask-app-1.0.0.tgz
```

### Créer un repository Helm local

```bash
# Créer un dossier pour le repo
mkdir helm-repo
mv flask-app-1.0.0.tgz helm-repo/

# Générer l'index
helm repo index helm-repo/ --url http://localhost:8080

# Le fichier index.yaml est créé
cat helm-repo/index.yaml
```

**index.yaml** :

```yaml
apiVersion: v1
entries:
  flask-app:
  - apiVersion: v2
    appVersion: 2.0.1
    created: "2025-10-25T15:30:00Z"
    description: Chart Helm pour application Flask
    digest: 1234567890abcdef...
    name: flask-app
    urls:
    - http://localhost:8080/flask-app-1.0.0.tgz
    version: 1.0.0
```

### Servir le repository localement

```bash
# Avec Python
cd helm-repo
python3 -m http.server 8080

# Avec Nginx (si installé)
nginx -c /path/to/nginx.conf
```

### Ajouter votre repository

```bash
# Ajouter le repo local
helm repo add my-repo http://localhost:8080

# Chercher vos charts
helm search repo my-repo

# Installer depuis votre repo
helm install my-flask my-repo/flask-app
```

### Repository GitHub Pages

Vous pouvez héberger gratuitement vos Charts sur GitHub Pages :

```bash
# Dans votre repo Git
mkdir -p docs/charts
helm package ./flask-app -d docs/charts/
helm repo index docs/charts/ --url https://votre-username.github.io/helm-charts/charts/

# Commit et push
git add docs/
git commit -m "Add Helm chart"
git push

# Activer GitHub Pages dans Settings → Pages → Source: docs/

# Ajouter le repo
helm repo add mon-repo https://votre-username.github.io/helm-charts/charts/
```

---

## Tests automatisés

### Tests Helm intégrés

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "flask-app.fullname" . }}-test-connection"
  labels:
    {{- include "flask-app.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  containers:
  - name: wget
    image: busybox
    command: ['wget']
    args: ['{{ include "flask-app.fullname" . }}:{{ .Values.service.port }}/health']
  restartPolicy: Never
```

**Exécuter les tests** :

```bash
# Installer le chart
helm install flask-app-test ./flask-app

# Exécuter les tests
helm test flask-app-test

# Résultat :
# Pod flask-app-test-connection pending
# Pod flask-app-test-connection running
# Pod flask-app-test-connection succeeded
# TEST SUITE: flask-app-test-connection
# Last Started: Sat Oct 25 16:00:00 2025
# Last Completed: Sat Oct 25 16:00:05 2025
# Phase: Succeeded
```

### Tests avancés

```yaml
# templates/tests/test-api.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "flask-app.fullname" . }}-test-api"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
  - name: test
    image: curlimages/curl:latest
    command:
    - sh
    - -c
    - |
      curl -f http://{{ include "flask-app.fullname" . }}:{{ .Values.service.port }}/api/status
      response=$(curl -s http://{{ include "flask-app.fullname" . }}:{{ .Values.service.port }}/api/version)
      if [ "$response" != "{{ .Chart.AppVersion }}" ]; then
        echo "Version mismatch!"
        exit 1
      fi
      echo "All tests passed!"
  restartPolicy: Never
```

---

## Hooks du cycle de vie

Les hooks permettent d'exécuter des actions à des moments précis.

### Pre-install hook

```yaml
# templates/hooks/pre-install-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ include "flask-app.fullname" . }}-pre-install"
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded,hook-failed
spec:
  template:
    spec:
      containers:
      - name: pre-install
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Preparing database..."
          # Logique de préparation
          echo "Database ready!"
      restartPolicy: Never
  backoffLimit: 3
```

### Post-upgrade hook

```yaml
# templates/hooks/post-upgrade-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ include "flask-app.fullname" . }}-post-upgrade"
  annotations:
    "helm.sh/hook": post-upgrade
    "helm.sh/hook-weight": "1"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
      - name: post-upgrade
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Running database migrations..."
          # Logique de migration
          echo "Migrations completed!"
      restartPolicy: Never
```

### Pre-delete hook (cleanup)

```yaml
# templates/hooks/pre-delete-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ include "flask-app.fullname" . }}-cleanup"
  annotations:
    "helm.sh/hook": pre-delete
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
      - name: cleanup
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Backing up data before deletion..."
          # Logique de backup
          echo "Backup completed!"
      restartPolicy: Never
```

**Types de hooks disponibles** :
- `pre-install` : Avant l'installation
- `post-install` : Après l'installation
- `pre-delete` : Avant la suppression
- `post-delete` : Après la suppression
- `pre-upgrade` : Avant la mise à jour
- `post-upgrade` : Après la mise à jour
- `pre-rollback` : Avant le rollback
- `post-rollback` : Après le rollback
- `test` : Pour les tests

---

## Documentation du Chart

### README.md complet

```markdown
# Flask App Helm Chart

Application Flask packagée pour Kubernetes.

## Prérequis

- Kubernetes 1.19+
- Helm 3.8+
- PV provisioner (pour le stockage persistant)

## Installation

### Ajout du repository

```bash
helm repo add my-repo https://charts.example.com
helm repo update
```

### Installation basique

```bash
helm install my-flask-app my-repo/flask-app
```

### Installation personnalisée

```bash
helm install my-flask-app my-repo/flask-app \
  --set replicaCount=3 \
  --set image.tag=v2.1.0 \
  --set ingress.enabled=true \
  --set ingress.hosts[0].host=myapp.example.com
```

### Installation avec fichier values

```bash
helm install my-flask-app my-repo/flask-app -f custom-values.yaml
```

## Configuration

Les paramètres suivants peuvent être configurés :

| Paramètre | Description | Défaut |
|-----------|-------------|--------|
| `replicaCount` | Nombre de replicas | `2` |
| `image.repository` | Repository de l'image | `localhost:32000/flask-app` |
| `image.tag` | Tag de l'image | `appVersion du Chart` |
| `image.pullPolicy` | Politique de pull | `IfNotPresent` |
| `service.type` | Type de service | `ClusterIP` |
| `service.port` | Port du service | `80` |
| `service.targetPort` | Port du conteneur | `5000` |
| `ingress.enabled` | Activer Ingress | `false` |
| `ingress.className` | Classe Ingress | `nginx` |
| `ingress.hosts` | Hosts Ingress | `[]` |
| `resources.limits.cpu` | Limite CPU | `500m` |
| `resources.limits.memory` | Limite mémoire | `512Mi` |
| `autoscaling.enabled` | Activer HPA | `true` |
| `autoscaling.minReplicas` | Replicas min | `2` |
| `autoscaling.maxReplicas` | Replicas max | `10` |
| `persistence.enabled` | Activer stockage persistant | `false` |
| `metrics.enabled` | Activer métriques Prometheus | `false` |

## Exemples

### Environnement de production

```yaml
# values-production.yaml
replicaCount: 5

image:
  tag: "v2.0.1"

ingress:
  enabled: true
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: app-tls
      hosts:
        - app.example.com

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

metrics:
  enabled: true
  serviceMonitor:
    enabled: true
```

### Avec base de données

```yaml
# values-with-db.yaml
persistence:
  enabled: true
  size: 20Gi

env:
  - name: DATABASE_URL
    value: postgresql://postgres:5432/mydb

# Ajouter PostgreSQL comme dépendance
postgresql:
  enabled: true
  auth:
    postgresPassword: MySecretPassword
```

## Mise à jour

```bash
helm upgrade my-flask-app my-repo/flask-app --version 1.1.0
```

## Rollback

```bash
helm rollback my-flask-app
```

## Désinstallation

```bash
helm uninstall my-flask-app
```

## Support

- Documentation : https://docs.example.com
- Issues : https://github.com/org/flask-app/issues
- Contact : support@example.com
```

---

## Bonnes pratiques récapitulatives

### 1. Structure et organisation

✅ **Bien organisé** :
```
flask-app/
├── Chart.yaml
├── values.yaml
├── values.schema.json
├── README.md
├── .helmignore
├── charts/
├── templates/
│   ├── _helpers.tpl
│   ├── NOTES.txt
│   ├── application/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   ├── networking/
│   │   └── ingress.yaml
│   ├── rbac/
│   │   ├── serviceaccount.yaml
│   │   └── role.yaml
│   ├── hooks/
│   │   └── pre-install-job.yaml
│   └── tests/
│       └── test-connection.yaml
```

### 2. Naming conventions

```yaml
# Utilisez les helpers pour la cohérence
name: {{ include "flask-app.fullname" . }}
labels:
  {{- include "flask-app.labels" . | nindent 4 }}
```

### 3. Valeurs par défaut sensées

```yaml
# values.yaml avec des valeurs raisonnables
replicaCount: 2  # Pas 1 (pas de HA) ni 100 (excessif)

resources:
  limits:
    cpu: 500m      # Limite raisonnable
    memory: 512Mi
  requests:
    cpu: 250m      # 50% des limites
    memory: 256Mi
```

### 4. Commentaires explicatifs

```yaml
# values.yaml
# -- Nombre de replicas du déploiement.
# Pour la production, utilisez au moins 2 pour la haute disponibilité.
replicaCount: 2

# -- Configuration de l'image Docker
image:
  # -- Repository de l'image (format: registry/namespace/image)
  repository: localhost:32000/flask-app
  # -- Politique de pull. Options: Always, IfNotPresent, Never
  pullPolicy: IfNotPresent
  # -- Tag de l'image. Par défaut utilise appVersion du Chart.yaml
  tag: ""
```

### 5. Validation des valeurs

```yaml
# Dans un template
{{- if not .Values.image.repository }}
  {{- fail "image.repository est requis" }}
{{- end }}

{{- if and .Values.ingress.enabled (not .Values.ingress.hosts) }}
  {{- fail "ingress.hosts est requis quand ingress est activé" }}
{{- end }}
```

### 6. Sécurité

```yaml
# Security context par défaut
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: true
```

### 7. Flexibility maximale

```yaml
# Permettre la surcharge de tout
{{- with .Values.deployment.annotations }}
annotations:
  {{- toYaml . | nindent 4 }}
{{- end }}

{{- with .Values.deployment.extraEnvVars }}
env:
  {{- toYaml . | nindent 2 }}
{{- end }}
```

---

## Récapitulatif

### Points clés

**Créer un Chart Helm, c'est** :
- Structurer votre application en templates réutilisables
- Définir des values pour la personnalisation
- Créer des helpers pour éviter la duplication
- Documenter pour faciliter l'utilisation
- Tester pour garantir la qualité

**Étapes de création** :
1. `helm create` pour la structure de base
2. Personnaliser Chart.yaml et values.yaml
3. Adapter les templates à votre application
4. Créer des helpers réutilisables
5. Ajouter des ressources supplémentaires
6. Valider avec `helm lint` et `--dry-run`
7. Packager avec `helm package`
8. Distribuer via un repository

**Bonnes pratiques** :
- Structure claire et organisée
- Values par défaut sensés
- Documentation complète
- Tests automatisés
- Hooks pour le cycle de vie
- Validation des valeurs
- Sécurité intégrée
- Flexibilité maximale

### Bénéfices

Avec vos propres Charts Helm :
- ✅ Applications packagées et versionnées
- ✅ Déploiements cohérents entre environnements
- ✅ Configuration centralisée et documentée
- ✅ Réutilisabilité maximale
- ✅ Maintenance simplifiée
- ✅ Partage facilité avec l'équipe

---

## Conclusion

Créer des Helm Charts transforme la façon dont vous gérez vos applications Kubernetes. Ce qui prenait des dizaines de fichiers YAML éparpillés devient un package élégant, versionné et réutilisable.

Vous avez maintenant toutes les connaissances pour créer des Charts professionnels, de la structure de base aux fonctionnalités avancées comme les hooks, les tests et les conditionnels complexes.

**Le cycle complet DevOps avec Helm** :

```
┌─────────┐   ┌────────┐   ┌──────────┐   ┌──────────┐   ┌─────────┐
│   Git   │ → │ CI/CD  │ → │ Registry │ → │   Helm   │ → │ ArgoCD  │
│ (code)  │   │(build) │   │ (image)  │   │ (chart)  │   │ (sync)  │
└─────────┘   └────────┘   └──────────┘   └──────────┘   └─────────┘
                                                  ↓
                                           ┌──────────┐
                                           │   K8s    │
                                           │ Cluster  │
                                           └──────────┘
```

Vos applications sont maintenant :
- **Packagées** : Charts Helm réutilisables
- **Versionnées** : Git + Helm versions
- **Configurables** : Values par environnement
- **Testées** : Tests automatisés
- **Déployables** : ArgoCD synchronisation
- **Maintenues** : Mises à jour et rollbacks faciles

**Félicitations !** Vous maîtrisez maintenant l'ensemble de la chaîne DevOps pour Kubernetes, de l'écriture du code au déploiement automatisé en production !

**Prêt pour les stratégies de déploiement avancées ?** Passons à la section suivante pour découvrir les déploiements Blue-Green et Canary !

⏭️ [Automated testing](/17-devops-et-cicd/08-automated-testing.md)
