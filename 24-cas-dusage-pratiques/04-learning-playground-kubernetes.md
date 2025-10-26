🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 24.4 Learning Playground Kubernetes

## Introduction

Un learning playground Kubernetes avec MicroK8s est un environnement d'apprentissage idéal pour découvrir et maîtriser Kubernetes de manière pratique. C'est un espace sûr où vous pouvez expérimenter, faire des erreurs, tout casser et tout reconstruire sans conséquences.

Dans ce chapitre, nous allons créer ensemble un environnement d'apprentissage complet qui vous guidera progressivement des concepts de base jusqu'aux techniques avancées de Kubernetes.

## Pourquoi un Playground Kubernetes ?

### Avantages d'un Environnement Dédié à l'Apprentissage

**Apprentissage par la pratique** : Kubernetes s'apprend en l'utilisant. Lire la documentation est important, mais rien ne remplace l'expérimentation pratique.

**Environnement isolé** : Vous pouvez faire des erreurs sans impact sur des systèmes de production ou d'autres projets.

**Expérimentation libre** : Testez différentes configurations, architectures et patterns sans contraintes.

**Coût zéro** : Un cluster local ne coûte rien, contrairement aux environnements cloud où chaque erreur peut être facturée.

**Disponibilité permanente** : Votre playground est toujours disponible, pas de limitation de temps comme avec les plateformes en ligne.

**Préparation aux certifications** : Idéal pour préparer les certifications CKA, CKAD ou CKS.

### Ce Que Vous Allez Apprendre

- Les concepts fondamentaux de Kubernetes
- Le déploiement et la gestion d'applications
- Le networking et les services
- Le stockage persistant
- La configuration et les secrets
- Le debugging et le troubleshooting
- Les patterns d'architecture cloud-native
- Les bonnes pratiques DevOps

## Configuration du Playground

### Configuration Système Recommandée

Pour un playground confortable :

**Configuration Minimale** :
- CPU : 4 cœurs
- RAM : 8 GB
- Stockage : 50 GB
- OS : Ubuntu 20.04+

**Configuration Idéale** :
- CPU : 6-8 cœurs
- RAM : 12-16 GB
- Stockage : 100 GB SSD
- OS : Ubuntu 22.04 LTS

### Installation et Configuration Initiale

```bash
# Installation de MicroK8s
sudo snap install microk8s --classic

# Ajouter votre utilisateur au groupe
sudo usermod -a -G microk8s $USER
newgrp microk8s

# Configuration de l'alias kubectl
echo "alias kubectl='microk8s kubectl'" >> ~/.bashrc
source ~/.bashrc

# Activer les addons essentiels
microk8s enable dns storage dashboard registry
```

### Création des Namespaces d'Apprentissage

Organisez votre apprentissage avec des namespaces thématiques :

```bash
# Namespace pour les concepts de base
kubectl create namespace basics

# Namespace pour les applications multi-tiers
kubectl create namespace apps

# Namespace pour les expérimentations réseau
kubectl create namespace networking

# Namespace pour le stockage
kubectl create namespace storage-lab

# Namespace pour la sécurité
kubectl create namespace security-lab

# Namespace pour les patterns avancés
kubectl create namespace advanced

# Namespace pour les erreurs volontaires (breaking things!)
kubectl create namespace sandbox

# Labelliser les namespaces
kubectl label namespace basics purpose=learning level=beginner
kubectl label namespace apps purpose=learning level=intermediate
kubectl label namespace advanced purpose=learning level=advanced
kubectl label namespace sandbox purpose=experimentation
```

### Outils Complémentaires Utiles

```bash
# k9s - Interface TUI pour Kubernetes
sudo snap install k9s

# kubectx et kubens - Changement rapide de contexte/namespace
sudo apt install kubectx

# stern - Visualisation des logs multi-pods
wget https://github.com/stern/stern/releases/download/v1.25.0/stern_1.25.0_linux_amd64.tar.gz
tar -xzf stern_1.25.0_linux_amd64.tar.gz
sudo mv stern /usr/local/bin/
```

## Parcours d'Apprentissage Progressif

### Niveau 1 : Les Fondamentaux

#### 1.1 Votre Premier Pod

