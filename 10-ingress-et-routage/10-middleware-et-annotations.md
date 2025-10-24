🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.10 Middleware et annotations

## Introduction

Vous savez maintenant créer des règles Ingress pour router le trafic vers vos applications. Mais Ingress peut faire bien plus que du simple routage ! Grâce aux **annotations** et aux **middlewares**, vous pouvez ajouter des fonctionnalités puissantes sans modifier vos applications.

Les annotations permettent de personnaliser le comportement de NGINX Ingress Controller : redirections, authentification, limitation de débit, gestion des en-têtes HTTP, compression, et bien plus encore. C'est comme ajouter des **super-pouvoirs** à votre Ingress.

Dans ce chapitre, nous allons explorer les annotations les plus utiles et apprendre à les utiliser pour améliorer, sécuriser et optimiser vos applications.

## Qu'est-ce qu'une Annotation ?

### Définition

Une **annotation** est une métadonnée que vous ajoutez à une ressource Kubernetes pour modifier son comportement. Dans le contexte d'Ingress, les annotations configurent NGINX pour chaque règle de routage.

**Analogie** : Les annotations sont comme des **post-it** collés sur vos règles Ingress, donnant des instructions spéciales à NGINX.

### Syntaxe de Base

Les annotations se placent dans la section `metadata` du manifeste :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  # ... règles de routage ...
```

**Structure** :
- `nginx.ingress.kubernetes.io/` : préfixe pour NGINX Ingress Controller
- `annotation-name` : nom de l'annotation
- `"valeur"` : valeur (toujours entre guillemets pour les booléens et nombres)

## Qu'est-ce qu'un Middleware ?

### Concept

Un **middleware** est un composant qui se place **entre** le client et votre application pour traiter ou modifier les requêtes et réponses.

**Flux sans middleware** :
```
Client → Ingress → Service → Application
```

**Flux avec middleware** :
```
Client → Ingress → [Middleware] → Service → Application
                   ↓
            Authentification
            Rate limiting
            Headers
            Redirections
            etc.
```

### Dans NGINX Ingress

NGINX Ingress n'a pas de système de "middleware" séparé comme Traefik. À la place, il utilise des **annotations** qui configurent NGINX pour obtenir le même résultat.

**Exemple** : L'annotation `nginx.ingress.kubernetes.io/auth-type: basic` active l'authentification HTTP Basic, qui agit comme un middleware d'authentification.

## Catégories d'Annotations

Les annotations NGINX Ingress se divisent en plusieurs catégories :

### 1. Redirections et Réécritures
Modifier les URLs avant le traitement

### 2. Authentification et Autorisation
Protéger l'accès aux applications

### 3. Limitations et Sécurité
Rate limiting, listes blanches/noires

### 4. Gestion des En-têtes HTTP
Ajouter, modifier, supprimer des headers

### 5. SSL/TLS et Certificats
Configuration HTTPS et certificats

### 6. Performance et Optimisation
Cache, compression, timeouts

### 7. CORS et APIs
Configuration pour applications web modernes

### 8. Backends et Load Balancing
Affinité de session, protocoles

## 1. Redirections et Réécritures

### Redirection HTTP vers HTTPS

Force toutes les requêtes HTTP à passer en HTTPS :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

**Résultat** :
```
http://mon-site.com  →  https://mon-site.com (redirection 301)
```

**Quand utiliser** : TOUJOURS en production pour la sécurité.

### Redirection Permanente

Rediriger un domaine vers un autre :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/permanent-redirect: https://nouveau-site.com
```

**Résultat** :
```
ancien-site.com  →  nouveau-site.com (redirection 301)
```

**Cas d'usage** : Migration de domaine, fusion de sites.

### Redirection Temporaire

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/temporal-redirect: https://site-maintenance.com
```

**Résultat** : Redirection 302 (temporaire).

**Cas d'usage** : Maintenance, événements temporaires.

### Réécriture d'URL Simple

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
```

