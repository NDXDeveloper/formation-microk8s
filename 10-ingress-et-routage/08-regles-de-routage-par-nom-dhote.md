🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.8 Règles de routage par nom d'hôte

## Introduction

Vous avez maintenant un nom de domaine configuré, le DNS qui pointe vers votre serveur, les ports redirigés, et un firewall qui protège votre infrastructure. Il est temps de mettre tout cela en pratique en créant vos **premières règles Ingress** !

Le **routage par nom d'hôte** (host-based routing) est la méthode la plus courante et la plus simple pour exposer plusieurs applications : chaque application a son propre sous-domaine. C'est exactement ce que font les grands services web que vous connaissez.

Dans ce chapitre, nous allons apprendre à créer des règles Ingress pour router le trafic vers différentes applications en fonction du nom de domaine demandé.

## Qu'est-ce que le Routage par Nom d'Hôte ?

### Concept de Base

Le **routage par nom d'hôte** (ou host-based routing) signifie que l'Ingress Controller examine le **nom de domaine** dans l'URL et dirige la requête vers l'application correspondante.

**Exemple concret** :
```
blog.mon-site.com      → Application Blog
api.mon-site.com       → API REST
admin.mon-site.com     → Tableau de bord admin
shop.mon-site.com      → Boutique en ligne
```

Chaque sous-domaine pointe vers un Service Kubernetes différent, qui lui-même dirige vers un ensemble de pods.

### Analogie du Standard Téléphonique

Imaginez un standard téléphonique d'entreprise :

**Standard** = Ingress Controller
**Numéro principal** = IP publique unique
**Extensions** = Sous-domaines

Quand quelqu'un appelle :
- **Extension 101** (blog) → Transféré au service Blog
- **Extension 102** (api) → Transféré au service API
- **Extension 103** (admin) → Transféré au service Admin

Le standard (Ingress) examine l'extension demandée et transfère l'appel au bon service.

### Flux Complet d'une Requête

Voici ce qui se passe quand un utilisateur visite `blog.mon-site.com` :

```
1. Utilisateur tape "blog.mon-site.com" dans le navigateur
   ↓
2. DNS résout vers l'IP publique (203.0.113.45)
   ↓
3. Requête HTTP arrive sur l'IP publique port 80
   ↓
4. Box/Routeur redirige vers le serveur MicroK8s (192.168.1.100:80)
   ↓
5. Firewall autorise le port 80
   ↓
6. NGINX Ingress Controller reçoit la requête
   ↓
7. Ingress examine l'en-tête "Host: blog.mon-site.com"
   ↓
8. Ingress consulte les règles configurées
   ↓
9. Trouve la règle : blog.mon-site.com → Service "blog"
   ↓
10. Service "blog" transmet à un pod disponible
   ↓
11. Pod traite et renvoie la réponse
   ↓
12. Réponse remonte jusqu'au navigateur de l'utilisateur
```

Tout cela se passe en quelques millisecondes !

## Prérequis

Avant de créer des règles Ingress, assurez-vous d'avoir :

### 1. Applications Déployées

Vous avez besoin d'au moins une application déployée dans Kubernetes avec :
- **Deployment** : qui gère les pods
- **Service** : qui expose le Deployment en interne

**Exemple** : Une application web simple écoutant sur le port 80.

### 2. NGINX Ingress Controller Actif

Vérifiez que l'Ingress Controller fonctionne :

```bash
microk8s kubectl get pods -n ingress
```

Vous devriez voir un pod en état `Running`.

### 3. DNS Configuré

Vos sous-domaines doivent pointer vers votre serveur (wildcard ou enregistrements A spécifiques).

### 4. Ports Accessibles

Les ports 80 et 443 doivent être accessibles depuis Internet (redirection de ports + firewall configurés).

## Anatomie d'une Règle Ingress

Voici à quoi ressemble une règle Ingress basique :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: mon-app.mon-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-service
            port:
              number: 80
```

### Décortiquons Chaque Section

#### 1. En-tête Standard

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
```