Le pod est l'unité de base dans Kubernetes. Commençons simple :

```yaml
# first-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-first-pod
  namespace: basics
  labels:
    app: learning
    type: demo
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

Déployez et explorez :

```bash
# Créer le pod
kubectl apply -f first-pod.yaml

# Observer la création
kubectl get pods -n basics -w

# Inspecter le pod
kubectl describe pod my-first-pod -n basics

# Voir les logs
kubectl logs my-first-pod -n basics

# Se connecter au pod
kubectl exec -it my-first-pod -n basics -- bash

# À l'intérieur du pod, testez
nginx -v
curl localhost
exit

# Supprimer le pod
kubectl delete pod my-first-pod -n basics
```

**Points d'apprentissage** :
- Un pod peut contenir un ou plusieurs conteneurs
- Les pods sont éphémères (supprimé = données perdues)
- Les labels permettent d'organiser et de sélectionner les ressources

#### 1.2 Comprendre les Deployments

Les Deployments gèrent automatiquement vos pods :

```yaml
# first-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
  namespace: basics
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

Explorez les fonctionnalités :

```bash
# Déployer
kubectl apply -f first-deployment.yaml

# Observer les pods créés
kubectl get pods -n basics -l app=web

# Voir le deployment
kubectl get deployment -n basics
kubectl describe deployment web-deployment -n basics

# Scaler le deployment
kubectl scale deployment web-deployment --replicas=5 -n basics

# Observer le scaling
kubectl get pods -n basics -l app=web -w

# Mettre à jour l'image
kubectl set image deployment/web-deployment nginx=nginx:1.22 -n basics

# Observer le rolling update
kubectl rollout status deployment/web-deployment -n basics

# Voir l'historique des versions
kubectl rollout history deployment/web-deployment -n basics

# Revenir en arrière
kubectl rollout undo deployment/web-deployment -n basics

# Supprimer un pod et observer la recréation automatique
POD_NAME=$(kubectl get pods -n basics -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $POD_NAME -n basics
kubectl get pods -n basics -l app=web -w
```

**Points d'apprentissage** :
- Les Deployments maintiennent automatiquement le nombre de réplicas
- Le rolling update permet de mettre à jour sans downtime
- L'historique permet de revenir en arrière si nécessaire

#### 1.3 Services et Exposition

Les Services permettent de communiquer avec vos pods :

```yaml
# first-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: basics
spec:
  selector:
    app: web
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
  namespace: basics
spec:
  selector:
    app: web
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
```

Testez la communication :

```bash
# Créer les services
kubectl apply -f first-service.yaml

# Voir les services
kubectl get svc -n basics

# Tester depuis un autre pod
kubectl run -it --rm debug --image=busybox -n basics --restart=Never -- sh
# À l'intérieur :
wget -O- http://web-service
nslookup web-service
exit

# Accéder via NodePort (depuis votre machine hôte)
curl http://localhost:30080

# Voir les endpoints
kubectl get endpoints web-service -n basics
```

**Points d'apprentissage** :
- ClusterIP : accessible seulement dans le cluster
- NodePort : accessible depuis l'extérieur via le port du nœud
- Les Services utilisent les labels pour sélectionner les pods

### Niveau 2 : Applications Multi-Composants

#### 2.1 Application Web avec Base de Données

Créez une application WordPress complète :

```yaml
# wordpress-stack.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: wordpress-lab
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: wordpress-lab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: wordpress-lab
type: Opaque
stringData:
  mysql-root-password: "rootpassword123"
  mysql-database: "wordpress"
  mysql-user: "wpuser"
  mysql-password: "wppassword123"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: wordpress-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-root-password
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-database
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-user
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: wordpress-lab
spec:
  selector:
    app: mysql
  ports:
  - protocol: TCP
    port: 3306
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
  namespace: wordpress-lab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: wordpress-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:latest
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql
        - name: WORDPRESS_DB_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-user
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-password
        - name: WORDPRESS_DB_NAME
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-database
        ports:
        - containerPort: 80
        volumeMounts:
        - name: wordpress-storage
          mountPath: /var/www/html
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: wordpress-storage
        persistentVolumeClaim:
          claimName: wordpress-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: wordpress-lab
spec:
  selector:
    app: wordpress
  ports:
  - protocol: TCP
    port: 80
    nodePort: 30081
  type: NodePort
```

