🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.4 Variables d'Environnement

## Introduction

Les variables d'environnement sont essentielles pour configurer vos applications de manière flexible. Elles permettent de modifier le comportement d'une application sans avoir à reconstruire l'image Docker ou modifier le code source.

Dans cette section, nous allons apprendre à :
- Définir des variables d'environnement simples
- Utiliser des informations du cluster (Downward API)
- Référencer d'autres variables
- Adopter les bonnes pratiques

## Qu'est-ce qu'une Variable d'Environnement ?

Une **variable d'environnement** est une paire clé-valeur accessible par une application en cours d'exécution.

**Analogie :** Pensez aux variables d'environnement comme aux réglages d'un appareil électronique. Vous pouvez changer la langue, le volume, ou la luminosité sans avoir à acheter un nouvel appareil.

### Exemples courants de variables d'environnement

| Variable | Exemple de valeur | Usage |
|----------|-------------------|-------|
| `DATABASE_URL` | `postgres://db:5432/myapp` | URL de connexion à la base de données |
| `API_KEY` | `abc123xyz789` | Clé d'API externe |
| `ENVIRONMENT` | `production` | Environnement d'exécution |
| `LOG_LEVEL` | `info` | Niveau de journalisation |
| `PORT` | `8080` | Port d'écoute de l'application |
| `MAX_CONNECTIONS` | `100` | Limite de connexions |

## Pourquoi Utiliser des Variables d'Environnement dans Kubernetes ?

### Avantages

✅ **Séparation de la configuration** : Le code reste identique entre dev, staging et production
✅ **Flexibilité** : Changez la configuration sans rebuild de l'image
✅ **Sécurité** : Ne pas coder en dur les secrets dans le code
✅ **Réutilisabilité** : Même image Docker pour plusieurs environnements
✅ **Facilité de déploiement** : Configuration déclarative via YAML

### Le principe des 12-Factor Apps

Les applications cloud-native suivent souvent les **12-Factor App** principes, dont :

> **III. Config** : Stockez la configuration dans l'environnement

Cela signifie que tout ce qui varie entre les déploiements (credentials, URLs, etc.) doit être dans des variables d'environnement.

## Méthode 1 : Variables Statiques Simples

### Définir des variables dans un Pod

Voici un exemple simple avec un Pod qui affiche des variables d'environnement :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
  labels:
    app: demo
spec:
  containers:
  - name: demo-container
    image: busybox
    command: ['sh', '-c', 'echo "Hello from $ENVIRONMENT! Port: $APP_PORT" && sleep 3600']
    env:
    - name: ENVIRONMENT
      value: "production"
    - name: APP_PORT
      value: "8080"
    - name: LOG_LEVEL
      value: "info"
```

**Décortiquons la section `env` :**

```yaml
env:
- name: ENVIRONMENT      # Nom de la variable
  value: "production"    # Valeur de la variable
- name: APP_PORT
  value: "8080"
- name: LOG_LEVEL
  value: "info"
```

**Points importants :**
- `env` : Liste de variables d'environnement
- `name` : Nom de la variable (en majuscules par convention)
- `value` : Valeur (toujours une chaîne de caractères, même pour les nombres)

### Déployer et tester

```bash
# Créer le Pod
microk8s kubectl apply -f env-demo.yaml

# Voir les logs (la commande echo affiche les variables)
microk8s kubectl logs env-demo

# Sortie attendue :
# Hello from production! Port: 8080
```

### Vérifier les variables dans un conteneur

```bash
# Accéder au shell du conteneur
microk8s kubectl exec -it env-demo -- sh

# Lister toutes les variables d'environnement
env

# Afficher une variable spécifique
echo $ENVIRONMENT
echo $APP_PORT
```

### Exemple avec une application web

Voici un exemple plus réaliste avec Nginx :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx:1.21-alpine
        ports:
        - containerPort: 80
        env:
        - name: ENVIRONMENT
          value: "production"
        - name: APP_NAME
          value: "My Web Application"
        - name: VERSION
          value: "1.0.0"
        - name: ENABLE_CACHE
          value: "true"
        - name: MAX_WORKERS
          value: "4"
```

**Notes sur les types de valeurs :**
- Les booléens sont des chaînes : `"true"` ou `"false"`
- Les nombres sont des chaînes : `"4"`, `"8080"`
- L'application doit convertir ces chaînes au type approprié

## Méthode 2 : Utiliser la Downward API

La **Downward API** permet d'injecter des informations sur le Pod ou le nœud dans vos conteneurs.

### Informations disponibles