- `apiVersion` : Version de l'API Kubernetes (v1 est la version stable)
- `kind` : Type de ressource (ici Ingress)

#### 2. Métadonnées

```yaml
metadata:
  name: mon-ingress
  namespace: default
```

- `name` : Nom unique de votre ressource Ingress
- `namespace` : Namespace où créer l'Ingress (généralement `default`)

#### 3. Spécification : IngressClass

```yaml
spec:
  ingressClassName: nginx
```

- `ingressClassName` : Indique quel Ingress Controller utiliser (nginx dans notre cas)
- Important si vous avez plusieurs Ingress Controllers

#### 4. Règles de Routage

```yaml
  rules:
  - host: mon-app.mon-site.com
```

- `host` : Le nom de domaine qui déclenche cette règle
- **C'est la partie clé du routage par nom d'hôte !**

#### 5. Configuration HTTP

```yaml
    http:
      paths:
      - path: /
        pathType: Prefix
```

- `path` : Chemin de l'URL (ici `/` = toute l'application)
- `pathType` :
  - `Prefix` : commence par ce chemin
  - `Exact` : correspond exactement

#### 6. Backend (Destination)

```yaml
        backend:
          service:
            name: mon-service
            port:
              number: 80
```

- `service.name` : Nom du Service Kubernetes cible
- `port.number` : Port du Service

**Signification complète** : "Quand quelqu'un visite `mon-app.mon-site.com`, dirige-le vers le Service `mon-service` sur le port 80"

## Premier Exemple Pratique : Application Unique

### Scénario

Vous avez déployé une application web simple et voulez la rendre accessible via `app.mon-site.com`.

### Étape 1 : Créer un Déploiement et un Service

D'abord, déployons une application simple (par exemple nginx) :

```yaml
# app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app
  namespace: default
spec:
  replicas: 2
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
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: mon-app-service
  namespace: default
spec:
  selector:
    app: mon-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

Appliquez :
```bash
microk8s kubectl apply -f app-deployment.yaml
```

Vérifiez :
```bash
microk8s kubectl get pods
microk8s kubectl get services
```

### Étape 2 : Créer la Règle Ingress

Créez le fichier `app-ingress.yaml` :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
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
            name: mon-app-service
            port:
              number: 80
```

Appliquez :
```bash
microk8s kubectl apply -f app-ingress.yaml
```

### Étape 3 : Vérifier l'Ingress

```bash
microk8s kubectl get ingress
```

Sortie attendue :
```
NAME          CLASS   HOSTS              ADDRESS         PORTS   AGE
app-ingress   nginx   app.mon-site.com   192.168.1.100   80      10s
```

**Points à vérifier** :
- `HOSTS` : doit montrer votre domaine
- `ADDRESS` : l'IP de votre serveur (peut prendre quelques secondes)
- `PORTS` : 80 (HTTP)

### Étape 4 : Tester

Ouvrez votre navigateur et allez sur :
```
http://app.mon-site.com
```

Vous devriez voir la page d'accueil par défaut de nginx !

**Si cela ne fonctionne pas**, consultez la section Dépannage plus bas.

## Deuxième Exemple : Plusieurs Applications

### Scénario

Vous voulez exposer trois applications différentes sur trois sous-domaines :
- `blog.mon-site.com` → Application blog
- `api.mon-site.com` → API REST
- `admin.mon-site.com` → Interface admin

### Architecture Cible

```
Internet
   ↓
[Ingress Controller]
   ↓
   ├─→ blog.mon-site.com  → Service "blog"  → Pods Blog
   ├─→ api.mon-site.com   → Service "api"   → Pods API
   └─→ admin.mon-site.com → Service "admin" → Pods Admin
```

### Méthode 1 : Ingress Séparés (Recommandé)

Créez un Ingress pour chaque application. C'est plus modulaire et facile à gérer.

**blog-ingress.yaml** :
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: blog-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: blog.mon-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: blog-service
            port:
              number: 80
