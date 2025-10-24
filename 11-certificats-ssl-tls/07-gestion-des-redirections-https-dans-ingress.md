🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.7 Gestion des redirections HTTPS dans Ingress

## Introduction : pourquoi forcer HTTPS ?

Vous avez maintenant des certificats SSL/TLS configurés sur vos applications. Super ! Mais il reste un problème : que se passe-t-il si un utilisateur tape `http://monapp.com` au lieu de `https://monapp.com` dans son navigateur ?

Par défaut, sans configuration spécifique, votre application sera accessible en **HTTP non sécurisé**. Les données circuleront en clair, sans chiffrement, annulant tous les bénéfices de vos certificats SSL/TLS.

**Analogie** : c'est comme avoir une porte blindée (HTTPS) et une porte normale (HTTP) pour entrer dans votre maison. Même si la porte blindée est plus sûre, si les gens continuent d'utiliser la porte normale, elle ne sert à rien !

La solution : **rediriger automatiquement** toutes les requêtes HTTP vers HTTPS. C'est ce que nous allons voir dans cette section.

## Qu'est-ce qu'une redirection HTTPS ?

Une **redirection HTTPS** est un mécanisme qui :

1. Détecte quand un utilisateur accède à votre site en HTTP (port 80)
2. Renvoie un code de statut HTTP **301** (redirection permanente) ou **302** (redirection temporaire)
3. Indique au navigateur de recharger la page en HTTPS (port 443)

### Le processus en détail

```
1. Utilisateur tape : http://monapp.com
2. Navigateur se connecte au port 80 (HTTP)
3. Ingress Controller détecte la connexion HTTP
4. Ingress Controller répond : "301 Moved Permanently - Location: https://monapp.com"
5. Navigateur se reconnecte automatiquement au port 443 (HTTPS)
6. Page chargée de manière sécurisée
```

Tout cela se passe en quelques millisecondes, de manière transparente pour l'utilisateur.

### HTTP 301 vs 302

- **301 (Moved Permanently)** : redirection permanente
  - Le navigateur met en cache cette redirection
  - Les prochaines fois, il ira directement en HTTPS
  - Recommandé pour les redirections HTTPS

- **302 (Found/Moved Temporarily)** : redirection temporaire
  - Le navigateur ne met pas en cache
  - Vérifie à chaque fois
  - Utile pour des redirections temporaires (maintenance, tests)

Pour HTTPS, utilisez toujours **301**.

## Redirection avec NGINX Ingress Controller

NGINX Ingress Controller est le plus utilisé dans Kubernetes et celui inclus dans MicroK8s. Voyons comment configurer les redirections.

### Annotation pour redirection automatique

La méthode la plus simple : une seule annotation !

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-app
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    # Cette annotation force la redirection HTTPS
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: public
  tls:
  - hosts:
    - monapp.com
    secretName: monapp-tls
  rules:
  - host: monapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-app
            port:
              number: 80
```

L'annotation clé :
```yaml
nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

C'est tout ! Toutes les requêtes HTTP seront automatiquement redirigées vers HTTPS.

### Exemple complet avec déploiement

Voici un exemple complet d'application avec redirection HTTPS :

```yaml
---
# Déploiement
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
    spec:
      containers:
      - name: app
        image: nginxdemos/hello
        ports:
        - containerPort: 80

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: secure-app
  namespace: default
spec:
  selector:
    app: secure-app
  ports:
  - port: 80
    targetPort: 80

---
# Ingress avec certificat et redirection HTTPS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-app
  namespace: default
  annotations:
    # Certificat automatique
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    # Redirection HTTPS forcée
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: public
  tls:
  - hosts:
    - secure.exemple.com
    secretName: secure-app-tls
  rules:
  - host: secure.exemple.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: secure-app
            port:
              number: 80
```

Appliquez cette configuration :

```bash
kubectl apply -f secure-app-complete.yaml
```

### Tester la redirection

Testez avec curl :

```bash
# Requête HTTP (noter le -L pour suivre les redirections)
curl -L http://secure.exemple.com

# Voir les en-têtes de redirection (sans -L)
curl -I http://secure.exemple.com
```

Vous devriez voir dans la réponse :

```
HTTP/1.1 301 Moved Permanently
Location: https://secure.exemple.com/
```

Le code 301 indique une redirection permanente vers l'URL HTTPS.

### Vérifier dans un navigateur

1. Ouvrez `http://secure.exemple.com` (sans le 's')
2. Observez la barre d'adresse : elle doit automatiquement changer pour `https://secure.exemple.com`
3. Le cadenas de sécurité doit apparaître

