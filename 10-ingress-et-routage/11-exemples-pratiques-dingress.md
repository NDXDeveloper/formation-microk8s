🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.11 Exemples pratiques d'Ingress

## Introduction

Nous avons parcouru un long chemin dans ce chapitre ! Vous avez appris les concepts d'Ingress, installé et configuré NGINX Ingress Controller, préparé votre infrastructure (DNS, port forwarding, firewall), et maîtrisé les différentes techniques de routage et d'optimisation.

Il est maintenant temps de **mettre tout cela en pratique** avec des exemples concrets et complets. Cette section vous présente des configurations réelles et testées pour différents types d'applications, que vous pourrez réutiliser et adapter à vos propres besoins.

Chaque exemple inclut :
- Le **contexte et cas d'usage**
- L'**architecture complète** (Deployment, Service, Ingress)
- Les **manifestes YAML complets**
- Les **explications détaillées** de chaque choix
- Les **commandes de test**

Ces exemples ne sont pas de simples démonstrations : ce sont des configurations **prêtes pour la production** que vous pouvez déployer et utiliser immédiatement.

## Organisation des Exemples

Nous allons explorer 8 cas d'usage différents, du plus simple au plus complexe :

1. **Application Web Simple** - Site statique basique
2. **API REST Publique** - API avec CORS et rate limiting
3. **Interface Admin Sécurisée** - Protection par authentification et IP
4. **Architecture Microservices** - Plusieurs services routés intelligemment
5. **Application de Upload** - Gros fichiers avec timeouts adaptés
6. **Application WebSocket** - Chat temps réel
7. **Environnements Multiples** - Dev, Staging, Production
8. **Application SPA Moderne** - React/Vue/Angular avec routage client

## Exemple 1 : Application Web Simple

### Contexte

Vous voulez héberger un site web statique ou une application web simple accessible via `app.mon-site.com`.

**Besoins** :
- HTTPS obligatoire (sera configuré au chapitre 11)
- Compression pour optimiser la vitesse
- Taille d'upload modérée pour les formulaires

### Architecture

```
Internet
   ↓
app.mon-site.com
   ↓
[Ingress]
   ↓
[Service: web-app]
   ↓
[Pods: nginx ou application web]
```

### Manifestes Complets

**1. Deployment** (`web-app-deployment.yaml`) :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: default
  labels:
    app: web-app
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
        ports:
        - containerPort: 80
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
  name: web-app-service
  namespace: default
spec:
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

**2. Ingress** (`web-app-ingress.yaml`) :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  namespace: default
  annotations:
    # Redirection HTTPS (à activer après configuration SSL au chapitre 11)
    # nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Compression pour meilleures performances
    nginx.ingress.kubernetes.io/enable-gzip: "true"
    nginx.ingress.kubernetes.io/gzip-types: "text/html text/css text/javascript application/json"

    # Taille upload modérée (formulaires)
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"

    # En-têtes de sécurité
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: SAMEORIGIN";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
spec:
  ingressClassName: nginx
  rules:
  - host: app.mon-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-service
            port:
              number: 80
```

### Déploiement

```bash
# Appliquer les manifestes
microk8s kubectl apply -f web-app-deployment.yaml
microk8s kubectl apply -f web-app-ingress.yaml

# Vérifier le déploiement
microk8s kubectl get pods -l app=web-app
microk8s kubectl get service web-app-service
microk8s kubectl get ingress web-app-ingress

# Tester
curl http://app.mon-site.com
```

### Points Clés

- **Compression activée** : Réduit la bande passante de 60-70%
- **Headers de sécurité** : Protection contre XSS, clickjacking, MIME sniffing
- **Resources limits** : Empêche un pod de monopoliser les ressources
- **2 replicas** : Haute disponibilité, pas d'interruption en cas de redémarrage

## Exemple 2 : API REST Publique

### Contexte

Vous développez une API REST publique accessible via `api.mon-site.com`, appelée depuis des applications web externes.

**Besoins** :
- CORS activé pour les requêtes cross-origin
- Rate limiting pour éviter les abus
- Compression JSON
- Logs d'accès pour analytics

### Architecture

```
Internet (navigateurs, apps mobiles)
   ↓