```

**api-ingress.yaml** :
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: default
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
            name: api-service
            port:
              number: 8080
```

**admin-ingress.yaml** :
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: admin-ingress
  namespace: default
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
            name: admin-service
            port:
              number: 3000
```

Appliquez tous les fichiers :
```bash
microk8s kubectl apply -f blog-ingress.yaml
microk8s kubectl apply -f api-ingress.yaml
microk8s kubectl apply -f admin-ingress.yaml
```

### Méthode 2 : Ingress Unique (Alternatif)

Vous pouvez aussi mettre toutes les règles dans un seul Ingress :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-app-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: blog.mon-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: blog-service
            port:
              number: 80
  - host: api.mon-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
  - host: admin.mon-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 3000
```

**Avantages de la méthode 1 (Ingress séparés)** :
- ✅ Plus facile à maintenir
- ✅ Modifications indépendantes
- ✅ Peut avoir des annotations différentes par app
- ✅ Suppression simple d'une app

**Avantages de la méthode 2 (Ingress unique)** :
- ✅ Un seul fichier à gérer
- ✅ Vue d'ensemble centralisée

**Recommandation** : Utilisez des Ingress séparés pour plus de flexibilité.

## Vérification et Tests

### Vérifier Tous les Ingress

```bash
microk8s kubectl get ingress
```

Sortie attendue :
```
NAME              CLASS   HOSTS                ADDRESS         PORTS   AGE
blog-ingress      nginx   blog.mon-site.com    192.168.1.100   80      5m
api-ingress       nginx   api.mon-site.com     192.168.1.100   80      5m
admin-ingress     nginx   admin.mon-site.com   192.168.1.100   80      5m
```

### Voir les Détails d'un Ingress

```bash
microk8s kubectl describe ingress blog-ingress
```

Sortie importante :
```
Name:             blog-ingress
Namespace:        default
Address:          192.168.1.100
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host                Path  Backends
  ----                ----  --------
  blog.mon-site.com
                      /   blog-service:80 (10.1.64.5:80,10.1.64.6:80)
Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  Sync    1m    nginx-ingress-controller  Scheduled for sync
```

**Points clés** :
- `Address` : IP de l'Ingress Controller
- `Backends` : Les IPs des pods cibles
- `Events` : Confirmations de synchronisation

### Tester Chaque Application

**Depuis votre navigateur** :
```
http://blog.mon-site.com
http://api.mon-site.com
http://admin.mon-site.com
```

**Depuis la ligne de commande** :
```bash
curl -H "Host: blog.mon-site.com" http://votre-ip-publique
curl -H "Host: api.mon-site.com" http://votre-ip-publique
curl -H "Host: admin.mon-site.com" http://votre-ip-publique
```

L'option `-H "Host: ..."` permet de tester même si le DNS n'est pas configuré.

### Vérifier les Logs de l'Ingress Controller

Pour voir les requêtes en temps réel :

```bash
microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx -f
```

Vous verrez défiler les requêtes HTTP :
```
192.168.1.50 - - [24/Oct/2025:10:15:22 +0000] "GET / HTTP/1.1" 200 612 "-" "Mozilla/5.0..."
```

## Utilisation de Wildcards DNS

Si vous avez configuré un wildcard DNS (`*.mon-site.com`), vous pouvez créer des Ingress pour n'importe quel sous-domaine sans modifier le DNS.

**Exemple** : Créer rapidement un environnement de test

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: test.mon-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: test-service
            port:
              number: 80
```

Appliquez et testez immédiatement `http://test.mon-site.com` - pas besoin de toucher au DNS !

## Règles Avancées

### 1. Domaine Racine et www

Pour gérer à la fois `mon-site.com` et `www.mon-site.com` :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  namespace: default
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
            name: frontend-service
            port:
              number: 80
  - host: www.mon-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

Les deux domaines pointent vers le même service.

### 2. Redirection www vers Non-www

