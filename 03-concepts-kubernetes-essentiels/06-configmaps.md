🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.6 ConfigMaps

## Introduction

Jusqu'à présent, nous avons appris à déployer des applications dans Kubernetes. Mais comment gérer la **configuration** de ces applications ? Comment éviter de reconstruire une image Docker à chaque fois que vous changez une URL, un port ou un paramètre ?

Les **ConfigMaps** sont la solution Kubernetes pour séparer la configuration du code de l'application, rendant vos déploiements flexibles et réutilisables.

## Le problème : configuration codée en dur

### Scénario problématique

Imaginez que vous avez une application web avec cette configuration :

```javascript
// config.js (dans l'image Docker)
const config = {
  apiUrl: "https://api.production.example.com",
  port: 8080,
  debugMode: false,
  maxConnections: 100,
  timeout: 30
};
```

**Problèmes rencontrés** :

1. **Environnements différents** :
   - En dev : `apiUrl: "http://localhost:3000"`
   - En staging : `apiUrl: "https://api.staging.example.com"`
   - En prod : `apiUrl: "https://api.production.example.com"`
   - → Besoin de 3 images Docker différentes !

2. **Changements fréquents** :
   - Modifier un timeout : reconstruire l'image entière
   - Changer une URL : nouveau build, nouveau push, nouveau déploiement
   - Ajuster des paramètres : processus long et risqué

3. **Gestion complexe** :
   - Configuration éparpillée dans le code
   - Difficile de voir toute la configuration d'un coup d'œil
   - Impossible de modifier sans redéployer

4. **Sécurité** :
   - Configuration visible dans le code source
   - Difficile de séparer les configurations sensibles

### La solution : ConfigMaps

Les ConfigMaps permettent de :
- **Externaliser** la configuration de vos applications
- **Réutiliser** la même image Docker dans tous les environnements
- **Modifier** la configuration sans reconstruire l'image
- **Centraliser** toute la configuration dans Kubernetes

## Qu'est-ce qu'un ConfigMap ?

### Définition simple

Un **ConfigMap** est un objet Kubernetes qui stocke des données de configuration sous forme de paires **clé-valeur** ou de fichiers, et qui peut être injecté dans vos Pods.

### Analogie : le fichier de configuration externe

Imaginez un **lecteur DVD** (votre application) :

```
┌─────────────────────────────────────────┐
│         Lecteur DVD (Application)       │
│                                         │
│  Code de l'application (fixe)           │
│  ┌────────────────────────────────┐     │
│  │                                │     │
│  │   Logique métier               │     │
│  │   Interface utilisateur        │     │
│  │   Moteur de traitement         │     │
│  │                                │     │
│  └────────────────────────────────┘     │
│                                         │
│  Paramètres (variables) ← ConfigMap     │
│  ┌────────────────────────────────┐     │
│  │ • Langue: FR                   │     │
│  │ • Sous-titres: ON              │     │
│  │ • Volume: 50                   │     │
│  │ • Mode: HD                     │     │
│  └────────────────────────────────┘     │
└─────────────────────────────────────────┘
```