Déployez et explorez :

```bash
# Déployer la stack complète
kubectl apply -f wordpress-stack.yaml

# Observer la création dans l'ordre
kubectl get all -n wordpress-lab -w

# Vérifier que MySQL est prêt
kubectl wait --for=condition=ready pod -l app=mysql -n wordpress-lab --timeout=300s

# Vérifier WordPress
kubectl wait --for=condition=ready pod -l app=wordpress -n wordpress-lab --timeout=300s

# Voir les logs de MySQL
kubectl logs -f -l app=mysql -n wordpress-lab

# Voir les logs de WordPress
kubectl logs -f -l app=wordpress -n wordpress-lab

# Accéder à WordPress
echo "WordPress disponible sur : http://localhost:30081"

# Tester la connexion à la base de données depuis WordPress
kubectl exec -it -n wordpress-lab deployment/wordpress -- wp db check

# Voir les secrets (encodés en base64)
kubectl get secret mysql-secret -n wordpress-lab -o yaml

# Décoder un secret
kubectl get secret mysql-secret -n wordpress-lab -o jsonpath='{.data.mysql-password}' | base64 -d
```

**Points d'apprentissage** :
- Communication entre services via DNS interne
- Utilisation de Secrets pour les données sensibles
- PersistentVolumeClaims pour la persistance des données
- Dépendances entre services (WordPress dépend de MySQL)

#### 2.2 Application Microservices

Créez une application avec plusieurs microservices :

```yaml
# microservices-demo.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: microservices-lab
---
# Frontend
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: microservices-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
      tier: frontend
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
        - name: config
          mountPath: /etc/nginx/conf.d
      volumes:
      - name: config
        configMap:
          name: frontend-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
  namespace: microservices-lab
data:
  default.conf: |
    server {
        listen 80;
        location /api/ {
            proxy_pass http://backend:8080/;
        }
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: microservices-lab
spec:
  selector:
    app: frontend
    tier: frontend
  ports:
  - protocol: TCP
    port: 80
    nodePort: 30082
  type: NodePort
---
# Backend API
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: microservices-lab
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
      tier: backend
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
        - "-text=Hello from backend pod: $(POD_NAME)"
        - "-listen=:8080"
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: CACHE_HOST
          value: "redis"
        - name: DB_HOST
          value: "database"
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: microservices-lab
spec:
  selector:
    app: backend
    tier: backend
  ports:
  - protocol: TCP
    port: 8080
---
# Cache Redis
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: microservices-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
      tier: cache
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
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: microservices-lab
spec:
  selector:
    app: redis
    tier: cache
  ports:
  - protocol: TCP
    port: 6379
---
# Database
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: microservices-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
      tier: database
  template:
    metadata:
      labels:
        app: database
        tier: database
    spec:
      containers:
      - name: postgres
        image: postgres:alpine
        env:
        - name: POSTGRES_PASSWORD
          value: "password"
        ports:
        - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: microservices-lab
spec:
  selector:
    app: database
    tier: database
  ports:
  - protocol: TCP
    port: 5432
```

Explorez l'architecture :

```bash
# Déployer
kubectl apply -f microservices-demo.yaml

# Visualiser l'architecture
kubectl get all -n microservices-lab

# Tester la communication entre services
kubectl run -it --rm test --image=busybox -n microservices-lab --restart=Never -- sh
# À l'intérieur :
wget -O- http://frontend
wget -O- http://backend:8080
wget -O- http://redis:6379
exit

# Observer le load balancing du backend
for i in {1..10}; do
  curl http://localhost:30082/api/
  sleep 1
done

# Voir les logs de tous les backends
kubectl logs -f -l app=backend -n microservices-lab --all-containers=true

# Scaler les différentes parties
kubectl scale deployment frontend --replicas=3 -n microservices-lab
kubectl scale deployment backend --replicas=5 -n microservices-lab

# Voir la topologie avec labels
kubectl get pods -n microservices-lab --show-labels
kubectl get pods -n microservices-lab -l tier=backend
```