Enlève le préfixe du chemin avant d'envoyer au backend.

**Exemple** :
```
Requête : mon-site.com/api/users
Backend reçoit : /users  (le /api est enlevé)
```

### Réécriture d'URL Avancée

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

Utilise des groupes de capture regex pour réorganiser l'URL.

### Snippet de Réécriture Personnalisé

Pour des besoins très spécifiques :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/configuration-snippet: |
      rewrite ^/old-api/(.*)$ /v2/api/$1 permanent;
```

**Attention** : Les snippets donnent un contrôle total mais peuvent casser NGINX si mal écrits.

## 2. Authentification et Autorisation

### Authentification HTTP Basic

Protège votre application avec login/mot de passe :

**Étape 1** : Créer le Secret avec identifiants

```bash
# Créer un fichier d'utilisateurs
htpasswd -c auth admin
# Entrez le mot de passe quand demandé

# Créer le Secret Kubernetes
microk8s kubectl create secret generic basic-auth --from-file=auth
```

**Étape 2** : Référencer le Secret dans l'Ingress

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required - Admin Area"
```

**Résultat** : Une popup d'authentification apparaît avant d'accéder à l'application.

**Cas d'usage** : Admin panels, environnements de staging, outils internes.

### Authentification via URL Externe

Délègue l'authentification à un service externe :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/auth-url: "https://auth.example.com/verify"
    nginx.ingress.kubernetes.io/auth-signin: "https://auth.example.com/login"
```

**Fonctionnement** :
1. NGINX envoie la requête à l'URL d'authentification
2. Si le service répond 200 : accès autorisé
3. Si le service répond 401/403 : redirection vers la page de login

**Cas d'usage** : SSO (Single Sign-On), OAuth2, authentification centralisée.

### Liste Blanche d'IPs

Autorise seulement certaines adresses IP :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/whitelist-source-range: "192.168.1.0/24,10.0.0.0/8"
```

**Résultat** : Seules les IPs dans ces plages peuvent accéder.

**Cas d'usage** :
- Admin accessible seulement depuis le bureau
- API interne accessible seulement depuis le réseau d'entreprise
- Environnement de staging restreint

## 3. Limitations et Sécurité

### Rate Limiting (Limitation de Débit)

Limite le nombre de requêtes par seconde depuis une même IP :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "10"
```

**Résultat** : Maximum 10 requêtes par seconde par IP. Au-delà, HTTP 503.

**Cas d'usage** :
- Protection contre les abus
- APIs publiques
- Prévention des attaques DDoS simples

### Limitation de Connexions

Limite le nombre de connexions simultanées par IP :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-connections: "5"
```

**Résultat** : Maximum 5 connexions en parallèle par IP.

**Cas d'usage** : Empêcher qu'une IP monopolise les ressources.

### Limitation de la Taille des Requêtes

Contrôle la taille maximale du corps de la requête :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
```

**Résultat** : Uploads jusqu'à 100 MB autorisés.

**Cas d'usage** :
- `"10m"` : Formulaires simples (défaut : 1m)
- `"100m"` : Upload de photos
- `"1g"` : Upload de vidéos
- `"0"` : Illimité (dangereux !)

### Limitation de la Bande Passante

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rate: "100k"
    nginx.ingress.kubernetes.io/limit-rate-after: "10m"
```

**Résultat** : Après 10 MB transférés, limite à 100 KB/s.

**Cas d'usage** : Téléchargements de gros fichiers sans saturer la bande passante.

## 4. Gestion des En-têtes HTTP

### Ajouter des En-têtes Personnalisés

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Custom-Header: valeur";
      more_set_headers "X-Request-ID: $request_id";
