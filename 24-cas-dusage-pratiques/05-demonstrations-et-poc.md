🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 24.5 Démonstrations et PoC

## Introduction

Les démonstrations (démos) et les Proof of Concept (PoC) sont des éléments essentiels pour convaincre, former ou valider des concepts techniques. Avec MicroK8s, vous disposez d'une plateforme idéale pour créer des environnements de démonstration impressionnants, reproductibles et rapides à déployer.

Dans ce chapitre, nous allons explorer comment utiliser MicroK8s pour créer des démonstrations percutantes et des PoC convaincants, que ce soit pour présenter Kubernetes à votre équipe, valider une architecture avant un projet, ou impressionner un client.

## Qu'est-ce qu'une Démo et un PoC ?

### Démonstration (Demo)

Une **démonstration** est une présentation pratique qui montre le fonctionnement d'une technologie ou d'une solution. L'objectif est de :

- **Éduquer** : Montrer comment quelque chose fonctionne
- **Impressionner** : Démontrer les capacités d'une solution
- **Convaincre** : Prouver la valeur d'une approche
- **Former** : Enseigner par l'exemple

**Exemples** :
- Présentation de Kubernetes à votre équipe
- Démonstration des capacités d'auto-scaling
- Showcase d'une architecture microservices
- Présentation du déploiement continu

### Proof of Concept (PoC)

Un **PoC** est une implémentation expérimentale qui prouve qu'une idée ou une approche est réalisable. L'objectif est de :

- **Valider** : Confirmer qu'une solution fonctionne
- **Tester** : Évaluer les performances et limitations
- **Comparer** : Mesurer différentes approches
- **Décider** : Fournir des données pour une décision

**Exemples** :
- Tester la migration d'une application vers Kubernetes
- Valider une architecture avant un grand projet
- Comparer différentes solutions de monitoring
- Évaluer la faisabilité d'une solution cloud-native

### Différences Clés

| Aspect | Démonstration | PoC |
|--------|--------------|-----|
| **Durée** | Court (minutes à heures) | Moyen (jours à semaines) |
| **Profondeur** | Surface, vue d'ensemble | Approfondi, détaillé |
| **Objectif** | Montrer, impressionner | Prouver, valider |
| **Audience** | Large (management, équipes) | Technique (architectes, dev) |
| **Données** | Fictives ou simplifiées | Réelles ou représentatives |

## Pourquoi MicroK8s pour les Démos et PoC ?

### Avantages de MicroK8s

**Installation rapide** : Déployé en quelques minutes, parfait pour les démos spontanées.

**Environnement complet** : Toutes les fonctionnalités d'un cluster Kubernetes réel.

**Portable** : Fonctionne sur laptop, serveur, VM, ou cloud.

**Léger** : Consomme peu de ressources, peut tourner en arrière-plan.

**Reproductible** : Les démos fonctionnent de la même manière à chaque fois.

**Addons simples** : Activation de fonctionnalités en une commande.

**Isolation** : Pas de risque pour d'autres environnements.

**Coût zéro** : Pas de frais de cloud pour les démonstrations.

## Préparation de l'Environnement de Démo

### Configuration Système Recommandée

Pour des démos fluides sans ralentissement :

**Configuration Minimale** :
- CPU : 4 cœurs
- RAM : 8 GB
- Stockage : 50 GB
- Bonne connexion réseau (pour télécharger les images)

**Configuration Idéale** :
- CPU : 6-8 cœurs
- RAM : 16 GB
- Stockage : 100 GB SSD
- Écran secondaire (pour présenter)

### Installation et Configuration de Base

```bash
# Installation rapide
sudo snap install microk8s --classic
sudo usermod -a -G microk8s $USER
newgrp microk8s

# Configuration de l'alias
alias kubectl='microk8s kubectl'
echo "alias kubectl='microk8s kubectl'" >> ~/.bashrc

# Activer les addons essentiels pour les démos
microk8s enable dns storage dashboard ingress registry metrics-server

# Vérifier que tout est prêt
microk8s status --wait-ready
```

### Namespace Dédié aux Démos

```bash
# Créer un namespace pour les démos
kubectl create namespace demo

# Le définir comme namespace par défaut
kubectl config set-context --current --namespace=demo

# Créer aussi un namespace pour les PoC
kubectl create namespace poc

# Labels pour identification
kubectl label namespace demo purpose=demonstration environment=showcase
kubectl label namespace poc purpose=proof-of-concept environment=validation
```