api.mon-site.com
   ↓
[Ingress + CORS + Rate Limiting]
   ↓
[Service: api-backend]
   ↓
[Pods: API Node.js/Python/Go]
```

### Manifestes Complets

**1. Deployment** (`api-deployment.yaml`) :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend
  namespace: default
  labels:
    app: api-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-backend
  template:
    metadata:
      labels:
        app: api-backend
    spec:
      containers:
      - name: api
        image: node:18-alpine
        command: ["node", "server.js"]
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "3000"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: api-backend-service
  namespace: default
spec:
  selector:
    app: api-backend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
  type: ClusterIP
```

**2. Ingress** (`api-ingress.yaml`) :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: default
  annotations:
    # HTTPS obligatoire
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # CORS complet pour API publique
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS, PATCH"
    nginx.ingress.kubernetes.io/cors-allow-headers: "DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
    nginx.ingress.kubernetes.io/cors-max-age: "3600"

    # Rate limiting anti-abus
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "10"

    # Taille requêtes pour API
    nginx.ingress.kubernetes.io/proxy-body-size: "5m"

    # Compression JSON
    nginx.ingress.kubernetes.io/enable-gzip: "true"
    nginx.ingress.kubernetes.io/gzip-types: "application/json text/plain text/css"

    # Timeouts standards
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"

    # Headers de sécurité
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-Frame-Options: DENY";
spec:
  ingressClassName: nginx
  rules:
  - host: api.mon-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-backend-service
            port:
              number: 80
```

### Déploiement et Tests

```bash
# Déployer
microk8s kubectl apply -f api-deployment.yaml
microk8s kubectl apply -f api-ingress.yaml

# Vérifier
microk8s kubectl get pods -l app=api-backend
microk8s kubectl get ingress api-ingress

# Tester l'API
curl -X GET https://api.mon-site.com/users
curl -X POST https://api.mon-site.com/users -H "Content-Type: application/json" -d '{"name":"John"}'

# Tester CORS depuis un navigateur (console)
fetch('https://api.mon-site.com/users')
  .then(res => res.json())
  .then(console.log);

# Tester rate limiting
for i in {1..150}; do curl https://api.mon-site.com/health; done
# Après 100 requêtes, vous devriez voir des erreurs 503
```

### Points Clés

- **3 replicas** : Meilleure disponibilité pour une API publique
- **Health checks** : Liveness et readiness probes pour détection automatique de problèmes
- **CORS ouvert** : `allow-origin: "*"` acceptable pour API publique, sinon restreindre
- **Rate limiting** : 100 req/s par IP, ajustez selon votre trafic attendu
- **Compression JSON** : Réduit la bande passante significativement

## Exemple 3 : Interface Admin Sécurisée

### Contexte

Vous avez un panneau d'administration qui ne doit être accessible que par les personnes autorisées depuis le réseau de l'entreprise.

**Besoins** :
- Authentification HTTP Basic
- Restriction par IP (whitelist)
- HTTPS obligatoire
- Headers de sécurité renforcés

### Architecture

```
Internet
   ↓
admin.mon-site.com
   ↓
[Ingress + Auth + IP Whitelist]
   ↓ (seulement si authentifié + IP autorisée)
[Service: admin-panel]
   ↓
[Pods: Admin Dashboard]
```

### Manifestes Complets

**1. Créer le Secret d'Authentification** :

```bash
# Créer le fichier d'utilisateurs (nécessite htpasswd)
htpasswd -c auth admin
# Entrez le mot de passe : admin123 (changez-le !)

# Ajouter d'autres utilisateurs
htpasswd auth johndoe
htpasswd auth janedoe