| Type | Exemples |
|------|----------|
| **Pod** | Nom, namespace, IP, labels, annotations |
| **Conteneur** | Limites CPU/RAM, requêtes |
| **Nœud** | Nom du nœud |

### Exemple : Injecter le nom et le namespace du Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-api-demo
  namespace: default
  labels:
    app: demo
    version: "1.0"
spec:
  containers:
  - name: demo
    image: busybox
    command: ['sh', '-c', 'echo "Pod: $POD_NAME, Namespace: $POD_NAMESPACE, IP: $POD_IP" && sleep 3600']
    env:
    # Nom du Pod
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name

    # Namespace du Pod
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace

    # IP du Pod
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP

    # Nom du nœud
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
```

**Nouveaux éléments :**

```yaml
- name: POD_NAME
  valueFrom:           # Au lieu de "value"
    fieldRef:          # Référence à un champ du Pod
      fieldPath: metadata.name
```

**Champs disponibles avec `fieldRef` :**

| fieldPath | Description |
|-----------|-------------|
| `metadata.name` | Nom du Pod |
| `metadata.namespace` | Namespace du Pod |
| `metadata.labels['<KEY>']` | Valeur d'un label spécifique |
| `metadata.annotations['<KEY>']` | Valeur d'une annotation |
| `status.podIP` | Adresse IP du Pod |
| `spec.nodeName` | Nom du nœud hébergeant le Pod |
| `spec.serviceAccountName` | Nom du compte de service |

### Exemple : Récupérer les labels

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: label-demo
  labels:
    app: myapp
    version: "2.0"
    tier: frontend
spec:
  containers:
  - name: demo
    image: busybox
    command: ['sh', '-c', 'echo "App: $APP_LABEL, Version: $VERSION_LABEL" && sleep 3600']
    env:
    - name: APP_LABEL
      valueFrom:
        fieldRef:
          fieldPath: metadata.labels['app']

    - name: VERSION_LABEL
      valueFrom:
        fieldRef:
          fieldPath: metadata.labels['version']
```

**⚠️ Attention :** Les clés de labels avec des caractères spéciaux doivent être entre guillemets simples.

### Exemple : Limites de ressources

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: demo
    image: busybox
    command: ['sh', '-c', 'echo "CPU Limit: $CPU_LIMIT, Memory Limit: $MEMORY_LIMIT" && sleep 3600']
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    env:
    - name: CPU_LIMIT
      valueFrom:
        resourceFieldRef:
          containerName: demo
          resource: limits.cpu

    - name: MEMORY_LIMIT
      valueFrom:
        resourceFieldRef:
          containerName: demo
          resource: limits.memory

    - name: CPU_REQUEST
      valueFrom:
        resourceFieldRef:
          containerName: demo
          resource: requests.cpu
```

**Nouveaux éléments :**

```yaml
valueFrom:
  resourceFieldRef:      # Pour les ressources du conteneur
    containerName: demo
    resource: limits.cpu
```

**Ressources disponibles avec `resourceFieldRef` :**
- `limits.cpu`
- `limits.memory`
- `limits.ephemeral-storage`
- `requests.cpu`
- `requests.memory`
- `requests.ephemeral-storage`

## Méthode 3 : Variables Dérivées (Substitution)

Kubernetes ne supporte pas nativement la substitution de variables (comme `$HOME/app`), mais vous pouvez référencer d'autres variables dans vos commandes.

### Exemple : Construire une URL

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: var-substitution
spec:
  containers:
  - name: app
    image: busybox
    env:
    - name: DB_HOST
      value: "postgres.default.svc.cluster.local"
    - name: DB_PORT
      value: "5432"
    - name: DB_NAME
      value: "myapp"
    - name: DB_USER
      value: "appuser"

    # Cette variable utilise les autres dans la commande
    command:
    - sh
    - -c
    - |
      export DB_URL="postgres://${DB_USER}@${DB_HOST}:${DB_PORT}/${DB_NAME}"
      echo "Database URL: $DB_URL"
      sleep 3600
```

**Fonctionnement :**
- Les variables `DB_HOST`, `DB_PORT`, etc., sont définies séparément
- La commande shell les combine avec `${VAR_NAME}`
- La variable `DB_URL` n'existe qu'à l'intérieur du conteneur

### Alternative : Utiliser un initContainer