```

**Résultat** : Ces headers sont ajoutés à toutes les réponses.

**Cas d'usage** :
- En-têtes de sécurité
- Traçabilité (X-Request-ID)
- Informations custom pour le frontend

### En-têtes de Sécurité

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
      more_set_headers "Referrer-Policy: strict-origin-when-cross-origin";
```

**Protection contre** :
- Clickjacking (X-Frame-Options)
- MIME sniffing attacks (X-Content-Type-Options)
- XSS attacks (X-XSS-Protection)
- Fuites d'informations (Referrer-Policy)

**Recommandé** : Toujours activer en production.

### Headers HSTS (HTTP Strict Transport Security)

Force les navigateurs à toujours utiliser HTTPS :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/hsts: "true"
    nginx.ingress.kubernetes.io/hsts-max-age: "31536000"
    nginx.ingress.kubernetes.io/hsts-include-subdomains: "true"
    nginx.ingress.kubernetes.io/hsts-preload: "true"
```

**Résultat** : Le navigateur se souviendra pendant 1 an de toujours utiliser HTTPS.

**Attention** : N'activez que si vous êtes certain d'avoir HTTPS partout !

### Transmettre l'IP Réelle du Client

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/use-forwarded-headers: "true"
    nginx.ingress.kubernetes.io/compute-full-forwarded-for: "true"
```

**Résultat** : Votre application voit la vraie IP du client, pas celle du proxy.

**Cas d'usage** : Logs, géolocalisation, analytics.

## 5. SSL/TLS et Certificats

### Backend HTTPS

Si votre backend utilise déjà HTTPS en interne :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
```

**Résultat** : NGINX se connecte au backend via HTTPS au lieu de HTTP.

### Protocoles et Chiffrements SSL

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.2 TLSv1.3"
    nginx.ingress.kubernetes.io/ssl-ciphers: "HIGH:!aNULL:!MD5"
    nginx.ingress.kubernetes.io/ssl-prefer-server-ciphers: "true"
```

**Résultat** : Sécurité SSL renforcée, désactive les vieux protocoles.

### Redirection Spéciale pour HTTPS

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-redirect-code: "301"
```

**Code 301** : Redirection permanente (SEO-friendly)
**Code 302** : Redirection temporaire

## 6. Performance et Optimisation

### Timeouts

Ajuste les délais d'attente pour les backends lents :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
```

**Valeurs en secondes**.

**Cas d'usage** :
- `60` : Applications web normales (défaut)
- `300` : Traitement long (génération de rapports, exports)
- `3600` : Traitement très long (analytics, ML)

### Buffers

Optimise la mise en mémoire tampon :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-buffer-size: "8k"
    nginx.ingress.kubernetes.io/proxy-buffers-number: "4"
    nginx.ingress.kubernetes.io/client-body-buffer-size: "8k"
```

**Résultat** : Meilleures performances pour les grosses réponses.

**Augmentez** si vous avez des réponses volumineuses (APIs avec beaucoup de données).

### Compression Gzip

Active la compression des réponses :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/enable-gzip: "true"
    nginx.ingress.kubernetes.io/gzip-level: "5"
    nginx.ingress.kubernetes.io/gzip-types: "text/html text/css text/javascript application/json"
```

**Résultat** : Réponses compressées = transfert plus rapide.

**Gzip level** :
- `1` : Compression rapide, faible
- `5` : Bon compromis (recommandé)
- `9` : Compression maximale, lente

**Cas d'usage** : Sites web, APIs JSON (pas pour images/vidéos déjà compressées).

### Upstream Keep-Alive

Réutilise les connexions aux backends :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/upstream-keepalive-timeout: "60"
    nginx.ingress.kubernetes.io/upstream-keepalive-requests: "100"
```

**Résultat** : Moins de connexions TCP = meilleure performance.

## 7. CORS et APIs

### Activation CORS Complète

Pour les APIs publiques appelées depuis des navigateurs :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
    nginx.ingress.kubernetes.io/cors-max-age: "3600"
```

