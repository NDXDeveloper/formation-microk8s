🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe B : Exemples de Helm Charts

## Introduction à Helm

### Qu'est-ce que Helm ?

Helm est le **gestionnaire de packages** pour Kubernetes. Il peut être comparé à :
- **apt/yum** pour Linux
- **npm** pour Node.js
- **pip** pour Python

Helm vous permet de :
- 📦 Packager des applications Kubernetes complexes
- 🚀 Déployer des applications en une seule commande
- 🔄 Gérer les versions et les mises à jour
- 🔧 Configurer facilement les paramètres
- 🔁 Partager et réutiliser des configurations

### Pourquoi utiliser Helm ?

**Sans Helm**, vous devez gérer de nombreux fichiers YAML séparément :
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f configmap.yaml
kubectl apply -f ingress.yaml
kubectl apply -f secret.yaml
# ... et ainsi de suite
```

**Avec Helm**, une seule commande suffit :
```bash
helm install mon-app ./mon-chart
```

### Concepts Clés

- **Chart** : Un package Helm qui contient tous les fichiers nécessaires pour déployer une application
- **Release** : Une instance d'un Chart déployé dans le cluster
- **Repository** : Un endroit où les Charts sont stockés et partagés
- **Values** : Fichier de configuration pour personnaliser un Chart

---

## Structure d'un Helm Chart

Un Chart Helm a toujours cette structure de répertoire :

```
mon-chart/
├── Chart.yaml              # Métadonnées du Chart (nom, version, description)
├── values.yaml             # Valeurs par défaut pour la configuration
├── charts/                 # Charts dépendants (optionnel)
├── templates/              # Templates Kubernetes
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── _helpers.tpl        # Fonctions helpers réutilisables
│   ├── NOTES.txt           # Instructions post-installation
│   └── tests/              # Tests du Chart
└── .helmignore             # Fichiers à ignorer (comme .gitignore)
```

---

## 1. Chart Simple - Application Web

Créons notre premier Chart pour déployer une application NGINX simple.

### 1.1 Chart.yaml

```yaml
# Métadonnées du Chart
apiVersion: v2  # Version de l'API Helm (v2 pour Helm 3)
name: simple-webapp
description: Un Chart Helm simple pour déployer une application web
type: application  # Type: application ou library
version: 1.0.0  # Version du Chart (SemVer)
appVersion: "1.21"  # Version de l'application déployée

# Informations optionnelles mais recommandées
maintainers:
  - name: Votre Nom
    email: votre.email@example.com
keywords:
  - nginx
  - web
  - frontend
home: https://github.com/votre-repo/simple-webapp
sources:
  - https://github.com/votre-repo/simple-webapp
icon: https://example.com/icon.png
```

### 1.2 values.yaml

```yaml
# Valeurs par défaut du Chart
# Les utilisateurs peuvent surcharger ces valeurs

# Configuration du Deployment
replicaCount: 2

# Configuration de l'image
image:
  repository: nginx
  tag: "1.21"
  pullPolicy: IfNotPresent

# Configuration du Service
service:
  type: ClusterIP
  port: 80

# Configuration de l'Ingress
ingress:
  enabled: false
  className: nginx
  annotations: {}
    # cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: webapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls: []
    # - secretName: webapp-tls
    #   hosts:
    #     - webapp.example.com

# Ressources par défaut
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

# Sondes de santé
livenessProbe:
  httpGet:
    path: /
    port: http
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /
    port: http
  initialDelaySeconds: 5
  periodSeconds: 5

# NodeSelector, tolerations et affinity
nodeSelector: {}
tolerations: []
affinity: {}
```

### 1.3 templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  # Utilisation de fonctions de templating Helm
  name: {{ include "simple-webapp.fullname" . }}
  labels:
    {{- include "simple-webapp.labels" . | nindent 4 }}
spec:
  # Utilisation des valeurs de values.yaml
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "simple-webapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "simple-webapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: 80
          protocol: TCP
        # Sondes de santé depuis values.yaml
        livenessProbe:
          {{- toYaml .Values.livenessProbe | nindent 10 }}
        readinessProbe:
          {{- toYaml .Values.readinessProbe | nindent 10 }}
        # Ressources depuis values.yaml
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
      # Paramètres optionnels
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### 1.4 templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "simple-webapp.fullname" . }}
  labels:
    {{- include "simple-webapp.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "simple-webapp.selectorLabels" . | nindent 4 }}
```

### 1.5 templates/ingress.yaml

```yaml
# Créer l'Ingress seulement si enabled: true
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "simple-webapp.fullname" . }}
  labels:
    {{- include "simple-webapp.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "simple-webapp.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
          {{- end }}
    {{- end }}
{{- end }}
```

### 1.6 templates/_helpers.tpl