Pour des configurations plus complexes, vous pouvez utiliser un **initContainer** qui prépare les variables :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-config
spec:
  initContainers:
  - name: config-builder
    image: busybox
    env:
    - name: API_HOST
      value: "api.example.com"
    - name: API_VERSION
      value: "v2"
    command:
    - sh
    - -c
    - |
      echo "API_ENDPOINT=https://${API_HOST}/${API_VERSION}" > /config/env.txt
    volumeMounts:
    - name: config
      mountPath: /config

  containers:
  - name: app
    image: busybox
    command:
    - sh
    - -c
    - |
      source /config/env.txt
      echo "Using endpoint: $API_ENDPOINT"
      sleep 3600
    volumeMounts:
    - name: config
      mountPath: /config

  volumes:
  - name: config
    emptyDir: {}
```

**Comment ça marche :**
1. L'initContainer s'exécute en premier
2. Il crée un fichier avec les variables combinées
3. Le conteneur principal lit ce fichier

## Exemple Complet : Application Multi-Environnements

Voici un exemple réaliste montrant comment gérer différents environnements :

### Fichier : webapp-dev.yaml (Développement)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: development
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
        environment: dev
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080
        env:
        # Configuration de base
        - name: ENVIRONMENT
          value: "development"
        - name: PORT
          value: "8080"

        # Base de données
        - name: DB_HOST
          value: "postgres-dev.development.svc.cluster.local"
        - name: DB_PORT
          value: "5432"
        - name: DB_NAME
          value: "myapp_dev"

        # Logging
        - name: LOG_LEVEL
          value: "debug"
        - name: LOG_FORMAT
          value: "json"

        # Features
        - name: ENABLE_DEBUG
          value: "true"
        - name: ENABLE_PROFILING
          value: "true"

        # Downward API
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP

        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
```

### Fichier : webapp-prod.yaml (Production)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
        environment: prod
    spec:
      containers:
      - name: app
        image: myapp:1.2.0  # Version fixe en production
        ports:
        - containerPort: 8080
        env:
        # Configuration de base
        - name: ENVIRONMENT
          value: "production"
        - name: PORT
          value: "8080"

        # Base de données
        - name: DB_HOST
          value: "postgres-prod.production.svc.cluster.local"
        - name: DB_PORT
          value: "5432"
        - name: DB_NAME
          value: "myapp_prod"

        # Logging
        - name: LOG_LEVEL
          value: "warn"
        - name: LOG_FORMAT
          value: "json"

        # Features (désactivées en prod pour la performance)
        - name: ENABLE_DEBUG
          value: "false"
        - name: ENABLE_PROFILING
          value: "false"

        # Downward API
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace

        resources:
          requests:
            memory: "256Mi"
            cpu: "500m"
          limits:
            memory: "512Mi"
            cpu: "1000m"
```

**Différences clés :**
- **Répliques** : 1 en dev, 3 en prod
- **Image** : `latest` en dev, version fixe en prod
- **Ressources** : Plus importantes en prod
- **Log level** : `debug` en dev, `warn` en prod
- **Features** : Debug activé en dev, désactivé en prod

## Variables Spéciales de Kubernetes

Kubernetes injecte automatiquement certaines variables d'environnement dans tous les conteneurs :

### Variables des Services

Pour chaque Service dans le même namespace, Kubernetes crée des variables :

```bash
# Format : <SERVICE_NAME>_SERVICE_HOST et <SERVICE_NAME>_SERVICE_PORT
NGINX_SERVICE_SERVICE_HOST=10.152.183.45
NGINX_SERVICE_SERVICE_PORT=80

POSTGRES_SERVICE_HOST=10.152.183.100
POSTGRES_SERVICE_PORT=5432
```

**Note :** Le nom du Service est converti en majuscules avec underscores.

### Vérifier les variables injectées

```bash
# Lister toutes les variables dans un Pod
microk8s kubectl exec <pod-name> -- env | sort

# Filtrer les variables de services
microk8s kubectl exec <pod-name> -- env | grep SERVICE
```

### Variables de système

```bash
HOSTNAME=webapp-5d59d67564-7k9xm        # Nom du conteneur
HOME=/root                              # Répertoire home
PATH=/usr/local/sbin:/usr/local/bin...  # PATH système
```

## Ordre de Priorité des Variables

Si une variable est définie à plusieurs endroits, voici l'ordre de priorité (du plus prioritaire au moins) :

1. **Commande/Arguments du conteneur** : Variables définies dans `command` ou `args`
2. **Section `env`** : Variables définies dans le manifeste
3. **ConfigMap/Secret** : Variables depuis ConfigMap ou Secret (voir section 4.5)
4. **Image Docker** : Variables définies dans le Dockerfile (ENV)

**Exemple :**

```yaml
# Dans le Dockerfile de l'image :
# ENV LOG_LEVEL=info