**Explication des options** :
- `allow-origin` : Domaines autorisés (`*` = tous, ou liste spécifique)
- `allow-methods` : Méthodes HTTP autorisées
- `allow-headers` : En-têtes autorisés
- `allow-credentials` : Autoriser les cookies
- `max-age` : Durée de cache de la réponse preflight (secondes)

### CORS Restreint (Plus Sécurisé)

Pour une API privée :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://mon-frontend.com,https://app.mon-site.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
```

**Plus sécurisé** : Seuls les domaines listés peuvent faire des requêtes.

## 8. Backends et Load Balancing

### Affinité de Session (Sticky Sessions)

Dirige toujours un utilisateur vers le même pod :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "route"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "3600"
    nginx.ingress.kubernetes.io/session-cookie-path: "/"
```

**Résultat** : Un cookie identifie le pod, l'utilisateur y reste "collé".

**Cas d'usage** :
- Applications avec état en mémoire
- WebSockets
- Sessions non distribuées

**Alternative moderne** : Utilisez un stockage de session externe (Redis, Memcached).

### Load Balancing Algorithm

Change l'algorithme de répartition de charge :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/load-balance: "round_robin"
```

**Options** :
- `round_robin` : Tour par tour (défaut)
- `least_conn` : Vers le pod avec le moins de connexions
- `ip_hash` : Basé sur l'IP du client

### Upstream Hashing

Hash les requêtes basé sur une valeur :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/upstream-hash-by: "$request_uri"
```

**Résultat** : Même URL → toujours le même pod.

**Cas d'usage** : Cache local dans les pods.

## 9. WebSockets

### Support WebSocket

Pour les applications temps réel :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/websocket-services: "chat-service"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
```

**Résultat** : Connexions WebSocket maintenues.

**Services** : Nommez les services qui utilisent WebSocket.

**Cas d'usage** : Chat, notifications temps réel, gaming, collaboration.

## 10. Logging et Monitoring

### Désactiver les Logs d'Accès

Pour réduire le bruit :

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/enable-access-log: "false"
```

**Cas d'usage** : Healthchecks, endpoints de monitoring.

### Format de Log Personnalisé

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/log-format-upstream: '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" $request_length $request_time'
```

**Résultat** : Logs avec format custom (plus d'infos, moins d'infos, format JSON, etc.).

## Exemples de Configurations Complètes

### 1. API REST Publique

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: public-api-ingress
  namespace: default
  annotations:
    # HTTPS obligatoire
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

    # CORS pour tous
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"

    # Rate limiting anti-abus
    nginx.ingress.kubernetes.io/limit-rps: "100"

    # Taille requêtes modérée
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"

    # Compression
    nginx.ingress.kubernetes.io/enable-gzip: "true"
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

### 2. Interface Admin Sécurisée

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: admin-ingress
  namespace: default
  annotations:
    # HTTPS obligatoire
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Authentification basique
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: admin-auth
    nginx.ingress.kubernetes.io/auth-realm: "Administration Access"

    # IP whitelist (bureau uniquement)
    nginx.ingress.kubernetes.io/whitelist-source-range: "203.0.113.0/24"

    # En-têtes de sécurité
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
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

### 3. Application de Upload de Fichiers

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: upload-ingress
  namespace: default
  annotations:
    # HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Gros fichiers autorisés
    nginx.ingress.kubernetes.io/proxy-body-size: "500m"

    # Timeouts augmentés
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"

    # Limite de connexions
    nginx.ingress.kubernetes.io/limit-connections: "3"
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

### 4. Application WebSocket (Chat)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: chat-ingress
  namespace: default
  annotations:
    # HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Support WebSocket
    nginx.ingress.kubernetes.io/websocket-services: "chat-service"

    # Timeouts longs pour connexions persistantes
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"

    # Sticky sessions
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "chat-route"
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

### 5. API avec Traitement Long

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: processing-api-ingress
  namespace: default
  annotations:
    # HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Timeouts très longs (génération de rapports, ML, etc.)
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "1800"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "1800"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "1800"

    # Pas de rate limiting (traitement peut être long)
    nginx.ingress.kubernetes.io/limit-rps: "5"

    # Uploads moyens
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
spec:
  ingressClassName: nginx
  rules:
  - host: processing.mon-site.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: processing-service
            port:
              number: 8080
```