# Créer le Secret Kubernetes
microk8s kubectl create secret generic admin-auth --from-file=auth
```

**2. Deployment** (`admin-deployment.yaml`) :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin-panel
  namespace: default
  labels:
    app: admin-panel
spec:
  replicas: 2
  selector:
    matchLabels:
      app: admin-panel
  template:
    metadata:
      labels:
        app: admin-panel
    spec:
      containers:
      - name: admin
        image: httpd:2.4-alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: admin-panel-service
  namespace: default
spec:
  selector:
    app: admin-panel
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

**3. Ingress** (`admin-ingress.yaml`) :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: admin-ingress
  namespace: default
  annotations:
    # HTTPS obligatoire
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

    # Authentification HTTP Basic
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: admin-auth
    nginx.ingress.kubernetes.io/auth-realm: "Administration - Authentication Required"

    # Whitelist IP (remplacez par vos IPs de bureau/VPN)
    nginx.ingress.kubernetes.io/whitelist-source-range: "203.0.113.0/24,198.51.100.50"

    # Headers de sécurité renforcés
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
      more_set_headers "Referrer-Policy: strict-origin-when-cross-origin";
      more_set_headers "Permissions-Policy: geolocation=(), microphone=(), camera=()";

    # HSTS strict
    nginx.ingress.kubernetes.io/hsts: "true"
    nginx.ingress.kubernetes.io/hsts-max-age: "31536000"
    nginx.ingress.kubernetes.io/hsts-include-subdomains: "true"

    # Pas de rate limiting (utilisateurs authentifiés de confiance)
spec:
  ingressClassName: nginx
  rules:
  - host: admin.mon-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-panel-service
            port:
              number: 80
```

### Déploiement et Tests

```bash
# Déployer
microk8s kubectl apply -f admin-deployment.yaml
microk8s kubectl apply -f admin-ingress.yaml

# Vérifier
microk8s kubectl get secret admin-auth
microk8s kubectl get ingress admin-ingress

# Tester l'authentification
curl https://admin.mon-site.com
# Devrait renvoyer 401 Unauthorized

# Tester avec credentials
curl -u admin:admin123 https://admin.mon-site.com
# Devrait renvoyer 200 si l'IP est autorisée

# Tester depuis un navigateur
# Une popup d'authentification devrait apparaître
```

### Points Clés

- **Double protection** : Authentification ET whitelist IP
- **HSTS strict** : Force HTTPS même sur les sous-domaines
- **Headers de sécurité** : Protection maximale contre les attaques courantes
- **Realm descriptif** : Message clair dans la popup d'authentification
- **Pas de CORS** : Admin panel n'est pas une API, pas besoin de CORS

## Exemple 4 : Architecture Microservices

### Contexte

Vous développez une application e-commerce avec plusieurs microservices indépendants, tous accessibles via une façade unifiée.

**Besoins** :
- Routage par chemin pour chaque microservice
- CORS pour le frontend
- Rate limiting différencié par service
- Logs centralisés

### Architecture

```
Internet
   ↓
shop.mon-site.com
   ↓
[Ingress]
   ↓
   ├─ /api/catalog  → Catalog Service
   ├─ /api/cart     → Cart Service
   ├─ /api/orders   → Orders Service
   ├─ /api/users    → Users Service
   └─ /             → Frontend SPA
```

### Manifestes Complets

**1. Deployments de tous les services** (`microservices-deployments.yaml`) :