```yaml
{{/*
Fonctions helpers réutilisables dans tous les templates
*/}}

{{/*
Nom complet de l'application
*/}}
{{- define "simple-webapp.fullname" -}}
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
Labels communs
*/}}
{{- define "simple-webapp.labels" -}}
helm.sh/chart: {{ include "simple-webapp.chart" . }}
{{ include "simple-webapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Labels de sélection
*/}}
{{- define "simple-webapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "simple-webapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Nom du Chart
*/}}
{{- define "simple-webapp.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Nom de l'application
*/}}
{{- define "simple-webapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}
```

### 1.7 templates/NOTES.txt

```
Merci d'avoir installé {{ .Chart.Name }} !

Votre release s'appelle : {{ .Release.Name }}

Pour accéder à votre application :

{{- if .Values.ingress.enabled }}

1. Via Ingress :
{{- range $host := .Values.ingress.hosts }}
  http{{ if $.Values.ingress.tls }}s{{ end }}://{{ $host.host }}
{{- end }}

{{- else if contains "NodePort" .Values.service.type }}

1. Obtenez le port NodePort :
   export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "simple-webapp.fullname" . }})
   export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
   echo http://$NODE_IP:$NODE_PORT

{{- else if contains "LoadBalancer" .Values.service.type }}

1. Obtenez l'IP externe :
   export SERVICE_IP=$(kubectl get svc --namespace {{ .Release.Namespace }} {{ include "simple-webapp.fullname" . }} --template "{{"{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"}}")
   echo http://$SERVICE_IP:{{ .Values.service.port }}

{{- else if contains "ClusterIP" .Values.service.type }}

1. Utilisez port-forward :
   export POD_NAME=$(kubectl get pods --namespace {{ .Release.Namespace }} -l "app.kubernetes.io/name={{ include "simple-webapp.name" . }},app.kubernetes.io/instance={{ .Release.Name }}" -o jsonpath="{.items[0].metadata.name}")
   kubectl --namespace {{ .Release.Namespace }} port-forward $POD_NAME 8080:80
   echo "Visitez http://127.0.0.1:8080"

{{- end }}

Pour voir les logs :
  kubectl logs -f deployment/{{ include "simple-webapp.fullname" . }} -n {{ .Release.Namespace }}

Pour supprimer cette release :
  helm uninstall {{ .Release.Name }} -n {{ .Release.Namespace }}
```

### 1.8 Utilisation du Chart

```bash
# Installer le Chart
helm install ma-webapp ./simple-webapp

# Installer avec des valeurs personnalisées
helm install ma-webapp ./simple-webapp \
  --set replicaCount=3 \
  --set image.tag=1.22

# Installer avec un fichier de valeurs personnalisé
helm install ma-webapp ./simple-webapp -f mes-valeurs.yaml

# Mettre à jour une release existante
helm upgrade ma-webapp ./simple-webapp

# Désinstaller
helm uninstall ma-webapp
```

---

## 2. Chart avec Base de Données - Application Full Stack

Ce Chart déploie une application web avec une base de données PostgreSQL.

### 2.1 Chart.yaml

```yaml
apiVersion: v2
name: fullstack-app
description: Application full stack avec base de données
type: application
version: 1.0.0
appVersion: "1.0"

dependencies:
  # Utilisation d'un Chart externe pour PostgreSQL
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
```

### 2.2 values.yaml

```yaml
# Configuration de l'application frontend
frontend:
  enabled: true
  replicaCount: 2
  image:
    repository: mon-frontend
    tag: "1.0"
    pullPolicy: IfNotPresent
  service:
    type: ClusterIP
    port: 80
  resources:
    limits:
      cpu: 200m
      memory: 256Mi
    requests:
      cpu: 100m
      memory: 128Mi

# Configuration de l'application backend
backend:
  enabled: true
  replicaCount: 3
  image:
    repository: mon-backend
    tag: "1.0"
    pullPolicy: IfNotPresent
  service:
    type: ClusterIP
    port: 8080
  env:
    LOG_LEVEL: "info"
    API_TIMEOUT: "30"
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 250m
      memory: 256Mi

# Configuration PostgreSQL (Chart externe)
postgresql:
  enabled: true
  auth:
    # Nom de la base de données
    database: myapp
    # Nom d'utilisateur (sera créé automatiquement)
    username: myapp
    # Mot de passe (à surcharger en production!)
    password: changeme
  primary:
    persistence:
      enabled: true
      size: 8Gi
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
      requests:
        cpu: 250m
        memory: 256Mi

# Configuration de l'Ingress
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
          service: frontend
        - path: /api
          pathType: Prefix
          service: backend
  tls:
    - secretName: myapp-tls
      hosts:
        - myapp.example.com
```

### 2.3 templates/frontend-deployment.yaml