**Points d'apprentissage** :
- Architecture microservices avec séparation des responsabilités
- Communication inter-services via DNS
- ConfigMaps pour la configuration
- Labels pour organiser et sélectionner les ressources
- Load balancing automatique entre réplicas

### Niveau 3 : Networking Avancé

#### 3.1 Network Policies

Apprenez à sécuriser la communication entre pods :

```yaml
# network-policies.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: network-lab
  labels:
    name: network-lab
---
# Pods de test
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: network-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        role: web
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: network-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        role: api
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo
        args: ["-text=Backend API"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: network-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
        role: db
    spec:
      containers:
      - name: postgres
        image: postgres:alpine
        env:
        - name: POSTGRES_PASSWORD
          value: "password"
---
# Services
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: network-lab
spec:
  selector:
    app: backend
  ports:
  - port: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: network-lab
spec:
  selector:
    app: database
  ports:
  - port: 5432
---
# Network Policy : Database accessible seulement par Backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: network-lab
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: api
    ports:
    - protocol: TCP
      port: 5432
---
# Network Policy : Backend accessible seulement par Frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: network-lab
spec:
  podSelector:
    matchLabels:
      role: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: web
    ports:
    - protocol: TCP
      port: 5678
---
# Network Policy : Frontend accessible par tous
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: network-lab
spec:
  podSelector:
    matchLabels:
      role: web
  policyTypes:
  - Ingress
  ingress:
  - {}
```

Testez les Network Policies :

```bash
# Déployer
kubectl apply -f network-policies.yaml

# Test 1 : Frontend peut accéder au Backend (AUTORISÉ)
FRONTEND_POD=$(kubectl get pod -n network-lab -l app=frontend -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $FRONTEND_POD -n network-lab -- wget -O- http://backend:5678 --timeout=5

# Test 2 : Frontend ne peut PAS accéder à la Database (BLOQUÉ)
kubectl exec -it $FRONTEND_POD -n network-lab -- nc -zv database 5432 -w 5

# Test 3 : Backend peut accéder à la Database (AUTORISÉ)
BACKEND_POD=$(kubectl get pod -n network-lab -l app=backend -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $BACKEND_POD -n network-lab -- nc -zv database 5432 -w 5

# Test 4 : Pod random ne peut pas accéder à la Database (BLOQUÉ)
kubectl run -it --rm test --image=busybox -n network-lab --restart=Never -- nc -zv database 5432 -w 5

# Voir les Network Policies
kubectl get networkpolicies -n network-lab
kubectl describe networkpolicy database-policy -n network-lab
```

**Points d'apprentissage** :
- Network Policies contrôlent le trafic entre pods
- Principe du moindre privilège : autoriser seulement ce qui est nécessaire
- Les politiques sont additives
- Utilisation des labels pour cibler les pods

### Niveau 4 : Patterns et Best Practices

#### 4.1 Health Checks et Readiness

Comprenez les probes pour la fiabilité :

```yaml
# health-checks.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-probes
  namespace: advanced
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app-with-probes
  template:
    metadata:
      labels:
        app: app-with-probes
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text=I'm healthy!"]
        ports:
        - containerPort: 5678
        # Liveness Probe : Redémarre le pod si échoue
        livenessProbe:
          httpGet:
            path: /
            port: 5678
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        # Readiness Probe : Retire du service si échoue
        readinessProbe:
          httpGet:
            path: /
            port: 5678
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        # Startup Probe : Pour les apps qui démarrent lentement
        startupProbe:
          httpGet:
            path: /
            port: 5678
          initialDelaySeconds: 0
          periodSeconds: 2
          failureThreshold: 30
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

Simulez des pannes :

```bash
# Déployer
kubectl create namespace advanced
kubectl apply -f health-checks.yaml

# Observer les pods
kubectl get pods -n advanced -w