**Avantages** :
- Le **lecteur DVD** reste le même (l'image Docker)
- Seuls les **paramètres** changent (ConfigMap)
- Vous pouvez ajuster les paramètres sans changer le lecteur

### Ce que contient un ConfigMap

Un ConfigMap peut contenir :

1. **Paires clé-valeur simples** :
   ```yaml
   API_URL: "https://api.example.com"
   PORT: "8080"
   DEBUG: "false"
   ```

2. **Fichiers de configuration complets** :
   ```yaml
   nginx.conf: |
     server {
       listen 80;
       server_name example.com;
       location / {
         proxy_pass http://backend:8080;
       }
     }
   ```

3. **Fichiers de configuration structurés** (JSON, YAML, etc.) :
   ```yaml
   app-config.json: |
     {
       "database": {
         "host": "db.example.com",
         "port": 5432
       },
       "cache": {
         "enabled": true,
         "ttl": 300
       }
     }
   ```

## Créer un ConfigMap

### Méthode 1 : Depuis des valeurs littérales

La méthode la plus simple pour des valeurs simples.

```bash
microk8s kubectl create configmap mon-config \
  --from-literal=API_URL=https://api.example.com \
  --from-literal=PORT=8080 \
  --from-literal=DEBUG=false \
  --from-literal=MAX_CONNECTIONS=100
```

**Résultat** : Un ConfigMap nommé "mon-config" avec 4 clés.

### Méthode 2 : Depuis un fichier de propriétés

Créez d'abord un fichier avec vos configurations :

**fichier : app.properties**
```properties
API_URL=https://api.example.com
PORT=8080
DEBUG=false
MAX_CONNECTIONS=100
TIMEOUT=30
```

Puis créez le ConfigMap :
```bash
microk8s kubectl create configmap mon-config --from-file=app.properties
```

**Résultat** : Un ConfigMap avec une clé "app.properties" contenant tout le fichier.

### Méthode 3 : Depuis plusieurs fichiers

```bash
# Créer depuis tous les fichiers d'un répertoire
microk8s kubectl create configmap mon-config --from-file=config/

# Créer depuis des fichiers spécifiques
microk8s kubectl create configmap mon-config \
  --from-file=nginx.conf \
  --from-file=app-config.json
```

### Méthode 4 : Depuis un fichier YAML (déclarative)

La méthode recommandée pour la production.

**fichier : mon-configmap.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mon-config
  namespace: default
  labels:
    app: mon-application
data:
  # Valeurs simples (clé-valeur)
  API_URL: "https://api.example.com"
  PORT: "8080"
  DEBUG: "false"
  MAX_CONNECTIONS: "100"

  # Fichier de configuration complet
  nginx.conf: |
    server {
      listen 80;
      server_name example.com;

      location / {
        proxy_pass http://backend:8080;
        proxy_set_header Host $host;
      }
    }

  # Fichier JSON
  app-config.json: |
    {
      "database": {
        "host": "postgres.default.svc.cluster.local",
        "port": 5432,
        "name": "myapp"
      },
      "cache": {
        "enabled": true,
        "ttl": 300,
        "maxSize": "100MB"
      },
      "features": {
        "newUI": true,
        "betaFeatures": false
      }
    }
```

Appliquer :
```bash
microk8s kubectl apply -f mon-configmap.yaml
```

### Voir les ConfigMaps

```bash
# Lister tous les ConfigMaps
microk8s kubectl get configmaps
# Ou version courte :
microk8s kubectl get cm

# Voir les détails d'un ConfigMap
microk8s kubectl describe configmap mon-config

# Voir le contenu complet en YAML
microk8s kubectl get configmap mon-config -o yaml
```

## Utiliser un ConfigMap dans un Pod

Il existe plusieurs façons d'utiliser un ConfigMap dans vos Pods.

### 1. Variables d'environnement : valeur unique

Injecter une seule valeur du ConfigMap dans une variable d'environnement.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-app
spec:
  containers:
  - name: app
    image: mon-application:v1
    env:
    - name: API_URL              # Nom de la variable dans le conteneur
      valueFrom:
        configMapKeyRef:
          name: mon-config       # Nom du ConfigMap
          key: API_URL           # Clé dans le ConfigMap
    - name: PORT
      valueFrom:
        configMapKeyRef:
          name: mon-config
          key: PORT
```

**Dans le conteneur**, les variables sont disponibles :
```bash
echo $API_URL
# https://api.example.com

echo $PORT
# 8080
```

### 2. Variables d'environnement : toutes les clés

Injecter **toutes** les clés d'un ConfigMap comme variables d'environnement.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-app
spec:
  containers:
  - name: app
    image: mon-application:v1
    envFrom:                     # Toutes les clés du ConfigMap
    - configMapRef:
        name: mon-config
```

**Résultat** : Toutes les clés du ConfigMap deviennent des variables d'environnement.

Si le ConfigMap contient :
```yaml
data:
  API_URL: "https://api.example.com"
  PORT: "8080"
  DEBUG: "false"
```

Le conteneur aura :
```bash
$API_URL = "https://api.example.com"
$PORT = "8080"
$DEBUG = "false"
```

### 3. Volume : fichier unique

Monter une clé spécifique du ConfigMap comme un fichier dans le conteneur.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    volumeMounts:
    - name: config-volume
      mountPath: /etc/nginx/nginx.conf    # Chemin du fichier dans le conteneur
      subPath: nginx.conf                 # Clé du ConfigMap à monter
  volumes:
  - name: config-volume
    configMap:
      name: mon-config
```

**Résultat** : Le fichier `/etc/nginx/nginx.conf` dans le conteneur contient la valeur de la clé "nginx.conf" du ConfigMap.

### 4. Volume : répertoire complet

Monter toutes les clés du ConfigMap comme des fichiers dans un répertoire.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-app
spec:
  containers:
  - name: app
    image: mon-application:v1
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config              # Répertoire de montage
  volumes:
  - name: config-volume
    configMap:
      name: mon-config
```

**Résultat** : Dans `/etc/config/`, chaque clé du ConfigMap devient un fichier.

Si le ConfigMap contient :
```yaml
data:
  database.conf: "host=db.example.com\nport=5432"
  cache.conf: "enabled=true\nttl=300"
```

Le conteneur aura :
```
/etc/config/
  ├── database.conf   (contenu: host=db.example.com\nport=5432)
  └── cache.conf      (contenu: enabled=true\nttl=300)
```

### 5. Volume : sélection de clés spécifiques

Monter seulement certaines clés du ConfigMap.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-app
spec:
  containers:
  - name: app
    image: mon-application:v1
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: mon-config
      items:                               # Sélectionner des clés spécifiques
      - key: nginx.conf                    # Clé dans le ConfigMap
        path: nginx/nginx.conf             # Chemin relatif dans le volume
      - key: app-config.json
        path: app/config.json
```

**Résultat** :
```
/etc/config/
  ├── nginx/
  │   └── nginx.conf
  └── app/
      └── config.json
```

## Exemple complet : Application multi-environnements

### ConfigMap pour Développement

**fichier : configmap-dev.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev
  labels:
    environment: dev
data:
  API_URL: "http://api.dev.local:3000"
  DATABASE_HOST: "postgres.dev.svc.cluster.local"
  DATABASE_PORT: "5432"
  DATABASE_NAME: "myapp_dev"
  LOG_LEVEL: "debug"
  CACHE_ENABLED: "false"
  MAX_CONNECTIONS: "10"
  TIMEOUT: "60"
```

### ConfigMap pour Production

**fichier : configmap-prod.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
  labels:
    environment: production
data:
  API_URL: "https://api.example.com"
  DATABASE_HOST: "postgres.production.svc.cluster.local"
  DATABASE_PORT: "5432"
  DATABASE_NAME: "myapp_prod"
  LOG_LEVEL: "info"
  CACHE_ENABLED: "true"
  MAX_CONNECTIONS: "100"
  TIMEOUT: "30"
```

### Deployment utilisant le ConfigMap

**fichier : deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-application
  namespace: production            # Ou "dev" selon l'environnement
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mon-application
  template:
    metadata:
      labels:
        app: mon-application
    spec:
      containers:
      - name: app
        image: mon-application:v1.2.3    # Même image pour tous les environnements !
        envFrom:
        - configMapRef:
            name: app-config             # Utilise le ConfigMap du namespace
        ports:
        - containerPort: 8080
```

**Résultat** :
- **Même image Docker** utilisée partout
- **Configuration différente** selon le namespace (dev/production)
- **Facile à modifier** : changez le ConfigMap, redémarrez les Pods

## Mise à jour d'un ConfigMap

### Modifier un ConfigMap existant

**Méthode 1 : Éditer directement**
```bash
microk8s kubectl edit configmap mon-config
```

**Méthode 2 : Appliquer un nouveau fichier YAML**
```bash
# Modifier le fichier mon-configmap.yaml
microk8s kubectl apply -f mon-configmap.yaml
```

**Méthode 3 : Patch via ligne de commande**
```bash
microk8s kubectl patch configmap mon-config \
  -p '{"data":{"API_URL":"https://new-api.example.com"}}'
```

### Important : Les Pods ne se mettent pas à jour automatiquement

**Comportement par défaut** :
- Les variables d'environnement ne changent **jamais** après le démarrage du Pod
- Les volumes montent le ConfigMap se mettent à jour après quelques minutes (mais l'application doit recharger la config)

**Pour appliquer les changements** :
```bash
# Redémarrer les Pods du Deployment
microk8s kubectl rollout restart deployment mon-application

# Ou supprimer les Pods (ils seront recréés automatiquement)
microk8s kubectl delete pods -l app=mon-application
```

### Stratégie immutable (recommandée en production)

Plutôt que de modifier un ConfigMap existant, créez un nouveau ConfigMap avec un nouveau nom :

```yaml
# Version 1
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v1          # Nom avec version
data:
  API_URL: "https://api.v1.example.com"
---
# Version 2
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2          # Nouveau nom
data:
  API_URL: "https://api.v2.example.com"
```

Puis mettez à jour le Deployment pour pointer vers le nouveau ConfigMap :
```yaml
envFrom:
- configMapRef:
    name: app-config-v2        # Changé de v1 à v2
```

**Avantages** :
- Rollback facile (revenir à `app-config-v1`)
- Pas de période de transition incohérente
- Historique des configurations

### ConfigMaps immutables (Kubernetes 1.21+)

Vous pouvez marquer un ConfigMap comme immutable pour garantir qu'il ne changera jamais :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v1
data:
  API_URL: "https://api.example.com"
immutable: true               # Ne peut plus être modifié
```

**Avantages** :
- Protection contre les modifications accidentelles
- Meilleures performances (Kubernetes n'a pas à surveiller les changements)
- Force l'utilisation de la stratégie de versioning

## Patterns d'utilisation avancés

### Pattern 1 : Configuration par environnement

Utilisez des ConfigMaps différents pour chaque environnement, mais le même Deployment.

```bash
# Créer les ConfigMaps
microk8s kubectl create configmap app-config --from-literal=ENV=dev -n dev
microk8s kubectl create configmap app-config --from-literal=ENV=staging -n staging
microk8s kubectl create configmap app-config --from-literal=ENV=production -n production

# Déployer dans chaque namespace avec le même manifeste
microk8s kubectl apply -f deployment.yaml -n dev
microk8s kubectl apply -f deployment.yaml -n staging
microk8s kubectl apply -f deployment.yaml -n production
```

### Pattern 2 : Configuration partagée + spécifique

Combiner une configuration commune et une configuration spécifique.

**ConfigMap commun** :
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-common
data:
  COMPANY_NAME: "My Company"
  SUPPORT_EMAIL: "support@example.com"
  TIMEZONE: "Europe/Paris"
```

**ConfigMap spécifique à l'environnement** :
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-env
  namespace: production
data:
  API_URL: "https://api.example.com"
  DATABASE_HOST: "postgres.production.svc.cluster.local"
```

**Utilisation dans le Pod** :
```yaml
spec:
  containers:
  - name: app
    image: mon-application:v1
    envFrom:
    - configMapRef:
        name: app-config-common     # Configuration commune
    - configMapRef:
        name: app-config-env        # Configuration spécifique
```

### Pattern 3 : Configuration avec fichier de template

Utiliser un fichier de configuration avec des placeholders, puis utiliser un init container pour les remplacer.

**ConfigMap avec template** :
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-template
data:
  config.template: |
    {
      "api": {
        "url": "${API_URL}",
        "timeout": ${TIMEOUT}
      },
      "database": {
        "host": "${DB_HOST}",
        "port": ${DB_PORT}
      }
    }
```

**Init container pour substitution** :
```yaml
spec:
  initContainers:
  - name: config-builder
    image: busybox
    command:
    - sh
    - -c
    - |
      envsubst < /config-template/config.template > /config/config.json
    env:
    - name: API_URL
      value: "https://api.example.com"
    - name: TIMEOUT
      value: "30"
    - name: DB_HOST
      value: "postgres.svc.cluster.local"
    - name: DB_PORT
      value: "5432"
    volumeMounts:
    - name: config-template
      mountPath: /config-template
    - name: config
      mountPath: /config
  containers:
  - name: app
    image: mon-application:v1
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config-template
    configMap:
      name: app-config-template
  - name: config
    emptyDir: {}
```

### Pattern 4 : Configuration de plusieurs applications

Un ConfigMap central pour configurer plusieurs applications.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: shared-endpoints
data:
  # URLs des services internes
  AUTH_SERVICE_URL: "http://auth.default.svc.cluster.local:8080"
  USER_SERVICE_URL: "http://users.default.svc.cluster.local:8080"
  PAYMENT_SERVICE_URL: "http://payments.default.svc.cluster.local:8080"
  NOTIFICATION_SERVICE_URL: "http://notifications.default.svc.cluster.local:8080"
```

Toutes les applications peuvent utiliser ce ConfigMap :
```yaml
envFrom:
- configMapRef:
    name: shared-endpoints
```

## Limites et considérations

### Taille maximale

**Limite** : Un ConfigMap ne peut pas dépasser **1 MiB** (1 048 576 bytes).

**Pour des données plus volumineuses** :
- Divisez en plusieurs ConfigMaps
- Utilisez des PersistentVolumes
- Stockez les données dans un service externe (S3, etc.)

### Pas pour les données sensibles

**ConfigMaps ne sont PAS chiffrés** et sont visibles en clair.

**✅ Bon pour ConfigMap** :
```yaml
data:
  API_URL: "https://api.example.com"
  PORT: "8080"
  FEATURE_FLAG: "true"
```

**❌ Mauvais pour ConfigMap (utilisez des Secrets)** :
```yaml
data:
  DATABASE_PASSWORD: "super_secret_password"    # ❌ Ne pas faire !
  API_KEY: "sk_live_xyz123"                     # ❌ Ne pas faire !
```

### Performance avec de nombreux ConfigMaps

Si vous avez des centaines de ConfigMaps et de Pods :
- Utilisez `immutable: true` pour réduire la charge sur Kubernetes
- Limitez le nombre de ConfigMaps par Pod
- Envisagez des solutions de configuration externe

## Bonnes pratiques

### 1. Nommer clairement vos ConfigMaps

**✅ Bon** :
```yaml
metadata:
  name: nginx-config-v1
  name: app-database-config
  name: logging-config-production
```

**❌ Mauvais** :
```yaml
metadata:
  name: config
  name: data
  name: conf123
```

### 2. Utiliser des labels

```yaml
metadata:
  name: app-config
  labels:
    app: mon-application
    environment: production
    config-type: application
    version: v1
```

Les labels permettent de :
- Filtrer les ConfigMaps : `kubectl get cm -l environment=production`
- Organiser et documenter
- Automatiser les opérations

### 3. Versionner vos ConfigMaps

```yaml
# Version 1
metadata:
  name: app-config-v1

# Version 2
metadata:
  name: app-config-v2
```

Ou utiliser des labels :
```yaml
metadata:
  name: app-config
  labels:
    version: "2.0"
```

### 4. Un ConfigMap par responsabilité

**✅ Bon (séparation claire)** :
```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    # Configuration nginx
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-env-vars
data:
  API_URL: "https://api.example.com"
  PORT: "8080"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: database-config
data:
  host: "postgres.svc.cluster.local"
  port: "5432"
```

**❌ Mauvais (tout mélangé)** :
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: all-configs
data:
  nginx.conf: |
    # Configuration nginx
  API_URL: "https://api.example.com"
  database_host: "postgres.svc.cluster.local"
  prometheus_config: |
    # Configuration prometheus
```

### 5. Documenter avec des annotations

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  annotations:
    description: "Configuration principale pour l'application frontend"
    owner: "team-frontend@example.com"
    last-updated: "2024-01-15"
    documentation: "https://wiki.example.com/app-config"
data:
  API_URL: "https://api.example.com"
```

### 6. Valider la configuration avant déploiement

```bash
# Vérifier la syntaxe YAML
microk8s kubectl apply -f configmap.yaml --dry-run=client

# Voir le résultat sans l'appliquer
microk8s kubectl apply -f configmap.yaml --dry-run=server -o yaml
```

### 7. Utiliser des valeurs par défaut dans l'application

Si un ConfigMap manque ou une clé n'existe pas, l'application ne doit pas crasher.

**Dans votre code** :
```python
# Python exemple
import os

api_url = os.getenv('API_URL', 'http://localhost:3000')  # Valeur par défaut
port = int(os.getenv('PORT', '8080'))
debug = os.getenv('DEBUG', 'false').lower() == 'true'
```

### 8. Tester les modifications en dev d'abord

```bash
# 1. Modifier le ConfigMap en dev
microk8s kubectl apply -f configmap-dev.yaml -n dev

# 2. Redémarrer les Pods en dev
microk8s kubectl rollout restart deployment mon-app -n dev

# 3. Tester

# 4. Si OK, déployer en staging puis production
```

### 9. Garder les ConfigMaps dans Git

Versionnez vos ConfigMaps avec votre code :

```
projet/
├── kubernetes/
│   ├── configmaps/
│   │   ├── app-config-dev.yaml
│   │   ├── app-config-staging.yaml
│   │   └── app-config-prod.yaml
│   ├── deployments/
│   └── services/
└── src/
```

### 10. Éviter les valeurs codées en dur dans les Deployments

**❌ Mauvais** :
```yaml
env:
- name: API_URL
  value: "https://api.example.com"      # Codé en dur
```

**✅ Bon** :
```yaml
env:
- name: API_URL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: API_URL                       # Depuis ConfigMap
```

## Commandes utiles

### Gestion des ConfigMaps

```bash
# Créer un ConfigMap depuis des valeurs
microk8s kubectl create configmap mon-config \
  --from-literal=key1=value1 \
  --from-literal=key2=value2

# Créer depuis un fichier
microk8s kubectl create configmap mon-config --from-file=config.properties

# Créer depuis plusieurs fichiers
microk8s kubectl create configmap mon-config --from-file=config/

# Créer depuis un YAML
microk8s kubectl apply -f configmap.yaml

# Lister les ConfigMaps
microk8s kubectl get configmaps
microk8s kubectl get cm

# Voir les détails
microk8s kubectl describe configmap mon-config

# Voir le contenu complet
microk8s kubectl get configmap mon-config -o yaml

# Voir une clé spécifique
microk8s kubectl get configmap mon-config -o jsonpath='{.data.API_URL}'
```

### Modification

```bash
# Éditer interactivement
microk8s kubectl edit configmap mon-config

# Appliquer un fichier modifié
microk8s kubectl apply -f configmap.yaml

# Patch une valeur
microk8s kubectl patch configmap mon-config \
  -p '{"data":{"API_URL":"https://new-url.example.com"}}'

# Supprimer un ConfigMap
microk8s kubectl delete configmap mon-config
```

### Débogage

```bash
# Vérifier quels Pods utilisent un ConfigMap
microk8s kubectl get pods -o json | \
  jq '.items[] | select(.spec.volumes[]?.configMap.name=="mon-config") | .metadata.name'

# Voir les variables d'environnement d'un Pod
microk8s kubectl exec mon-pod -- env

# Voir le contenu d'un fichier monté depuis un ConfigMap
microk8s kubectl exec mon-pod -- cat /etc/config/app.conf

# Voir tous les volumes montés
microk8s kubectl exec mon-pod -- df -h
```

## Dépannage

### Problème 1 : ConfigMap non trouvé

**Symptôme** :
```
Error: configmap "mon-config" not found
```

**Vérifications** :
```bash
# Vérifier que le ConfigMap existe
microk8s kubectl get configmap mon-config

# Vérifier dans le bon namespace
microk8s kubectl get configmap mon-config -n production

# Lister tous les ConfigMaps
microk8s kubectl get configmaps --all-namespaces | grep mon-config
```

**Solution** : Créez le ConfigMap ou vérifiez le namespace.

### Problème 2 : Pod en CrashLoopBackOff à cause d'une clé manquante

**Symptôme** :
```
Pod mon-app: CrashLoopBackOff
```

**Cause** : Le Pod référence une clé qui n'existe pas dans le ConfigMap.

```yaml
env:
- name: API_URL
  valueFrom:
    configMapKeyRef:
      name: mon-config
      key: API_URL_TYPO       # ❌ Cette clé n'existe pas !
```

**Vérification** :
```bash
# Voir les clés disponibles
microk8s kubectl get configmap mon-config -o yaml

# Voir les logs du Pod
microk8s kubectl logs mon-app
```

**Solution** : Corriger le nom de la clé ou ajouter la clé manquante au ConfigMap.

### Problème 3 : Configuration non mise à jour

**Symptôme** : Vous avez modifié le ConfigMap mais les Pods utilisent encore l'ancienne configuration.

**Causes** :
1. **Variables d'environnement** : Ne sont jamais mises à jour après le démarrage du Pod
2. **Volumes** : Sont mis à jour, mais l'application doit recharger la config

**Solution** :
```bash
# Redémarrer les Pods
microk8s kubectl rollout restart deployment mon-app

# Ou forcer la suppression (ils seront recréés)
microk8s kubectl delete pods -l app=mon-app
```

### Problème 4 : ConfigMap trop volumineux

**Symptôme** :
```
Error: ConfigMap "mon-config" is invalid:
data: Too long: must have at most 1048576 characters
```

**Solution** :
- Diviser en plusieurs ConfigMaps plus petits
- Utiliser un PersistentVolume pour les gros fichiers
- Stocker dans un système externe (S3, etc.)

### Problème 5 : Erreur de syntaxe dans le fichier de configuration

**Symptôme** : L'application plante car le fichier de configuration est invalide.

**Vérification** :
```bash
# Extraire le fichier du ConfigMap
microk8s kubectl get configmap mon-config -o jsonpath='{.data.app\.json}' > app.json

# Vérifier la syntaxe (exemple pour JSON)
cat app.json | jq .

# Vérifier la syntaxe (exemple pour YAML)
cat app.yaml | yamllint -
```

**Solution** : Corriger la syntaxe dans le ConfigMap.

## Résumé des points clés

- Les **ConfigMaps** séparent la configuration du code de l'application
- Permettent de **réutiliser** la même image Docker dans tous les environnements
- Peuvent contenir des **paires clé-valeur** ou des **fichiers complets**
- S'injectent dans les Pods via **variables d'environnement** ou **volumes**
- **Ne sont pas chiffrés** : utilisez des Secrets pour les données sensibles
- Les **modifications** ne sont pas automatiquement propagées aux Pods
- Taille maximale : **1 MiB**
- Utilisez la stratégie **immutable** pour la production
- Versionnez et documentez vos ConfigMaps
- Testez toujours en dev avant de déployer en production

## Prochaines étapes

Maintenant que vous maîtrisez les ConfigMaps, vous êtes prêt à découvrir :
- **Les Secrets** : comment stocker et gérer les données sensibles (mots de passe, clés API, certificats)
- **Les Volumes persistants** : comment persister les données au-delà du cycle de vie des Pods
- **Les StatefulSets** : comment déployer des applications avec état qui nécessitent des identités stables

Les ConfigMaps sont essentiels pour créer des applications cloud-native flexibles et portables !

⏭️ [Secrets](/03-concepts-kubernetes-essentiels/07-secrets.md)