```yaml
{{- if .Values.frontend.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "fullstack-app.fullname" . }}-frontend
  labels:
    {{- include "fullstack-app.labels" . | nindent 4 }}
    app.kubernetes.io/component: frontend
spec:
  replicas: {{ .Values.frontend.replicaCount }}
  selector:
    matchLabels:
      {{- include "fullstack-app.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: frontend
  template:
    metadata:
      labels:
        {{- include "fullstack-app.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: frontend
    spec:
      containers:
      - name: frontend
        image: "{{ .Values.frontend.image.repository }}:{{ .Values.frontend.image.tag }}"
        imagePullPolicy: {{ .Values.frontend.image.pullPolicy }}
        ports:
        - name: http
          containerPort: 80
          protocol: TCP
        env:
        # URL du backend
        - name: BACKEND_URL
          value: "http://{{ include "fullstack-app.fullname" . }}-backend:{{ .Values.backend.service.port }}"
        resources:
          {{- toYaml .Values.frontend.resources | nindent 10 }}
{{- end }}
```

### 2.4 templates/backend-deployment.yaml

```yaml
{{- if .Values.backend.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "fullstack-app.fullname" . }}-backend
  labels:
    {{- include "fullstack-app.labels" . | nindent 4 }}
    app.kubernetes.io/component: backend
spec:
  replicas: {{ .Values.backend.replicaCount }}
  selector:
    matchLabels:
      {{- include "fullstack-app.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: backend
  template:
    metadata:
      labels:
        {{- include "fullstack-app.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: backend
    spec:
      containers:
      - name: backend
        image: "{{ .Values.backend.image.repository }}:{{ .Values.backend.image.tag }}"
        imagePullPolicy: {{ .Values.backend.image.pullPolicy }}
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        env:
        # Variables d'environnement personnalisées
        {{- range $key, $value := .Values.backend.env }}
        - name: {{ $key }}
          value: {{ $value | quote }}
        {{- end }}
        # Configuration de la base de données
        - name: DB_HOST
          value: "{{ include "fullstack-app.fullname" . }}-postgresql"
        - name: DB_PORT
          value: "5432"
        - name: DB_NAME
          value: {{ .Values.postgresql.auth.database | quote }}
        - name: DB_USER
          value: {{ .Values.postgresql.auth.username | quote }}
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ include "fullstack-app.fullname" . }}-postgresql
              key: password
        resources:
          {{- toYaml .Values.backend.resources | nindent 10 }}
{{- end }}
```

### 2.5 templates/frontend-service.yaml

```yaml
{{- if .Values.frontend.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "fullstack-app.fullname" . }}-frontend
  labels:
    {{- include "fullstack-app.labels" . | nindent 4 }}
    app.kubernetes.io/component: frontend
spec:
  type: {{ .Values.frontend.service.type }}
  ports:
    - port: {{ .Values.frontend.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "fullstack-app.selectorLabels" . | nindent 4 }}
    app.kubernetes.io/component: frontend
{{- end }}
```

### 2.6 templates/backend-service.yaml

```yaml
{{- if .Values.backend.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "fullstack-app.fullname" . }}-backend
  labels:
    {{- include "fullstack-app.labels" . | nindent 4 }}
    app.kubernetes.io/component: backend
spec:
  type: {{ .Values.backend.service.type }}
  ports:
    - port: {{ .Values.backend.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "fullstack-app.selectorLabels" . | nindent 4 }}
    app.kubernetes.io/component: backend
{{- end }}
```

