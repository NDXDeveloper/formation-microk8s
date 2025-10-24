🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.3 Configuration NGINX Ingress Controller

## Introduction

Maintenant que NGINX Ingress Controller est installé, il est temps d'apprendre à le configurer pour répondre à vos besoins spécifiques. La bonne nouvelle ? NGINX Ingress Controller offre une flexibilité remarquable tout en restant simple à configurer.

Dans ce chapitre, nous allons explorer les différentes méthodes de configuration, des réglages globaux aux paramètres spécifiques à chaque application. Vous apprendrez à personnaliser le comportement de votre Ingress pour obtenir exactement ce que vous souhaitez.

## Les Trois Niveaux de Configuration

NGINX Ingress Controller peut être configuré à trois niveaux différents :

### 1. Configuration Globale (ConfigMap)
Affecte **tous les Ingress** du cluster.
📍 Exemple : timeout global, taille maximale des requêtes

### 2. Configuration par Ingress (Annotations)
Affecte **un Ingress spécifique** et ses règles.
📍 Exemple : redirection HTTPS pour une application particulière

### 3. Configuration par Service (Annotations de Service)
Affecte le comportement pour **un service backend spécifique**.
📍 Exemple : timeouts personnalisés pour une API lente

Cette hiérarchie permet une grande flexibilité : vous définissez des valeurs par défaut globales, puis affinez au besoin pour des cas particuliers.

## 1. Configuration Globale via ConfigMap

### Qu'est-ce qu'un ConfigMap ?

Un **ConfigMap** est une ressource Kubernetes qui stocke des données de configuration sous forme de paires clé-valeur. Pour NGINX Ingress Controller, le ConfigMap principal s'appelle `nginx-load-balancer-microk8s-conf`.

### Lister les ConfigMaps de l'Ingress

```bash
microk8s kubectl get configmap -n ingress
```

Sortie :
```
NAME                                DATA   AGE
nginx-load-balancer-microk8s-conf   0      1h
nginx-ingress-tcp-microk8s-conf     0      1h
nginx-ingress-udp-microk8s-conf     0      1h
```

Le ConfigMap principal (`nginx-load-balancer-microk8s-conf`) contient la configuration globale de NGINX.

### Voir le Contenu du ConfigMap

```bash
microk8s kubectl get configmap nginx-load-balancer-microk8s-conf -n ingress -o yaml
```

Par défaut, il peut être vide (DATA: 0) :
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-load-balancer-microk8s-conf
  namespace: ingress
data: {}
```

### Éditer le ConfigMap

Pour modifier la configuration globale :

```bash
microk8s kubectl edit configmap nginx-load-balancer-microk8s-conf -n ingress
```

Cela ouvrira l'éditeur par défaut (vi/vim ou nano selon votre configuration). Ajoutez vos paramètres sous la section `data:`.

### Options de Configuration Globale Courantes

Voici les options les plus utiles à connaître :

#### Timeouts

Contrôlent combien de temps NGINX attend avant d'abandonner une requête :

```yaml
data:
  proxy-connect-timeout: "60"      # Timeout de connexion au backend (en secondes)
  proxy-read-timeout: "60"         # Timeout de lecture de la réponse (en secondes)
  proxy-send-timeout: "60"         # Timeout d'envoi de la requête (en secondes)
  upstream-keepalive-timeout: "60" # Timeout des connexions persistantes
```

**Cas d'usage** : Si vous avez des APIs lentes ou du traitement long, augmentez ces valeurs.

#### Taille des Requêtes

Limite la taille des requêtes envoyées par les clients :

```yaml
data:
  proxy-body-size: "50m"           # Taille max de la requête (body)
  client-max-body-size: "50m"      # Taille max du corps de la requête
```

**Cas d'usage** : Pour permettre l'upload de fichiers volumineux (images, vidéos, documents).

#### Buffers

Contrôle la mise en mémoire tampon des requêtes et réponses :

```yaml
data:
  proxy-buffer-size: "8k"
  proxy-buffers-number: "4"
  client-body-buffer-size: "8k"