### Pre-Pull des Images Courantes

Pour éviter les temps d'attente pendant les démos, téléchargez les images à l'avance :

```bash
# Images couramment utilisées
docker pull nginx:latest
docker pull redis:alpine
docker pull postgres:15
docker pull mongo:latest
docker pull mysql:8.0

# Images pour les démos
docker pull nginx:alpine
docker pull hashicorp/http-echo
docker pull curlimages/curl
docker pull busybox

# Images applicatives
docker pull wordpress:latest
docker pull ghost:latest
docker pull grafana/grafana:latest
docker pull prom/prometheus:latest
```

### Script de Préparation Automatique

Créez un fichier `prepare-demo.sh` :

```bash
#!/bin/bash

echo "🎬 Preparing demo environment..."

# Vérifier que MicroK8s est prêt
microk8s status --wait-ready

# Créer les namespaces si nécessaire
kubectl create namespace demo --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace poc --dry-run=client -o yaml | kubectl apply -f -

# Nettoyer les anciennes démos
kubectl delete all --all -n demo
kubectl delete all --all -n poc

# Pre-pull des images essentielles
echo "📦 Pre-pulling essential images..."
docker pull nginx:alpine
docker pull redis:alpine
docker pull postgres:15

# Vérifier les addons
echo "🔧 Checking addons..."
microk8s status

# Vérifier les ressources disponibles
echo "💻 System resources:"
kubectl top nodes 2>/dev/null || echo "Metrics server not available"

echo "✅ Demo environment ready!"
echo "📊 Dashboard: kubectl port-forward -n kube-system service/kubernetes-dashboard 10443:443"
echo "🚀 Happy demoing!"
```

Rendez-le exécutable :

```bash
chmod +x prepare-demo.sh
./prepare-demo.sh
```

## Scénarios de Démonstration

### Scénario 1 : Déploiement Rapide d'Application

**Objectif** : Montrer la simplicité du déploiement dans Kubernetes

**Durée** : 5 minutes

**Audience** : Managers, équipes qui découvrent Kubernetes

#### Script de Démonstration

```bash
# 1. Montrer l'état du cluster
echo "Voici notre cluster Kubernetes vide..."
kubectl get all -n demo

# 2. Déployer une application web en une commande
echo "Je déploie une application web avec une seule commande..."
kubectl create deployment webapp --image=nginx --replicas=3 -n demo

# 3. Observer la création
echo "Observons la création automatique des pods..."
kubectl get pods -n demo -w
# Arrêtez avec Ctrl+C après quelques secondes

# 4. Exposer l'application
echo "Rendons l'application accessible..."
kubectl expose deployment webapp --port=80 --type=NodePort -n demo

# 5. Obtenir l'URL
echo "L'application est accessible ici:"
kubectl get svc webapp -n demo

# 6. Scaler l'application
echo "Augmentons la capacité à 10 instances..."
kubectl scale deployment webapp --replicas=10 -n demo

# 7. Observer le scaling
kubectl get pods -n demo

# 8. Montrer la résilience
echo "Supprimons un pod, Kubernetes va automatiquement le recréer..."
POD=$(kubectl get pods -n demo -l app=webapp -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $POD -n demo
kubectl get pods -n demo -w
```

**Points à souligner** :
- Déploiement en quelques secondes
- Scaling facile
- Auto-guérison automatique
- Gestion déclarative

### Scénario 2 : Rolling Update Sans Downtime

**Objectif** : Démontrer les mises à jour sans interruption de service

**Durée** : 7 minutes

**Audience** : Équipes DevOps, managers IT

#### Préparation

```yaml
# rolling-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-demo
  namespace: demo
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: rolling-demo
  template:
    metadata:
      labels:
        app: rolling-demo
    spec:
      containers:
      - name: web
        image: hashicorp/http-echo
        args:
        - "-text=Version 1.0"
        - "-listen=:8080"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: rolling-demo
  namespace: demo
spec:
  selector:
    app: rolling-demo
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
    nodePort: 30090
  type: NodePort
```

#### Script de Démonstration