Pour rediriger automatiquement `www.mon-site.com` vers `mon-site.com` :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: www-redirect
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/permanent-redirect: https://mon-site.com$request_uri
spec:
  ingressClassName: nginx
  rules:
  - host: www.mon-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### 3. Plusieurs Domaines vers Même Application

Si vous avez plusieurs domaines pointant vers la même application :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-domain-ingress
  namespace: default
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
            name: app-service
            port:
              number: 80
  - host: mon-autre-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

### 4. Environnements Multiples (Dev, Staging, Prod)

Organisation typique pour différents environnements :

```yaml
# Production
- host: app.mon-site.com → app-prod-service

# Staging
- host: staging.mon-site.com → app-staging-service

# Développement
- host: dev.mon-site.com → app-dev-service
```

Chaque environnement a son propre Service et ses propres pods.

## Annotations Utiles pour le Routage par Hôte

### Redirection HTTPS

Pour forcer HTTPS (nous verrons les certificats au chapitre 11) :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

### Taille des Requêtes

Pour permettre les uploads de fichiers :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
```

### Timeouts Personnalisés

Pour les applications qui prennent du temps à répondre :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
```

### CORS pour les APIs

Pour permettre les requêtes cross-origin :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
```

## Dépannage

### Problème 1 : 404 Not Found

**Symptôme** : Vous visitez votre domaine mais obtenez "404 Not Found" de NGINX.

**Causes possibles** :
1. Le nom d'hôte dans l'Ingress ne correspond pas au domaine visité
2. Le Service n'existe pas
3. Le Service ne pointe vers aucun pod

**Solutions** :

1. Vérifiez l'orthographe du host :
   ```bash
   microk8s kubectl get ingress -o yaml | grep host
   ```

2. Vérifiez que le Service existe :
   ```bash
   microk8s kubectl get service
   ```

3. Vérifiez que le Service a des endpoints :
   ```bash
   microk8s kubectl get endpoints
   ```

4. Vérifiez les logs de l'Ingress :
   ```bash
   microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx --tail=50
   ```

### Problème 2 : 503 Service Unavailable

**Symptôme** : Erreur 503 lors de la visite du site.

**Causes** :
- Aucun pod n'est disponible pour le Service
- Les pods ne sont pas prêts (not ready)
- Le Service pointe vers le mauvais port

**Solutions** :

1. Vérifiez les pods :
   ```bash
   microk8s kubectl get pods
   ```

2. Vérifiez que les pods sont READY :
   ```bash
   NAME           READY   STATUS    RESTARTS   AGE
   mon-app-xxx    1/1     Running   0          5m
   ```

3. Vérifiez les endpoints du Service :
   ```bash
   microk8s kubectl describe service mon-app-service
   ```

4. Assurez-vous que le port du Service correspond au containerPort des pods

### Problème 3 : Connexion Impossible

**Symptôme** : Le site ne charge pas du tout (timeout).

**Causes** :
- DNS non propagé
- Ports non redirigés
- Firewall bloque le trafic
- Ingress Controller non démarré

**Solutions** :

1. Vérifiez le DNS :
   ```bash
   dig mon-site.com +short
   ```

2. Vérifiez que l'Ingress Controller fonctionne :
   ```bash
   microk8s kubectl get pods -n ingress
   ```

3. Vérifiez les ports sur le serveur :
   ```bash
   sudo ss -tlnp | grep -E ':80|:443'
   ```

4. Testez en local pour isoler le problème :
   ```bash
   curl -H "Host: mon-site.com" http://localhost
   ```

### Problème 4 : Mauvaise Application Affichée

**Symptôme** : Vous visitez `blog.mon-site.com` mais voyez le contenu de `api.mon-site.com`.

**Cause** : Règles Ingress qui se chevauchent ou règle par défaut.

**Solutions** :

1. Listez tous les Ingress :
   ```bash
   microk8s kubectl get ingress -A
   ```

2. Vérifiez qu'il n'y a pas de duplication de hosts :
   ```bash
   microk8s kubectl get ingress -o yaml | grep -A 1 "host:"
   ```

3. Supprimez les Ingress en conflit :
   ```bash
   microk8s kubectl delete ingress <nom-ingress-conflit>
   ```

### Problème 5 : Ingress Ne S'applique Pas

**Symptôme** : Vous créez/modifiez l'Ingress mais rien ne change.

**Causes** :
- Erreur de syntaxe YAML
- Mauvais namespace
- IngressClass incorrecte

**Solutions** :

1. Vérifiez la création :
   ```bash
   microk8s kubectl get ingress
   ```

2. Regardez les événements :
   ```bash
   microk8s kubectl describe ingress mon-ingress
   ```

3. Vérifiez les erreurs :
   ```bash
   microk8s kubectl get events --sort-by='.lastTimestamp'
   ```

4. Validez votre YAML :
   ```bash
   microk8s kubectl apply -f mon-ingress.yaml --dry-run=client
   ```

## Gestion des Ingress

### Lister Tous les Ingress

```bash
# Dans le namespace par défaut
microk8s kubectl get ingress