```

**Cas d'usage** : Pour optimiser les performances avec des réponses volumineuses.

#### En-têtes HTTP

Gestion des en-têtes forwarded :

```yaml
data:
  use-forwarded-headers: "true"    # Utiliser X-Forwarded-* headers
  compute-full-forwarded-for: "true"
  use-proxy-protocol: "false"      # Protocole PROXY (pour load balancers)
```

**Cas d'usage** : Quand vous êtes derrière un autre proxy/load balancer.

#### Logs

Configuration de la journalisation :

```yaml
data:
  error-log-level: "notice"        # Niveau de log : debug, info, notice, warn, error
  access-log-off: "false"          # Désactiver les logs d'accès (true/false)
  log-format-upstream: '$remote_addr - $remote_user [$time_local] "$request" $status'
```

**Cas d'usage** : Pour le débogage (debug) ou pour réduire l'espace disque (access-log-off).

#### SSL/TLS

Configuration SSL globale :

```yaml
data:
  ssl-protocols: "TLSv1.2 TLSv1.3"
  ssl-ciphers: "HIGH:!aNULL:!MD5"
  ssl-prefer-server-ciphers: "true"
  hsts: "true"                     # HTTP Strict Transport Security
  hsts-max-age: "31536000"         # Durée HSTS (1 an)
  hsts-include-subdomains: "true"
```

**Cas d'usage** : Renforcer la sécurité SSL/TLS de toutes vos applications.

### Exemple de Configuration Complète

Voici un exemple de ConfigMap bien configuré pour un usage général :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-load-balancer-microk8s-conf
  namespace: ingress
data:
  # Timeouts augmentés pour les APIs lentes
  proxy-connect-timeout: "120"
  proxy-read-timeout: "120"
  proxy-send-timeout: "120"

  # Upload de fichiers jusqu'à 100MB
  proxy-body-size: "100m"
  client-max-body-size: "100m"

  # Support des proxies en amont
  use-forwarded-headers: "true"

  # Sécurité SSL renforcée
  ssl-protocols: "TLSv1.2 TLSv1.3"
  hsts: "true"
  hsts-max-age: "31536000"

  # Logs plus détaillés
  error-log-level: "warn"
```

### Appliquer les Modifications

Après avoir édité le ConfigMap, les modifications sont automatiquement détectées par NGINX Ingress Controller. Il recharge sa configuration sous quelques secondes **sans interruption de service**.

Pour vérifier que les changements ont été appliqués, consultez les logs :

```bash
microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx --tail=50
```

Vous devriez voir un message indiquant le rechargement :
```
Backend successfully reloaded
Configuration changes detected, backend reload required
```

## 2. Configuration par Ingress via Annotations

Les **annotations** permettent de configurer des paramètres spécifiques pour un Ingress donné, sans affecter les autres.

### Qu'est-ce qu'une Annotation ?

Une annotation est une métadonnée que vous ajoutez à une ressource Kubernetes. Pour les Ingress, les annotations commencent généralement par `nginx.ingress.kubernetes.io/`.

### Anatomie d'une Annotation

Dans un manifeste Ingress :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  # ... règles de routage ...
```

Les annotations vont dans la section `metadata` du manifeste.

### Annotations Essentielles

#### Redirection HTTPS

Force la redirection HTTP vers HTTPS :

```yaml
annotations:
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

**Cas d'usage** : Toujours utiliser HTTPS pour la sécurité.

#### Réécriture d'URL (Rewrite)

Modifie le chemin de la requête avant de l'envoyer au backend :

```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /
```

**Exemple** :
- Requête : `mon-site.com/app/login`
- Avec rewrite : le backend reçoit `/login`
- Sans rewrite : le backend reçoit `/app/login`

**Cas d'usage** : Quand votre application ne connaît pas le préfixe `/app`.

#### Timeouts Personnalisés

Modifier les timeouts pour ce seul Ingress :

```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-connect-timeout: "300"
  nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
```

**Cas d'usage** : API avec traitement long (génération de rapports, traitement vidéo...).