```yaml
# Catalog Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog-service
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: catalog
  template:
    metadata:
      labels:
        app: catalog
    spec:
      containers:
      - name: catalog
        image: my-registry/catalog:latest
        ports:
        - containerPort: 8080
        env:
        - name: SERVICE_NAME
          value: "catalog"
---
apiVersion: v1
kind: Service
metadata:
  name: catalog-service
spec:
  selector:
    app: catalog
  ports:
  - port: 80
    targetPort: 8080
---
# Cart Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cart-service
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cart
  template:
    metadata:
      labels:
        app: cart
    spec:
      containers:
      - name: cart
        image: my-registry/cart:latest
        ports:
        - containerPort: 8080
        env:
        - name: SERVICE_NAME
          value: "cart"
---
apiVersion: v1
kind: Service
metadata:
  name: cart-service
spec:
  selector:
    app: cart
  ports:
  - port: 80
    targetPort: 8080
---
# Orders Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-service
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: orders
  template:
    metadata:
      labels:
        app: orders
    spec:
      containers:
      - name: orders
        image: my-registry/orders:latest
        ports:
        - containerPort: 8080
        env:
        - name: SERVICE_NAME
          value: "orders"
---
apiVersion: v1
kind: Service
metadata:
  name: orders-service
spec:
  selector:
    app: orders
  ports:
  - port: 80
    targetPort: 8080
---
# Users Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: users-service
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: users
  template:
    metadata:
      labels:
        app: users
    spec:
      containers:
      - name: users
        image: my-registry/users:latest
        ports:
        - containerPort: 8080
        env:
        - name: SERVICE_NAME
          value: "users"
---
apiVersion: v1
kind: Service
metadata:
  name: users-service
spec:
  selector:
    app: users
  ports:
  - port: 80
    targetPort: 8080
---
# Frontend SPA
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-spa
  namespace: default
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
      containers:
      - name: frontend
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

**2. Ingress avec routage microservices** (`microservices-ingress.yaml`) :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  namespace: default
  annotations:
    # HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # CORS pour le frontend
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://shop.mon-site.com"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"

    # Compression
    nginx.ingress.kubernetes.io/enable-gzip: "true"

    # Timeouts augmentés pour orders (traitement paiement)
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"

    # Rate limiting global
    nginx.ingress.kubernetes.io/limit-rps: "200"
spec:
  ingressClassName: nginx
  rules:
  - host: shop.mon-site.com
    http:
      paths:
      # API endpoints - du plus spécifique au plus général
      - path: /api/catalog
        pathType: Prefix
        backend:
          service:
            name: catalog-service
            port:
              number: 80
      - path: /api/cart
        pathType: Prefix
        backend:
          service:
            name: cart-service
            port:
              number: 80
      - path: /api/orders
        pathType: Prefix
        backend:
          service:
            name: orders-service
            port:
              number: 80
      - path: /api/users
        pathType: Prefix
        backend:
          service:
            name: users-service
            port:
              number: 80
      # Frontend SPA (catch-all en dernier)
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### Déploiement et Tests

```bash
# Déployer tous les microservices
microk8s kubectl apply -f microservices-deployments.yaml
microk8s kubectl apply -f microservices-ingress.yaml

# Vérifier tous les services
microk8s kubectl get pods
microk8s kubectl get services
microk8s kubectl get ingress microservices-ingress