Si vous ouvrez les outils de développement (F12) → onglet Réseau, vous verrez :
- Premier appel : `http://secure.exemple.com` → 301
- Deuxième appel : `https://secure.exemple.com` → 200 OK

## Redirection permanente vs temporaire

Par défaut, `force-ssl-redirect` utilise le code 301 (permanent). Si vous voulez un 302 (temporaire), utilisez :

```yaml
annotations:
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  nginx.ingress.kubernetes.io/ssl-redirect-code: "302"
```

**Recommandation** : utilisez toujours 301 pour HTTPS, sauf si vous avez une raison spécifique d'utiliser 302 (par exemple, pour des tests).

## Configuration avancée : redirections conditionnelles

Parfois, vous voulez rediriger seulement certains chemins ou conditions.

### Redirection pour certains chemins seulement

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: partial-redirect
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    # Configuration de snippet NGINX personnalisée
    nginx.ingress.kubernetes.io/server-snippet: |
      # Rediriger uniquement /admin vers HTTPS
      location /admin {
        if ($scheme = http) {
          return 301 https://$host$request_uri;
        }
      }
spec:
  ingressClassName: public
  tls:
  - hosts:
    - monapp.com
    secretName: monapp-tls
  rules:
  - host: monapp.com
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

Dans cet exemple :
- `/admin` → redirigé vers HTTPS
- Autres chemins → HTTP autorisé (non recommandé en production !)

### Redirection basée sur l'User-Agent

Rediriger seulement certains navigateurs (cas très rare) :

```yaml
annotations:
  nginx.ingress.kubernetes.io/server-snippet: |
    if ($http_user_agent ~* "Chrome") {
      if ($scheme = http) {
        return 301 https://$host$request_uri;
      }
    }
```

**Note** : ces configurations avancées avec `server-snippet` nécessitent une bonne connaissance de NGINX. Pour la plupart des cas, `force-ssl-redirect: "true"` suffit amplement.

## HTTP Strict Transport Security (HSTS)

HSTS est un mécanisme de sécurité complémentaire qui indique au navigateur de **toujours** utiliser HTTPS pour ce site, même si l'utilisateur tape HTTP.

### Comment fonctionne HSTS

1. La première fois, l'utilisateur accède à votre site en HTTPS
2. Votre serveur envoie un en-tête `Strict-Transport-Security`
3. Le navigateur mémorise cette instruction
4. Pour toutes les visites futures (pendant la durée spécifiée), le navigateur forcera automatiquement HTTPS avant même d'envoyer la requête

**Avantage** : élimine complètement le premier appel HTTP, donc plus rapide et plus sécurisé.

### Activer HSTS avec NGINX Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hsts-app
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    # Activer HSTS
    nginx.ingress.kubernetes.io/hsts: "true"
    # Durée de validité HSTS (1 an en secondes)
    nginx.ingress.kubernetes.io/hsts-max-age: "31536000"
    # Inclure les sous-domaines
    nginx.ingress.kubernetes.io/hsts-include-subdomains: "true"
    # Demander l'inclusion dans la liste HSTS preload des navigateurs
    nginx.ingress.kubernetes.io/hsts-preload: "true"
spec:
  ingressClassName: public
  tls:
  - hosts:
    - monapp.com
    secretName: monapp-tls
  rules:
  - host: monapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-app
            port:
              number: 80