# Simuler une panne d'un pod (kill le processus)
POD_NAME=$(kubectl get pods -n advanced -l app=app-with-probes -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n advanced $POD_NAME -- killall http-echo

# Observer le redémarrage automatique
kubectl get pods -n advanced -w

# Voir les événements
kubectl describe pod $POD_NAME -n advanced

# Voir les restart counts
kubectl get pods -n advanced -o custom-columns=NAME:.metadata.name,RESTARTS:.status.containerStatuses[0].restartCount
```

#### 4.2 ConfigMaps et Secrets Avancés

Gestion sophistiquée de la configuration :

```yaml
# config-management.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: advanced
data:
  # Configuration en format properties
  application.properties: |
    server.port=8080
    server.name=my-app
    database.host=postgres
    database.port=5432
    log.level=INFO

  # Configuration JSON
  config.json: |
    {
      "app": {
        "name": "my-app",
        "version": "1.0.0",
        "features": {
          "feature1": true,
          "feature2": false
        }
      }
    }

  # Configuration simple
  APP_ENV: "production"
  MAX_CONNECTIONS: "100"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: advanced
type: Opaque
stringData:
  # Secrets en clair (seront encodés automatiquement)
  database-username: admin
  database-password: super-secret-password
  api-key: sk-1234567890abcdef
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-demo
  namespace: advanced
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-demo
  template:
    metadata:
      labels:
        app: config-demo
    spec:
      containers:
      - name: app
        image: busybox
        command: ['sh', '-c', 'while true; do cat /config/application.properties; sleep 30; done']
        # Variables d'environnement depuis ConfigMap
        env:
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV
        - name: MAX_CONNECTIONS
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: MAX_CONNECTIONS
        # Variables d'environnement depuis Secret
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-password
        # Monter les fichiers de config
        volumeMounts:
        - name: config-volume
          mountPath: /config
        - name: secret-volume
          mountPath: /secrets
          readOnly: true
      volumes:
      - name: config-volume
        configMap:
          name: app-config
      - name: secret-volume
        secret:
          secretName: app-secrets
```

Explorez les configurations :

```bash
# Déployer
kubectl apply -f config-management.yaml

# Voir les variables d'environnement du pod
POD_NAME=$(kubectl get pods -n advanced -l app=config-demo -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n advanced $POD_NAME -- env

# Voir les fichiers montés
kubectl exec -n advanced $POD_NAME -- ls -la /config
kubectl exec -n advanced $POD_NAME -- cat /config/application.properties
kubectl exec -n advanced $POD_NAME -- cat /config/config.json

# Voir les secrets montés
kubectl exec -n advanced $POD_NAME -- ls -la /secrets
kubectl exec -n advanced $POD_NAME -- cat /secrets/api-key

# Modifier le ConfigMap
kubectl edit configmap app-config -n advanced
# Changez APP_ENV de "production" à "development"

# Observer le changement (peut prendre jusqu'à 60 secondes)
kubectl exec -n advanced $POD_NAME -- cat /config/application.properties

# Forcer le rechargement en recréant le pod
kubectl rollout restart deployment config-demo -n advanced
```

#### 4.3 InitContainers et Sidecar Pattern

Patterns avancés de conteneurs :

```yaml
# container-patterns.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: patterns-demo
  namespace: advanced
spec:
  replicas: 1
  selector:
    matchLabels:
      app: patterns-demo
  template:
    metadata:
      labels:
        app: patterns-demo
    spec:
      # InitContainers s'exécutent AVANT le conteneur principal
      initContainers:
      # InitContainer 1 : Attendre qu'un service soit prêt
      - name: wait-for-database
        image: busybox
        command: ['sh', '-c']
        args:
        - |
          until nslookup database.network-lab.svc.cluster.local; do
            echo "Waiting for database..."
            sleep 2
          done
          echo "Database is ready!"

      # InitContainer 2 : Télécharger des données
      - name: download-data
        image: busybox
        command: ['sh', '-c']
        args:
        - |
          echo "Downloading initial data..."
          mkdir -p /data
          echo "Initial data" > /data/startup.txt
          echo "Data downloaded!"
        volumeMounts:
        - name: shared-data
          mountPath: /data

      # Conteneur principal
      containers:
      # Conteneur applicatif principal
      - name: main-app
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
        - name: logs
          mountPath: /var/log/nginx

      # Sidecar : Collecteur de logs
      - name: log-collector
        image: busybox
        command: ['sh', '-c']
        args:
        - |
          while true; do
            if [ -f /logs/access.log ]; then
              echo "=== Access Logs ==="
              tail -n 5 /logs/access.log
            fi
            sleep 10
          done
        volumeMounts:
        - name: logs
          mountPath: /logs

      volumes:
      - name: shared-data
        emptyDir: {}
      - name: logs
        emptyDir: {}
```

Observez les patterns :

```bash
# Déployer
kubectl apply -f container-patterns.yaml

# Observer l'ordre de démarrage
kubectl get pods -n advanced -w

# Voir les logs des initContainers
POD_NAME=$(kubectl get pods -n advanced -l app=patterns-demo -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD_NAME -n advanced -c wait-for-database
kubectl logs $POD_NAME -n advanced -c download-data

# Voir les logs du conteneur principal
kubectl logs $POD_NAME -n advanced -c main-app

# Voir les logs du sidecar
kubectl logs -f $POD_NAME -n advanced -c log-collector

# Générer du trafic pour voir les logs
kubectl port-forward -n advanced $POD_NAME 8080:80
# Dans un autre terminal :
curl http://localhost:8080

# Voir tous les conteneurs d'un pod
kubectl get pod $POD_NAME -n advanced -o jsonpath='{.spec.containers[*].name}'
kubectl get pod $POD_NAME -n advanced -o jsonpath='{.spec.initContainers[*].name}'
```

**Points d'apprentissage** :
- InitContainers préparent l'environnement avant le démarrage
- Utiles pour attendre des dépendances, télécharger des données, etc.
- Sidecars étendent les fonctionnalités (logs, monitoring, proxy)
- Les conteneurs d'un pod partagent le réseau et peuvent partager des volumes

### Niveau 5 : Debugging et Troubleshooting

#### 5.1 Techniques de Debugging

Apprenez à résoudre les problèmes courants :

```bash
# Créer un pod en erreur volontairement
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: broken-pod
  namespace: sandbox
spec:
  containers:
  - name: app
    image: nginx:wrong-tag
    ports:
    - containerPort: 80
EOF

# Observer l'erreur
kubectl get pods -n sandbox

# Voir les détails de l'erreur
kubectl describe pod broken-pod -n sandbox

# Voir les événements du namespace
kubectl get events -n sandbox --sort-by='.lastTimestamp'

# Créer un pod qui crash au démarrage
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: crashloop-pod
  namespace: sandbox
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'echo "Starting..."; exit 1']
EOF

# Observer le CrashLoopBackOff
kubectl get pods -n sandbox -w

# Voir les logs du pod qui crash
kubectl logs crashloop-pod -n sandbox
kubectl logs crashloop-pod -n sandbox --previous

# Créer un pod avec problème de ressources
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oom-pod
  namespace: sandbox
spec:
  containers:
  - name: app
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "500M"]
    resources:
      limits:
        memory: "50Mi"
EOF

# Observer l'OOMKilled
kubectl get pods -n sandbox -w
kubectl describe pod oom-pod -n sandbox

# Debugging interactif
kubectl run -it --rm debug --image=nicolaka/netshoot -n sandbox --restart=Never -- bash
# À l'intérieur, testez la connectivité réseau :
nslookup kubernetes.default
curl -v http://backend.microservices-lab:8080
traceroute google.com
exit
```

#### 5.2 Inspection et Monitoring en Temps Réel

```bash
# Voir l'utilisation des ressources
kubectl top nodes
kubectl top pods -n advanced
kubectl top pods --all-namespaces --sort-by=memory

# Logs en temps réel de plusieurs pods
kubectl logs -f -l app=backend -n microservices-lab --all-containers=true

# Utiliser stern pour les logs avancés
stern backend -n microservices-lab

# Voir tous les événements du cluster
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Vérifier la santé du cluster
kubectl get componentstatuses
kubectl get nodes
kubectl get pods --all-namespaces | grep -v Running

# Inspecter les ressources
kubectl get all --all-namespaces
kubectl get pv,pvc --all-namespaces

# Utiliser k9s pour une vue interactive
k9s
```

### Niveau 6 : Simulations de Scénarios Réels

#### 6.1 Simulation de Montée en Charge

```yaml
# load-test.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: sandbox
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: sandbox
spec:
  selector:
    app: web-app
  ports:
  - port: 80
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: sandbox
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Testez l'autoscaling :

```bash
# Activer metrics-server si pas déjà fait
microk8s enable metrics-server

# Déployer
kubectl apply -f load-test.yaml

# Observer l'HPA
kubectl get hpa -n sandbox -w

# Générer de la charge
kubectl run -it --rm load-generator --image=busybox -n sandbox --restart=Never -- sh
# À l'intérieur :
while true; do wget -q -O- http://web-app; done

# Dans un autre terminal, observer le scaling
kubectl get hpa -n sandbox -w
kubectl get pods -n sandbox -l app=web-app -w

# Voir l'utilisation CPU
kubectl top pods -n sandbox -l app=web-app
```

#### 6.2 Simulation de Panne et Récupération

```bash
# Créer une application avec plusieurs réplicas
kubectl create deployment resilient-app --image=nginx --replicas=5 -n sandbox

# Exposer l'application
kubectl expose deployment resilient-app --port=80 --type=NodePort -n sandbox

# Obtenir le NodePort
kubectl get svc resilient-app -n sandbox

# Générer du trafic continu (dans un terminal séparé)
while true; do curl -s http://localhost:<nodeport> > /dev/null; echo "Request sent"; sleep 0.5; done

# Supprimer des pods pour simuler des pannes
kubectl get pods -n sandbox -l app=resilient-app
kubectl delete pod <pod-name-1> <pod-name-2> -n sandbox

# Observer la recréation automatique
kubectl get pods -n sandbox -l app=resilient-app -w

# Le trafic continue grâce aux autres réplicas !

# Simuler une panne du nœud (drain)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Observer la migration des pods
kubectl get pods -n sandbox -o wide -w

# Restaurer le nœud
kubectl uncordon <node-name>
```

## Ressources d'Apprentissage

### Documentation et Tutoriels

**Documentation officielle Kubernetes** :
- [kubernetes.io/docs](https://kubernetes.io/docs)
- Référence complète et à jour

**Kubernetes by Example** :
- [kubernetesbyexample.com](http://kubernetesbyexample.com)
- Exemples courts et pratiques

**Play with Kubernetes** :
- [labs.play-with-k8s.com](https://labs.play-with-k8s.com)
- Environnement en ligne gratuit

### Livres Recommandés

- **Kubernetes Up & Running** (O'Reilly)
- **Kubernetes Patterns** (O'Reilly)
- **The Kubernetes Book** (Nigel Poulton)

### Formations en Ligne

- **Kubernetes for Absolute Beginners** (KodeKloud)
- **Certified Kubernetes Administrator** (Linux Foundation)
- **Kubernetes Mastery** (Udemy)

### Communauté

- **Reddit** : r/kubernetes
- **Slack** : kubernetes.slack.com
- **Stack Overflow** : tag [kubernetes]
- **GitHub** : kubernetes/kubernetes

## Préparation aux Certifications

### CKA (Certified Kubernetes Administrator)

Compétences à maîtriser :
- Installation et configuration du cluster
- Gestion du cycle de vie des applications
- Networking
- Stockage
- Troubleshooting

### CKAD (Certified Kubernetes Application Developer)

Compétences à maîtriser :
- Définition et déploiement d'applications
- Services et networking
- Configuration
- Observabilité
- Multi-container pods

### CKS (Certified Kubernetes Security Specialist)

Compétences à maîtriser :
- Sécurité du cluster
- Minimisation des surfaces d'attaque
- Sécurité des supply chains
- Monitoring et logging
- Runtime security

### Conseils pour les Certifications

**Pratiquez, pratiquez, pratiquez** : Les examens sont 100% pratiques, pas de QCM.

**Maîtrisez kubectl** : Vous devez être rapide avec les commandes.

**Utilisez la documentation** : Autorisée pendant l'examen, apprenez à naviguer dedans rapidement.

**Chronométrez-vous** : Entraînez-vous avec des contraintes de temps.

**Créez des alias** : Gagnez du temps avec des raccourcis.

```bash
# Alias utiles
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployments'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias ke='kubectl exec -it'

# Générer des manifestes rapidement
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment web --image=nginx --dry-run=client -o yaml > deployment.yaml
```

## Progression Méthodique

### Semaine 1-2 : Fondamentaux
- [ ] Installer et configurer MicroK8s
- [ ] Créer des pods, deployments, services
- [ ] Comprendre les labels et selectors
- [ ] Utiliser ConfigMaps et Secrets basiques

### Semaine 3-4 : Applications
- [ ] Déployer des applications multi-tiers
- [ ] Utiliser le stockage persistant
- [ ] Configurer Ingress et SSL
- [ ] Mettre en place monitoring basique

### Semaine 5-6 : Networking
- [ ] Comprendre le networking Kubernetes
- [ ] Configurer Network Policies
- [ ] Explorer les différents types de Services
- [ ] Debugger les problèmes réseau

### Semaine 7-8 : Avancé
- [ ] Patterns avancés (InitContainers, Sidecars)
- [ ] Health checks et readiness
- [ ] Autoscaling
- [ ] StatefulSets

### Semaine 9-10 : Opérations
- [ ] Troubleshooting avancé
- [ ] Backup et restore
- [ ] Upgrades
- [ ] Security best practices

### Semaine 11-12 : Maîtrise
- [ ] CI/CD avec Kubernetes
- [ ] Operators
- [ ] Multi-cluster
- [ ] Préparation certification

## Scripts Utiles pour le Playground

### Script de Réinitialisation

```bash
#!/bin/bash
# reset-playground.sh

echo "🔄 Resetting Kubernetes playground..."

# Supprimer tous les namespaces d'apprentissage
kubectl delete namespace basics apps networking storage-lab security-lab advanced sandbox --ignore-not-found=true

# Nettoyer les ressources cluster-wide
kubectl delete pv --all

# Nettoyer Docker
docker system prune -af

# Recréer les namespaces
kubectl create namespace basics
kubectl create namespace apps
kubectl create namespace networking
kubectl create namespace storage-lab
kubectl create namespace security-lab
kubectl create namespace advanced
kubectl create namespace sandbox

echo "✅ Playground reset complete!"
```

### Script de Diagnostic

```bash
#!/bin/bash
# diagnose.sh

echo "🔍 Kubernetes Cluster Diagnostics"
echo "=================================="

echo -e "\n📊 Cluster Info:"
kubectl cluster-info

echo -e "\n🖥️  Nodes:"
kubectl get nodes -o wide

echo -e "\n📦 Pods (All Namespaces):"
kubectl get pods --all-namespaces | grep -v Running | grep -v Completed

echo -e "\n💾 Storage:"
kubectl get pv,pvc --all-namespaces

echo -e "\n🌐 Services:"
kubectl get svc --all-namespaces

echo -e "\n⚠️  Events (Last 10):"
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -10

echo -e "\n📈 Resource Usage:"
kubectl top nodes
kubectl top pods --all-namespaces --sort-by=memory | head -10

echo -e "\n✅ Diagnostics complete!"
```

## Conclusion

Votre learning playground Kubernetes avec MicroK8s est maintenant prêt ! Cet environnement vous offre :

✅ Un espace sûr pour apprendre et expérimenter
✅ Des scénarios pratiques progressifs
✅ Des exemples concrets et réutilisables
✅ Une préparation aux certifications
✅ Une compréhension profonde de Kubernetes

**Points clés à retenir** :

- **Pratiquez régulièrement** : La répétition est la clé de l'apprentissage
- **Cassez les choses** : N'ayez pas peur d'expérimenter et d'échouer
- **Documentez** : Prenez des notes sur ce que vous apprenez
- **Progression graduelle** : Maîtrisez chaque niveau avant de passer au suivant
- **Utilisez la communauté** : Posez des questions, partagez vos découvertes

**Prochaines étapes** :

1. Commencez par le Niveau 1 (Fondamentaux)
2. Pratiquez chaque concept jusqu'à le maîtriser
3. Créez vos propres scénarios
4. Explorez la documentation officielle
5. Préparez-vous aux certifications si c'est votre objectif

**Ressources** :
- Revisitez ce guide régulièrement
- Explorez les autres chapitres pour des cas d'usage spécifiques
- Rejoignez la communauté Kubernetes pour continuer à apprendre

**Bon apprentissage ! 📚🚀**

⏭️ [Démonstrations et PoC](/24-cas-dusage-pratiques/05-demonstrations-et-poc.md)