# Dans le manifeste Kubernetes :
env:
- name: LOG_LEVEL
  value: "debug"  # Cette valeur écrase celle du Dockerfile
```

## Conventions de Nommage

### Bonnes pratiques

✅ **MAJUSCULES avec underscores**
```yaml
- name: DATABASE_URL
- name: API_KEY
- name: MAX_CONNECTIONS
```

✅ **Noms descriptifs**
```yaml
- name: SMTP_HOST
- name: SMTP_PORT
- name: SMTP_USER
```

✅ **Préfixes pour grouper**
```yaml
- name: DB_HOST
- name: DB_PORT
- name: DB_NAME
- name: REDIS_HOST
- name: REDIS_PORT
```

❌ **À éviter :**
```yaml
- name: host          # Trop vague
- name: DatabaseUrl   # PascalCase (peu courant)
- name: db-host       # Tirets (préférez underscores)
```

## Commandes Utiles

### Lister les variables d'un Pod

```bash
# Voir toutes les variables
microk8s kubectl exec <pod-name> -- env

# Filtrer une variable spécifique
microk8s kubectl exec <pod-name> -- env | grep DATABASE

# Afficher une variable spécifique
microk8s kubectl exec <pod-name> -- sh -c 'echo $ENVIRONMENT'
```

### Voir les variables dans la définition du Pod

```bash
# Obtenir le YAML complet d'un Pod en cours d'exécution
microk8s kubectl get pod <pod-name> -o yaml

# Extraire uniquement la section env
microk8s kubectl get pod <pod-name> -o jsonpath='{.spec.containers[0].env}'
```

### Déboguer les variables

```bash
# Démarrer un shell interactif
microk8s kubectl exec -it <pod-name> -- sh

# Dans le shell, tester les variables
echo $DATABASE_URL
echo $LOG_LEVEL
env | grep -i db
```

## Debugging des Variables d'Environnement

### Problème : Variable non définie

**Symptôme :** L'application se plaint qu'une variable est absente.

**Vérifications :**

```bash
# 1. Vérifier le manifeste
microk8s kubectl get deployment webapp -o yaml | grep -A 10 "env:"

# 2. Vérifier dans le Pod en cours
microk8s kubectl exec <pod-name> -- env | grep VAR_NAME

# 3. Vérifier les logs du conteneur
microk8s kubectl logs <pod-name>
```

**Solutions courantes :**
- La variable est mal orthographiée dans le manifeste
- Le Pod n'a pas été redémarré après la modification
- La variable est définie mais avec une valeur vide

### Problème : Variable incorrecte

**Symptôme :** L'application utilise une valeur inattendue.

**Diagnostic :**

```bash
# Voir toutes les variables et chercher des doublons
microk8s kubectl exec <pod-name> -- env | sort

# Vérifier l'ordre de priorité
# 1. Regarder dans l'image Docker (si accessible)
docker inspect myapp:latest | grep Env

# 2. Regarder dans le manifeste
microk8s kubectl get pod <pod-name> -o yaml
```

### Problème : Downward API ne fonctionne pas

**Symptôme :** Les variables référençant des champs Pod sont vides.

**Vérifications :**

```bash
# Vérifier la syntaxe du fieldPath
microk8s kubectl describe pod <pod-name>

# Chercher des erreurs dans les événements
microk8s kubectl get events --field-selector involvedObject.name=<pod-name>
```

**Erreurs courantes :**
- `fieldPath` incorrect : `metadata.labels[app]` au lieu de `metadata.labels['app']`
- Référence à un champ qui n'existe pas
- Utilisation de `resourceFieldRef` au lieu de `fieldRef`

## Bonnes Pratiques

### 1. Séparer les configurations par environnement

Créez des fichiers séparés :
```
manifests/
├── base/
│   ├── deployment.yaml
│   └── service.yaml
├── dev/
│   └── env-config.yaml
├── staging/
│   └── env-config.yaml
└── production/
    └── env-config.yaml
```

### 2. Documenter les variables requises

Ajoutez des commentaires dans vos manifestes :

```yaml
env:
# Base de données (REQUIS)
- name: DATABASE_URL
  value: "postgres://..."

# Logging (OPTIONNEL, défaut: info)
- name: LOG_LEVEL
  value: "debug"

# Features (OPTIONNEL, défaut: false)
- name: ENABLE_DEBUG
  value: "true"
```

### 3. Utiliser des valeurs par défaut dans l'application

Votre application devrait gérer les variables absentes :

```python
# Exemple en Python
import os