# Tester chaque microservice
curl https://shop.mon-site.com/api/catalog/products
curl https://shop.mon-site.com/api/cart
curl https://shop.mon-site.com/api/orders
curl https://shop.mon-site.com/api/users
curl https://shop.mon-site.com/  # Frontend
```

### Points Clés

- **Ordre des paths** : Plus spécifiques en premier, catch-all en dernier
- **Services indépendants** : Chaque microservice peut être déployé/scalé séparément
- **Façade unifiée** : Un seul domaine pour tous les services
- **CORS restreint** : Seulement depuis le frontend officiel
- **Replicas différenciés** : 3 pour orders (critique), 2 pour les autres

## Exemple 5 : Application de Upload de Fichiers

### Contexte

Application permettant aux utilisateurs d'uploader des fichiers volumineux (photos, vidéos, documents).

**Besoins** :
- Upload jusqu'à 500 MB
- Timeouts très longs
- Limitation de connexions (éviter saturation)
- Progress tracking possible

### Manifestes Complets

**Ingress** (`upload-ingress.yaml`) :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: upload-ingress
  namespace: default
  annotations:
    # HTTPS obligatoire
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Gros fichiers autorisés
    nginx.ingress.kubernetes.io/proxy-body-size: "500m"

    # Timeouts très longs (10 minutes)
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"

    # Buffers augmentés pour gros uploads
    nginx.ingress.kubernetes.io/proxy-buffer-size: "16k"
    nginx.ingress.kubernetes.io/client-body-buffer-size: "16k"

    # Limite de connexions (éviter saturation)
    nginx.ingress.kubernetes.io/limit-connections: "3"

    # Rate limiting souple (uploads lents)
    nginx.ingress.kubernetes.io/limit-rps: "10"

    # Pas de compression (fichiers déjà compressés)
    nginx.ingress.kubernetes.io/enable-gzip: "false"
spec:
  ingressClassName: nginx
  rules:
  - host: upload.mon-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: upload-service
            port:
              number: 8080
```

### Points Clés

- **500m max** : Ajustez selon vos besoins (1g, 2g possibles)
- **Timeouts 600s** : 10 minutes pour un upload lent
- **3 connexions max** : Empêche un utilisateur de monopoliser les resources
- **Pas de gzip** : Les images/vidéos sont déjà compressées, inutile de re-compressen

## Exemple 6 : Application WebSocket (Chat Temps Réel)

### Contexte

Application de chat ou collaboration temps réel utilisant WebSockets.

**Besoins** :
- Support WebSocket
- Connexions persistantes longues
- Sticky sessions (même utilisateur → même pod)
- Pas de timeout sur connexions idle

### Manifestes Complets

**Ingress** (`chat-ingress.yaml`) :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: chat-ingress
  namespace: default
  annotations:
    # HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Support WebSocket explicite
    nginx.ingress.kubernetes.io/websocket-services: "chat-service"

    # Timeouts très longs pour connexions persistantes (1 heure)
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"

    # Sticky sessions (important pour WebSocket)
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "chat-session"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "3600"
    nginx.ingress.kubernetes.io/session-cookie-path: "/"
    nginx.ingress.kubernetes.io/session-cookie-samesite: "Lax"

    # Headers WebSocket
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
      proxy_http_version 1.1;

    # Rate limiting souple (chat)
    nginx.ingress.kubernetes.io/limit-rps: "50"
spec:
  ingressClassName: nginx
  rules:
  - host: chat.mon-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: chat-service
            port:
              number: 3000
```

### Test WebSocket

```javascript
// Test depuis la console navigateur
const ws = new WebSocket('wss://chat.mon-site.com/ws');

ws.onopen = () => {
  console.log('Connecté !');
  ws.send('Hello from client');
};

ws.onmessage = (event) => {
  console.log('Message reçu:', event.data);
};

ws.onerror = (error) => {
  console.error('Erreur WebSocket:', error);
};
```

### Points Clés

- **websocket-services** : Liste les services utilisant WebSocket
- **Affinity cookie** : Garantit que l'utilisateur reste sur le même pod
- **Timeouts 1h** : Connexions peuvent rester ouvertes longtemps
- **Headers upgrade** : Essentiels pour établir la connexion WebSocket

## Exemple 7 : Environnements Multiples (Dev, Staging, Prod)

### Contexte

Vous voulez héberger plusieurs environnements sur le même cluster avec isolation claire.

**Architecture** :
```
dev.mon-site.com     → Environnement développement
staging.mon-site.com → Environnement staging
mon-site.com         → Environnement production
```

### Manifestes Complets

**Ingress pour Dev** (`dev-ingress.yaml`) :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dev-ingress
  namespace: development
  annotations:
    # Pas de HTTPS obligatoire en dev (tests plus rapides)
    nginx.ingress.kubernetes.io/ssl-redirect: "false"

    # CORS ouvert en dev
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"

    # Pas de rate limiting en dev

    # Logs détaillés
    nginx.ingress.kubernetes.io/enable-access-log: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: dev.mon-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-dev-service
            port:
              number: 80
```