```bash
# 1. Déployer la version 1
echo "Déploiement de la Version 1.0..."
kubectl apply -f rolling-demo.yaml

# Attendre que tout soit prêt
kubectl wait --for=condition=available deployment/rolling-demo -n demo --timeout=120s

# 2. Montrer la version actuelle
echo "Version actuelle de l'application:"
curl http://localhost:30090

# 3. Générer du trafic continu (dans un terminal séparé ou en arrière-plan)
echo "Générons du trafic continu pour voir qu'il n'y a pas d'interruption..."
# Dans un terminal séparé : while true; do curl -s http://localhost:30090; sleep 1; done

# 4. Mettre à jour vers la version 2
echo "Mise à jour vers la Version 2.0..."
kubectl set image deployment/rolling-demo web=hashicorp/http-echo -n demo
kubectl set env deployment/rolling-demo WEB_TEXT="Version 2.0" -n demo

# Alternative : modifier directement les arguments
kubectl patch deployment rolling-demo -n demo --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/args", "value": ["-text=Version 2.0", "-listen=:8080"]}]'

# 5. Observer le rolling update en temps réel
echo "Observons le rolling update..."
kubectl rollout status deployment/rolling-demo -n demo

# 6. Voir l'historique
kubectl rollout history deployment/rolling-demo -n demo

# 7. Rollback si nécessaire
echo "En cas de problème, on peut revenir en arrière instantanément..."
kubectl rollout undo deployment/rolling-demo -n demo
```

**Points à souligner** :
- Zéro downtime pendant la mise à jour
- Rollback instantané si problème
- Contrôle fin du processus de déploiement
- Traffic toujours servi pendant la transition

### Scénario 3 : Auto-Scaling en Action

**Objectif** : Montrer l'adaptation automatique aux charges

**Durée** : 10 minutes

**Audience** : Architectes, équipes techniques

#### Préparation

```yaml
# autoscaling-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
          limits:
            cpu: 500m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  namespace: demo
spec:
  selector:
    app: php-apache
  ports:
  - protocol: TCP
    port: 80
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

#### Script de Démonstration

```bash
# 1. Activer metrics-server si nécessaire
microk8s enable metrics-server

# 2. Déployer l'application avec HPA
echo "Déploiement d'une application avec auto-scaling..."
kubectl apply -f autoscaling-demo.yaml

# Attendre que tout soit prêt
sleep 30

# 3. Montrer l'état initial
echo "État initial - 1 replica:"
kubectl get pods -n demo -l app=php-apache
kubectl get hpa -n demo

# 4. Montrer la consommation CPU actuelle
echo "Utilisation CPU actuelle:"
kubectl top pods -n demo -l app=php-apache

# 5. Générer de la charge
echo "Générons une charge importante..."
kubectl run -it --rm load-generator --image=busybox -n demo --restart=Never -- sh -c "while true; do wget -q -O- http://php-apache; done"

# Dans un terminal séparé, observer le scaling
kubectl get hpa -n demo -w
kubectl get pods -n demo -l app=php-apache -w

# 6. Montrer le scaling en action
echo "Kubernetes ajoute automatiquement des pods..."
watch -n 2 "kubectl get hpa -n demo; echo ''; kubectl get pods -n demo -l app=php-apache"

# 7. Arrêter la charge et observer le scale down
# Arrêtez le load-generator avec Ctrl+C

echo "Sans charge, Kubernetes réduit automatiquement le nombre de pods..."
kubectl get hpa -n demo -w
```

**Points à souligner** :
- Adaptation automatique à la charge
- Optimisation des coûts (scale down)
- Basé sur des métriques réelles (CPU, mémoire, custom)
- Réaction rapide aux pics de trafic

### Scénario 4 : Application Multi-Tiers Complète

**Objectif** : Démontrer une architecture réaliste avec plusieurs composants

**Durée** : 15 minutes

**Audience** : Stakeholders techniques, clients potentiels

#### Architecture

```
[Frontend Web] → [Backend API] → [Redis Cache]
                                ↓
                           [Database PostgreSQL]
```

#### Déploiement

```yaml
# fullstack-demo.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: fullstack-demo
---
# PostgreSQL Database
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: fullstack-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
        tier: database
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_DB
          value: appdb
        - name: POSTGRES_USER
          value: appuser
        - name: POSTGRES_PASSWORD
          value: demopassword
        ports:
        - containerPort: 5432
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: fullstack-demo
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
---
# Redis Cache
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: fullstack-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
        tier: cache
    spec:
      containers:
      - name: redis
        image: redis:alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: fullstack-demo