## Combiner Plusieurs Annotations

Vous pouvez (et devez) combiner plusieurs annotations pour créer une configuration robuste :

```yaml
metadata:
  annotations:
    # Sécurité
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/whitelist-source-range: "192.168.1.0/24"

    # Performance
    nginx.ingress.kubernetes.io/enable-gzip: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"

    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://mon-frontend.com"

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "50"

    # Timeouts
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
```

**Testez toujours** après avoir ajouté plusieurs annotations !

## Bonnes Pratiques

### 1. Commencez Simple

Ne surchargez pas d'annotations dès le début. Ajoutez-les au fur et à mesure des besoins.

✅ **Bon parcours** :
1. Créer l'Ingress basique
2. Tester que ça fonctionne
3. Ajouter SSL redirect
4. Ajouter rate limiting
5. Ajouter CORS si nécessaire
6. etc.

### 2. Documentez Vos Choix

Ajoutez des commentaires pour expliquer pourquoi vous avez ajouté telle annotation :

```yaml
metadata:
  annotations:
    # Timeout augmenté car la génération de rapports prend jusqu'à 5 minutes
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"

    # Rate limiting pour éviter les abus sur l'API publique
    nginx.ingress.kubernetes.io/limit-rps: "100"
```

### 3. Testez Chaque Modification

Après avoir ajouté/modifié une annotation :
1. Appliquez le manifeste
2. Attendez quelques secondes (NGINX recharge)
3. Testez l'application
4. Consultez les logs si problème

### 4. Validez les Valeurs

Les annotations ont des formats spécifiques :
- Booléens : `"true"` ou `"false"` (avec guillemets !)
- Nombres : `"100"` (avec guillemets)
- Listes : `"val1, val2, val3"` ou `"val1,val2,val3"`

❌ **Incorrect** :
```yaml
nginx.ingress.kubernetes.io/ssl-redirect: true  # Pas de guillemets
```