# Dans tous les namespaces
microk8s kubectl get ingress -A

# Avec plus de détails
microk8s kubectl get ingress -o wide
```

### Voir la Configuration Complète

```bash
microk8s kubectl get ingress mon-ingress -o yaml
```

### Éditer un Ingress

```bash
microk8s kubectl edit ingress mon-ingress
```

Cela ouvre l'éditeur pour modifier directement.

### Supprimer un Ingress

```bash
microk8s kubectl delete ingress mon-ingress
```

Ou via le fichier :
```bash
microk8s kubectl delete -f mon-ingress.yaml
```

### Exporter un Ingress

Pour sauvegarder la configuration :

```bash
microk8s kubectl get ingress mon-ingress -o yaml > mon-ingress-backup.yaml
```

## Organisation et Bonnes Pratiques

### 1. Nommage Cohérent

Utilisez des noms descriptifs et cohérents :

```
blog-ingress        # Pour le blog
api-ingress         # Pour l'API
admin-ingress       # Pour l'admin
shop-ingress        # Pour la boutique
```

### 2. Namespaces par Projet

Organisez vos applications par namespace :

```
namespace: blog       → blog-ingress
namespace: api        → api-ingress
namespace: shop       → shop-ingress
```

### 3. Labels et Annotations

Ajoutez des labels pour organiser :

```yaml
metadata:
  name: blog-ingress
  labels:
    app: blog
    environment: production
    team: content
```

### 4. Documentation dans les Annotations

Documentez vos choix :

```yaml
metadata:
  annotations:
    description: "Ingress pour le blog de production"
    owner: "team-content@example.com"
    created-by: "Jean Dupont"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

### 5. Versionning avec Git

Stockez tous vos manifestes dans Git :

```
my-k8s-configs/
├── ingress/
│   ├── blog-ingress.yaml
│   ├── api-ingress.yaml
│   └── admin-ingress.yaml
├── deployments/
└── services/
```

Commitez chaque changement avec un message clair.

### 6. Utiliser Kustomize ou Helm

Pour les projets complexes, utilisez des outils de templating :

**Kustomize** (intégré à kubectl) :
```bash
microk8s kubectl apply -k ./ingress/
```

**Helm** (gestionnaire de packages) :
- Templates réutilisables
- Gestion de versions
- Rollbacks faciles

### 7. Tester Avant la Production

Créez d'abord un Ingress pour un environnement de staging :

```yaml
- host: staging.mon-site.com
```

Testez complètement, puis créez l'Ingress de production :

```yaml
- host: mon-site.com
```

## Template de Base Réutilisable