spec:
  selector:
    app: redis
  ports:
  - port: 6379
---
# Backend API
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: fullstack-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        tier: backend
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo
        args:
        - "-text=Backend API - Connected to DB: postgres:5432, Cache: redis:6379"
        - "-listen=:8080"
        env:
        - name: DATABASE_URL
          value: "postgresql://appuser:demopassword@postgres:5432/appdb"
        - name: REDIS_URL
          value: "redis://redis:6379"
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: fullstack-demo
spec:
  selector:
    app: backend
  ports:
  - port: 8080
---
# Frontend
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: fullstack-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
      volumes:
      - name: nginx-config
        configMap:
          name: frontend-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
  namespace: fullstack-demo
data:
  default.conf: |
    server {
        listen 80;
        location /api/ {
            proxy_pass http://backend:8080/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        location / {
            root /usr/share/nginx/html;
            index index.html;
            try_files $uri $uri/ /index.html;
        }
    }
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: fullstack-demo
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    nodePort: 30100
  type: NodePort
```

#### Script de Démonstration

```bash
# 1. Déployer toute la stack
echo "🚀 Déploiement d'une application full-stack complète..."
kubectl apply -f fullstack-demo.yaml

# 2. Observer le déploiement progressif
echo "📦 Observation de la création des composants..."
kubectl get all -n fullstack-demo

# 3. Attendre que tout soit prêt
echo "⏳ Attente de la disponibilité de tous les services..."
kubectl wait --for=condition=available deployment --all -n fullstack-demo --timeout=300s

# 4. Montrer l'architecture
echo "🏗️  Architecture déployée:"
echo "- Frontend (2 replicas) → http://localhost:30100"
echo "- Backend API (3 replicas)"
echo "- Redis Cache (1 replica)"
echo "- PostgreSQL Database (1 replica)"

# 5. Visualiser la topologie
kubectl get pods -n fullstack-demo -o wide --show-labels

