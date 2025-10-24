🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.9 Règles de routage par chemin

## Introduction

Dans la section précédente, vous avez appris à router le trafic en fonction du **nom de domaine** (host-based routing). Mais il existe une autre méthode tout aussi puissante : le **routage par chemin** (path-based routing).

Au lieu d'avoir un sous-domaine différent pour chaque application, vous pouvez utiliser différents **chemins d'URL** sur un même domaine. Par exemple :
- `mon-site.com/blog` → Application blog
- `mon-site.com/api` → API REST
- `mon-site.com/admin` → Interface admin

Cette approche est particulièrement utile pour les architectures microservices et les APIs, où vous voulez une façade unifiée pour plusieurs services.

## Routage par Chemin vs Routage par Nom d'Hôte

### Comparaison Visuelle

**Routage par nom d'hôte** (section 10.8) :
```
blog.mon-site.com    → Service Blog
api.mon-site.com     → Service API
admin.mon-site.com   → Service Admin
```

**Routage par chemin** (cette section) :
```
mon-site.com/blog    → Service Blog
mon-site.com/api     → Service API
mon-site.com/admin   → Service Admin
```

### Quand Utiliser Chaque Approche ?

| Critère | Routage par nom d'hôte | Routage par chemin |
|---------|------------------------|---------------------|
| **Séparation logique** | ⭐⭐⭐⭐⭐ Excellente | ⭐⭐⭐ Bonne |
| **Certificats SSL** | Un par sous-domaine ou wildcard | Un seul pour le domaine |
| **Configuration DNS** | Nécessite des enregistrements | Un seul enregistrement |
| **Lisibilité URLs** | ⭐⭐⭐⭐ Claire | ⭐⭐⭐⭐ Claire |
| **Microservices** | ⭐⭐⭐ Bien | ⭐⭐⭐⭐⭐ Parfait |
| **APIs publiques** | ⭐⭐⭐ Bien | ⭐⭐⭐⭐⭐ Idéal |
| **Complexité** | ⭐⭐⭐⭐ Simple | ⭐⭐⭐ Moyenne |

**Routage par nom d'hôte** : Préféré quand les applications sont vraiment distinctes et indépendantes.

**Routage par chemin** : Préféré pour les microservices, APIs, ou quand vous voulez une seule façade unifiée.

**Combinaison des deux** : Possible et courante ! Ex: `api.mon-site.com/v1`, `api.mon-site.com/v2`

## Analogie de l'Immeuble

Imaginez un **grand immeuble commercial** :

**Routage par nom d'hôte** = Immeubles séparés
- Immeuble A (blog.mon-site.com) = Tout l'immeuble pour le blog
- Immeuble B (api.mon-site.com) = Tout l'immeuble pour l'API
- Immeuble C (admin.mon-site.com) = Tout l'immeuble pour l'admin

**Routage par chemin** = Un seul immeuble avec différents étages/départements
- Rez-de-chaussée (mon-site.com/) = Accueil
- 1er étage (mon-site.com/blog) = Département blog
- 2ème étage (mon-site.com/api) = Département API
- 3ème étage (mon-site.com/admin) = Département admin

Le **réceptionniste** (Ingress) regarde l'étage demandé et vous dirige au bon endroit.

## Les Types de Chemin (PathType)

Kubernetes supporte trois types de correspondance de chemin :

### 1. Prefix (Le Plus Courant)

**Définition** : Correspond si l'URL **commence par** le chemin spécifié.

```yaml
pathType: Prefix
path: /blog
```

**Exemples de correspondance** :
- ✅ `mon-site.com/blog` → Correspond
- ✅ `mon-site.com/blog/` → Correspond
- ✅ `mon-site.com/blog/article-1` → Correspond
- ✅ `mon-site.com/blog/category/tech` → Correspond
- ❌ `mon-site.com/bloggers` → Ne correspond PAS (pas un préfixe valide)
- ❌ `mon-site.com/myblog` → Ne correspond PAS

**Cas d'usage** : Applications complètes montées sur un sous-chemin.

### 2. Exact

**Définition** : Correspond seulement si l'URL est **exactement** le chemin spécifié.

```yaml
pathType: Exact
path: /api
```