#### Taille de Requête

Augmenter la limite pour un Ingress spécifique :

```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-body-size: "500m"
```

**Cas d'usage** : Application d'upload de vidéos.

#### CORS (Cross-Origin Resource Sharing)

Activer CORS pour les APIs :

```yaml
annotations:
  nginx.ingress.kubernetes.io/enable-cors: "true"
  nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
  nginx.ingress.kubernetes.io/cors-allow-origin: "*"
  nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
```

**Cas d'usage** : API REST appelée depuis des applications web externes.

#### Rate Limiting

Limiter le nombre de requêtes :

```yaml
annotations:
  nginx.ingress.kubernetes.io/limit-rps: "10"        # 10 requêtes par seconde
  nginx.ingress.kubernetes.io/limit-connections: "5" # 5 connexions simultanées
```

**Cas d'usage** : Protection contre les abus ou les attaques DDoS.

#### Authentification Basique

Protéger une application avec login/mot de passe :

```yaml
annotations:
  nginx.ingress.kubernetes.io/auth-type: basic
  nginx.ingress.kubernetes.io/auth-secret: basic-auth
  nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
```

Note : nécessite de créer un Secret `basic-auth` avec les identifiants.

**Cas d'usage** : Protéger un environnement de staging ou un outil d'administration.

#### Whitelist IP

Autoriser seulement certaines IPs :

```yaml
annotations:
  nginx.ingress.kubernetes.io/whitelist-source-range: "192.168.1.0/24,10.0.0.0/8"
```

**Cas d'usage** : Accès restreint à un réseau d'entreprise.

#### Backend Protocol

Spécifier le protocole backend (si HTTPS par exemple) :

```yaml
annotations:
  nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
```

**Cas d'usage** : Backend qui écoute déjà en HTTPS.

#### Session Affinity (Sticky Sessions)

Diriger un utilisateur toujours vers le même pod :

```yaml
annotations:
  nginx.ingress.kubernetes.io/affinity: "cookie"
  nginx.ingress.kubernetes.io/session-cookie-name: "route"
  nginx.ingress.kubernetes.io/session-cookie-max-age: "3600"
```

**Cas d'usage** : Applications qui stockent l'état de session en mémoire.

### Exemple d'Ingress Complet avec Annotations

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-application
  namespace: default
  annotations:
    # Redirection HTTPS automatique
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Upload jusqu'à 200MB
    nginx.ingress.kubernetes.io/proxy-body-size: "200m"

    # Timeouts augmentés
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "20"

    # CORS activé
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://mon-frontend.com"
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
            name: mon-api-service
            port:
              number: 8080
  tls:
  - hosts:
    - api.mon-site.com
    secretName: api-tls-secret
```

## 3. Snippets de Configuration Personnalisés

Pour des besoins très avancés, vous pouvez injecter directement de la configuration NGINX via des snippets.

### Configuration Snippet

Ajoute de la configuration NGINX directement :

```yaml
annotations:
  nginx.ingress.kubernetes.io/configuration-snippet: |
    more_set_headers "X-Custom-Header: valeur";
    if ($request_method = 'OPTIONS') {
      return 204;
    }
```

**⚠️ Attention** : Les snippets donnent un contrôle total mais peuvent casser la configuration NGINX s'ils sont mal écrits. À utiliser avec précaution !

### Server Snippet

Ajoute de la configuration au niveau du bloc server :

```yaml
annotations:
  nginx.ingress.kubernetes.io/server-snippet: |
    location /health {
      access_log off;
      return 200 "healthy\n";
    }
```

**Cas d'usage** : Ajouter des endpoints personnalisés ou des règles complexes.

## 4. Configuration TCP/UDP

Pour exposer des services non-HTTP (bases de données, services custom...) :

### ConfigMap TCP

Éditez le ConfigMap TCP :

```bash
microk8s kubectl edit configmap nginx-ingress-tcp-microk8s-conf -n ingress
```

Exemple pour exposer une base de données PostgreSQL sur le port 5432 :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-ingress-tcp-microk8s-conf
  namespace: ingress
data:
  "5432": "default/postgres-service:5432"
```