# 6. Tester la connectivité entre services
echo "🔍 Test de connectivité inter-services..."
BACKEND_POD=$(kubectl get pod -n fullstack-demo -l app=backend -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n fullstack-demo $BACKEND_POD -- nc -zv postgres 5432
kubectl exec -n fullstack-demo $BACKEND_POD -- nc -zv redis 6379

# 7. Tester l'application
echo "🌐 Test de l'application:"
curl http://localhost:30100/api/

# 8. Montrer les logs en temps réel
echo "📝 Logs du backend:"
kubectl logs -f -l app=backend -n fullstack-demo --tail=20

# 9. Scaler les différentes parties
echo "📈 Scaling des composants:"
kubectl scale deployment frontend --replicas=4 -n fullstack-demo
kubectl scale deployment backend --replicas=6 -n fullstack-demo

# 10. Simuler une panne
echo "💥 Simulation d'une panne - suppression d'un pod backend..."
BACKEND_POD=$(kubectl get pod -n fullstack-demo -l app=backend -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $BACKEND_POD -n fullstack-demo

echo "🔄 Kubernetes recrée automatiquement le pod..."
kubectl get pods -n fullstack-demo -l app=backend -w
```

**Points à souligner** :
- Architecture réaliste multi-tiers
- Communication sécurisée entre services
- Séparation des responsabilités
- Scaling indépendant de chaque composant
- Résilience automatique

### Scénario 5 : GitOps et Déploiement Continu

**Objectif** : Montrer le déploiement automatisé depuis Git

**Durée** : 12 minutes

**Audience** : Équipes DevOps, développeurs

#### Préparation avec ArgoCD

```bash
# 1. Installer ArgoCD
microk8s enable argocd

# Attendre que ArgoCD soit prêt
kubectl wait --for=condition=available deployment --all -n argocd --timeout=300s

# 2. Obtenir le mot de passe admin
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d

# 3. Port-forward pour accéder à l'UI
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
```

#### Application de Démonstration

Créez un repository Git (peut être local pour la démo) avec cette structure :

```
demo-app/
├── deployment.yaml
├── service.yaml
└── ingress.yaml
```

#### Script de Démonstration

```bash
# 1. Montrer le repository Git
echo "📂 Voici notre repository Git avec la configuration de l'application..."
ls -la demo-app/

# 2. Créer l'application ArgoCD
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/votre-org/demo-app.git
    targetRevision: HEAD
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

# 3. Observer le déploiement automatique
echo "🔄 ArgoCD synchronise automatiquement depuis Git..."
kubectl get application -n argocd

# 4. Montrer l'UI ArgoCD
echo "🎨 Interface ArgoCD disponible sur https://localhost:8080"
echo "Username: admin"
echo "Password: [mot de passe récupéré précédemment]"

# 5. Faire un changement dans Git
echo "✏️  Modification du nombre de replicas dans Git..."
# Modifier deployment.yaml : replicas: 3 → replicas: 5
# Commit et push

# 6. Observer la synchronisation automatique
echo "🔄 ArgoCD détecte le changement et déploie automatiquement..."
kubectl get pods -n demo -w

# 7. Montrer l'historique des déploiements
kubectl get events -n demo --sort-by='.lastTimestamp'
```

**Points à souligner** :
- Single source of truth (Git)
- Déploiement automatique
- Traçabilité complète
- Rollback facile via Git
- Audit trail

## PoC Techniques

### PoC 1 : Migration d'Application Legacy

**Objectif** : Prouver qu'une application existante peut être conteneurisée et déployée sur Kubernetes

**Durée** : 1-2 semaines

#### Phase 1 : Analyse de l'Application

```bash
# Documentation de l'application existante
- Architecture actuelle
- Dépendances (base de données, services externes)
- Volumes et données persistantes
- Configuration et secrets
- Ports et protocoles utilisés
```

#### Phase 2 : Conteneurisation

```dockerfile
# Dockerfile pour l'application legacy
FROM ubuntu:20.04

# Installation des dépendances
RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

# Copie de l'application
WORKDIR /app
COPY . /app

# Installation des dépendances Python
RUN pip3 install -r requirements.txt

# Configuration
ENV PORT=8080
EXPOSE 8080

# Démarrage
CMD ["python3", "app.py"]
```

#### Phase 3 : Déploiement Kubernetes

```yaml
# legacy-app-poc.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: legacy-migration-poc
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-app
  namespace: legacy-migration-poc
spec:
  replicas: 2
  selector:
    matchLabels:
      app: legacy-app
  template:
    metadata:
      labels:
        app: legacy-app
    spec:
      containers:
      - name: app
        image: localhost:32000/legacy-app:1.0
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: postgres
        - name: DB_NAME
          value: legacydb
        volumeMounts:
        - name: config
          mountPath: /app/config
        - name: data
          mountPath: /app/data
      volumes:
      - name: config
        configMap:
          name: legacy-app-config
      - name: data
        persistentVolumeClaim:
          claimName: legacy-app-data
```

#### Phase 4 : Tests et Validation

```bash
# Script de test
#!/bin/bash

echo "🧪 Tests du PoC de migration..."

# Test 1 : Application démarre correctement
echo "Test 1: Démarrage de l'application"
kubectl wait --for=condition=ready pod -l app=legacy-app -n legacy-migration-poc --timeout=300s

# Test 2 : Connectivité base de données
echo "Test 2: Connexion à la base de données"
POD=$(kubectl get pod -n legacy-migration-poc -l app=legacy-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n legacy-migration-poc $POD -- curl -f http://localhost:8080/health

# Test 3 : Persistance des données
echo "Test 3: Persistance des données"
# Écrire des données
# Redémarrer le pod
# Vérifier que les données sont toujours là

# Test 4 : Performance
echo "Test 4: Tests de performance"
# Utiliser Apache Bench ou similar
ab -n 1000 -c 10 http://localhost:30000/

# Test 5 : Résilience
echo "Test 5: Tests de résilience"
kubectl delete pod -l app=legacy-app -n legacy-migration-poc
sleep 10
kubectl get pods -n legacy-migration-poc
```

#### Phase 5 : Documentation des Résultats

Documentez :
- ✅ Fonctionnalités qui marchent
- ⚠️ Limitations identifiées
- 📊 Métriques de performance (avant/après)
- 💰 Estimation des coûts
- 🔄 Plan de migration complet
- 📝 Recommandations

### PoC 2 : Comparaison de Solutions de Monitoring

**Objectif** : Évaluer différentes solutions de monitoring pour choisir la meilleure

**Durée** : 1 semaine

#### Solutions à Comparer

1. **Prometheus + Grafana** (solution open-source)
2. **Datadog** (solution SaaS)
3. **ELK Stack** (logs + métriques)

#### Critères d'Évaluation

```yaml
# Grille d'évaluation
evaluation:
  facilite_installation:
    weight: 15%
    scoring: 1-10

  facilite_utilisation:
    weight: 20%
    scoring: 1-10

  fonctionnalites:
    weight: 25%
    required:
      - Métriques système
      - Métriques applicatives
      - Alerting
      - Dashboards
      - Logs centralisés

  performance:
    weight: 15%
    metrics:
      - Latence de collecte
      - Consommation CPU
      - Consommation mémoire

  cout:
    weight: 15%
    factors:
      - Licence
      - Infrastructure
      - Maintenance

  integration:
    weight: 10%
    aspects:
      - APIs disponibles
      - Intégrations tierces
      - Extensibilité
```

#### Déploiement des Solutions

```bash
# Solution 1 : Prometheus + Grafana
microk8s enable prometheus

# Solution 2 : Datadog (avec agent)
kubectl apply -f datadog-agent.yaml

# Solution 3 : ELK Stack
kubectl apply -f elasticsearch.yaml
kubectl apply -f kibana.yaml
kubectl apply -f filebeat.yaml
```

#### Tests Comparatifs

```bash
# Générer une charge applicative
kubectl run load-test --image=williamyeh/wrk -n poc -- \
  wrk -t12 -c400 -d30s http://test-app

# Mesurer les performances de chaque solution
# CPU/Mémoire utilisés
kubectl top pods -n monitoring

# Latence de collecte
# (à mesurer manuellement dans chaque UI)

# Facilité d'utilisation
# (score subjectif après utilisation)
```

#### Rapport de Comparaison

```markdown
# Rapport de Comparaison - Solutions de Monitoring

## Résumé Exécutif
[Recommandation finale basée sur les scores]

## Scores Détaillés

### Prometheus + Grafana
- Installation: 8/10 (addon MicroK8s très simple)
- Utilisation: 7/10 (courbe d'apprentissage PromQL)
- Fonctionnalités: 9/10 (très complètes)
- Performance: 9/10 (léger et efficace)
- Coût: 10/10 (open-source, gratuit)
- Intégration: 9/10 (excellente)
**Score Total: 8.6/10**

### Datadog
- Installation: 9/10 (simple avec agent)
- Utilisation: 9/10 (interface intuitive)
- Fonctionnalités: 10/10 (toutes présentes)
- Performance: 8/10 (agent consomme des ressources)
- Coût: 4/10 (cher pour production)
- Intégration: 10/10 (excellente)
**Score Total: 7.9/10**

### ELK Stack
- Installation: 6/10 (complexe)
- Utilisation: 7/10 (puissant mais complexe)
- Fonctionnalités: 8/10 (focus sur les logs)
- Performance: 6/10 (gourmand en ressources)
- Coût: 8/10 (open-source mais nécessite infrastructure)
- Intégration: 8/10 (bonne)
**Score Total: 7.1/10**

## Recommandation
Prometheus + Grafana pour notre cas d'usage, avec possibilité
d'ajouter Loki pour les logs centralisés.
```

### PoC 3 : Stratégie de Déploiement Blue-Green

**Objectif** : Valider une stratégie de déploiement sans risque

**Durée** : 3-5 jours

#### Configuration

```yaml
# blue-green-poc.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
  namespace: poc
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: app
        image: myapp:1.0
        env:
        - name: VERSION
          value: "Blue (1.0)"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
  namespace: poc
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: app
        image: myapp:2.0
        env:
        - name: VERSION
          value: "Green (2.0)"
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: poc
spec:
  selector:
    app: myapp
    version: blue  # Initialement sur blue
  ports:
  - port: 80
    targetPort: 8080
```

#### Script de Bascule

```bash
# switch-blue-green.sh
#!/bin/bash

CURRENT=$(kubectl get svc myapp -n poc -o jsonpath='{.spec.selector.version}')
echo "Version actuelle: $CURRENT"

if [ "$CURRENT" == "blue" ]; then
    NEW="green"
else
    NEW="blue"
fi

echo "Bascule vers: $NEW"

# Tests de smoke sur la nouvelle version
echo "Tests de la version $NEW..."
if [ "$NEW" == "green" ]; then
    TEST_POD=$(kubectl get pod -n poc -l version=green -o jsonpath='{.items[0].metadata.name}')
else
    TEST_POD=$(kubectl get pod -n poc -l version=blue -o jsonpath='{.items[0].metadata.name}')
fi

kubectl exec -n poc $TEST_POD -- curl -f http://localhost:8080/health

if [ $? -eq 0 ]; then
    echo "✅ Tests passés, bascule du trafic..."
    kubectl patch svc myapp -n poc -p "{\"spec\":{\"selector\":{\"version\":\"$NEW\"}}}"
    echo "✅ Trafic basculé vers $NEW"
else
    echo "❌ Tests échoués, annulation de la bascule"
    exit 1
fi
```

#### Validation

```bash
# Tests de validation
1. Déployer les deux versions
2. Vérifier blue et green fonctionnent
3. Exécuter le script de bascule
4. Vérifier que le trafic bascule instantanément
5. Vérifier qu'aucune requête n'est perdue
6. Mesurer le temps de bascule
7. Tester le rollback
```

## Conseils pour des Démos Réussies

### Avant la Démo

**1. Préparez votre environnement**
```bash
# Checklist de préparation
- [ ] MicroK8s démarré et fonctionnel
- [ ] Tous les addons nécessaires activés
- [ ] Images Docker pre-pulled
- [ ] Scripts testés au moins 3 fois
- [ ] Namespaces créés et nettoyés
- [ ] Connexion réseau stable
- [ ] Batterie chargée (si laptop)
- [ ] Plan B préparé
```

**2. Testez votre démo**
- Répétez la démo au moins 3 fois
- Chronométrez chaque étape
- Notez les commandes exactes à utiliser
- Préparez des screenshots de backup

**3. Préparez votre discours**
- Définissez les points clés à couvrir
- Préparez des réponses aux questions probables
- Adaptez le niveau technique à l'audience
- Préparez des analogies pour expliquer les concepts

### Pendant la Démo

**1. Commencez fort**
```bash
# Impression immédiate avec un one-liner
kubectl create deployment demo --image=nginx --replicas=5 && \
kubectl expose deployment demo --port=80 --type=NodePort && \
echo "Application déployée en 5 secondes !"
```

**2. Racontez une histoire**
- Contexte : "Imaginez que nous devons..."
- Problème : "Sans Kubernetes, nous devrions..."
- Solution : "Avec Kubernetes, nous pouvons simplement..."
- Résultat : "Et voilà, c'est déployé et scalable !"

**3. Gérez les imprévus**
```bash
# Si quelque chose échoue
echo "Voyons ce qui se passe..."
kubectl describe pod <pod-name>
kubectl logs <pod-name>

# Ou utilisez votre backup
echo "Passons à la version pré-préparée..."
```

**4. Interagissez avec l'audience**
- Posez des questions
- Invitez à participer
- Adaptez le rythme selon les réactions

### Après la Démo

**1. Récapitulez**
- Résumez les points clés
- Répondez aux questions
- Partagez des ressources

**2. Suivez**
- Envoyez les scripts et configs
- Proposez un atelier hands-on
- Restez disponible pour des questions

**3. Collectez les retours**
- Ce qui a bien fonctionné
- Ce qui peut être amélioré
- Questions non résolues

## Gestion des Risques en Démo

### Plan B : Backup de Démo

```bash
# Enregistrer une session complète avec asciinema
asciinema rec demo-backup.cast

# Durant la démo, si problème technique :
asciinema play demo-backup.cast
```

### Screenshots de Backup

Préparez des screenshots de chaque étape importante :
- État initial du cluster
- Application déployée
- Scaling en action
- Dashboards de monitoring

### Mode Hors-Ligne

Si la démo ne nécessite pas d'accès internet :

```bash
# Sauvegarder toutes les images Docker nécessaires
docker save nginx redis postgres > demo-images.tar

# Sur la machine de démo
docker load < demo-images.tar
```

### Script de Récupération Rapide

```bash
#!/bin/bash
# emergency-reset.sh

echo "🚨 Emergency reset..."

# Supprimer tout dans le namespace demo
kubectl delete all --all -n demo --grace-period=0 --force

# Nettoyer Docker
docker system prune -af

# Redémarrer MicroK8s
microk8s stop
microk8s start

# Attendre que tout soit prêt
microk8s status --wait-ready

echo "✅ System ready for retry!"
```

## Outils Utiles pour les Démos

### Enregistrement de Terminal

```bash
# asciinema - Enregistrer et rejouer des sessions terminal
sudo apt install asciinema

# Enregistrer
asciinema rec ma-demo.cast

# Rejouer
asciinema play ma-demo.cast

# Partager en ligne
asciinema upload ma-demo.cast
```

### Présentation de Code

```bash
# bat - Syntax highlighting pour les fichiers
sudo apt install bat

# Utilisation
bat deployment.yaml

# Avec numéros de ligne
bat -n deployment.yaml
```

### Terminal Recorder

```bash
# terminalizer - Créer des GIFs animés de votre terminal
npm install -g terminalizer

# Enregistrer
terminalizer record demo

# Générer le GIF
terminalizer render demo
```

### Dashboard Live

```bash
# k9s - TUI interactif pour Kubernetes
# Parfait pour montrer l'état du cluster en temps réel
sudo snap install k9s

# Lancer
k9s

# Navigation intuitive pour impressionner l'audience
```

## Modèles de Documentation

### Template de Présentation de Démo

```markdown
# Démo : [Titre]

## Objectif
[Qu'est-ce que cette démo montre ?]

## Audience
[Qui est le public cible ?]

## Durée
[Temps estimé]

## Prérequis
- MicroK8s installé
- Addons activés : [liste]
- Images pre-pulled : [liste]

## Scénario

### Étape 1 : [Titre]
**Durée** : X minutes
**Message clé** : [Point important à transmettre]

```bash
# Commandes
kubectl create ...
```

**Points à souligner** :
- Point 1
- Point 2

### Étape 2 : [Titre]
...

## Questions Anticipées

**Q: Pourquoi utiliser Kubernetes plutôt que Docker Compose ?**
R: [Votre réponse préparée]

**Q: Quel est le coût de Kubernetes en production ?**
R: [Votre réponse préparée]

## Ressources à Partager
- [Lien vers la documentation]
- [Lien vers les scripts]
- [Lien vers les exemples]
```

### Template de Rapport de PoC

```markdown
# Rapport de PoC : [Titre]

## Résumé Exécutif
[Résumé en 2-3 phrases de ce qui a été testé et de la conclusion]

## Objectif du PoC
[Qu'est-ce qu'on cherchait à prouver ?]

## Méthodologie
[Comment le PoC a été conduit]

## Configuration Technique
- Infrastructure : MicroK8s sur [specs]
- Version Kubernetes : [version]
- Applications testées : [liste]
- Durée du PoC : [dates]

## Résultats

### Ce Qui Fonctionne ✅
- Fonctionnalité 1 : [détails]
- Fonctionnalité 2 : [détails]

### Limitations Identifiées ⚠️
- Limitation 1 : [détails et solution possible]
- Limitation 2 : [détails et solution possible]

### Métriques
- Performance : [chiffres]
- Ressources utilisées : [chiffres]
- Temps de déploiement : [chiffres]

## Recommandations

### Actions Recommandées
1. [Action 1]
2. [Action 2]

### Prochaines Étapes
1. [Étape 1]
2. [Étape 2]

## Conclusion
[Conclusion finale : go/no-go pour le projet]

## Annexes
- Logs et traces
- Configurations utilisées
- Scripts développés
```

## Conclusion

Les démonstrations et les PoC sont des outils puissants pour convaincre, valider et former. Avec MicroK8s, vous disposez d'une plateforme idéale pour :

✅ Créer des démos impressionnantes et reproductibles
✅ Valider des concepts avant investissement
✅ Former vos équipes de manière pratique
✅ Convaincre les stakeholders avec des preuves concrètes
✅ Tester des architectures en conditions réelles

**Points clés à retenir** :

- **Préparez minutieusement** : Une bonne démo se prépare
- **Testez plusieurs fois** : Rien ne doit échouer pendant la présentation
- **Ayez un plan B** : Les imprévus arrivent
- **Racontez une histoire** : Les gens retiennent les histoires, pas les commandes
- **Adaptez au public** : Technique pour les dev, business pour les managers
- **Documentez tout** : Scripts, résultats, recommandations

**Prochaines étapes** :

1. Choisissez un scénario de démo à préparer
2. Testez-le au moins 3 fois
3. Présentez-le à votre équipe
4. Collectez les retours et améliorez
5. Créez votre bibliothèque de démos réutilisables

**Bonne démo ! 🎬**

⏭️ [Formation et certification](/24-cas-dusage-pratiques/06-formation-et-certification.md)