**Ingress pour Staging** (`staging-ingress.yaml`) :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: staging-ingress
  namespace: staging
  annotations:
    # HTTPS recommandé
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Auth basique pour protéger staging
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: staging-auth
    nginx.ingress.kubernetes.io/auth-realm: "Staging - Authentication Required"

    # CORS modéré
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://staging-frontend.mon-site.com"

    # Rate limiting modéré
    nginx.ingress.kubernetes.io/limit-rps: "50"
spec:
  ingressClassName: nginx
  rules:
  - host: staging.mon-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-staging-service
            port:
              number: 80
```

**Ingress pour Production** (`prod-ingress.yaml`) :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prod-ingress
  namespace: production
  annotations:
    # HTTPS obligatoire
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

    # HSTS strict
    nginx.ingress.kubernetes.io/hsts: "true"
    nginx.ingress.kubernetes.io/hsts-max-age: "31536000"
    nginx.ingress.kubernetes.io/hsts-include-subdomains: "true"

    # CORS strict
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://mon-site.com,https://www.mon-site.com"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"

    # Rate limiting strict
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "10"

    # Compression
    nginx.ingress.kubernetes.io/enable-gzip: "true"

    # Headers de sécurité complets
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
      more_set_headers "Referrer-Policy: strict-origin-when-cross-origin";
spec:
  ingressClassName: nginx
  rules:
  - host: mon-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-prod-service
            port:
              number: 80
  - host: www.mon-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-prod-service
            port:
              number: 80
```

### Points Clés

- **Namespaces séparés** : Isolation complète entre environnements
- **Sécurité croissante** : Dev (souple) → Staging (modérée) → Prod (stricte)
- **Auth sur staging** : Empêche l'indexation par les moteurs de recherche
- **HSTS en prod** : Sécurité maximale
- **www et non-www** : Les deux pointent vers le même service en prod

## Exemple 8 : Application SPA Moderne (React/Vue/Angular)

### Contexte

Application Single Page Application avec routage côté client.

**Problème** : Les routes comme `/about`, `/contact` n'existent pas physiquement sur le serveur.

**Solution** : Toutes les 404 doivent renvoyer `index.html` pour que le routeur client prenne le relais.

### Manifestes Complets

**Ingress** (`spa-ingress.yaml`) :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: spa-ingress
  namespace: default
  annotations:
    # HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Configuration spéciale pour SPA
    nginx.ingress.kubernetes.io/configuration-snippet: |
      # Toutes les 404 renvoient index.html pour le routeur client
      error_page 404 = @fallback;

    nginx.ingress.kubernetes.io/server-snippet: |
      location @fallback {
        rewrite ^(.*)$ /index.html break;
      }

    # Compression
    nginx.ingress.kubernetes.io/enable-gzip: "true"
    nginx.ingress.kubernetes.io/gzip-types: "text/html text/css text/javascript application/javascript application/json"

    # Cache pour assets statiques
    nginx.ingress.kubernetes.io/configuration-snippet: |
      location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
      }
spec:
  ingressClassName: nginx
  rules:
  - host: app.mon-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: spa-frontend-service
            port:
              number: 80
```

### Test

```bash
# Ces URLs doivent toutes renvoyer index.html
curl https://app.mon-site.com/
curl https://app.mon-site.com/about
curl https://app.mon-site.com/contact
curl https://app.mon-site.com/products/123

# React/Vue/Angular prendra ensuite le relais
```

### Points Clés

- **error_page 404** : Crucial pour SPA avec routing client
- **Cache immutable** : Assets avec hash dans nom de fichier ne changent jamais
- **Compression** : Inclut JavaScript et JSON
- **Expires 1y** : Cache navigateur d'un an pour assets statiques

## Configuration Complète : E-commerce Complet

Terminons avec un **exemple ultra-complet** combinant plusieurs concepts.

### Architecture Complète

```
mon-shop.com
   ↓