```

### Comprendre les paramètres HSTS

#### hsts

```yaml
nginx.ingress.kubernetes.io/hsts: "true"
```

Active HSTS. Le serveur enverra l'en-tête `Strict-Transport-Security`.

#### hsts-max-age

```yaml
nginx.ingress.kubernetes.io/hsts-max-age: "31536000"
```

Durée pendant laquelle le navigateur doit forcer HTTPS (en secondes) :
- `31536000` = 1 an (recommandé)
- `63072000` = 2 ans
- `15768000` = 6 mois

**Important** : commencez avec une valeur courte (ex: `300` = 5 minutes) pour tester, puis augmentez progressivement.

#### hsts-include-subdomains

```yaml
nginx.ingress.kubernetes.io/hsts-include-subdomains: "true"
```

Applique HSTS également à tous les sous-domaines.

**Attention** : n'activez ceci que si **tous** vos sous-domaines supportent HTTPS !

#### hsts-preload

```yaml
nginx.ingress.kubernetes.io/hsts-preload: "true"
```

Demande au navigateur de considérer ce site pour la "preload list", une liste de sites qui sont forcés en HTTPS **avant même la première visite**.

**Prérequis pour preload** :
- HSTS activé
- `max-age` >= 1 an (31536000 secondes)
- `includeSubDomains` activé
- Tous les sous-domaines doivent supporter HTTPS
- Vous devez soumettre votre domaine sur [hstspreload.org](https://hstspreload.org)

**Attention** : le preload est quasiment irréversible (prend des mois à retirer). N'activez qu'en production stable.

### Vérifier HSTS

Testez les en-têtes HSTS :

```bash
curl -I https://monapp.com
```

Vous devriez voir :

```
HTTP/2 200
strict-transport-security: max-age=31536000; includeSubDomains; preload
```

Ou dans le navigateur (F12 → Réseau → sélectionnez une requête → onglet En-têtes) :

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

### Stratégie progressive HSTS

Pour déployer HSTS en toute sécurité :

**Étape 1** : Testez sans HSTS pendant quelques jours
```yaml
nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
# Pas de HSTS encore
```

**Étape 2** : HSTS avec max-age court (5 minutes)
```yaml
nginx.ingress.kubernetes.io/hsts: "true"
nginx.ingress.kubernetes.io/hsts-max-age: "300"
```

**Étape 3** : Augmentez progressivement (1 jour, 1 semaine, 1 mois)
```yaml
nginx.ingress.kubernetes.io/hsts-max-age: "2592000"  # 1 mois
```

**Étape 4** : Valeur finale (1 an) après validation complète
```yaml
nginx.ingress.kubernetes.io/hsts-max-age: "31536000"
nginx.ingress.kubernetes.io/hsts-include-subdomains: "true"
```

**Étape 5** : Preload (optionnel, après 6+ mois en production stable)
```yaml
nginx.ingress.kubernetes.io/hsts-preload: "true"
```

## Ingress sans certificat : redirection impossible

**Important** : vous ne pouvez activer la redirection HTTPS que si vous avez un certificat SSL/TLS configuré !

Si vous essayez d'activer `force-ssl-redirect` sans section `tls` :

```yaml
# ❌ Mauvais - redirection sans certificat
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: broken-redirect
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  rules:
  - host: monapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
  # ❌ Pas de section tls !
```

Résultat : boucle de redirection infinie ou erreur SSL.

**Toujours** combiner redirection HTTPS + certificat :

```yaml
# ✅ Bon
metadata:
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - monapp.com
    secretName: monapp-tls
  rules:
  - host: monapp.com
    # ...
```

## Plusieurs domaines avec redirections

Vous pouvez gérer plusieurs domaines dans le même Ingress :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-domain-secure
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: public
  tls:
  - hosts:
    - app1.com
    - www.app1.com
    secretName: app1-tls
  - hosts:
    - app2.com
    - www.app2.com
    secretName: app2-tls
  rules:
  - host: app1.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  - host: www.app1.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  - host: app2.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
  - host: www.app2.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
```

La redirection HTTPS s'applique à **tous** les domaines configurés.

## Redirection www vers non-www (ou inverse)