Format : `"PORT_EXTERNE": "NAMESPACE/SERVICE:PORT_SERVICE"`

### ConfigMap UDP

Même principe pour les services UDP :

```bash
microk8s kubectl edit configmap nginx-ingress-udp-microk8s-conf -n ingress
```

Exemple pour un serveur DNS :

```yaml
data:
  "53": "kube-system/kube-dns:53"
```

## 5. Vérification de la Configuration

### Voir la Configuration Active de NGINX

Pour voir la configuration NGINX générée automatiquement :

```bash
microk8s kubectl exec -n ingress -it nginx-ingress-microk8s-controller-xxxxx -- cat /etc/nginx/nginx.conf
```

Remplacez `xxxxx` par l'ID réel de votre pod.

### Tester la Configuration

Pour vérifier que la configuration NGINX est valide :

```bash
microk8s kubectl exec -n ingress nginx-ingress-microk8s-controller-xxxxx -- nginx -t
```

Sortie attendue :
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### Consulter les Logs en Direct

Pour suivre les requêtes en temps réel :

```bash
microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx -f
```

L'option `-f` (follow) affiche les nouveaux logs au fur et à mesure.

## 6. Bonnes Pratiques de Configuration

### 1. Commencez Simple

Ne surchargez pas votre configuration dès le début. Commencez avec les valeurs par défaut et ajoutez des options au fur et à mesure des besoins.

### 2. Utilisez la Hiérarchie

- **ConfigMap** : pour les paramètres communs à toutes les applications
- **Annotations** : pour les besoins spécifiques de chaque application

### 3. Documentez vos Choix

Ajoutez des commentaires dans vos manifestes YAML pour expliquer pourquoi vous avez choisi telle ou telle configuration :

```yaml
annotations:
  # Timeout augmenté car le backend génère des rapports longs (jusqu'à 5 minutes)
  nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
```

### 4. Testez les Changements

Avant d'appliquer en production :
1. Testez dans un environnement de développement
2. Vérifiez les logs après application
3. Surveillez les métriques de performance

### 5. Versionnez vos Configurations

Stockez vos manifestes YAML dans Git pour :
- Tracer l'historique des changements
- Faciliter les rollbacks en cas de problème
- Partager avec l'équipe

### 6. Surveillez les Performances

Après un changement de configuration, surveillez :
- Les temps de réponse
- Le taux d'erreurs
- L'utilisation des ressources (CPU, RAM)

## 7. Exemples de Configurations par Cas d'Usage

### Site Web Statique

Simple, avec redirection HTTPS :

```yaml
annotations:
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  nginx.ingress.kubernetes.io/rewrite-target: /
```

### API REST avec CORS

Pour une API publique :

```yaml
annotations:
  nginx.ingress.kubernetes.io/enable-cors: "true"
  nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
  nginx.ingress.kubernetes.io/cors-allow-origin: "*"
  nginx.ingress.kubernetes.io/limit-rps: "100"
```

### Application avec Upload de Fichiers

Pour permettre l'upload de gros fichiers :

```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-body-size: "1000m"
  nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
```

### Outil d'Administration Protégé

Avec authentification et restriction IP :

```yaml
annotations:
  nginx.ingress.kubernetes.io/auth-type: basic
  nginx.ingress.kubernetes.io/auth-secret: admin-auth
  nginx.ingress.kubernetes.io/auth-realm: "Administration"
  nginx.ingress.kubernetes.io/whitelist-source-range: "192.168.1.0/24"
```

### WebSocket

Pour les applications temps réel :

```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
  nginx.ingress.kubernetes.io/proxy-connect-timeout: "3600"
  nginx.ingress.kubernetes.io/websocket-services: "mon-service-ws"
```

## 8. Résolution de Problèmes Courants

### Problème 1 : Configuration Non Appliquée

**Symptôme** : Vous avez modifié le ConfigMap mais les changements ne sont pas pris en compte.