# Avec valeur par défaut
log_level = os.getenv('LOG_LEVEL', 'info')
port = int(os.getenv('PORT', '8080'))
debug = os.getenv('ENABLE_DEBUG', 'false').lower() == 'true'
```

### 4. Valider les variables au démarrage

Faites échouer rapidement si des variables critiques manquent :

```python
# Exemple en Python
required_vars = ['DATABASE_URL', 'API_KEY']
missing = [var for var in required_vars if not os.getenv(var)]

if missing:
    print(f"ERROR: Missing required environment variables: {missing}")
    sys.exit(1)
```

### 5. Ne jamais mettre de secrets en clair

❌ **Mauvais :**
```yaml
env:
- name: DATABASE_PASSWORD
  value: "super_secret_password_123"  # JAMAIS ça !
```

✅ **Bon :**
```yaml
env:
- name: DATABASE_PASSWORD
  valueFrom:
    secretKeyRef:  # Utiliser un Secret (voir section 4.7)
      name: db-credentials
      key: password
```

Nous verrons les Secrets en détail dans la section 4.7.

### 6. Grouper logiquement les variables

```yaml
env:
# === Configuration de base ===
- name: ENVIRONMENT
  value: "production"
- name: PORT
  value: "8080"

# === Base de données ===
- name: DB_HOST
  value: "postgres.default.svc.cluster.local"
- name: DB_PORT
  value: "5432"

# === Cache ===
- name: REDIS_HOST
  value: "redis.default.svc.cluster.local"
- name: REDIS_PORT
  value: "6379"

# === Monitoring ===
- name: METRICS_ENABLED
  value: "true"
- name: METRICS_PORT
  value: "9090"
```

### 7. Utiliser la Downward API pour les métadonnées

Injectez toujours les informations utiles du Pod :

```yaml
env:
- name: POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
- name: POD_NAMESPACE
  valueFrom:
    fieldRef:
      fieldPath: metadata.namespace
- name: POD_IP
  valueFrom:
    fieldRef:
      fieldPath: status.podIP
```

Cela aide pour :
- Le logging structuré
- Le distributed tracing
- Le debugging

### 8. Éviter les valeurs hardcodées

❌ **Mauvais :**
```yaml
env:
- name: API_URL
  value: "http://192.168.1.100:8080/api"
```

✅ **Bon :**
```yaml
env:
- name: API_HOST
  value: "api-service.default.svc.cluster.local"
- name: API_PORT
  value: "8080"
```

Utilisez les noms DNS Kubernetes qui sont stables.

## Limites et Considérations

### Taille maximale

- **Variable individuelle** : Aucune limite stricte, mais restez raisonnable
- **Total des variables** : Environ 1 MB pour tous les paramètres d'un Pod

**Conseil :** Si vous avez beaucoup de configuration, utilisez plutôt ConfigMaps (section suivante).

### Sensibilité aux changements

**⚠️ Important :** Les variables d'environnement sont définies au **démarrage du conteneur**.

Pour appliquer des changements :
1. Modifiez le manifeste
2. Appliquez avec `kubectl apply`
3. **Le Pod doit redémarrer** pour prendre en compte les changements

```bash
# Après modification, forcer le redémarrage
microk8s kubectl rollout restart deployment webapp
```

### Performance

Les variables d'environnement ont un impact minimal sur les performances. Cependant :
- Des milliers de variables peuvent ralentir le démarrage
- Préférez ConfigMaps pour de grandes configurations

## Récapitulatif

Dans cette section, vous avez appris à :

✅ Définir des **variables d'environnement statiques** dans les manifestes
✅ Utiliser la **Downward API** pour injecter des informations du Pod
✅ Référencer les **ressources du conteneur** (CPU, RAM)
✅ Combiner des variables dans les commandes shell
✅ Gérer différents **environnements** (dev, staging, production)
✅ Appliquer les **bonnes pratiques** de nommage et d'organisation

**Points clés :**

- Les variables d'environnement permettent la **séparation de la configuration**
- Utilisez `env` avec `value` pour les valeurs statiques
- Utilisez `valueFrom.fieldRef` pour la Downward API
- Utilisez `valueFrom.resourceFieldRef` pour les limites de ressources
- Ne mettez **jamais** de secrets en clair (utilisez des Secrets)
- Documentez les variables requises et leurs valeurs par défaut
- Les Pods doivent redémarrer pour prendre en compte les changements

La prochaine section abordera les **ConfigMaps**, une meilleure façon de gérer de grandes configurations.

---

**Prochaine section :** 4.5 Gestion des données sensibles

⏭️ [Gestion des données sensibles](/04-premiers-deploiements/05-gestion-des-donnees-sensibles.md)