Une pratique courante : rediriger `www.exemple.com` vers `exemple.com` (ou l'inverse).

### Redirection www → non-www

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: redirect-www
  namespace: default
  annotations:
    # Rediriger www.exemple.com vers exemple.com
    nginx.ingress.kubernetes.io/permanent-redirect: "https://exemple.com$request_uri"
spec:
  ingressClassName: public
  rules:
  - host: www.exemple.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dummy-service  # Service fictif (non utilisé)
            port:
              number: 80
---
# Ingress principal pour exemple.com
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-app
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: public
  tls:
  - hosts:
    - exemple.com
    - www.exemple.com  # Certificat couvre les deux
    secretName: exemple-tls
  rules:
  - host: exemple.com
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

### Redirection non-www → www

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: redirect-no-www
  namespace: default
  annotations:
    # Rediriger exemple.com vers www.exemple.com
    nginx.ingress.kubernetes.io/permanent-redirect: "https://www.exemple.com$request_uri"
spec:
  ingressClassName: public
  rules:
  - host: exemple.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dummy-service
            port:
              number: 80
---
# Ingress principal pour www.exemple.com
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-app-www
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: public
  tls:
  - hosts:
    - exemple.com
    - www.exemple.com
    secretName: exemple-tls
  rules:
  - host: www.exemple.com
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

**Note** : `$request_uri` préserve le chemin et les paramètres de requête lors de la redirection.

## Redirection avec Traefik Ingress Controller

Si vous utilisez Traefik plutôt que NGINX, la syntaxe est différente.

### Redirection HTTPS avec Traefik

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: traefik-secure
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    # Traefik utilise un middleware pour les redirections
    traefik.ingress.kubernetes.io/router.middlewares: default-redirect-https@kubernetescrd
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - monapp.com
    secretName: monapp-tls
  rules:
  - host: monapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-app
            port:
              number: 80
---
# Middleware pour la redirection HTTPS
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
  namespace: default
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

Traefik utilise des ressources **Middleware** pour configurer les redirections.

## Configuration globale vs par Ingress

### Par Ingress (recommandé)

Ajouter l'annotation à chaque Ingress :

```yaml
annotations:
  nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

**Avantages** :
- ✅ Contrôle fin par application
- ✅ Flexibilité (certaines apps peuvent rester HTTP)
- ✅ Facile à comprendre et maintenir

### Globale (tous les Ingress)

Configurer NGINX Ingress Controller pour forcer HTTPS partout :

```yaml
# ConfigMap NGINX Ingress Controller
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configuration
  namespace: ingress
data:
  ssl-redirect: "true"
  force-ssl-redirect: "true"
```

**Avantages** :
- ✅ Configuration unique
- ✅ Garantit que rien n'est oublié

**Inconvénients** :
- ❌ Moins de flexibilité
- ❌ Peut casser des services qui nécessitent HTTP (webhooks, API publiques sans SSL)

**Recommandation** : utilisez la configuration par Ingress pour plus de contrôle.

## Cas particuliers

### API publiques sans certificat

Certaines API publiques ou webhooks peuvent avoir besoin d'être accessibles en HTTP (rare, mais possible).

Créez un Ingress séparé sans redirection :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: public-api-http
  namespace: default
  # Pas d'annotation force-ssl-redirect !
spec:
  ingressClassName: public
  rules:
  - host: api-public.exemple.com
    http:
      paths:
      - path: /webhook
        pathType: Prefix
        backend:
          service:
            name: webhook-service
            port:
              number: 80
```

**Attention** : utilisez cette approche avec précaution et uniquement quand absolument nécessaire !

### Health checks et métriques

Les health checks Kubernetes se font généralement en HTTP (interne au cluster). Assurez-vous qu'ils ne passent pas par l'Ingress, mais directement au Service.

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080  # Port interne, pas via Ingress
  initialDelaySeconds: 30
  periodSeconds: 10
```

### Backends mixtes HTTP/HTTPS

Si votre backend nécessite HTTPS (peu courant), configurez-le :

```yaml
annotations:
  nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
  nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

Cela indique à NGINX de se connecter au backend en HTTPS au lieu de HTTP.

## Dépannage des redirections

### Boucle de redirection infinie

**Symptôme** : le navigateur affiche "ERR_TOO_MANY_REDIRECTS" ou "Trop de redirections".

**Causes possibles** :
1. Force-ssl-redirect activé sans certificat configuré
2. Problème de détection du protocole (derrière un load balancer)
3. Configuration conflictuelle

**Solutions** :

#### Vérifier le certificat

```bash
kubectl get certificate
kubectl describe certificate <nom>
```

Le certificat doit être `READY: True`.

#### Vérifier la section tls de l'Ingress

```bash
kubectl get ingress <nom> -o yaml
```

La section `tls` doit exister et être correcte.

#### Désactiver temporairement la redirection

Commentez l'annotation pour tester :

```yaml
annotations:
  # nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

### La redirection ne fonctionne pas

**Symptôme** : HTTP reste accessible, pas de redirection vers HTTPS.

**Causes possibles** :
1. Annotation mal orthographiée ou avec mauvaise valeur
2. Ingress Controller ne supporte pas cette annotation
3. Configuration globale qui override

**Solutions** :

#### Vérifier l'annotation

```bash
kubectl get ingress <nom> -o yaml | grep force-ssl
```

Doit afficher :
```yaml
nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

Attention à la casse et aux tirets !

#### Vérifier les logs Ingress Controller

```bash
kubectl logs -n ingress <ingress-controller-pod>
```

Cherchez des erreurs ou avertissements.

#### Tester manuellement

```bash
curl -I http://monapp.com
```

Vous devriez voir `301 Moved Permanently` et `Location: https://...`

### HSTS ne fonctionne pas

**Symptôme** : pas d'en-tête `Strict-Transport-Security` dans la réponse.

**Causes** :
1. HSTS activé mais requête faite en HTTP (HSTS n'est envoyé qu'en HTTPS)
2. Annotation mal configurée
3. Valeur max-age = 0

**Solution** :

```bash
# Tester en HTTPS (pas HTTP !)
curl -I https://monapp.com | grep -i strict
```

### Certificat invalide après redirection

**Symptôme** : la redirection fonctionne, mais le certificat est invalide.

**Causes** :
1. Le certificat ne couvre pas le domaine
2. Certificat expiré ou non encore émis
3. Mauvais Secret référencé

**Solutions** :

```bash
# Vérifier le certificat
kubectl describe certificate <nom>

# Vérifier que le domaine est bien dans dnsNames
kubectl get certificate <nom> -o jsonpath='{.spec.dnsNames}'

# Tester le certificat avec openssl
openssl s_client -connect monapp.com:443 -servername monapp.com
```

## Tests automatisés des redirections

Pour valider vos redirections dans un pipeline CI/CD :

```bash
#!/bin/bash

# Test de redirection HTTP → HTTPS
response=$(curl -s -o /dev/null -w "%{http_code} %{redirect_url}" http://monapp.com)
http_code=$(echo $response | cut -d' ' -f1)
redirect_url=$(echo $response | cut -d' ' -f2)

if [ "$http_code" = "301" ] && [ "$redirect_url" = "https://monapp.com/" ]; then
  echo "✅ Redirection HTTPS OK"
else
  echo "❌ Redirection HTTPS échouée"
  exit 1
fi

# Test HSTS
hsts=$(curl -s -I https://monapp.com | grep -i strict-transport-security)
if [ -n "$hsts" ]; then
  echo "✅ HSTS OK: $hsts"
else
  echo "❌ HSTS manquant"
  exit 1
fi
```

## Bonnes pratiques

### 1. Toujours combiner certificat + redirection

Ne configurez jamais de redirection HTTPS sans certificat valide :

```yaml
# ✅ Bon
annotations:
  cert-manager.io/cluster-issuer: "letsencrypt-prod"
  nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - monapp.com
    secretName: monapp-tls
```

### 2. Utilisez 301 (permanent) pour HTTPS

La redirection permanente (301) est la bonne pratique pour HTTPS.

### 3. Déployez HSTS progressivement

Commencez avec un `max-age` court, validez, puis augmentez.

### 4. Testez avant de déployer en production

Testez toujours vos redirections dans un environnement de staging.

### 5. Surveillez les boucles de redirection

Mettez en place du monitoring pour détecter les boucles de redirection.

### 6. Documentez vos exceptions

Si vous avez des services qui doivent rester en HTTP, documentez pourquoi.

### 7. N'activez preload qu'après validation longue

Le preload HSTS est presque irréversible. Attendez 6+ mois en production stable.

### 8. Un certificat par Ingress

Chaque Ingress doit avoir son propre certificat ou utiliser un certificat wildcard partagé.

## Récapitulatif des annotations NGINX

Voici toutes les annotations vues dans cette section :

```yaml
annotations:
  # Redirection HTTPS basique
  nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

  # Alternative avec contrôle du code HTTP
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  nginx.ingress.kubernetes.io/ssl-redirect-code: "301"

  # HSTS
  nginx.ingress.kubernetes.io/hsts: "true"
  nginx.ingress.kubernetes.io/hsts-max-age: "31536000"
  nginx.ingress.kubernetes.io/hsts-include-subdomains: "true"
  nginx.ingress.kubernetes.io/hsts-preload: "true"

  # Redirection permanente personnalisée
  nginx.ingress.kubernetes.io/permanent-redirect: "https://autre-domaine.com"

  # Backend HTTPS
  nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"

  # Configuration NGINX personnalisée
  nginx.ingress.kubernetes.io/server-snippet: |
    # Configuration NGINX brute
```

## Concepts clés à retenir

- **Redirection HTTPS** : forcer tous les utilisateurs à utiliser HTTPS au lieu de HTTP
- **HTTP 301** : redirection permanente (recommandé pour HTTPS)
- **force-ssl-redirect** : annotation NGINX pour redirection automatique
- **HSTS** : mécanisme qui force le navigateur à toujours utiliser HTTPS
- **max-age** : durée pendant laquelle HSTS est actif dans le navigateur
- **Preload** : liste de sites forcés en HTTPS avant même la première visite
- **Toujours combiner** : certificat + redirection = sécurité complète

## Prochaines étapes

Dans la prochaine section, nous allons explorer le renouvellement automatique des certificats en profondeur, comprendre comment monitorer vos certificats, et mettre en place des alertes pour être prévenu en cas de problème.

La redirection HTTPS est la dernière pièce du puzzle pour une configuration SSL/TLS complète et sécurisée. Vos utilisateurs sont maintenant automatiquement protégés, qu'ils tapent HTTP ou HTTPS !

---


⏭️ [Renouvellement automatique](/11-certificats-ssl-tls/08-renouvellement-automatique.md)