### 2.7 templates/ingress.yaml

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "fullstack-app.fullname" . }}
  labels:
    {{- include "fullstack-app.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                {{- if eq .service "frontend" }}
                name: {{ include "fullstack-app.fullname" $ }}-frontend
                port:
                  number: {{ $.Values.frontend.service.port }}
                {{- else if eq .service "backend" }}
                name: {{ include "fullstack-app.fullname" $ }}-backend
                port:
                  number: {{ $.Values.backend.service.port }}
                {{- end }}
          {{- end }}
    {{- end }}
{{- end }}
```

---

## 3. Chart Microservices avec ConfigMap et Secrets

Ce Chart démontre la gestion de configuration complexe pour une architecture microservices.

### 3.1 values.yaml

```yaml
# Configuration globale partagée
global:
  imagePullPolicy: IfNotPresent
  storageClass: microk8s-hostpath

# Microservice de gestion des utilisateurs
userService:
  enabled: true
  replicaCount: 2
  image:
    repository: mon-org/user-service
    tag: "1.0.0"
  service:
    port: 8081
  config:
    logLevel: info
    cacheEnabled: true
    cacheTTL: 300
  secrets:
    jwtSecret: "super-secret-key-change-me"
    databasePassword: "db-password-change-me"

# Microservice de gestion des produits
productService:
  enabled: true
  replicaCount: 3
  image:
    repository: mon-org/product-service
    tag: "1.0.0"
  service:
    port: 8082
  config:
    logLevel: info
    maxProductsPerPage: 50
  secrets:
    apiKey: "product-api-key-change-me"

# Microservice de paiement
paymentService:
  enabled: true
  replicaCount: 2
  image:
    repository: mon-org/payment-service
    tag: "1.0.0"
  service:
    port: 8083
  config:
    logLevel: warn
    timeoutSeconds: 30
  secrets:
    stripeApiKey: "sk_test_xxxxx"
    webhookSecret: "whsec_xxxxx"

# Configuration Redis (cache partagé)
redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: true
    password: "redis-password-change-me"
```

### 3.2 templates/configmap-userservice.yaml

```yaml
{{- if .Values.userService.enabled }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "microservices.fullname" . }}-userservice-config
  labels:
    {{- include "microservices.labels" . | nindent 4 }}
    app.kubernetes.io/component: userservice
data:
  # Configuration de l'application
  application.yml: |
    server:
      port: 8081

    logging:
      level:
        root: {{ .Values.userService.config.logLevel | upper }}

    cache:
      enabled: {{ .Values.userService.config.cacheEnabled }}
      ttl: {{ .Values.userService.config.cacheTTL }}

    redis:
      host: {{ include "microservices.fullname" . }}-redis-master
      port: 6379

    services:
      product:
        url: http://{{ include "microservices.fullname" . }}-productservice:{{ .Values.productService.service.port }}
      payment:
        url: http://{{ include "microservices.fullname" . }}-paymentservice:{{ .Values.paymentService.service.port }}

  # Configuration nginx (si reverse proxy intégré)
  nginx.conf: |
    worker_processes auto;
    events {
      worker_connections 1024;
    }
    http {
      upstream backend {
        server localhost:8081;
      }
      server {
        listen 80;
        location / {
          proxy_pass http://backend;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
        }
      }
    }
{{- end }}
```

### 3.3 templates/secret-userservice.yaml

```yaml
{{- if .Values.userService.enabled }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "microservices.fullname" . }}-userservice-secret
  labels:
    {{- include "microservices.labels" . | nindent 4 }}
    app.kubernetes.io/component: userservice
type: Opaque
stringData:
  # Secrets en clair (Helm les encodera en base64)
  jwt-secret: {{ .Values.userService.secrets.jwtSecret | quote }}
  database-password: {{ .Values.userService.secrets.databasePassword | quote }}
  # Secret généré aléatoirement si non fourni
  session-secret: {{ .Values.userService.secrets.sessionSecret | default (randAlphaNum 32) | quote }}
{{- end }}
```

### 3.4 templates/deployment-userservice.yaml

```yaml
{{- if .Values.userService.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "microservices.fullname" . }}-userservice
  labels:
    {{- include "microservices.labels" . | nindent 4 }}
    app.kubernetes.io/component: userservice