Voici un template que vous pouvez réutiliser pour vos applications :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <APP-NAME>-ingress
  namespace: <NAMESPACE>
  labels:
    app: <APP-NAME>
    environment: <ENV>
  annotations:
    # Redirection HTTPS (décommentez quand SSL configuré)
    # nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Taille max des uploads (ajustez selon besoin)
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"

    # Timeouts (ajustez si app lente)
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
spec:
  ingressClassName: nginx
  rules:
  - host: <APP-SUBDOMAIN>.<YOUR-DOMAIN>.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: <SERVICE-NAME>
            port:
              number: <SERVICE-PORT>
  # TLS (décommentez quand certificats configurés - Chapitre 11)
  # tls:
  # - hosts:
  #   - <APP-SUBDOMAIN>.<YOUR-DOMAIN>.com
  #   secretName: <APP-NAME>-tls
```

Remplacez les `<...>` par vos valeurs.

## Commandes de Référence Rapide

```bash
# Créer un Ingress
microk8s kubectl apply -f mon-ingress.yaml

# Lister les Ingress
microk8s kubectl get ingress
microk8s kubectl get ingress -A

# Voir les détails
microk8s kubectl describe ingress mon-ingress

# Voir la config YAML
microk8s kubectl get ingress mon-ingress -o yaml

# Éditer
microk8s kubectl edit ingress mon-ingress

# Supprimer
microk8s kubectl delete ingress mon-ingress

# Logs de l'Ingress Controller
microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx -f

# Tester avec curl (sans DNS)
curl -H "Host: mon-site.com" http://localhost

# Vérifier les endpoints
microk8s kubectl get endpoints

# Événements récents
microk8s kubectl get events --sort-by='.lastTimestamp'
```

## Checklist

Avant de passer au chapitre suivant, assurez-vous que :

- [ ] Vous avez créé au moins un Ingress fonctionnel
- [ ] Votre application est accessible via son nom de domaine
- [ ] Vous comprenez la structure d'un manifeste Ingress
- [ ] Vous savez lister et vérifier vos Ingress (`kubectl get ingress`)
- [ ] Vous savez consulter les logs de l'Ingress Controller
- [ ] Vous avez testé le routage vers plusieurs applications (si applicable)
- [ ] Vos manifestes sont sauvegardés (Git ou backup)
- [ ] Vous avez documenté vos règles de routage

## Points Clés à Retenir

🔑 **Routage par nom d'hôte** : chaque sous-domaine pointe vers un Service différent

🔑 **Une règle Ingress** = un mapping "domaine → Service"

🔑 **IngressClassName** : spécifie quel contrôleur utiliser (nginx)

🔑 **Host** : le nom de domaine qui déclenche la règle

🔑 **Backend** : le Service Kubernetes cible et son port

🔑 **Wildcard DNS** : facilite la création rapide de nouveaux sous-domaines

🔑 **Ingress séparés** : plus facile à maintenir que tout dans un seul

🔑 **Annotations** : personnalisent le comportement (CORS, timeouts, tailles)

🔑 **Logs essentiels** : consultez-les pour débugger (`kubectl logs -n ingress`)

## Prochaines Étapes

Maintenant que vous maîtrisez le routage par nom d'hôte, vous êtes prêt à :

1. **Apprendre le routage par chemin** (section 10.9) pour des URLs plus complexes comme `mon-site.com/blog`, `mon-site.com/api`
2. **Configurer des certificats SSL/TLS** (chapitre 11) pour sécuriser vos applications en HTTPS
3. **Ajouter des fonctionnalités avancées** (middleware, authentification, rate limiting)

Le routage par nom d'hôte est la base de l'exposition d'applications avec Ingress. Vous avez maintenant les compétences pour exposer autant d'applications que vous voulez, chacune sur son propre sous-domaine !

---

**📚 Résumé du chapitre** : Le routage par nom d'hôte permet d'exposer plusieurs applications sur différents sous-domaines (blog.mon-site.com, api.mon-site.com). Chaque règle Ingress mappe un nom de domaine vers un Service Kubernetes. Utilisez `ingressClassName: nginx`, spécifiez le `host`, et définissez le `backend` (service cible). Les Ingress séparés par application facilitent la maintenance. Consultez les logs avec `kubectl logs -n ingress` pour débugger.

⏭️ [Règles de routage par chemin](/10-ingress-et-routage/09-regles-de-routage-par-chemin.md)