✅ **Correct** :
```yaml
nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

### 5. Environnements Séparés

Utilisez des annotations différentes selon l'environnement :

**Développement** :
```yaml
# Pas de rate limiting
# CORS ouvert
# Logs détaillés
```

**Staging** :
```yaml
# Rate limiting modéré
# CORS restreint
# Auth basique pour tests
```

**Production** :
```yaml
# Rate limiting strict
# CORS très restreint
# Sécurité maximale
# Monitoring activé
```

### 6. Référez-vous à la Documentation

La liste complète des annotations : https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/

Consultez toujours la doc officielle pour :
- Syntaxe exacte
- Valeurs par défaut
- Exemples d'utilisation
- Limitations connues

## Dépannage

### Problème 1 : Annotation Ignorée

**Symptôme** : Vous ajoutez une annotation mais rien ne change.

**Causes possibles** :
1. Faute de frappe dans le nom de l'annotation
2. Format de valeur incorrect
3. NGINX n'a pas rechargé

**Solutions** :

1. Vérifiez l'orthographe :
   ```bash
   microk8s kubectl get ingress mon-ingress -o yaml | grep annotations -A 10
   ```

2. Consultez les logs de l'Ingress Controller :
   ```bash
   microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx --tail=50
   ```

3. Forcez un rechargement :
   ```bash
   microk8s kubectl delete pod -n ingress -l app.kubernetes.io/name=ingress-nginx
   ```

### Problème 2 : Erreur 500 après Ajout d'Annotation

**Symptôme** : L'application fonctionne puis renvoie 500 après ajout d'une annotation.

**Cause** : Configuration NGINX invalide (souvent avec `configuration-snippet`).

**Solutions** :

1. Consultez les logs NGINX :
   ```bash
   microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx | grep error
   ```

2. Supprimez l'annotation problématique :
   ```bash
   microk8s kubectl edit ingress mon-ingress
   # Supprimez la dernière annotation ajoutée
   ```

3. Testez la syntaxe NGINX séparément avant de l'ajouter

### Problème 3 : Rate Limiting Trop Strict

**Symptôme** : Utilisateurs légitimes reçoivent des erreurs 503.

**Cause** : Valeur de `limit-rps` trop faible.

**Solutions** :

1. Augmentez progressivement :
   ```yaml
   nginx.ingress.kubernetes.io/limit-rps: "50"  # Au lieu de 10
   ```

2. Ou supprimez temporairement pour tester :
   ```bash
   microk8s kubectl edit ingress mon-ingress
   # Commentez ou supprimez limit-rps
   ```

3. Analysez le trafic réel avant de définir la limite

### Problème 4 : CORS Ne Fonctionne Pas

**Symptôme** : Erreurs CORS dans la console navigateur malgré l'activation.

**Causes possibles** :
1. `allow-origin` incorrect
2. `allow-methods` ne contient pas la méthode utilisée
3. `allow-headers` ne contient pas les headers envoyés

**Solutions** :

1. Vérifiez les headers CORS retournés :
   ```bash
   curl -I -H "Origin: https://mon-frontend.com" https://api.mon-site.com
   ```

2. Activez CORS pour tous pendant le debug :
   ```yaml
   nginx.ingress.kubernetes.io/cors-allow-origin: "*"
   ```

3. Vérifiez la console navigateur pour voir quel header/méthode est rejeté

### Problème 5 : Authentification Basique Ne Fonctionne Pas

**Symptôme** : Pas de popup d'authentification.

**Causes** :
1. Le Secret n'existe pas
2. Le Secret est dans le mauvais namespace
3. Le fichier `auth` est mal formé

**Solutions** :

1. Vérifiez le Secret :
   ```bash
   microk8s kubectl get secret basic-auth
   microk8s kubectl get secret basic-auth -o yaml
   ```

2. Recréez le Secret si nécessaire :
   ```bash
   htpasswd -c auth admin
   microk8s kubectl delete secret basic-auth
   microk8s kubectl create secret generic basic-auth --from-file=auth
   ```

3. Vérifiez le namespace :
   ```yaml
   # Le Secret et l'Ingress doivent être dans le même namespace
   ```

## Annotations par Cas d'Usage

### Site Web Public

```yaml
annotations:
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  nginx.ingress.kubernetes.io/hsts: "true"
  nginx.ingress.kubernetes.io/hsts-max-age: "31536000"
  nginx.ingress.kubernetes.io/enable-gzip: "true"
  nginx.ingress.kubernetes.io/proxy-body-size: "10m"
```

### API Publique

```yaml
annotations:
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  nginx.ingress.kubernetes.io/enable-cors: "true"
  nginx.ingress.kubernetes.io/cors-allow-origin: "*"
  nginx.ingress.kubernetes.io/limit-rps: "100"
  nginx.ingress.kubernetes.io/proxy-body-size: "10m"
```

### Interface Admin

```yaml
annotations:
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  nginx.ingress.kubernetes.io/auth-type: basic
  nginx.ingress.kubernetes.io/auth-secret: admin-auth
  nginx.ingress.kubernetes.io/whitelist-source-range: "192.168.1.0/24"
```

### Service de Upload

```yaml
annotations:
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  nginx.ingress.kubernetes.io/proxy-body-size: "1g"
  nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
  nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
  nginx.ingress.kubernetes.io/limit-connections: "5"