**Exemples de correspondance** :
- ✅ `mon-site.com/api` → Correspond
- ❌ `mon-site.com/api/` → Ne correspond PAS (le slash final fait une différence)
- ❌ `mon-site.com/api/users` → Ne correspond PAS
- ❌ `mon-site.com/apis` → Ne correspond PAS

**Cas d'usage** : Endpoints spécifiques, healthchecks, webhooks.

### 3. ImplementationSpecific

**Définition** : Comportement défini par l'Ingress Controller (NGINX a des règles spécifiques).

**Généralement évité** : Utilisez Prefix ou Exact pour un comportement prévisible.

### Recommandation

**Utilisez `Prefix`** dans 90% des cas. C'est le plus intuitif et le plus utilisé.

## Premier Exemple : Routage Simple par Chemin

### Scénario

Vous voulez exposer deux services sur un même domaine :
- `mon-site.com/` → Frontend (page d'accueil)
- `mon-site.com/api` → API backend

### Manifeste Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: mon-site.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### Points Importants

#### 1. Ordre des Chemins

**L'ordre compte !** Les chemins **plus spécifiques** doivent venir **avant** les chemins génériques.

✅ **Correct** :
```yaml
paths:
  - path: /api         # Plus spécifique en premier
  - path: /            # Générique en dernier
```

❌ **Incorrect** :
```yaml
paths:
  - path: /            # Générique en premier = capture tout !
  - path: /api         # Ne sera jamais atteint
```

**Pourquoi ?** NGINX évalue les règles dans l'ordre. Si `/` vient en premier, **toutes** les requêtes correspondront et `/api` ne sera jamais utilisé.

#### 2. Le Chemin Racine "/"

Le chemin `/` avec `Prefix` correspond à **tout** :
- `mon-site.com/`
- `mon-site.com/about`
- `mon-site.com/contact`
- `mon-site.com/any/thing`

C'est votre **route par défaut** (catch-all).

### Test de la Configuration

```bash
# Appliquer l'Ingress
microk8s kubectl apply -f path-based-ingress.yaml

# Vérifier
microk8s kubectl get ingress path-based-ingress

# Tester l'API
curl http://mon-site.com/api
curl http://mon-site.com/api/users

# Tester le frontend
curl http://mon-site.com/
curl http://mon-site.com/about
```

## Exemple Avancé : Architecture Microservices

### Scénario

Vous avez une architecture microservices typique :
- `mon-site.com/` → Frontend web
- `mon-site.com/api/users` → Service utilisateurs
- `mon-site.com/api/products` → Service produits
- `mon-site.com/api/orders` → Service commandes
- `mon-site.com/admin` → Interface administration

### Manifeste Ingress Complet

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: mon-site.com
    http:
      paths:
      # API endpoints - du plus spécifique au plus général
      - path: /api/users
        pathType: Prefix
        backend:
          service:
            name: users-service
            port:
              number: 8080
      - path: /api/products
        pathType: Prefix
        backend:
          service:
            name: products-service
            port:
              number: 8080
      - path: /api/orders
        pathType: Prefix
        backend:
          service:
            name: orders-service
            port:
              number: 8080
      # Admin
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 3000
      # Frontend (catch-all en dernier)
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### Flux de Requêtes

```
Requête : mon-site.com/api/users/123
   ↓
Ingress examine le chemin "/api/users/123"
   ↓
Correspond à la règle "/api/users" (Prefix)
   ↓
Dirige vers users-service:8080
   ↓
Service transmet à un pod users
```

```
Requête : mon-site.com/contact
   ↓
Ingress examine le chemin "/contact"
   ↓
Ne correspond ni à /api/users, ni /api/products, ni /api/orders, ni /admin
   ↓
Correspond à "/" (catch-all)
   ↓
Dirige vers frontend-service:80
```

## Réécriture d'URL (Rewrite Target)

### Le Problème

Par défaut, NGINX envoie le **chemin complet** au backend.

**Exemple** :
```
Requête : mon-site.com/api/users
         ↓
Backend reçoit : /api/users
```

Mais si votre service `users-service` écoute directement sur `/users` (sans le préfixe `/api`), vous aurez une erreur 404.

### La Solution : rewrite-target

L'annotation `rewrite-target` modifie le chemin avant de l'envoyer au backend.

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
```

**Avec rewrite-target** :
```
Requête : mon-site.com/api/users
         ↓
Backend reçoit : /users (le /api est enlevé)
```

### Exemple Complet avec Réécriture

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: default
  annotations:
    # Enlève le préfixe du chemin avant d'envoyer au backend
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: mon-site.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

**Explication** :
- `path: /api(/|$)(.*)` : Expression régulière qui capture tout après `/api`
- `$2` : Référence au deuxième groupe capturé (ce qui vient après `/api`)

**Résultat** :
```
mon-site.com/api/users    → Backend reçoit : /users
mon-site.com/api/products → Backend reçoit : /products
mon-site.com/api          → Backend reçoit : /
```

### Rewrite Simple (Plus Courant)

Pour la plupart des cas, utilisez cette version simplifiée :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: mon-site.com
    http:
      paths:
      - path: /blog
        pathType: Prefix
        backend:
          service:
            name: blog-service
            port:
              number: 80
```

**Comportement** :
```
mon-site.com/blog         → Backend reçoit : /
mon-site.com/blog/article → Backend reçoit : /article
```

Le préfixe `/blog` est supprimé.

## Combinaison : Routage par Hôte ET Chemin

Vous pouvez combiner les deux méthodes de routage pour une flexibilité maximale.

### Exemple : API Versionnée

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-versioned-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  # API v1
  - host: api.mon-site.com
    http:
      paths:
      - path: /v1/users
        pathType: Prefix
        backend:
          service:
            name: users-v1-service
            port:
              number: 8080
      - path: /v1/products
        pathType: Prefix
        backend:
          service:
            name: products-v1-service
            port:
              number: 8080
  # API v2
  - host: api.mon-site.com
    http:
      paths:
      - path: /v2/users
        pathType: Prefix
        backend:
          service:
            name: users-v2-service
            port:
              number: 8080
      - path: /v2/products
        pathType: Prefix
        backend:
          service:
            name: products-v2-service
            port:
              number: 8080
```

**Résultat** :
```
api.mon-site.com/v1/users    → users-v1-service
api.mon-site.com/v2/users    → users-v2-service
api.mon-site.com/v1/products → products-v1-service
api.mon-site.com/v2/products → products-v2-service
```

### Exemple : Multi-Tenant

```yaml
spec:
  rules:
  # Tenant A
  - host: tenant-a.mon-site.com
    http:
      paths:
      - path: /dashboard
        pathType: Prefix
        backend:
          service:
            name: dashboard-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-tenant-a
            port:
              number: 8080
  # Tenant B
  - host: tenant-b.mon-site.com
    http:
      paths:
      - path: /dashboard
        pathType: Prefix
        backend:
          service:
            name: dashboard-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-tenant-b
            port:
              number: 8080
```

## Applications SPA (Single Page Applications)

Les applications React, Vue, Angular ont besoin d'une configuration spéciale.

### Le Problème

Les SPA utilisent le routing côté client. Quand l'utilisateur visite :
```
mon-site.com/about
mon-site.com/contact
mon-site.com/products/123
```

Ces routes n'existent pas physiquement sur le serveur. L'application doit **toujours** renvoyer `index.html`.

### La Solution

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: spa-ingress
  namespace: default
  annotations:
    # Important pour SPA : toutes les erreurs 404 renvoient index.html
    nginx.ingress.kubernetes.io/configuration-snippet: |
      error_page 404 = @fallback;
    nginx.ingress.kubernetes.io/server-snippet: |
      location @fallback {
        rewrite ^(.*)$ /index.html break;
      }
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
            name: frontend-spa-service
            port:
              number: 80
```

**Comportement** :
```
mon-site.com/          → index.html
mon-site.com/about     → index.html (même si le fichier n'existe pas)
mon-site.com/contact   → index.html
```

React/Vue/Angular prend ensuite le relais pour afficher la bonne page.

## Cas d'Usage Réels

### 1. Blog avec API Backend

```yaml
spec:
  rules:
  - host: mon-site.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: blog-api
            port:
              number: 3000
      - path: /
        pathType: Prefix
        backend:
          service:
            name: blog-frontend
            port:
              number: 80
```

### 2. E-commerce Multi-Services

```yaml
spec:
  rules:
  - host: shop.mon-site.com
    http:
      paths:
      - path: /api/cart
        pathType: Prefix
        backend:
          service:
            name: cart-service
            port:
              number: 8080
      - path: /api/checkout
        pathType: Prefix
        backend:
          service:
            name: checkout-service
            port:
              number: 8080
      - path: /api/catalog
        pathType: Prefix
        backend:
          service:
            name: catalog-service
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: shop-frontend
            port:
              number: 80
```

### 3. API Gateway Pattern

```yaml
spec:
  rules:
  - host: api.mon-site.com
    http:
      paths:
      - path: /auth
        pathType: Prefix
        backend:
          service:
            name: auth-service
            port:
              number: 8080
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: users-service
            port:
              number: 8080
      - path: /notifications
        pathType: Prefix
        backend:
          service:
            name: notifications-service
            port:
              number: 8080
```

### 4. Environnements Multiples

```yaml
spec:
  rules:
  - host: mon-site.com
    http:
      paths:
      - path: /staging
        pathType: Prefix
        backend:
          service:
            name: app-staging
            port:
              number: 80
      - path: /dev
        pathType: Prefix
        backend:
          service:
            name: app-dev
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-prod
            port:
              number: 80
```

## Priorité et Ordre des Règles

### Règles de Priorité NGINX Ingress

NGINX Ingress évalue les chemins dans cet ordre :

1. **Exact match** en premier
2. **Prefix match** plus long au plus court
3. **Regex match** (ImplementationSpecific)

### Exemple de Priorité

```yaml
paths:
  - path: /api/users/123
    pathType: Exact
    # Priorité 1 : Correspond exactement à /api/users/123

  - path: /api/users
    pathType: Prefix
    # Priorité 2 : Correspond à /api/users/anything

  - path: /api
    pathType: Prefix
    # Priorité 3 : Correspond à /api/anything

  - path: /
    pathType: Prefix
    # Priorité 4 : Catch-all
```

**Requête** : `mon-site.com/api/users/456`
- ❌ Ne correspond pas à `/api/users/123` (Exact)
- ✅ Correspond à `/api/users` (Prefix) → **Règle utilisée**
- (Les suivantes ne sont pas évaluées)

### Bonne Pratique

**Organisez toujours du plus spécifique au plus général** :

```yaml
paths:
  # Plus spécifique
  - path: /api/admin/users
  - path: /api/admin
  - path: /api
  - path: /static
  # Plus général (catch-all)
  - path: /
```

## Dépannage

### Problème 1 : 404 Not Found sur un Sous-Chemin

**Symptôme** : `mon-site.com/` fonctionne mais `mon-site.com/api` renvoie 404.

**Causes possibles** :
1. L'ordre des paths est incorrect (/ avant /api)
2. Le backend ne gère pas le chemin `/api`
3. Besoin de `rewrite-target`

**Solutions** :

1. Vérifiez l'ordre :
   ```bash
   microk8s kubectl get ingress mon-ingress -o yaml | grep -A 5 "paths:"
   ```

2. Testez le backend directement :
   ```bash
   microk8s kubectl port-forward service/api-service 8080:8080
   curl http://localhost:8080/api
   ```

3. Ajoutez `rewrite-target` si nécessaire :
   ```yaml
   annotations:
     nginx.ingress.kubernetes.io/rewrite-target: /
   ```

### Problème 2 : Mauvais Service Routé

**Symptôme** : `/api` montre le contenu du frontend.

**Cause** : Le chemin `/` (catch-all) capture tout car il est placé avant `/api`.

**Solution** : Réorganisez les paths, plus spécifiques en premier :

```yaml
paths:
  - path: /api      # Spécifique en premier
  - path: /         # Général en dernier
```

### Problème 3 : CSS/JS Ne Chargent Pas (SPA)

**Symptôme** : Page HTML charge mais CSS/JS donnent 404.

**Cause** : Les ressources statiques ont des chemins absolus qui ne correspondent pas.

**Solutions** :

1. Utilisez des chemins relatifs dans votre SPA
2. Configurez `<base href="/">` dans index.html
3. Assurez-vous que le serveur web (nginx dans le pod) sert bien les fichiers statiques

### Problème 4 : Regex Ne Fonctionne Pas

**Symptôme** : Votre regex dans `path` ne correspond pas.

**Solution** : Utilisez `pathType: ImplementationSpecific` (pas Prefix ou Exact) et testez votre regex :

```yaml
- path: /api/v[0-9]+/users
  pathType: ImplementationSpecific
```

**Attention** : Les regex sont complexes et fragiles. Préférez Prefix quand possible.

### Problème 5 : Backend Reçoit le Mauvais Chemin

**Symptôme** : Le backend se plaint que le chemin demandé n'existe pas.

**Exemple** :
```
Requête : mon-site.com/api/users
Backend : "Path /api/users not found" (attendait /users)
```

**Solution** : Utilisez `rewrite-target` :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: mon-site.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

## Annotations Utiles pour le Routage par Chemin

### Réécriture d'URL

```yaml
# Simple : enlève le préfixe
nginx.ingress.kubernetes.io/rewrite-target: /

# Avancé : utilise une regex
nginx.ingress.kubernetes.io/rewrite-target: /$2
```

### Configuration Snippet

Pour des besoins très spécifiques :

```yaml
nginx.ingress.kubernetes.io/configuration-snippet: |
  rewrite ^/old-path(.*)$ /new-path$1 permanent;
```

### Proxy Body Size

Pour les uploads sur des endpoints spécifiques :

```yaml
nginx.ingress.kubernetes.io/proxy-body-size: "50m"
```

### CORS sur API

```yaml
nginx.ingress.kubernetes.io/enable-cors: "true"
nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
```

### Authentification sur Admin

```yaml
nginx.ingress.kubernetes.io/auth-type: basic
nginx.ingress.kubernetes.io/auth-secret: admin-auth
nginx.ingress.kubernetes.io/auth-realm: "Administration"
```

## Bonnes Pratiques

### 1. Préfixez Vos APIs

Utilisez toujours un préfixe explicite pour les APIs :

✅ **Bon** :
```
mon-site.com/api/users
mon-site.com/api/products
```

❌ **Évitez** :
```
mon-site.com/users     # Peut entrer en conflit avec une page frontend
mon-site.com/products  # Ambiguïté : API ou page ?
```

### 2. Versionnez Vos APIs

Incluez la version dans le chemin :

```
mon-site.com/api/v1/users
mon-site.com/api/v2/users
```

Cela permet de maintenir plusieurs versions simultanément.

### 3. Utilisez des Chemins Cohérents

Adoptez une convention et respectez-la :

```
/api/resource     (REST : collection)
/api/resource/id  (REST : élément)
```

### 4. Documentez les Routes

Ajoutez des commentaires dans vos manifestes :

```yaml
paths:
  # API publique - pas d'authentification
  - path: /api/public
    pathType: Prefix
    backend:
      service:
        name: public-api
        port:
          number: 8080

  # API admin - requiert authentification
  - path: /api/admin
    pathType: Prefix
    backend:
      service:
        name: admin-api
        port:
          number: 8080
```

### 5. Séparez Frontend et Backend

Gardez les responsabilités claires :

```yaml
paths:
  # Backend
  - path: /api
    pathType: Prefix
    backend:
      service:
        name: backend-service

  # Frontend (catch-all)
  - path: /
    pathType: Prefix
    backend:
      service:
        name: frontend-service
```

### 6. Testez Chaque Chemin

Après modification, testez tous les chemins :

```bash
curl mon-site.com/api/users
curl mon-site.com/api/products
curl mon-site.com/admin
curl mon-site.com/
```

### 7. Logs pour Débugger

Activez les logs détaillés temporairement :

```yaml
annotations:
  nginx.ingress.kubernetes.io/enable-access-log: "true"
```

Puis consultez :
```bash
microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx -f
```

## Template Réutilisable

Voici un template pour vos applications avec routage par chemin :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <APP-NAME>-path-ingress
  namespace: <NAMESPACE>
  annotations:
    # Réécriture si nécessaire (décommentez et ajustez)
    # nginx.ingress.kubernetes.io/rewrite-target: /

    # CORS si API (décommentez si besoin)
    # nginx.ingress.kubernetes.io/enable-cors: "true"

    # Taille uploads
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  rules:
  - host: <YOUR-DOMAIN>.com
    http:
      paths:
      # API paths (du plus spécifique au plus général)
      - path: /api/specific
        pathType: Prefix
        backend:
          service:
            name: <API-SERVICE-NAME>
            port:
              number: <API-PORT>

      # Admin path
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: <ADMIN-SERVICE-NAME>
            port:
              number: <ADMIN-PORT>

      # Static/Assets path (optionnel)
      - path: /static
        pathType: Prefix
        backend:
          service:
            name: <STATIC-SERVICE-NAME>
            port:
              number: <STATIC-PORT>

      # Frontend catch-all (TOUJOURS EN DERNIER)
      - path: /
        pathType: Prefix
        backend:
          service:
            name: <FRONTEND-SERVICE-NAME>
            port:
              number: <FRONTEND-PORT>
```

## Commandes de Référence

```bash
# Créer l'Ingress
microk8s kubectl apply -f path-ingress.yaml

# Voir les chemins configurés
microk8s kubectl get ingress -o yaml | grep -A 10 "paths:"

# Tester un chemin spécifique
curl http://mon-site.com/api/users
curl -H "Host: mon-site.com" http://localhost/api

# Voir la config NGINX générée
POD=$(microk8s kubectl get pods -n ingress -l app.kubernetes.io/name=ingress-nginx -o name)
microk8s kubectl exec -n ingress $POD -- cat /etc/nginx/nginx.conf | grep -A 20 "server_name mon-site.com"

# Logs avec filtrage par path
microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx | grep "/api"

# Débug : voir toutes les locations NGINX
microk8s kubectl exec -n ingress $POD -- nginx -T | grep "location"
```

## Checklist

Avant de terminer cette section, assurez-vous que :

- [ ] Vous comprenez la différence entre routage par hôte et par chemin
- [ ] Vous connaissez les trois PathType (Prefix, Exact, ImplementationSpecific)
- [ ] Vous savez ordonner les chemins (spécifique avant générique)
- [ ] Vous avez testé un Ingress avec plusieurs chemins
- [ ] Vous comprenez quand utiliser `rewrite-target`
- [ ] Vous savez combiner routage par hôte et par chemin
- [ ] Vous pouvez débugger les problèmes de routage avec les logs
- [ ] Vos manifestes sont sauvegardés et documentés

## Points Clés à Retenir

🔑 **Routage par chemin** : différents chemins d'URL sur un même domaine pointent vers différents services

🔑 **PathType Prefix** : le plus courant, correspond si l'URL commence par le chemin

🔑 **Ordre critique** : chemins spécifiques AVANT chemins génériques

🔑 **Catch-all** : `path: /` avec `Prefix` capture tout (placer en dernier)

🔑 **rewrite-target** : modifie le chemin avant de l'envoyer au backend

🔑 **Microservices** : le routage par chemin est idéal pour ce pattern

🔑 **SPA** : nécessitent une configuration spéciale (404 → index.html)

🔑 **Combinaison** : vous pouvez mélanger routage par hôte ET par chemin

🔑 **Logs** : essentiels pour débugger le routage

## Prochaines Étapes

Maintenant que vous maîtrisez le routage par nom d'hôte ET par chemin, vous êtes prêt à :

1. **Sécuriser vos applications avec HTTPS** (chapitre 11) en ajoutant des certificats SSL/TLS
2. **Optimiser les performances** avec la mise en cache et la compression
3. **Ajouter de l'authentification** pour protéger certaines routes
4. **Monitorer votre Ingress** avec Prometheus et Grafana (chapitre 12)

Le routage par chemin est particulièrement puissant pour les architectures microservices et les APIs. Vous avez maintenant toutes les connaissances pour exposer des applications complexes avec une façade unifiée !

---

**📚 Résumé du chapitre** : Le routage par chemin permet d'exposer plusieurs services sur différents chemins d'un même domaine (mon-site.com/api, mon-site.com/admin). Utilisez `pathType: Prefix` dans la plupart des cas. L'ordre est critique : chemins spécifiques avant génériques, catch-all (`/`) en dernier. L'annotation `rewrite-target` modifie le chemin envoyé au backend. Idéal pour les microservices et APIs. Combinez avec le routage par hôte pour une flexibilité maximale.

⏭️ [Middleware et annotations](/10-ingress-et-routage/10-middleware-et-annotations.md)