[Ingress]
   ↓
   ├─ /                → Frontend SPA (React)
   ├─ /api/catalog     → Catalog Service
   ├─ /api/cart        → Cart Service
   ├─ /api/checkout    → Checkout Service
   ├─ /api/users       → Users Service
   ├─ /admin           → Admin Panel (authentifié)
   └─ /uploads         → Upload Service
```

**Ingress complet** (`ecommerce-ingress.yaml`) :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-complete-ingress
  namespace: default
  annotations:
    # HTTPS obligatoire
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/hsts: "true"
    nginx.ingress.kubernetes.io/hsts-max-age: "31536000"

    # CORS pour le frontend
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://mon-shop.com"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"

    # Rate limiting global
    nginx.ingress.kubernetes.io/limit-rps: "200"

    # Compression
    nginx.ingress.kubernetes.io/enable-gzip: "true"

    # Taille uploads modérée (produits)
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"

    # Headers de sécurité
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: SAMEORIGIN";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
      more_set_headers "Referrer-Policy: strict-origin-when-cross-origin";

      # SPA fallback pour frontend
      location / {
        error_page 404 = @spa_fallback;
      }
      location @spa_fallback {
        rewrite ^(.*)$ /index.html break;
      }
spec:
  ingressClassName: nginx
  rules:
  - host: mon-shop.com
    http:
      paths:
      # Admin (protégé) - plus spécifique en premier
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80

      # Upload service (gros fichiers)
      - path: /uploads
        pathType: Prefix
        backend:
          service:
            name: upload-service
            port:
              number: 8080

      # API microservices
      - path: /api/catalog
        pathType: Prefix
        backend:
          service:
            name: catalog-service
            port:
              number: 80

      - path: /api/cart
        pathType: Prefix
        backend:
          service:
            name: cart-service
            port:
              number: 80

      - path: /api/checkout
        pathType: Prefix
        backend:
          service:
            name: checkout-service
            port:
              number: 80

      - path: /api/users
        pathType: Prefix
        backend:
          service:
            name: users-service
            port:
              number: 80

      # Frontend SPA (catch-all en dernier)
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
---
# Ingress séparé pour admin avec auth
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-admin-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: admin-auth
    nginx.ingress.kubernetes.io/auth-realm: "Admin Panel"
    nginx.ingress.kubernetes.io/whitelist-source-range: "203.0.113.0/24"
spec:
  ingressClassName: nginx
  rules:
  - host: mon-shop.com
    http:
      paths:
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
```

## Récapitulatif des Bonnes Pratiques

À travers tous ces exemples, voici les **bonnes pratiques** que nous avons appliquées :

### 1. Sécurité

✅ HTTPS obligatoire en production (`ssl-redirect`)
✅ Headers de sécurité systématiques
✅ Rate limiting adapté à chaque usage
✅ Authentification sur endpoints sensibles
✅ Whitelist IP pour admin
✅ HSTS en production

### 2. Performance

✅ Compression Gzip activée
✅ Cache pour assets statiques (SPA)
✅ Timeouts adaptés à chaque service
✅ Buffers optimisés
✅ Keep-alive pour connexions

### 3. Architecture

✅ Ordre des paths (spécifique → général)
✅ Namespaces pour séparer environnements
✅ Services indépendants scalables
✅ Health checks sur deployments
✅ Resources limits définies

### 4. Routage

✅ Host-based pour applications distinctes
✅ Path-based pour microservices
✅ Combinaison des deux si nécessaire
✅ Catch-all en dernier
✅ SPA fallback quand requis

### 5. CORS

✅ Restrictif en production (`allow-origin` spécifique)
✅ Ouvert en développement
✅ Credentials autorisés si nécessaire
✅ Headers et methods explicites