```

### Application WebSocket

```yaml
annotations:
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  nginx.ingress.kubernetes.io/websocket-services: "mon-service"
  nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
  nginx.ingress.kubernetes.io/affinity: "cookie"
```

## Commandes de Référence

```bash
# Voir les annotations d'un Ingress
microk8s kubectl get ingress mon-ingress -o yaml | grep annotations -A 20

# Éditer les annotations
microk8s kubectl edit ingress mon-ingress

# Tester une annotation spécifique
curl -I https://mon-site.com

# Voir la config NGINX générée
POD=$(microk8s kubectl get pods -n ingress -l app.kubernetes.io/name=ingress-nginx -o name)
microk8s kubectl exec -n ingress $POD -- cat /etc/nginx/nginx.conf | grep -A 10 "server_name mon-site.com"

# Logs de l'Ingress Controller
microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx -f

# Recharger NGINX manuellement
microk8s kubectl delete pod -n ingress -l app.kubernetes.io/name=ingress-nginx
```

## Checklist

Avant de passer au chapitre suivant, assurez-vous que :

- [ ] Vous comprenez ce qu'est une annotation Ingress
- [ ] Vous connaissez les principales catégories d'annotations
- [ ] Vous avez testé au moins 3 annotations différentes
- [ ] Vous savez combiner plusieurs annotations
- [ ] Vous connaissez les annotations de sécurité essentielles
- [ ] Vous savez débugger une annotation qui ne fonctionne pas
- [ ] Vous documentez vos annotations avec des commentaires
- [ ] Vous avez consulté la documentation officielle

## Points Clés à Retenir

🔑 **Annotations** : métadonnées qui configurent le comportement de NGINX

🔑 **Préfixe** : toutes les annotations commencent par `nginx.ingress.kubernetes.io/`

🔑 **Format** : valeurs toujours entre guillemets (`"true"`, `"100"`, `"valeur"`)

🔑 **Catégories** : redirections, auth, rate limiting, headers, SSL, performance, CORS, load balancing

🔑 **Middleware** : dans NGINX Ingress, les annotations jouent le rôle de middlewares

🔑 **Sécurité** : `ssl-redirect`, `auth-type`, `whitelist-source-range`, headers de sécurité

🔑 **Performance** : `enable-gzip`, timeouts, buffers, keep-alive

🔑 **APIs** : `enable-cors`, `limit-rps`, `proxy-body-size`

🔑 **Testez** : toujours tester après modification, consulter les logs

🔑 **Documentation** : consultez la doc officielle pour syntaxe et exemples

## Prochaines Étapes

Maintenant que vous maîtrisez les annotations et middlewares, vous êtes prêt à :

1. **Sécuriser vos applications avec HTTPS** (chapitre 11) en ajoutant des certificats SSL/TLS automatiques
2. **Monitorer votre Ingress** (chapitre 12) avec Prometheus et Grafana
3. **Optimiser les performances** avec des configurations avancées
4. **Implémenter des stratégies de déploiement** (blue-green, canary)

Les annotations sont extrêmement puissantes et permettent d'ajouter des fonctionnalités avancées sans toucher au code de vos applications. Vous avez maintenant toutes les connaissances pour créer des Ingress robustes, sécurisés et performants !

---

**📚 Résumé du chapitre** : Les annotations NGINX Ingress permettent d'ajouter des fonctionnalités puissantes : redirections HTTPS, authentification, rate limiting, CORS, compression, timeouts, etc. Format : `nginx.ingress.kubernetes.io/nom-annotation: "valeur"`. Combinez plusieurs annotations pour créer des configurations robustes. Testez chaque modification et consultez les logs. La documentation officielle liste toutes les annotations disponibles. Les annotations agissent comme des middlewares pour personnaliser le comportement de NGINX.

⏭️ [Exemples pratiques d'Ingress](/10-ingress-et-routage/11-exemples-pratiques-dingress.md)