**Solution** :
1. Vérifiez que vous avez édité le bon ConfigMap (dans le namespace `ingress`)
2. Attendez 30-60 secondes (le contrôleur détecte les changements périodiquement)
3. Consultez les logs pour voir si le rechargement a eu lieu :
   ```bash
   microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx --tail=20
   ```

### Problème 2 : Annotation Non Reconnue

**Symptôme** : Votre annotation ne semble pas fonctionner.

**Solution** :
1. Vérifiez l'orthographe (les annotations sont sensibles à la casse)
2. Assurez-vous d'utiliser le bon préfixe : `nginx.ingress.kubernetes.io/`
3. Consultez la [documentation officielle](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/) pour la syntaxe exacte

### Problème 3 : Erreur 413 (Request Entity Too Large)

**Symptôme** : Erreur lors de l'upload de fichiers.

**Solution** :
Augmentez la taille maximale dans le ConfigMap et/ou l'annotation :
```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-body-size: "100m"
```

### Problème 4 : Timeout 504 Gateway Timeout

**Symptôme** : Les requêtes longues échouent.

**Solution** :
Augmentez les timeouts :
```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-connect-timeout: "300"
  nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
```

## Commandes de Référence Rapide

```bash
# Éditer le ConfigMap global
microk8s kubectl edit configmap nginx-load-balancer-microk8s-conf -n ingress

# Voir la config NGINX générée
microk8s kubectl exec -n ingress <pod-name> -- cat /etc/nginx/nginx.conf

# Tester la config NGINX
microk8s kubectl exec -n ingress <pod-name> -- nginx -t

# Voir les logs en direct
microk8s kubectl logs -n ingress -l app.kubernetes.io/name=ingress-nginx -f

# Recharger manuellement (si nécessaire)
microk8s kubectl rollout restart daemonset nginx-ingress-microk8s-controller -n ingress

# Voir toutes les annotations disponibles
microk8s kubectl explain ingress.metadata.annotations
```

## Documentation et Ressources

Pour aller plus loin, consultez :

- **Documentation officielle** : https://kubernetes.github.io/ingress-nginx/
- **Liste complète des annotations** : https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/
- **Options du ConfigMap** : https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/
- **Exemples** : https://github.com/kubernetes/ingress-nginx/tree/main/docs/examples

## Points Clés à Retenir

🔑 **Trois niveaux de configuration** : ConfigMap (global), Annotations (par Ingress), Snippets (avancé)

🔑 **ConfigMap pour les valeurs communes** : timeouts, taille des requêtes, SSL/TLS

🔑 **Annotations pour les besoins spécifiques** : CORS, rate limiting, authentification

🔑 **Rechargement automatique** : les changements sont appliqués sans interruption de service

🔑 **Testez toujours** : vérifiez les logs après chaque modification

🔑 **Commencez simple** : ajoutez des options au fur et à mesure des besoins

🔑 **Documentez vos choix** : commentez vos configurations pour l'équipe et vous-même

🔑 **La hiérarchie est votre amie** : configurez globalement, affinez localement

## Prochaines Étapes

Maintenant que vous maîtrisez la configuration de NGINX Ingress Controller, vous êtes prêt à :

1. **Acquérir un nom de domaine** (section 10.4)
2. **Configurer votre DNS** (section 10.5)
3. **Créer vos règles de routage** (sections 10.8 et 10.9)
4. **Ajouter des certificats SSL/TLS** (chapitre 11)

La configuration que vous avez apprise ici sera la base de toutes vos applications exposées via Ingress. Prenez le temps d'expérimenter avec différentes options pour bien les comprendre !

---

**📚 Résumé du chapitre** : NGINX Ingress Controller se configure à trois niveaux : ConfigMap pour les paramètres globaux, annotations pour les besoins spécifiques par Ingress, et snippets pour les cas avancés. Les modifications sont appliquées automatiquement sans interruption. Commencez avec une configuration simple et ajoutez des options au fur et à mesure de vos besoins.

⏭️ [Acquisition d'un nom de domaine](/10-ingress-et-routage/04-acquisition-dun-nom-de-domaine.md)