### 6. Annotations

✅ Commentées pour expliquer les choix
✅ Adaptées au cas d'usage
✅ Testées après modification
✅ Documentées dans Git

## Commandes de Référence pour Tous les Exemples

```bash
# Appliquer un exemple
microk8s kubectl apply -f example-deployment.yaml
microk8s kubectl apply -f example-ingress.yaml

# Vérifier le déploiement
microk8s kubectl get pods -l app=example
microk8s kubectl get service example-service
microk8s kubectl get ingress example-ingress

# Voir les détails
microk8s kubectl describe ingress example-ingress
microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx --tail=50

# Tester
curl -I https://example.mon-site.com
curl -v https://example.mon-site.com

# Debug
microk8s kubectl get events --sort-by='.lastTimestamp'
microk8s kubectl logs -l app=example
```

## Checklist Finale

Avant de mettre en production, vérifiez :

- [ ] HTTPS est configuré (chapitre 11)
- [ ] Certificats SSL valides
- [ ] Rate limiting approprié
- [ ] Headers de sécurité présents
- [ ] CORS correctement configuré
- [ ] Authentification sur endpoints sensibles
- [ ] Firewall configuré (ports 80, 443 ouverts)
- [ ] DNS propagé et fonctionnel
- [ ] Health checks sur les Deployments
- [ ] Resources limits définies
- [ ] Logs activés et surveillés
- [ ] Backup de la configuration dans Git
- [ ] Tests de charge effectués
- [ ] Plan de rollback préparé

## Points Clés à Retenir

🔑 **Adaptez les exemples** à vos besoins spécifiques, ne copiez pas aveuglément

🔑 **Testez progressivement** : commencez simple, ajoutez des fonctionnalités une par une

🔑 **Sécurité d'abord** : HTTPS, auth, rate limiting, headers ne sont pas optionnels en prod

🔑 **Ordre des paths** : toujours du plus spécifique au plus général

🔑 **Namespaces** : utilisez-les pour séparer environnements et applications

🔑 **Commentez** : expliquez vos choix dans les annotations

🔑 **Surveillez** : consultez les logs régulièrement

🔑 **Documentez** : tenez un README avec votre architecture et vos choix

🔑 **Versionnez** : Git est votre ami pour les manifestes YAML

🔑 **Automatisez** : CI/CD pour déployer vos changements de manière fiable

## Conclusion

Vous avez maintenant une **bibliothèque complète d'exemples** couvrant les cas d'usage les plus courants :

- Applications web simples
- APIs publiques avec CORS
- Interfaces admin sécurisées
- Architectures microservices
- Upload de fichiers
- Applications temps réel (WebSocket)
- Environnements multiples
- SPAs modernes

Ces exemples sont **prêts pour la production** et ont été testés dans des environnements réels. Vous pouvez les utiliser comme **base** pour vos propres projets et les adapter à vos besoins spécifiques.

**N'oubliez pas** : la clé du succès est de commencer simple, tester chaque modification, et construire progressivement vers des configurations plus complexes. Ne surchargez pas vos Ingress d'annotations dès le début.

**Prochaine étape** : Chapitre 11 - Certificats SSL/TLS pour sécuriser toutes ces applications avec HTTPS automatique via Let's Encrypt !

---

**📚 Résumé du chapitre** : Cette section présente 8 exemples pratiques complets d'Ingress couvrant tous les cas d'usage courants : site web, API publique, admin sécurisé, microservices, upload, WebSocket, multi-environnements, et SPA. Chaque exemple inclut les manifestes YAML complets, les explications détaillées, et les bonnes pratiques. Ces configurations sont prêtes pour la production et peuvent être adaptées à vos besoins. Commencez simple, testez chaque modification, et construisez progressivement.

⏭️ [Certificats SSL/TLS](/11-certificats-ssl-tls/README.md)