spec:
  replicas: {{ .Values.userService.replicaCount }}
  selector:
    matchLabels:
      {{- include "microservices.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: userservice
  template:
    metadata:
      labels:
        {{- include "microservices.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: userservice
      annotations:
        # Redémarrer si ConfigMap ou Secret change
        checksum/config: {{ include (print $.Template.BasePath "/configmap-userservice.yaml") . | sha256sum }}
        checksum/secret: {{ include (print $.Template.BasePath "/secret-userservice.yaml") . | sha256sum }}
    spec:
      containers:
      - name: userservice
        image: "{{ .Values.userService.image.repository }}:{{ .Values.userService.image.tag }}"
        imagePullPolicy: {{ .Values.global.imagePullPolicy }}
        ports:
        - name: http
          containerPort: 8081
          protocol: TCP
        env:
        # Variables d'environnement depuis Secret
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: {{ include "microservices.fullname" . }}-userservice-secret
              key: jwt-secret
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ include "microservices.fullname" . }}-userservice-secret
              key: database-password
        - name: SESSION_SECRET
          valueFrom:
            secretKeyRef:
              name: {{ include "microservices.fullname" . }}-userservice-secret
              key: session-secret
        # Redis password
        {{- if .Values.redis.enabled }}
        - name: REDIS_PASSWORD
          value: {{ .Values.redis.auth.password | quote }}
        {{- end }}
        volumeMounts:
        # Monter le ConfigMap
        - name: config
          mountPath: /app/config
          readOnly: true
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: config
        configMap:
          name: {{ include "microservices.fullname" . }}-userservice-config
{{- end }}
```

---

## 4. Chart avec Jobs de Migration et Init Containers

Ce Chart montre comment exécuter des tâches d'initialisation.

### 4.1 values.yaml

```yaml
# Configuration de l'application
app:
  image:
    repository: mon-app
    tag: "1.0.0"

  # Job de migration de base de données
  migration:
    enabled: true
    image:
      repository: mon-app-migrations
      tag: "1.0.0"
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
      requests:
        cpu: 100m
        memory: 128Mi

# Configuration de la base de données
database:
  host: postgresql
  port: 5432
  name: mydb
  username: myuser
  password: mypassword

# Init containers
initContainers:
  # Attendre que la base de données soit prête
  waitForDB:
    enabled: true
    image: busybox:1.35
  # Télécharger des ressources
  downloadAssets:
    enabled: true
    image: curlimages/curl:latest
    url: https://example.com/assets.tar.gz
```

### 4.2 templates/migration-job.yaml

```yaml
{{- if .Values.app.migration.enabled }}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "app.fullname" . }}-migration-{{ .Release.Revision }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
    app.kubernetes.io/component: migration
  annotations:
    # Exécuter avant le déploiement principal
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "1"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  # Ne conserver qu'un seul Job réussi
  ttlSecondsAfterFinished: 100
  backoffLimit: 3
  template:
    metadata:
      labels:
        {{- include "app.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: migration
    spec:
      restartPolicy: OnFailure
      initContainers:
      # Attendre que la base de données soit disponible
      - name: wait-for-db
        image: {{ .Values.initContainers.waitForDB.image }}
        command:
        - sh
        - -c
        - |
          echo "Attente de la base de données..."
          until nc -z -v -w30 {{ .Values.database.host }} {{ .Values.database.port }}
          do
            echo "Attente de {{ .Values.database.host }}:{{ .Values.database.port }}..."
            sleep 5
          done
          echo "Base de données accessible!"
      containers:
      - name: migration
        image: "{{ .Values.app.migration.image.repository }}:{{ .Values.app.migration.image.tag }}"
        command:
        - sh
        - -c
        - |
          echo "Début des migrations..."
          # Exemple avec Flyway
          flyway migrate
          echo "Migrations terminées avec succès!"
        env:
        - name: DB_HOST
          value: {{ .Values.database.host | quote }}
        - name: DB_PORT
          value: {{ .Values.database.port | quote }}
        - name: DB_NAME
          value: {{ .Values.database.name | quote }}
        - name: DB_USER
          value: {{ .Values.database.username | quote }}
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ include "app.fullname" . }}-db-secret
              key: password
        resources:
          {{- toYaml .Values.app.migration.resources | nindent 10 }}
{{- end }}
```

### 4.3 templates/deployment-with-init.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.fullname" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
spec:
  replicas: 3
  selector:
    matchLabels:
      {{- include "app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "app.selectorLabels" . | nindent 8 }}
    spec:
      # Init Containers - s'exécutent avant le conteneur principal
      initContainers:

      # 1. Attendre la base de données
      {{- if .Values.initContainers.waitForDB.enabled }}
      - name: wait-for-database
        image: {{ .Values.initContainers.waitForDB.image }}
        command:
        - sh
        - -c
        - |
          until nc -z {{ .Values.database.host }} {{ .Values.database.port }}; do
            echo "Attente de la base de données..."
            sleep 2
          done
          echo "Base de données disponible!"
      {{- end }}

      # 2. Télécharger des assets
      {{- if .Values.initContainers.downloadAssets.enabled }}
      - name: download-assets
        image: {{ .Values.initContainers.downloadAssets.image }}
        command:
        - sh
        - -c
        - |
          echo "Téléchargement des assets..."
          curl -L {{ .Values.initContainers.downloadAssets.url }} -o /assets/assets.tar.gz
          cd /assets && tar -xzf assets.tar.gz
          echo "Assets téléchargés!"
        volumeMounts:
        - name: assets
          mountPath: /assets
      {{- end }}

      # 3. Vérifier la configuration
      - name: check-config
        image: busybox:1.35
        command:
        - sh
        - -c
        - |
          echo "Vérification de la configuration..."
          if [ ! -f /config/application.yml ]; then
            echo "ERREUR: Fichier de configuration manquant!"
            exit 1
          fi
          echo "Configuration valide!"
        volumeMounts:
        - name: config
          mountPath: /config

      # Conteneur principal de l'application
      containers:
      - name: app
        image: "{{ .Values.app.image.repository }}:{{ .Values.app.image.tag }}"
        ports:
        - name: http
          containerPort: 8080
        env:
        - name: DB_HOST
          value: {{ .Values.database.host | quote }}
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ include "app.fullname" . }}-db-secret
              key: password
        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
        - name: assets
          mountPath: /app/assets
          readOnly: true

      # Volumes partagés
      volumes:
      - name: config
        configMap:
          name: {{ include "app.fullname" . }}-config
      - name: assets
        emptyDir: {}
```

---

## 5. Chart avec HPA et Resource Management

Ce Chart montre la gestion avancée des ressources et l'autoscaling.

### 5.1 values.yaml

```yaml
app:
  replicaCount: 2
  image:
    repository: mon-app
    tag: "1.0.0"

  # Ressources détaillées par environnement
  resources:
    limits:
      cpu: 1000m
      memory: 1Gi
    requests:
      cpu: 500m
      memory: 512Mi

# Configuration HPA (Horizontal Pod Autoscaler)
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  # Métriques CPU
  targetCPUUtilizationPercentage: 70
  # Métriques mémoire
  targetMemoryUtilizationPercentage: 80
  # Comportement de scaling
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
      - type: Pods
        value: 4
        periodSeconds: 30

# PodDisruptionBudget pour haute disponibilité
podDisruptionBudget:
  enabled: true
  minAvailable: 1
  # OU
  # maxUnavailable: 1

# Stratégie de mise à jour
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0

# Configuration des sondes
healthCheck:
  liveness:
    path: /healthz
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 3
  readiness:
    path: /ready
    initialDelaySeconds: 5
    periodSeconds: 5
    timeoutSeconds: 3
    failureThreshold: 3
```

### 5.2 templates/hpa.yaml

```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "app.fullname" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "app.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
  {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
  {{- end }}
  {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
  {{- end }}
  {{- with .Values.autoscaling.behavior }}
  behavior:
    {{- toYaml . | nindent 4 }}
  {{- end }}
{{- end }}
```

### 5.3 templates/poddisruptionbudget.yaml

```yaml
{{- if .Values.podDisruptionBudget.enabled }}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "app.fullname" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
spec:
  {{- if .Values.podDisruptionBudget.minAvailable }}
  minAvailable: {{ .Values.podDisruptionBudget.minAvailable }}
  {{- end }}
  {{- if .Values.podDisruptionBudget.maxUnavailable }}
  maxUnavailable: {{ .Values.podDisruptionBudget.maxUnavailable }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "app.selectorLabels" . | nindent 6 }}
{{- end }}
```

### 5.4 templates/deployment-optimized.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.fullname" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.app.replicaCount }}
  {{- end }}
  strategy:
    {{- toYaml .Values.updateStrategy | nindent 4 }}
  selector:
    matchLabels:
      {{- include "app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "app.selectorLabels" . | nindent 8 }}
    spec:
      # Anti-affinity pour distribuer les pods
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  {{- include "app.selectorLabels" . | nindent 18 }}
              topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: "{{ .Values.app.image.repository }}:{{ .Values.app.image.tag }}"
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        # Sondes de santé optimisées
        livenessProbe:
          httpGet:
            path: {{ .Values.healthCheck.liveness.path }}
            port: http
          initialDelaySeconds: {{ .Values.healthCheck.liveness.initialDelaySeconds }}
          periodSeconds: {{ .Values.healthCheck.liveness.periodSeconds }}
          timeoutSeconds: {{ .Values.healthCheck.liveness.timeoutSeconds }}
          failureThreshold: {{ .Values.healthCheck.liveness.failureThreshold }}
        readinessProbe:
          httpGet:
            path: {{ .Values.healthCheck.readiness.path }}
            port: http
          initialDelaySeconds: {{ .Values.healthCheck.readiness.initialDelaySeconds }}
          periodSeconds: {{ .Values.healthCheck.readiness.periodSeconds }}
          timeoutSeconds: {{ .Values.healthCheck.readiness.timeoutSeconds }}
          failureThreshold: {{ .Values.healthCheck.readiness.failureThreshold }}
        # Ressources définies précisément
        resources:
          {{- toYaml .Values.app.resources | nindent 10 }}
        # Gestion du cycle de vie
        lifecycle:
          preStop:
            exec:
              command:
              - sh
              - -c
              - |
                # Attendre que les connexions se terminent proprement
                sleep 15
```

---

## 6. Chart Multi-Environnements

Structure pour gérer plusieurs environnements (dev, staging, production).

### 6.1 Structure des fichiers

```
mon-chart/
├── Chart.yaml
├── values.yaml                    # Valeurs communes
├── values-dev.yaml               # Surcharges pour dev
├── values-staging.yaml           # Surcharges pour staging
├── values-production.yaml        # Surcharges pour production
└── templates/
    └── ...
```

### 6.2 values.yaml (commun)

```yaml
# Valeurs communes à tous les environnements
app:
  name: myapp
  image:
    repository: mon-org/myapp
    pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /

monitoring:
  enabled: true
  prometheus:
    scrape: true
```

### 6.3 values-dev.yaml

```yaml
# Surcharges pour l'environnement de développement
environment: development

app:
  replicaCount: 1
  image:
    tag: "dev-latest"
  resources:
    limits:
      cpu: 200m
      memory: 256Mi
    requests:
      cpu: 100m
      memory: 128Mi

database:
  host: postgresql-dev
  name: myapp_dev

ingress:
  enabled: true
  hosts:
    - host: myapp-dev.example.com
      paths:
        - path: /
          pathType: Prefix
  tls: []  # Pas de TLS en dev

autoscaling:
  enabled: false

# Logs plus verbeux en dev
logging:
  level: debug

# Désactiver le cache pour le développement
cache:
  enabled: false
```

### 6.4 values-staging.yaml

```yaml
# Surcharges pour l'environnement de staging
environment: staging

app:
  replicaCount: 2
  image:
    tag: "staging-1.0.0"
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 250m
      memory: 256Mi

database:
  host: postgresql-staging
  name: myapp_staging

ingress:
  enabled: true
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
  hosts:
    - host: myapp-staging.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: myapp-staging-tls
      hosts:
        - myapp-staging.example.com

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70

logging:
  level: info

cache:
  enabled: true
  ttl: 300
```

### 6.5 values-production.yaml

```yaml
# Surcharges pour l'environnement de production
environment: production

app:
  replicaCount: 5
  image:
    tag: "1.0.0"  # Tag fixe en production
    pullPolicy: Always
  resources:
    limits:
      cpu: 1000m
      memory: 1Gi
    requests:
      cpu: 500m
      memory: 512Mi

database:
  host: postgresql-prod-primary
  name: myapp_production

ingress:
  enabled: true
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: myapp-production-tls
      hosts:
        - myapp.example.com

autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 20
  targetCPUUtilizationPercentage: 60
  targetMemoryUtilizationPercentage: 70

podDisruptionBudget:
  enabled: true
  minAvailable: 3

logging:
  level: warn

cache:
  enabled: true
  ttl: 3600

# Backup automatique en production
backup:
  enabled: true
  schedule: "0 2 * * *"
  retention: 30
```

### 6.6 Déploiement selon l'environnement

```bash
# Développement
helm install myapp ./mon-chart -f values-dev.yaml -n dev

# Staging
helm install myapp ./mon-chart -f values-staging.yaml -n staging

# Production
helm install myapp ./mon-chart -f values-production.yaml -n production

# Mise à jour en production avec confirmation
helm upgrade myapp ./mon-chart \
  -f values-production.yaml \
  -n production \
  --wait \
  --timeout 10m
```

---

## 7. Fonctions et Techniques Avancées

### 7.1 Utilisation de range et boucles

```yaml
# Dans templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: multi-env-config
data:
  {{- range $key, $value := .Values.environment }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}

  # Créer plusieurs clés
  {{- range .Values.servers }}
  server-{{ .name }}: {{ .url | quote }}
  {{- end }}
```

```yaml
# Dans values.yaml
environment:
  APP_ENV: production
  LOG_LEVEL: info
  CACHE_ENABLED: true

servers:
  - name: api
    url: http://api.example.com
  - name: auth
    url: http://auth.example.com
```

### 7.2 Conditions complexes

```yaml
{{- if and .Values.ingress.enabled (eq .Values.environment "production") }}
# Créer l'Ingress seulement si enabled ET en production
apiVersion: networking.k8s.io/v1
kind: Ingress
# ...
{{- end }}

{{- if or (eq .Values.database.type "postgresql") (eq .Values.database.type "mysql") }}
# Utiliser un Secret seulement pour PostgreSQL ou MySQL
apiVersion: v1
kind: Secret
# ...
{{- end }}
```

### 7.3 Fonctions de manipulation de chaînes

```yaml
# Majuscules/minuscules
{{ .Values.environment | upper }}  # PRODUCTION
{{ .Values.environment | lower }}  # production

# Remplacement
{{ .Values.url | replace "http://" "https://" }}

# Troncature
{{ .Values.longName | trunc 63 | trimSuffix "-" }}

# Valeur par défaut
{{ .Values.optionalField | default "valeur-par-defaut" }}

# Quote (ajouter des guillemets)
{{ .Values.text | quote }}
```

### 7.4 Génération de valeurs aléatoires

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: random-secret
stringData:
  # Générer un mot de passe aléatoire de 32 caractères
  password: {{ randAlphaNum 32 | quote }}

  # Générer un UUID
  session-id: {{ uuidv4 | quote }}

  # Nombre aléatoire
  pin: {{ randNumeric 4 | quote }}
```

### 7.5 Inclure d'autres fichiers

```yaml
# Dans templates/_config-snippet.tpl
{{- define "app.configSnippet" -}}
database:
  host: {{ .Values.database.host }}
  port: {{ .Values.database.port }}
{{- end -}}

# Dans templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  config.yml: |
    {{- include "app.configSnippet" . | nindent 4 }}
```

---

## 8. Commandes Helm Essentielles

### 8.1 Installation et gestion

```bash
# Installer un Chart
helm install [NOM-RELEASE] [CHART]
helm install myapp ./mon-chart

# Installer avec namespace
helm install myapp ./mon-chart -n production --create-namespace

# Installer avec valeurs personnalisées
helm install myapp ./mon-chart -f values-custom.yaml
helm install myapp ./mon-chart --set replicaCount=3

# Mode dry-run (simulation)
helm install myapp ./mon-chart --dry-run --debug

# Mettre à jour une release
helm upgrade myapp ./mon-chart
helm upgrade myapp ./mon-chart -f values-new.yaml

# Installer ou mettre à jour
helm upgrade --install myapp ./mon-chart

# Rollback
helm rollback myapp 1  # Revenir à la révision 1
helm rollback myapp 0  # Revenir à la révision précédente
```

### 8.2 Gestion des releases

```bash
# Lister les releases
helm list
helm list -n production
helm list --all-namespaces

# Voir l'historique
helm history myapp

# Voir le status
helm status myapp

# Obtenir les valeurs d'une release
helm get values myapp
helm get values myapp --revision 2

# Obtenir le manifeste complet
helm get manifest myapp

# Désinstaller
helm uninstall myapp
helm uninstall myapp --keep-history  # Garder l'historique
```

### 8.3 Développement de Charts

```bash
# Créer un nouveau Chart
helm create mon-chart

# Valider un Chart
helm lint mon-chart

# Template (générer les manifestes)
helm template myapp ./mon-chart
helm template myapp ./mon-chart -f values-prod.yaml

# Packager un Chart
helm package mon-chart
# Crée: mon-chart-1.0.0.tgz

# Vérifier les dépendances
helm dependency list mon-chart
helm dependency update mon-chart
```

### 8.4 Repositories

```bash
# Ajouter un repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add stable https://charts.helm.sh/stable

# Mettre à jour les repos
helm repo update

# Lister les repos
helm repo list

# Chercher un Chart
helm search repo nginx
helm search repo bitnami/postgresql

# Supprimer un repo
helm repo remove bitnami
```

---

## 9. Bonnes Pratiques

### 9.1 Structure et Organisation

✅ **Recommandations** :
1. Un Chart par application/service
2. Utiliser des sous-charts pour les dépendances
3. Séparer les valeurs par environnement
4. Versionner les Charts avec SemVer
5. Documenter dans Chart.yaml et NOTES.txt

### 9.2 Sécurité

✅ **Recommandations** :
1. Ne jamais committer de secrets en clair
2. Utiliser `.helmignore` pour exclure des fichiers sensibles
3. Valider les inputs utilisateur
4. Définir des limits et requests
5. Utiliser des ServiceAccounts dédiés

```yaml
# .helmignore
*.tmp
*.backup
.git/
secrets/
*.key
*.pem
```

### 9.3 Performance

✅ **Recommandations** :
1. Utiliser des checksums pour redémarrer les pods lors de changements de config
2. Définir des sondes appropriées
3. Utiliser l'anti-affinity pour distribuer les pods
4. Configurer le HPA pour l'autoscaling
5. Limiter les ressources

### 9.4 Maintenance

✅ **Recommandations** :
1. Tester avec `helm lint` et `helm template`
2. Utiliser `--dry-run` avant les déploiements
3. Maintenir un CHANGELOG.md
4. Taguer les versions d'images
5. Documenter les breaking changes

### 9.5 Template Checklist

```yaml
# Checklist pour un bon template
✅ apiVersion et kind définis
✅ Métadonnées avec labels et annotations
✅ Utilisation de fonctions helpers (_helpers.tpl)
✅ Valeurs paramétrables dans values.yaml
✅ Conditions pour ressources optionnelles
✅ Ressources requests/limits définies
✅ Sondes liveness et readiness
✅ Commentaires pour expliquer la logique
✅ Gestion des secrets sécurisée
✅ Tests inclus
```

---

## 10. Dépannage

### 10.1 Problèmes Courants

**Erreur: Chart.yaml manquant**
```bash
# Solution: Vérifier la structure du répertoire
ls -la mon-chart/
```

**Erreur de template**
```bash
# Debug avec --debug
helm install myapp ./mon-chart --dry-run --debug

# Vérifier la syntaxe
helm lint mon-chart
```

**Release bloquée**
```bash
# Voir le status
helm status myapp

# Forcer la suppression
helm uninstall myapp --no-hooks

# Nettoyer une release échouée
kubectl delete secret sh.helm.release.v1.myapp.v1 -n default
```

### 10.2 Validation

```bash
# Valider avant installation
helm template myapp ./mon-chart | kubectl apply --dry-run=server -f -

# Vérifier les valeurs calculées
helm template myapp ./mon-chart --debug > output.yaml
cat output.yaml
```

---

## Conclusion

Les Helm Charts sont un outil puissant pour :
- 📦 Simplifier les déploiements Kubernetes
- 🔄 Standardiser les configurations
- 🚀 Accélérer les mises à jour
- 🔧 Gérer plusieurs environnements
- 📚 Partager des applications packagées

**Prochaines étapes** :
1. Créez votre premier Chart simple
2. Ajoutez progressivement de la complexité
3. Partagez vos Charts dans un repository
4. Explorez les Charts communautaires (Bitnami, Stable)
5. Automatisez vos déploiements avec CI/CD

Pour aller plus loin :
- Documentation Helm : https://helm.sh/docs/
- Artifact Hub (Charts publics) : https://artifacthub.io/
- Helm Chart best practices : https://helm.sh/docs/chart_best_practices/

Bon développement avec Helm! 🚀

⏭️ [Exemples de pipelines CI/CD](/annexes/annexe-b-templates-et-exemples/exemples-de-pipelines-cicd.md)
